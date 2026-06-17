# PROC-008 — Menu Management Process

**Process ID**: PROC-008  
**Source**: `DishmanagementImpl` (Java) · `DishService` / `DishController` (inferred from domain model)  
**Source (Angular)**: `MenuService` · `MenuComponent`  
**Role Required**: `ROLE_Manager` for create/update/delete; Anonymous for read  
**Participants**: Manager · Dish Management System  
**Trigger**: Manager navigates to the dish management interface  

---

## Process Description

The Menu Management process allows a **Manager** to maintain the restaurant's dish catalogue. The menu is hierarchical:

- **Category** → groups of dishes (e.g., Starters, Mains, Desserts)
- **Dish** → individual menu items with name, price, description, and image
- **Extras (Ingredients)** → optional add-ons that a customer can select when ordering (each has its own price)

All customers (including anonymous users) can browse the menu. Only managers can modify it.

---

## Main BPMN Diagram

```mermaid
flowchart TD
    %% ═══════════════════════════════════════════════════
    %% MANAGER LANE
    %% ═══════════════════════════════════════════════════
    subgraph MGR["👔 Manager"]
        M_Start([Start: Navigate to\nDish Management UI])
        M_Action{Select\nAction}
        M_FillAdd[Fill Add Dish Form:\nName · Description · Price\nCategory · Image\nExtra Ingredients]
        M_SubmitAdd[Submit New Dish\nPOST /dish]
        M_SelectDish[Select Existing Dish]
        M_FillEdit[Edit Dish Fields:\nUpdate any attributes]
        M_SubmitEdit[Submit Updates\nPUT /dish/id]
        M_ConfirmDelete[Confirm Delete\nAre you sure?]
        M_SubmitDelete[Submit Delete\nDELETE /dish/id]
    end

    %% ═══════════════════════════════════════════════════
    %% DISH MANAGEMENT SYSTEM LANE
    %% ═══════════════════════════════════════════════════
    subgraph SYS["⚙️ Dish Management System"]
        S_AuthCheck{ROLE_Manager\nin JWT?}
        S_Reject[Return HTTP 403 Forbidden]

        S_ValidateAdd{New Dish\nData Valid?\nName + Price required}
        S_ErrAdd[Return Validation Error]
        S_SaveDish[Save DishEntity to DB\nwith CategoryId]
        S_SaveExtras[Save Extra Ingredients\nLink to DishEntity]
        S_SaveImage[Store Image Reference\nLink to DishEntity]

        S_FindDish[Fetch Dish by ID\ndishDao.find id]
        S_DishExistsQ{Dish\nFound?}
        S_ErrNotFound[Return HTTP 404\nDish Not Found]
        S_UpdateDish[Update DishEntity Fields\nSave to DB]

        S_FindDishDel[Fetch Dish by ID\nfor Delete]
        S_DishDelQ{Dish\nFound?}
        S_ErrNotFoundDel[Return HTTP 404]
        S_DeleteExtras[Remove Extra Ingredient Links]
        S_DeleteDishRecord[Delete DishEntity]

        S_RefreshMenu[Menu Updated\nNew data available\nvia GET /dish/search]
    end

    %% ═══════════════════════════════════════════════════
    %% END EVENTS
    %% ═══════════════════════════════════════════════════
    END_ADDED([Dish Added to Menu\nVisible to Customers])
    END_UPDATED([Dish Updated\nChanges Live])
    END_DELETED([Dish Removed from Menu])
    END_FORBIDDEN([Error: HTTP 403\nInsufficient Role])
    END_VALERR([Error: Validation Failed])
    END_NOTFOUND([Error: Dish Not Found])

    %% ═══════════════════════════════════════════════════
    %% FLOW
    %% ═══════════════════════════════════════════════════
    M_Start --> S_AuthCheck
    S_AuthCheck -->|"No ROLE_Manager"| S_Reject
    S_Reject --> END_FORBIDDEN
    S_AuthCheck -->|"Authorised"| M_Action

    %% ADD DISH PATH
    M_Action -->|"Add Dish"| M_FillAdd
    M_FillAdd --> M_SubmitAdd
    M_SubmitAdd --> S_ValidateAdd
    S_ValidateAdd -->|"Invalid"| S_ErrAdd
    S_ErrAdd --> END_VALERR
    S_ValidateAdd -->|"Valid"| S_SaveDish
    S_SaveDish --> S_SaveExtras
    S_SaveExtras --> S_SaveImage
    S_SaveImage --> S_RefreshMenu
    S_RefreshMenu --> END_ADDED

    %% EDIT DISH PATH
    M_Action -->|"Edit Dish"| M_SelectDish
    M_SelectDish --> S_FindDish
    S_FindDish --> S_DishExistsQ
    S_DishExistsQ -->|"Not found"| S_ErrNotFound
    S_ErrNotFound --> END_NOTFOUND
    S_DishExistsQ -->|"Found"| M_FillEdit
    M_FillEdit --> M_SubmitEdit
    M_SubmitEdit --> S_UpdateDish
    S_UpdateDish --> S_RefreshMenu
    S_RefreshMenu --> END_UPDATED

    %% DELETE DISH PATH
    M_Action -->|"Delete Dish"| M_SelectDish
    M_SelectDish --> S_FindDishDel
    S_FindDishDel --> S_DishDelQ
    S_DishDelQ -->|"Not found"| S_ErrNotFoundDel
    S_ErrNotFoundDel --> END_NOTFOUND
    S_DishDelQ -->|"Found"| M_ConfirmDelete
    M_ConfirmDelete --> M_SubmitDelete
    M_SubmitDelete --> S_DeleteExtras
    S_DeleteExtras --> S_DeleteDishRecord
    S_DeleteDishRecord --> S_RefreshMenu
    S_RefreshMenu --> END_DELETED

    %% ═══════════════════════════════════════════════════
    %% STYLING
    %% ═══════════════════════════════════════════════════
    style M_Start fill:#DDA0DD,stroke:#9370DB,color:#000
    style END_ADDED fill:#90EE90,stroke:#2E8B57,color:#000
    style END_UPDATED fill:#90EE90,stroke:#2E8B57,color:#000
    style END_DELETED fill:#90EE90,stroke:#2E8B57,color:#000
    style END_FORBIDDEN fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_VALERR fill:#FFB6C1,stroke:#DC143C,color:#000
    style END_NOTFOUND fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_Reject fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrAdd fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrNotFound fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_ErrNotFoundDel fill:#FFB6C1,stroke:#DC143C,color:#000
    style S_AuthCheck fill:#FFD700,stroke:#B8860B,color:#000
    style M_Action fill:#FFD700,stroke:#B8860B,color:#000
    style S_ValidateAdd fill:#FFD700,stroke:#B8860B,color:#000
    style S_DishExistsQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_DishDelQ fill:#FFD700,stroke:#B8860B,color:#000
    style S_SaveDish fill:#DDA0DD,stroke:#9370DB,color:#000
    style S_UpdateDish fill:#DDA0DD,stroke:#9370DB,color:#000
    style S_DeleteDishRecord fill:#FFA07A,stroke:#FF4500,color:#000
    style S_RefreshMenu fill:#87CEEB,stroke:#4169E1,color:#000
```

---

## Menu Data Model Hierarchy

```mermaid
flowchart TD
    subgraph Menu["Restaurant Menu"]
        Cat1[Category: Starters]
        Cat2[Category: Mains]
        Cat3[Category: Desserts]
        Cat4[Category: Drinks]
    end

    subgraph Dishes["Dishes  (DishEntity)"]
        D1["Dish: Tom Yum Soup\nprice: $8.50\ncategoryId → Starters"]
        D2["Dish: Pad Thai\nprice: $12.00\ncategoryId → Mains"]
        D3["Dish: Mango Sticky Rice\nprice: $6.00\ncategoryId → Desserts"]
    end

    subgraph Extras["Extra Ingredients  (IngredientEntity)"]
        E1["Extra: Extra Chili\nprice: $0.50"]
        E2["Extra: Tofu\nprice: $1.50"]
        E3["Extra: Shrimp\nprice: $2.00"]
        E4["Extra: Coconut Milk\nprice: $0.75"]
    end

    Cat1 --> D1
    Cat2 --> D2
    Cat3 --> D3

    D1 --> E1
    D2 --> E1
    D2 --> E2
    D2 --> E3
    D3 --> E4

    style Cat1 fill:#87CEEB,stroke:#4169E1,color:#000
    style Cat2 fill:#87CEEB,stroke:#4169E1,color:#000
    style Cat3 fill:#87CEEB,stroke:#4169E1,color:#000
    style Cat4 fill:#87CEEB,stroke:#4169E1,color:#000
    style D1 fill:#98FB98,stroke:#228B22,color:#000
    style D2 fill:#98FB98,stroke:#228B22,color:#000
    style D3 fill:#98FB98,stroke:#228B22,color:#000
    style E1 fill:#FFFACD,stroke:#DAA520,color:#000
    style E2 fill:#FFFACD,stroke:#DAA520,color:#000
    style E3 fill:#FFFACD,stroke:#DAA520,color:#000
    style E4 fill:#FFFACD,stroke:#DAA520,color:#000
```

---

## Customer Menu Browsing (Read-Only — No Auth Required)

```mermaid
flowchart LR
    Browse([Customer Opens Menu]) --> API["GET /dish/search\nDishSearchCriteriaTo\nOptional: category filter\nPaginated"]
    API --> Render["Render Menu Cards:\nDish image · Name\nDescription · Price\nAvailable extras"]
    Render --> Select["Customer Selects Dishes\nfor Order PROC-003"]

    style Browse fill:#90EE90,stroke:#2E8B57,color:#000
    style Select fill:#87CEEB,stroke:#4169E1,color:#000
    style API fill:#FFFACD,stroke:#DAA520,color:#000
```

---

## Process Elements Summary

| Element Type | Count | Notes |
|-------------|-------|-------|
| Start Events | 1 | Manager navigates to dish management |
| End Events | 6 | Added · Updated · Deleted · Forbidden · Validation Error · Not Found |
| User Tasks | 6 | Fill add form · Submit add · Select dish · Fill edit · Confirm delete · Submit delete |
| Service Tasks | 9 | Auth check · Validate · Save dish · Save extras · Save image · Find dish ×2 · Update · Delete |
| Decision Gateways | 5 | Auth? · Action choice · Data valid? · Dish exists? ×2 |

---

## Access Control Matrix

| Operation | ROLE_Customer | ROLE_Waiter | ROLE_Manager |
|-----------|:---:|:---:|:---:|
| Browse Menu (GET) | ✅ | ✅ | ✅ |
| View Dish Details (GET) | ✅ | ✅ | ✅ |
| Add New Dish (POST) | ❌ | ❌ | ✅ |
| Update Dish (PUT) | ❌ | ❌ | ✅ |
| Delete Dish (DELETE) | ❌ | ❌ | ✅ |
| Manage Categories | ❌ | ❌ | ✅ |
| Manage Extras | ❌ | ❌ | ✅ |
