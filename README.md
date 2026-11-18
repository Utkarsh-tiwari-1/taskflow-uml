# TaskFlow UML

UML documentation for the [TaskFlow API](https://github.com/Utkarsh-tiwari-1/taskflow-api), a Spring Boot task-management REST API.

GitHub renders the Mermaid previews below. Editable PlantUML sources are included for each view.

| Diagram | PlantUML source |
|---|---|
| [Architecture](#architecture) | [TaskFlow-Architecture.puml](TaskFlow-Architecture.puml) |
| [Class diagram](#domain-class-diagram) | [TaskFlow-Domain.puml](TaskFlow-Domain.puml) |
| [Object diagram](#object-diagram) | [TaskFlow-Object-Diagram.puml](TaskFlow-Object-Diagram.puml) |
| [Database ER diagram](#database-er-diagram) | [TaskFlow-ERD.puml](TaskFlow-ERD.puml) |
| [Use case diagram](#use-case-diagram) | [TaskFlow-Use-Cases.puml](TaskFlow-Use-Cases.puml) |
| [Authentication sequence](#authentication-sequence) | [TaskFlow-Authentication-Sequence.puml](TaskFlow-Authentication-Sequence.puml) |
| [Activity diagram](#task-request-activity) | [TaskFlow-Task-Request-Activity.puml](TaskFlow-Task-Request-Activity.puml) |

## Architecture

```mermaid
flowchart LR
    client["API Client"] -->|HTTP / JSON| controllers["REST Controllers"]
    controllers -->|request and response DTOs| services["Application Services"]
    services -->|domain operations| domain["Domain Model"]
    services --> repositories["Spring Data Repositories"]
    repositories -->|JPA| database[("MySQL")]
    migrations["Flyway Migrations"] -->|create and evolve schema| database
    auth["Spring Security + JWT"] -. protects .-> controllers
    exception["API Exception Handler"] -. formats errors from .-> controllers
```

## Domain class diagram

```mermaid
classDiagram
    direction LR

    class AppUser {
        +Long id
        +String email
        +String passwordHash
        +String displayName
        +Role role
        +Instant createdAt
    }
    class Task {
        +Long id
        +String title
        +String description
        +TaskStatus status
        +TaskPriority priority
        +Instant dueAt
        +Instant createdAt
        +Instant updatedAt
        +long version
    }
    class TaskComment {
        +Long id
        +String body
        +Instant createdAt
    }
    class Role {
        <<enumeration>>
        USER
        ADMIN
    }
    class TaskStatus {
        <<enumeration>>
        OPEN
        IN_PROGRESS
        COMPLETED
        ARCHIVED
    }
    class TaskPriority {
        <<enumeration>>
        LOW
        MEDIUM
        HIGH
        URGENT
    }

    Task "0..*" --> "1" AppUser : createdBy
    Task "0..*" --> "0..1" AppUser : assignee
    TaskComment "0..*" --> "1" Task : task
    TaskComment "0..*" --> "1" AppUser : author
    AppUser ..> Role
    Task ..> TaskStatus
    Task ..> TaskPriority
```

A task has exactly one creator and may have one assignee. Each comment belongs to one task and has one author. The enum values and relationships match the JPA entities and the initial Flyway migration in the API project.

## Object diagram

This sample object snapshot shows a user creating a task, another user assigned to it, and a comment authored by the creator.

```mermaid
flowchart LR
    creator["creator : AppUser<br/>id = 10<br/>email = alex@example.com<br/>role = USER"]
    assignee["assignee : AppUser<br/>id = 12<br/>email = sam@example.com<br/>role = USER"]
    task["task : Task<br/>id = 44<br/>title = Prepare sprint plan<br/>status = IN_PROGRESS<br/>priority = HIGH"]
    comment["comment : TaskComment<br/>id = 80<br/>body = Milestones are outlined"]

    creator -->|createdBy| task
    assignee -->|assignee| task
    task -->|has comment| comment
    creator -->|author| comment
```
## Database ER diagram

```mermaid
erDiagram
    APP_USERS {
        BIGINT id PK
        VARCHAR email UK
        VARCHAR password_hash
        VARCHAR display_name
        VARCHAR role
        TIMESTAMP created_at
    }
    TASKS {
        BIGINT id PK
        VARCHAR title
        TEXT description
        VARCHAR status
        VARCHAR priority
        TIMESTAMP due_at
        TIMESTAMP created_at
        TIMESTAMP updated_at
        BIGINT version
        BIGINT created_by_id FK
        BIGINT assignee_id FK "nullable"
    }
    TASK_COMMENTS {
        BIGINT id PK
        BIGINT task_id FK
        BIGINT author_id FK
        VARCHAR body
        TIMESTAMP created_at
    }

    APP_USERS ||--o{ TASKS : creates
    APP_USERS o|--o{ TASKS : assigned_to
    TASKS ||--o{ TASK_COMMENTS : has
    APP_USERS ||--o{ TASK_COMMENTS : authors
```

`created_by_id`, `task_id`, and `author_id` are required foreign keys. `assignee_id` is nullable. Deleting a task cascades to its comments.

## Use case diagram

```mermaid
flowchart LR
    user["User actor"]
    admin["Admin actor"]
    subgraph system["TaskFlow API"]
        register([Register / log in])
        profile([View / update own profile])
        browse([Browse accessible tasks])
        create([Create task])
        edit([Update task / status])
        assign([Assign / delete own task])
        comment([Comment on accessible task])
        directory([Manage user directory])
        allTasks([Manage any task])
    end
    admin -. inherits user capabilities .-> user
    user --- register
    user --- profile
    user --- browse
    user --- create
    user --- edit
    user --- assign
    user --- comment
    admin --- directory
    admin --- allTasks
```

Task edits require creator, assignee, or admin access. Assignment and deletion require the creator or an admin. Admins can manage the user directory and all tasks.

## Authentication sequence

```mermaid
sequenceDiagram
    actor Client
    participant Controller as AuthController
    participant Service as AuthService
    participant Users as UserRepository
    participant DB as MySQL
    participant Encoder as PasswordEncoder
    participant JWT as JwtEncoder
    participant Errors as API Exception Handler

    alt Register
        Client->>Controller: POST /api/auth/register
        Controller->>Service: register(request)
        Service->>Users: existsByEmail(normalized email)
        Users->>DB: check email
        DB-->>Users: result
        Users-->>Service: exists / absent
        alt Email already exists
            Service-->>Errors: Conflict
            Errors-->>Client: 409 Conflict
        else New account
            Service->>Encoder: encode(password)
            Encoder-->>Service: password hash
            Service->>Users: save(user with USER role)
            Users->>DB: INSERT app_users
            DB-->>Users: saved user
            Users-->>Service: optional user
            Service->>JWT: sign HS256 token
            JWT-->>Service: access token
            Service-->>Controller: AuthResponse
            Controller-->>Client: 201 Created + token
        end
    else Log in
        Client->>Controller: POST /api/auth/login
        Controller->>Service: login(request)
        Service->>Users: findByEmail(normalized email)
        Users->>DB: SELECT app_users
        DB-->>Users: user or absent
        Users-->>Service: optional user
        Service->>Encoder: check password if user found
        Encoder-->>Service: match result
        alt Credentials valid
            Service->>JWT: sign HS256 token
            JWT-->>Service: access token
            Service-->>Controller: AuthResponse
            Controller-->>Client: 200 OK + token
        else Email missing / invalid password
            Service-->>Errors: Unauthorized
            Errors-->>Client: 401 Unauthorized
        end
    end
```

## Task request activity

```mermaid
flowchart TD
    start([Request task endpoint]) --> bearer["Send request with bearer token"]
    bearer --> verify{"JWT valid?"}
    verify -->|No| unauthorized["Return 401 Unauthorized"]
    verify -->|Yes| controller["TaskController validates the request (and body when present)"]
    controller --> service["TaskService checks visibility and operation permissions"]
    service --> allowed{"Operation allowed?"}
    allowed -->|No| denied["Return API access error"]
    allowed -->|Yes| data["Read or write through repositories"]
    data --> db[("MySQL")]
    db --> response["Return JSON response"]
    unauthorized --> done([End])
    denied --> done
    response --> done
```

Task routes require a valid bearer token. The service applies task visibility and creator, assignee, or admin rules before performing protected operations. Repository reads and writes depend on the selected endpoint.
