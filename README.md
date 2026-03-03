# SQLiteData Usage Guide

SQLiteData is a fast, lightweight replacement for SwiftData from [Point-Free](https://github.com/pointfreeco/sqlite-data), powered by SQL and supporting CloudKit synchronization (and even CloudKit sharing). It is built on top of [GRDB](https://github.com/groue/GRDB.swift) and [StructuredQueries](https://github.com/pointfreeco/swift-structured-queries).

## Key Dependencies

- **GRDB**: The underlying SQLite interface library. Used for database connections, transactions, migrations, and observation.
- **StructuredQueries**: Provides the `@Table` macro and type-safe query building APIs.
- **Swift Dependencies**: Used for dependency injection (`@Dependency`, `prepareDependencies`).

## Table of Contents

1. [Defining Your Schema](#1-defining-your-schema)
2. [Preparing the Database](#2-preparing-the-database)
3. [Fetching Data](#3-fetching-data)
4. [Observing Changes](#4-observing-changes)
5. [Dynamic Queries](#5-dynamic-queries)
6. [CRUD Operations](#6-crud-operations)
7. [Associations & Joins](#7-associations--joins)
8. [CloudKit Synchronization](#8-cloudkit-synchronization)

---

## 1. Defining Your Schema

Use the `@Table` macro from StructuredQueries to define your data types. Unlike SwiftData's `@Model` (which requires classes), `@Table` works with **structs**.

### Basic Table Definition

```swift
import SQLiteData

@Table
struct Item: Identifiable {
  let id: Int            // Primary key (auto-generated for Int)
  var title = ""
  var isInStock = true
  var notes = ""
}
```

### UUID Primary Key

```swift
@Table
struct RemindersList: Identifiable {
  let id: UUID
  var title = ""
  var position = 0
}
```

## 2. Preparing the Database

```swift
import OSLog
import SQLiteData

func appDatabase() throws -> any DatabaseWriter {
  @Dependency(\.context) var context
  var configuration = Configuration()

  let database = try defaultDatabase(configuration: configuration)
  logger.info("open '\(database.path)'")

  var migrator = DatabaseMigrator()

  migrator.registerMigration("Create tables") { db in
    try #sql("""
      CREATE TABLE "items" (
        "id" INTEGER PRIMARY KEY AUTOINCREMENT,
        "title" TEXT NOT NULL DEFAULT '',
        "isInStock" INTEGER NOT NULL DEFAULT 1,
        "notes" TEXT NOT NULL DEFAULT ''
      ) STRICT
      """)
      .execute(db)
  }

  try migrator.migrate(database)
  return database
}
```

## 3. Fetching Data

Fetch all rows with default ordering:

```swift
@FetchAll var items: [Item]
```

## 4. Observing Changes

Property wrappers automatically observe database changes and re-render in SwiftUI Views:

```swift
struct ItemsView: View {
  @FetchAll var items: [Item]

  var body: some View {
    ForEach(items) { item in
      Text(item.name)
    }
  }
}
```

## 5. Dynamic Queries

Use `$items.load(...)` to dynamically change the query based on user input:

```swift
struct ContentView: View {
  @FetchAll var items: [Item]
  @State var filterDate: Date?

  var body: some View {
    List { /* ... */ }
    .task(id: filterDate) {
      await updateQuery()
    }
  }

  private func updateQuery() async {
    do {
      try await $items.load(
        Item
          .where { $0.timestamp > #bind(filterDate ?? .distantPast) }
          .limit(10)
      )
    } catch { /* Handle error */ }
  }
}
```

## 6. CRUD Operations

All write operations go through the `defaultDatabase` dependency.

```swift
@Dependency(\.defaultDatabase) var database

// Create
try database.write { db in
  try Item.insert {
    Item(id: UUID(), title: "New Item", isInStock: true, notes: "")
  }
  .execute(db)
}
```

## 7. Associations & Joins

SQLiteData does **not** use an ORM pattern. Instead, use SQL joins to efficiently fetch related data.

```swift
@FetchAll(
  Sport
    .group(by: \.id)
    .leftJoin(Team.all) { $0.id.eq($1.sportID) }
    .select {
      SportWithTeamCount.Columns(sport: $0, teamCount: $1.count())
    }
)
var sportsWithTeamCounts
```
