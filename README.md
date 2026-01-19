# Juicy Forest Wiki

Welcome to the official documentation for **Juicy Forest** — a collaborative garden management platform.

## About

Juicy Forest helps teams manage their community gardens with features for inventory tracking, task management, interactive maps, and real-time chat.

## Features

- **[Settings](features/admin-dashboard.md)** — User profile and garden administration
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

- **Hotfix branches:** when a hotfix has to take place, a branch for this is made titled "hotfix/hotfix-name", in which the issue is resolved. Hotfixes will only take place and be merged on the development branch, if the hotfix is essential to the development of a feature, it should be pulled into that feature after it has been fixed in the development branch. by fixing all issues of the versions of the development branch that are being merged into release, hotfixes won't have to take place on the release or main branches. In exceptional cases where something must be hotfixed in main, this is still possible as seen at the end of the diagram, though this is not preferred.

<img width="959" height="484" alt="image" src="https://github.com/user-attachments/assets/56a495ce-9e12-4503-bd60-a5e90a7fbd02" />

<details style="padding: 12px 0; margin-top: 16px;">
<summary style="font-weight: 600; cursor: pointer; user-select: none; color: #777; display: flex; align-items: center; gap: 8px;">
<span style="font-size: 16px;">→</span> Why This Branching Model?
</summary>
<div style="margin-top: 12px; color: #888; line-height: 1.6; font-size: 15px;">

The project uses **Gitflow** — a structured branching model with develop, main, feature, and hotfix branches. While simpler strategies like GitHub Flow work for rapid deployment and Trunk-Based Development favors speed, Gitflow was chosen for its ability to balance **stability and organization**. It provides clear separation between active development and production releases, ensuring production-ready code on main while features are safely isolated. This is ideal for a growing team with defined roles and complex feature work that needs proper staging before deployment.

</div>
</details>

## Pipelines
The two images below show sequence diagrams for the frontend and backend GitHub actions pipelines. The main things happening in them are checking out the code, setting up node, installing dependencies, running lint, running the tests with a coverage report, getting an advanced code analysis from sonar cloud running npm run build and sending out a discord notification if something failed or passing if everything was successful. for the backend, an additional set of actions take place to test if the docker builds correctly and if so to push the container images to GitHub Container Registry as well.

**Pipeline frontend:**
<img width="" height="" alt="image" src="https://github.com/user-attachments/assets/1c337d8f-82e5-45d9-abda-d2287640cd07" />

**Pipeline backend:**
<img width="" height="" alt="image" src="https://github.com/user-attachments/assets/0e65ec22-47cf-4307-93cc-a9b7e174ac87" />

