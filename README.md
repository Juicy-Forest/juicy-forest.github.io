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

The project uses **Gitflow** - a structured branching model with develop, main, feature, and hotfix branches. While simpler strategies like GitHub Flow work for rapid deployment and Trunk-Based Development favors speed, Gitflow was chosen for its ability to balance **stability and organization**. It provides clear separation between active development and production releases, ensuring production-ready code on main while features are safely isolated. This is ideal for a growing team with defined roles and complex feature work that needs proper staging before deployment.

**Pros**
-> Overall a very clear and simple structure since main, development, features and hotfixes all happen separately. Therefore makes it easy to work with new/large teams.
-> The hotfixes allow production emergencies to be cleanly separated which reduces the disk of releasing broken code
-> Very clear separation of responsibilities since each user can work on their feature on their branch.
-> Features don't interfere with each other until merged due to their own branch (unlike a simple main/dev model)

**Cons**
-> Quite slow because there are many branches and a lot of merging happening. 
-> Can result in a lot of merge conflicts, especially with larger features that get added closer to release time.
-> Hotfixes have to be merged into main/development which can be easy to forget and can lead to bugs reappearing later.

**Alternative we could have used:**
**Github flow**

**Pros:**
-> Very simple since features directly go into main, no development, release or hotfix branches so with fewer branches, you have less mental overhead.
-> Feature branches (PRs) are generally much smaller. Results in faster feedback because tests fail quickly, and less code = less code to fix if issues do appear.
-> Much fewer merge conflicts because branches don't exist for that long (merged continuously) and so there is less divergence from the main branch

**Cons:**
-> Required discipline because it removes the development branch. A feature merged to main can break production immediately which can cause downtime to customer.
-> PRs must be small because large PRs are harder to review, take longer to merge and increase the risk to main branch in case of issues (bascially more, PRs but each is lower risk).
-> There is no explicit release phase because every merge is basically a release.

**Our choice:**
We chose the gitflow branching model because it's a lot safer, easier to manage for a team and we needed more explicit release phases (CIN). Gitflow also provides a very unambiguous answer to where to develop features and where integrations happen. Having an explicit release branch also allows for final testing and bug fixing before merging to main which was helpful because last minute fixes are sometimes unavoidable.

</div>
</details>

## Pipelines
The two images below show sequence diagrams for the frontend and backend GitHub actions pipelines. The main things happening in them are checking out the code, setting up node, installing dependencies, running lint, running the tests with a coverage report, getting an advanced code analysis from sonar cloud running npm run build and sending out a discord notification if something failed or passing if everything was successful. for the backend, an additional set of actions take place to test if the docker builds correctly and if so to push the container images to GitHub Container Registry as well.

**Pipeline frontend:**
<img width="" height="" alt="image" src="https://github.com/user-attachments/assets/148a8b9d-845b-465d-8c55-f13f59d8bc4c" />

**Pipeline backend:**
<img width="" height="" alt="image" src="https://github.com/user-attachments/assets/0e65ec22-47cf-4307-93cc-a9b7e174ac87" />

