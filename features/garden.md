# Garden Feature

This document provides a comprehensive overview of the Garden feature in Juicy Forest, covering backend architecture, database models, API endpoints, frontend structure, and data flow.

---

## Table of Contents

- Architecture Overview
- Backend
  - Microservice Structure
  - Database Models
- REST API Endpoints
- Services
- Gateway Configuration
- Data Samples & JSON Formats
- Frontend
  - Component Structure
  - State Management
- Garden Membership & Permissions
- Section Management
- Data Flow Diagrams
- Security & Validation

---

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

Frontend: Manages garden selection, creation, and section display  
API Gateway: Proxies garden-related requests  
Server Service: Handles garden logic and persistence  
Database: MongoDB collections for gardens and sections

---

## Backend

### Microservice Structure

Garden logic resides in the main Server Service:

backend/server/
├── routes.js
├── controllers/
│ └── gardenController.js
├── models/
│ ├── Garden.js
│ └── GardenSection.js
└── services/
└── gardenService.js

---

## Database Models

### Garden Model

Represents a single garden shared by one or more users.

| Field | Type | Description |
|------|------|-------------|
| name | String | Garden name |
| ownerId | ObjectId | User who created the garden |
| members | ObjectId[] | Users with access |
| createdAt | Date | Creation timestamp |
| updatedAt | Date | Last update timestamp |

Indexes:

- `{ ownerId: 1, name: 1 }` – Prevents duplicate garden names per owner

---

### GardenSection Model

Represents a logical subdivision within a garden.

| Field | Type | Description |
|------|------|-------------|
| name | String | Section name |
| gardenId | ObjectId | Parent garden reference |
| order | Number | Display order |

Indexes:

- `{ gardenId: 1, name: 1 }` – Unique section names per garden

---

## REST API Endpoints

All garden endpoints are accessed through the API Gateway using the `/gardens` prefix.

### Garden Endpoints

| Method | Endpoint | Description |
|------|----------|-------------|
| GET | /gardens | Retrieve all gardens for user |
| POST | /gardens | Create a new garden |
| GET | /gardens/:id | Retrieve garden details |
| PUT | /gardens/:id | Update garden |
| DELETE | /gardens/:id | Delete garden |

### Section Endpoints

| Method | Endpoint | Description |
|------|----------|-------------|
| GET | /gardens/:id/sections | Retrieve sections |
| POST | /gardens/:id/sections | Create section |
| PUT | /sections/:sectionId | Update section |
| DELETE | /sections/:sectionId | Delete section |

---

## Services

### Garden Service (`gardenService.js`)

| Function | Description |
|--------|-------------|
| getGardens(userId) | Fetch gardens for user |
| createGarden(data) | Create garden |
| updateGarden(id, data) | Update garden |
| deleteGarden(id) | Delete garden and children |
| addMember(gardenId, userId) | Add member |

---

## Gateway Configuration

The API Gateway routes garden requests to the Server Service:

js
const services = {
  server: {
    url: process.env.SERVER_SERVICE_URL || 'http://localhost:3031',
    routes: ['/gardens', '/sections']
  }
};

Requests to http://localhost:3030/gardens are forwarded to the Server Service.

## Data Samples & JSON Formats

### 1. GET /gardens

**Response (200 OK):**
```
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
2. POST /gardens
Request Body:
```
json
Code kopieren
{
  "name": "Community Garden"
}
Response (201 Created):

json
Code kopieren
{
  "_id": "65aa999...",
  "name": "Community Garden",
  "ownerId": "658aa123...",
  "members": ["658aa123..."]
}
```
3. POST /gardens/:id/sections
Request Body:
```
{
  "name": "Greenhouse",
  "order": 1
}
Response (201 Created):

{
  "_id": "65bb111...",
  "name": "Greenhouse",
  "gardenId": "65aa999...",
  "order": 1
}
```
Frontend
Component Structure
The garden UI is built with SvelteKit and organized as follows:

```
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
State Management
The gardenStore manages garden state globally.

Reactive State Properties
Property	Description
gardens	Accessible gardens
selectedGardenId	Active garden
sections	Sections for selected garden
isModalOpen	Modal visibility state
modalMode	create / edit / delete

Garden Membership & Permissions
Owner
Full control

Can delete garden

Can manage members

Member
Access to garden features

Cannot delete garden

Permissions are enforced server-side using the authenticated user ID.

Section Management
Sections provide organizational structure for plants, inventory, and tasks. They are optional but recommended for larger gardens.

Data Flow Diagrams
Creating a Garden
```
Client → API Gateway → Server Service → MongoDB
POST /gardens        createGarden()     Garden.create()
```
Security & Validation
All routes require authentication

User must be owner or member of the garden

Garden data is isolated using gardenId

Cascading deletes remove sections and related data

Schema validation is enforced at the database level

