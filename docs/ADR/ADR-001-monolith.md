# ADR-001: Start with Monolith Architecture

## 📌 Context
We need to choose an initial architecture for the **Billiard Training App** backend.  
Options considered: starting with a modular monolith vs. microservices.

The app is still in its MVP phase, aiming for **fast delivery of first value** (sign up → create plan → log shot → see one stat).  
Microservices would introduce extra complexity (deployment, CI/CD, inter-service communication) before we actually need it.

## 💡 Decision
We will **start with a monolithic backend** (Spring Boot application with modular layers).

## 🔄 Consequences
**Positive**:
- Faster development and simpler deployment
- Easier debugging and testing in early iterations
- Lower infrastructure cost

**Negative**:
- Potential refactor required if scaling demands microservices later
- Some modules may be tightly coupled initially

## ⚖ Alternatives
- **Microservices from day 1** → rejected: too much complexity, slows down MVP.
- **Serverless functions** → rejected: less control, higher cold start complexity, overkill for MVP.  
