# EventFlow

EventFlow is a backend-only ASP.NET Core Web API for managing free, in-person
events and attendee registrations. It gives organizers a way to publish and
manage events while allowing attendees to discover eligible events, register,
track registration status, and cancel when permitted.

The project is designed as a database-first API demonstration. It uses seeded
demo identities rather than production authentication, and Swagger/OpenAPI is
the primary way to explore and verify the API.

## Contents

- [What it solves](#what-it-solves)
- [Features](#features)
- [How it works](#how-it-works)
- [Technology stack](#technology-stack)
- [Project structure](#project-structure)
- [Getting started](#getting-started)
- [Using the API](#using-the-api)
- [Database model](#database-model)
- [Testing](#testing)
- [Status and limitations](#status-and-limitations)
- [Documentation](#documentation)

## What it solves

Event registration commonly requires more than storing an attendee and an
event. Capacity, approval workflows, ownership, scheduling conflicts, event
state changes, and registration history all need consistent rules. EventFlow
implements those rules behind a REST API so clients can interact with a
single, predictable service boundary.

The intended users are:

- **Attendees**, who browse public events and manage their own registrations.
- **Organizers**, who create and manage their own events and process attendee
  registration requests.
- **Developers and reviewers**, who need a small, testable example of an
  ASP.NET Core API with EF Core, validation, authorization, and Swagger.

## Features

### Event discovery and management

- Search public, active events by optional title.
- Retrieve event details.
- Create and update events as an organizer.
- Enforce venue capacity and prevent overlapping active events at a venue.
- Close, postpone, reschedule, or cancel owned events.
- Preserve event and registration history through status transitions.

### Registration workflows

- Register as an attendee for an eligible event.
- Support both approval-free and approval-required events.
- Enforce duplicate-registration and capacity rules.
- List the current user's pending and confirmed registrations.
- Cancel a pending or confirmed registration as its attendee.
- Let organizers view guest lists and approve, reject, or cancel registrations.
- Generate confirmation references for confirmed registrations.

### API and operational behavior

- Validate request DTOs at the API boundary.
- Return consistent Problem Details responses for authorization and business
  rule failures.
- Resolve seeded demo identities from the `X-Demo-Identity` header.
- Provide an API/database connectivity health endpoint.
- Expose Swagger UI in the Development environment.
- Log notification events through a replaceable notification-service interface.

## How it works

1. A client calls the API directly or through Swagger UI.
2. Public event discovery does not require an identity. Protected operations
   resolve a seeded user from `X-Demo-Identity`.
3. Controllers authorize the caller and pass validated DTOs to the service
   layer.
4. `EventService` and `RegistrationService` enforce ownership, lifecycle,
   capacity, scheduling, and registration rules.
5. Entity Framework Core persists data in SQL Server.
6. Notification events are currently written to the application log through
   `LoggingNotificationService`.

The production database schema is created by
[`database/001_CreateEventFlowDatabase.sql`](database/001_CreateEventFlowDatabase.sql).
It is intentionally non-destructive and includes seeded demo users and venues.

## Technology stack

| Area | Technology |
| --- | --- |
| Language and runtime | C#, .NET 8 |
| Web framework | ASP.NET Core Web API |
| Data access | Entity Framework Core 8 |
| Production database | Microsoft SQL Server, configured for LocalDB by default |
| Test database | SQLite in-memory |
| API documentation | Swashbuckle.AspNetCore / Swagger UI |
| Object mapping | AutoMapper |
| Automated tests | xUnit, Moq, Microsoft.NET.Test.Sdk, Coverlet |
| Database tooling | `dotnet-ef` 8.0.31 |

## Project structure

```text
.
├── database/
│   └── 001_CreateEventFlowDatabase.sql   # SQL Server schema and demo seed
├── docs/                                  # Design, verification, and phase docs
├── src/
│   └── EventFlow.Api/
│       ├── Auth/                          # Demo identity and authorization
│       ├── Controllers/                   # HTTP API endpoints
│       ├── Data/                          # EF Core DbContext
│       ├── Dtos/                          # Request and response contracts
│       ├── Mapping/                       # AutoMapper profiles
│       ├── Models/                        # Database-backed entities
│       └── Services/                      # Event, registration, and notification logic
├── tests/
│   └── EventFlow.Api.Tests/               # Isolated service and contract tests
├── EventFlow.sln
└── dotnet-tools.json
```

## Getting started

### Prerequisites

- .NET 8 SDK
- Microsoft SQL Server with permission to create/use `EventFlowDb`
- SQL Server LocalDB on Windows, or another SQL Server instance configured in
  `src/EventFlow.Api/appsettings.json`

### 1. Restore tools and dependencies

From the repository root:

```bash
dotnet tool restore
dotnet restore
```

### 2. Create the database

Run [`database/001_CreateEventFlowDatabase.sql`](database/001_CreateEventFlowDatabase.sql)
against SQL Server. The default application connection string targets:

```text
(localdb)\MSSQLLocalDB
```

To use another SQL Server instance, update the `EventFlowDb` connection string
in `src/EventFlow.Api/appsettings.json` or provide an equivalent environment
configuration value. Do not commit credentials or secrets to configuration
files.

### 3. Build and run the API

```bash
dotnet build EventFlow.sln
dotnet run --project src/EventFlow.Api/EventFlow.Api.csproj --urls http://localhost:5200
```

With the Development environment enabled, open
[`http://localhost:5200/swagger`](http://localhost:5200/swagger) to use Swagger
UI. The checked-in launch profile already uses this URL and enables the
Development environment.

### 4. Check connectivity

```bash
curl http://localhost:5200/api/health
```

A healthy response reports both API availability and a connected
`EventFlowDb` database.

## Using the API

### Demo identities

This project intentionally uses a development-only identity mechanism. Add one
of these headers to protected requests:

| Header | Role |
| --- | --- |
| `X-Demo-Identity: demo-attendee` | Attendee |
| `X-Demo-Identity: demo-organizer` | Organizer |

For example:

```bash
curl -H "X-Demo-Identity: demo-attendee" \
  http://localhost:5200/api/identity/me
```

### Endpoint overview

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Check API and database connectivity |
| `GET` | `/api/identity/me` | Resolve the current demo identity |
| `GET` | `/api/events` | Search public active events |
| `GET` | `/api/events/{eventId}` | View a public event |
| `POST` | `/api/events` | Create an event as an organizer |
| `GET` | `/api/events/owned/{eventId}` | View an owned event |
| `PUT` | `/api/events/{eventId}` | Update an owned active event |
| `POST` | `/api/events/{eventId}/close` | Close an owned event |
| `POST` | `/api/events/{eventId}/postpone` | Postpone an owned event |
| `POST` | `/api/events/{eventId}/reschedule` | Reschedule a postponed event |
| `POST` | `/api/events/{eventId}/cancel` | Cancel an owned event |
| `POST` | `/api/events/{eventId}/registrations` | Register as an attendee |
| `GET` | `/api/registrations` | List the current user's registrations |
| `POST` | `/api/registrations/{registrationId}/cancel` | Cancel an attendee registration |
| `GET` | `/api/events/{eventId}/registrations` | View an owned event's guest list |
| `POST` | `/api/registrations/{registrationId}/approve` | Approve a pending registration |
| `POST` | `/api/registrations/{registrationId}/reject` | Reject a pending registration |
| `POST` | `/api/registrations/{registrationId}/organizer-cancel` | Cancel a registration as organizer |

Request and response schemas, validation rules, and status codes are available
in Swagger after the API starts.

## Database model

The database contains four primary entities:

- `User` — seeded attendee and organizer identities.
- `Venue` — event locations and venue capacity.
- `Event` — schedule, visibility, capacity, approval mode, and lifecycle status.
- `Registration` — attendee/event relationship, status, confirmation reference,
  and decision reason.

The schema uses foreign keys, unique constraints, indexes, and controlled status
values. Application logic additionally enforces venue schedule overlap and
atomic registration-capacity checks. See the
[ERD and integrity rules](docs/EventFlow_ERD.md) for the verified relationships
and constraints.

## Testing

Run the automated suite from the repository root:

```bash
dotnet test EventFlow.sln
```

The tests use isolated SQLite in-memory databases, so the automated service
tests do not require SQL Server or a running API process. They cover request
validation, approval-required registration behavior, notification behavior,
duplicate registration handling, and postponed-event rescheduling.

Manual Swagger verification is documented in
[`docs/Swagger_Manual_Test_Plan.md`](docs/Swagger_Manual_Test_Plan.md).

## Status and limitations

EventFlow is an implemented academic/demo backend with automated service tests
and documented Swagger verification scenarios. It is suitable for demonstrating
API architecture and business-rule implementation, but it is not presented as
a production-ready event platform.

- Authentication is not production authentication. The `X-Demo-Identity`
  header resolves seeded users and does not provide passwords, tokens, account
  recovery, or identity verification.
- Notifications are logged locally; no external email or messaging provider is
  integrated.
- There is no frontend application in this repository.
- The default database setup targets SQL Server LocalDB and requires a separate
  database setup step.

## Documentation

- [Final system planning paper](docs/EventFlow_Final_System_Planning_Paper.md)
- [Database setup](docs/Phase1_Database_Setup.md)
- [API and EF Core setup](docs/Phase2_Api_EFCore_Setup.md)
- [Demo identity and authorization](docs/Phase4_Demo_Identity_Authorization.md)
- [Event services and endpoints](docs/Phase5_Event_Services_Endpoints.md)
- [Registration services](docs/Phase6_Registration_Services.md)
- [Organizer registration management](docs/Phase7_Organizer_Registration_Management.md)
- [Event lifecycle](docs/Phase8_Event_Lifecycle.md)
- [Notifications](docs/Phase9_Notifications.md)
- [Swagger verification](docs/Phase10_Verification_Swagger.md)
- [Automated tests](docs/Phase11_Automated_Tests.md)

## License

No license file or license declaration is included in the repository.
