# SQLiteData AI Agent Skill

This repository provides an AI Agent "Skill" crafted for large language model coding assistants (such as Cursor, Goose, Cline, or Claude MCP tools). It empowers the AI with comprehensive knowledge on how to leverage the [SQLiteData](https://github.com/pointfreeco/sqlite-data) library—a fast, lightweight, SQL-powered replacement for SwiftData supporting CloudKit synchronization.

## What is this?

Agent "Skills" are specialized folders containing instructions, patterns, and examples that extend an AI agent's capabilities. The core of this integration is the `SKILL.md` file, which includes detailed context, explanations, and best-practice use cases for working with SQLiteData. By reading this skill, agents learn exactly how to configure databases, define table structs via macros, properly execute `.write` transactions, and write type-safe GRDB and StructuredQueries code.

## Installation / Usage

To introduce this skill to your AI assistant, expose the `SKILL.md` file or this repository directory to your agent's context.

### Depending on your Setup:

**Generic Platform (Goose, Cursor, IDEs):**
1. Clone this repository to your computer:
   ```bash
   git clone https://github.com/bitxeno/sqlite-data-skill.git
   ```
2. Move the folder to your agent's dedicated `skills/` or `.agents/` directory.

**IDE-Specific Shortcuts:**
- Alternatively, copy the `SKILL.md` file into your Xcode/app project root and rename it to `.cursorrules` (or simply refer the AI to read `.cursorrules`). The AI will naturally incorporate the instructions into its generated code.

**Manual Chat Prompts:**
- Open `SKILL.md`, copy its entire content, and paste it to your conversational LLM companion (like ChatGPT, Claude). Ask it to use the information as a reference to help you build or rewrite features using SQLiteData.

## Content Overview

The included `SKILL.md` manual teaches the AI handler how to:
1. **Define Schemas**: Use the `@Table` macro with structs instead of `@Model` classes.
2. **Database Preparation**: Implement `appDatabase()` setup, tracing, testing configurations, and migrations.
3. **Data Fetching**: Use `@FetchAll`, `@FetchOne`, `@Fetch`, properly order items, and resolve joins.
4. **Change Observation**: Correctly wire updates to SwiftUI views and handle `@Observable` states alongside `@ObservationIgnored`.
5. **Dynamic Queries**: Build reactive queries leveraging `$property.load()`.
6. **CRUD Operations**: Centralizing writes via dependency injection (`defaultDatabase`).
7. **Associations**: Write proper native SQL joints rather than ORM pattern links.

