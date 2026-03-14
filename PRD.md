
# Billiard Training App
## Product Requirements Document (PRD)

Version: 1.0  
Scope: V1 — Knowledge Base Explorer + Admin

---

# 1. Product Overview

## Purpose
The Billiard Training App transforms a spreadsheet-based billiards knowledge base into a structured web application that enables efficient browsing, consultation, and maintenance of shot knowledge.

The application allows players to:
- locate shots quickly
- consult diagrams, videos, and references
- maintain structured notes
- manage the knowledge base

Training session tracking and statistics are **out of scope for V1**.

---

# 2. Product Goals

Primary goals:
- Fast shot discovery
- Efficient filtering
- Clear shot visualization
- Structured knowledge management

Secondary goals:
- Provide a maintainable data model
- Support incremental knowledge growth
- Enable future training features

---

# 3. Scope

## In Scope (V1)

- Shot browsing
- Shot filtering
- Shot detail consultation
- Resource visualization (image/video/pdf)
- Comment system
- Admin CRUD for shots
- Admin CRUD for categories
- Admin CRUD for books
- Resource management

## Out of Scope (V1)

- Training sessions
- Shot success tracking
- Statistics
- AI shot suggestions
- Multi-user accounts
- Authentication
- Social features

---

# 4. System Architecture

## Backend

Technology stack:
- Spring Boot
- Spring Data JPA
- PostgreSQL (or compatible)
- OpenAPI

Architecture rules:
- DTO-based API responses
- JPA Specifications for filtering
- Server-side pagination
- Lazy loading for heavy relations

## Frontend

Technology stack:
- Angular
- Angular Material
- Tailwind CSS
- RxJS

Responsibilities:
- data grid rendering
- filter interaction
- detail view rendering
- resource embedding
- admin forms

---

# 5. Domain Model

## Shot

Represents a billiards concept or system extracted from a learning resource.

Fields:

| Field | Description |
|------|-------------|
| id | unique identifier |
| name | shot name |
| description | explanation of the shot |
| category | conceptual grouping |
| type | direct or indirect |
| topology | numbering system or shot |
| priority | training importance |
| pageNumber | reference page |
| book | source reference |

---

## Category

Logical grouping of shots.

Examples:
- Americana
- Striscio
- Filotto
- Traversino
- Sfaccio

Fields:

| Field | Description |
|------|-------------|
| id | identifier |
| name | category name |
| description | explanation |
| priority | UI ordering |
| color | UI display color |

---

## Book

Represents the learning source of a shot.

Fields:

| Field | Description |
|------|-------------|
| id | identifier |
| title | book title |
| author | author |
| cover | cover image |
| numOfPages | number of pages |
| published | publication date |
| source | optional digital resource |

---

## Resource

External media associated with a shot.

Fields:

| Field | Description |
|------|-------------|
| id | identifier |
| shot | owning shot |
| title | resource label |
| description | optional description |
| url | external link |
| type | IMAGE / VIDEO / PDF |
| orderIndex | rendering order |

---

## Comment

User annotations about a shot.

Fields:

| Field | Description |
|------|-------------|
| id | identifier |
| message | comment text |
| createdAt | creation timestamp |
| updatedAt | last update timestamp |

---

# 6. Enumerations

## Type
DIRECT  
INDIRECT

## Topology
NUMBERING  
SHOT

## Priority
MUST_HAVE  
RECOMMENDED  
NICE_TO_HAVE  
INTERESTED  
UNRELIABLE  
NOT_NEEDED

---

# 7. Shot Explorer

## Shot List

The main application screen.

Columns:
- Name
- Category
- Type
- Topology
- Priority
- Book
- Page

---

## Filtering

Supported filters:
- Category
- Type
- Topology
- Priority
- Search text

Filter rules:
- Within same filter → OR
- Across filters → AND

Example:

(Category = Filotto OR Traversino)  
AND  
(Priority = MUST_HAVE)

---

## Search

Search applies to:
- Shot.name
- Shot.description

Behavior:
- case insensitive
- partial match

---

# 8. Shot Detail View

Sections:

1. Shot metadata
2. Description
3. Resources
4. Comments

Metadata fields:
- Name
- Category
- Type
- Topology
- Priority
- Book
- Page

---

## Resources

Supported types:
- IMAGE
- VIDEO
- PDF

Rendering:

| Type | Rendering |
|------|----------|
| IMAGE | img element |
| VIDEO | iframe |
| PDF | iframe viewer |

Google Drive URLs must be converted for embedding.

---

## Comments

Users can:
- add comment
- edit comment
- delete comment

Sorted by newest first.

---

# 9. Admin Module

Routes:

/admin  
/admin/shots  
/admin/shots/new  
/admin/shots/:id/edit  
/admin/categories  
/admin/books

---

# 10. Shot Administration

Admin can:
- create shot
- edit shot
- delete shot

Editable fields:

- name
- description
- category
- type
- topology
- priority
- book
- pageNumber
- resources

---

# 11. Resource Management

Resource fields:

- title
- url
- type
- description
- orderIndex

Resources are added by pasting external URLs.

---

# 12. Category Management

Fields:

- name
- description
- priority
- color

Admin can create, update, and delete categories.

---

# 13. Book Management

Fields:

- title
- author
- cover
- numOfPages
- published
- source

Admin can create, update, and delete books.

---

# 14. API Specification

## Shot Search

GET /shots

Parameters:
- categoryIds
- types
- topologies
- priorities
- search
- page
- size
- sort

---

## Shot Detail

GET /shots/{id}

---

## Comments

POST /shots/{id}/comments  
PUT /comments/{id}  
DELETE /comments/{id}

---

## Resources

POST /shots/{id}/resources  
PUT /resources/{id}  
DELETE /resources/{id}

---

# 15. DTO Strategy

Entities must never be returned directly.

## ShotSummaryDTO

Fields:

- id
- name
- categoryId
- categoryName
- type
- topology
- priority
- bookTitle
- pageNumber

---

## ShotDetailDTO

Fields:

- id
- name
- description
- category
- type
- topology
- priority
- book
- resources
- comments

---

# 16. Query Strategy

Filtering must use **JPA Specifications**.

Example predicates:

- category.id IN (...)
- type IN (...)
- topology IN (...)
- priority IN (...)
- LOWER(name) LIKE %search%
- LOWER(description) LIKE %search%

---

# 17. Pagination

Server-side pagination.

Default page size:
25

Optional:
50  
100

---

# 18. Performance Strategy

Rules:

- Do not load resources in shot list
- Load resources only in detail view
- Use DTO projections
- Use lazy loading

---

# 19. Indexing

Recommended indexes:

- shots.category_id
- shots.priority
- shots.type
- shots.topology
- shots.name

---

# 20. Frontend Architecture

Angular feature modules:

- core
- shared
- layout
- shots
- admin

Core services:

- ShotService
- CategoryService
- BookService
- ResourceService

---

# 21. Styling Strategy

Angular Material:
- tables
- forms
- dialogs
- buttons

Tailwind:
- layout
- spacing
- responsive design

---

# 22. Performance Targets

| Action | Target |
|------|------|
| Shot list load | <300ms |
| Filter update | <300ms |
| Shot detail load | <400ms |

---

# 23. Future Evolution

Potential future modules:

- Training sessions
- Shot logging
- Performance statistics
- Training planning
- AI assistance
