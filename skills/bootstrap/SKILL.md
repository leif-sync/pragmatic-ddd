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
3. **Workspaces & Billing:** Will the app be managed by individual user accounts only, or by Workspaces/Organizations?
   - *Constraint:* If billing or payments are involved, you MUST explicitly ask if the billing is tied to the Workspace or to the individual User. This dictates the UI sidebar layout.
4. **Internationalization (i18n):** Will the app support multiple languages?
   - *Constraint:* If yes, you MUST use the custom library-free i18n implementation and explicitly separate public (`app/[locale]`) and private (`app/dashboard`) route handling.
5. **Project Scope:** Is this a serious production-ready project from day one, or a quick draft/boceto?
   - *Constraint:* If it is a draft, you MUST NOT install or configure testing libraries (`vitest`, `testcontainers`) initially to save time. Remind the user that the Pragmatic DDD architecture allows adding tests easily later. If it is serious, testing libraries MUST be installed from the start.

## 3. Product Requirements Document (PRD)

After the initial technical boundaries are set, you MUST ask the user for the core idea of the application. 
- You SHOULD suggest features to help flesh out the idea.
- Once the idea is clear, you MUST generate a Product Requirements Document (PRD) and save it as a separate markdown file (e.g., `docs/PRD.md`).
- The PRD MUST ONLY contain business features and user stories. It MUST NOT mention specific technologies or libraries.

## 4. Architecture & Bounded Contexts Recognition

Once the PRD is defined, you MUST explicitly identify the main domain entities of the SaaS. These entities will dictate the **Bounded Contexts** (isolated modules) of the application.
- You MUST also recognize core/cross-cutting contexts that are not tied to a specific business entity, such as `auth` (for authentication) or `shared` (for common utilities and cross-module infrastructure).

You MUST evaluate what technologies to add based on our strict serverless philosophy:
- **Deployment:** Default to Vercel or Google Cloud Run.
- **Async & Background Jobs:** If the app needs to process information that is slow, takes time, or does not need an immediate synchronous response, you MUST use **Google Cloud Run Jobs** triggered via **Inngest**.
- **Long-running/Scheduled Workflows:** If the app requires logic that spans across days or has specific scheduling per client, you MUST use **Inngest**.

## 5. UI Libraries & Component Strategies

This architecture leverages the `shadcn/ui` CLI to integrate components like **shadcn/ui** and **Magic UI**. You MUST apply the following UI construction rules:

### Preferential Hierarchy
When building the UI, you MUST follow this hierarchy in order of preference:
1. **Building Blocks**: High-level page structures.
2. **Individual Components**: Primitives like buttons, dialogs, etc.
3. **Templates**: You MUST NOT use templates unless explicitly requested by the user.

### Tooling and MCP Initialization
You MUST check if the `shadcn` skill and the respective MCP servers (`shadcnui` and `magicui`) are installed.
- **To install the shadcn skill**: run `npx skills add https://github.com/shadcn/ui --skill shadcn`
- **To install the shadcn MCP**: run `npx shadcn@latest mcp init --client <cliente>` (where `<cliente>` is `claude`, `cursor`, `vscode`, or `opencode` depending on the user's IDE). If the IDE is not listed, consult the shadcn documentation.
- **To install the Magic UI MCP**: update the MCP configuration file (generated by shadcn or currently active) and append the following server:
  ```json
  {
    "mcpServers": {
      "magicuidesign-mcp": {
        "command": "npx",
        "args": ["-y", "@magicuidesign/mcp@latest"]
      }
    }
  }
  ```

### Authentication UI
You MUST consult the shadcn MCP to suggest building blocks for the authentication pages.
- **Login Page Default**: `npx shadcn@latest add login-04`. You MUST verify no existing login page conflicts before running this.
- **Signup Page Default**: `npx shadcn@latest add signup-04`.
- **Adaptation**: After installing these blocks, you MUST adapt them to the project's styling rules and ensure only the active authentication methods (e.g., specific social logins, magic links) are rendered, removing passwords if they are disabled.

### Dashboard Sidebar Layout
You MUST avoid placing the main navigation in a top bar. You MUST prefer a traditional navigation sidebar on the left using shadcn building blocks.
- **Workspaces & User Profiles**: If the app supports workspaces, you MUST default to `npx shadcn@latest add sidebar-07`, which allows quick switching between profiles and workspaces. Place billing/payments settings accurately on the user section or workspace section based on the assessment answer.
- **User Profiles Only**: If the app DOES NOT use workspaces, you MUST still use `sidebar-07`. However, you MUST manually remove the Workspaces segment from the component and move the User Profile selector to the top of the sidebar (replacing the dashboard/workspace switcher).

### Micro-interactions & Aesthetics
The application SHOULD feel highly polished using micro-interactions and animations.
- Every action that takes time (e.g., submitting a form, executing a Server Action) MUST be visually represented using components from the aforementioned libraries (e.g., loading spinners on buttons, disabled states, loading text).
- Use the MCP to find specialized interactive components. If a suitable component cannot be found, you MUST create or modify the existing ones.
- **Custom Animations**: If you need to create custom interactive components or complex animations from scratch, you MUST use **Framer Motion** (now known as **motion.dev**).
- You SHOULD suggest the component library websites to the user so they can visually browse and choose blocks/components they like.

## 6. The Pre-Flight Proposal

Before creating any files or writing code, you MUST present a comprehensive report (the "Pre-Flight Proposal") to the user. This proposal acts as a final confirmation step and MUST include:
- **What** is going to be built (Summary of the PRD and chosen domain contexts).
- **With what** technologies (Summarizing the decisions from the assessment wizard).
- **How** it will be structured, specifically outlining the proposed folder structure (including the planned Bounded Contexts in `src/` and the routing structure in `src/app/`).

You MUST ask the user if they agree with this proposal or if they want to make any changes before you begin generating the scaffolding.

---

## 7. Project Execution & Scaffolding Flow

If the Pre-Flight Proposal is approved, you MUST start scaffolding the codebase following this exact sequential order:

### Phase 1: Next.js Initialization
1. Determine the installation directory.
   - If the current folder is mostly empty (e.g., only containing `skills` or a README), you MUST initialize the application in the current directory using `.` to avoid deep nesting.
   - If the target folder is occupied, prompt the user for a new folder name and apply it to the CLI.
2. Run the Next.js scaffold command:
   ```bash
   npx create-next-app@latest . --ts --tailwind --biome --app --src-dir --import-alias "@/*" --yes
   ```

### Phase 2: Shadcn & Dependencies
Once Next.js is installed, you MUST initialize `shadcn` before anything else.
1. Run `npx shadcn@latest init` and configure it according to Next.js strict standards.
2. Proceed to install the `shadcn` skill and MCP servers as described in Section 5 (Tooling and MCP Initialization).
3. Then, begin adding the building blocks and components (e.g., login, sidebar) requested in the UX phase.

---

## 8. Infrastructure Setup

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
