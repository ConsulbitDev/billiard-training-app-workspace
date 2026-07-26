# Billiard Training App — API Contract & Integration Reference

This document is a **lightweight bridge** between the Spring Boot backend and Angular frontend. It focuses on concerns that **cannot be auto-generated** from OpenAPI specifications.

**For endpoint contracts, DTO schemas, and request/response details:** See the OpenAPI specification (auto-generated from Spring Boot controllers via `springdoc-openapi`).

OpenAPI is fully integrated on the BE (see section 2). This document stays in its current
lightweight form going forward: enum mappings, known bugs, setup instructions, and status
tracking. All endpoint/DTO details live in the OpenAPI spec, not here.

---

## 1. Project Topology

- All sibling projects live as flat folders under one common parent (see the workspace root's `CLAUDE.md` for the full map).
- **Backend:** `billiard-training-app-be/` (Spring Boot 3, Java 17, runs on **http://localhost:8080**)
- **Frontend:** `billiard-training-app-fe/` (Angular 19, TypeScript 5.7, runs on **http://localhost:4200**)
- **GitHub project board:** https://github.com/orgs/ConsulbitDev/projects/1

---

## 2. Running the Full Stack

**Start the backend:**
```bash
cd billiard-training-app-be
mvn spring-boot:run
```
Backend will listen at `http://localhost:8080`.

**Start the frontend (in another terminal):**
```bash
cd billiard-training-app-fe
npm start
```
Frontend dev server will listen at `http://localhost:4200`.

**Access OpenAPI documentation:**
- **Swagger UI (interactive):** http://localhost:8080/openapi/swagger-ui.html
- **OpenAPI JSON spec:** http://localhost:8080/openapi/v3/api-docs
- **OpenAPI YAML spec:** http://localhost:8080/openapi/v3/api-docs.yaml

---

## 3. Enum Mapping Reference

This is the **authoritative list** of all enums, their wire formats, FE equivalents, and UI display labels. This cannot be auto-generated and must be maintained here.

### 3.1 Type

| BE Java Constant | JSON wire value | FE constant | Display label | Color   |
|------------------|-----------------|-------------|---------------|---------|
| `DIRECT`         | `"DIRECT"`      | `DIRECT`    | "Diretto"     | #0a53a8 |
| `INDIRECT`       | `"INDIRECT"`    | `INDIRECT`  | "Indiretto"   | #11734b |

### 3.2 Topology

| BE Java Constant | JSON wire value | FE constant | Display label  | Color   |
|------------------|-----------------|-------------|----------------|---------|
| `NUMBERING`      | `"NUMBERING"`   | `NUMBERING` | "Numerazione"  | #0a53a8 |
| `SHOT`           | `"SHOT"`        | `SHOT`      | "Tiro"         | #11734b |

### 3.3 Priority

| BE Java Constant | JSON wire value | FE constant    | Display label   | Color   |
|------------------|-----------------|----------------|-----------------|---------|
| `NICE_TO_HAVE`   | `"NICE_TO_HAVE"`| `NICE_TO_HAVE` | "Nice to Have"  | #0a53a8 |
| `MUST_HAVE`      | `"MUST_HAVE"`   | `MUST_HAVE`    | "Must Have"     | #11734b |
| `NOT_NEEDED`     | `"NOT_NEEDED"`  | `NOT_NEEDED`   | "Not Needed"    | #e6cff2 |
| `UNRELIABLE`     | `"UNRELIABLE"`  | `UNRELIABLE`   | "Unreliable"    | #b10202 |
| `RECOMMENDED`    | `"RECOMMENDED"` | `RECOMMENDED`  | "Recommended"   | #e8eaed |
| `INTERESTED`     | `"INTERESTED"`  | `INTERESTED`   | "Interested"    | #5a3286 |

### 3.4 ResourceType

| BE Java Constant | JSON wire value | FE constant | Purpose                 |
|------------------|-----------------|-------------|-------------------------|
| `IMAGE`          | `"IMAGE"`       | `IMAGE`     | Photo/diagram resource  |
| `VIDEO`          | `"VIDEO"`       | `VIDEO`     | Video URL resource      |
| `PDF`            | `"PDF"`         | `PDF`       | PDF document resource   |

---

## 4. Known Issues & Bugs

These are discrepancies that need workarounds or special handling. They cannot be expressed in OpenAPI and must be documented here.

### 4.1 BookController.DELETE Returns 200 Instead of 204 — ✅ RESOLVED

**Location:** BE `src/main/java/net/academy/consulbit/java/be/controller/BookController.java`

**Status:** Resolved

`DELETE /api/books/{id}` now returns `ResponseEntity.noContent().build()` (204 No Content), matching the other DELETE endpoints.

### 4.2 BookController.PATCH Uses ReflectionUtils — ✅ RESOLVED

**Location:** BE `src/main/java/net/academy/consulbit/java/be/controller/BookController.java`

**Status:** Resolved (TASKS.md section 6)

The `PATCH /api/books/{id}` endpoint was removed. Book updates now go through `PUT /api/books/{id}` with a full `BookRequestDTO` payload, mapped explicitly — no more reflection-based partial patching.

**FE note:** If any FE code still targets `PATCH /api/books/{id}`, switch it to `PUT` with the complete `BookRequestDTO` shape (see OpenAPI spec).

---

## 5. Implementation Status Matrix

This matrix tracks which endpoints are fully implemented (LIVE) vs planned (PENDING).

| Resource | Endpoint | Method | Status | PR/Commit |
|----------|----------|--------|--------|-----------|
| **Shots** | `/api/shots` | GET | ✅ LIVE | #36 |
| | `/api/shots/{id}` | GET | ✅ LIVE | #36 |
| | `/api/shots` | POST | ✅ LIVE | #36 |
| | `/api/shots/{id}` | PUT | ✅ LIVE | #36 |
| | `/api/shots/{id}` | DELETE | ✅ LIVE | #36 |
| **Books** | `/api/books` | GET | ✅ LIVE | #35 |
| | `/api/books/{id}` | GET | ✅ LIVE | #35 |
| | `/api/books` | POST | ✅ LIVE | #35 |
| | `/api/books/{id}` | PUT | ✅ LIVE | #35 (now the only update path — PATCH removed, see 4.2) |
| | `/api/books/{id}` | DELETE | ✅ LIVE | #35 (now returns 204, see 4.1) |
| **Categories** | `/api/categories` | GET | ✅ LIVE | #53 (Task #50) |
| | `/api/categories/{id}` | GET | ✅ LIVE | #53 (Task #50) |
| | `/api/categories` | POST | ✅ LIVE | #53 (Task #50) |
| | `/api/categories/{id}` | PUT | ✅ LIVE | #53 (Task #50) |
| | `/api/categories/{id}` | DELETE | ✅ LIVE | #53 (Task #50) |
| **Comments** | `/api/shots/{shotId}/comments` | POST | ✅ LIVE | #49 (Task #42) |
| | `/api/comments/{commentId}` | PUT | ✅ LIVE | #49 (Task #42) |
| | `/api/comments/{commentId}` | DELETE | ✅ LIVE | #49 (Task #42) |
| **Resources** | `/api/shots/{shotId}/resources` | POST | ✅ LIVE | #53/#54 (Task #51) |
| | `/api/resources/{id}` | PUT | ✅ LIVE | #53/#54 (Task #51) |
| | `/api/resources/{id}` | DELETE | ✅ LIVE | #53/#54 (Task #51) |

**Note:** There is no standalone `GET` for comments or resources — they are returned nested inside `GET /api/shots/{id}` (`ShotDetailDTO.comments` / `.resources`), ordered newest-first and by `orderIndex` respectively.

**Pagination/sorting on `GET /api/shots`:** page size is clamped server-side to `25` (default), `50`, or `100`; sort is restricted to `id`, `name`, `type`, `topology`, `priority`, `pageNumber` — any other value returns `400`.

---

## 6. FE-BE Model Alignment Issues

These are mapping concerns specific to the frontend that the OpenAPI spec cannot express.

### Active Gaps (Awaiting Resolution)

| # | Severity | Field/Concept | FE (current) | BE (authoritative) | Impact | Fix needed |
|---|----------|---------------|--------------|-------------------|--------|-----------|
| 1 | **HIGH** | `Shot.title` | Field name is `title` | DTO field is `name` | FE will fail to parse JSON (key mismatch) | Rename FE `Shot.title` → `Shot.name` |
| 2 | **HIGH** | `Category` | Flat enum with Italian strings | Entity with `id: number`, `name: string`, `color: string` | Category selector, shot creation form will break | Replace FE Category enum with `Category` interface; load from `/api/categories` |
| 3 | **HIGH** | Enum values | Stored as Italian display strings | Sent over JSON as Java constant names | FE will fail to match enum values from API | Change FE enum values to match BE constant names; create separate label maps |
| 4 | **MEDIUM** | `Shot` missing fields | No `id`, `description`, `resources`, `comments` | All present on `ShotDetailDTO` | FE will not display shot details, resources, or comments | Add these fields to FE interfaces |
| 5 | **MEDIUM** | `Shot.referenceBook` | String field (title only) | Object field (`book: BookDTO`) | Shot creation form won't display book correctly | Replace with `book: Book \| null` |
| 6 | **MEDIUM** | `Shot.videoUrl` / `Shot.schemaUrl` | Arrays of strings on Shot | Arrays of `ResourceDTO` with type filter | Video/schema display will break | Replace with `resources: Resource[]`; filter by type |
| 7 | **LOW** | `pageNumber` type | `bookPage?: number` (numeric) | `pageNumber: String` | Values like "14-15" (page ranges) cannot be represented | Change to `pageNumber: string` |

### How to Resolve

1. **Check the OpenAPI spec** (`http://localhost:8080/openapi/swagger-ui.html`) for the actual DTO structure
2. **Update FE interfaces** to match BE DTOs exactly
3. **Use code generation** (see section 8) to auto-sync TypeScript interfaces from OpenAPI schemas
4. **Update this table** when gaps are resolved (move to "Resolved" section)

---

## 7. FE Integration Setup

See `INTEGRATION.md` in each repo for detailed setup:

- **FE:** `billiard-training-app-fe/INTEGRATION.md` — environment.ts, proxy, HttpClient, enum mapping patterns
- **BE:** `billiard-training-app-be/INTEGRATION.md` — CORS, enum serialization, known issues

---

## 8. Code Generation: Auto-Sync FE Interfaces from OpenAPI — ✅ SET UP

The BE's OpenAPI spec is fully documented, so the FE uses **code generation** (`ng-openapi-gen`)
to automatically create TypeScript interfaces from the API schemas instead of hand-maintaining DTO
equivalents.

### Using ng-openapi-gen

Already installed as a devDependency in `billiard-training-app-fe`. The package is
`ng-openapi-gen` itself (not `@ngtools/schematics`, which is an unrelated internal Angular CLI
package — an earlier version of this doc had that wrong).

**Regenerate anytime the BE API changes** (requires the BE running locally on port 8080):
```bash
cd billiard-training-app-fe
npm run gen:api
```

This runs `ng-openapi-gen --input http://localhost:8080/openapi/v3/api-docs --output src/app/api-generated`
(the script is defined in `package.json`) and generates into `src/app/api-generated/`:
- TypeScript interfaces for all 14 BE DTOs/schemas (`models/`)
- HTTP service functions for all 5 endpoint groups — Shot, Book, Category, Comment, Resource (`fn/`)
- `api-configuration.ts`, `api.ts`, `request-builder.ts` (client scaffolding)

Verified against the live spec: passes `tsc --noEmit`, and doesn't affect the production bundle
size (nothing imports it yet — FE services still need to be wired up to use it, see FE `TASKS.md`
section 3).

**Still open (tracked in FE `TASKS.md` section 2):**
- Whether to replace the manually-written interfaces from FE `TASKS.md` section 1 with the
  generated ones
- Whether to commit `src/app/api-generated/` or gitignore it and regenerate on each BE change
  (currently untracked/uncommitted)

**Note:** You'll still need to map FE enums to BE enums (the label maps from INTEGRATION.md), since that's business logic not API schema.

---

## 9. Maintenance & Evolution

### Current State (OpenAPI-First)

- The BE exposes a live, auto-generated OpenAPI spec (`springdoc-openapi`) — the source of truth
  for all endpoint definitions, DTO field lists/types, request/response shapes, and validation
  rules. Nothing here duplicates that.
- `API_CONTRACT.md` stays lightweight: enum mapping tables, known issues/bugs, implementation
  status, and FE-BE alignment gaps — the handful of things OpenAPI can't express.
- `INTEGRATION.md` files (BE and FE) hold side-specific setup, enum transformation logic, and
  workarounds for known bugs.
- FE interfaces are auto-generated from the spec via `ng-openapi-gen` (section 8) rather than
  hand-maintained.

### When to Update This Document

**Update enum mapping tables (section 3):**
- If a new enum value is added
- If a display label or color changes
- In the same commit as the code change

**Update known issues (section 4):**
- When a bug is discovered or fixed
- When a workaround is found
- Move between "Open" and "Resolved" sections

**Update implementation status (section 5):**
- When an endpoint transitions from PENDING to LIVE
- Add PR/commit reference
- In the same PR that implements the endpoint

**Update FE-BE gaps (section 6):**
- When a misalignment is discovered
- When a gap is resolved, move to "Resolved" section
- Reference the PR that fixed it

---

## 10. Quick Reference for Agents

**When working on the backend:**
- Check the **implementation status** (section 5) to see what's LIVE vs PENDING
- Read **BE INTEGRATION.md** for CORS, enum serialization, and known issues
- Add endpoint details to OpenAPI via `@Operation` and `@Parameter` annotations in code

**When working on the frontend:**
- Check the **FE-BE gaps** (section 6) to understand what needs fixing
- Read **FE INTEGRATION.md** for setup, enum mapping, and HttpClient patterns
- Use **OpenAPI spec** (section 2) as the source of truth for endpoint contracts and DTOs
- Consider using `ng-openapi-gen` to auto-generate TypeScript interfaces

**For both:**
- Check the **enum mapping tables** (section 3) when dealing with enum values
- Consult **known issues** (section 4) when troubleshooting API integration
- Keep this document in sync with code changes (update in same PR/commit)
