# Garden Feature

This document provides a comprehensive overview of the **Garden** feature in **Juicy Forest**, covering backend architecture, database models, API endpoints, frontend structure, and data flow.

---

## Table of Contents

* Architecture Overview
* Backend

  * Microservice Structure
  * Database Models
* REST API Endpoints
* Services
* Gateway Configuration
* Data Samples & JSON Formats
* Frontend

  * Component Structure
  * State Management
* Garden Membership & Permissions
* Section Management
* Data Flow Diagrams
* Security & Validation

---

## Architecture Overview

```text
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

**Responsibilities**

* **Frontend**: Manages garden selection, creation, and section display
* **API Gateway**: Proxies garden-related requests
* **Server Service**: Handles garden logic and persistence
* **Database**: MongoDB collections for gardens and sections

---

## Backend

### Microservice Structure

Garden logic resides in the main **Server Service**:

```text
backend/server/
├── routes.js
├── controllers/
│   └── gardenController.js
├── models/
│   ├── Garden.js
│   └── GardenSection.js
└── services/
    └── gardenService.js
```

---

## Database Models

### Garden Model

Represents a single garden shared by one or more users.

| Field     | Type       | Description                 |
| --------- | ---------- | --------------------------- |
| name      | String     | Garden name                 |
| ownerId   | ObjectId   | User who created the garden |
| members   | ObjectId[] | Users with access           |
| createdAt | Date       | Creation timestamp          |
| updatedAt | Date       | Last update timestamp       |

**Indexes**

* `{ ownerId: 1, name: 1 }` – Prevents duplicate garden names per owner

---

### GardenSection Model

Represents a logical subdivision within a garden.

| Field    | Type     | Description             |
| -------- | -------- | ----------------------- |
| name     | String   | Section name            |
| gardenId | ObjectId | Parent garden reference |
| order    | Number   | Display order           |

**Indexes**

* `{ gardenId: 1, name: 1 }` – Unique section names per garden

---

## REST API Endpoints

All garden endpoints are accessed through the API Gateway using the `/gardens` prefix.

### Garden Endpoints

| Method | Endpoint     | Description                   |
| ------ | ------------ | ----------------------------- |
| GET    | /gardens     | Retrieve all gardens for user |
| POST   | /gardens     | Create a new garden           |
| GET    | /gardens/:id | Retrieve garden details       |
| PUT    | /gardens/:id | Update garden                 |
| DELETE | /gardens/:id | Delete garden                 |

### Section Endpoints

| Method | Endpoint              | Description       |
| ------ | --------------------- | ----------------- |
| GET    | /gardens/:id/sections | Retrieve sections |
| POST   | /gardens/:id/sections | Create section    |
| PUT    | /sections/:sectionId  | Update section    |
| DELETE | /sections/:sectionId  | Delete section    |

---

## Services

### Garden Service (`gardenService.js`)

| Function                    | Description                |
| --------------------------- | -------------------------- |
| getGardens(userId)          | Fetch gardens for user     |
| createGarden(data)          | Create garden              |
| updateGarden(id, data)      | Update garden              |
| deleteGarden(id)            | Delete garden and children |
| addMember(gardenId, userId) | Add member to garden       |

---

## Gateway Configuration

The API Gateway routes garden requests to the Server Service:

```js
const services = {
  server: {
    url: process.env.SERVER_SERVICE_URL || 'http://localhost:3031',
    routes: ['/gardens', '/sections']
  }
};
```

Requests to `http://localhost:3030/gardens` are forwarded to the Server Service.

---

## Data Samples & JSON Formats

### 1. GET /gardens

**Response (200 OK)**

```json
[
  {
    "_id": "6599b864...",
    "name": "Backyard Garden",
    "ownerId": "658aa123...",
    "members": ["658aa123...", "658bb456..."],
    "createdAt": "2024-01-10T12:00:00Z"
  }
]
```

---

### 2. POST /gardens

**Request Body**

```json
{
  "name": "Community Garden"
}
```

**Response (201 Created)**

```json
{
  "_id": "65aa999...",
  "name": "Community Garden",
  "ownerId": "658aa123...",
  "members": ["658aa123..."]
}
```

---

### 3. POST /gardens/:id/sections

**Request Body**

```json
{
  "name": "Greenhouse",
  "order": 1
}
```

**Response (201 Created)**

```json
{
  "_id": "65bb111...",
  "name": "Greenhouse",
  "gardenId": "65aa999...",
  "order": 1
}
```

---

## Frontend

### Component Structure

```text
client/src/
├── routes/gardens/
│   ├── +page.svelte
│   └── +page.js
├── lib/
│   ├── stores/
│   │   └── gardenStore.svelte.ts
│   └── components/Garden/
│       ├── GardenCreateModal.svelte
│       ├── GardenSwitcher.svelte
│       ├── GardenSettingsModal.svelte
│       ├── GardenSectionList.svelte
│       └── GardenSectionItem.svelte
```

---

## GardenCreationStore

The `GardenCreationStore` manages all state related to creating, editing, and deleting gardens using **Svelte 5 runes**. It centralizes modal control, form handling, validation, and API interactions for the full garden lifecycle.

### Reactive State Properties

| Property         | Type          | Description                      |
| ---------------- | ------------- | -------------------------------- |
| selectedGardenId | string | null | Currently selected garden ID     |
| isModalOpen      | boolean       | Controls garden modal visibility |
| modalMode        | string        | "create", "edit", or "delete"    |
| selectedGarden   | any           | Garden being edited or deleted   |
| formData         | object        | Form state for create/edit       |
| errors           | object        | Validation errors                |
| showToast        | boolean       | Toast visibility flag            |
| toastMessage     | object        | Toast title and message          |
| isSubmitting     | boolean       | Prevents duplicate submissions   |

### Key Methods

| Method          | Description                            |
| --------------- | -------------------------------------- |
| openCreateModal | Opens modal in create mode             |
| openEditModal   | Opens modal prefilled with garden data |
| openDeleteModal | Opens delete confirmation modal        |
| closeModal      | Closes modal and resets local state    |
| handleSubmit    | Routes create/update/delete logic      |
| createGarden    | Sends POST request                     |
| updateGarden    | Sends PUT request                      |
| deleteGarden    | Sends DELETE request                   |
| validateForm    | Validates inputs and populates errors  |
| resetForm       | Clears form and validation state       |

### Garden Form Data Shape

```ts
formData = {
  name: "",
  location: "",
  description: "",
  size: "",
  createdAt: null
};
```

---

## Context Usage

### Context Setup

```svelte
<script lang="ts">
  const selectedGardenId = $state({ value: null });
  setContext("selectedGardenId", selectedGardenId);
</script>
```

### Context Consumption

```svelte
<script lang="ts">
  const selectedGardenId = getContext("selectedGardenId");
</script>
```

This keeps inventory, task, and analytics pages synchronized with the active garden.

---

## Page Loading & Data Fetching

```ts
export const load = async ({ fetch }) => {
  const response = await fetch("http://localhost:3030/gardens");
  const gardens = await response.json();
  return { gardens };
};
```

```ts
const activeGarden = $derived(
  data.gardens.find((g) => g._id === gardenCreationStore.selectedGardenId)
);
```

---

## Form Validation

| Field       | Validation Rules                   |
| ----------- | ---------------------------------- |
| name        | Required, minimum 2 characters     |
| location    | Required, non-empty string         |
| description | Optional, max 500 characters       |
| size        | Optional, must be a positive value |

---

## Toast Notifications

* Garden created successfully
* Garden updated successfully
* Garden deleted successfully
* Validation or API error feedback

Toast state is automatically reset after dismissal.

---

## Data Flow Diagrams

### Creating a Garden

```text
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │   API   │           │ Server  │           │ MongoDB │
│   UI    │           │ Gateway │           │ Service │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │ POST /gardens       │                     │                     │
     │ {name, location}   │                     │                     │
     │────────────────────▶│                     │                     │
     │                     │ Forward to          │                     │
     │                     │ :3031/gardens       │                     │
     │                     │────────────────────▶│                     │
     │                     │                     │ createGarden()      │
     │                     │                     │────────────────────▶│
     │                     │                     │ Garden.create()     │
     │                     │                     │                     │
     │                     │                     │◀────────────────────│
     │                     │◀────────────────────│                     │
     │ 201 Created         │                     │                     │
     │◀────────────────────│                     │                     │

```

### Updating a Garden

```text
PUT /gardens/:id
→ updateGarden(id, data)
→ findByIdAndUpdate({ runValidators: true })
→ 200 OK
```

### Deleting a Garden

```text
DELETE /gardens/:id
→ deleteGarden(id)
→ findByIdAndDelete()
→ 204 No Content
```

---

## Security & Validation

### CORS

* Restricted to `CLIENT_URL` (default: `localhost:5173`)
* Credentials enabled for authenticated sessions

### Data Integrity

* Garden name must be unique per user
* Ownership checks on every mutation
* Schema-level validation on create and update

### Authorization

* All garden mutations require a valid authenticated session
* Ownership verified before update or delete operations

---

## Summary

The **GardenCreationStore** provides a centralized, maintainable state management layer for garden lifecycle operations. By combining Svelte 5 runes, context sharing, and strict validation, the Garden feature ensures consistency, security, and scalability across all garden-related workflows.
