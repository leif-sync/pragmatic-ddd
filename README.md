# Pragmatic DDD Architecture Skills

This repository contains a unified collection of highly cohesive "skills" (rules, context, conventions, and best practices) for AI coding assistants (such as GitHub Copilot, Cursor) to learn and apply when working within a Next.js 15+ project architecture that emphasizes **Pragmatic Domain-Driven Design (DDD)** and **Railway-Oriented Programming (ROP)** patterns.

## 📜 Global Conventions

To ensure consistency and prevent token bloat across individual skill files, the following global rules apply to all skills in this repository:

1. **RFC 2119 Language**: All guidelines strictly use RFC 2119 terminology (`MUST`, `MUST NOT`, `SHOULD`, `SHOULD NOT`, `MAY`). This ensures the AI model understands the exact constraints of the architecture.
2. **Visual Patterns**: Instructions heavily utilize ✅ (Correct Pattern) and ❌ (Anti-Pattern) to demonstrate code examples.
3. **Single Source of Truth**: Each skill focuses *exclusively* on its domain. For example, file naming conventions and folder structures are defined globally in the `architecture` skill, and are not repeated in the `entities` or `use-cases` skills.

## 🏗 The Core Tech Stack

This architecture is pragmatically coupled to a specific, modern tech stack. We do not reinvent the wheel for non-domain concerns. The stack includes:
- **Next.js 16+** (App Router, Server Actions, strict RSC/Client boundaries)
- **neverthrow** (Railway-Oriented Programming for domain flows and strictly branded TypeScript errors)
- **Zod** (Data validation at infrastructure and presentation boundaries)
- **Drizzle ORM & PostgreSQL** (Strict relational models, repository patterns, typed DB transactions)
- **Vitest & Testcontainers** (Fully integrated transactional testing helper `txTest` with ephemeral databases)
- **Better Auth** (Auth flows tightly coupled to our Server Action standard)
- **React Email & Resend** (Transactional email templates)

By incorporating these skills, the AI will learn exactly how to orchestrate a data flow starting from Drizzle DB Repositories, passing securely as immutable Value Objects through Neverthrow `Result` wrappers in Use Cases, and cleanly resolving in Next.js Server Actions with Zod-validated inputs.

## 🚀 Usage & Installation

You can download and integrate these skills directly into your local project environment using the skills CLI.

Add a specific skill by providing the repository URL and the skill name:

```bash
# Example for adding the entire skill set
npx skills add https://github.com/leif-sync/pragmatic-ddd

# Example for adding specific skills
npx skills add https://github.com/leif-sync/pragmatic-ddd --skill architecture
npx skills add https://github.com/leif-sync/pragmatic-ddd --skill server-actions
```

Running this command will fetch the targeted skill folder (including its `SKILL.md` and any associated reference files) and place it directly into your local workspace. Because the architecture relies on shared concepts (such as domain errors and results), we heavily recommend downloading the core skills (`architecture`, `value-objects`, `errors`, `use-cases`) together.

## 📚 Available Skills

- **`architecture`**: General guide on Domain-Driven Design (DDD), Modular Monolith Bounded Contexts, and project structure.
- **`auth`**: Better Auth configurations, session handling workflows, and email template setups.
- **`bootstrap`**: Scaffolding instructions for initializing new Bounded Contexts conforming to the stack.
- **`components`**: Proper boundaries and patterns for Next.js Server vs. Client Components, serialization boundaries for Value Objects.
- **`drizzle-schema`**: Rules and naming conventions for declaring robust PostgreSQL tables and relations via Drizzle.
- **`entities`**: Designing DDD Entities and Aggregates with private state encapsulation and explicit getters.
- **`errors`**: Constructing highly-typed, branded domain errors for `neverthrow`.
- **`i18n`**: Zero-dependency internationalization setup for both client and server, tightly integrated with component bounds.
- **`repositories`**: Writing Database interfaces and infrastructure logic (Drizzle, node-postgres) with strictly typed transactions.
- **`routing`**: Managing Next.js middleware, layouts, i18n URL localization, cookies, and protected route logic.
- **`server-actions`**: Safely authoring Zod-validated Next.js Server Actions returning discriminated union types mapped from Use Case results.
- **`testing`**: Stack-specific guidelines for Vitest integration utilizing Testcontainers and transactional DB helpers (`txTest`).
- **`use-cases`**: Structuring DDD Use Cases to orchestrate data flow between repositories and the presentation layer, relying heavily on `neverthrow` Results.
- **`value-objects`**: Patterns for Value Object immutability, private constructors, robust validation, and co-located error handling.
