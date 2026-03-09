# ADR-002: Adopt GitHub Project + Issue Templates Workflow

## 📌 Context
We need a consistent process to capture requirements, plan work, and track delivery.  
Several methods exist (Jira, Trello, Notion, etc.), but we want to leverage **GitHub Projects** tightly integrated with our repos.

## 💡 Decision
We will use **GitHub Projects** as our single source of truth, supported by:
- **Epics, Stories, Tasks** as issues
- **Labels** for type, priority, and area
- **Custom Issue Templates** for consistency
- **Pull Request Template** for review quality
- **Decisions Log** (`DECISIONS.md`) with ADR files for architecture decisions

## 🔄 Consequences
**Positive**:
- Clear, repeatable structure for all work
- Fast onboarding for contributors (everything in GitHub)
- Direct link between requirements → code → review

**Negative**:
- Requires discipline to keep labels and templates consistent
- May feel rigid for ad-hoc brainstorming (handled in HQ chat)

## ⚖ Alternatives
- **Jira/Trello/Notion** → rejected: adds more tools, slower, less integrated with code.
- **No formal workflow** → rejected: leads to chaos, lack of traceability.  
