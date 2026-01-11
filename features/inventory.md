# Inventory Feature

This document provides a comprehensive overview of the Inventory feature in Juicy Forest, covering the backend implementation, database models, and API structure.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Backend](#backend)
    - [Microservice Structure](#microservice-structure)
    - [Database Models](#database-models)
    - [REST API Endpoints](#rest-api-endpoints)
    - [Services](#services)
3. [Gateway Configuration](#gateway-configuration)
4. [Data Samples & JSON Formats](#data-samples--json-formats)
5. [Frontend](#frontend)
6. [Security & Validation](#security--validation)

---

## Architecture Overview

The Inventory feature is part of the core Server Service, managed through the API Gateway:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   SvelteKit     │────▶│   API Gateway   │────▶│  Server Service │
│   Frontend      │     │   (Port 3030)   │     │  (Port 3031)    │
│   (Port 5173)   │     └─────────────────┘     └────────┬────────┘
│                 │                                      │
└─────────────────┘                                      │
                                                         ▼
                                               ┌─────────────────┐
                                               │    MongoDB      │
                                               │  (juicy-forest) │
                                               └─────────────────┘
```

- **Frontend**: Handles user interaction for adding, updating, and viewing garden tools and supplies
- **API Gateway**: Proxies `/inventory` requests from the frontend to the Server Service
- **Server Service**: Manages business logic and database persistence for inventory items
- **Database**: MongoDB storage using the Inventory collection

---

## Backend

### Microservice Structure

The inventory logic is contained within the main server microservice:

```
backend/server/
├── routes.js                # Main router mapping /inventory to the controller
├── controllers/
│   └── inventoryController.js # REST endpoints for inventory CRUD
├── models/
│   └── Inventory.js         # Mongoose inventory schema
└── services/
    └── inventoryService.js   # Inventory business logic
```

---

### Database Models

#### Inventory Model

Represents a physical item (tool, seed, or supply) belonging to a specific garden.

| Field             | Type       | Description                                           |
|-------------------|------------|-------------------------------------------------------|
| `name`            | String     | Name of the item (e.g., "Tomato Seeds")              |
| `type`            | String     | Category (e.g., "Seeds", "Tools", "Fertilizer")       |
| `quantity`        | Number     | Current stock level (Minimum: 0)                      |
| `quantityType`    | String     | Unit of measurement (e.g., "Packets", "Units", "KG")  |
| `isImportant`     | Boolean    | Enables low-stock notifications                       |
| `desiredQuantity` | Number     | Threshold for low-stock alerts                        |
| `gardenId`        | ObjectId   | Reference to the parent Garden                        |

**Indexes:**
- `{ gardenId: 1, name: 1 }` - Unique compound index (case-insensitive) to prevent duplicate items within the same garden

---

### REST API Endpoints

All inventory endpoints are accessed through the API Gateway via the `/inventory` prefix.

| Method | Endpoint         | Description                   |
|--------|------------------|-------------------------------|
| GET    | `/inventory`     | Retrieve all inventory items  |
| POST   | `/inventory`     | Create a new inventory item   |
| PUT    | `/inventory/:id` | Update an existing item       |
| DELETE | `/inventory/:id` | Remove an item from database  |

---

### Services

#### Inventory Service (`inventoryService.js`)

| Function               | Description                                                  |
|------------------------|--------------------------------------------------------------|
| `getItems()`           | Fetches all items from the collection                        |
| `createItem(data)`     | Validates and persists a new item                            |
| `updateItem(id, data)` | Finds item by ID and applies updates with validation         |
| `deleteItem(id)`       | Removes the item document by ID                              |

---

## Gateway Configuration

The API Gateway (`backend/gateway/index.js`) acts as the entry point, routing requests based on the service map:

```javascript
const services = {
  server: {
    url: process.env.SERVER_SERVICE_URL || 'http://localhost:3031',
    routes: ['/inventory', ...]
  }
};
```

When the client calls `http://localhost:3030/inventory`, the Gateway forwards the request to `http://localhost:3031/inventory`.

---

## Data Samples & JSON Formats

### 1. GET /inventory

**Response (200 OK):**
```json
[
  {
    "_id": "65aa123...",
    "name": "Tomato Seeds",
    "type": "Seeds",
    "quantity": 50,
    "quantityType": "Packets",
    "isImportant": true,
    "desiredQuantity": 10,
    "gardenId": "6599b...",
    "__v": 0
  },
  {
    "_id": "65aa456...",
    "name": "Spade",
    "type": "Tools",
    "quantity": 2,
    "quantityType": "Units",
    "isImportant": false,
    "gardenId": "6599b...",
    "__v": 0
  }
]
```

### 2. POST /inventory

**Request Body:**
```json
{
  "name": "Fertilizer",
  "type": "Supplies",
  "quantity": 5,
  "quantityType": "KG",
  "isImportant": true,
  "desiredQuantity": 2,
  "gardenId": "6599b864..."
}
```

**Response (201 Created):**
```json
{
  "_id": "65aa789...",
  "name": "Fertilizer",
  "type": "Supplies",
  "quantity": 5,
  "quantityType": "KG",
  "isImportant": true,
  "desiredQuantity": 2,
  "gardenId": "6599b864...",
  "__v": 0
}
```

### 3. PUT /inventory/:id

**Request Body:**
```json
{
  "quantity": 4
}
```

**Response (200 OK):**
```json
{
  "_id": "65aa789...",
  "name": "Fertilizer",
  "type": "Supplies",
  "quantity": 4,
  "quantityType": "KG",
  "isImportant": true,
  "desiredQuantity": 2,
  "gardenId": "6599b864...",
  "__v": 0
}
```

### 4. DELETE /inventory/:id

**Response (204 No Content):**
(Empty Body)

### 5. Error Response

**Response (500 Internal Server Error):**
```json
{
  "message": "E11000 duplicate key error collection: juicy-forest.inventories index: gardenId_1_name_1"
}
```

---

## Frontend

### Component Structure

The inventory UI is built with SvelteKit and organized as follows:

```
client/src/
├── routes/inventory/
│   ├── +page.svelte           # Main inventory page
│   └── +page.js               # Data loader (fetches inventory)
├── lib/
│   ├── stores/
│   │   └── inventoryStore.svelte.ts # InventoryStore class (reactive state)
│   └── components/Inventory/
│       ├── InventoryCreateEditModal.svelte # Create/edit form modal
│       ├── InventoryDeleteModal.svelte     # Delete confirmation modal
│       ├── InventoryFilter.svelte          # Category filter sidebar
│       ├── InventoryHeader.svelte          # Page header with search
│       ├── InventoryItem.svelte            # Individual inventory item card
│       ├── InventoryItemList.svelte        # Filtered item list container
│       ├── InventorySearchBar.svelte       # Search input component
│       └── InventoryWarning.svelte         # Low stock alert banner
```

### InventoryStore

The `InventoryStore` class (`inventoryStore.svelte.ts`) manages all inventory state using Svelte 5 runes:

#### Reactive State Properties

| Property            | Type      | Description                              |
|---------------------|-----------|------------------------------------------|
| `selectedGardenId`  | `string`  | Currently active garden ID               |
| `isModalOpen`       | `boolean` | Modal visibility state                   |
| `modalMode`         | `string`  | Modal type: "create", "edit", "delete"   |
| `selectedItem`      | `any`     | Item being edited/deleted                |
| `formData`          | `object`  | Form state for create/edit operations    |
| `errors`            | `object`  | Form validation errors                   |
| `showToast`         | `boolean` | Toast notification visibility            |
| `toastMessage`      | `object`  | Toast title and message content          |

#### Key Methods

| Method                        | Description                                   |
|-------------------------------|-----------------------------------------------|
| `openCreateModal()`           | Opens modal in create mode                    |
| `openEditModal(item)`         | Opens modal in edit mode with item data       |
| `openDeleteModal(item)`       | Opens modal in delete mode                    |
| `closeModal()`                | Closes modal and resets state                 |
| `handleSubmit(inventory)`     | Processes create/edit/delete based on mode    |
| `validateForm()`              | Validates form data and sets errors           |

#### Context Usage

The inventory page uses Svelte context to share state between components:

```svelte
<!-- +page.svelte -->
<script lang="ts">
  const selectedInventoryType = $state({ selectedInventoryType: "all" });
  setContext("selectedInventoryType", selectedInventoryType);

  let searchBarInput = $state({value: ""});
  setContext("inventorySearchBarInput", searchBarInput);
</script>
```

Child components access it via:

```svelte
<script lang="ts">
  const selectedInventoryType = getContext("selectedInventoryType");
  const searchBarInput = getContext("inventorySearchBarInput");
</script>
```

---

### Page Loading

The inventory page uses SvelteKit's `+page.js` loader to fetch data on navigation:

```javascript
export const load = async ({fetch}) => {
    const inventoryItems = await fetch('http://localhost:3030/inventory/');
    const inventoryItemsJson = await inventoryItems.json();
    return {inventory: inventoryItemsJson}
}
```

This data is then filtered by garden ID on the client:

```svelte
const itemsForThisGarden = $derived(
    data.inventory.filter((item) => item.gardenId === inventoryStore.selectedGardenId)
);
```

---

### Filtering & Search

The inventory supports multiple filtering mechanisms:

#### Category Filtering

Users can filter by:
- **All Items** - Shows everything in stock
- **Important Items** - Items marked with `isImportant: true`
- **Plants** - Items with `type: "plant"`
- **Seeds** - Items with `type: "seed"`
- **Tools** - Items with `type: "tool"`
- **Supplies** - Items with `type: "supply"`

Filter logic in `InventoryItemList.svelte`:

```svelte
const filteredItems = $derived(
    selectedInventoryType.selectedInventoryType === "all"
        ? inventory
        : inventory.filter((item) =>
            selectedInventoryType.selectedInventoryType === true
                ? item.isImportant
                : item.type === selectedInventoryType.selectedInventoryType
        )
);
```

#### Search Filtering

The search bar filters items by name (case-insensitive):

```svelte
const filteredSearchItems = $derived(
    searchBarInput.value != ""
        ? filteredItems.filter((item) => 
            item.name.toLowerCase().includes(searchBarInput.value.toLowerCase())
        )
        : filteredItems
);
```

---

### Low Stock Alerts

The `InventoryWarning` component displays a collapsible alert banner when items fall below their desired quantity:

```svelte
const warningItems = $derived(
    inventory.filter((item) => 
        item.quantity < item.desiredQuantity && item.isImportant === true
    )
);
```

Features:
- Only shows for items marked as important (`isImportant: true`)
- Displays current quantity vs. desired quantity
- Collapsible to minimize screen space
- Grid layout showing all affected items

---

### Form Validation

The create/edit modal validates the following fields:

| Field              | Validation Rules                          |
|--------------------|-------------------------------------------|
| `name`             | Required, non-empty string                |
| `quantity`         | Required, must be ≥ 0                     |
| `desiredQuantity`  | Required if `isImportant` is true, ≥ 0    |

Validation runs on form submit and displays inline error messages below invalid fields.

---

## Data Flow Diagrams

### Creating an Inventory Item

```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │   API   │           │ Server  │           │ MongoDB │
│   UI    │           │ Gateway │           │ Service │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ POST /inventory     │                     │                     │
     │ {name, type,        │                     │                     │
     │  quantity, ...}     │                     │                     │
     │─────────────────────▶│                     │                     │
     │                     │ Forward to          │                     │
     │                     │ :3031/inventory     │                     │
     │                     │─────────────────────▶│                     │
     │                     │                     │ createItem()        │
     │                     │                     │─────────────────────▶│
     │                     │                     │ Inventory.create()  │
     │                     │                     │                     │
     │                     │                     │◀─────────────────────│
     │                     │                     │ New item document   │
     │                     │◀─────────────────────│                     │
     │                     │ 201 Created         │                     │
     │◀─────────────────────│                     │                     │
     │ {_id, name, ...}    │                     │                     │
     │                     │                     │                     │
```

### Updating an Inventory Item

```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │   API   │           │ Server  │           │ MongoDB │
│   UI    │           │ Gateway │           │ Service │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ PUT /inventory/:id  │                     │                     │
     │ {quantity: 4}       │                     │                     │
     │─────────────────────▶│                     │                     │
     │                     │ Forward to          │                     │
     │                     │ :3031/inventory/:id │                     │
     │                     │─────────────────────▶│                     │
     │                     │                     │ updateItem(id, data)│
     │                     │                     │─────────────────────▶│
     │                     │                     │ findByIdAndUpdate() │
     │                     │                     │ with runValidators  │
     │                     │                     │                     │
     │                     │                     │◀─────────────────────│
     │                     │                     │ Updated document    │
     │                     │◀─────────────────────│                     │
     │                     │ 200 OK              │                     │
     │◀─────────────────────│                     │                     │
     │ Updated item        │                     │                     │
     │                     │                     │                     │
```

### Deleting an Inventory Item

```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │   API   │           │ Server  │           │ MongoDB │
│   UI    │           │ Gateway │           │ Service │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ DELETE              │                     │                     │
     │ /inventory/:id      │                     │                     │
     │─────────────────────▶│                     │                     │
     │                     │ Forward to          │                     │
     │                     │ :3031/inventory/:id │                     │
     │                     │─────────────────────▶│                     │
     │                     │                     │ deleteItem(id)      │
     │                     │                     │─────────────────────▶│
     │                     │                     │ findByIdAndDelete() │
     │                     │                     │                     │
     │                     │                     │◀─────────────────────│
     │                     │                     │ Deletion confirmed  │
     │                     │◀─────────────────────│                     │
     │                     │ 204 No Content      │                     │
     │◀─────────────────────│                     │                     │
     │ (Empty response)    │                     │                     │
     │                     │                     │                     │
```

---

## Security & Validation

1. **CORS**: The Gateway restricts access to the `CLIENT_URL` (default: `localhost:5173`) and allows credentials for authenticated sessions
2. **Data Integrity**:
    - `quantity` cannot be less than 0
    - `gardenId` is required for every item to ensure data isolation between gardens
    - Compound index on `gardenId` and `name` prevents duplicate items within the same garden
3. **Validation**: The `updateItem` service uses `{ runValidators: true }` to ensure that partial updates still adhere to the Mongoose schema constraints

---