# ClinicalFlow

## Project Description
ClinicalFlow is a RESTful ASP.NET Core Web API for managing patient accounts, doctor accounts, patient records, clinical encounters, and multi-medication prescriptions. It uses JWT bearer authentication, role-based authorization, DTO validation, Entity Framework Core, SQL Server, password hashing, transactional persistence, and centralized exception handling.

## Overview
ClinicalFlow supports a basic clinical workflow: authenticated users manage patient records, create encounters linked to patients and doctors, document and update encounters while in progress, complete encounters, and create prescriptions containing one or more medication entries.

This is a backend project/reference implementation, not a certified electronic health record or clinical decision-support system. Use synthetic data in development and demos.

## Features
- Patient creation, listing, retrieval, and updates.
- Patient accounts backed by ApplicationUser with email/password authentication and hashed passwords.
- Admin-only doctor provisioning with an ApplicationUser account and Doctor profile.
- Admin-only doctor listing.
- Encounter creation, retrieval, patient-based listing, updating, and completion.
- Encounter lifecycle rules: encounters start in progress; updates and completion are restricted to in-progress encounters.
- Prescription creation with multiple medication items and retrieval by encounter.
- Transactional persistence for prescription and medication records.
- JWT login and bearer-token protection for patient, encounter, prescription, and Admin endpoints.
- Role-based access for Admin, Doctor, Nurse, and Patient roles.
- DTO-based request validation and centralized exception handling.
- EF Core persistence using Microsoft SQL Server.
- Optional development-only seed user with hashed password storage.

## Technology Stack
- .NET 10 / ASP.NET Core Web API
- C#
- Entity Framework Core 10
- Microsoft SQL Server
- JWT Bearer authentication
- ASP.NET Core password hashing
- Postman collection for manual API exploration

## Repository Structure
- `ClinicalFlow/Controllers` — HTTP endpoints and response mapping
- `ClinicalFlow/Configuration` — typed configuration
- `ClinicalFlow/Data` — EF Core context and persistence setup
- `ClinicalFlow/Dtos` and `ClinicalFlow/DTOs` — request/response contracts
- `ClinicalFlow/Interfaces` — service contracts
- `ClinicalFlow/Middleware` — centralized exception handling
- `ClinicalFlow/Models` — application and persistence entities
- `ClinicalFlow/Services` — application/business logic
- `ClinicalFlow/Migrations` — EF Core migrations and model snapshot
- `ClinicalFlow/Program.cs` — dependency injection, authentication, and HTTP pipeline
- `Clinical-WorkFlow.postman_collection.json` — Postman request collection

## Prerequisites
- .NET 10 SDK
- Microsoft SQL Server (local or reachable instance)
- Git
- Optional: Postman
- Optional: EF Core CLI (`dotnet-ef`) for migrations

Check the SDK installation:

```bash
dotnet --version
```

## Local Setup

### 1. Clone the repository

```bash
git clone https://github.com/chauhan12harsh/ClinicalFlow.git
cd ClinicalFlow/ClinicalFlow
```

### 2. Configure the database

The API expects a connection string named `ConnectionStrings:DefaultConnection`. Use User Secrets for local development instead of committing credentials:

```bash
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=ClinicalFlowDb;Trusted_Connection=True;TrustServerCertificate=True;"
```

Adjust the connection string for your SQL Server installation. If using SQL authentication, keep the username and password in a secure local secret store.

### 3. Configure JWT

The application requires `Jwt:Issuer`, `Jwt:Audience`, and `Jwt:SecretKey`. The signing key must be at least 32 characters long.

```bash
dotnet user-secrets set "Jwt:Issuer" "ClinicalFlow"
dotnet user-secrets set "Jwt:Audience" "ClinicalFlowClient"
dotnet user-secrets set "Jwt:SecretKey" "replace-with-a-long-random-secret-key-of-at-least-32-characters"
```

Use a cryptographically random secret. Never commit signing keys or production secrets.

### 4. Optional development seed user

In the Development environment, `Program.cs` can create a Doctor seed account when email, password, and full name are configured. The password is hashed before persistence.

```bash
dotnet user-secrets set "DevelopmentSeed:Email" "doctor@example.com"
dotnet user-secrets set "DevelopmentSeed:Password" "use-a-strong-local-password"
dotnet user-secrets set "DevelopmentSeed:FullName" "Development Doctor"
```

The current seed creates the account with the `Doctor` role; `DevelopmentSeed:Role` is not read by the current implementation.

This is for local development only and is not a production user-provisioning mechanism.

### 5. Restore, build, and apply migrations

From the `ClinicalFlow` directory:

```bash
dotnet restore
dotnet build
```

If EF Core migrations are present in the repository, apply them:

```bash
dotnet ef database update
```

If no initial migration exists, create one only after checking the current model and migration history:

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

Do not create a duplicate initial migration.

### 6. Run the API

```bash
dotnet run
```

Use the local address printed by ASP.NET Core. HTTPS redirection is enabled; trust the development certificate if prompted.

## Authentication
`POST /api/auth/login` is the public login endpoint. Submit the configured email and password, then send the returned JWT with protected requests:

```http
Authorization: Bearer <access-token>
```

JWT validation checks issuer, audience, signature, and token lifetime. Patient, encounter, and prescription controller routes require authentication.

## API Reference
All endpoints are relative to the API base URL. Except login, the routes below require a valid bearer token.

### Authentication

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/login` | Authenticate and receive an access token |

### Patients

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/patients` | Create a patient |
| GET | `/api/patients` | List patients |
| GET | `/api/patients/{id}` | Get a patient by ID |
| PUT | `/api/patients/{id}` | Update patient details |

Patient DTOs validate required names and MRN, applicable length limits, and email/phone formats. Patient creation also requires a password because a Patient ApplicationUser account is created. MRN is not included in the update request.

### Admin / Doctors

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/admin/createdoctor` | Admin-only doctor + ApplicationUser creation |
| GET | `/api/admin/doctors` | Admin-only doctor listing |

### Encounters

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/encounters` | Create an encounter |
| GET | `/api/encounters/{id}` | Get an encounter by ID |
| GET | `/api/encounters/patient/{patientId}` | List encounters for a patient |
| PUT | `/api/encounters/{id}` | Update an in-progress encounter |
| PATCH | `/api/encounters/{id}/complete` | Mark an encounter complete |

Encounter creation references patient and doctor IDs and includes a chief complaint. Encounter updates and completion are limited to encounters that are in progress.

### Prescriptions

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/encounters/{encounterId}/prescriptions` | Create a prescription with medication entries |
| GET | `/api/encounters/{encounterId}/prescriptions` | Get prescriptions for an encounter |

Medication entries include medication name, dosage, frequency, duration, quantity, and optional instructions. The service requires an existing in-progress encounter and saves the prescription and medication items transactionally.

## Typical HTTP Responses
- `200 OK` — successful retrieval or update.
- `201 Created` — successful resource creation.
- `400 Bad Request` — invalid request data; automatic model validation is enabled.
- `401 Unauthorized` — missing/invalid token or invalid login credentials.
- `404 Not Found` — requested resource or parent encounter was not found.
- Other exceptions are routed through centralized exception middleware. Exact response payloads are defined by the implementation.

## Postman
Import `Clinical-WorkFlow.postman_collection.json` into Postman. Set the `base_url` collection/environment variable to your local API origin (for example, `https://localhost:<port>`). Log in and use the returned bearer token for protected requests. Replace placeholder request bodies and IDs with values from your API instance. Do not commit credentials or real patient data to the collection.

## Validation and Testing
Build the project using:

```bash
dotnet build
```

Run automated tests with `dotnet test` if a test project is available. The Postman collection can be used for manual endpoint verification. A comprehensive verification pass should cover valid and invalid DTO inputs, unauthenticated access, missing IDs, encounter state transitions, and prescriptions containing multiple medication entries.

## Security and Data Handling
- Do not commit database passwords, JWT secrets, or production credentials.
- Use User Secrets locally and a managed secret store/environment configuration in deployed environments.
- Use HTTPS outside isolated local development.
- Use synthetic data for development and testing; apply applicable privacy, access-control, retention, and security requirements before handling clinical data.
- The current controllers enforce authentication and role-based authorization. Encounter creation currently accepts `DoctorId` from the request; deriving the Doctor from the authenticated JWT identity is a planned authorization hardening step. Review service/domain-level authorization and deployment security before production use.
- Plan database backups, access controls, monitoring, logging, and secret rotation for deployments.

## Future Improvements
Potential follow-up work includes deriving DoctorId from the authenticated user, making ApplicationUser.Email the single source of truth instead of duplicating email on Doctor/Patient, expanded automated integration tests, richer OpenAPI/Swagger documentation, pagination/filtering, audit logging, and deployment-specific observability/configuration.

## License
No license is currently specified in this repository. Unless a license is added, reuse and redistribution remain subject to the repository owner's rights.
