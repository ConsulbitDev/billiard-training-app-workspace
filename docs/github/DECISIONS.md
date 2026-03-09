# 📘 Decisions Log

This file tracks key architecture and product decisions for the **Billiard Training App**.  
Each entry points to a full ADR file stored in `/docs/adr/`.

---

## Format
`YYYY-MM-DD – Decision – Reason – Impact – Link`

---

## Decisions

- 2025-09-24 – Start with a **monolith architecture** – Avoid premature complexity, focus on speed – Simplifies first release – [ADR-001](docs/adr/ADR-001-monolith.md)
- 2025-09-25 – Adopt **GitHub Project + Issue Templates** – Ensure consistent workflow – Keeps backlog structured – [ADR-002](docs/adr/ADR-002-workflow.md)

---

## How to Add a New Decision
1. Create a new ADR under `/docs/adr/`
    - File name: `ADR-XXX-title.md` (increment number).
    - Use template in `/docs/templates/ADR.md`.
2. Add an entry in this log with date, summary, and link.

---

✅ This log helps us (and future contributors) quickly see **what was decided, when, and why**.
