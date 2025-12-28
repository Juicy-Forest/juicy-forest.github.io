# Juicy Forest Wiki

Welcome to the official documentation for **Juicy Forest** — a collaborative garden management platform.

## About

Juicy Forest helps teams manage their community gardens with features for inventory tracking, task management, interactive maps, and real-time chat.

## Features

- **[Chat](features/chat.md)** — Real-time messaging with channels and typing indicators
- **[Inventory](features/inventory.md)** — Track garden supplies and equipment
- **[Tasks](features/tasks.md)** — Manage and assign garden tasks
- **[Map](features/map.md)** — Interactive garden layout visualization
- **[Garden](features/garden.md)** — Garden and section management

## Architecture

The platform uses a microservices architecture:

| Service   | Port | Description                          |
|-----------|------|--------------------------------------|
| Gateway   | 3030 | API Gateway / Reverse Proxy          |
| Server    | 3031 | Main backend (auth, inventory, etc.) |
| Chat      | 3033 | Real-time chat microservice          |
| Client    | 5173 | SvelteKit frontend                   |

## Repositories

- **[backend](https://github.com/juicy-forest/backend)** — Backend services (Gateway, Server, Chat)
- **[client](https://github.com/juicy-forest/client)** — SvelteKit frontend application

## Tech Stack

**Frontend:**
- SvelteKit 2.x with Svelte 5
- TailwindCSS
- TypeScript

**Backend:**
- Node.js + Express
- MongoDB + Mongoose
- WebSocket (ws library)
- JWT Authentication

## Branching Model 

- **Main branch:** for the major releases, only production-ready functional code goes here merged from the release branch.

- **Release branch:** for final testing and staging before merges are made to the main, code here comes from merges from the development branch and should be production-ready.

- **Development branch:** for active development of features, the integration point for all feature work. Code here should be functional but can still contain small errors, these must however be fixed before release branch merges.

- **Feature branches:** branch off of development and are used to work on individual features, they should be titled "feature/feature-name" and merged into only the development branch when completed.

- **Hotfix branches:** when a hotfix has to take place, a branch for this is made titled "hotfix/hotfix-name", in which the issue is resolved. Hotfixes will only take place and be merged on the development branch, if the hotfix is essential to the development of a feature, it should be pulled into that feature after it has been fixed in the development branch. by fixing all issues of the versions of the development branch that are being merged into release, hotfixes won't have to take place on the release or main branches.

<img width="1230" height="500" alt="image" src="https://github.com/user-attachments/assets/6d61d401-98ec-41f8-9235-312686cf1d24" />

---
> 📝 This wiki is a work in progress. Some sections may be incomplete.

