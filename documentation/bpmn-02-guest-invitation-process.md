# PROC-002 — Guest Invitation Response Process

**Process ID**: PROC-002  
**Source**: `BookingmanagementImpl.acceptInvite()` · `BookingmanagementImpl.declineInvite()` (Java)  
**Source (Node)**: `UpdateInvitedGuestUseCase.updateInvitedGuest()` (Node.js)  
**Participants**: Guest · Booking Management System · Host · Email Service  
**Trigger**: Guest clicks Accept or Decline link in invitation email  

---

## Process Description

After a host creates a group booking (INVITED type), each invited guest receives an email containing:
- A personal **GB_ token** (unique per guest)
- An **Accept** link (`/booking/acceptInvite/GB_...`)
- A **Decline** link (`/booking/rejectInvite/GB_...`)

The guest's response determines whether they can later place a food order. Accepting sets `accepted = true`, which is a hard prerequisite for the order placement process. Declining cancels any already-placed orders and frees their slot.

The host is notified of every accept and decline action.

---

## Main BPMN Diagram

```mermaid
flowchart TD
    %% ═══════════════════════════════════════════════════
    %% GUEST LANE
    %% ═══════════════════════════════════════════════════
    subgraph GUEST["🧑 Invited Guest"]
        G_Email([Receive Invitation Email\nwith Accept + Decline Links])
        G_Decision{Guest\nDecision}
        G_Accept[Click Accept Link\nGET /booking/acceptInvite/GB_token]
        G_Decline[Click Decline Link\nGET /booking/rejectInvite/GB_token]
    end

    %% ═══════════════════════════════════════════════════
    %% SYSTEM LANE
    %% ═══════════════════════════════════════════════════
    subgraph SYS["⚙️ Booking Management System"]
        S_LookupA[Look Up InvitedGuest\nby GB_ Token]
        S_NullA{Guest\nRecord Found?}
        S_ErrA[Throw NullPointerException\nToken Invalid]

        S_SetAccepted[Set accepted = true\nSave InvitedGuest]

        S_LookupD[Look Up InvitedGuest\nby GB_ Token]
        S_NullD{Guest\nRecord Found?}
        S_ErrD[Throw NullPointerException\nToken Invalid]

        S_SetDeclined[Set accepted = false\nSave InvitedGuest]
        S_FindOrders[Find All Orders\nfor This InvitedGuest]
        S_OrderQ{Existing\nOrders?}
        S_DeleteOrders[Delete Each Order\nand Its Order Lines]
    end

    %% ═══════════════════════════════════════════════════
    %% EMAIL SERVICE LANE
    %% ═══════════════════════════════════════════════════
    subgraph EMAIL["📧 Email Service"]
        E_AcceptGuest[Send Acceptance Confirmation to Guest:\nHost Name · Booking Date\nGB_ Token · Decline Link]
        E_AcceptHost[Notify Host:\nGuest name has accepted your invitation]

        E_DeclineGuest[Send Decline Confirmation to Guest:\nHost Name · Booking Date]
        E_DeclineHost[Notify Host:\nGuest name has declined your invitation]
    end

    %% ═══════════════════════════════════════════════════
    %% END EVENTS
    %% ═══════════════════════════════════════════════════
    END_ACCEPTED([Guest Accepted\nCan Now Place Orders\nusing GB_ Token])
    END_DECLINED([Guest Declined\nOrders Cancelled\nSlot Released])
    END_ERR_A([Error: Invalid Token\nAccept failed])
    END_ERR_D([Error: Invalid Token\nDecline failed])

    %% ═══════════════════════════════════════════════════
    %% FLOW — ACCEPT PATH
    %% ═══════════════════════════════════════════════════
    G_Email --> G_Decision
    G_Decision -->|Accept| G_Accept
    G_Accept --> S_LookupA
    S_LookupA --> S_NullA
    S_NullA -->|Not found| S_ErrA
    S_ErrA --> END_ERR_A
    S_NullA -->|Found| S_SetAccepted
    S_SetAccepted --> E_AcceptGuest
    S_SetAccepted --> E_AcceptHost
    E_AcceptGuest --> END_ACCEPTED
    E_AcceptHost --> END_ACCEPTED

    %% ═══════════════════════════════════════════════════
    %% FLOW — DECLINE PATH
    %% ═══════════════════════════════════════════════════
    G_Decision -->|Decline| G_Decline
    G_Decline --> S_LookupD
    S_LookupD --> S_NullD
    S_NullD -->|Not found| S_ErrD
    S_ErrD --> END_ERR_D
    S_NullD -->|Found| S_SetDeclined
    S_SetDeclined --> S_FindOrders
    S_FindOrders --> S_OrderQ
    S_OrderQ -->|No orders| E_DeclineGuest
    S_OrderQ -->|Has orders| S_DeleteOrders
    S_DeleteOrders --> E_DeclineGuest
    E_DeclineGuest --> END_DECLINED
    E_DeclineGuest --> E_DeclineHost
    E_DeclineHost --> END_DECLINED

    %% ═══════════════════════════════════════════════════
    %% STYLING
    %% ═══════════════════════════════════════════════════
    style G_Email fill:#90EE90,stroke:#2E8B57,color:#000
    style END_ACCEPTED fill:#90EE90,stroke:#2E8B57,color:#000
    style END_DECLINED fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_ERR_A fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_ERR_D fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrA fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrD fill:#FFB6C1,stroke:#DC143C,color:#000
    style G_Decision fill:#FFD700,stroke:#B8860B,color:#000
    style S_NullA fill:#FFD700,stroke:#B8860B,color:#000
    style S_NullD fill:#FFD700,stroke:#B8860B,color:#000
    style S_OrderQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_SetAccepted fill:#87CEEB,stroke:#4169E1,color:#000
    style S_SetDeclined fill:#87CEEB,stroke:#4169E1,color:#000
    style S_DeleteOrders fill:#FFA07A,stroke:#FF4500,color:#000
    style E_AcceptGuest fill:#DDA0DD,stroke:#9370DB,color:#000
    style E_AcceptHost fill:#DDA0DD,stroke:#9370DB,color:#000
    style E_DeclineGuest fill:#DDA0DD,stroke:#9370DB,color:#000
    style E_DeclineHost fill:#DDA0DD,stroke:#9370DB,color:#000
```

---

## State Transition: InvitedGuest.accepted

```mermaid
stateDiagram-v2
    [*] --> Pending : GB_ token generated\naccepted = false
    Pending --> Accepted : acceptInvite called\naccepted = true
    Pending --> Declined : declineInvite called\naccepted = false
    Accepted --> Declined : declineInvite called\nexisting orders deleted
    Declined --> Accepted : acceptInvite called\naccepted = true
    Accepted --> [*] : Guest places order\norder created
    Declined --> [*] : Booking cancelled\nor event date passed
```

---

## Email Notification Matrix

| Event | Recipient | Subject | Key Content |
|-------|-----------|---------|-------------|
| Invitation sent (PROC-001) | Guest | "Event invite" | Host name, booking date, accept/decline links |
| Guest accepts | Guest | "Invite accepted" | Booking date, GB_ token, decline link for later change |
| Guest accepts | Host | "Invite accepted" | Which guest accepted |
| Guest declines | Guest | "Invite declined" | Host name, booking date |
| Guest declines | Host | "Invite declined" | Which guest declined |
| Booking cancelled (PROC-005) | Guest | "Event cancellation" | Host email, booking date |

---

## Process Elements Summary

| Element Type | Count | Notes |
|-------------|-------|-------|
| Start Events | 2 | Accept click / Decline click (both triggered by email link) |
| End Events | 4 | Accepted, Declined, Error ×2 |
| User Tasks | 2 | Click Accept, Click Decline |
| Service Tasks | 8 | Token lookup ×2, DB updates ×2, order deletion, emails ×4 |
| Decision Gateways | 4 | Guest decision · Record found? (×2) · Existing orders? |

---

## Business Rules

| Rule | Detail |
|------|--------|
| `accepted = true` is mandatory for placing an order | Enforced in `CreateOrderUseCase` — `GuestNotAcceptedException` thrown otherwise |
| Declining removes all guest orders | `declineInvite()` fetches and deletes all guest orders before saving `accepted=false` |
| Guests can change their mind | A guest who accepted can later decline (and vice versa) |
| Host always notified | Every accept/decline triggers a host notification email |
| Token required | Both accept and decline endpoints require a valid GB_ token |
