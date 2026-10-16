RaceDay — Event Management System 

Overview 

RaceDay is a full-stack, web-based event management system built for South Africa's road running, walking, and cycling community. It replaces the paper-based registration, spreadsheets, and disconnected communication that many local race organisers currently rely on. 

The platform allows Event Organisers to create and manage events, define entry categories, and capture participant results. Participants can browse upcoming events, enter events by selecting a category, track their personal performance history, and prepare for race day using live weather and route information. 

This repository contains Part 1 of the RaceDay Portfolio of Evidence: system planning, including the database design (ERD), the API endpoint plan, and the SQL database script. No application code is written in this part — Parts 2 (RESTful API) and 3 (MVC web application) build on top of this foundation. 

User Roles 

RaceDay supports two distinct user roles: 

Organiser — can create, edit, and delete events; manage event categories; capture participant results; and view all enrolments for their own events. 

Participant — can create an account, browse events, enter an event by selecting a category, view their own enrolments, and track their own results. 

Role-based access will be enforced at the API level in Part 2 and reflected consistently in the MVC interface in Part 3. 

Repository Structure 

├── .github/ 
│   └── workflows/ 
│       └── validate-repo-structure.yml   # CI/CD: validates required files exist 
├── docs/ 
│   ├── RaceDay_ERD.png                   # Entity Relationship Diagram 
│   ├── RaceDay_API_Endpoint_Plan.md      # Full API endpoint plan 
│   └── RaceDay_Database.sql              # Database creation + seed script 
└── README.md 

Part 1 Deliverables 

1. Entity Relationship Diagram 

Located at docs/RaceDay_ERD.png. Models 6 entities: Roles, Users, Events, Categories, Enrolments, and Results, with primary keys, foreign keys, and cardinality shown for every relationship. 

A few deliberate design decisions worth noting: 

Users is a single table with a RoleId foreign key (rather than separate Organiser/Participant tables), since both roles share identical attributes — the difference in permissions is enforced at the application layer, not the schema layer. 

Events includes Latitude/Longitude (to support the live weather feature) and RouteMapUrl (to support Azure Blob Storage integration in Part 3), even though those features aren't built until later parts — this avoids schema changes further into the project. 

Categories supports both age-based (e.g. "Under 20", "Senior") and distance-based (e.g. "10km", "21km") categories via a CategoryType field, with Distance or MinAge/MaxAge populated accordingly. This is enforced with a CHECK constraint in the SQL script. 

Enrolments enforces one enrolment per participant per category via a UNIQUE(ParticipantId, CategoryId) constraint. 

Results has a Status field (Finished/DNF/DNS/DQ) and nullable FinishTime/Position, since not every participant who enrols finishes the race. 

2. API Endpoint Plan 

Located at docs/RaceDay_API_Endpoint_Plan.md. Covers all required areas — Authentication, User Profile, Events, Categories, Event Enrolments, and Results — with HTTP method, route, description, required role, request body, and expected response (including failure cases) specified for every endpoint. 

3. SQL Database Script 

Located at docs/RaceDay_Database.sql. 

Setup Instructions — Running the Database Script 

Open SQL Server Management Studio (SSMS) and connect to your local or remote SQL Server instance. 

Open a new query window. 

Open docs/RaceDay_Database.sql and execute it. 

The script will:  

Create a new RaceDay database (if one doesn't already exist) 

Create all 6 tables with primary keys, foreign keys, and constraints 

Seed the database with sample data: 2 Organisers, 2 Participants, 3 Events, 6 Categories, 4 Enrolments, and 3 Results 

To verify, expand Databases → RaceDay → Tables in Object Explorer, or run: 

sql 

  SELECT * FROM RaceDay.dbo.Events; 

No additional setup (no application code, no dependencies) is required for Part 1, since this part is planning and database design only. 

CI/CD 

A GitHub Actions workflow (.github/workflows/validate-repo-structure.yml) runs automatically on every push to main. It validates that the /docs folder exists and contains the ERD, API endpoint plan, and SQL script, and that a README is present. 

Video Walkthrough 

An unlisted YouTube video walking through the planning documents, ERD design decisions, endpoint plan choices, and a live run of the SQL script in SSMS: 

https://youtu.be/yRt3eMqK7yQ - YouTube Link 







------------------------------------------------------------------------------------------------------------------------------------------------------------






## Part 2 — RESTful API

### Repository Structure (updated)

```
├── .github/
│   └── workflows/
│       └── build-and-test.yml            # Part 2 CI: builds the API and runs the unit tests
├── RaceDay.Api/                          # ASP.NET Core Web API (Part 2)
│   ├── Controllers/                      # Auth, Users, Events, Categories, Enrolments, Results
│   ├── Models/                           # EF Core entities (match the Part 1 ERD exactly)
│   ├── Data/                             # DbContext, Code-First configuration, seed data
│   ├── DTOs/                             # Request/response shapes per the Part 1 endpoint plan
│   ├── Auth/                             # Session-based authentication + role enforcement
│   ├── Services/                         # Weather service (Open-Meteo integration)
│   └── Migrations/                       # EF Core migration that builds the database
├── RaceDay.Api.Tests/                    # xUnit test project
├── RaceDay.sln
├── RaceDay ERD.png                       # Part 1: Entity Relationship Diagram
├── RaceDay_API_Endpoint_Plan.md          # Part 1: API endpoint plan
├── RaceDay_Database (2).sql              # Part 1: database creation + seed script
└── README.md
```

### What Part 2 Implements

A RESTful API built with ASP.NET Core Web API (C#), following the endpoint plan approved in Part 1 exactly. Every route, role, and failure case from `RaceDay_API_Endpoint_Plan.md` is implemented here (19 endpoints in total).

**Database:** Entity Framework Core, Code-First. The model classes in `RaceDay.Api/Models/` and the Fluent API configuration in `RaceDay.Api/Data/RaceDayDbContext.cs` reproduce the Part 1 ERD and SQL script field-for-field, including the `CHECK` constraints (event type, category type/age/distance combination) and `UNIQUE` constraints (one email per user, one enrolment per participant per category, one result per enrolment).

**Authentication:** Session-based, not token-based. On login, the server stores `UserId` and `Role` in a server-side session (`RaceDay.Api/Auth/RequireSessionAttribute.cs`), and every subsequent request is authenticated and authorised from that session. Passwords are hashed with BCrypt and never stored in plain text.

**Role enforcement:** A custom `[RequireSession]` authorisation attribute enforces both "must be logged in" (401 if not) and "must be the right role" (403 if wrong role) on every protected endpoint. For example, only the Organiser who created an event can edit or delete it, and only a Participant can enrol in a category.

**Weather integration:** `GET /api/events/{id}/weather` calls the free Open-Meteo API using the event's `Latitude`/`Longitude` from the Part 1 ERD.

**API documentation:** Every endpoint is documented in Swagger UI, with summaries, request bodies, and response codes.

### Setup Instructions

**Prerequisites:** .NET 8 SDK, and SQL Server (or SQL Server Express/LocalDB).

1. Clone the repository and restore dependencies:
   ```
   dotnet restore RaceDay.sln
   ```

2. Update the connection string in `RaceDay.Api/appsettings.json` (and `appsettings.Development.json`) if your SQL Server instance isn't on `localhost` with Windows authentication. The API uses its own database, `RaceDayApi`, so it does not clash with the Part 1 `RaceDay` database.

3. Run the API:
   ```
   dotnet run --project RaceDay.Api
   ```
   The EF Core migration is included in `RaceDay.Api/Migrations`. On first run the database is created automatically and seeded with the Part 1 sample data: 2 Organisers, 2 Participants, 3 Events, 6 Categories, 4 Enrolments, 3 Results.

4. Open Swagger UI at `http://localhost:5000/swagger` (or `https://localhost:5001/swagger`) to view and test every endpoint in the browser.

   **Seeded test accounts** (password for all: `Password123!`):
   - Organiser: `thabo.nkosi@raceday.co.za`
   - Participant: `sarah.vdm@example.com`

### Running the Tests

```
dotnet test RaceDay.Api.Tests/RaceDay.Api.Tests.csproj
```

23 tests across 4 test classes, covering both success and failure scenarios for each required area:
- `AuthTests`: registration and login (valid/invalid roles, duplicate email, wrong password, protected routes without a session)
- `EventManagementTests`: Organiser create/update/delete, ownership enforcement, invalid event types
- `RoleEnforcementTests`: Participants rejected from Organiser-only actions and vice versa, anonymous requests rejected
- `EnrolmentTests`: correct Participant + Category + Event linkage, duplicate enrolment rejection, full-category rejection, Organiser visibility into their event's enrolments

Tests run against the real HTTP pipeline (via `WebApplicationFactory`) backed by an isolated EF Core in-memory database per test run, so they exercise the actual controllers, routing, and session middleware, not mocks.

### CI/CD

A GitHub Actions workflow (`.github/workflows/build-and-test.yml`) runs on every push to `main`. It restores, builds the full solution, and runs all 23 unit tests. A failing test fails the build.

**Successful build:**

<!-- Paste your screenshot of the green Part 2 GitHub Actions build here -->

### Video Walkthrough

An unlisted video demonstrating the running API, the code structure, the endpoints, authentication, and the unit tests:

**[Video link here]**
