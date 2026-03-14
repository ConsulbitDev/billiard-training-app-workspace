# Billiard Training App - Microservice Architecture

**Version**: 1.0
**Date**: 2026-03-13
**Status**: Design Phase
**Scope**: V2+ Future Evolution

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Rationale](#architecture-rationale)
3. [Service Catalog](#service-catalog)
4. [Service Details](#service-details)
5. [Inter-Service Communication](#inter-service-communication)
6. [Data Consistency Patterns](#data-consistency-patterns)
7. [Deployment Architecture](#deployment-architecture)
8. [Development Roadmap](#development-roadmap)
9. [Learning Objectives](#learning-objectives)

---

## Overview

The Billiard Training App is transitioning from a monolithic backend to a distributed microservice architecture. This enables:

- **Scalability**: Independent scaling of high-demand services
- **Team autonomy**: Services owned by focused teams
- **Technology diversity**: Different tech stacks per service if needed
- **Resilience**: Failure isolation and circuit breakers
- **Learning**: Production patterns for distributed systems

### Current State (V1)
```
Monolithic Backend (Spring Boot)
├── Shot Knowledge Base
├── Category Management
├── Book Management
└── (Authentication - stubbed)
```

### Future State (V2+)
```
API Gateway / Load Balancer
├── Billiard Backend (Shot Knowledge)
├── User Profile Service
├── Training Session Service
├── Statistics & Analytics Service
└── External Services (Google Sheets, etc.)
```

---

## Architecture Rationale

### Why These 3 Services?

| Service | Rationale | Alignment |
|---------|-----------|-----------|
| **Training Session Service** | Manages player drills and workouts; high business value; drives analytics | PRD section 23: "Training sessions" |
| **Statistics & Analytics Service** | Tracks performance; enables training recommendations; separate scaling needs | PRD section 23: "Performance statistics" |
| **User Profile Service** | Enables multi-user support; personalization; central identity hub | Future: "Multi-user accounts" |

### Design Principles

1. **Single Responsibility**: Each service owns one business capability
2. **Domain-Driven Design**: Service boundaries follow bounded contexts
3. **Loose Coupling**: Services communicate via events and APIs, not shared databases
4. **Data Isolation**: Each service has its own data store
5. **Async-First**: Prefer async messaging for inter-service communication
6. **Resilience**: Implement timeouts, retries, circuit breakers

---

## Service Catalog

```
┌──────────────────────────────────────────────────────────┐
│                    API Gateway                            │
│            (Route requests to appropriate service)        │
└──────┬───────────────┬──────────────────┬────────────────┘
       │               │                  │
       ▼               ▼                  ▼
┌─────────────────┐┌──────────────┐┌────────────────────┐
│   Billiard      ││   Training   ││    Statistics      │
│   Backend       ││   Session    ││   & Analytics      │
│   Service       ││   Service    ││   Service          │
│                 ││              ││                    │
│ Port: 8080      ││ Port: 8081   ││ Port: 8082         │
└────────┬────────┘└──────┬───────┘└────────┬───────────┘
         │                │                 │
         └────────────────┼─────────────────┘
                          │
                   ┌──────▼──────┐
                   │ User Profile│
                   │  Service    │
                   │ Port: 8083  │
                   └──────┬──────┘
                          │
                   ┌──────▼──────────────┐
                   │  Event Bus/Kafka    │
                   │  (Async messaging)  │
                   └─────────────────────┘
```

---

## Service Details

### 1. Billiard Backend Service (Existing)

**Port**: `8080`
**Language**: Java / Spring Boot 3
**Database**: PostgreSQL / H2

**Responsibilities**:
- Shot knowledge base management
- Category and book administration
- Resource (image/video/PDF) management
- Comment system
- Shot search and filtering

**Owned Entities**:
- `Shot`
- `Category`
- `Book`
- `Resource`
- `Comment`

**APIs**:
```
GET    /api/shots?categoryIds=...&types=...&search=...&page=0&size=25
GET    /api/shots/{id}
POST   /api/shots
PUT    /api/shots/{id}
DELETE /api/shots/{id}

GET    /api/categories
POST   /api/categories
PUT    /api/categories/{id}
DELETE /api/categories/{id}

GET    /api/books
POST   /api/books
PUT    /api/books/{id}
DELETE /api/books/{id}

POST   /api/shots/{id}/comments
PUT    /api/comments/{id}
DELETE /api/comments/{id}

POST   /api/shots/{id}/resources
PUT    /api/resources/{id}
DELETE /api/resources/{id}
```

**Outbound Dependencies**:
- Google Sheets API (data import)
- User Profile Service (user context)

**Inbound Events**: None (producer only)

**Outbound Events**: None initially (future: shot viewed, comment added)

---

### 2. Training Session Service

**Port**: `8081`
**Language**: Java / Spring Boot 3
**Database**: PostgreSQL
**Message Queue**: Kafka / RabbitMQ

**Purpose**: Manage player training sessions, drills, and workout execution

**Responsibilities**:
- Create and manage training sessions
- Assign shots to training sessions (drills)
- Track session progress and state
- Execute drill sequences
- Publish training events for analytics

**Owned Entities**:

```
TrainingSession
├── id (UUID)
├── playerId (FK → User Profile Service)
├── name (e.g., "3-Cushion Drills")
├── description
├── status (PLANNED | ACTIVE | PAUSED | COMPLETED | ARCHIVED)
├── scheduledDate
├── startedAt
├── completedAt
├── totalDrills (count)
├── completedDrills (count)
├── notes
├── createdAt
└── updatedAt

Drill (line item in session)
├── id (UUID)
├── sessionId (FK)
├── shotId (FK → Billiard Backend Service)
├── sequenceNumber
├── targetRepetitions
├── completedRepetitions
├── successCount
├── notes
├── status (PENDING | IN_PROGRESS | COMPLETED | SKIPPED)
└── updatedAt

SessionTemplate (reusable training plans)
├── id (UUID)
├── name (e.g., "Beginner 3-Cushion Progression")
├── description
├── shots (array of {shotId, repetitions})
├── difficulty (EASY | MEDIUM | HARD | EXPERT)
├── estimatedDuration (minutes)
├── ownerId (FK → User Profile Service)
└── isPublic (boolean)
```

**APIs**:

```
# Session Management
POST   /api/training-sessions
       {
         "playerId": "uuid",
         "name": "String",
         "scheduledDate": "ISO-8601",
         "drills": [
           {"shotId": "uuid", "targetRepetitions": 10},
           ...
         ]
       }

GET    /api/training-sessions?playerId=...&status=ACTIVE
GET    /api/training-sessions/{id}
PUT    /api/training-sessions/{id}
DELETE /api/training-sessions/{id}

# Session State Management
PATCH  /api/training-sessions/{id}/status
       {"status": "ACTIVE"}

PATCH  /api/training-sessions/{id}/pause
PATCH  /api/training-sessions/{id}/resume
PATCH  /api/training-sessions/{id}/complete

# Drill Management
POST   /api/training-sessions/{sessionId}/drills
GET    /api/training-sessions/{sessionId}/drills
PUT    /api/drills/{drillId}
PATCH  /api/drills/{drillId}/complete
       {"successCount": 7}

# Templates
GET    /api/session-templates?difficulty=MEDIUM
POST   /api/session-templates
GET    /api/session-templates/{id}
POST   /api/training-sessions/from-template/{templateId}
```

**Service Dependencies**:
- **Billiard Backend Service**: Fetch shot details (synchronous REST calls)
- **User Profile Service**: Validate player exists, get skill level

**Outbound Dependencies**:
- Kafka/RabbitMQ for event publishing

**Inbound Events**: None

**Outbound Events**:
```
SessionCreated
├── sessionId
├── playerId
├── timestamp

SessionStarted
├── sessionId
├── playerId
├── startedAt

DrillCompleted
├── sessionId
├── drillId
├── shotId
├── successCount
├── totalAttempts
├── completedAt

SessionCompleted
├── sessionId
├── playerId
├── totalDrills
├── successRate
├── completedAt
├── duration (minutes)
```

**Learning Focus**:
- State machine pattern (session lifecycle)
- Event-driven communication
- Transaction management across distributed systems
- Saga pattern for multi-step operations

---

### 3. Statistics & Analytics Service

**Port**: `8082`
**Language**: Java / Spring Boot 3
**Database**: PostgreSQL (time-series optimized) + Elasticsearch/InfluxDB (optional)
**Message Queue**: Kafka / RabbitMQ

**Purpose**: Aggregate and analyze player performance data to enable insights and recommendations

**Responsibilities**:
- Consume training events from Training Session Service
- Compute performance metrics and statistics
- Identify player weak shots and strengths
- Generate progress reports
- Enable time-series analysis and trends
- Support data visualization for frontend

**Owned Entities**:

```
PlayerStats (aggregated metrics per player)
├── id
├── playerId (FK → User Profile Service)
├── totalSessionsCompleted
├── totalDrillsCompleted
├── totalDrillsAttempted
├── overallSuccessRate (%)
├── hoursSpent (decimal)
├── lastSessionDate
├── createdAt
└── updatedAt

ShotPerformance (per-shot statistics)
├── id
├── playerId
├── shotId (FK → Billiard Backend Service)
├── totalAttempts
├── successCount
├── successRate (%)
├── averageAttempts (per drill)
├── difficulty (inferred)
├── trend (IMPROVING | STABLE | DECLINING)
├── lastAttemptDate
└── updated

PerformanceTimeSeries (historical data points)
├── id
├── playerId
├── shotId
├── timestamp (hourly/daily buckets)
├── successRate
├── attemptCount
└── notes

PlayerProgress (weekly/monthly summaries)
├── id
├── playerId
├── period (WEEK | MONTH)
├── startDate
├── endDate
├── sessionsCompleted
├── drillsCompleted
├── avgSuccessRate
├── mostImprovedShots
├── weakestShots
└── estimatedSkillLevel
```

**APIs**:

```
# Player Statistics
GET    /api/players/{playerId}/stats
       Returns: {
         "totalSessions": 15,
         "totalDrills": 247,
         "overallSuccessRate": 72.5,
         "hoursSpent": 23.5,
         "estimatedSkillLevel": "INTERMEDIATE"
       }

GET    /api/players/{playerId}/shots/{shotId}/performance
       Returns: {
         "successRate": 65.0,
         "totalAttempts": 100,
         "trend": "IMPROVING",
         "averageSessionDuration": 12.5
       }

# Progress Reports
GET    /api/players/{playerId}/progress?period=MONTH
GET    /api/players/{playerId}/progress?period=WEEK&weeks=4
GET    /api/players/{playerId}/progress/detailed?startDate=2026-01-01&endDate=2026-03-13

# Trending & Analytics
GET    /api/players/{playerId}/weak-shots
       Returns: [
         {"shotId": "...", "successRate": 45.0, "attempts": 50},
         ...
       ]

GET    /api/players/{playerId}/strong-shots
GET    /api/players/{playerId}/skill-trajectory?period=MONTH

# Leaderboards (if multi-player)
GET    /api/leaderboards/success-rate?category=Filotto&limit=10
GET    /api/leaderboards/most-sessions?timeframe=WEEK

# Health Check
GET    /api/health/stats
```

**Service Dependencies**:
- **Billiard Backend Service**: Fetch shot metadata for context
- **User Profile Service**: Player info, skill level confirmation

**Inbound Dependencies**:
- Kafka topics from Training Session Service
  - `training-session.drill-completed`
  - `training-session.session-completed`

**Outbound Events**:
```
PlayerStatsUpdated
├── playerId
├── overallSuccessRate
├── updatedAt

SkillLevelChanged
├── playerId
├── previousLevel (BEGINNER | INTERMEDIATE | ADVANCED | EXPERT)
├── newLevel
├── confidence (0-100)
├── triggeredAt

WeakShotIdentified
├── playerId
├── shotId
├── successRate
├── recommendation (practice more)
```

**Learning Focus**:
- Event stream processing
- Time-series data aggregation
- Complex analytics queries
- CQRS pattern (Command Query Responsibility Segregation)
- Read models for performance
- Elasticsearch/NoSQL alternatives
- Real-time dashboarding foundations

---

### 4. User Profile Service

**Port**: `8083`
**Language**: Java / Spring Boot 3
**Database**: PostgreSQL
**Cache**: Redis (optional, for session management)

**Purpose**: Central identity and profile management for multi-user support

**Responsibilities**:
- Manage player profiles and metadata
- Store player preferences and settings
- Infer and track skill levels
- Manage authentication/authorization
- Provide user context to other services
- Handle profile updates and customization

**Owned Entities**:

```
Player (User Profile)
├── id (UUID)
├── email (unique)
├── firstName
├── lastName
├── displayName
├── bio
├── avatar (URL)
├── skillLevel (BEGINNER | INTERMEDIATE | ADVANCED | EXPERT)
├── skillLevelConfidence (0-100)
├── joinedDate
├── lastActiveDate
├── isActive (boolean)
└── updatedAt

PlayerPreferences
├── id
├── playerId (FK)
├── preferredLanguage
├── defaultPageSize (25 | 50 | 100)
├── defaultCategory (optional)
├── notificationsEnabled
├── darkModeEnabled
├── hideCompletedSessions (boolean)
├── favoriteBooks (array of bookIds)
└── updatedAt

PlayerStats (reference)
├── playerId (FK → Statistics Service)
├── totalSessionsCompleted (read-only)
├── estimatedSkillLevel (read-only, comes from stats)
├── lastSessionDate (read-only)

SkillAssessment (history)
├── id
├── playerId
├── assessedSkillLevel
├── confidence
├── basedOnSessions (count)
├── basedOnShots (count)
├── assessedAt
└── notes

Credentials (Authentication)
├── id
├── playerId (FK)
├── passwordHash
├── jwtSecret
├── lastLogin
├── lastPasswordChange
└── loginAttempts
```

**APIs**:

```
# Player Management
POST   /api/players
       {
         "email": "player@example.com",
         "firstName": "John",
         "lastName": "Smith",
         "password": "..."
       }

GET    /api/players/{id}
PUT    /api/players/{id}
DELETE /api/players/{id}

GET    /api/players/by-email/{email}

# Profile & Preferences
GET    /api/players/{id}/profile
PUT    /api/players/{id}/profile
       {
         "firstName": "John",
         "lastName": "Smith",
         "bio": "3-cushion enthusiast",
         "avatar": "url"
       }

GET    /api/players/{id}/preferences
PUT    /api/players/{id}/preferences
       {
         "preferredLanguage": "en",
         "defaultPageSize": 50,
         "darkModeEnabled": true
       }

# Authentication (temp - will expand)
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout
POST   /api/auth/refresh-token

# Skill Level (read from stats)
GET    /api/players/{id}/skill-level
       Returns: {
         "level": "INTERMEDIATE",
         "confidence": 85,
         "assessedAt": "ISO-8601",
         "basedOnSessions": 15,
         "basedOnShots": 247
       }

# Validation Endpoints (for other services)
GET    /api/players/{id}/validate
       Returns: {status: "ACTIVE" | "INACTIVE" | "DELETED"}

GET    /api/players/batch?ids=uuid1,uuid2,uuid3
```

**Service Dependencies**:
- **Statistics Service**: Fetch stats and infer skill level
- **Billiard Backend Service**: Favorite books/categories

**Inbound Dependencies**:
- Kafka topics from Statistics Service
  - `player-stats.skill-level-changed`

**Outbound Events**:
```
PlayerCreated
├── playerId
├── email
├── createdAt

PlayerProfileUpdated
├── playerId
├── updatedFields (array)
├── updatedAt

SkillLevelInferred
├── playerId
├── newSkillLevel
├── confidence
├── source ("statistics-service")
```

**Learning Focus**:
- User/identity management patterns
- Authentication and JWT tokens
- Authorization and access control
- Service-to-service authentication
- Distributed cache (Redis)
- User preference management
- Multi-tenancy considerations

---

## Inter-Service Communication

### Communication Patterns

#### 1. **Synchronous (REST)**
Used for immediate data needs and strong consistency.

```
Training Session Service
    ↓ (HTTP GET)
Billiard Backend Service
    ← Returns shot details

User Profile Service
    ↓ (HTTP GET)
Statistics Service
    ← Returns player stats
```

**When to use**:
- Fetching shot metadata for drill display
- Validating player existence
- Immediate UI rendering

**Implementation**:
```java
// Training Session Service
@Service
public class ShotServiceClient {
    private RestTemplate restTemplate;

    public ShotDTO getShotById(UUID shotId) {
        return restTemplate.getForObject(
            "http://billiard-backend:8080/api/shots/" + shotId,
            ShotDTO.class
        );
    }
}
```

#### 2. **Asynchronous (Event-Driven)**
Used for loose coupling and eventual consistency.

```
Training Session Service
    ↓ (publishes SessionCompleted event)
Message Bus (Kafka/RabbitMQ)
    ↓
Statistics Service (subscribes, processes)
    ↓ (publishes SkillLevelChanged event)
User Profile Service (subscribes, updates)
```

**When to use**:
- Drill completion triggers analytics
- Skill level changes trigger profile updates
- Fire-and-forget notifications
- Deferred processing

**Event Schema Example**:
```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "training-session.drill-completed",
  "timestamp": "2026-03-13T14:30:00Z",
  "version": "1.0",
  "source": "training-session-service",
  "data": {
    "sessionId": "uuid",
    "drillId": "uuid",
    "shotId": "uuid",
    "playerId": "uuid",
    "successCount": 7,
    "totalAttempts": 10,
    "duration": 45,
    "completedAt": "2026-03-13T14:30:00Z"
  }
}
```

**Implementation**:
```java
// Training Session Service - Event Producer
@Service
public class TrainingSessionService {
    private KafkaTemplate<String, DrillCompletedEvent> kafkaTemplate;

    public void completeDrill(Drill drill) {
        // ... business logic ...

        kafkaTemplate.send("training-session.drill-completed",
            new DrillCompletedEvent(...));
    }
}

// Statistics Service - Event Consumer
@Component
public class DrillCompletedEventListener {
    @KafkaListener(topics = "training-session.drill-completed")
    public void handle(DrillCompletedEvent event) {
        // Update shot performance statistics
        updateShotStats(event.getShotId(), event.getPlayerId());
    }
}
```

### Service Call Graph

```
External Clients (Web/Mobile)
    ↓ (HTTP)
API Gateway / Load Balancer
    ↓
┌───────────────────────────────────────────────────┐
│                                                   │
▼                    ▼                     ▼         ▼
Billiard Backend    Training Session    User Profile
  Service             Service            Service
  (8080)              (8081)             (8083)
                        ↓                  ↓
                        ├─→ Billiard Backend (validate shots)
                        ├─→ User Profile (validate player)
                        │
                        └→ Kafka Bus
                           ↓
                    Statistics Service
                      (8082)
                        ├─→ Billiard Backend (shot metadata)
                        └─→ User Profile (updates stats)
```

### Resilience Patterns

#### Circuit Breaker (Hystrix/Resilience4j)
```java
@Service
public class ShotServiceClient {
    @CircuitBreaker(name = "shotService",
        fallbackMethod = "fallbackGetShot")
    public ShotDTO getShotById(UUID shotId) {
        return restTemplate.getForObject(...);
    }

    public ShotDTO fallbackGetShot(UUID shotId, Exception e) {
        // Return cached data or empty response
        return new ShotDTO(); // minimal data
    }
}
```

#### Timeout & Retry
```java
@Retry(name = "shotServiceRetry")
@Timeout(duration = "5s")
public ShotDTO getShotById(UUID shotId) {
    return restTemplate.getForObject(...);
}
```

---

## Data Consistency Patterns

### Problem: Distributed Transactions

When Training Session Service completes a drill and Statistics Service needs to update metrics, we must handle:
- Network failures
- Partial failures
- Race conditions
- Eventual consistency

### Solutions

#### 1. **Saga Pattern** (Choreography)
Service-to-service coordination via events.

```
Training Session Service completes drill
    ↓ publishes DrillCompleted event
Statistics Service receives event
    ↓ updates shot stats, publishes StatisticsUpdated event
User Profile Service receives event
    ↓ updates skill level
Failure? → Event replay from log
```

**Pros**: Simple, decoupled
**Cons**: Hard to debug, no central visibility

#### 2. **Saga Pattern** (Orchestration)
Central orchestrator coordinates steps.

```
Training Session Service completes drill
    ↓
Saga Orchestrator (in Training Session Service)
    ├→ Step 1: Update local drill status
    ├→ Step 2: Send UpdateStats command to Statistics Service
    ├→ Step 3: Wait for acknowledgment
    └→ Step 4: Send UpdateProfile command to User Profile Service
Failure? → Compensating transactions (rollback)
```

**Pros**: Observable, failure handling clear
**Cons**: Orchestrator becomes bottleneck

#### 3. **Event Sourcing** (Long-term)
Store all state changes as immutable events.

```
Instead of: Player state (skill_level = "INTERMEDIATE")
Store:      [Event: PlayerCreated(..),
             Event: SessionCompleted(sessionId=X),
             Event: DrillCompleted(drillId=Y, success=7/10),
             Event: SkillLevelInferred(newLevel=INTERMEDIATE)]
```

**Current Phase**: Not required for V2, but design with this in mind

### Recommendation for V2

Use **Event-based Choreography** with message log (Kafka):
- Simple to implement
- Services remain decoupled
- Kafka handles durability/replay
- Add orchestration in V3 if needed

---

## Deployment Architecture

### Local Development
```
Docker Compose with 5 containers:

1. billiard-backend (Spring Boot)
   - Port 8080
   - H2 or MySQL
   - Volume: ./data

2. training-session (Spring Boot)
   - Port 8081
   - PostgreSQL
   - Volume: ./data

3. statistics (Spring Boot)
   - Port 8082
   - PostgreSQL
   - Volume: ./data

4. user-profile (Spring Boot)
   - Port 8083
   - PostgreSQL
   - Volume: ./data

5. kafka / message-broker
   - Port 9092
   - Zookeeper on 2181
   - Volume: ./kafka-data

6. postgres (shared or per-service)
   - Port 5432
   - Volume: ./postgres-data

Optional:
7. redis (caching)
   - Port 6379
8. pgadmin (database UI)
   - Port 5050
```

### docker-compose.yml (Example)
```yaml
version: '3.9'

services:
  # Message Broker
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
    ports:
      - "9092:9092"
    volumes:
      - kafka-data:/var/lib/kafka/data

  # Shared Database
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: billiard
      POSTGRES_PASSWORD: billiard123
      POSTGRES_MULTIPLE_DATABASES: "billiard_backend,training_session,statistics,user_profile"
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./scripts/init-databases.sh:/docker-entrypoint-initdb.d/init-databases.sh

  # Microservices
  billiard-backend:
    build:
      context: ./billiard-training-app-be
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    depends_on:
      - postgres
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/billiard_backend
      SPRING_DATASOURCE_USERNAME: billiard
      SPRING_DATASOURCE_PASSWORD: billiard123
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate
      SPRING_FLYWAY_ENABLED: "true"

  training-session:
    build:
      context: ./training-session-service
      dockerfile: Dockerfile
    ports:
      - "8081:8081"
    depends_on:
      - postgres
      - kafka
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/training_session
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate

  statistics:
    build:
      context: ./statistics-service
      dockerfile: Dockerfile
    ports:
      - "8082:8082"
    depends_on:
      - postgres
      - kafka
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/statistics
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate

  user-profile:
    build:
      context: ./user-profile-service
      dockerfile: Dockerfile
    ports:
      - "8083:8083"
    depends_on:
      - postgres
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/user_profile
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate

volumes:
  postgres-data:
  kafka-data:
```

### Production Deployment (Future)

```
AWS/Kubernetes:

├── API Gateway (AWS API Gateway / Nginx)
│
├── EKS / Kubernetes Cluster
│   ├── Pod: billiard-backend (replicas: 2-3)
│   ├── Pod: training-session (replicas: 2)
│   ├── Pod: statistics (replicas: 2)
│   ├── Pod: user-profile (replicas: 2)
│   └── Pod: kafka (StatefulSet, replicas: 3)
│
├── RDS PostgreSQL (Multi-AZ)
│
├── ElastiCache Redis (optional, for caching)
│
└── CloudWatch / ELK Stack (logging & monitoring)
```

---

## Development Roadmap

### Phase 1: Foundation (Weeks 1-2)
**Goal**: Core service scaffolding

- [ ] Create 3 new Spring Boot projects
- [ ] Set up Kafka/RabbitMQ locally
- [ ] Create docker-compose for local development
- [ ] Implement basic APIs (CRUD endpoints)
- [ ] Set up database schemas per service

**Deliverables**:
- 3 running microservices
- Docker Compose stack
- Basic REST APIs
- Database migrations (Flyway)

### Phase 2: Communication (Weeks 3-4)
**Goal**: Inter-service communication

- [ ] Implement REST clients in services
- [ ] Implement Kafka producers/consumers
- [ ] Create event schemas and documentation
- [ ] Add circuit breaker patterns
- [ ] Test service-to-service calls

**Deliverables**:
- Synchronous REST integration tests
- Event-driven integration tests
- API documentation
- Resilience patterns

### Phase 3: Business Logic (Weeks 5-6)
**Goal**: Core domain logic

- [ ] Training Session Service: state machine, drill management
- [ ] Statistics Service: aggregation logic, skill inference
- [ ] User Profile Service: auth, preferences, profile management
- [ ] Event processing pipelines
- [ ] Integration with Billiard Backend Service

**Deliverables**:
- Fully functional services
- End-to-end workflows
- Integration tests

### Phase 4: Production Ready (Weeks 7-8)
**Goal**: Resilience and observability

- [ ] Add logging (SLF4J + ELK Stack)
- [ ] Add distributed tracing (Jaeger/Zipkin)
- [ ] Add metrics (Prometheus/Micrometer)
- [ ] Add health checks and readiness probes
- [ ] Load testing
- [ ] Security hardening (JWT, service auth)

**Deliverables**:
- Monitoring dashboard
- Load test results
- Security audit
- Deployment playbooks

### Phase 5: Frontend Integration (Weeks 9+)
**Goal**: Full-stack integration

- [ ] Update Angular frontend for multi-service calls
- [ ] Implement user authentication UI
- [ ] Implement training session UI
- [ ] Implement analytics dashboard
- [ ] Performance optimization

**Deliverables**:
- Full-stack application
- User acceptance testing

---

## Learning Objectives

### For Java/Spring Boot Developers

**Training Session Service**:
- ✅ State machine patterns
- ✅ Event-driven architecture
- ✅ Async processing with Spring
- ✅ Entity relationships and inheritance
- ✅ Transaction management in distributed systems

**Statistics Service**:
- ✅ Time-series data handling
- ✅ Aggregation and rollup patterns
- ✅ CQRS (Command Query Responsibility Segregation)
- ✅ Read models and materialized views
- ✅ Performance optimization with indexing
- ✅ Handling at-least-once delivery semantics

**User Profile Service**:
- ✅ User/identity management
- ✅ Authentication (JWT)
- ✅ Authorization and access control
- ✅ Distributed caching (Redis)
- ✅ Service-to-service authentication

### Architecture Patterns

- ✅ Microservice decomposition
- ✅ Service boundaries and bounded contexts
- ✅ Synchronous vs asynchronous communication
- ✅ Event sourcing concepts
- ✅ Saga pattern for distributed transactions
- ✅ Circuit breaker and resilience patterns
- ✅ API versioning
- ✅ Data consistency in distributed systems
- ✅ Service discovery
- ✅ Containerization (Docker)
- ✅ Container orchestration (Docker Compose → Kubernetes)

### DevOps & Operations

- ✅ Docker and Docker Compose
- ✅ Database migrations (Flyway/Liquibase)
- ✅ Environment-specific configurations
- ✅ Logging and centralized log aggregation
- ✅ Distributed tracing
- ✅ Metrics and monitoring
- ✅ Health checks and readiness probes
- ✅ Load testing

---

## Tools & Technologies

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Language** | Java 17+ | Primary development |
| **Framework** | Spring Boot 3 | Application framework |
| **Database** | PostgreSQL | Persistent storage |
| **Message Bus** | Kafka / RabbitMQ | Async communication |
| **Caching** | Redis | Session & data caching |
| **Monitoring** | Prometheus | Metrics collection |
| **Logging** | ELK Stack | Centralized logging |
| **Tracing** | Jaeger / Zipkin | Distributed tracing |
| **Container** | Docker | Containerization |
| **Orchestration** | Docker Compose | Local; Kubernetes for prod |
| **IaC** | Terraform | Infrastructure as code |
| **Testing** | JUnit 5, Testcontainers | Unit/integration tests |

---

## API Gateway (Future Enhancement)

Route requests to appropriate microservices:

```
Request: GET /api/shots/123
    ↓
API Gateway (Kong / AWS API Gateway)
    ↓
Route to: billiard-backend:8080/api/shots/123

Request: POST /api/training-sessions
    ↓
API Gateway
    ↓
Route to: training-session:8081/api/training-sessions

Request: GET /api/players/abc/stats
    ↓
API Gateway
    ↓
Route to: user-profile:8083/api/players/abc
         + Call statistics:8082 for stats
```

**Benefits**:
- Single entry point for clients
- Service discovery abstraction
- Rate limiting
- Request/response transformation
- Authentication/authorization at gateway level

---

## Security Considerations

### Service-to-Service Communication
```java
@Configuration
public class SecurityConfig {
    // API Key authentication
    // JWT token validation
    // mTLS (mutual TLS) for service-to-service
    // Service discovery with authentication
}
```

### Per-Service Authentication
- Training Session Service: Validate playerId from User Profile Service
- Statistics Service: Validate player access to their own stats
- Billiard Backend Service: Admin checks for write operations

### Data Isolation
- Each service has own database (no shared DB)
- Each service validates cross-service references
- Async events don't leak sensitive data

---

## Monitoring & Observability

### Key Metrics per Service

**Training Session Service**:
- Sessions created/day
- Average session duration
- Drill completion rate
- Latency: GET /training-sessions

**Statistics Service**:
- Stats calculated/min
- Query latency
- Event lag (time to process DrillCompleted)
- Skill level inference accuracy

**User Profile Service**:
- User registrations/day
- Login success rate
- Profile update latency
- Cache hit rate

### Logging Strategy
```
Level: DEBUG in dev, INFO in prod
Format: JSON with correlation IDs
Sample fields:
  - timestamp
  - service-name
  - request-id (tracing)
  - user-id
  - method, endpoint, status_code
  - response-time
  - error (if applicable)
```

---

## References & Further Reading

### Books
- "Building Microservices" by Sam Newman
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Patterns of Enterprise Application Architecture" by Martin Fowler

### Patterns
- [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)

### Frameworks & Libraries
- Spring Cloud (service discovery, config, circuit breaker)
- Resilience4j (fault tolerance)
- Testcontainers (integration testing)
- Spring Cloud Stream (event streaming)

---

## Appendix: Service Checklist

### When Creating a New Service

- [ ] Create Spring Boot project scaffold
- [ ] Configure application.properties / application.yml
- [ ] Add dockerfile and docker-compose.yml entries
- [ ] Create database migration scripts (Flyway)
- [ ] Define JPA entities and repositories
- [ ] Create REST controllers with proper error handling
- [ ] Add OpenAPI/Swagger documentation
- [ ] Implement Kafka producer/consumer (if needed)
- [ ] Add unit tests (JUnit 5)
- [ ] Add integration tests (Testcontainers)
- [ ] Add logging with SLF4J
- [ ] Add health checks (@Endpoint)
- [ ] Add metrics (@Timed, @Counted)
- [ ] Document service in this file
- [ ] Add service to docker-compose.yml
- [ ] Test end-to-end with other services

---

## Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2026-03-13 | 1.0 | Initial architecture design |

