# Jira-like Ticketing System Design
## System Design Interview Solution

### Table of Contents
1. [Requirements Clarification](#requirements-clarification)
2. [Scale Estimation](#scale-estimation)
3. [High-Level Architecture](#high-level-architecture)
4. [Component Design](#component-design)
5. [Data Model](#data-model)
6. [API Design](#api-design)
7. [Security & Access Control](#security--access-control)
8. [Scalability Considerations](#scalability-considerations)
9. [Trade-offs & Discussion Points](#trade-offs--discussion-points)

---

## Requirements Clarification

### Functional Requirements
- **Ticket Management**: Create, read, update, delete tickets/issues
- **User Management**: User registration, authentication, role-based access
- **Project Management**: Multiple projects with different workflows
- **Workflow Management**: Customizable status transitions and workflows
- **Search & Filtering**: Advanced search with JQL (Jira Query Language)
- **Comments & Attachments**: Add comments and file attachments to tickets
- **Notifications**: Real-time and email notifications
- **Reporting**: Analytics and reporting dashboards
- **Bulk Operations**: Batch operations on multiple tickets

### Non-Functional Requirements
- **Availability**: 99.9% uptime
- **Latency**: < 200ms for read operations, < 500ms for write operations
- **Scalability**: Handle thousands of users and millions of tickets
- **Consistency**: Strong consistency for critical operations, eventual consistency for others
- **Security**: OAuth2/JWT authentication, role-based access control
- **Audit**: Complete audit trail for all operations

---

## Scale Estimation

### Traffic Estimates (Typical Enterprise)
- **Users**: 20,000 registered users
- **Tickets**: 2 million tickets across all projects
- **Comments**: 10 million comments
- **Requests**: 2,000 requests/second (peak)
- **Storage**: 65GB of data

### Traffic Estimates (Large Enterprise)
- **Users**: 200,000 registered users
- **Tickets**: 20 million tickets across all projects
- **Comments**: 100 million comments
- **Requests**: 10,000 requests/second (peak)
- **Storage**: 650GB of data

### Traffic Estimates (SaaS/Multi-tenant)
- **Users**: 1 million+ users across multiple companies
- **Tickets**: 50 million+ tickets
- **Comments**: 500 million+ comments
- **Requests**: 25,000+ requests/second (peak)
- **Storage**: 2TB+ of data

### Storage Calculations (Typical Enterprise - 20K users, 2M tickets)
- **Ticket metadata**: ~2KB per ticket = 4GB
- **Comments**: ~500B per comment = 10GB (5 comments per ticket average)
- **Attachments**: ~50GB (25MB per ticket average)
- **User data**: ~1GB
- **Total**: ~65GB

### Storage Calculations (Large Enterprise - 200K users, 20M tickets)
- **Ticket metadata**: ~2KB per ticket = 40GB
- **Comments**: ~500B per comment = 100GB
- **Attachments**: ~500GB
- **User data**: ~10GB
- **Total**: ~650GB

### Performance Requirements
- **Read operations**: < 200ms (95th percentile)
- **Write operations**: < 500ms (95th percentile)
- **Search operations**: < 1 second
- **Concurrent users**: 2,000+ (typical), 20,000+ (large enterprise)

### Scale Context for Interview
**Typical Enterprise Scenario:**
- Single company with internal Jira deployment
- 20,000 employees using the system
- 2 million tickets across various projects
- Peak usage during business hours

**Large Enterprise Scenario:**
- Fortune 500 company with global teams
- 200,000 employees across multiple locations
- 20 million tickets with complex workflows
- 24/7 usage across different time zones

**SaaS/Multi-tenant Scenario:**
- Multiple companies using the same platform
- Each company: 1K-50K users
- Total platform: 1M+ users across all companies
- Similar to Atlassian's Jira Cloud

---

## High-Level Architecture

### Architecture Overview

```markdown:jira-system-design-final.md
┌─────────────────────────────────────────────────────────────┐
│                    Client Applications                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │   Web Client│ │ Mobile Client│ │ Desktop App │         │
│  │(React/Angular)│ │(React Native)│ │  (JavaFX)   │         │
│  │             │ │             │ │             │         │
│  │ • Browser   │ │ • iOS/Android│ │ • Windows   │         │
│  │ • Rich UI   │ │ • Native UI  │ │ • macOS     │         │
│  │ • Real-time │ │ • Offline    │ │ • Linux     │         │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer (AWS ALB)                 │
│  • Health checks for API Gateway instances                │
│  • SSL/TLS termination                                    │
│  • Basic traffic distribution                             │
│  • Connection draining during deployments                 │
│  • Geographic routing (if multi-region)                  │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway                              │
│  • Authentication & Authorization                          │
│  • Rate Limiting                                          │
│  • Request Routing (to BFF services)                      │
│  • CORS Management                                        │
│  • Request/Response Transformation                        │
│  • Circuit Breaking                                       │
└─────────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────┐
│                    BFF Layer (Backend for Frontend)        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │  Web BFF    │ │ Mobile BFF  │ │ Desktop BFF │         │
│  │ (Rich data, │ │ (Minimal    │ │ (Advanced   │         │
│  │  real-time) │ │  data,      │ │  features,  │         │
│  │             │ │  offline)   │ │  bulk ops)  │         │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────┐
│                    Microservices Layer                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │Ticket Service│ │User Service │ │Project Service│        │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │Search Service│ │Notification │ │Workflow     │         │
│  │             │ │Service      │ │Service      │         │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │Authentication│ │Authorization│ │Audit        │         │
│  │Service      │ │Service      │ │Service      │         │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   PostgreSQL    │  │     Redis       │  │  Elasticsearch  │
│   (Primary DB)  │  │   (Cache)       │  │   (Search)      │
└─────────────────┘  └─────────────────┘  └─────────────────┘
                │               │               │
                ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────┐
│                    Infrastructure Layer                     │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │   AWS S3    │ │   Apache    │ │   Prometheus│         │
│  │ (File Storage)│ │   Kafka     │ │  (Monitoring)│        │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### Why BFF Pattern?
**Problem with Direct Client-to-Service Communication:**
- **Security**: Exposes internal services to internet
- **Complexity**: Clients need to know about all microservices
- **Performance**: Multiple round trips for single user action
- **Maintenance**: Each client needs service discovery logic

**BFF Benefits:**
- **Security**: Single entry point with centralized security
- **Performance**: Data aggregation reduces network calls
- **Client Optimization**: Tailored responses for each client type
- **Maintenance**: Clients only know about BFF services

---

## Component Design

### 1. API Gateway
**Responsibilities:**
- Authentication and authorization
- Rate limiting and throttling
- Request routing and load balancing
- CORS management
- Request/response transformation
- Circuit breaking

**Technology Choice:** Spring Cloud Gateway or AWS API Gateway

### 2. BFF Services
**Web BFF:**
- Rich data aggregation for web UI
- Real-time updates via WebSocket
- Progressive loading for large datasets
- Browser caching optimization

**Mobile BFF:**
- Minimal data payload for mobile performance
- Offline support with local storage
- Push notification integration
- Battery optimization

**Desktop BFF:**
- Advanced features like bulk operations
- File system integration
- Background synchronization
- Native UI components

### 3. Core Microservices

#### Ticket Service
**Responsibilities:**
- CRUD operations for tickets
- Status transitions and workflow validation
- Business logic enforcement
- Event publishing for changes

**Data Storage:** PostgreSQL (primary), Redis (cache)

#### User Service
**Responsibilities:**
- User management and authentication
- Role-based access control (RBAC)
- Profile management
- Session management

**Data Storage:** PostgreSQL, Redis (sessions)

#### Project Service
**Responsibilities:**
- Project CRUD operations
- Project settings and configuration
- User permissions and project membership
- Project templates

#### Search Service
**Responsibilities:**
- Full-text search implementation
- Advanced filtering with JQL
- Search suggestions and autocomplete
- Search analytics

**Data Storage:** Elasticsearch

#### Notification Service
**Responsibilities:**
- Email notifications
- In-app notifications
- Push notifications
- Notification preferences

#### Workflow Service
**Responsibilities:**
- Custom workflow definitions
- Status transition rules
- Workflow validation
- Workflow templates

### 4. Data Storage Layer

#### Primary Database (PostgreSQL)
- **ACID compliance** for critical operations
- **Relational data** for tickets, users, projects
- **Complex queries** for reporting and analytics
- **Data consistency** for business rules

#### Cache Layer (Redis)
- **Session storage** for user sessions
- **Frequently accessed data** (tickets, user profiles)
- **Rate limiting** counters
- **Real-time features** (notifications, live updates)

#### Search Engine (Elasticsearch)
- **Full-text search** for tickets and comments
- **Advanced filtering** with complex queries
- **Search suggestions** and autocomplete
- **Search analytics** and insights

#### File Storage (AWS S3)
- **Attachments** (images, documents, files)
- **CDN integration** for global distribution
- **Backup and archival** storage
- **Cost-effective** for large files

---

## Data Model

### Core Entities

#### User
- **id**: Unique identifier
- **email**: Email address (unique)
- **username**: Username (unique)
- **firstName, lastName**: User names
- **role**: User role (USER, ADMIN, PROJECT_LEAD)
- **isActive**: Account status
- **createdAt, updatedAt**: Timestamps

#### Project
- **id**: Unique identifier
- **key**: Project key (e.g., "PROJ")
- **name**: Project name
- **description**: Project description
- **ownerId**: Project owner
- **leadId**: Project lead
- **isActive**: Project status
- **createdAt, updatedAt**: Timestamps

#### Ticket
- **id**: Unique identifier
- **key**: Ticket key (e.g., "PROJ-123")
- **projectId**: Associated project
- **title**: Ticket title
- **description**: Ticket description
- **status**: Current status (TO_DO, IN_PROGRESS, DONE)
- **priority**: Priority level (LOW, MEDIUM, HIGH, CRITICAL)
- **type**: Ticket type (BUG, TASK, STORY, EPIC)
- **assigneeId**: Assigned user
- **reporterId**: User who created the ticket
- **createdAt, updatedAt, resolvedAt**: Timestamps

#### Comment
- **id**: Unique identifier
- **ticketId**: Associated ticket
- **userId**: Comment author
- **content**: Comment text
- **createdAt, updatedAt**: Timestamps

#### Attachment
- **id**: Unique identifier
- **ticketId**: Associated ticket
- **filename**: Stored filename
- **originalFilename**: Original filename
- **contentType**: MIME type
- **fileSize**: File size in bytes
- **s3Key**: S3 storage key
- **uploadedBy**: User who uploaded
- **createdAt**: Timestamp

### Relationships
- **User** → **Ticket** (1:N) - User can create multiple tickets
- **Project** → **Ticket** (1:N) - Project can have multiple tickets
- **Ticket** → **Comment** (1:N) - Ticket can have multiple comments
- **Ticket** → **Attachment** (1:N) - Ticket can have multiple attachments
- **User** → **Project** (M:N) - Users can be members of multiple projects

### Database Schema

```sql
-- Users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'USER',
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Projects table
CREATE TABLE projects (
    id BIGSERIAL PRIMARY KEY,
    key VARCHAR(10) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id BIGINT REFERENCES users(id),
    lead_id BIGINT REFERENCES users(id),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Project members table
CREATE TABLE project_members (
    id BIGSERIAL PRIMARY KEY,
    project_id BIGINT REFERENCES projects(id),
    user_id BIGINT REFERENCES users(id),
    role VARCHAR(50) NOT NULL DEFAULT 'MEMBER',
    joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(project_id, user_id)
);

-- Tickets table
CREATE TABLE tickets (
    id BIGSERIAL PRIMARY KEY,
    key VARCHAR(50) UNIQUE NOT NULL,
    project_id BIGINT REFERENCES projects(id),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    status VARCHAR(50) NOT NULL DEFAULT 'TO_DO',
    priority VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',
    type VARCHAR(50) NOT NULL DEFAULT 'TASK',
    assignee_id BIGINT REFERENCES users(id),
    reporter_id BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    resolved_at TIMESTAMP,
    INDEX idx_project_status (project_id, status),
    INDEX idx_assignee (assignee_id),
    INDEX idx_reporter (reporter_id),
    INDEX idx_created_at (created_at)
);

-- Comments table
CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT REFERENCES tickets(id),
    user_id BIGINT REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_ticket_id (ticket_id)
);

-- Attachments table
CREATE TABLE attachments (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT REFERENCES tickets(id),
    filename VARCHAR(255) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    content_type VARCHAR(100) NOT NULL,
    file_size BIGINT NOT NULL,
    s3_key VARCHAR(500) NOT NULL,
    uploaded_by BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Watchers table
CREATE TABLE watchers (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT REFERENCES tickets(id),
    user_id BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(ticket_id, user_id)
);

-- Audit log table
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    entity_type VARCHAR(50) NOT NULL,
    entity_id BIGINT NOT NULL,
    action VARCHAR(50) NOT NULL,
    user_id BIGINT REFERENCES users(id),
    old_values JSONB,
    new_values JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_entity (entity_type, entity_id),
    INDEX idx_created_at (created_at)
);
```

---

## API Design

### RESTful API Structure

#### Authentication Endpoints
```http
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
POST   /api/v1/auth/register
```

#### User Management
```http
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/{userId}
PUT    /api/v1/users/{userId}
DELETE /api/v1/users/{userId}
GET    /api/v1/users/{userId}/tickets
GET    /api/v1/users/{userId}/assigned-tickets
GET    /api/v1/users/{userId}/reported-tickets
GET    /api/v1/users/{userId}/watched-tickets
```

#### Project Management
```http
GET    /api/v1/projects
POST   /api/v1/projects
GET    /api/v1/projects/{projectId}
PUT    /api/v1/projects/{projectId}
DELETE /api/v1/projects/{projectId}
GET    /api/v1/projects/{projectId}/members
POST   /api/v1/projects/{projectId}/members
DELETE /api/v1/projects/{projectId}/members/{userId}
GET    /api/v1/projects/{projectId}/settings
PUT    /api/v1/projects/{projectId}/settings
```

#### Ticket Management
```http
GET    /api/v1/projects/{projectId}/tickets
POST   /api/v1/projects/{projectId}/tickets
GET    /api/v1/tickets/{ticketId}
PUT    /api/v1/tickets/{ticketId}
DELETE /api/v1/tickets/{ticketId}
PATCH  /api/v1/tickets/{ticketId}/status
PATCH  /api/v1/tickets/{ticketId}/assignee
PATCH  /api/v1/tickets/{ticketId}/priority
POST   /api/v1/tickets/{ticketId}/watchers
DELETE /api/v1/tickets/{ticketId}/watchers/{userId}
```

#### Comments & Attachments
```http
GET    /api/v1/tickets/{ticketId}/comments
POST   /api/v1/tickets/{ticketId}/comments
PUT    /api/v1/tickets/{ticketId}/comments/{commentId}
DELETE /api/v1/tickets/{ticketId}/comments/{commentId}
GET    /api/v1/tickets/{ticketId}/attachments
POST   /api/v1/tickets/{ticketId}/attachments
DELETE /api/v1/tickets/{ticketId}/attachments/{attachmentId}
GET    /api/v1/attachments/{attachmentId}/download
```

#### Search & Filtering
```http
GET /api/v1/search/tickets?q={query}&project={projectId}&status={status}&assignee={userId}&priority={priority}&created_after={date}&created_before={date}
GET /api/v1/search/advanced?jql={jql_query}
GET /api/v1/search/suggestions?q={partial_query}
```

#### Workflow & Transitions
```http
GET    /api/v1/tickets/{ticketId}/transitions
POST   /api/v1/tickets/{ticketId}/transitions
GET    /api/v1/projects/{projectId}/workflows
POST   /api/v1/projects/{projectId}/workflows
PUT    /api/v1/projects/{projectId}/workflows/{workflowId}
```

#### Notifications
```http
GET    /api/v1/notifications
PUT    /api/v1/notifications/{notificationId}/read
PUT    /api/v1/notifications/read-all
GET    /api/v1/users/{userId}/notification-settings
PUT    /api/v1/users/{userId}/notification-settings
```

#### Reporting & Analytics
```http
GET /api/v1/reports/velocity?project={projectId}&sprint={sprintId}
GET /api/v1/reports/burndown?project={projectId}&sprint={sprintId}
GET /api/v1/reports/assignee-workload?project={projectId}
GET /api/v1/reports/issue-distribution?project={projectId}&group_by={field}
GET /api/v1/reports/time-tracking?project={projectId}&user={userId}
```

### BFF-Specific Endpoints

#### Web BFF
```http
GET /api/v1/web/dashboard
GET /api/v1/web/tickets/{ticketId}/detail
GET /api/v1/web/projects/{projectId}/overview
GET /api/v1/web/reports/summary
```

#### Mobile BFF
```http
GET /api/v1/mobile/tickets
GET /api/v1/mobile/notifications
GET /api/v1/mobile/profile
GET /api/v1/mobile/quick-actions
```

#### Desktop BFF
```http
GET /api/v1/desktop/advanced-search
POST /api/v1/desktop/bulk-operations
GET /api/v1/desktop/export
GET /api/v1/desktop/analytics
```

### API Design Principles
- **RESTful conventions** for resource management
- **Consistent error responses** across all endpoints
- **Pagination** for list endpoints
- **Filtering and sorting** capabilities
- **Versioning** strategy for API evolution
- **Rate limiting** per client type

### Request/Response Examples

#### Create Ticket Request
```json
POST /api/v1/projects/PROJ-123/tickets
{
  "title": "Fix login bug",
  "description": "Users cannot login with correct credentials",
  "priority": "HIGH",
  "type": "BUG",
  "assigneeId": "user123",
  "labels": ["frontend", "authentication"],
  "components": ["login-module"]
}
```

#### Create Ticket Response
```json
{
  "id": "PROJ-123-456",
  "key": "PROJ-123-456",
  "title": "Fix login bug",
  "description": "Users cannot login with correct credentials",
  "status": "TO_DO",
  "priority": "HIGH",
  "type": "BUG",
  "assignee": {
    "id": "user123",
    "name": "John Doe",
    "email": "john@example.com"
  },
  "reporter": {
    "id": "user456",
    "name": "Jane Smith",
    "email": "jane@example.com"
  },
  "project": {
    "id": "proj123",
    "key": "PROJ",
    "name": "Web Application"
  },
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

#### Search Tickets Request
```http
GET /api/v1/search/tickets?project=PROJ-123&status=IN_PROGRESS&assignee=user123&priority=HIGH&created_after=2024-01-01
```

#### Search Tickets Response
```json
{
  "tickets": [
    {
      "id": "PROJ-123-456",
      "key": "PROJ-123-456",
      "title": "Fix login bug",
      "status": "IN_PROGRESS",
      "priority": "HIGH",
      "assignee": {
        "id": "user123",
        "name": "John Doe"
      },
      "updatedAt": "2024-01-15T14:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "size": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

---

## Security & Access Control

### Authentication Strategy
- **JWT tokens** for stateless authentication
- **OAuth2** for third-party integrations
- **Multi-factor authentication** for sensitive operations
- **Session management** for web clients

### Authorization Model
- **Role-based access control (RBAC)**
  - **USER**: Basic ticket operations
  - **PROJECT_LEAD**: Project management
  - **ADMIN**: System administration
- **Resource-level permissions** for fine-grained control
- **Project-based access** for team isolation

### Security Measures
- **HTTPS/TLS** for all communications
- **Input validation** and sanitization
- **SQL injection prevention** with parameterized queries
- **XSS protection** with content security policies
- **CSRF protection** for web applications
- **Rate limiting** to prevent abuse

### Data Protection
- **Encryption at rest** for sensitive data
- **Encryption in transit** for all communications
- **Audit logging** for compliance
- **Data retention policies** for GDPR compliance
- **Backup encryption** for disaster recovery

---

## Scalability Considerations

### Horizontal Scaling
- **Stateless services** for easy scaling
- **Load balancing** across multiple instances
- **Database read replicas** for read-heavy workloads
- **Sharding strategy** for large datasets
- **CDN** for static content delivery

### Performance Optimization
- **Caching strategy** (Redis for frequently accessed data)
- **Database indexing** on frequently queried columns
- **Connection pooling** for database connections
- **Asynchronous processing** for non-critical operations
- **Background job processing** for heavy operations

### Data Partitioning
- **Project-based sharding** for ticket data
- **Time-based partitioning** for audit logs
- **User-based partitioning** for user-specific data
- **Geographic partitioning** for global deployments

### Caching Strategy
- **L1 Cache (Application)**: In-memory caching for frequently accessed data
- **L2 Cache (Redis)**: Distributed caching for shared data
- **CDN**: Static asset caching for global performance
- **Cache invalidation**: Event-driven cache invalidation

---

## Trade-offs & Discussion Points

### Consistency vs Availability
**Strong Consistency:**
- **Pros**: Data integrity, ACID compliance
- **Cons**: Higher latency, reduced availability
- **Use Cases**: Ticket status changes, user permissions

**Eventual Consistency:**
- **Pros**: Higher availability, better performance
- **Cons**: Potential data staleness, complex conflict resolution
- **Use Cases**: Search indexes, analytics data

### Monolith vs Microservices
**Monolith:**
- **Pros**: Simpler development, easier testing, single deployment
- **Cons**: Harder to scale, technology lock-in, deployment risk

**Microservices:**
- **Pros**: Independent scaling, technology diversity, fault isolation
- **Cons**: Distributed system complexity, network latency, data consistency

### Database Choices
**PostgreSQL:**
- **Pros**: ACID compliance, complex queries, mature ecosystem
- **Cons**: Scaling challenges, higher cost for large scale

**NoSQL (MongoDB/Cassandra):**
- **Pros**: Horizontal scaling, flexible schema, high availability
- **Cons**: Eventual consistency, complex transactions, learning curve

### Caching Strategy
**Write-through:**
- **Pros**: Data consistency, simple invalidation
- **Cons**: Higher write latency, cache misses

**Write-behind:**
- **Pros**: Better write performance, reduced database load
- **Cons**: Potential data loss, complex recovery

### Message Queue vs Direct Communication
**Synchronous:**
- **Pros**: Immediate feedback, simpler error handling
- **Cons**: Higher latency, tight coupling

**Asynchronous:**
- **Pros**: Better performance, loose coupling, fault tolerance
- **Cons**: Eventual consistency, complex error handling

---

## Interview Discussion Points

### Architecture Decisions
1. **Why BFF pattern?** Client-specific optimizations and security
2. **Why microservices?** Independent scaling and technology diversity
3. **Why PostgreSQL?** ACID compliance for business-critical data
4. **Why Redis?** High-performance caching and session storage
5. **Why Elasticsearch?** Advanced search capabilities

### Scalability Questions
1. **How to handle 10x traffic increase?** Horizontal scaling, caching, CDN
2. **How to handle database bottlenecks?** Read replicas, sharding, caching
3. **How to handle search performance?** Elasticsearch optimization, caching
4. **How to handle file storage?** S3 with CDN, compression, lazy loading

### Security Questions
1. **How to prevent unauthorized access?** JWT tokens, RBAC, input validation
2. **How to handle sensitive data?** Encryption, audit logging, access controls
3. **How to prevent common attacks?** Input validation, rate limiting, HTTPS

### Operational Questions
1. **How to monitor the system?** Prometheus, Grafana, distributed tracing
2. **How to handle failures?** Circuit breakers, retry mechanisms, fallbacks
3. **How to deploy safely?** Blue-green deployment, feature flags, rollback

### Future Considerations
1. **AI/ML integration** for smart ticket routing
2. **Mobile app development** with React Native
3. **Third-party integrations** (Slack, GitHub, GitLab)
4. **Advanced analytics** and predictive insights
5. **Global deployment** with multi-region support

---

## Conclusion

This design provides a scalable, secure, and maintainable ticketing system that can handle typical enterprise workloads and scale to larger deployments. The key architectural decisions focus on:

- **Security**: API Gateway with centralized authentication
- **Scalability**: Microservices with horizontal scaling
- **Performance**: Multi-layer caching and optimized data access
- **Maintainability**: Clear separation of concerns and client-specific optimizations

The BFF pattern ensures optimal performance for different client types while maintaining security and reducing complexity for frontend applications.

### Key Takeaways for Interview
1. **Start with realistic scale** - Design for typical enterprise first, then discuss scaling
2. **Estimate appropriately** - Use realistic numbers that reflect actual usage patterns
3. **Discuss trade-offs** - Show you understand the pros and cons of different approaches
4. **Consider security** - Address authentication, authorization, and data protection
5. **Plan for scalability** - Discuss horizontal scaling, caching, and performance optimization
6. **Think about operations** - Consider monitoring, deployment, and failure handling

This solution demonstrates a solid understanding of system design principles while being appropriate for a 60-minute interview format.

## 1. SSL/TLS Termination at Load Balancer Level

### **What is SSL/TLS Termination?**

SSL/TLS termination means the Load Balancer handles the encrypted HTTPS traffic and converts it to unencrypted HTTP traffic before forwarding to backend services.

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Client Request Flow                      │
│                                                             │
│  Client (HTTPS) → Load Balancer → API Gateway (HTTP)       │
│                                                             │
│  1. Client sends: HTTPS://api.company.com/api/v1/tickets  │
│  2. Load Balancer: Terminates SSL, validates certificate  │
│  3. Load Balancer forwards: HTTP://internal/api/v1/tickets │
│  4. API Gateway receives unencrypted HTTP request          │
└─────────────────────────────────────────────────────────────┘
```

### **Why Terminate at Load Balancer?**

#### **Performance Benefits:**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    SSL/TLS Termination Benefits             │
│                                                             │
│  • Reduces CPU load on backend services                    │
│  • Centralized certificate management                       │
│  • Easier certificate renewal and rotation                 │
│  • Better connection pooling                               │
│  • Reduced latency for internal communication              │
└─────────────────────────────────────────────────────────────┘
```

#### **Security Benefits:**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Security Advantages                      │
│                                                             │
│  • Single point for SSL/TLS configuration                  │
│  • Centralized certificate validation                      │
│  • Easier to implement security policies                   │
│  • Better monitoring of SSL/TLS traffic                    │
└─────────────────────────────────────────────────────────────┘
```

### **SSL/TLS Termination Process:**

```markdown
1. Client Request:
   ┌─────────────────────────────────────────────────────────┐
   │  GET https://api.company.com/api/v1/tickets          │
   │  Host: api.company.com                               │
   │  User-Agent: Mozilla/5.0...                          │
   └─────────────────────────────────────────────────────────┘

2. Load Balancer Processing:
   ┌─────────────────────────────────────────────────────────┐
   │  • Receives HTTPS request                             │
   │  • Validates SSL certificate                          │
   │  • Decrypts HTTPS traffic                             │
   │  • Extracts HTTP request                              │
   │  • Adds X-Forwarded-For header                        │
   │  • Adds X-Forwarded-Proto: https header              │
   └─────────────────────────────────────────────────────────┘

3. Forwarded to API Gateway:
   ┌─────────────────────────────────────────────────────────┐
   │  GET http://internal/api/v1/tickets                  │
   │  Host: api.company.com                               │
   │  X-Forwarded-For: 192.168.1.100                     │
   │  X-Forwarded-Proto: https                            │
   │  User-Agent: Mozilla/5.0...                          │
   └─────────────────────────────────────────────────────────┘
```

## 2. Authentication/Authorization at API Gateway Level

### **Authentication Flow:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Process                   │
│                                                             │
│  1. Client Request with JWT Token                         │
│  2. API Gateway validates JWT token                       │
│  3. API Gateway extracts user claims                      │
│  4. API Gateway forwards request with user context        │
│  5. Backend service processes with user info              │
└─────────────────────────────────────────────────────────────┘
```

### **JWT Token Validation:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    JWT Token Structure                     │
│                                                             │
│  Header: {"alg": "HS256", "typ": "JWT"}                  │
│  Payload: {                                                │
│    "sub": "user123",                                      │
│    "email": "john@company.com",                           │
│    "role": "USER",                                        │
│    "permissions": ["read:tickets", "write:tickets"],      │
│    "iat": 1642234567,                                     │
│    "exp": 1642238167                                      │
│  }                                                         │
│  Signature: HMACSHA256(base64UrlEncode(header) + "." +    │
│             base64UrlEncode(payload), secret)             │
└─────────────────────────────────────────────────────────────┘
```

### **API Gateway Authentication Steps:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Steps                     │
│                                                             │
│  1. Extract JWT from Authorization header                 │
│  2. Verify JWT signature using secret key                │
│  3. Check token expiration (exp claim)                   │
│  4. Validate issuer (iss claim) if configured            │
│  5. Extract user claims (sub, role, permissions)         │
│  6. Add user context to request headers                   │
│  7. Forward to backend service                            │
└─────────────────────────────────────────────────────────────┘
```

### **Authorization Process:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Authorization Flow                      │
│                                                             │
│  Request: GET /api/v1/tickets/123                         │
│  JWT Claims: {role: "USER", permissions: ["read:tickets"]}│
│                                                             │
│  1. Check if user has "read:tickets" permission          │
│  2. Check if user can access ticket 123                  │
│  3. If authorized, forward to Ticket Service              │
│  4. If not authorized, return 403 Forbidden              │
└─────────────────────────────────────────────────────────────┘
```

### **Implementation Example:**

#### **API Gateway Configuration (Spring Cloud Gateway):**
```yaml
# Gateway Configuration
spring:
  cloud:
    gateway:
      routes:
        - id: ticket-service
          uri: lb://ticket-service
          predicates:
            - Path=/api/v1/tickets/**
          filters:
            - name: JwtAuthenticationFilter
            - name: AuthorizationFilter
```

#### **JWT Authentication Filter:**
```java
@Component
public class JwtAuthenticationFilter implements GatewayFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = extractToken(exchange.getRequest());
        
        if (token != null && jwtValidator.isValid(token)) {
            UserClaims claims = jwtValidator.extractClaims(token);
            exchange.getRequest().mutate()
                .header("X-User-ID", claims.getUserId())
                .header("X-User-Role", claims.getRole())
                .header("X-User-Permissions", claims.getPermissions())
                .build();
        }
        
        return chain.filter(exchange);
    }
}
```

#### **Authorization Filter:**
```java
@Component
public class AuthorizationFilter implements GatewayFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().value();
        String userRole = exchange.getRequest().getHeaders().getFirst("X-User-Role");
        
        if (path.startsWith("/api/v1/admin") && !"ADMIN".equals(userRole)) {
            exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
            return exchange.getResponse().setComplete();
        }
        
        return chain.filter(exchange);
    }
}
```

### **Benefits of API Gateway Authentication:**

#### **Centralized Security:**
- **Single point** for authentication logic
- **Consistent** security policies across services
- **Easier** to implement security updates

#### **Performance:**
- **Reduces** authentication overhead on backend services
- **Caches** user sessions and permissions
- **Faster** request processing

#### **Security:**
- **Validates** tokens before reaching backend
- **Prevents** unauthorized access to services
- **Audits** all authentication attempts

### **Security Headers Added by API Gateway:**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Security Headers                        │
│                                                             │
│  X-User-ID: user123                                       │
│  X-User-Role: USER                                        │
│  X-User-Permissions: read:tickets,write:tickets           │
│  X-Request-ID: req-123456                                 │
│  X-Forwarded-For: 192.168.1.100                          │
│  X-Forwarded-Proto: https                                 │
└─────────────────────────────────────────────────────────────┘
```

This architecture ensures that all requests are properly authenticated and authorized before reaching any backend service, providing a secure and scalable authentication mechanism.

## JWT Token Flow in the Architecture

### **Two Types of JWT Tokens:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    JWT Token Types                         │
│                                                             │
│  1. Client JWT Token (for external clients)               │
│     • Generated by Authentication Service                  │
│     • Used by Web Client, Mobile Client, Desktop App      │
│     • Contains user permissions and roles                  │
│                                                             │
│  2. Service-to-Service JWT Token (for internal services)  │
│     • Generated by API Gateway or Service Mesh            │
│     • Used for inter-service communication                 │
│     • Contains service identity and permissions            │
└─────────────────────────────────────────────────────────────┘
```

## 1. Client JWT Token Flow

### **Web Client → BFF → API Gateway → Backend Services:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Client JWT Token Flow                   │
│                                                             │
│  1. Web Client sends JWT token to BFF                     │
│  2. BFF forwards JWT token to API Gateway                 │
│  3. API Gateway validates JWT token                       │
│  4. API Gateway forwards request to Backend Service       │
│  5. Backend Service processes with user context            │
└─────────────────────────────────────────────────────────────┘
```

### **Detailed Flow:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Step-by-Step Flow                       │
│                                                             │
│  Web Client Request:                                       │
│  GET /api/v1/tickets/123                                  │
│  Authorization: Bearer eyJhbGciOiJIUzI1NiIs...           │
│                                                             │
│  BFF Layer:                                                │
│  • Receives request with JWT                              │
│  • Forwards to API Gateway                                │
│  • No token generation/modification                       │
│                                                             │
│  API Gateway:                                              │
│  • Validates JWT token                                    │
│  • Extracts user claims                                   │
│  • Adds user context headers                              │
│  • Forwards to Ticket Service                             │
│                                                             │
│  Ticket Service:                                           │
│  • Receives request with user context                     │
│  • Processes business logic                               │
│  • Returns ticket data                                    │
└─────────────────────────────────────────────────────────────┘
```

## 2. Service-to-Service JWT Token Flow

### **Backend Services → Other Backend Services:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Service-to-Service JWT Flow             │
│                                                             │
│  Ticket Service needs User data → User Service             │
│                                                             │
│  1. Ticket Service requests service JWT from API Gateway  │
│  2. API Gateway generates service JWT                     │
│  3. Ticket Service uses service JWT to call User Service  │
│  4. User Service validates service JWT                    │
│  5. User Service returns user data                        │
└─────────────────────────────────────────────────────────────┘
```

## Answers to Your Questions:

### **1. Does web client need to know how backend services communicate?**

**Answer: NO**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Client Isolation                        │
│                                                             │
│  Web Client:                                                │
│  • Only knows about BFF endpoints                         │
│  • Doesn't know about backend services                    │
│  • Doesn't know about service-to-service communication    │
│  • Only handles client JWT tokens                         │
│                                                             │
│  Backend Services:                                         │
│  • Handle service-to-service JWT tokens                   │
│  • Communicate directly with each other                   │
│  • Use internal service discovery                         │
└─────────────────────────────────────────────────────────────┘
```

### **2. Is API Gateway responsible for generating JWT tokens?**

**Answer: PARTIALLY**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    JWT Token Responsibilities               │
│                                                             │
│  Client JWT Tokens:                                        │
│  • Generated by: Authentication Service                    │
│  • Validated by: API Gateway                              │
│  • Used by: Web Client, Mobile Client                     │
│                                                             │
│  Service-to-Service JWT Tokens:                           │
│  • Generated by: API Gateway or Service Mesh              │
│  • Validated by: Backend Services                         │
│  • Used by: Backend Services for inter-service calls      │
└─────────────────────────────────────────────────────────────┘
```

## Detailed Architecture with JWT Tokens:

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Complete JWT Flow                       │
│                                                             │
│  Web Client                                                │
│  ├── Login → Authentication Service                        │
│  │   └── Returns: Client JWT Token                        │
│  └── API Request → BFF                                    │
│      └── Forwards: Client JWT Token                       │
│                                                             │
│  BFF Layer                                                │
│  ├── Receives: Client JWT Token                           │
│  └── Forwards: Client JWT Token to API Gateway           │
│                                                             │
│  API Gateway                                              │
│  ├── Validates: Client JWT Token                          │
│  ├── Generates: Service JWT Token (for inter-service)    │
│  └── Forwards: Request with user context                  │
│                                                             │
│  Backend Services                                         │
│  ├── Ticket Service → User Service (with Service JWT)    │
│  ├── User Service → Project Service (with Service JWT)   │
│  └── All services validate Service JWT tokens             │
└─────────────────────────────────────────────────────────────┘
```

## Implementation Example:

### **Authentication Service (Generates Client JWT):**
```java
@Service
public class AuthenticationService {
    
    public String generateClientJWT(User user) {
        return Jwts.builder()
            .setSubject(user.getId())
            .claim("email", user.getEmail())
            .claim("role", user.getRole())
            .claim("permissions", user.getPermissions())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 3600000))
            .signWith(SignatureAlgorithm.HS256, clientJwtSecret)
            .compact();
    }
}
```

### **API Gateway (Generates Service JWT):**
```java
@Component
public class ServiceJwtGenerator {
    
    public String generateServiceJWT(String serviceName, String targetService) {
        return Jwts.builder()
            .setSubject(serviceName)
            .claim("target_service", targetService)
            .claim("permissions", getServicePermissions(serviceName))
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 300000)) // 5 min
            .signWith(SignatureAlgorithm.HS256, serviceJwtSecret)
            .compact();
    }
}
```

### **Backend Service (Uses Service JWT):**
```java
@Service
public class TicketService {
    
    public Ticket getTicket(String ticketId) {
        // Get user data from User Service using Service JWT
        String serviceJwt = apiGateway.generateServiceJWT("ticket-service", "user-service");
        
        User user = userServiceClient.getUser(userId, serviceJwt);
        
        return ticketRepository.findById(ticketId);
    }
}
```

## Summary:

1. **Web clients** only handle **client JWT tokens** and don't know about backend service communication
2. **API Gateway** validates client JWT tokens and generates service JWT tokens
3. **Backend services** use service JWT tokens for inter-service communication
4. **Authentication Service** generates client JWT tokens
5. **Service-to-service communication** is completely hidden from clients

This architecture provides clean separation of concerns and proper security isolation between client-facing and internal service communication.

## Authentication Service in the Architecture

### **Where Authentication Service Fits:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Microservices Layer                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │Ticket Service│ │User Service │ │Project Service│        │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │Search Service│ │Notification │ │Workflow     │         │
│  │             │ │Service      │ │Service      │         │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │Authentication│ │Authorization│ │Audit        │         │
│  │Service      │ │Service      │ │Service      │         │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

## Authentication Service Implementation

### **Authentication Service Responsibilities:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Service                   │
│                                                             │
│  • User login/logout                                       │
│  • Password validation                                     │
│  • JWT token generation                                    │
│  • Token refresh                                           │
│  • Password reset                                          │
│  • Multi-factor authentication                             │
│  • OAuth2 integration                                      │
└─────────────────────────────────────────────────────────────┘
```

### **Authentication Service API Endpoints:**

```http
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh
POST   /api/v1/auth/forgot-password
POST   /api/v1/auth/reset-password
POST   /api/v1/auth/register
POST   /api/v1/auth/verify-2fa
```

## Complete Authentication Flow

### **Step 1: User Login**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Login Flow                              │
│                                                             │
│  1. Web Client sends login request:                       │
│     POST /api/v1/auth/login                               │
│     {                                                      │
│       "email": "john@company.com",                        │
│       "password": "password123"                           │
│     }                                                      │
│                                                             │
│  2. Authentication Service:                                │
│     • Validates credentials against database              │
│     • Checks if user is active                            │
│     • Generates JWT token with user claims               │
│     • Returns JWT token to client                         │
│                                                             │
│  3. Response:                                              │
│     {                                                      │
│       "token": "eyJhbGciOiJIUzI1NiIs...",               │
│       "refreshToken": "eyJhbGciOiJIUzI1NiIs...",        │
│       "expiresIn": 3600,                                  │
│       "user": {                                            │
│         "id": "user123",                                   │
│         "email": "john@company.com",                      │
│         "role": "USER"                                     │
│       }                                                    │
│     }                                                      │
└─────────────────────────────────────────────────────────────┘
```

### **Step 2: Using the JWT Token**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    API Request Flow                        │
│                                                             │
│  1. Web Client makes API request:                         │
│     GET /api/v1/tickets/123                               │
│     Authorization: Bearer eyJhbGciOiJIUzI1NiIs...        │
│                                                             │
│  2. Request flows through:                                │
│     Web Client → BFF → API Gateway → Ticket Service       │
│                                                             │
│  3. API Gateway validates JWT token                       │
│  4. Ticket Service processes request                      │
│  5. Response flows back to client                         │
└─────────────────────────────────────────────────────────────┘
```

## Authentication Service Implementation

### **Technology Stack:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Service Tech Stack         │
│                                                             │
│  • Framework: Spring Boot / Node.js / Go                  │
│  • Database: PostgreSQL (user accounts)                   │
│  • Cache: Redis (sessions, rate limiting)                │
│  • JWT Library: jjwt / jsonwebtoken                       │
│  • Password Hashing: BCrypt / Argon2                     │
│  • OAuth2: Spring Security OAuth2 / Passport.js          │
└─────────────────────────────────────────────────────────────┘
```

### **Authentication Service Code Example:**

```java
@Service
public class AuthenticationService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private JwtTokenProvider jwtTokenProvider;
    
    public LoginResponse login(LoginRequest request) {
        // 1. Find user by email
        User user = userRepository.findByEmail(request.getEmail())
            .orElseThrow(() -> new AuthenticationException("Invalid credentials"));
        
        // 2. Validate password
        if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
            throw new AuthenticationException("Invalid credentials");
        }
        
        // 3. Check if user is active
        if (!user.isActive()) {
            throw new AuthenticationException("Account is disabled");
        }
        
        // 4. Generate JWT token
        String token = jwtTokenProvider.generateToken(user);
        String refreshToken = jwtTokenProvider.generateRefreshToken(user);
        
        // 5. Return response
        return LoginResponse.builder()
            .token(token)
            .refreshToken(refreshToken)
            .expiresIn(3600)
            .user(UserDto.from(user))
            .build();
    }
    
    public String refreshToken(String refreshToken) {
        // Validate refresh token and generate new access token
        return jwtTokenProvider.refreshToken(refreshToken);
    }
}
```

### **JWT Token Provider:**

```java
@Component
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.expiration}")
    private long jwtExpiration;
    
    public String generateToken(User user) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + jwtExpiration);
        
        return Jwts.builder()
            .setSubject(user.getId())
            .claim("email", user.getEmail())
            .claim("role", user.getRole())
            .claim("permissions", user.getPermissions())
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(SignatureAlgorithm.HS256, jwtSecret)
            .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parser().setSigningKey(jwtSecret).parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
    
    public UserClaims extractClaims(String token) {
        Claims claims = Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody();
        
        return UserClaims.builder()
            .userId(claims.getSubject())
            .email(claims.get("email", String.class))
            .role(claims.get("role", String.class))
            .permissions(claims.get("permissions", List.class))
            .build();
    }
}
```

## Database Schema for Authentication

```sql
-- Users table (part of Authentication Service)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL DEFAULT 'USER',
    is_active BOOLEAN DEFAULT TRUE,
    last_login_at TIMESTAMP,
    failed_login_attempts INTEGER DEFAULT 0,
    locked_until TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Refresh tokens table
CREATE TABLE refresh_tokens (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    token_hash VARCHAR(255) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Summary:

1. **Authentication Service** is a **backend microservice**
2. **It handles** user login, JWT generation, password management
3. **It's separate** from other services for security and scalability
4. **It generates** client JWT tokens that are used by web clients
5. **It's called** by the login process, not by API Gateway

The Authentication Service is the **source of truth** for user authentication and JWT token generation, while the API Gateway is responsible for **validating** those tokens during API requests.

---

## Front-End Application Provisioning

### Front-End Deployment Architecture

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Front-End Provisioning Flow              │
│                                                             │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                Development Environment                  │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   React     │ │   Angular   │ │   Vue.js    │     │ │
│  │  │  (Web App)  │ │  (Web App)  │ │ (Web App)   │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │React Native │ │   Flutter   │ │   Ionic     │     │ │
│  │  │ (Mobile App)│ │ (Mobile App)│ │(Mobile App) │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                               │
│                              ▼                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                CI/CD Pipeline                           │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   GitHub    │ │   GitLab    │ │   Bitbucket │     │ │
│  │  │  (Source)   │ │  (Source)   │ │  (Source)   │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  │                              │                         │ │
│  │                              ▼                         │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   Jenkins   │ │ GitHub      │ │   CircleCI  │     │ │
│  │  │   (Build)   │ │ Actions     │ │   (Build)   │     │ │
│  │  │             │ │ (Build)     │ │             │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  │                              │                         │ │
│  │                              ▼                         │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   Jest      │ │   Cypress   │ │   SonarQube │     │ │
│  │  │ (Unit Tests)│ │ (E2E Tests) │ │ (Code Quality)│   │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                               │
│                              ▼                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                Container Registry                       │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   Docker    │ │   AWS ECR   │ │   GCR       │     │ │
│  │  │   Hub       │ │ (Registry)  │ │ (Registry)  │     │ │
│  │  │             │ │             │ │             │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                               │
│                              ▼                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                Deployment Platforms                     │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   AWS S3    │ │   Netlify   │ │   Vercel    │     │ │
│  │  │ + CloudFront│ │ (Static)    │ │ (Static)    │     │ │
│  │  │ (Web Apps)  │ │             │ │             │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   AWS ECS   │ │   GKE       │ │   Azure     │     │ │
│  │  │ (Container) │ │ (Container) │ │ Container   │     │ │
│  │  │             │ │             │ │ Instances   │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   App Store │ │ Google Play │ │   Expo      │     │ │
│  │  │ (iOS App)   │ │ (Android)   │ │ (React Native)│   │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                              │                               │
│                              ▼                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                Production Environment                   │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   CDN       │ │   Load      │ │   API       │     │ │
│  │  │ (Static)    │ │ Balancer    │ │ Gateway     │     │ │
│  │  │             │ │             │ │             │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  │                              │                         │ │
│  │                              ▼                         │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐     │ │
│  │  │   Web       │ │   Mobile    │ │   Desktop   │     │ │
│  │  │   Clients   │ │   Clients   │ │   Clients   │     │ │
│  │  │             │ │             │ │             │     │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘     │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Front-End Technology Stack

#### **Web Applications**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Web Application Stack                    │
│                                                             │
│  • Framework: React.js / Angular / Vue.js                  │
│  • State Management: Redux / NgRx / Vuex                   │
│  • UI Library: Material-UI / Ant Design / Vuetify          │
│  • Build Tool: Webpack / Vite / Angular CLI                │
│  • Testing: Jest / Cypress / Playwright                    │
│  • Package Manager: npm / yarn / pnpm                      │
│  • TypeScript: For type safety and better DX               │
└─────────────────────────────────────────────────────────────┘
```

#### **Mobile Applications**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Mobile Application Stack                 │
│                                                             │
│  • Cross-platform: React Native / Flutter / Ionic          │
│  • Native: Swift (iOS) / Kotlin (Android)                  │
│  • State Management: Redux / MobX / Provider               │
│  • UI Components: Native Base / Flutter Widgets            │
│  • Navigation: React Navigation / Flutter Navigation        │
│  • Testing: Detox / Flutter Test / Appium                  │
│  • Build: Expo / Xcode / Android Studio                    │
└─────────────────────────────────────────────────────────────┘
```

#### **Desktop Applications**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Desktop Application Stack                │
│                                                             │
│  • Framework: Electron / Tauri / JavaFX                    │
│  • Web Technologies: HTML/CSS/JavaScript                   │
│  • Native Integration: File system, notifications          │
│  • Auto-updates: Electron Updater / Sparkle                │
│  • Packaging: electron-builder / cargo                     │
│  • Distribution: GitHub Releases / App Stores              │
└─────────────────────────────────────────────────────────────┘
```

### Front-End Deployment Strategies

#### **Static Site Deployment**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Static Site Deployment                   │
│                                                             │
│  1. Build Process:                                         │
│     • npm run build                                        │
│     • Creates optimized static files                       │
│     • Minifies CSS/JS                                      │
│     • Optimizes images                                     │
│                                                             │
│  2. Deployment Options:                                    │
│     • AWS S3 + CloudFront                                  │
│     • Netlify / Vercel                                     │
│     • GitHub Pages                                         │
│     • Azure Static Web Apps                                │
│                                                             │
│  3. Benefits:                                              │
│     • Fast loading                                         │
│     • Global CDN                                           │
│     • Cost-effective                                       │
│     • Auto-scaling                                         │
└─────────────────────────────────────────────────────────────┘
```

#### **Container Deployment**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Container Deployment                     │
│                                                             │
│  1. Dockerfile Example:                                    │
│     FROM node:18-alpine                                    │
│     WORKDIR /app                                           │
│     COPY package*.json ./                                  │
│     RUN npm ci --only=production                           │
│     COPY . .                                               │
│     RUN npm run build                                      │
│     FROM nginx:alpine                                      │
│     COPY --from=0 /app/build /usr/share/nginx/html         │
│     EXPOSE 80                                              │
│                                                             │
│  2. Deployment Platforms:                                  │
│     • AWS ECS / EKS                                        │
│     • Google GKE                                           │
│     • Azure Container Instances                            │
│     • Docker Swarm                                         │
│                                                             │
│  3. Benefits:                                              │
│     • Consistent environment                               │
│     • Easy scaling                                         │
│     • Blue-green deployments                               │
│     • Rollback capability                                  │
└─────────────────────────────────────────────────────────────┘
```

### Front-End CI/CD Pipeline

#### **GitHub Actions Example**
```yaml
name: Front-End CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run test
      - run: npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run build
      - uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      - run: aws s3 sync build/ s3://my-app-bucket --delete
      - run: aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

### Front-End Environment Configuration

#### **Environment Variables**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Environment Configuration                │
│                                                             │
│  Development (.env.development):                           │
│  REACT_APP_API_URL=http://localhost:8080/api/v1           │
│  REACT_APP_ENVIRONMENT=development                         │
│  REACT_APP_ENABLE_LOGGING=true                            │
│                                                             │
│  Staging (.env.staging):                                   │
│  REACT_APP_API_URL=https://staging-api.company.com/api/v1 │
│  REACT_APP_ENVIRONMENT=staging                             │
│  REACT_APP_ENABLE_LOGGING=true                            │
│                                                             │
│  Production (.env.production):                             │
│  REACT_APP_API_URL=https://api.company.com/api/v1        │
│  REACT_APP_ENVIRONMENT=production                          │
│  REACT_APP_ENABLE_LOGGING=false                           │
└─────────────────────────────────────────────────────────────┘
```

### Front-End Performance Optimization

#### **Build Optimization**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Performance Optimization                 │
│                                                             │
│  • Code Splitting: Lazy loading of routes                  │
│  • Tree Shaking: Remove unused code                        │
│  • Minification: Compress CSS/JS files                     │
│  • Compression: Gzip/Brotli compression                    │
│  • Caching: Browser and CDN caching                        │
│  • Image Optimization: WebP format, lazy loading           │
│  • Bundle Analysis: webpack-bundle-analyzer                │
└─────────────────────────────────────────────────────────────┘
```

#### **Runtime Optimization**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Runtime Optimization                     │
│                                                             │
│  • Virtual Scrolling: For large lists                      │
│  • Memoization: React.memo, useMemo, useCallback          │
│  • Debouncing: Search input, API calls                     │
│  • Throttling: Scroll events, resize events                │
│  • Service Workers: Offline support, caching               │
│  • Progressive Web App: Installable, offline-first         │
└─────────────────────────────────────────────────────────────┘
```

### Front-End Security Considerations

#### **Security Best Practices**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Security Measures                       │
│                                                             │
│  • HTTPS Only: Force HTTPS in production                   │
│  • Content Security Policy: Prevent XSS attacks            │
│  • Input Validation: Client-side validation                │
│  • JWT Storage: Secure storage in memory/cookies           │
│  • Dependency Scanning: npm audit, Snyk                    │
│  • Environment Variables: Never expose secrets             │
│  • CORS Configuration: Proper CORS headers                 │
└─────────────────────────────────────────────────────────────┘
```

### Front-End Monitoring and Analytics

#### **Monitoring Tools**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Monitoring Stack                        │
│                                                             │
│  • Error Tracking: Sentry, LogRocket                       │
│  • Performance: Lighthouse, Web Vitals                     │
│  • Analytics: Google Analytics, Mixpanel                   │
│  • User Behavior: Hotjar, FullStory                        │
│  • Real User Monitoring: New Relic, DataDog                │
│  • A/B Testing: Optimizely, Google Optimize                │
└─────────────────────────────────────────────────────────────┘
```

### Front-End Testing Strategy

#### **Testing Pyramid**
```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Testing Strategy                         │
│                                                             │
│  Unit Tests (70%):                                         │
│  • Jest + React Testing Library                            │
│  • Component testing                                       │
│  • Utility function testing                                │
│                                                             │
│  Integration Tests (20%):                                  │
│  • API integration testing                                 │
│  • State management testing                                │
│  • Router testing                                          │
│                                                             │
│  E2E Tests (10%):                                          │
│  • Cypress / Playwright                                    │
│  • Critical user journeys                                  │
│  • Cross-browser testing                                   │
└─────────────────────────────────────────────────────────────┘
```

This front-end provisioning diagram provides a comprehensive overview of how front-end applications are developed, tested, deployed, and maintained in the Jira-like ticketing system architecture. It covers the complete lifecycle from development to production deployment, including various deployment strategies, performance optimization, security considerations, and monitoring approaches.