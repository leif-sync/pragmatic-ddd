---
name: bootstrap
description: "[Pragmatic DDD Architecture] Interactive guide for scaffolding and bootstrapping a new project or module from scratch. Use this skill when the user asks to start a new project or add a massive new feature. It instructs the agent to run an assessment wizard, define the PRD, evaluate serverless tech, and set up the foundation."
---

# Project Bootstrap & Scaffolding Wizard

## 1. The Bootstrapping Process

When a user requests to create a new project or a complex new feature, you MUST NOT start writing code immediately. Instead, you MUST act as a technical product manager. Your goal is to build **purely serverless applications** using the **Pragmatic DDD** architecture.

You MUST ask the user a series of assessment questions (detailed below). The outputs of this assessment MUST be explicitly separated into two files:
1. **`AGENTS.md`**: Global, high-level technical configurations (e.g., "Pragmatic DDD, Neon DB, Better Auth"). This file is automatically attached to all agent prompts, so it MUST be kept concise.
2. **Product Requirements Document (PRD)**: A separate markdown file (e.g., `docs/PRD.md`) containing the detailed business features and user stories. The agent MUST consult this file whenever it needs feature context, but it MUST NOT be placed inside `AGENTS.md`.

## 2. Assessment Questions

You MUST ask the user the following questions to shape the architecture:

1. **Persistence:** Is this a simple static page, or a full application requiring a database?
   - *Constraint:* If a DB is required, you MUST use Docker Compose for local development. For production, you MUST suggest **Neon** (since it scales to zero and auto-awakes) and discourage Supabase (which doesn't auto-awake when paused).
2. **Authentication:** Does the app need authentication? 
   - *Constraint:* If yes, ask which type. Suggest Email OTP, Magic Links, or OAuth2. You MUST strongly discourage traditional email/password for security reasons. If email auth is chosen, you MUST use `Better Auth` + `Resend` + `@react-email/components` (required to compile React components for emails).
3. **Internationalization (i18n):** Will the app support multiple languages?
   - *Constraint:* If yes, you MUST use the custom library-free i18n implementation and explicitly separate public (`app/[locale]`) and private (`app/dashboard`) route handling.
4. **Project Scope:** Is this a serious production-ready project from day one, or a quick draft/boceto?
   - *Constraint:* If it is a draft, you MUST NOT install or configure testing libraries (`vitest`, `testcontainers`) initially to save time. Remind the user that the Pragmatic DDD architecture allows adding tests easily later. If it is serious, testing libraries MUST be installed from the start.

## 3. Product Requirements Document (PRD)

After the initial technical boundaries are set, you MUST ask the user for the core idea of the application. 
- You SHOULD suggest features to help flesh out the idea.
- Once the idea is clear, you MUST generate a Product Requirements Document (PRD) and save it as a separate markdown file (e.g., `docs/PRD.md`).
- The PRD MUST ONLY contain business features and user stories. It MUST NOT mention specific technologies or libraries.

## 4. Serverless Tech Evaluation

Once the PRD is defined, you MUST evaluate what technologies to add based on our strict serverless philosophy:
- **Deployment:** Default to Vercel or Google Cloud Run.
- **Async & Background Jobs:** If the app needs to process information that is slow, takes time, or does not need an immediate synchronous response, you MUST use **Google Cloud Run Jobs** triggered via **Inngest**.
- **Long-running/Scheduled Workflows:** If the app requires logic that spans across days or has specific scheduling per client, you MUST use **Inngest**.

---

## 5. Infrastructure Setup

*(Apply these rules once the coding phase begins and Docker is required)*

### Docker Compose Setup (PostgreSQL 18)

This architecture relies heavily on Testcontainers for integration testing. For local development, Docker Compose is used.

**Crucial Constraints for Postgres 18:**
1. You MUST use **PostgreSQL 18** (e.g., `postgres:18.1-alpine`).
2. Default mounting behavior requires you to map the volume strictly to `/var/lib/postgresql` to avoid permission errors.
3. You MUST provide fallback defaults for environment variables (e.g. `${POSTGRES_USER:-postgres}`) so the containers can run from a DevContainer without a pre-existing `.env` file.
4. Healthchecks are generally unnecessary for standard local development without strict orchestration loops like Swarm. You MAY omit them to keep the file simple.

### DevContainers (Optional)

The use of DevContainers is **optional**. However, if requested by the user or utilized to maintain a uniform environment, the `.devcontainer/devcontainer.json` MUST be wired to use the `docker-compose` or `compose.dev.yml` file as its backend.

**Critical Requirements for the DevContainer:**
1. **Docker-outside-of-Docker (DoOD)**: Because our integration tests use `@testcontainers/postgresql`, the DevContainer environment MUST have Docker socket access. Always include `"ghcr.io/devcontainers/features/docker-outside-of-docker:1"` with `"enableNonRootDocker": true` in the features list.
2. **Mounts and Users**: The `remoteUser` SHOULD typically be `node` and the environment MUST map `/workspace`.

### References

You MUST refer to the following foundation files to see a complete working setup of DevContainers and Docker Compose orchestration:
- `[dockerfile](references/dockerfile)`: Multistage setup.
- `[compose.dev.yml](references/compose.dev.yml)`: Local compose orchestration (showing volume mounts and fallback env variables).
- `[.devcontainer.json](references/devcontainer.json)`: The VSCode DevContainer configuration with DoOD enabled.
