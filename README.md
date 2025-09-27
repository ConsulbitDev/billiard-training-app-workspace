# 🛠️ Billiard Training App – Workflow

This document explains how we collect requirements, turn them into work items, and track them through delivery.  
Our guiding principle: **ship small slices of value fast** (sign up → create plan → log shot → see one stat).

---

## 1. Functional Requirements
- **Entry point**: all ideas/questions start in HQ chat.
- **Format**: capture as **Job Stories**
  > *When … I want to … so I can …*
- **Owner**: Product Owner + team discussion.

---

## 2. Epics
- **Purpose**: group related user needs under a clear goal.
- **Format**:
    - **Goal**
    - **Must / Should / Won’t**
    - **Success Metrics**
    - **Risks / Tests**
    - **Linked Stories**
- **GitHub**: issue labeled `type:epic`, linked to stories.

---

## 3. Stories
- **Purpose**: represent a single user job.
- **Format**:
    - **Job Story**
    - **Acceptance Criteria** (checkboxes)
    - **Area / Size / Priority**
    - **Notes**
- **GitHub**: issue labeled `type:story`, linked to epic.

---

## 4. Tasks
- **Purpose**: concrete developer steps (<30m–2h).
- **Format**:
    - **Action** → what to do
    - **Done criteria** → how we know it’s finished
- **GitHub**: issue labeled `type:task`, linked to story.

---

## 5. Architecture Decision Records (ADR)
- **Purpose**: log key technical/structural choices.
- **Format**: Context → Decision → Consequences → Alternatives.
- **GitHub**: markdown file under `/docs/adr/` and referenced in issues.

---

## 6. Execution & Tracking
- **Tool**: GitHub Project Board is the **source of truth**.
- **Columns**: Backlog → In Progress → Review → Done.
- **Labels**:
    - `type:epic | story | task`
    - `prio:P1 | P2`
    - `area:backend | frontend | docs | infra`

---

## 7. Cadence
- **Daily**: 5-min async standup in HQ → *Shipped / Next / Blockers*.
- **Weekly**: review metrics, pick 3 outcomes, log decisions.

---

## 8. Principles
- Deliver value early, not perfection.
- Keep scope small (one Epic = one clear goal).
- Start simple (monolith first, evolve later).
- Privacy & security by default.
- Everything must be **GitHub-ready** (copy-pasteable issue/PR text).

---

✅ With this workflow, any contributor can jump in and know:  
**where requirements come from, how work is shaped, and how progress is tracked.**
