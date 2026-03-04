---
name: SQLiteData Usage Guide
description: Comprehensive guide for using the SQLiteData library — a fast, lightweight replacement for SwiftData powered by SQL with CloudKit synchronization support.
---

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
9. [Testing & Previews](#9-testing--previews)
10. [Complete Examples](#10-complete-examples)

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

### Custom Column Mapping

```swift
@Table
struct RemindersList: Hashable, Identifiable {
  let id: UUID
  @Column(as: Color.HexRepresentation.self)
  var color: Color = Self.defaultColor
  var position = 0
  var title = ""
}
```

### Custom Primary Key

```swift
@Table
struct Tag: Hashable, Identifiable {
  @Column(primaryKey: true)
  var title: String
  var id: String { title }
}
```

### Enums in Tables

Enums must conform to `QueryBindable`:

```swift
@Table
struct Reminder: Identifiable {
  let id: UUID
  var priority: Priority?
  var status: Status = .incomplete

  enum Priority: Int, QueryBindable {
    case low = 1
    case medium
    case high
  }
  enum Status: Int, QueryBindable {
    case incomplete = 0
    case completed = 1
    case completing = 2
  }
}
```

### Computed Query Expressions

You can define computed query expressions on your table's `TableColumns`:

```swift
nonisolated extension Reminder.TableColumns {
  var isCompleted: some QueryExpression<Bool> {
    status.neq(Reminder.Status.incomplete)
  }
  var isPastDue: some QueryExpression<Bool> {
    @Dependency(\.date.now) var now
    return !isCompleted && #sql("coalesce(date(\(dueDate)) < date(\(now)), 0)")
  }
  var isToday: some QueryExpression<Bool> {
    @Dependency(\.date.now) var now
    return !isCompleted && #sql("coalesce(date(\(dueDate)) = date(\(now)), 0)")
  }
  var isScheduled: some QueryExpression<Bool> {
    !isCompleted && dueDate.isNot(nil)
  }
}
```

### Pre-defined Query Shortcuts

```swift
extension Reminder {
  static let incomplete = Self.where { !$0.isCompleted }
  static let withTags = group(by: \.id)
    .leftJoin(ReminderTag.all) { $0.id.eq($1.reminderID) }
    .leftJoin(Tag.all) { $1.tagID.eq($2.primaryKey) }
}
```

### @Selection Macro for Custom Result Types

Use `@Selection` for custom types that hold the results of joins or partial selects:

```swift
@Selection
struct ReminderListState: Identifiable {
  var id: RemindersList.ID { remindersList.id }
  var remindersCount: Int
  var remindersList: RemindersList
  @Column(as: CKShare?.SystemFieldsRepresentation.self)
  var share: CKShare?
}

@Selection
struct Stats {
  var allCount = 0
  var flaggedCount = 0
  var scheduledCount = 0
  var todayCount = 0
}
```

### Junction Tables (Many-to-Many)

```swift
@Table("remindersTags")
struct ReminderTag: Identifiable {
  let id: UUID
  let reminderID: Reminder.ID
  let tagID: Tag.ID
}
```

### Full-Text Search (FTS5)

```swift
@Table
struct ReminderText: FTS5 {
  let rowid: Int
  let title: String
  let notes: String
  let tags: String
}
```

---

## 2. Preparing the Database

### Step 1: Create `appDatabase()` Function

```swift
import OSLog
import SQLiteData

func appDatabase() throws -> any DatabaseWriter {
  @Dependency(\.context) var context
  var configuration = Configuration()

  // Optional: Enable query tracing for debugging
  #if DEBUG
    configuration.prepareDatabase { db in
      db.trace(options: .profile) {
        if context == .preview {
          print("\($0.expandedDescription)")
        } else {
          logger.debug("\($0.expandedDescription)")
        }
      }
    }
  #endif

  // Create database (auto-provisions unique temp DBs for previews/tests)
  let database = try defaultDatabase(configuration: configuration)
  logger.info("open '\(database.path)'")

  // Migrate
  var migrator = DatabaseMigrator()
  #if DEBUG
    migrator.eraseDatabaseOnSchemaChange = true
  #endif

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

  // Register more migrations as your app evolves
  migrator.registerMigration("Add 'description' column") { db in
    try #sql("""
      ALTER TABLE "items"
      ADD COLUMN "description" TEXT
      """)
      .execute(db)
  }

  try migrator.migrate(database)
  return database
}

private let logger = Logger(subsystem: "MyApp", category: "Database")
```

### Step 2: Set Default Database in App Entry Point

**SwiftUI:**

```swift
import SQLiteData
import SwiftUI

@main
struct MyApp: App {
  init() {
    prepareDependencies {
      $0.defaultDatabase = try! appDatabase()
    }
  }

  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}
```

**UIKit AppDelegate:**

```swift
class AppDelegate: NSObject, UIApplicationDelegate {
  func applicationDidFinishLaunching(_ application: UIApplication) {
    prepareDependencies {
      $0.defaultDatabase = try! appDatabase()
    }
  }
}
```

### Bootstrap Pattern (Recommended for Multiple Dependencies)

Group database + sync engine setup:

```swift
extension DependencyValues {
  mutating func bootstrapDatabase() throws {
    defaultDatabase = try appDatabase()
    defaultSyncEngine = try SyncEngine(
      for: defaultDatabase,
      tables: RemindersList.self, Reminder.self, Tag.self, ReminderTag.self
    )
  }
}

// In App entry point:
@main
struct MyApp: App {
  init() {
    try! prepareDependencies {
      try $0.bootstrapDatabase()
    }
  }
  // ...
}
```

---

## 3. Fetching Data

### @FetchAll — Fetch a Collection

Fetch all rows with default ordering:

```swift
@FetchAll var items: [Item]
```

With ordering:

```swift
@FetchAll(Item.order(by: \.title))
var items
```

With ordering by multiple columns:

```swift
@FetchAll(Item.order(by: \.isInStock, \.title))
var items
```

With filtering:

```swift
@FetchAll(Reminder.where(\.isCompleted).order { $0.title.desc() })
var completedReminders
```

With ordering by multiple columns using a closure:

```swift
@FetchAll(
  Item.order {
    ($0.isInStock,
     $0.title)
  }
)
var items
```

With mixed sort directions across multiple columns:

```swift
@FetchAll(
  Item.order {
    ($0.isInStock,
     $0.title.desc())
  }
)
var items
```

With animation:

```swift
@FetchAll(Item.order { $0.id.desc() }, animation: .default)
private var items
```

Using SQL string (`#sql` macro):

```swift
@FetchAll(#sql("SELECT * FROM reminders WHERE isCompleted ORDER BY title DESC"))
var completedReminders: [Reminder]
```

Using schema-safe SQL interpolation:

```swift
@FetchAll(
  #sql("""
    SELECT \(Reminder.columns)
    FROM \(Reminder.self)
    WHERE \(Reminder.isCompleted)
    ORDER BY \(Reminder.title) DESC
    """)
)
var completedReminders: [Reminder]
```

### @FetchOne — Fetch a Single Value

Count:

```swift
@FetchOne(Reminder.count())
var remindersCount = 0
```

Filtered count:

```swift
@FetchOne(Reminder.where(\.isCompleted).count())
var completedRemindersCount = 0
```

Aggregate computation with `@Selection`:

```swift
@FetchOne(
  Reminder.select {
    Stats.Columns(
      allCount: $0.count(filter: !$0.isCompleted),
      flaggedCount: $0.count(filter: $0.isFlagged && !$0.isCompleted),
      scheduledCount: $0.count(filter: $0.isScheduled),
      todayCount: $0.count(filter: $0.isToday)
    )
  }
)
var stats = Stats()
```

### @Fetch — Multiple Queries in One Transaction

Define a `FetchKeyRequest`:

```swift
struct Facts: FetchKeyRequest {
  struct Value {
    var facts: [Fact] = []
    var count = 0
  }
  func fetch(_ db: Database) throws -> Value {
    try Value(
      facts: Fact.order { $0.id.desc() }.fetchAll(db),
      count: Fact.fetchCount(db)
    )
  }
}
```

Use it:

```swift
@Fetch(Facts(), animation: .default)
private var facts = Facts.Value()

// Access:
facts.facts   // [Fact]
facts.count   // Int
```

### Joins and Custom Selections

```swift
@FetchAll(
  RemindersList
    .group(by: \.id)
    .order(by: \.position)
    .leftJoin(Reminder.all) { $0.id.eq($1.remindersListID) && !$1.isCompleted }
    .leftJoin(SyncMetadata.all) { $0.syncMetadataID.eq($2.id) }
    .select {
      ReminderListState.Columns(
        remindersCount: $1.id.count(),
        remindersList: $0,
        share: $2.share
      )
    },
  animation: .default
)
var remindersLists
```

---

## 4. Observing Changes

### SwiftUI Views

Property wrappers automatically observe database changes and re-render:

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

### @Observable Models

> **Important:** Must annotate with `@ObservationIgnored` due to macro interactions; SQLiteData handles its own observation.

```swift
@Observable
@MainActor
class ItemsModel {
  @ObservationIgnored
  @FetchAll(Item.order { $0.id.desc() }, animation: .default)
  var items

  @ObservationIgnored
  @FetchOne(Item.count(), animation: .default)
  var itemsCount = 0

  @ObservationIgnored
  @Dependency(\.defaultDatabase) var database

  func deleteItem(indices: IndexSet) {
    withErrorReporting {
      try database.write { db in
        let ids = indices.map { items[$0].id }
        try Item
          .where { $0.id.in(ids) }
          .delete()
          .execute(db)
      }
    }
  }
}
```

### UIKit View Controllers

Use `$items.publisher` (Combine) or `observe` (Swift Navigation):

```swift
class ItemsViewController: UICollectionViewController {
  @FetchAll(Fact.order { $0.id.desc() }, animation: .default)
  private var facts

  override func viewDidLoad() {
    super.viewDidLoad()
    // Using Swift Navigation's observe:
    observe { [weak self] in
      guard let self else { return }
      var snapshot = NSDiffableDataSourceSnapshot<Section, Fact>()
      snapshot.appendSections([.facts])
      snapshot.appendItems(facts, toSection: .facts)
      dataSource.apply(snapshot, animatingDifferences: true)
    }
  }
}
```

---

## 5. Dynamic Queries

Use `$items.load(...)` to dynamically change the query based on user input:

### In a SwiftUI View

```swift
struct ContentView: View {
  @FetchAll var items: [Item]
  @State var filterDate: Date?
  @State var order: SortOrder = .reverse

  var body: some View {
    List { /* ... */ }
    .task(id: [filterDate, order] as [AnyHashable]) {
      await updateQuery()
    }
  }

  private func updateQuery() async {
    do {
      try await $items.load(
        Item
          .where { $0.timestamp > #bind(filterDate ?? .distantPast) }
          .order {
            if order == .forward {
              $0.timestamp
            } else {
              $0.timestamp.desc()
            }
          }
          .limit(10)
      )
    } catch { /* Handle error */ }
  }
}
```

### Dynamic Query with @Fetch

```swift
@Fetch(Facts(), animation: .default) private var facts = Facts.Value()
@State var query = ""

var body: some View {
  List { /* ... */ }
  .searchable(text: $query)
  .task(id: query) {
    await withErrorReporting {
      try await $facts.load(Facts(query: query), animation: .default).task
    }
  }
}

private struct Facts: FetchKeyRequest {
  var query = ""
  struct Value {
    var facts: [Fact] = []
    var searchCount = 0
    var totalCount = 0
  }
  func fetch(_ db: Database) throws -> Value {
    let search = Fact
      .where { $0.body.contains(query) }
      .order { $0.id.desc() }
    return try Value(
      facts: search.fetchAll(db),
      searchCount: search.fetchCount(db),
      totalCount: Fact.fetchCount(db)
    )
  }
}
```

> **Important:** If a parent view refreshes, a dynamically-updated query can be overwritten with the initial `@FetchAll`'s value. Use `@State @FetchAll` to keep query state local to the view.

### Pagination

#### Offset-based Pagination

Use `.limit(_:offset:)` to fetch a specific page of results:

```swift
let pageSize = 20
let page = 2  // 0-indexed

@FetchAll(
  Item
    .order { $0.title }
    .limit(pageSize, offset: page * pageSize)
)
var items
```

Dynamically load a page in response to user input:

```swift
struct ItemsView: View {
  @State @FetchAll var items: [Item]
  @State var page = 0
  let pageSize = 20

  var body: some View {
    List(items) { item in Text(item.title) }
    HStack {
      Button("Previous") { page = max(0, page - 1) }
      Button("Next") { page += 1 }
    }
    .task(id: page) {
      await withErrorReporting {
        try await $items.load(
          Item
            .order { $0.title }
            .limit(pageSize, offset: page * pageSize)
        )
      }
    }
  }
}
```

#### Cursor (Keyset) Pagination

Keyset pagination uses the last seen value as a cursor, which is more efficient and stable than offset pagination for large datasets:

```swift
// First page — no cursor
@FetchAll(
  Item
    .order { $0.id }
    .limit(20)
)
var items
```

```swift
// Next page — pass the last seen id as cursor
let lastID = items.last?.id

@FetchAll(
  Item
    .where { $0.id > #bind(lastID ?? 0) }
    .order { $0.id }
    .limit(20)
)
var nextItems
```

Dynamically load the next page inside a model:

```swift
@Observable
@MainActor
class ItemsModel {
  @ObservationIgnored
  @State @FetchAll var items: [Item]

  @Dependency(\.defaultDatabase) var database

  var lastID: Item.ID? = nil

  func loadNextPage() async {
    await withErrorReporting {
      try await $items.load(
        Item
          .where { $0.id > #bind(lastID ?? 0) }
          .order { $0.id }
          .limit(20)
      )
      lastID = items.last?.id
    }
  }
}
```

---

## 6. CRUD Operations

All write operations go through the `defaultDatabase` dependency.

### Access Database

```swift
@Dependency(\.defaultDatabase) var database
```

### Create (Insert)

```swift
try database.write { db in
  try Item.insert {
    Item(id: UUID(), title: "New Item", isInStock: true, notes: "")
  }
  .execute(db)
}
```

Using `Draft` (auto-generated type without primary key):

```swift
try database.write { db in
  try Fact.insert {
    Fact.Draft(body: "Some fact text")
  }
  .execute(db)
}
```

Insert with defaults (using table defaults):

```swift
try database.write { db in
  try Item.insert().execute(db)
}
```

### Read (Fetch)

Use `@FetchAll`, `@FetchOne`, or `@Fetch` property wrappers (covered above).

For one-off reads within a write transaction:

```swift
try database.write { db in
  let count = try Reminder.fetchCount(db)
  let items = try Item.where(\.isInStock).fetchAll(db)
}
```

### Update

Update an existing row:

```swift
existingItem.title = "New Title"
try database.write { db in
  try Item.update(existingItem).execute(db)
}
```

Conditional update with query:

```swift
try database.write { db in
  try Reminder
    .where { $0.status.eq(#bind(.completing)) }
    .update { $0.status = #bind(.completed) }
    .execute(db)
}
```

### Delete

Delete by value:

```swift
try database.write { db in
  try Item.delete(existingItem).execute(db)
}
```

Delete by query:

```swift
try database.write { db in
  try Item
    .where { $0.id.in(idsToDelete) }
    .delete()
    .execute(db)
}
```

Delete with subquery:

```swift
try database.write { db in
  try Tag
    .where { $0.title.in(tagTitles) }
    .delete()
    .execute(db)
}
```

### Batch Reordering

```swift
try database.write { db in
  var ids = items.map(\.id)
  ids.move(fromOffsets: source, toOffset: destination)
  try Item
    .where { $0.id.in(ids) }
    .update {
      let indexedIDs = Array(ids.enumerated())
      let (first, rest) = (indexedIDs.first!, indexedIDs.dropFirst())
      $0.position = rest
        .reduce(Case($0.id).when(first.element, then: first.offset)) { cases, id in
          cases.when(id.element, then: id.offset)
        }
        .else($0.position)
    }
    .execute(db)
}
```

---

## 7. Associations & Joins

SQLiteData does **not** use an ORM pattern. Instead, use SQL joins to efficiently fetch related data.

### One-to-Many (Join + Group)

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

### Many-to-Many via Junction Table

```swift
extension Tag {
  static let withReminders = group(by: \.primaryKey)
    .leftJoin(ReminderTag.all) { $0.primaryKey.eq($1.tagID) }
    .leftJoin(Reminder.all) { $1.reminderID.eq($2.id) }
}

// Use:
@FetchAll(
  Tag
    .order(by: \.title)
    .withReminders
    .having { $2.count().gt(0) }
    .select { tag, _, _ in tag },
  animation: .default
)
var tags
```

### Three-Way Join

```swift
@FetchAll(
  RemindersList
    .group(by: \.id)
    .order(by: \.position)
    .leftJoin(Reminder.all) { $0.id.eq($1.remindersListID) && !$1.isCompleted }
    .leftJoin(SyncMetadata.all) { $0.syncMetadataID.eq($2.id) }
    .select {
      ReminderListState.Columns(
        remindersCount: $1.id.count(),
        remindersList: $0,
        share: $2.share
      )
    },
  animation: .default
)
var remindersLists
```

---

## 8. CloudKit Synchronization

### Setup SyncEngine

```swift
@main
struct MyApp: App {
  init() {
    try! prepareDependencies {
      $0.defaultDatabase = try appDatabase()
      $0.defaultSyncEngine = try SyncEngine(
        for: $0.defaultDatabase,
        tables: RemindersList.self, Reminder.self, Tag.self
      )
    }
  }
}
```

### Attach Metadatabase

In your database configuration, attach the metadata database required for sync:

```swift
configuration.prepareDatabase { db in
  try db.attachMetadatabase()
}
```

### SyncEngine Delegate

```swift
@MainActor
@Observable
class MySyncEngineDelegate: SyncEngineDelegate {
  var isDeleteLocalDataAlertPresented = false
  func syncEngine(
    _ syncEngine: SQLiteData.SyncEngine,
    accountChanged changeType: CKSyncEngine.Event.AccountChange.ChangeType
  ) async {
    switch changeType {
    case .signIn: break
    case .signOut, .switchAccounts:
      isDeleteLocalDataAlertPresented = true
    @unknown default: break
    }
  }
}
```

### Scene Delegate for Sharing

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
  @Dependency(\.defaultSyncEngine) var syncEngine

  func windowScene(
    _ windowScene: UIWindowScene,
    userDidAcceptCloudKitShareWith cloudKitShareMetadata: CKShare.Metadata
  ) {
    Task {
      try await syncEngine.acceptShare(metadata: cloudKitShareMetadata)
    }
  }
}
```

### Manual Sync

```swift
@Dependency(\.defaultSyncEngine) var syncEngine

// Pull changes:
try await syncEngine.syncChanges()

// Delete local data:
try await syncEngine.deleteLocalData()
```

### Table Definitions for CloudKit Sync

Use `ON CONFLICT REPLACE DEFAULT` for columns to handle sync conflicts:

```swift
try #sql("""
  CREATE TABLE "counters" (
    "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
    "count" INT NOT NULL ON CONFLICT REPLACE DEFAULT 0
  ) STRICT
  """)
  .execute(db)
```

---

## 9. Testing & Previews

### Xcode Previews

```swift
#Preview {
  let _ = prepareDependencies {
    $0.defaultDatabase = try! appDatabase()
  }
  NavigationStack {
    ContentView()
  }
}
```

### With Seed Data

```swift
#if DEBUG
extension DatabaseWriter {
  func seedSampleData() throws {
    try write { db in
      try db.seed {
        Item(id: UUID(), title: "Sample 1", isInStock: true, notes: "")
        Item(id: UUID(), title: "Sample 2", isInStock: false, notes: "Note")
      }
    }
  }
}
#endif

#Preview {
  let _ = try! prepareDependencies {
    try $0.bootstrapDatabase()
    try? $0.defaultDatabase.seedSampleData()
  }
  NavigationStack {
    ContentView()
  }
}
```

### In-Memory Database for Testing

```swift
extension DatabaseWriter where Self == DatabaseQueue {
  static var testDatabase: Self {
    let databaseQueue = try! DatabaseQueue()
    var migrator = DatabaseMigrator()
    migrator.registerMigration("Create tables") { db in
      try #sql("""
        CREATE TABLE "facts" (
          "id" INTEGER PRIMARY KEY AUTOINCREMENT,
          "body" TEXT NOT NULL
        ) STRICT
        """)
        .execute(db)
    }
    try! migrator.migrate(databaseQueue)
    return databaseQueue
  }
}
```

### Unit Tests

```swift
@Test(.dependency(\.defaultDatabase, try appDatabase()))
func feature() {
  // ...
}
```

---

## 10. Complete Examples

### Minimal SwiftUI App (SwiftData Template Replacement)

```swift
import SQLiteData
import SwiftUI

@main
struct MyApp: App {
  init() {
    prepareDependencies {
      $0.defaultDatabase = .appDatabase
    }
  }
  var body: some Scene {
    WindowGroup {
      ContentView()
    }
  }
}

struct ContentView: View {
  @FetchAll(Item.all, animation: .default) private var items
  @Dependency(\.defaultDatabase) private var database

  var body: some View {
    NavigationStack {
      List {
        ForEach(items) { item in
          Text(item.timestamp, format: Date.FormatStyle(date: .numeric, time: .standard))
        }
        .onDelete(perform: deleteItems)
      }
      .toolbar {
        ToolbarItem(placement: .navigationBarTrailing) { EditButton() }
        ToolbarItem {
          Button(action: addItem) {
            Label("Add Item", systemImage: "plus")
          }
        }
      }
    }
  }

  private func addItem() {
    withErrorReporting {
      try database.write { db in
        try Item.insert().execute(db)
      }
    }
  }

  private func deleteItems(offsets: IndexSet) {
    withErrorReporting {
      try database.write { db in
        try Item.where { $0.id.in(offsets.map { items[$0].id }) }.delete().execute(db)
      }
    }
  }
}

@Table
nonisolated struct Item: Identifiable {
  let id: Int
  var timestamp: Date
}

extension DatabaseWriter where Self == DatabaseQueue {
  static var appDatabase: Self {
    let databaseQueue = try! DatabaseQueue()
    var migrator = DatabaseMigrator()
    migrator.registerMigration("Create items table") { db in
      try #sql("""
        CREATE TABLE "items" (
          "id" INTEGER PRIMARY KEY AUTOINCREMENT,
          "timestamp" TEXT NOT NULL DEFAULT CURRENT_TIMESTAMP
        ) STRICT
        """)
        .execute(db)
    }
    try! migrator.migrate(databaseQueue)
    return databaseQueue
  }
}
```

### @Observable Model Pattern

```swift
@Observable
@MainActor
class FeatureModel {
  @ObservationIgnored
  @FetchAll(Item.order { $0.id.desc() }, animation: .default)
  var items

  @ObservationIgnored
  @FetchOne(Item.count(), animation: .default)
  var itemsCount = 0

  @ObservationIgnored
  @Dependency(\.defaultDatabase) var database

  func addItem(body: String) async {
    await withErrorReporting {
      try await database.write { db in
        try Item.insert { Item.Draft(body: body) }.execute(db)
      }
    }
  }

  func deleteItems(indices: IndexSet) {
    withErrorReporting {
      try database.write { db in
        let ids = indices.map { items[$0].id }
        try Item.where { $0.id.in(ids) }.delete().execute(db)
      }
    }
  }
}
```

### UIKit View Controller Pattern

```swift
final class ItemsViewController: UICollectionViewController {
  @FetchAll(Item.order { $0.id.desc() }, animation: .default)
  private var items

  @Dependency(\.defaultDatabase) var database

  private var dataSource: UICollectionViewDiffableDataSource<Section, Item>!

  override func viewDidLoad() {
    super.viewDidLoad()

    let cellRegistration = UICollectionView.CellRegistration<UICollectionViewListCell, Item> {
      cell, indexPath, item in
      var configuration = cell.defaultContentConfiguration()
      configuration.text = item.title
      cell.contentConfiguration = configuration
    }

    dataSource = UICollectionViewDiffableDataSource<Section, Item>(
      collectionView: collectionView
    ) { collectionView, indexPath, item in
      collectionView.dequeueConfiguredReusableCell(
        using: cellRegistration, for: indexPath, item: item
      )
    }

    // Observe database changes and update UI
    observe { [weak self] in
      guard let self else { return }
      var snapshot = NSDiffableDataSourceSnapshot<Section, Item>()
      snapshot.appendSections([.items])
      snapshot.appendItems(items, toSection: .items)
      dataSource.apply(snapshot, animatingDifferences: true)
    }
  }

  enum Section: Hashable { case items }
}
```

### Database Triggers Pattern

```swift
try database.write { db in
  // Auto-set position on insert
  try Item.createTemporaryTrigger(
    after: .insert { new in
      Item
        .find(new.id)
        .update { $0.position = Item.select { ($0.position.max() ?? -1) + 1 } }
    }
  )
  .execute(db)

  // Cascade FTS updates on insert
  try Reminder.createTemporaryTrigger(
    after: .insert { new in
      ReminderText.insert {
        ReminderText.Columns(
          rowid: new.rowid,
          title: new.title,
          notes: new.notes.replace("\\n", " "),
          tags: ""
        )
      }
    }
  )
  .execute(db)

  // Cascade FTS updates on column change
  try Reminder.createTemporaryTrigger(
    after: .update {
      ($0.title, $0.notes)
    } forEachRow: { _, new in
      ReminderText
        .where { $0.rowid.eq(new.rowid) }
        .update {
          $0.title = new.title
          $0.notes = new.notes.replace("\\n", " ")
        }
    }
  )
  .execute(db)

  // Cascade FTS delete
  try Reminder.createTemporaryTrigger(
    after: .delete { old in
      ReminderText
        .where { $0.rowid.eq(old.rowid) }
        .delete()
    }
  )
  .execute(db)
}
```

---

## Quick Reference

| SwiftData                      | SQLiteData                                         |
| ------------------------------ | -------------------------------------------------- |
| `@Model class`                 | `@Table struct`                                    |
| `@Query var items: [Item]`     | `@FetchAll var items: [Item]`                      |
| `@Query(sort:)`                | `@FetchAll(Item.order(by:))`                       |
| `@Query(filter:)`              | `@FetchAll(Item.where(...))`                       |
| N/A                            | `@FetchOne(Item.count())`                          |
| N/A                            | `@Fetch(CustomRequest())`                          |
| `@Environment(\.modelContext)` | `@Dependency(\.defaultDatabase)`                   |
| `modelContext.insert(item)`    | `try Item.insert { item }.execute(db)`             |
| `try modelContext.save()`      | (auto-saved within `database.write`)               |
| `modelContext.delete(item)`    | `try Item.delete(item).execute(db)`                |
| `ModelContainer(...)`          | `prepareDependencies { $0.defaultDatabase = ... }` |
| iOS 17+ only                   | iOS 13+ supported                                  |
| Views only (`@Query`)          | Views, `@Observable`, UIKit, anywhere              |

## Platform Support

SQLiteData supports: **iOS 13+, macOS 10.15+, tvOS 13+, watchOS 6+** — much broader than SwiftData's iOS 17+ requirement.

## References

- [GitHub Repository](https://github.com/pointfreeco/sqlite-data)
- [StructuredQueries Documentation](https://swiftpackageindex.com/pointfreeco/swift-structured-queries/main/documentation/structuredqueriescore/)
- [GRDB Documentation](https://swiftpackageindex.com/groue/grdb.swift/documentation/grdb)
- [Swift Dependencies](https://github.com/pointfreeco/swift-dependencies)
