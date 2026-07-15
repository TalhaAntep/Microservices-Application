# Microservices Application

An ASP.NET MVC and Web API application that demonstrates service decomposition, API communication, user-scoped data access, asynchronous forwarding, and parallel batch processing through a DevExtreme data-grid interface.

## Overview

The solution manages geometric shape records and calculates their areas through separate application layers:

1. **MVC Web Application** provides authentication and the DevExtreme user interface.
2. **Shape Forward API** acts as a gateway between the web client and processing services.
3. **Shape Processing API** owns persistence and calculation logic.
4. Multiple processing instances can run in parallel, allowing the forwarder to distribute queued batches between workers.

The project is built as a practical microservices architecture example on .NET Framework 4.8. It demonstrates how a browser-facing MVC application can communicate with independent HTTP services instead of containing all business logic in one process.

## Key Features

- User registration and sign-in with ASP.NET Identity and OWIN cookie authentication.
- Authenticated shape-management screen backed by a DevExtreme DataGrid.
- Create, read, update, and delete operations over HTTP APIs.
- Per-user record isolation through `UserId` filtering.
- Support for squares, rectangles, and triangles.
- Validation for required fields and numeric dimensions.
- Asynchronous communication between the forwarding and processing APIs.
- Batch creation for uncalculated records.
- Queue-based distribution across two processing-service instances.
- Area calculation with completion status and persisted results.
- API-instance identification through the `X-Api-Name` header.
- SQL Server persistence through Entity Framework 6.
- Database logging for processing activity.

## Architecture

```text
+---------------------------+
| ASP.NET MVC Web App       |
| Identity + DevExtreme UI  |
+-------------+-------------+
              |
              | HTTP / JSON
              v
+---------------------------+
| Shape Forward API         |
| Gateway + batch queue     |
+-----------+---------------+
            |
      +-----+-----+
      |           |
      v           v
+-----------+ +-----------+
| Processor | | Processor |
| Instance 1| | Instance 2|
| Port 81   | | Port 82   |
+-----+-----+ +-----+-----+
      |             |
      +------+------+ 
             |
             v
+---------------------------+
| SQL Server                |
| Shapes, users, and logs   |
+---------------------------+
```

## Solution Components

### DevExtremeMvcApp2

The presentation layer contains:

- ASP.NET MVC views and controllers.
- ASP.NET Identity registration and login flows.
- A DevExtreme DataGrid for editing shape records.
- Client-side validation and shape-specific behavior.
- AJAX calls to the forwarding API.
- Entity Framework migrations and shared data models.

The shape page expects the forwarding service at `http://localhost:5001`.

### ShapeForwardAPI

The gateway service exposes `api/shape-forward` and:

- Forwards CRUD requests to the processing service.
- Retrieves uncalculated records.
- Groups records into small batches.
- Maintains an in-memory queue.
- Dispatches work to two processing-service instances.
- Tracks worker availability before assigning the next batch.

The current configuration expects processing instances at:

- `http://localhost:81/api/shape-processing`
- `http://localhost:82/api/shape-processing`

### ShapeProcessingAPI

The processing service:

- Validates incoming records and user identifiers.
- Persists shape data with Entity Framework.
- Restricts reads and mutations to the owning user.
- Calculates square, rectangle, and triangle areas.
- Marks completed records as calculated.
- Records the worker name and processing details in SQL Server logs.

### Shape.Common

Contains shared model definitions intended for reuse between solution components.

## API Summary

### Forwarding API

- `GET /api/shape-forward?userId={id}` - list a user's shapes.
- `POST /api/shape-forward` - create a shape.
- `PUT /api/shape-forward/{id}?userId={id}` - update a shape.
- `DELETE /api/shape-forward/{id}?userId={id}` - delete a shape.
- `POST /api/shape-forward/calculate?userId={id}` - queue uncalculated shapes.

### Processing API

- `GET /api/shape-processing?userId={id}` - query persisted shapes.
- `POST /api/shape-processing` - validate and persist a shape.
- `PUT /api/shape-processing/{id}?userId={id}` - update a user-owned shape.
- `DELETE /api/shape-processing/{id}?userId={id}` - remove a user-owned shape.
- `POST /api/shape-processing/calculate-batch?userId={id}` - process a batch.

## Tech Stack

- C# and .NET Framework 4.8
- ASP.NET MVC 5
- ASP.NET Web API 2
- Entity Framework 6
- ASP.NET Identity and OWIN
- DevExtreme / DevExpress 25.1
- jQuery and AJAX
- SQL Server
- Newtonsoft.Json
- Visual Studio 2022 solution format

## Project Structure

```text
Microservices-Application/
|-- DevExtremeMvcApp2.sln
|-- DevExtremeMvcApp2/       # MVC UI, Identity, migrations, and models
|-- ShapeForwardAPI/         # Gateway, forwarding, and batch distribution
|-- ShapeProcessingAPI/      # Persistence, calculations, and logging
|-- Shape.Common/            # Shared contracts
`-- lib/DevExtreme/          # DevExtreme MVC integration assemblies
```

## Prerequisites

- Windows with Visual Studio 2022
- .NET Framework 4.8 Developer Pack
- SQL Server or SQL Server Express
- IIS Express
- NuGet package restore
- A compatible DevExpress / DevExtreme 25.1 installation or license where required

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/TalhaAntep/Microservices-Application.git
cd Microservices-Application
```

Open `DevExtremeMvcApp2.sln` in Visual Studio.

### 2. Restore dependencies

Restore NuGet packages for all solution projects. Confirm that the DevExtreme MVC assemblies are available before building.

### 3. Configure SQL Server

Update each `DefaultConnection` entry in the relevant `Web.config` files so the MVC and processing applications use the same SQL Server database.

The processing logger also requires a valid connection to the logging database and a `log_Table` table with fields for API name, user ID, message, and level.

Do not commit real database passwords. Move connection details to protected configuration or environment-specific settings before deployment.

### 4. Apply migrations

Select the MVC project as the migrations project and apply the Entity Framework migrations to create the Identity and shape tables.

```powershell
Update-Database
```

### 5. Configure local ports

Run the services with the URLs expected by the current source code:

- Shape Forward API: `http://localhost:5001`
- Shape Processing API instance 1: `http://localhost:81`
- Shape Processing API instance 2: `http://localhost:82`

A second processing instance can be launched with a separate IIS Express profile or IIS application that points to the same code and database.

### 6. Run the solution

Start the forwarding API, both processing instances, and the MVC application. Register a user, open the shape grid, add records, and choose **Calculate** to distribute pending calculations across the workers.

## Calculation Rules

- Square: width multiplied by width; height is synchronized with width.
- Rectangle: width multiplied by height.
- Triangle: width multiplied by height, divided by two.

Processed rows are marked with `IsCalculated` and store the result in `CalculationResult`.

## Current Design Notes

- The batch queue and worker status are stored in memory and reset when the forwarding process restarts.
- Service URLs are currently configured in source code for local development.
- Both processing instances share the same database.
- Authentication is handled by the MVC application; service-to-service authentication can be added for production use.
- The current project is an educational microservices demonstration and requires production hardening before public deployment.

## Recommended Improvements

- Move service URLs and connection strings into environment-specific configuration.
- Add service authentication and authorization.
- Replace the in-memory queue with RabbitMQ, Azure Service Bus, or another durable broker.
- Add health checks, retries, timeouts, and circuit breakers.
- Use dependency injection for HTTP clients and database contexts.
- Add automated unit, integration, and load-balancing tests.
- Containerize services for repeatable multi-instance deployment.
- Introduce centralized structured logging and distributed tracing.

## License

This project is intended for educational and portfolio use.