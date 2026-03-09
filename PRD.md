Product Requirements Document (PRD)
===================================

**Project:** Billiard Training App**  
Variant:** 5-Pins (Italiana, Goriziana)**  
Version:** v1 – First Value Release**  
Audience:** Solo developer + AI-assisted engineering tools

1\. Goal & First Value
----------------------

### 1.1 Product Goal (v1)

Enable a player to:
*   Browse a structured catalog of billiards shots
*   Build a personal training plan
*   Log shot execution during training
*   See objective improvement over time

### 1.2 First Value Moment

> User selects shots → creates a training session → logs attempts → immediately sees success/failure statistics.

This is the **activation point**. Anything not supporting this flow is out of scope.

2\. Target User (v1)
--------------------

*   **Primary:** Single advanced amateur player (initially the author)
*   **Design intent:** Multi-user ready, but **no auth in v1**
*   **Assumption:** All data is scoped to a single logical user

_No accounts, no profiles, no sharing._

3\. Non-Goals (Hard Exclusions)
-------------------------------

Explicitly **out of scope** for v1:

*   Video analysis
*   AI-based shot suggestions
*   Social / community features
*   Coach dashboards
*   Advanced aiming systems (e.g. Edge Aiming System logic)
*   Monetization
*   Notifications

If it smells like “smart”, it’s v2+.

4\. System Context
------------------

### 4.1 Backend (Current State)

*   Spring Boot
*   REST APIs with OpenAPI
*   Database already populated from Google Sheets (shots catalog)
*   Single-user assumption
*   No authentication

### 4.2 Frontend (Target State)

*   Angular v19
*   Tailwind CSS (to be integrated if not present)
*   Service-based state sharing (no NgRx)
*   Responsive, desktop-first bias

5\. Core Domain Concepts (v1)
-----------------------------

### 5.1 Shot

Read-only reference entity.

**Key attributes (assumed, adjust if needed):**

*   id
*   name
*   category / system
*   description
*   difficulty (optional)
*   notes (read-only reference notes)

Shots are **not user-owned**.

### 5.2 Training Session

A user-created container.

**Attributes:**

*   id
*   title
*   date
*   optional free-text comment
*   list of logged shot attempts

### 5.3 Shot Attempt (Log Entry)

The atomic unit of tracking.

**Minimum required fields:**

*   shotId
*   success (boolean)
*   timestamp
*   optional comment

No physics. No ball positions. Just outcome + context.

6\. Core User Flows (v1)
------------------------

### 6.1 Browse Shots

**Purpose:** Discover and select shots.

*   Show list of shots
*   Filter by category / system
*   Open shot detail view
*   Read static shot information
*   Read and add **user comments** (important)

### 6.2 Create Training Session

**Purpose:** Define intent before execution.

Steps:

1.  Select one or more shots
2.  Create a named training session
3.  Optionally add session comment

_No scheduling, no recurrence._

### 6.3 Log Shots Quickly (Critical UX)

**Purpose:** Avoid friction during practice.

Requirements:

*   Rapid success/failure input
*   Minimal navigation
*   Inline comments per attempt
*   Undo / edit last entries

This is the **highest UX priority**.

### 6.4 Session Summary

Immediately after or during training:

*   Total attempts
*   Success %
*   Breakdown per shot
*   Session comments

### 6.5 Historical Stats

Across sessions:
*   Overall success %
*   Per-shot success %
*   Trend over time (simple)

Assumption: server computes aggregates, FE visualizes.

### 6.6 Export Data

*   Export sessions and logs (CSV or JSON)
*   Manual trigger
*   No automation

7\. Functional Requirements (Explicit)
--------------------------------------

### Must Have (v1)

*   View shot list
*   View shot detail
*   Add comments to shots
*   Create training session
*   Log multiple shot attempts quickly
*   Edit / delete attempts
*   Session summary
*   Historical statistics
*   Data export

### Should Have (v1 if cheap)

*   Session filtering
*   Shot favorites

### Won’t Have (v1)

Everything in section 3.

8\. Statistics & Feedback
-------------------------

### 8.1 Metrics

*   Success / Failure %
*   Attempts count
*   Per-shot aggregation

### 8.2 Update Strategy

*   Real-time update after each log
*   Backend returns updated aggregates
*   FE does not compute statistics

9\. Frontend Structure (AI-Friendly)
------------------------------------

### 9.1 Pages / Routes

*   /shots
*   /shots/:id
*   /sessions
*   /sessions/new
*   /sessions/:id
*   /stats
*   /export

### 9.2 Components (Suggested)

*   ShotListComponent
*   ShotDetailComponent
*   ShotCommentComponent
*   SessionBuilderComponent
*   ShotLoggerComponent
*   SessionSummaryComponent
*   StatsDashboardComponent

Each component:

*   Stateless where possible
*   Data via services
*   Inputs/Outputs explicit

10\. API Interaction Principles
-------------------------------

*   FE strictly consumes OpenAPI-defined endpoints
*   No hidden client logic
*   All mutations confirmed by backend response
*   Errors surfaced minimally, no UX overengineering

11\. AI-Assisted Engineering Guidelines
---------------------------------------

This PRD is optimized to:
*   Generate FE components one by one
*   Generate services aligned to backend resources
*   Create GitHub issues directly from sections
*   Enable step-by-step AI prompting (one feature per prompt)

**Rule:**

> One PR = one user-visible capability.

12\. Success Criteria (v1)
--------------------------

The app is successful if:
*   You can run a full training session without friction
*   Logging shots feels faster than pen & paper
*   Stats are visible immediately
*   No auth or setup overhead exists

13\. Open Assumptions (Confirm Later)
-------------------------------------

*   Exact shot attributes
*   Export format preference
*   Visualization library (if any)
*   Persistence strategy for comments