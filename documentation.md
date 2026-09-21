# User — Technical Documentation

> Implements a user account microservice providing login, registration, customer, address, and card management over HTTP with a MongoDB-backed persistence layer.

*Generated from `microservices-demo/user` at `e1a79e778607` — 27 source files, 246 symbols, 1220 relationships analysed.*

---

## Contents

- [Project Overview](#project-overview)
- [How the System Works](#how-the-system-works)
- [Technology Stack](#technology-stack)
- [Repository Structure](#repository-structure)
- [Architecture](#architecture)
- [Components](#components)
- [Execution Flows](#execution-flows)
- [Business Logic](#business-logic)
- [Build, Tooling & CI](#build-tooling--ci)
- [File Reference](#file-reference)

---

## Project Overview

Implements a user account microservice providing login, registration, customer, address, and card management over HTTP with a MongoDB-backed persistence layer.

|  |  |
|---|---|
| Domain | user accounts, authentication, and customer profile management |
| Languages | Go |
| Entry points | `main.main`, `api.MakeHTTPHandler`, `GitHub Actions workflow main.yaml` |

*Source: `api/transport.go:102-115`, `api/service_test.go:26-39`*

---

## How the System Works

> **Key idea**
> HTTP handlers decode requests and call service methods, which in turn call the db package that delegates to a configured database implementation.

The system's main paths are:

1. HTTP request → decode*Request → endpoint → instrumentingService → fixedService → db.* → encodeResponse/encodeError → HTTP response

**The rules that shape its behaviour**

- Login requires a valid username and a password whose SHA1(salt+password) hash matches the stored hash; otherwise ErrUnauthorized is returned and an empty user object is produced.
- On successful login, user attributes (addresses and cards) are loaded and credit card numbers are masked before returning the user.
- Registration and user creation always hash the password with a per-user salt before persisting, and new users are created with links added for HATEOAS-style navigation.
- GET operations without an ID return full collections with HATEOAS links, while GET with an ID returns a single resource wrapped in a one-element slice or a single object.
- Health checks report both application and database status, marking the DB as 'err' if db.Ping fails.

---

## Technology Stack

Every entry below was detected in the repository — from source files, route annotations, import statements or build manifests.

| Technology | Purpose | Evidence |
|---|---|---|
| **Go** | Implementation language | 19 parsed source file(s) |
| **Javascript** | Implementation language | 5 parsed source file(s) |
| **Shell** | Implementation language | 3 parsed source file(s) |
| **Docker** | Container image build | Dockerfile is present |
| **Docker Compose** | Local multi-service run | docker-compose.yml is present |
| **Make** | Task runner | Makefile is present |
| **GitHub Actions** | Continuous integration | 1 workflow file(s) |

---

## Repository Structure

Directories that contain analysed source, largest first.

| Directory | Files | Role |
|---|---|---|
| `api/` | 7 | Encapsulate business rules for user authentication and profile management. |
| `users/` | 7 | — |
| `docker/user-db/scripts/` | 5 | — |
| `db/` | 2 | Provide a facade over concrete database implementations for users, addresses, and cards. |
| `db/mongodb/` | 2 | — |
| `scripts/` | 2 | — |
| `./` | 1 | — |
| `apispec/` | 1 | — |

> **Note**
> 1 file(s) were excluded from analysis: 1 × binary or media file.

---

## Architecture

Layered microservice with Go kit-style transport/endpoints/service separation and a pluggable database backend.

**Layers:** `HTTP transport (routing, request/response encoding)` → `Endpoint layer (tracing, orchestration)` → `Service layer (business logic, security rules)` → `Database abstraction layer (db facade over concrete DB implementations)`

Instrumentation via api.instrumentingService wraps the service layer to record Prometheus metrics for each method, but does not alter business behavior.

---

## Components

### fixedService

`api.fixedService` · *Encapsulate business rules for user authentication and profile management.*

Validate credentials, hash passwords, orchestrate loading of related entities, and call db functions to persist or retrieve data.

|  |  |
|---|---|
| Inputs | Typed request data such as username/password, user IDs, and user/address/card structs. |
| Outputs | Domain objects (User, Address, Card), slices of these, health status slices, and errors. |

Implements login by fetching a user, comparing hashed passwords, loading attributes, and masking cards; implements registration and posting by hashing passwords and delegating to db.Create*; wraps db reads to add HATEOAS links and health information.

> **Failure behaviour**
> Propagates db errors directly and returns ErrUnauthorized for bad credentials, which higher layers convert into HTTP 401 responses.

*Source: `api/service.go:40`*

### db

`db` · *Provide a facade over concrete database implementations for users, addresses, and cards.*

Select the appropriate DB implementation, delegate CRUD operations, and enrich returned entities with HATEOAS links.

|  |  |
|---|---|
| Inputs | Entity identifiers, user/address/card structs, and configuration for database type. |
| Outputs | Persisted entities, slices of entities, and errors. |
| Depends on | `users` |
| Key symbols | `TestInit`, `TestSet`, `TestRegister`, `TestCreateUser`, `TestGetUser`, `TestGetUserByName`, `TestGetUserAttributes`, `TestPing` |

On Init, ensures a database type is selected and calls DefaultDb.Init; for reads, calls DefaultDb methods and then adds links to returned entities; for writes, forwards calls to DefaultDb.

> **Failure behaviour**
> Returns ErrNoDatabaseSelected if no DB is configured, or a formatted ErrNoDatabaseFound if the configured type is unknown, and propagates errors from DefaultDb methods.

*Source: `db/db_test.go:106-108`, `db/db_test.go:1-160`*

---

## Execution Flows

Each flow below follows resolved call edges through the code. Every step names the symbol that performs it.

### User login flow

**Trigger:** `HTTP GET request to /login with Basic Auth credentials.`

**Entry point:** `api.MakeHTTPHandler → go-kit HTTP server for GET /login`

```text
  +======================================================+
  | HTTP GET request to /login with Basic Auth           |
  | credentials.                                         |
  +======================================================+
                              |
                              v
  +------------------------------------------------------+
  | Decode Basic Auth credentials from the HTTP request, |
  | failing with ErrUnauthorized if not present.         |
  +------------------------------------------------------+
                              |
                              |  decodeLoginRequest returned without error
                              v
  +------------------------------------------------------+
  | Endpoint constructs a loginRequest from decoded data |
  | and calls Service.Login, returning userResponse and  |
  | error.                                               |
  +------------------------------------------------------+
                              |
                              |  Service.Login invoked and database configured
                              v
  +------------------------------------------------------+
  | db.GetUserByName delegates to                        |
  | DefaultDb.GetUserByName and adds HATEOAS links if    |
  | successful.                                          |
  +------------------------------------------------------+
                              |
                              |  Endpoint returned non-nil error
                              v
  +------------------------------------------------------+
  | If any error is returned from the endpoint,          |
  | encodeError writes an HTTP error response with JSON  |
  | body and appropriate status code.                    |
  +------------------------------------------------------+
                              |
                              |  Endpoint returned nil error
                              v
  +------------------------------------------------------+
  | On success, encodeResponse sets Content-Type and     |
  | JSON-encodes the userResponse.                       |
  +------------------------------------------------------+
                              |
                              v
  +======================================================+
  | Either a 200 OK with the authenticated user          |
  | attributes loaded and cards masked or an error       |
  | response with 401 or 500 and a JSON error body.      |
  +======================================================+
```

| # | Kind | What happens | Source |
|---|---|---|---|
| 1 | `validation` | Decode Basic Auth credentials from the HTTP request, failing with ErrUnauthorized if not present. *(if BasicAuth present (ok == true))* | `api/transport.go:117-127` |
| 2 | `call` | Endpoint constructs a loginRequest from decoded data and calls Service.Login, returning userResponse and error. *(if decodeLoginRequest returned without error)* | `api/endpoints.go:49-59` |
| 3 | `db_read` | db.GetUserByName delegates to DefaultDb.GetUserByName and adds HATEOAS links if successful. *(if Service.Login invoked and database configured)* | `db/db_test.go:109-111` |
| 4 | `error` | If any error is returned from the endpoint, encodeError writes an HTTP error response with JSON body and appropriate status code. *(if Endpoint returned non-nil error)* | `api/transport.go:102-115` |
| 5 | `response` | On success, encodeResponse sets Content-Type and JSON-encodes the userResponse. *(if Endpoint returned nil error)* | `api/transport.go:199-203` |

**Branches:** If BasicAuth is missing, decodeLoginRequest returns ErrUnauthorized and the flow ends with HTTP 401.; If db.GetUserByName returns an error, Service.Login returns an empty user and the error, leading to HTTP 500.; If password hash does not match, Service.Login returns ErrUnauthorized, leading to HTTP 401.

**Fallback paths:** On any non-ErrUnauthorized error, encodeError falls back to HTTP 500 with a generic error JSON.

**Errors:** ErrUnauthorized for missing or invalid credentials; Database errors from db.GetUserByName or db.GetUserAttributes

> **Outcome**
> Either a 200 OK with the authenticated user (attributes loaded and cards masked) or an error response with 401 or 500 and a JSON error body.

---

## Business Logic

> **Key idea**
> These are decisions the software makes about its domain — not framework behaviour and not plumbing.

### Login authorization rule

A login succeeds only if Basic Auth credentials are provided and the SHA1(salt+password) hash matches the stored password hash; otherwise ErrUnauthorized is returned and no user data is exposed.

|  |  |
|---|---|
| Category | `authentication` |
| Implemented in | `api.decodeLoginRequest` |
| Triggered when | GET /login with Basic Auth credentials |

*Source: `api/transport.go:117-127`*

---

## Build, Tooling & CI

Every entry below is present in the repository as a real file.

| Tool | Kind | Purpose | Configured in |
|---|---|---|---|
| **Docker** | build | Container image build | `Dockerfile` |
| **Docker Compose** | build | Local multi-service run | `docker-compose.yml` |
| **Make** | build | Task runner | `Makefile` |
| **GitHub Actions** | ci | Continuous integration | `.github/workflows/main.yaml` |

### Continuous integration

1 GitHub Actions workflow(s) run against this repository.

| Workflow | File |
|---|---|
| `main` | `.github/workflows/main.yaml` |

### Build and tooling entry points

| Entry point | Kind | File |
|---|---|---|
| `mongo_create_insert.sh` | script | `docker/user-db/scripts/mongo_create_insert.sh` |
| `push.sh` | script | `scripts/push.sh` |
| `testcontainer.sh` | script | `scripts/testcontainer.sh` |

---

## File Reference

small repository: all 27 analysed source files are listed

| File | Role | Symbols | Key symbols | Uses | Used by | Documented above |
|---|---|---|---|---|---|---|
| `db/mongodb/mongodb.go` | source | 30 | `init`, `Mongo`, `Init`, `MongoUser`, `New` | `addresses.go` | `main.go` | — |
| `api/endpoints.go` | source | 27 | `Endpoints`, `MakeEndpoints`, `MakeLoginEndpoint`, `MakeRegisterEndpoint`, `MakeUserGetEndpoint` | `db.go`, `addresses.go` | `main.go` | yes |
| `api/middlewares.go` | source | 25 | `Middleware`, `LoggingMiddleware`, `loggingMiddleware`, `Login`, `Register` | `addresses.go` | — | — |
| `db/db_test.go` | Provide a facade over concrete database implementations for users, addresses, and cards. | 23 | `TestInit`, `TestSet`, `TestRegister`, `TestCreateUser`, `TestGetUser` | `addresses.go` | — | yes |
| `db/db.go` | source | 18 | `Database`, `init`, `Init`, `Set`, `Register` | `addresses.go` | `endpoints.go`, `service.go`, `main.go` | — |
| `api/service.go` | Encapsulate business rules for user authentication and profile management. | 15 | `Service`, `NewFixedService`, `fixedService`, `Health`, `Login` | `db.go`, `addresses.go` | — | yes |
| `db/mongodb/mongodb_test.go` | source | 14 | `init`, `TestMain`, `exitTest`, `TestInit`, `TestNew` | `addresses.go` | — | — |
| `api/transport.go` | source | 12 | `MakeHTTPHandler`, `encodeError`, `decodeLoginRequest`, `decodeRegisterRequest`, `decodeDeleteRequest` | `addresses.go` | — | yes |
| `users/links.go` | source | 8 | `init`, `Links`, `AddLink`, `AddAttrLink`, `AddCustomer` | — | — | — |
| `scripts/push.sh` | script | 6 | `push`, `tag_and_push_all`, `DOCKER_PUSH`, `TAG`, `REPO` | — | — | — |
| `users/users.go` | source | 6 | `User`, `New`, `Validate`, `MaskCCs`, `AddLinks` | — | — | — |
| `api/middlewares_test.go` | source | 4 | `bogusLogger`, `newBogusLogger`, `Log`, `TestLoginMiddleWare` | — | — | — |
| `api/service_test.go` | source | 4 | `init`, `TestLogin`, `TestRegister`, `TestCalculatePassHash` | `addresses.go` | — | yes |
| `api/endpoints_test.go` | source | 3 | `TestMakeEndpoints`, `TestMakeLoginEndpoint`, `TestMakeRegisterEndpoint` | — | — | — |
| `docker/user-db/scripts/mongo_create_insert.sh` | script | 3 | `SCRIPT_DIR`, `FILES` | — | — | — |
| `main.go` | source | 3 | `init`, `main`, `ServiceName` | `endpoints.go`, `db.go`, `mongodb.go` | — | — |
| `users/cards.go` | source | 3 | `Card`, `MaskCC`, `AddLinks` | — | — | — |
| `users/users_test.go` | source | 3 | `TestNew`, `TestValidate`, `TestMaskCCs` | — | — | — |
| `docker/user-db/scripts/address-insert.js` | source | 2 | `get_results`, `insert_address` | — | — | — |
| `docker/user-db/scripts/card-insert.js` | source | 2 | `get_results`, `insert_card` | — | — | — |
| `docker/user-db/scripts/customer-insert.js` | source | 2 | `get_results`, `insert_customer` | — | — | — |
| `users/addresses.go` | source | 2 | `Address`, `AddLinks` | — | `endpoints.go`, `middlewares.go`, `service.go` +6 | — |
| `users/cards_test.go` | source | 2 | `TestAddLinksCard`, `TestMaskCC` | — | — | — |
| `scripts/testcontainer.sh` | test | 1 | `RESPONSECODE` | — | — | — |
| `users/addresses_test.go` | source | 1 | `TestAddLinksAdd` | — | — | — |
| `apispec/hooks.js` | source | 0 | — | — | — | — |
| `docker/user-db/scripts/accounts-create.js` | source | 0 | — | — | — | — |

---
