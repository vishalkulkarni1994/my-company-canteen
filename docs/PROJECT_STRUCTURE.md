# canteen-connect: Project Structure

This document explains how the repository is laid out and why. For the system design see [DESIGN.md](../DESIGN.md); for the build phases see [PLAN.md](../PLAN.md).

## Overview

canteen-connect is an npm-workspaces monorepo. Every deployable piece and every shared library lives in one repository, so the API, the workers and the web app share one set of event types and one toolchain.

```text
canteen-connect/
├── apps/
│   └── web/                 React + TypeScript app
├── services/
│   ├── api/                 GraphQL API
│   └── worker/              Outbox relay, triage worker, daily summary job
├── packages/
│   └── events/              Shared event types and validation
├── infra/                   Bicep templates (Azure)
├── docs/                    Diagrams and documentation
├── .github/workflows/       CI pipeline
├── docker-compose.yml       Local MongoDB and RabbitMQ
├── package.json             Workspace root and shared scripts
├── tsconfig.base.json       Shared TypeScript settings
├── .env.example             Template for local environment variables
├── DESIGN.md                System design
└── PLAN.md                  Phase-wise delivery plan
```

## Folders

### `apps/web`
The React + TypeScript front end, with role-based views for employees and canteen staff and the Feedback Hub widget. It currently holds only a placeholder `package.json`; the React app (Vite) is added in Phase 1.

### `services/api` (`@canteen/api`)
The Node.js + TypeScript GraphQL API: queries, mutations and subscriptions over WebSocket. It owns authentication, role checks, ordering, menu and stock management, and writes events to the `outbox` collection in the same transaction as the change they describe. Entry point: `src/index.ts`.

### `services/worker` (`@canteen/worker`)
Background processes that run separately from the API:

- the **outbox relay**, which publishes saved events to RabbitMQ
- the **feedback triage worker**, which calls the LLM and validates its output
- the **daily summary job**, which runs on a schedule

Entry point: `src/index.ts`. These are split into separate entry points as they are built.

### `packages/events` (`@canteen/events`)
The event types and validation shared by the API and the workers (for example `OrderPlaced` and `FeedbackSubmitted`). Keeping them in one package means a producer and a consumer cannot disagree about an event's shape.

### `infra`
Bicep templates for Azure Container Apps, Container Registry and Key Vault. Empty until Phase 6.

### `docs`
Diagrams and supporting documentation, including `architecture.svg`, which `DESIGN.md` links to.

### `.github/workflows`
`ci.yml` runs on every pull request and on pushes to `main`: install, type check, test. Image builds, schema checks and deployment are added in Phase 6.

## Root files

| File | Purpose |
|---|---|
| `package.json` | Declares the workspaces (`apps/*`, `services/*`, `packages/*`) and the root scripts |
| `tsconfig.base.json` | Strict TypeScript settings that every package extends |
| `docker-compose.yml` | MongoDB 7 as a replica set (`rs0`, needed for transactions) and RabbitMQ with the management UI |
| `.env.example` | Names of the environment variables the services read; copy to `.env` and fill in |
| `.gitignore` | Excludes `node_modules`, `dist`, `.env` and logs |

## Package layout

Each package under `services/` and `packages/` has the same shape:

```text
<package>/
├── package.json     name, scripts (build, typecheck)
├── tsconfig.json    extends ../../tsconfig.base.json, compiles src/ to dist/
└── src/
    └── index.ts     entry point
```

Packages are named `@canteen/<name>`. A package that needs another one lists it in `dependencies`, and npm links it from the workspace.

## Common commands

Run from the repository root.

| Command | What it does |
|---|---|
| `npm install` | Installs all workspace dependencies |
| `npm run typecheck` | Type checks every package |
| `npm run build` | Builds every package that defines a build script |
| `npm test` | Runs every package's tests (none defined yet) |
| `npm run infra:up` | Starts MongoDB and RabbitMQ in Docker |
| `npm run infra:down` | Stops them |

## Local services

| Service | Address | Notes |
|---|---|---|
| MongoDB | `mongodb://localhost:27017/?directConnection=true` | Replica set `rs0`; connect MongoDB Compass with this string |
| RabbitMQ | `amqp://canteen:canteen@localhost:5672` | Development credentials only |
| RabbitMQ management UI | http://localhost:15672 | User `canteen`, password `canteen` |

## Adding a new package

1. Create the folder under `apps/`, `services/` or `packages/`.
2. Add a `package.json` named `@canteen/<name>` with `build` and `typecheck` scripts.
3. Add a `tsconfig.json` that extends `../../tsconfig.base.json`.
4. Run `npm install` at the root so npm links the workspace.
