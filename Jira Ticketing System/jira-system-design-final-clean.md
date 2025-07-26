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

```markdown
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

## SSL/TLS Termination at Load Balancer Level

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

## Authentication/Authorization at API Gateway Level

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

### **Client JWT Token Flow:**

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

### **Service-to-Service JWT Token Flow:**

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

### **Complete JWT Flow:**

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

### **Complete Authentication Flow:**

```markdown
┌─────────────────────────────────────────────────────────────┐
│                    Login Flow                              │
│                                                             │
│  1. Web Client sends login request                        │
│  2. Authentication Service validates credentials           │
│  3. Authentication Service generates JWT token             │
│  4. JWT token returned to client                          │
│                                                             │
│  API Request Flow:                                         │
│  1. Web Client makes API request with JWT token          │
│  2. Request flows through: BFF → API Gateway → Services  │
│  3. API Gateway validates JWT token                       │
│  4. Services process request with user context            │
└─────────────────────────────────────────────────────────────┘
```

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
│  1. Deployment Platforms:                                  │
│     • AWS ECS / EKS                                        │
│     • Google GKE                                           │
│     • Azure Container Instances                            │
│     • Docker Swarm                                         │
│                                                             │
│  2. Benefits:                                              │
│     • Consistent environment                               │
│     • Easy scaling                                         │
│     • Blue-green deployments                               │
│     • Rollback capability                                  │
└─────────────────────────────────────────────────────────────┘
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

---

## Interview Questions & Answer Reference

This section maps potential interview questions to the relevant sections within this design document, serving as a study guide.

### High-Level Architecture & Core Concepts

* **Question:** "Can you walk me through the high-level architecture of your system?"
    * **Refer to:** [`High-Level Architecture`](#high-level-architecture) > `Architecture Overview`
	
* **Question:** "Why did you choose a microservices architecture? What are the trade-offs you considered?"
    * **Refer to:** `Trade-offs & Discussion Points` > `Monolith vs Microservices`
    * **Also mention:** `Scalability Considerations` > `Horizontal Scaling`

* **Question:** "What is the purpose of the BFF (Backend for Frontend) layer in your design?"
    * **Refer to:** `High-Level Architecture` > `Why BFF Pattern?`
    * **Also mention:** `Component Design` > `2. BFF Services` to give specific examples.

* **Question:** "You have both an API Gateway and a BFF layer. Can you explain the distinct roles of each?"
    * **Refer to:** `Component Design` > `1. API Gateway` and `2. BFF Services`.

---

### Component Deep Dive

* **Question:** "Let's talk about the Ticket Service. What are its primary responsibilities?"
    * **Refer to:** `Component Design` > `3. Core Microservices` > `Ticket Service`

* **Question:** "How does search functionality work in your system? Why did you choose Elasticsearch for this?"
    * **Refer to:** `Component Design` > `3. Core Microservices` > `Search Service`
    * **Also mention:** `Data Storage Layer` > `Search Engine (Elasticsearch)` for the "why".

* **Question:** "How are users notified of changes or comments on a ticket?"
    * **Refer to:** `Component Design` > `3. Core Microservices` > `Notification Service`

---

### Data & Scalability

* **Question:** "How would you handle a sudden 10x spike in traffic?"
    * **Refer to:** `Scalability Considerations` > `Horizontal Scaling` & `Performance Optimization` (especially Caching).

* **Question:** "You've chosen PostgreSQL as your primary database. Why a relational database? How would you handle it if the database became a bottleneck?"
    * **Refer to:** `Data Storage Layer` > `Primary Database (PostgreSQL)` for the "why".
    * **Refer to:** `Scalability Considerations` > `Horizontal Scaling` > `Database read replicas` and `Sharding strategy` for handling bottlenecks.

* **Question:** "What is your caching strategy?"
    * **Refer to:** `Scalability Considerations` > `Caching Strategy`
    * **Also refer to:** `Data Storage Layer` > `Cache Layer (Redis)`.

---

### Security & Authentication

* **Question:** "Where and how do you handle user authentication and authorization?"
    * **Refer to:** `Authentication/Authorization at API Gateway Level`
    * **Explain the flow using:** `Authentication Flow` and `Authorization Flow` diagrams from that section.

* **Question:** "Walk me through your JWT strategy. How does a service know if a request is legitimate?"
    * **Refer to:** `JWT Token Flow in the Architecture`
    * **Tip:** Highlight the distinction between a `Client JWT Token` and a `Service-to-Service JWT Token`.

* **Question:** "What is SSL/TLS termination and why are you doing it at the Load Balancer?"
    * **Refer to:** `SSL/TLS Termination at Load Balancer Level`

---

### Front-End Architecture

* **Question:** "How does the front-end application (the React app) get delivered to the end-user?"
    * **Refer to:** `Front-End Application Provisioning` > `Front-End Deployment Architecture` diagram.
    * **Explain the flow using:** `Front-End Deployment Strategies` > `Static Site Deployment`.

* **Question:** "How do you manage configuration for different environments (development, production) in your front-end app?"
    * **Refer to:** `Front-End Application Provisioning` > `Front-End Environment Configuration`

---

### Operational & Process Questions

* **Question:** "How would you monitor the health of this entire system?"
    * **Refer to:** `Interview Discussion Points` > `Operational Questions` > point 1 (Prometheus, Grafana, distributed tracing).
    * **Also refer to:** `Infrastructure Layer` diagram which shows `Prometheus`.

* **Question:** "How would you deploy a new version of the Ticket Service without causing downtime?"
    * **Refer to:** `Interview Discussion Points` > `Operational Questions` > point 3 (Blue-green deployment, feature flags, rollback).