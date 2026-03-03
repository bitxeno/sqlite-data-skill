# SQLiteData AI Agent Skill

This repository provides an AI Agent "Skill" crafted for large language model coding assistants (such as Cursor, Goose, Cline, or Claude MCP tools). It empowers the AI with comprehensive knowledge on how to leverage the [SQLiteData](https://github.com/pointfreeco/sqlite-data) library—a fast, lightweight, SQL-powered replacement for SwiftData supporting CloudKit synchronization.

## What is this?

Agent "Skills" are specialized folders containing instructions, patterns, and examples that extend an AI agent's capabilities. The core of this integration is the `SKILL.md` file, which includes detailed context, explanations, and best-practice use cases for working with SQLiteData. By reading this skill, agents learn exactly how to configure databases, define table structs via macros, properly execute `.write` transactions, and write type-safe GRDB and StructuredQueries code.

## Installation / Usage

Depending on your agent platform, there are multiple ways to install and use this skill:

### Using `npx` (Vercel Agent Skills / Others)

If your agent supports standard npm-based skill installation configurations, you can easily add this directly via `npx`:

```bash
npx skills add https://github.com/bitxeno/sqlite-data-skill
```

This single command will clone and resolve the instructions of `SKILL.md` within your project's `/.skills` registry so your AI agent understands all rules related to the `SQLiteData` package.

### Clone Locally (Generic approach)

Using basic Git cloning, you can pull this directly to your agent's knowledge workspace:

```bash
git clone https://github.com/bitxeno/sqlite-data-skill.git .skills/sqlite-data
```

### IDE-Specific Shortcuts (Cursor, Cline, etc.)

- Alternatively, copy the `SKILL.md` file into your Xcode/app project root and rename it to `.cursorrules` (or simply refer the AI to read `.cursorrules`). The AI will naturally incorporate the instructions into its generated code.

### Manual Chat Prompts

- Open `SKILL.md`, copy its entire content, and paste it to your conversational LLM companion (like ChatGPT, Claude). Ask it to use the information as a reference to help you build or rewrite features using SQLiteData.

## Content Overview

The included `SKILL.md` manual teaches the AI handler how to:

1. **Define Schemas**: Use the `@Table` macro with structs instead of `@Model` classes.
2. **Database Preparation**: Implement `appDatabase()` setup, tracing, testing configurations, and migrations.
3. **Data Fetching**: Use `@FetchAll`, `@FetchOne`, `@Fetch`, properly order items, and resolve joins.
4. **Change Observation**: Correctly wire updates to SwiftUI views and handle `@Observable` states alongside `@ObservationIgnored`.
5. **Dynamic Queries**: Build reactive queries leveraging `$property.load()`.
6. **CRUD Operations**: Centralizing writes via dependency injection (`defaultDatabase`).
7. **Associations**: Write proper native SQL joins rather than ORM pattern links.
