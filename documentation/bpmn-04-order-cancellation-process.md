# PROC-004 — Order Cancellation Process

**Process ID**: PROC-004  
**Source**: `OrdermanagementImpl.deleteOrder()` · `cancellationAllowed()` (Java)  
**Source (Node)**: `DeleteOrderUseCase.deleteOrder()` (Node.js)  
**Participants**: Customer / Guest · Order Management System  
**Trigger**: Customer clicks the cancellation link in the order confirmation email  

---

## Process Description

A customer or guest can cancel their placed order up until a configurable number of hours before the booking date (`mythaistar.hourslimitcancellation`). The cancellation window is calculated as:

```
cancellationAllowed = now < (bookingDate - hoursLimit × 3,600,000 ms)
```

If the window has passed, a `CancelNotAllowedException` is thrown. If allowed, all order lines are deleted first (referential integrity), then the order itself is removed.

---

## Main BPMN Diagram

```mermaid
flowchart TD
    %% ═══════════════════════════════════════════════════
    %% CUSTOMER LANE
    %% ═══════════════════════════════════════════════════
    subgraph CUST["🧑 Customer / Guest"]
        C_Start([Start: Click Cancel Order Link\nfrom Confirmation Email\nDELETE /order/orderId])
    end

    %% ═══════════════════════════════════════════════════
    %% ORDER SYSTEM LANE
    %% ═══════════════════════════════════════════════════
    subgraph SYS["⚙️ Order Management System"]
        S_FindOrder[Find Order by orderId\ngetOrderDao.find orderId]
        S_OrderQ{Order\nFound?}
        S_ErrNotFound[Throw WrongIdException\nOrder does not exist]

        S_FindBooking[Fetch Related Booking\nbookingManagement.findBooking\nbookingId]
        S_CalcWindow["Calculate Cancellation Window:\ncancellationLimit =\nbookingDate.toEpochMilli\n  - hoursLimit × 3,600,000\nnow = Instant.now().toEpochMilli()"]
        S_TimeQ{now < cancellationLimit\ni.e. still within\nwindow?}
        S_ErrLate[Throw CancelNotAllowedException\nCancellation window has passed]

        S_FindLines[Fetch All Order Lines\nfor This Order]
        S_DeleteLines[Delete Each OrderLine\norderLineDao.deleteById]
        S_DeleteOrder[Delete Order\norderDao.delete order]
    end

    %% ═══════════════════════════════════════════════════
    %% END EVENTS
    %% ═══════════════════════════════════════════════════
    END_OK([Order Cancelled Successfully\nLines and Order Deleted])
    END_NOT_FOUND([Error: WrongIdException\nOrder ID not found])
    END_TOO_LATE([Error: CancelNotAllowedException\nCancellation window has closed])

    %% ═══════════════════════════════════════════════════
    %% FLOW
    %% ═══════════════════════════════════════════════════
    C_Start --> S_FindOrder
    S_FindOrder --> S_OrderQ
    S_OrderQ -->|"Not found"| S_ErrNotFound
    S_ErrNotFound --> END_NOT_FOUND
    S_OrderQ -->|"Found"| S_FindBooking
    S_FindBooking --> S_CalcWindow
    S_CalcWindow --> S_TimeQ
    S_TimeQ -->|"now >= cancellationLimit\nToo late"| S_ErrLate
    S_ErrLate --> END_TOO_LATE
    S_TimeQ -->|"now < cancellationLimit\nAllowed"| S_FindLines
    S_FindLines --> S_DeleteLines
    S_DeleteLines --> S_DeleteOrder
    S_DeleteOrder --> END_OK

    %% ═══════════════════════════════════════════════════
    %% STYLING
    %% ═══════════════════════════════════════════════════
    style C_Start fill:#90EE90,stroke:#2E8B57,color:#000
    style END_OK fill:#90EE90,stroke:#2E8B57,color:#000
    style END_NOT_FOUND fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_TOO_LATE fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrNotFound fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrLate fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_OrderQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_TimeQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_CalcWindow fill:#87CEEB,stroke:#4169E1,color:#000
    style S_DeleteLines fill:#FFA07A,stroke:#FF4500,color:#000
    style S_DeleteOrder fill:#FFA07A,stroke:#FF4500,color:#000
```

---

## Cancellation Time Window Visualisation

```mermaid
gantt
    title Order Cancellation Window
    dateFormat YYYY-MM-DD HH:mm
    section Timeline
    Booking Created        :done,    b1, 2024-01-10 10:00, 1h
    Cancellation Allowed   :active,  c1, 2024-01-10 11:00, 83h
    CANCELLATION DEADLINE  :crit,    d1, 2024-01-14 18:00, 1h
    Booking Date           :milestone, m1, 2024-01-14 19:00, 0h
```

> **Note**: The `hoursLimit` value is configurable via `mythaistar.hourslimitcancellation` in `application.properties`. The diagram above assumes `hoursLimit = 1` for illustration.

---

## Deletion Cascade Detail

```mermaid
flowchart LR
    Order["Order\n id = orderId"] --> Lines["OrderLines\norderLineDao.findOrderLines\norderId"]
    Lines --> DeleteLines["DELETE each OrderLine\norderLineDao.deleteById\nlineId"]
    DeleteLines --> DeleteOrder["DELETE Order\norderDao.delete order"]

    style DeleteLines fill:#FFA07A,stroke:#FF4500,color:#000
    style DeleteOrder fill:#FFB6C1,stroke:#DC143C,color:#000
```

---

## Process Elements Summary

| Element Type | Count | Notes |
|-------------|-------|-------|
| Start Events | 1 | Customer clicks cancellation link |
| End Events | 3 | Success · Order Not Found · Too Late |
| Service Tasks | 5 | Find order · Find booking · Calc window · Delete lines · Delete order |
| Decision Gateways | 2 | Order found? · Within cancellation window? |

---

## Business Rules

| Rule | Detail |
|------|--------|
| Time window formula | `cancellationAllowed = now < bookingDate − (hoursLimit × 3,600,000 ms)` |
| Configuration key | `mythaistar.hourslimitcancellation` in `application.properties` |
| Deletion order | Order lines MUST be deleted before the parent order (referential integrity) |
| No email on cancel | Order cancellation does not send an email (booking cancellation does) |

---

## Exception Catalogue

| Exception | Trigger |
|-----------|---------|
| `WrongIdException` | Order ID not found in database |
| `CancelNotAllowedException` | Current time is past the cancellation deadline |
