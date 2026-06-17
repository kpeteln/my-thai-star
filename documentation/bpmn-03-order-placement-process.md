# PROC-003 — Order Placement Process

**Process ID**: PROC-003  
**Source**: `OrdermanagementImpl.saveOrder()` · `getValidatedOrder()` (Java)  
**Source (Node)**: `CreateOrderUseCase.createOrder()` (Node.js)  
**Participants**: Customer / Guest · Order Management System · Email Service  
**Trigger**: Customer or Guest submits an order using their booking token  

---

## Process Description

A customer (or invited guest) places a food order by submitting their booking token alongside a list of chosen dishes and extra ingredients. The system enforces strict token-based access control before creating the order:

- **Host** uses their `CB_` token — the booking must exist and have no prior order.
- **Guest** uses their `GB_` token — the invite must exist, the guest must have accepted, and no prior order may exist.
- Prices are calculated server-side: `(dish.price + Σ extras.price) × quantity`.
- An itemised confirmation email is sent after successful order creation.

---

## Main BPMN Diagram

```mermaid
flowchart TD
    %% ═══════════════════════════════════════════════════
    %% CUSTOMER / GUEST LANE
    %% ═══════════════════════════════════════════════════
    subgraph USER["🧑 Customer / Guest"]
        U_Start([Start: Access Order Page\nwith Booking Token])
        U_Browse[Browse Menu\nView Dishes and Categories]
        U_Select[Select Dishes\nChoose Extra Ingredients\nSet Quantities]
        U_Submit[Submit Order\nPOST /order with token\nand order line items]
    end

    %% ═══════════════════════════════════════════════════
    %% ORDER SYSTEM LANE
    %% ═══════════════════════════════════════════════════
    subgraph SYS["⚙️ Order Management System"]
        S_ParseToken{Token\nPrefix?}
        S_ErrBadToken[Throw WrongTokenException\nToken does not start with CB_ or GB_]

        %% CB_ Path
        S_LookupBooking[Look Up Booking\nby CB_ Token]
        S_BookingQ{Booking\nFound?}
        S_ErrNoBooking[Throw NoBookingException]
        S_CheckHostOrder{Existing\nOrder for\nBooking?}
        S_ErrHostDupe[Throw OrderAlreadyExistException]
        S_SetBookingId[Set orderEntity.bookingId\n= booking.id]

        %% GB_ Path
        S_LookupGuest[Look Up InvitedGuest\nby GB_ Token]
        S_GuestQ{Guest\nFound?}
        S_ErrNoGuest[Throw NoInviteException]
        S_AcceptedQ{Guest\naccepted = true?}
        S_ErrNotAccepted[Throw GuestNotAcceptedException]
        S_CheckGuestOrder{Existing\nOrder for\nGuest?}
        S_ErrGuestDupe[Throw OrderAlreadyExistException]
        S_SetGuestId[Set orderEntity.bookingId\nSet orderEntity.invitedGuestId]

        %% Price calculation and save
        S_SaveOrder[Save Order Entity to DB]
        S_CalcPrice[For Each Order Line:\nlinePrice = dish.price x amount\nAdd extra.price for each selected extra\nTotal = sum of all line prices]
        S_SaveLines[Save Each OrderLine\nwith dishId · amount\nextras · comment]
    end

    %% ═══════════════════════════════════════════════════
    %% EMAIL SERVICE LANE
    %% ═══════════════════════════════════════════════════
    subgraph EMAIL["📧 Email Service"]
        E_Confirm[Send Order Confirmation Email:\nItemised dish list with quantities\nExtra ingredients per dish\nPer-line cost and total price\nCancellation link: /booking/cancelOrder/orderId]
    end

    %% ═══════════════════════════════════════════════════
    %% END EVENTS
    %% ═══════════════════════════════════════════════════
    END_OK([Order Created Successfully\nConfirmation Email Sent])
    END_BAD_TOKEN([Error: WrongTokenException\nUnrecognised token prefix])
    END_NO_BOOKING([Error: NoBookingException\nBooking not found for CB_ token])
    END_DUPE_HOST([Error: OrderAlreadyExistException\nBooking already has an order])
    END_NO_GUEST([Error: NoInviteException\nInvite not found for GB_ token])
    END_NOT_ACCEPTED([Error: GuestNotAcceptedException\nGuest must accept invitation first])
    END_DUPE_GUEST([Error: OrderAlreadyExistException\nGuest already has an order])

    %% ═══════════════════════════════════════════════════
    %% FLOW CONNECTIONS
    %% ═══════════════════════════════════════════════════
    U_Start --> U_Browse
    U_Browse --> U_Select
    U_Select --> U_Submit

    U_Submit --> S_ParseToken

    %% Bad token
    S_ParseToken -->|"Unknown prefix"| S_ErrBadToken
    S_ErrBadToken --> END_BAD_TOKEN

    %% CB_ path
    S_ParseToken -->|"Starts with CB_"| S_LookupBooking
    S_LookupBooking --> S_BookingQ
    S_BookingQ -->|"Not found"| S_ErrNoBooking
    S_ErrNoBooking --> END_NO_BOOKING
    S_BookingQ -->|"Found"| S_CheckHostOrder
    S_CheckHostOrder -->|"Order exists"| S_ErrHostDupe
    S_ErrHostDupe --> END_DUPE_HOST
    S_CheckHostOrder -->|"No order yet"| S_SetBookingId

    %% GB_ path
    S_ParseToken -->|"Starts with GB_"| S_LookupGuest
    S_LookupGuest --> S_GuestQ
    S_GuestQ -->|"Not found"| S_ErrNoGuest
    S_ErrNoGuest --> END_NO_GUEST
    S_GuestQ -->|"Found"| S_AcceptedQ
    S_AcceptedQ -->|"accepted = false"| S_ErrNotAccepted
    S_ErrNotAccepted --> END_NOT_ACCEPTED
    S_AcceptedQ -->|"accepted = true"| S_CheckGuestOrder
    S_CheckGuestOrder -->|"Order exists"| S_ErrGuestDupe
    S_ErrGuestDupe --> END_DUPE_GUEST
    S_CheckGuestOrder -->|"No order yet"| S_SetGuestId

    %% Common path after validation
    S_SetBookingId --> S_SaveOrder
    S_SetGuestId --> S_SaveOrder
    S_SaveOrder --> S_CalcPrice
    S_CalcPrice --> S_SaveLines
    S_SaveLines --> E_Confirm
    E_Confirm --> END_OK

    %% ═══════════════════════════════════════════════════
    %% STYLING
    %% ═══════════════════════════════════════════════════
    style U_Start fill:#90EE90,stroke:#2E8B57,color:#000
    style END_OK fill:#90EE90,stroke:#2E8B57,color:#000
    style END_BAD_TOKEN fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_NO_BOOKING fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_DUPE_HOST fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_NO_GUEST fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_NOT_ACCEPTED fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_DUPE_GUEST fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrBadToken fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrNoBooking fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrHostDupe fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrNoGuest fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrNotAccepted fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrGuestDupe fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ParseToken fill:#FFD700,stroke:#B8860B,color:#000
    style S_BookingQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_CheckHostOrder fill:#FFD700,stroke:#B8860B,color:#000
    style S_GuestQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_AcceptedQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_CheckGuestOrder fill:#FFD700,stroke:#B8860B,color:#000
    style S_CalcPrice fill:#87CEEB,stroke:#4169E1,color:#000
    style S_SaveOrder fill:#87CEEB,stroke:#4169E1,color:#000
    style S_SaveLines fill:#87CEEB,stroke:#4169E1,color:#000
    style E_Confirm fill:#DDA0DD,stroke:#9370DB,color:#000
```

---

## Price Calculation Detail

```mermaid
flowchart TD
    OrderLines[Order Line Items\nfrom Customer Request] --> ForEach[For Each Order Line]
    ForEach --> DishPrice["Fetch dish.price from DB\nbasePrice = dish.price"]
    DishPrice --> ExtrasLoop[For Each Selected Extra Ingredient]
    ExtrasLoop --> AddExtra["linePrice += extra.price"]
    AddExtra --> MultiplyQty["linePrice = linePrice × orderLine.amount"]
    MultiplyQty --> AddTotal["totalOrderPrice += linePrice"]
    AddTotal --> NextLine{More\nLines?}
    NextLine -->|Yes| ForEach
    NextLine -->|No| FinalPrice[Total Order Price\nAppended to Email]

    style OrderLines fill:#87CEEB,stroke:#4169E1,color:#000
    style FinalPrice fill:#90EE90,stroke:#2E8B57,color:#000
    style NextLine fill:#FFD700,stroke:#B8860B,color:#000
```

---

## Token Validation Decision Tree

```mermaid
flowchart TD
    Token[Incoming bookingToken] --> PrefixCheck{Token\nPrefix?}

    PrefixCheck -->|CB_| CB_Flow[COMMON / Host Flow]
    PrefixCheck -->|GB_| GB_Flow[INVITED / Guest Flow]
    PrefixCheck -->|Other| WrongToken[WrongTokenException\nHTTP 400]

    CB_Flow --> CB_1{Booking exists\nfor token?}
    CB_1 -->|No| NoBooking[NoBookingException\nHTTP 400]
    CB_1 -->|Yes| CB_2{booking.orderId\n= null?}
    CB_2 -->|No - already ordered| DupeOrder1[OrderAlreadyExistException\nHTTP 409]
    CB_2 -->|Yes - proceed| CB_OK[Set bookingId\nProceed to Save]

    GB_Flow --> GB_1{InvitedGuest\nexists for token?}
    GB_1 -->|No| NoInvite[NoInviteException\nHTTP 400]
    GB_1 -->|Yes| GB_2{guest.accepted\n= true?}
    GB_2 -->|No| NotAccepted[GuestNotAcceptedException\nHTTP 403]
    GB_2 -->|Yes| GB_3{guest.orderId\n= null?}
    GB_3 -->|No - already ordered| DupeOrder2[OrderAlreadyExistException\nHTTP 409]
    GB_3 -->|Yes - proceed| GB_OK[Set bookingId + invitedGuestId\nProceed to Save]

    style WrongToken fill:#FFB6C1,stroke:#DC143C,color:#000
    style NoBooking fill:#FFB6C1,stroke:#DC143C,color:#000
    style DupeOrder1 fill:#FFB6C1,stroke:#DC143C,color:#000
    style NoInvite fill:#FFB6C1,stroke:#DC143C,color:#000
    style NotAccepted fill:#FFB6C1,stroke:#DC143C,color:#000
    style DupeOrder2 fill:#FFB6C1,stroke:#DC143C,color:#000
    style CB_OK fill:#90EE90,stroke:#2E8B57,color:#000
    style GB_OK fill:#90EE90,stroke:#2E8B57,color:#000
    style PrefixCheck fill:#FFD700,stroke:#B8860B,color:#000
    style CB_1 fill:#FFD700,stroke:#B8860B,color:#000
    style CB_2 fill:#FFD700,stroke:#B8860B,color:#000
    style GB_1 fill:#FFD700,stroke:#B8860B,color:#000
    style GB_2 fill:#FFD700,stroke:#B8860B,color:#000
    style GB_3 fill:#FFD700,stroke:#B8860B,color:#000
```

---

## Process Elements Summary

| Element Type | Count | Notes |
|-------------|-------|-------|
| Start Events | 1 | Customer accesses order page |
| End Events | 7 | Success (×1), Exceptions (×6) |
| User Tasks | 3 | Browse, select, submit |
| Service Tasks | 8 | Token lookup, DB saves, price calc, email |
| Decision Gateways | 6 | Token prefix · booking found · order exists · guest found · accepted · guest order exists |

---

## Exception Catalogue

| Exception | HTTP | Trigger |
|-----------|------|---------|
| `WrongTokenException` | 400 | Token does not start with `CB_` or `GB_` |
| `NoBookingException` | 404 | No booking found for CB_ token |
| `OrderAlreadyExistException` | 409 | Booking or guest already has an order |
| `NoInviteException` | 404 | No invite found for GB_ token |
| `GuestNotAcceptedException` | 403 | Guest has not accepted the invitation yet |
