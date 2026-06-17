# PROC-007 — Waiter Cockpit Process

**Process ID**: PROC-007  
**Source**: `BookingmanagementImpl.findBookingsByPost()` · `OrdermanagementImpl.findOrdersByPost()` (Java)  
**Source (Angular)**: `WaiterCockpitService` · `ReservationCockpitComponent`  
**Role Required**: `ROLE_Waiter` (Java: `@RolesAllowed(PERMISSION_FIND_BOOKING)`, `@RolesAllowed(PERMISSION_FIND_ORDER)`)  
**Participants**: Waiter · System  
**Trigger**: Waiter logs in and navigates to the Waiter Cockpit  

---

## Process Description

The Waiter Cockpit is the operational view for restaurant staff. A waiter (or manager) can:

1. Log in with their role credentials to receive a JWT token that includes `ROLE_Waiter`
2. Navigate to the Waiter Cockpit UI
3. Select a date to view all bookings for that day
4. Inspect individual bookings and their associated orders
5. Filter and sort the reservation list

All data access is read-only from the waiter's perspective — they view but do not modify bookings or orders through this interface.

---

## Main BPMN Diagram

```mermaid
flowchart TD
    %% ═══════════════════════════════════════════════════
    %% WAITER LANE
    %% ═══════════════════════════════════════════════════
    subgraph WAITER["👔 Waiter"]
        W_Start([Start: Navigate to Waiter Cockpit\nMust be authenticated])
        W_SelectDate[Select Date\nDate Picker UI]
        W_ViewList[View Booking List\nfor Selected Date]
        W_FilterSort[Apply Filters / Sort\nBy time · party size\nguest name · status]
        W_SelectBooking[Click on a Booking\nto Expand Details]
        W_ViewOrders[View Orders\nfor Selected Booking]
        W_ViewDishes[View Dish Details\nper Order Line]
        W_End([End: Complete Daily Review])
    end

    %% ═══════════════════════════════════════════════════
    %% SYSTEM LANE
    %% ═══════════════════════════════════════════════════
    subgraph SYS["⚙️ System"]
        S_AuthCheck{ROLE_Waiter\nPresent in JWT?}
        S_Reject[Return HTTP 403 Forbidden\nAccess Denied]
        S_FetchBookings["Fetch Bookings by Date Criteria\nPOST /booking/search\nBookingSearchCriteriaTo\nPaginated response"]
        S_BookingQ{Results\nFound?}
        S_EmptyState[Display Empty State\nNo bookings for this date]
        S_RenderList[Render Paginated Booking List\nBooking token · Host name\nDate · Party size · Guest count]
        S_FetchOrders["Fetch Orders for Booking\nPOST /order/search\nOrderSearchCriteriaTo"]
        S_RenderOrders[Render Order Details\nOrder lines · Dishes\nExtras · Quantities · Prices]
    end

    %% ═══════════════════════════════════════════════════
    %% END EVENTS
    %% ═══════════════════════════════════════════════════
    END_FORBIDDEN([Error: HTTP 403\nInsufficient Role])

    %% ═══════════════════════════════════════════════════
    %% FLOW
    %% ═══════════════════════════════════════════════════
    W_Start --> S_AuthCheck
    S_AuthCheck -->|"No ROLE_Waiter"| S_Reject
    S_Reject --> END_FORBIDDEN

    S_AuthCheck -->|"Authorised"| W_SelectDate
    W_SelectDate --> S_FetchBookings
    S_FetchBookings --> S_BookingQ
    S_BookingQ -->|"No bookings"| S_EmptyState
    S_EmptyState --> W_SelectDate
    S_BookingQ -->|"Has bookings"| S_RenderList
    S_RenderList --> W_ViewList
    W_ViewList --> W_FilterSort
    W_FilterSort --> W_SelectBooking
    W_SelectBooking --> S_FetchOrders
    S_FetchOrders --> S_RenderOrders
    S_RenderOrders --> W_ViewOrders
    W_ViewOrders --> W_ViewDishes
    W_ViewDishes --> W_End

    %% ═══════════════════════════════════════════════════
    %% STYLING
    %% ═══════════════════════════════════════════════════
    style W_Start fill:#B0C4DE,stroke:#4682B4,color:#000
    style W_End fill:#90EE90,stroke:#2E8B57,color:#000
    style END_FORBIDDEN fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_Reject fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_AuthCheck fill:#FFD700,stroke:#B8860B,color:#000
    style S_BookingQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_FetchBookings fill:#87CEEB,stroke:#4169E1,color:#000
    style S_FetchOrders fill:#87CEEB,stroke:#4169E1,color:#000
    style S_RenderList fill:#87CEEB,stroke:#4169E1,color:#000
    style S_RenderOrders fill:#87CEEB,stroke:#4169E1,color:#000
    style S_EmptyState fill:#FFFACD,stroke:#DAA520,color:#000
```

---

## Booking Search Criteria Flow

```mermaid
flowchart LR
    Input["Waiter Selects:\nDate\nOptional: booking name\nOptional: email"] --> Criteria["BookingSearchCriteriaTo\nbookingDate = selected date\npageable = page 0, size 10"]
    Criteria --> API["POST /booking/search\nRequires PERMISSION_FIND_BOOKING"]
    API --> Results["Page of BookingCto\nBooking + Table\n+ InvitedGuests\n+ Orders + User"]
    Results --> UI["Rendered as\nPaginated Table\nin Angular UI"]

    style API fill:#87CEEB,stroke:#4169E1,color:#000
    style Results fill:#90EE90,stroke:#2E8B57,color:#000
```

---

## Paginated Order View

```mermaid
flowchart LR
    SelectBooking["Waiter selects\na booking row"] --> FetchOrders["POST /order/search\nOrderSearchCriteriaTo\nbookingId = selected ID\nRequires PERMISSION_FIND_ORDER"]
    FetchOrders --> OrderList["Page of OrderCto:\nOrder\n+ BookingEto\n+ InvitedGuestEto\n+ OrderLineCtos"]
    OrderList --> Lines["Each OrderLineCto:\nDish name + price\nExtras list\nQuantity · Comment"]
    Lines --> Display["Rendered Order Detail\nPanel in UI"]

    style FetchOrders fill:#87CEEB,stroke:#4169E1,color:#000
    style OrderList fill:#90EE90,stroke:#2E8B57,color:#000
```

---

## Process Elements Summary

| Element Type | Count | Notes |
|-------------|-------|-------|
| Start Events | 1 | Waiter navigates to cockpit |
| End Events | 2 | Success (daily review done) · Forbidden (wrong role) |
| User Tasks | 5 | Select date · View list · Filter/sort · Select booking · View orders |
| Service Tasks | 4 | Auth check · Fetch bookings · Render list · Fetch orders · Render orders |
| Decision Gateways | 2 | ROLE_Waiter present? · Bookings found? |

---

## Security Requirements

| Permission | Java Annotation | Grants Access To |
|-----------|----------------|-----------------|
| `PERMISSION_FIND_BOOKING` | `@RolesAllowed(...)` on `findBookingsByPost` | All booking search |
| `PERMISSION_FIND_ORDER` | `@RolesAllowed(...)` on `findOrdersByPost` | All order search |

Both permissions are held by `ROLE_Waiter` and `ROLE_Manager`.  
`ROLE_Customer` can only access their own booking/order data via token.
