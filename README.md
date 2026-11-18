# TaskFlow UML

UML documentation for the [TaskFlow API](https://github.com/Utkarsh-tiwari-1/taskflow-api), a Spring Boot task-management REST API.

The diagrams below describe the current application layers and core JPA domain model. GitHub renders the Mermaid diagrams in this README. Editable PlantUML sources are in [`TaskFlow-Architecture.puml`](TaskFlow-Architecture.puml) and [`TaskFlow-Domain.puml`](TaskFlow-Domain.puml).

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
    auth -. token validation .-> services
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
