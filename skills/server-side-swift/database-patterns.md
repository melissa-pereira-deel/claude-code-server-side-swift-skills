# Database Patterns

Fluent ORM patterns including models, migrations, queries, relationships, and performance.

## Fluent Model Definition

### Basic Model

```swift
import Fluent
import Vapor

final class Todo: Model, Content, @unchecked Sendable {
    static let schema = "todos"

    @ID(key: .id)
    var id: UUID?

    @Field(key: "title")
    var title: String

    @Field(key: "is_complete")
    var isComplete: Bool

    @Timestamp(key: "created_at", on: .create)
    var createdAt: Date?

    @Timestamp(key: "updated_at", on: .update)
    var updatedAt: Date?

    @Timestamp(key: "deleted_at", on: .delete)
    var deletedAt: Date?

    init() {}

    init(id: UUID? = nil, title: String, isComplete: Bool = false) {
        self.id = id
        self.title = title
        self.isComplete = isComplete
    }
}
```

### Model with Relationships

```swift
final class User: Model, Content, @unchecked Sendable {
    static let schema = "users"

    @ID(key: .id)
    var id: UUID?

    @Field(key: "name")
    var name: String

    @Field(key: "email")
    var email: String

    // One-to-many: User has many Todos
    @Children(for: \.$user)
    var todos: [Todo]

    // One-to-one: User has one Profile
    @OptionalChild(for: \.$user)
    var profile: Profile?

    init() {}

    init(id: UUID? = nil, name: String, email: String) {
        self.id = id
        self.name = name
        self.email = email
    }
}

final class Todo: Model, Content, @unchecked Sendable {
    static let schema = "todos"

    @ID(key: .id)
    var id: UUID?

    @Field(key: "title")
    var title: String

    // Many-to-one: Todo belongs to User
    @Parent(key: "user_id")
    var user: User

    // Many-to-many: Todo has many Tags (via pivot)
    @Siblings(through: TodoTag.self, from: \.$todo, to: \.$tag)
    var tags: [Tag]

    init() {}

    init(id: UUID? = nil, title: String, userID: UUID) {
        self.id = id
        self.title = title
        self.$user.id = userID
    }
}
```

### Pivot Model (Many-to-Many)

```swift
final class TodoTag: Model, @unchecked Sendable {
    static let schema = "todo_tags"

    @ID(key: .id)
    var id: UUID?

    @Parent(key: "todo_id")
    var todo: Todo

    @Parent(key: "tag_id")
    var tag: Tag

    init() {}

    init(id: UUID? = nil, todoID: UUID, tagID: UUID) {
        self.id = id
        self.$todo.id = todoID
        self.$tag.id = tagID
    }
}
```

### Enum Fields

```swift
enum TodoPriority: String, Codable, CaseIterable {
    case low, medium, high, critical
}

final class Todo: Model, Content, @unchecked Sendable {
    static let schema = "todos"

    @ID(key: .id)
    var id: UUID?

    @Enum(key: "priority")
    var priority: TodoPriority

    init() {}
}
```

## Migrations

### Create Table Migration

```swift
struct CreateTodo: AsyncMigration {
    func prepare(on database: Database) async throws {
        try await database.schema("todos")
            .id()
            .field("title", .string, .required)
            .field("is_complete", .bool, .required, .custom("DEFAULT FALSE"))
            .field("priority", .string, .required, .custom("DEFAULT 'medium'"))
            .field("user_id", .uuid, .required, .references("users", "id", onDelete: .cascade))
            .field("created_at", .datetime)
            .field("updated_at", .datetime)
            .field("deleted_at", .datetime)
            .create()
    }

    func revert(on database: Database) async throws {
        try await database.schema("todos").delete()
    }
}
```

### Add Index Migration

```swift
struct AddTodoIndexes: AsyncMigration {
    func prepare(on database: Database) async throws {
        try await database.schema("todos")
            .unique(on: "title", "user_id")  // Composite unique
            .create()

        // Custom SQL for complex indexes
        if let sql = database as? SQLDatabase {
            try await sql.raw("""
                CREATE INDEX idx_todos_user_complete
                ON todos (user_id, is_complete)
                WHERE deleted_at IS NULL
            """).run()
        }
    }

    func revert(on database: Database) async throws {
        if let sql = database as? SQLDatabase {
            try await sql.raw("DROP INDEX IF EXISTS idx_todos_user_complete").run()
        }
    }
}
```

### Add Column Migration

```swift
struct AddDueDateToTodos: AsyncMigration {
    func prepare(on database: Database) async throws {
        try await database.schema("todos")
            .field("due_date", .datetime)     // Nullable by default
            .update()
    }

    func revert(on database: Database) async throws {
        try await database.schema("todos")
            .deleteField("due_date")
            .update()
    }
}
```

### Seed Data Migration

```swift
struct SeedDefaultCategories: AsyncMigration {
    func prepare(on database: Database) async throws {
        let categories = ["Work", "Personal", "Shopping", "Health"]
        for name in categories {
            try await Category(name: name).save(on: database)
        }
    }

    func revert(on database: Database) async throws {
        try await Category.query(on: database)
            .filter(\.$name ~~ ["Work", "Personal", "Shopping", "Health"])
            .delete()
    }
}
```

### Migration Registration Order

```swift
// ✅ Always register in dependency order
app.migrations.add(CreateUser())      // 1. Users first (no dependencies)
app.migrations.add(CreateTag())       // 2. Tags (no dependencies)
app.migrations.add(CreateTodo())      // 3. Todos (depends on Users)
app.migrations.add(CreateTodoTag())   // 4. Pivot (depends on Todos + Tags)
app.migrations.add(SeedDefaultCategories())  // 5. Seeds last
app.migrations.add(AddDueDateToTodos())      // 6. Schema changes after creation
```

## Queries

### Basic CRUD

```swift
// Create
let todo = Todo(title: "Buy groceries", userID: userID)
try await todo.save(on: db)

// Read by ID
let todo = try await Todo.find(todoID, on: db)

// Read with filter
let todos = try await Todo.query(on: db)
    .filter(\.$isComplete == false)
    .sort(\.$createdAt, .descending)
    .all()

// Update
todo.title = "Updated title"
try await todo.save(on: db)

// Delete
try await todo.delete(on: db)

// Soft delete (if @Timestamp(on: .delete) exists)
try await todo.delete(on: db)           // Sets deleted_at
try await todo.delete(force: true, on: db) // Actually deletes row
```

### Advanced Queries

```swift
// Pagination
let page = try await Todo.query(on: db)
    .filter(\.$isComplete == false)
    .sort(\.$createdAt, .descending)
    .paginate(for: req)

// Count
let count = try await Todo.query(on: db)
    .filter(\.$isComplete == false)
    .count()

// Aggregate
let avgPriority = try await Todo.query(on: db)
    .average(\.$priorityValue)

// Group filtering
let results = try await Todo.query(on: db)
    .group(.or) { group in
        group.filter(\.$title ~~ "important")  // Contains
        group.filter(\.$priority == .critical)
    }
    .all()

// Range filter
let recentTodos = try await Todo.query(on: db)
    .filter(\.$createdAt >= Date().addingTimeInterval(-86400))  // Last 24h
    .all()

// First or fail
let todo = try await Todo.query(on: db)
    .filter(\.$id == todoID)
    .first() ?? throw Abort(.notFound)
```

### Eager Loading (Avoiding N+1)

```swift
// ❌ Bad — N+1 query problem
let users = try await User.query(on: db).all()
for user in users {
    let todos = try await user.$todos.get(on: db)  // 1 query per user!
}

// ✅ Good — Eager load with .with()
let users = try await User.query(on: db)
    .with(\.$todos)          // Single JOIN query
    .all()

// Nested eager loading
let users = try await User.query(on: db)
    .with(\.$todos) { todo in
        todo.with(\.$tags)   // Load tags for each todo too
    }
    .with(\.$profile)
    .all()
```

### Sibling Relationships

```swift
// Attach tag to todo (many-to-many)
try await todo.$tags.attach(tag, on: db)

// Detach
try await todo.$tags.detach(tag, on: db)

// Check if attached
let isAttached = try await todo.$tags.isAttached(to: tag, on: db)

// Query through siblings
let todosWithTag = try await Tag.query(on: db)
    .filter(\.$name == "urgent")
    .with(\.$todos)
    .first()?
    .todos ?? []
```

### Raw SQL Queries

```swift
// When Fluent can't express the query
if let sql = req.db as? SQLDatabase {
    let results = try await sql.raw("""
        SELECT u.name, COUNT(t.id) as todo_count
        FROM users u
        LEFT JOIN todos t ON t.user_id = u.id
        WHERE t.deleted_at IS NULL
        GROUP BY u.id, u.name
        HAVING COUNT(t.id) > \(bind: minCount)
        ORDER BY todo_count DESC
        LIMIT \(bind: limit)
    """).all(decoding: UserTodoCount.self)
}

struct UserTodoCount: Decodable {
    let name: String
    let todoCount: Int

    enum CodingKeys: String, CodingKey {
        case name
        case todoCount = "todo_count"
    }
}
```

## Database Configuration

### PostgreSQL

```swift
app.databases.use(
    .postgres(
        configuration: .init(
            hostname: Environment.get("DB_HOST") ?? "localhost",
            port: Environment.get("DB_PORT").flatMap(Int.init) ?? 5432,
            username: Environment.get("DB_USER") ?? "vapor",
            password: Environment.get("DB_PASS") ?? "vapor",
            database: Environment.get("DB_NAME") ?? "app_db",
            tls: .prefer(try .init(configuration: .clientDefault))
        ),
        maxConnectionsPerEventLoop: 4,
        connectionPoolTimeout: .seconds(10)
    ),
    as: .psql
)
```

### SQLite (Development)

```swift
app.databases.use(.sqlite(.file("db.sqlite")), as: .sqlite)

// In-memory for testing
app.databases.use(.sqlite(.memory), as: .sqlite)
```

### Multiple Databases

```swift
// Primary (PostgreSQL)
app.databases.use(.postgres(configuration: primaryConfig), as: .psql)

// Analytics (separate database)
app.databases.use(.postgres(configuration: analyticsConfig), as: .analytics)

// Use specific database
let events = try await AnalyticsEvent.query(on: req.db(.analytics)).all()
```

## Transactions

```swift
// ✅ Use transactions for multi-step operations
try await req.db.transaction { db in
    let user = User(name: "John", email: "john@example.com")
    try await user.save(on: db)

    let todo = Todo(title: "Welcome task", userID: user.id!)
    try await todo.save(on: db)

    // If any step fails, entire transaction rolls back
}
```

## Performance Patterns

### Field Selection

```swift
// ✅ Only select needed fields
let titles = try await Todo.query(on: db)
    .field(\.$id)
    .field(\.$title)
    .all()
```

### Batch Operations

```swift
// ✅ Batch insert
let todos = titles.map { Todo(title: $0, userID: userID) }
try await todos.create(on: db)

// ✅ Batch update
try await Todo.query(on: db)
    .filter(\.$user.$id == userID)
    .set(\.$isComplete, to: true)
    .update()

// ✅ Batch delete
try await Todo.query(on: db)
    .filter(\.$isComplete == true)
    .filter(\.$updatedAt < oneMonthAgo)
    .delete()
```

### Connection Pool Monitoring

```swift
// Log slow queries in development
if app.environment == .development {
    app.databases.middleware.use(QueryLogMiddleware())
}

struct QueryLogMiddleware: AsyncModelMiddleware {
    func create(model: some Model, on db: Database, next: AnyAsyncModelResponder) async throws {
        let start = Date()
        try await next.create(model, on: db)
        let elapsed = Date().timeIntervalSince(start)
        if elapsed > 0.1 {
            db.logger.warning("Slow create: \(type(of: model)) took \(elapsed)s")
        }
    }
}
```

## Anti-Patterns

### ❌ Missing Migration Revert

```swift
// ❌ Bad — No revert means you can't rollback
struct CreateTodo: AsyncMigration {
    func prepare(on database: Database) async throws {
        try await database.schema("todos").id().field("title", .string).create()
    }
    func revert(on database: Database) async throws {
        // Empty! Can't rollback.
    }
}

// ✅ Good — Always implement revert
func revert(on database: Database) async throws {
    try await database.schema("todos").delete()
}
```

### ❌ Non-Optional Foreign Keys Without Cascade

```swift
// ❌ Bad — Deleting user will fail if todos exist
.field("user_id", .uuid, .required, .references("users", "id"))

// ✅ Good — Define cascade behavior
.field("user_id", .uuid, .required, .references("users", "id", onDelete: .cascade))
```

### ❌ Querying Inside Loops

```swift
// ❌ Bad — N queries
for id in todoIDs {
    let todo = try await Todo.find(id, on: db)
}

// ✅ Good — Single query
let todos = try await Todo.query(on: db)
    .filter(\.$id ~~ todoIDs)
    .all()
```

## References

- [Fluent Documentation](https://docs.vapor.codes/fluent/overview/)
- [PostgresNIO](https://github.com/vapor/postgres-nio)
- **vapor-patterns.md** — Vapor integration
- **hummingbird-patterns.md** — Hummingbird integration
