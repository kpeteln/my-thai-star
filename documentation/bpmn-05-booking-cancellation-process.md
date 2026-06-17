# PROC-005 — Booking Cancellation Process

**Process ID**: PROC-005  
**Source**: `BookingmanagementImpl.cancelInvite()` · `deleteBooking()` · `cancelInviteAllowed()` (Java)  
**Source (Node)**: `CancelBookingUseCase.cancelBooking()` (Node.js)  
**Participants**: Customer · Booking Management System · Email Service  
**Trigger**: Host clicks the cancellation link in the booking confirmation email  

---

## Process Description

The booking host can cancel an entire booking (and all associated guest invitations and orders) within the cancellation window. The process is a **cascade delete** that:

1. Validates the booking token and time window
2. Deletes each invited guest's orders first
3. Deletes the invited guest records
4. Deletes the booking's own orders
5. Deletes the booking itself
6. Sends cancellation emails to the host and all guests

The time window uses the same formula as order cancellation:

```
cancelAllowed = now < (bookingDate - hoursLimit × 3,600,000 ms)
```

In the Node.js implementation, the expiration date stored at booking creation time is compared directly:

```
cancelAllowed = moment().isBefore(moment(booking.expirationDate))
```

---

## Main BPMN Diagram

```mermaid
flowchart TD
    %% ═══════════════════════════════════════════════════
    %% CUSTOMER LANE
    %% ═══════════════════════════════════════════════════
    subgraph CUST["🧑 Booking Host (Customer)"]
        C_Start([Start: Click Cancel Booking Link\nfrom Confirmation Email\nGET /booking/cancel/CB_token])
    end

    %% ═══════════════════════════════════════════════════
    %% BOOKING SYSTEM LANE
    %% ═══════════════════════════════════════════════════
    subgraph SYS["⚙️ Booking Management System"]
        S_FindBooking[Find Booking by CB_ Token\nbookingDao.findBookingByToken token]
        S_TokenQ{Booking\nFound?}
        S_ErrToken[Throw InvalidTokenException\nToken not recognised]

        S_TimeCheck["Check Cancellation Window:\nJava: now < bookingDate - hoursLimit × 3600000\nNode: moment().isBefore booking.expirationDate"]
        S_TimeQ{Within\nCancellation\nWindow?}
        S_ErrLate[Throw CancelInviteNotAllowedException\nCancellation window has closed]

        S_FindGuests[Fetch All InvitedGuests\nfor This Booking]
        S_GuestQ{Any\nInvited\nGuests?}

        S_GuestLoop["For Each Invited Guest:\n1. Find guest orders\n2. Delete guest orders\n3. Delete guest record"]

        S_DeleteBookingOrders[Find and Delete\nAll Orders for Booking]
        S_DeleteBooking[Delete Booking Record]
    end

    %% ═══════════════════════════════════════════════════
    %% EMAIL SERVICE LANE
    %% ═══════════════════════════════════════════════════
    subgraph EMAIL["📧 Email Service"]
        E_GuestMail[Send Cancellation Email\nto Each Guest:\nEvent has been cancelled\nHost email · Booking date]
        E_HostMail[Send Cancellation Email\nto Host:\nEvent has been cancelled\nBooking date]
    end

    %% ═══════════════════════════════════════════════════
    %% END EVENTS
    %% ═══════════════════════════════════════════════════
    END_OK([Booking Cancelled\nAll Guests and Orders Removed])
    END_BAD_TOKEN([Error: InvalidTokenException\nBooking not found])
    END_TOO_LATE([Error: CancelInviteNotAllowedException\nCancellation window closed])

    %% ═══════════════════════════════════════════════════
    %% FLOW
    %% ═══════════════════════════════════════════════════
    C_Start --> S_FindBooking
    S_FindBooking --> S_TokenQ
    S_TokenQ -->|"Not found"| S_ErrToken
    S_ErrToken --> END_BAD_TOKEN

    S_TokenQ -->|"Found"| S_TimeCheck
    S_TimeCheck --> S_TimeQ
    S_TimeQ -->|"Too late"| S_ErrLate
    S_ErrLate --> END_TOO_LATE

    S_TimeQ -->|"Allowed"| S_FindGuests
    S_FindGuests --> S_GuestQ
    S_GuestQ -->|"Has guests"| S_GuestLoop
    S_GuestLoop --> E_GuestMail
    S_GuestLoop --> S_DeleteBookingOrders
    S_GuestQ -->|"No guests"| S_DeleteBookingOrders

    S_DeleteBookingOrders --> S_DeleteBooking
    S_DeleteBooking --> E_HostMail
    E_GuestMail --> END_OK
    E_HostMail --> END_OK

    %% ═══════════════════════════════════════════════════
    %% STYLING
    %% ═══════════════════════════════════════════════════
    style C_Start fill:#90EE90,stroke:#2E8B57,color:#000
    style END_OK fill:#90EE90,stroke:#2E8B57,color:#000
    style END_BAD_TOKEN fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_TOO_LATE fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrToken fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrLate fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_TokenQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_TimeQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_GuestQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_GuestLoop fill:#FFA07A,stroke:#FF4500,color:#000
    style S_DeleteBookingOrders fill:#FFA07A,stroke:#FF4500,color:#000
    style S_DeleteBooking fill:#FFA07A,stroke:#FF4500,color:#000
    style E_GuestMail fill:#DDA0DD,stroke:#9370DB,color:#000
    style E_HostMail fill:#DDA0DD,stroke:#9370DB,color:#000
```

---

## Cascade Delete Sequence

```mermaid
flowchart LR
    B["Booking\ntoken=CB_..."] --> IG["InvitedGuests\nList for Booking"]
    B --> BO["Booking Orders\nList for Booking"]

    IG --> IGO["Each Guest's Orders"]
    IGO --> D1["DELETE OrderLines\nper Guest Order"]
    D1 --> D2["DELETE Guest Order"]
    D2 --> D3["DELETE InvitedGuest\nrecord"]

    BO --> D4["DELETE OrderLines\nper Booking Order"]
    D4 --> D5["DELETE Booking Order"]

    D3 --> D6["DELETE Booking\nrecord"]
    D5 --> D6

    style D1 fill:#FFA07A,stroke:#FF4500,color:#000
    style D2 fill:#FFA07A,stroke:#FF4500,color:#000
    style D3 fill:#FFB6C1,stroke:#DC143C,color:#000
    style D4 fill:#FFA07A,stroke:#FF4500,color:#000
    style D5 fill:#FFA07A,stroke:#FF4500,color:#000
    style D6 fill:#FFB6C1,stroke:#DC143C,color:#000
```

---

## Implementation Differences: Java vs Node.js

| Aspect | Java Implementation | Node.js Implementation |
|--------|--------------------|-----------------------|
| Time check | Calculate at runtime: `now < bookingDate − hoursLimit × 3.6M` | Compare against stored `expirationDate`: `moment().isBefore(expirationDate)` |
| Cascade | Manual iteration over guests and orders | `bookingRepository.deleteCascadeBooking(booking)` |
| Email trigger | Inside main method after each guest deleted | After all deletes, `bookingMailerService.sendCanceledEmails(booking, invitedGuests)` |

---

## Process Elements Summary

| Element Type | Count | Notes |
|-------------|-------|-------|
| Start Events | 1 | Host clicks cancel link |
| End Events | 3 | Success · Invalid Token · Too Late |
| Service Tasks | 6 | Find booking, time check, find guests, guest loop (delete cascade), delete orders, delete booking |
| Message Flows | 2 | Host cancellation email, guest cancellation emails |
| Decision Gateways | 3 | Booking found? · Within window? · Has guests? |

---

## Business Rules

| Rule | Detail |
|------|--------|
| Time window | Java: `now < bookingDate − (hoursLimit × 3,600,000 ms)` |
| Expiration field | Node: `expirationDate = bookingDate − 1 hour` (stored at creation time) |
| Cascade order | Guest orders → Guest records → Booking orders → Booking |
| Email on cancel | Both host and all guests receive cancellation notifications |
| Config key | `mythaistar.hourslimitcancellation` in `application.properties` |

---

## Exception Catalogue

| Exception | Trigger |
|-----------|---------|
| `InvalidTokenException` | CB_ token not found in database |
| `CancelInviteNotAllowedException` | Current time is past the cancellation deadline |
