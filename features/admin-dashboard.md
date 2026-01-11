# Settings / Admin Dashboard Feature

This document provides an overview of the Settings dashboard in Juicy Forest, covering user profile management and garden administration.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Backend](#backend)
   - [API Endpoints](#api-endpoints)
   - [Database Models](#database-models)
3. [Frontend](#frontend)
   - [Settings Tabs](#settings-tabs)
   - [Components](#components)
4. [Data Flows](#data-flows)
5. [Security](#security)

---

## Architecture Overview

The Settings dashboard allows users to manage their profile and gardens through a centralized interface:

```
┌─────────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   SvelteKit         │────▶│   API Gateway    │────▶│   Server        │
│   Frontend          │     │   (Port 3030)    │     │   (Port 3031)   │
│   (Port 5173)       │     └──────────────────┘     └────────┬────────┘
│                     │                                       │
│  Settings:         │                                        │
│  - Profile         │◀───────────────────────────────────────┤
│  - Garden Admin    │    User & Garden Data                  │
└─────────────────────┘                                       │
                                                              ▼
                                                    ┌─────────────────┐
                                                    │    MongoDB      │
                                                    │  (juicy-forest) │
                                                    └─────────────────┘
```

---

## Backend

### API Endpoints

#### User Management

| Method | Endpoint              | Description           |
|--------|----------------------|----------------------|
| GET    | `/users/`            | Get current user      |
| POST   | `/users/changePassword` | Update password     |
| POST   | `/users/changeUsername` | Update username     |
| POST   | `/users/changeEmail`    | Update email        |

#### Garden Management

| Method | Endpoint                   | Description              |
|--------|----------------------------|-----------------------|
| GET    | `/garden/user`             | Get user's gardens    |
| PUT    | `/garden/:id`              | Update garden name    |
| DELETE | `/garden/:id`              | Delete garden         |
| POST   | `/garden/:id/removeMember` | Remove member         |

### Database Models

#### User Model

| Field             | Type       | Description                      |
|-------------------|------------|----------------------------------|
| `username`        | String     | Display name (min 4 chars)       |
| `email`           | String     | Email address (unique)           |
| `hashedPassword`  | String     | Bcrypt-hashed password           |
| `avatarColor`     | String     | Random pastel color (HEX)        |

#### Garden Model

| Field       | Type          | Description                    |
|-------------|---------------|--------------------------------|
| `name`      | String        | Garden name (unique)           |
| `owner`     | ObjectId Ref  | Garden owner (User)            |
| `members`   | ObjectId[] Ref| Garden members (Users)         |
| `joinCode`  | String        | Unique join code               |
| `grid`      | GridTile[]    | 20×20 garden grid (400 tiles)  |

---

## Frontend

### Settings Tabs

#### Profile Tab
Manage user account:
- **Change Username** - Update display name (triggers page reload)
- **Change Email** - Update email address with duplicate checking
- **Change Password** - Update password (minimum 8 characters)

#### Garden Tab
Manage garden (if owner):
- **Update Garden Name** - Rename the garden
- **View Members** - See all garden members and remove non-owners
- **Access Code** - Display and copy join code for sharing
- **Delete Garden** - Permanently delete the garden

Non-owners see a read-only view of the garden.

### Components

```
routes/settings/
├── +page.svelte                    # Main page layout
├── +page.server.ts                 # Form actions
└── settingHelpers.ts               # Form helpers

lib/components/Settings/
├── SettingsContent.svelte          # Tab router
├── ProfileSettings.svelte          # Profile forms
├── GardenSettings.svelte           # Garden forms
├── GardenMembers.svelte            # Member management
├── GardenAccessCode.svelte         # Join code display
├── SettingSection.svelte           # Form container
├── Option.svelte                   # Tab button
└── Logout.svelte                   # Logout button
```

#### Form State
```typescript
{
  username: { value: string; error: string; success: string };
  email: { value: string; error: string; success: string };
  password: { value: string; error: string; success: string };
  gardenName: { value: string; error: string; success: string };
}
```

---

## Data Flows

### Changing Username Flow

```
┌─────────┐           ┌──────────────┐           ┌──────────────┐
│ Browser │           │  SvelteKit   │           │  Server      │
└────┬────┘           └──────┬───────┘           └──────┬───────┘
     │                        │                         │
     │ 1. Enter new username  │                         │
     │ 2. Submit form         │                         │
     │────────────────────────▶                         │
     │ (?/updateUsername)     │                         │
     │                        │ 3. Validate            │
     │                        │ 4. POST /users/changeUsername
     │                        │─────────────────────────▶
     │                        │                         │
     │                        │                         │ 5. Update DB
     │                        │                         │ 6. New JWT
     │                        │◀─────────────────────────│
     │◀────────────────────────│ {accessToken, message}  │
     │ Success response       │                         │
     │ 7. Reload page         │                         │
```

### Updating Garden Name Flow

```
┌─────────┐           ┌──────────────┐           ┌──────────────┐
│ Browser │           │  SvelteKit   │           │  Server      │
└────┬────┘           └──────┬───────┘           └──────┬───────┘
     │                        │                         │
     │ 1. Enter garden name   │                         │
     │ 2. Submit form         │                         │
     │────────────────────────▶                         │
     │ (?/updateGardenName)   │                         │
     │                        │ 3. Validate             │
     │                        │ 4. PUT /garden/:id      │
     │                        │─────────────────────────▶
     │                        │                         │
     │                        │                         │ 5. Verify owner
     │                        │                         │ 6. Update DB
     │                        │◀─────────────────────────│
     │◀────────────────────────│ {message, newGardenName}│
     │ Success response       │                         │
     │ Update local data      │                         │
```

### Removing Garden Member Flow

```
┌─────────┐           ┌──────────────┐           ┌──────────────┐
│ Browser │           │  SvelteKit   │           │  Server      │
└────┬────┘           └──────┬───────┘           └──────┬───────┘
     │                        │                         │
     │ 1. Click "Remove"      │                         │
     │ 2. Submit form         │                         │
     │────────────────────────▶                         │
     │ (?/removeMember)       │                         │
     │ {gardenId, memberId}   │                         │
     │                        │ 3. POST /garden/:id/removeMember
     │                        │─────────────────────────▶
     │                        │                         │
     │                        │                         │ 4. Verify owner
     │                        │                         │ 5. Remove member
     │                        │◀─────────────────────────│
     │◀────────────────────────│ {success: true}         │
     │ Reload page            │                         │
```

---

## Security

### Authentication
- JWT tokens stored in HTTP-only cookies
- All endpoints require valid JWT in `x-authorization` header
- Tokens blacklisted on logout (in-memory, cleared on restart)

### Authorization
| Operation | Requirement |
|-----------|------------|
| Update own profile | Must be authenticated |
| Update garden | Must be garden owner |
| Remove member | Must be garden owner |
| Delete garden | Must be garden owner |

### Validation
- **Passwords**: Minimum 8 characters, bcrypt hashed
- **Emails**: Format validation, case-insensitive duplicates prevented
- **Usernames**: Minimum 4 characters, case-insensitive duplicates prevented
- **Garden Names**: Unique, case-insensitive

---

## API Examples

### Change Username
```json
POST /users/changeUsername
{
  "newUsername": "garden_lover"
}

Response (200):
{
  "accessToken": "eyJhbGc...",
  "message": "Username changed successfully!"
}
```

### Update Garden
```json
PUT /garden/64abc123def456
{
  "name": "New Garden Name"
}

Response (200):
{
  "_id": "64abc123def456",
  "name": "New Garden Name",
  "owner": {...},
  "members": [...],
  "joinCode": "ABC123XYZ"
}
```

### Remove Member
```json
POST /garden/64abc123def456/removeMember
{
  "memberId": "64xyz789abc123"
}

Response (200):
{
  "message": "Member removed successfully!"
}
```

---

*Last Updated: January 2026*

*Want to contribute? See the [Contributing Guide](../guides/contributing.md).*

