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

---

> 📝 This wiki is a work in progress. Some sections may be incomplete.

