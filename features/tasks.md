# Tasks Feature

This document provides a comprehensive overview of the Tasks feature in Juicy Forest, covering the backend implementation, database models, and API structure.

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

The Tasks feature is part of the core Server Service, managed through the API Gateway:


```
┌─────────────────┐     ┌─────────────────┐      ┌─────────────────┐
│   SvelteKit     │────▶│   API Gateway     │────▶│  Server Service   │
│   Frontend      │     │   (Port 3030)   │        │  (Port 3031)      │
│   (Port 5173)   │     └─────────────────┘         └────────┬────────┘
│                 │                                           │
└─────────────────┘                                          │
                                                              ▼
                                               ┌─────────────────┐
                                               │    MongoDB       │
                                               │  (juicy-forest)  │
                                               └─────────────────┘
```

- **Frontend**: Handles user interaction for adding, updating, and toggling completion states of tasks.
- **API Gateway**: Proxies `/tasks` requests from the frontend to the Server Service (Port 3031).
- **Server Service**: Manages business logic and database persistence for tasks.
- **Database**: MongoDB storage using the Tasks collection.

---

## Backend

### Microservice Structure

The task logic is contained within the main server microservice:

```
backend/server/
├── routes.js                # Main router mapping /tasks to the controller
├── controllers/
│   └── tasksController.js   # REST endpoints for tasks CRUD and toggle
├── models/
│   └── Tasks.js             # Mongoose task schema
└── services/
    └── taskService.js       # Task business logic and DB interaction
```
---

### Database Models

#### Task Model

Represents a specific task or to-do item associated with a garden section.

| Field         | Type      | Description                               |
|---------------|-----------|-------------------------------------------|
| `name`        | String    | Name/Title of the task (Required)         |
| `description` | String    | Optional details about the task           |
| `isComplete`  | Boolean   | Completion status (Default: `false`)      |
| `sectionId`   | String    | Reference to the parent Section (Required)|

**Indexes:**
- `{ sectionId: 1 }` - Optimized for fast retrieval of tasks within a specific garden section.

---

### REST API Endpoints

All inventory endpoints are accessed through the API Gateway via the `/inventory` prefix.

| Method | Endpoint             | Description                                   |
|--------|----------------------|-----------------------------------------------|
| GET    | `/tasks`             | Retrieve all tasks(Filterable by sectionId)   |
| POST   | `/tasks`             | Create a new tasks                            |
| PUT    | `/tasks/:id`         | Update an existing task                       |
| PUT    | `/tasks/:id/toggle`  | Toggle the isComplete status                  |
| DELETE | `/tasks/:id`         | Remove a task from database                   |

---

### Services

#### Task Service (`taskService.js`)

| Function               | Description                                                          |
|------------------------|----------------------------------------------------------------------|
| `getTasks(sectionId)`  | Fetches tasks. If sectionId is provided, it filters by that section  |
| `createTask(data)`     | Persists a new task to the database.                                 |
| `updateTask(id, data)` | Finds task by ID and applies updates with schema validation.         |
| `toggleCheckBox(id)`   | Specifically toggles the isComplete boolean and saves the document.  |
| `deleteItem(id)`       | Removes the task document by ID                                      |

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

When the client calls `http://localhost:3030/tasks`, the Gateway forwards the request to `http://localhost:3031/tasks`.

---

## Data Samples & JSON Formats

### 1. GET /tasks

**Response (200 OK):**
```json
[
    {
        "_id": "695ba1caab4eafd87d779859",
        "name": "carrots 2",
        "description": "carrots 2",
        "isComplete": false,
        "sectionId": "695ba1a2ab4eafd87d779836",
        "__v": 0
    },
    {
        "_id": "695ba1fbab4eafd87d779873",
        "name": "onions",
        "description": "onions",
        "isComplete": false,
        "sectionId": "695ba1f0ab4eafd87d779866",
        "__v": 0
    },
    {
        "_id": "695ba2adab4eafd87d7798a6",
        "name": "olives",
        "description": "olives",
        "isComplete": false,
        "sectionId": "695ba2a0ab4eafd87d779899",
        "__v": 0
    }
]
```

### 2. POST /tasks

**Request Body:**
```json
{
    "name": "Prune Tomato plants",
    "sectionId": "695ba1f0ab4eafd87d779866",
    "description": "Prune Tomato plants to encourage better growth and fruit production"
}
```

**Response (201 Created):**
```json
{
    "name": "Prune Tomato plants",
    "description": "Prune Tomato plants to encourage better growth and fruit production",
    "isComplete": false,
    "sectionId": "695ba1f0ab4eafd87d779866",
    "_id": "6963ac55ea9440431b277515",
    "__v": 0
}
```

### 3. PUT /tasks/:id

**Request Body:**
```json
{
    "name": "Prune Tomato"
}
```

**Response (200 OK):**
```json
{
    "_id": "6963ac55ea9440431b277515",
    "name": "Prune Tomato",
    "description": "Prune Tomato plants to encourage better growth and fruit production",
    "isComplete": false,
    "sectionId": "695ba1f0ab4eafd87d779866",
    "__v": 0
}
```

### 4. DELETE /tasks/:id

**Response (204 No Content):**
(Empty Body)

---

## Frontend

### Component Structure

The tasks UI is built with SvelteKit and organized as follows:

```
client/src/
├── routes/tasks/
│   ├── +page.svelte           # Main tasks dashboard
│   └── +page.js               # Data loader (fetches tasks by section)
├── lib/
│   ├── components/Tasks/
│   │   ├── SideBar.svelte      # Sidebar navigation for sections
│   │   ├── Task.svelte         # Individual task item
│   │   ├── TaskHeader.svelte   # Search and title header
│   │   ├── TaskModel.svelte    # Create/Edit modal
│   │   └── TaskSection.svelte  # Task list container
```
---

### Page Loading

The inventory page uses SvelteKit's `+page.js` loader to fetch data on navigation:

```javascript
const apiUrl = 'http://localhost:3030/tasks/'

export const load = async ({fetch}) => {
    const tasksItems = await fetch(apiUrl);
    const tasksItemsJson = await tasksItems.json()
    return {tasks: tasksItemsJson}
}
```

---

### Form Validation

The create/edit modal validates the following fields to ensure data integrity before submission:

| Field              | Validation Rules                                           |
|--------------------|------------------------------------------------------------|
| `name`             | Required, non-empty string. Database rejects null values.  |
| `sectionId`        | Required; ensures task is mapped to a specific garden area.|
| `isComplete`       | Boolean; defaults to `false` upon creation.                |

Validation runs on form submit, and the backend reinforces these rules using Mongoose schema constraints.

---

## Data Flow Diagrams

### Creating a Task
```
┌─────────┐           ┌─────────┐           ┌─────────┐         ┌─────────┐
│ Client  │           │   API   │           │ Server  │           │ MongoDB │
│   UI    │           │ Gateway │           │ Service │           │         │
└────┬────┘           └────┬────┘           └────┬────┘          └────┬────┘
     │                     │                     │                     │
     │ POST /tasks         │                     │                     │
     │ {name, sectionId,   │                     │                     │
     │  description}       │                     │                     │
     │───────────────────▶│                     │                     │
     │                     │ Forward to          │                     │
     │                     │ :3031/tasks         │                     │
     │                     │───────────────────▶│                     │
     │                     │                     │ createTask(data)    │
     │                     │                     │───────────────────▶│
     │                     │                     │ Task.create()       │
     │                     │                     │                     │
     │                     │                     │◀───────────────────│
     │                     │                     │ New task document   │
     │                     │◀───────────────────│                     │
     │                     │ 201 Created         │                     │
     │◀───────────────────│                     │                     │
     │ {_id, name, ...}    │                     │                     │
     │                     │                     │                     │
```

### Updating a Task
```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │   API   │           │ Server  │           │ MongoDB │
│   UI    │           │ Gateway │           │ Service │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ PUT /tasks/:id      │                     │                     │
     │ {name: "New Name"}  │                     │                     │
     │───────────────────▶│                     │                     │
     │                     │ Forward to          │                     │
     │                     │ :3031/tasks/:id     │                     │
     │                     │───────────────────▶│                     │
     │                     │                     │ updateTask(id, data)│
     │                     │                     │───────────────────▶│
     │                     │                     │ findByIdAndUpdate() │
     │                     │                     │ with runValidators  │
     │                     │                     │                     │
     │                     │                     │◀───────────────────│
     │                     │                     │ Updated document    │
     │                     │◀───────────────────│                     │
     │                     │ 200 OK              │                     │
     │◀───────────────────│                     │                     │
     │ Updated task        │                     │                     │
     │                     │                     │                     │
```
### Deleting a Task
```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │   API   │           │ Server  │           │ MongoDB │
│   UI    │           │ Gateway │           │ Service │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ DELETE              │                     │                     │
     │ /tasks/:id          │                     │                     │
     │───────────────────▶│                     │                     │
     │                     │ Forward to          │                     │
     │                     │ :3031/tasks/:id     │                     │
     │                     │───────────────────▶│                     │
     │                     │                     │ deleteTask(id)      │
     │                     │                     │───────────────────▶│
     │                     │                     │ findByIdAndDelete() │
     │                     │                     │                     │
     │                     │                     │◀───────────────────│
     │                     │                     │ Deletion confirmed  │
     │                     │◀───────────────────│                     │
     │                     │ 204 No Content      │                     │
     │◀───────────────────│                     │                     │
     │ (Empty response)    │                     │                     │
     │                     │                     │                     │
```
---

## Security & Validation

1. **CORS**: The Gateway restricts access to the `CLIENT_URL` (default: `localhost:5173`) and allows credentials for authenticated sessions.

2. **Data Integrity**:
    - **`name`** is a required field; the database will reject any task without one.
    - **`sectionId`** is required to ensure every task is correctly mapped to a specific area of the garden.

3. **Schema Validation**: 
    - The `updateTask` service uses `{ runValidators: true }` to ensure that any updates (like renaming a task) still follow the Mongoose schema rules.

4. **Status Toggle**: 
    - The `toggleCheckBox` function uses an atomic `save()` after flipping the boolean
