# Map Feature

This document provides a comprehensive overview of the Map feature in Juicy Forest, covering both the backend and frontend implementations for garden visualization, section management, and interactive grid editing.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Backend](#backend)
   - [Service Structure](#service-structure)
   - [Database Models](#database-models)
   - [REST API Endpoints](#rest-api-endpoints)
   - [Services](#services)
3. [Frontend](#frontend)
   - [Component Structure](#component-structure)
   - [State Management](#state-management)
   - [Grid System](#grid-system)
   - [Edit Mode](#edit-mode)
4. [Data Flow Diagrams](#data-flow-diagrams)
5. [API Request/Response Formats](#api-requestresponse-formats)

---

## Architecture Overview

The Map feature follows a **microservice architecture** with REST API communication:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   SvelteKit     │────▶│   API Gateway   │────▶│  Server Service │
│   Frontend      │     │   (Port 3030)   │     │  (Port 3031)    │
│   (Port 5173)   │     └─────────────────┘     └────────┬────────┘
│                 │                                      │
│                 │◀────── REST API (http://localhost:3030) ──────▶│
└─────────────────┘                                      │
                                                         ▼
                                               ┌─────────────────┐
                                               │    MongoDB      │
                                               │  (juicy-forest) │
                                               └─────────────────┘
```

- **Frontend**: SvelteKit application with reactive state management using Svelte 5 runes
- **API Gateway**: Express.js proxy routing REST requests to the server service
- **Server Service**: Express.js service handling garden and section operations
- **Database**: MongoDB for persistent storage of gardens, sections, and grid data

---

## Backend

### Service Structure

The map and section functionality is part of the main server service (`backend/server/`):

```
backend/server/
├── index.js                    # Entry point, Express server
├── routes.js                    # Route definitions
├── controllers/
│   ├── gardenController.js      # REST endpoints for gardens
│   └── sectionController.js     # REST endpoints for sections
├── models/
│   ├── Garden.js                # Mongoose garden schema
│   └── Section.js               # Mongoose section schema
├── services/
│   ├── gardenService.js         # Garden business logic
│   └── sectionService.js        # Section business logic
└── middlewares/
    └── auth.js                  # JWT authentication middleware
```

#### Entry Point (`index.js`)

The server service runs on port `3031` and initializes:
- Express.js server for REST endpoints
- MongoDB connection via Mongoose
- CORS configuration for client access
- Authentication middleware for protected routes

---

### Database Models

#### Garden Model

Represents a garden with an interactive grid layout.

| Field         | Type       | Description                                    |
|---------------|------------|------------------------------------------------|
| `name`        | String     | Garden name (required, unique)                 |
| `description` | String     | Optional garden description                    |
| `location`    | Object     | Location data (lat, lng, address)              |
| `joinCode`    | String     | Unique code for joining the garden (required)  |
| `owner`       | ObjectId   | Reference to User (owner)                      |
| `members`     | [ObjectId] | Array of User references (members)             |
| `maxMembers`  | Number     | Maximum members allowed (default: 10)          |
| `grid`        | [GridTile] | Array of 400 grid tiles (20x20 grid)          |
| `createdAt`   | Date       | Auto-generated timestamp                       |

**GridTile Schema:**
| Field     | Type       | Description                          |
|-----------|------------|--------------------------------------|
| `index`   | Number     | Tile position (0-399)                |
| `section` | ObjectId   | Reference to Section (nullable)      |
| `plant`   | String     | Plant type enum (Plant, Tree, Bush, Flower, Greenhouse, Pathway, Pond, Fence, '') |

**Indexes:**
- `{ name: 1 }` - unique index with case-insensitive collation
- `{ joinCode: 1 }` - unique index for join codes

**Grid Initialization:**
When a garden is created, a default 20x20 grid (400 tiles) is initialized:
```javascript
const initGrid = []
for (let i = 0; i < 400; i++) {
    const tile = {index: i, section: null, plant: ''}
    initGrid.push(tile)
}
```

#### Section Model

Represents a section within a garden that can be assigned to grid tiles.

| Field          | Type       | Description                              |
|----------------|------------|------------------------------------------|
| `sectionName`  | String     | Section name (required)                  |
| `temperature`  | String     | Temperature reading (default: "")         |
| `humidityLevel`| Number     | Humidity percentage (default: 0)         |
| `soilMoisture` | String     | Soil moisture level (default: "")         |
| `lastWatered`  | String     | Last watering date (default: "")         |
| `issues`       | [Issue]    | Array of issue objects                   |
| `plants`       | [String]   | Array of plant names (default: [])        |
| `color`        | String     | Color identifier for visualization (default: "#FFFFFF") |
| `assignedTo`   | ObjectId   | Reference to User (optional)             |
| `garden`       | ObjectId   | Reference to Garden (required)           |

**Issue Schema:**
| Field         | Type     | Description                    |
|---------------|----------|--------------------------------|
| `type`        | String   | Issue type (required)          |
| `severity`    | String   | Severity level                 |
| `description` | String   | Issue description              |

**Indexes:**
- `{ garden: 1 }` - for querying sections by garden

---

### REST API Endpoints

All endpoints are proxied through the API Gateway (`/section` → Server Service).


#### Section Endpoints

| Method | Endpoint                    | Description              | Request Body                    |
|--------|-----------------------------|--------------------------|---------------------------------|
| GET    | `/section/:gardenId`        | Get sections by garden   | -                               |
| GET    | `/section/:sectionId`       | Get section by ID        | -                               |
| POST   | `/section/:gardenId`         | Create a new section     | `{ sectionName, color, plants[], ... }` |
| PUT    | `/section/:sectionId`       | Update section           | `{ sectionName?, color?, plants?, ... }` |
| DELETE | `/section/:sectionId`       | Delete section           | -                               |

**Response format (Section):**
```json
{
  "_id": "64def456...",
  "sectionName": "North Section",
  "temperature": "22°C",
  "humidityLevel": 65,
  "soilMoisture": "Moist",
  "lastWatered": "2024-01-15",
  "issues": [],
  "plants": ["Tomato", "Lettuce"],
  "color": "bg-green-300",
  "assignedTo": { "_id": "...", "username": "user1" },
  "garden": { "_id": "...", "name": "My Garden" }
}
```

---

### Services

#### Garden Service (`gardenService.js`)

| Function                          | Description                                    |
|-----------------------------------|------------------------------------------------|
| `createGarden(ownerId, name, description, location, grid)` | Creates a new garden with auto-generated join code |
| `getAllGardens()`                 | Retrieves all gardens with populated owner/members |
| `getGardensByUserId(userId)`      | Gets gardens where user is a member            |
| `getGardenById(id)`               | Gets single garden by ID                       |
| `joinGarden(gardenId, userId)`    | Adds user to garden members                    |
| `joinGardenByCode(joinCode, userId)` | Joins garden using join code               |
| `leaveGarden(gardenId, userId)`  | Removes user from garden (owner cannot leave)  |
| `updateGarden(id, data, userId)` | Updates garden (owner only, validates grid)   |
| `deleteGarden(id, userId)`       | Deletes garden (owner only)                   |
| `removeMember(gardenId, memberId, userId)` | Removes member from garden (owner only) |

**Grid Update Validation:**
- Only garden owner can update the grid
- Grid must be an array of GridTile objects
- Each tile must have `index`, `section` (nullable), and `plant` fields

#### Section Service (`sectionService.js`)

| Function                    | Description                           |
|-----------------------------|---------------------------------------|
| `getSectionsByGarden(gardenId)` | Retrieves all sections for a garden (populated) |
| `getSectionById(id)`        | Gets single section by ID             |
| `createSection(data)`       | Creates a new section                 |
| `updateSection(id, data)`   | Updates section properties            |
| `deleteSection(id)`         | Deletes a section                     |

---

## Frontend

### Component Structure

The map UI is built with SvelteKit and organized as follows:

```
client/src/
├── routes/
│   ├── map/
│   │   └── +page.svelte           # Main map page
│   └── api/
│       ├── garden/
│       │   └── [id]/
│       │       └── +server.ts     # Garden update API route
│       └── section/
│           ├── +server.ts         # Section create API route
│           └── [sectionId]/
│               └── +server.ts     # Section delete API route
├── lib/
│   ├── components/Map/
│   │   ├── MapSidebar.svelte      # Sidebar with sections and plants
│   │   ├── CreateNewSection.svelte # Section creation form
│   │   └── SidebarSections.svelte # Section list display
│   ├── types/
│   │   ├── garden.ts              # Garden and GridBoxType types
│   │   └── section.ts             # SectionInfo and SectionData types
│   └── utils/
│       └── grid.ts                # Grid styling utilities
```

### State Management

The map page uses Svelte 5 runes for reactive state management:

#### Main State Variables (`+page.svelte`)

| Variable            | Type              | Description                          |
|---------------------|-------------------|--------------------------------------|
| `selectedSectionId` | `string`          | Currently selected section ID         |
| `isEditMode`       | `boolean`         | Whether edit mode is active           |
| `localSectionData` | `SectionData`     | Local copy of section data            |
| `grid`              | `GridBoxType[]`   | Current saved grid state              |
| `editingGrid`       | `GridBoxType[]`   | Grid state during editing             |
| `gridToShow`        | `GridBoxType[]`   | Grid to display (switches between saved/editing) |
| `selectedIcon`      | `IconType \| null`| Currently selected plant type         |
| `currentGarden`     | `Garden`          | Currently viewed garden               |

#### Key Functions

| Function                    | Description                              |
|-----------------------------|------------------------------------------|
| `enterEditMode()`           | Clones grid and enables edit mode        |
| `exitEditMode()`            | Discards changes and exits edit mode     |
| `saveEdit()`                | Saves grid changes to backend            |
| `cancelEdit()`              | Cancels editing and discards changes     |
| `updateCell(i)`             | Updates a single grid cell               |
| `updateSelectSectionId(id)` | Sets the selected section for placement  |
| `updateSelectedIcon(icon)`   | Sets the selected plant type             |
| `handleIconPlacement(grid)` | Places plant icon on grid tile           |

### Grid System

#### Grid Structure

The grid is a 20x20 layout (400 tiles) rendered using CSS Grid:

```svelte
<div class="grid grid-cols-20 grid-rows-20 border-black/30 border">
  {#each gridToShow as gridItem, i (gridItem.index)}
    <button onclick={() => updateCell(i)}>
      <!-- Tile content -->
    </button>
  {/each}
</div>
```

#### Grid Tile Types

Each tile (`GridBoxType`) contains:
- `index`: Position in grid (0-399)
- `section`: Section ID or `null`
- `plant`: Plant type string or empty string

#### Plant Types

Available plant types with emoji icons:

| Type        | Icon | Description        |
|-------------|------|-------------------|
| Plant       | 🌱   | Generic plant     |
| Bush        | 🌿   | Bush              |
| Tree        | 🌾   | Tree              |
| Flower      | 🌸   | Flower            |
| Greenhouse  | 🏡   | Greenhouse        |
| Pathway     | ⬜   | Pathway           |
| Pond        | 🐸   | Pond              |
| Fence       | 🪜   | Fence             |

#### Grid Styling (`grid.ts`)

The `handleReturnGridClasses` function applies visual styling:
- **Section color**: Applied from section's `color` property
- **Issue indicator**: Red background (`bg-rose-800/65`) if section has issues
- **Empty tiles**: No special styling

```typescript
export const handleReturnGridClasses = function (
  gridSectionId: string | null,
  sectionData: SectionData
) {
  if (!gridSectionId) return;
  const section = sectionData?.find((s: SectionInfo) => s._id === gridSectionId);
  if (!section) return "";
  
  let concatClasses = "";
  if (section.issues && section.issues.length > 0) {
    concatClasses = "bg-rose-800/65";
  }
  concatClasses += ` ${section.color}`;
  return concatClasses;
};
```

### Edit Mode

#### Entering Edit Mode

1. User clicks "Edit" button
2. `enterEditMode()` clones current grid to `editingGrid`
3. `gridToShow` switches to `editingGrid`
4. `isEditMode` set to `true`
5. Sidebar shows section creation and plant selection UI

#### Placing Sections

1. User selects a section from sidebar
2. `selectedSectionId` is set
3. User clicks on grid tile
4. `updateCell(i)` toggles section assignment:
   - If tile has section: removes it (`section: null`)
   - If tile has no section: assigns selected section

#### Placing Plants

1. User selects a plant type from sidebar
2. `selectedIcon` is set
3. User clicks on grid tile
4. `updateCell(i)` sets `plant` property to selected type

#### Saving Changes

1. User clicks "Save" button
2. `saveEdit()` compares `editingGrid` with `grid`
3. If changes detected:
   - Creates updated garden object with new grid
   - Sends PUT request to `/api/garden/:id`
   - Updates local `grid` state on success
4. Exits edit mode

#### Canceling Changes

1. User clicks "Cancel" button
2. `cancelEdit()` calls `exitEditMode()`
3. `gridToShow` reverts to saved `grid`
4. `editingGrid` cleared
5. `isEditMode` set to `false`

---

## Data Flow Diagrams

### Loading Map Data

```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │ Gateway │           │ Server  │           │ MongoDB │
│   UI    │           │ (3030)  │           │ (3031)  │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ GET /garden/user    │                     │                     │
     │────────────────────▶│                     │                     │
     │                     │ GET /garden/user    │                     │
     │                     │────────────────────▶│                     │
     │                     │                     │ Garden.find()       │
     │                     │                     │────────────────────▶│
     │                     │                     │◀────────────────────│
     │                     │◀────────────────────│                     │
     │◀────────────────────│                     │                     │
     │                     │                     │                     │
     │ GET /section/:id    │                     │                     │
     │ (for each garden)   │                     │                     │
     │────────────────────▶│                     │                     │
     │                     │ GET /section/:id    │                     │
     │                     │────────────────────▶│                     │
     │                     │                     │ Section.find()      │
     │                     │                     │────────────────────▶│
     │                     │                     │◀────────────────────│
     │                     │◀────────────────────│                     │
     │◀────────────────────│                     │                     │
     │                     │                     │                     │
```

### Updating Grid

```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │ Gateway │           │ Server  │           │ MongoDB │
│   UI    │           │ (3030)  │           │ (3031)  │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ User edits grid      │                     │                     │
     │ (edit mode)          │                     │                     │
     │                     │                     │                     │
     │ PUT /api/garden/:id  │                     │                     │
     │ {updatedGarden}      │                     │                     │
     │────────────────────▶│                     │                     │
     │                     │ PUT /garden/:id      │                     │
     │                     │ {grid: [...]}       │                     │
     │                     │────────────────────▶│                     │
     │                     │                     │ Verify ownership    │
     │                     │                     │ Update grid         │
     │                     │                     │────────────────────▶│
     │                     │                     │                     │
     │                     │                     │◀────────────────────│
     │                     │◀────────────────────│                     │
     │◀────────────────────│                     │                     │
     │ Success response     │                     │                     │
     │                     │                     │                     │
```

### Creating Section

```
┌─────────┐           ┌─────────┐           ┌─────────┐           ┌─────────┐
│ Client  │           │ Gateway │           │ Server  │           │ MongoDB │
│   UI    │           │ (3030)  │           │ (3031)  │           │         │
└────┬────┘           └────┬────┘           └────┬────┘           └────┬────┘
     │                     │                     │                     │
     │ POST /api/section   │                     │                     │
     │ {sectionName,       │                     │                     │
     │  color, plants,     │                     │                     │
     │  gardenId}          │                     │                     │
     │────────────────────▶│                     │                     │
     │                     │ POST /section/:id   │                     │
     │                     │────────────────────▶│                     │
     │                     │                     │ Section.create()    │
     │                     │                     │────────────────────▶│
     │                     │                     │                     │
     │                     │                     │◀────────────────────│
     │                     │◀────────────────────│                     │
     │◀────────────────────│                     │                     │
     │ New section data    │                     │                     │
     │                     │                     │                     │
```

---

## API Request/Response Formats

### Client → Server (via API Gateway)

#### Update Garden Grid

**Request:**
```http
PUT /api/garden/:id
Content-Type: application/json
X-Authorization: <JWT_TOKEN>

{
  "updatedGarden": {
    "_id": "...",
    "name": "My Garden",
    "grid": [
      { "index": 0, "section": null, "plant": "" },
      { "index": 1, "section": "64def456...", "plant": "Plant" },
      ...
    ],
    ...
  }
}
```

**Response:**
```json
{
  "status": 200,
  "message": "Successfully updated gardendata",
  "data": { /* updated garden object */ }
}
```

#### Create Section

**Request:**
```http
POST /api/section
Content-Type: application/json

{
  "sectionName": "North Section",
  "color": "bg-green-300",
  "plants": ["Tomato", "Lettuce"],
  "gardenId": "64abc123..."
}
```

**Response:**
```json
{
  "_id": "64def456...",
  "sectionName": "North Section",
  "color": "bg-green-300",
  "plants": ["Tomato", "Lettuce"],
  "garden": "64abc123...",
  ...
}
```

#### Delete Section

**Request:**
```http
DELETE /api/section/:sectionId
X-Authorization: <JWT_TOKEN>
```

**Response:**
```json
{
  "status": 200,
  "message": "Section removed."
}
```

---

## Security Considerations

1. **Authentication**: All API requests require valid JWT token in `X-Authorization` header or `auth-token` cookie
2. **Authorization**: 
   - Only garden owner can update garden grid
   - Only garden owner can delete garden
   - Section operations are scoped to garden membership
3. **Input Validation**: 
   - Grid tiles must have valid structure (index, section, plant)
   - Section names are required
   - Plant types must match enum values
4. **CORS**: API Gateway restricts origins to configured client URL

---

## Configuration

### Environment Variables

| Variable              | Default                           | Description                    |
|-----------------------|-----------------------------------|--------------------------------|
| `PORT` (Gateway)      | `3030`                            | API Gateway port               |
| `PORT` (Server)       | `3031`                            | Server service port            |
| `SERVER_SERVICE_URL`  | `http://localhost:3031`           | Server service URL             |
| `MONGO_URI`           | `mongodb://localhost:27017/juicy-forest` | MongoDB connection string |
| `CLIENT_URL`          | `http://localhost:5173`           | Allowed CORS origin            |

### Gateway Configuration

The API Gateway (`backend/gateway/index.js`) proxies `/garden` and `/section` routes to the server service:

```javascript
services: {
  server: {
    url: process.env.SERVER_SERVICE_URL || 'http://localhost:3031',
    routes: ['/users', '/inventory', '/garden', '/section', '/tasks']
  }
}
```

---

## Future Improvements

- [ ] Real-time grid updates using WebSocket (similar to chat feature)
- [ ] Grid size customization (currently fixed at 20x20)
- [ ] Drag-and-drop section placement
- [ ] Undo/redo functionality for grid edits
- [ ] Grid export/import functionality
- [ ] Section templates for quick creation
- [ ] Visual indicators for section health status
- [ ] Grid zoom and pan functionality
- [ ] Multi-select for bulk grid operations
- [ ] Grid history/versioning