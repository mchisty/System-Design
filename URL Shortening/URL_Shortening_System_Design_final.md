````markdown
# URL Shortening Service: A Scalable System Design

---
## Workflow 1: Creating a Short URL (Write Path)

This workflow describes the process when a user wants to create a new short URL.

1.  User Visits Website: An end-user wants to shorten a long URL. They open their browser and go to our website, `www.urlshorten.com`.

2.  CDN Serves Frontend: The **CDN** intercepts this request. It serves the static assets for the website (HTML, CSS, JavaScript) from an edge server close to the user, ensuring the page loads quickly.

3.  User Submits URL: The user pastes their long URL (`www.dm.co.uk/very_long_url`) into the input box and clicks "Shorten." The browser sends a `POST` request to `/api/v1/urls`.

4.  Request Routing:
    * The request first hits the **CDN**, which routes it to the nearest and healthiest cloud region (e.g., EU-WEST-1).
    * The request then goes to the **Load Balancer** in that region, which forwards it to an available **API Gateway** instance.

5.  Security & Validation: The **API Gateway** validates the request. It checks for a valid user token (authentication) and ensures the user hasn't exceeded their usage limits (rate limiting).

6.  Application Logic: The API Gateway forwards the valid request to the **Application Layer**. The application server performs the following steps:
    * It fetches a pre-generated, unique short code from the **Redis Cache** (the key pool).
    * It creates a new record containing the `short_code` and the `original_url`.
    * It saves this new record to the master "write" region of the **Global NoSQL Database**.

7.  Response to User: The application server sends a success response back up through the API Gateway. The user's browser receives the new short URL (e.g., `www.urlshorten.com/xyz789`) and displays it on the screen.

---

## Workflow 2: Clicking a Short URL (Read Path)

This workflow describes what happens when a user clicks on a previously generated short URL.

1.  User Clicks Link: A user clicks on a short link: `www.urlshorten.com/xyz789`.

2.  Request Routing: The request flow is the same as before: **CDN** -> nearest **Cloud Region** -> **Load Balancer** -> **API Gateway**. The API Gateway performs a quick check (e.g., for malicious patterns) and forwards the request.

3.  Application Logic (Cache Hit - 99% of cases):
    * The **Application Layer** receives the request for `/xyz789`.
    * It first checks the **Redis Cache** for the key `xyz789`.
    * The key is found in the cache (a "cache hit"). The cache instantly returns the corresponding long URL.
	* If the key is not found in Redis, the application then queries the main database (e.g., DynamoDB), which is the authoritative source of truth, for the record associated with xyz789. The database returns the original_url. 
	* The application then sends the 301 Redirect response to the user and, crucially, writes the original_url into the Redis Cache with an expiration time (e.g., 24 hours). This "populates the cache" so the next request for this key will be a fast cache hit (also described in step #5 below)

4.  Asynchronous Analytics: Simultaneously, the application server publishes a small message (e.g., `{ "short_code": "xyz789" }`) to the **SQS Message Queue**. This action is extremely fast and does not delay the user.

5.  Redirect User: The application server immediately sends a `301 Redirect` response back to the user's browser, containing the long URL. The user's browser is then redirected to `www.dm.co.uk/very_long_url`. **The user-facing part of the workflow ends here.**

6.  Background Analytics Processing: Later, the **Analytics Service** (a separate worker) pulls messages from the SQS queue in batches. It updates the `click_count` for `xyz789` in the **NoSQL Database**. This happens behind the scenes with no impact on user performance.

---


This document outlines the proposed system design for a large-scale, global, and highly available URL shortening service. The architecture prioritizes low-latency redirection, security, and operational resilience.

---

## 1. Requirements

### Functional Requirements
* URL Shortening: Users can submit a long URL and receive a unique, short URL.
* URL Redirection: Accessing a short URL redirects the user to the original long URL.
* Custom URLs: Users can optionally request a custom, human-readable short URL.
* Analytics: The system will track the number of clicks for each short URL.

### Non-Functional Requirements
* High Availability: 99.99% uptime, achieved through a multi-region architecture with no single point of failure.
* Low Latency: Redirects must be exceptionally fast, with a P99 latency of under 50ms for users worldwide.
* Scalability: The system must handle a high volume of traffic, estimated at 100,000 redirect requests per second and 10,000 new URL creations per minute.

---

## 2. High-Level System Design

The architecture is designed for global scale, featuring a multi-region deployment, an API Gateway for security, and an asynchronous pipeline for processing analytics to ensure fast redirect performance.

```text
                               +-----------------+
                               |   End Users     | (Global Audience)
                               | (Browser/Mobile)|
                               +--------+--------+
                                        |
+---------------------------------------|------------------------------------------+
|                                       v                                          |
|                +------------------------------------------------+                |
|                |         CDN (e.g., CloudFront, Cloudflare)     |                |
|                | (Geo-DNS Routing, Static Caching, DDoS Protect)|                |
|                +------------------------+-----------------------+                |
|                                         |                                        |
|                (Routes user to nearest & healthiest region)                      |
|                                         |                                        |
|                  +----------------------v----------------------+                 |
|                  |          Cloud Provider Network             |                 |
+------------------|---------------------------------------------|-----------------+
                   |             (e.g., AWS, GCP, Azure)         |
                   |                                             |
+------------------|----------------------+----------------------|-----------------+
|         REGION: US-EAST-1              |      REGION: EU-WEST-1                 |
|                                        |                                        |
|  +----------------------------------+  |   +----------------------------------+ |
|  | Load Balancer                    |  |   | Load Balancer                    | |
|  +-----------------+----------------+  |   +-----------------+----------------+ |
|                    |                   |                     |                  |
|  +-----------------v----------------+  |   +-----------------v----------------+ |
|  | API Gateway                      |  |   | API Gateway                      | |
|  | (Auth, Rate Limiting, Logging)   |  |   | (Auth, Rate Limiting, Logging)   | |
|  +-----------------+----------------+  |   +-----------------+----------------+ |
|                    |                   |                     |                  |
|  +-----------------v----------------+  |   +-----------------v----------------+ |
|  | Application Layer (Stateless)    |  |   | Application Layer (Stateless)    | |
|  | (Backend Microservices)          |  |   | (Backend Microservices)          | |
|  +----------------------------------+  |   +----------------------------------+ |
|   |         |              |           |    |         |              |          |
|   | (Cache) |              | (DB)      |    | (Cache) |              | (DB)     |
|   v         v              v           |    v         v              v          |
| +----+   +----+         +----+         |  +----+   +----+         +----+        |
| |Redis| | SQS|         |DB-R|         |  |Redis| | SQS|         |DB-R|        |
+------------------------------------------+------------------------------------------+
|  ^         ^              ^                                                        |
|  | (Async Analytics)      | (Read Replica)                                         |
|  +------------------------+                                                        |
|                                                                                    |
|                      +--------------------------+                                  |
|                      | Analytics Service (Worker) |                                  |
|                      +------------+-------------+                                  |
|                                   |                                                |
|                (Updates counts)   v                                                |
|      +------------------------------------------------------+                      |
|      |        Global NoSQL Database (e.g., DynamoDB Global Table) |                      |
|      |        [ Master Write Region: US-EAST-1 ]                |                      |
|      +------------------------------------------------------+                      |
|                                                                                    |
+------------------------------------------------------------------------------------+

````

-----

## 3\. Justification of Components

  * CDN (Content Delivery Network): As the global entry point, the CDN routes users to the nearest cloud region based on latency. It also caches the service's minimal static assets (CSS, logos) and provides a crucial layer of DDoS protection.

  * Multi-Region Deployment: To serve a global audience with low latency and ensure high availability, the entire application stack is replicated across multiple geographic regions (e.g., North America, Europe, Asia-Pacific).

  * Load Balancer: Within each region, a load balancer distributes incoming requests evenly across the API Gateway instances, preventing any single gateway from being overwhelmed.

  * API Gateway: This is the secure front door to our application logic. It enforces critical security policies like **authentication** (validating API keys or user tokens) and **rate limiting**. It then routes valid requests to the appropriate backend microservice.

  * Application Layer (Stateless Microservices): This layer contains the core business logic. It's built as stateless services, meaning any server can process any request. This allows for seamless horizontal scaling to handle fluctuating traffic loads.

  * Cache (Redis): An in-memory cache stores frequently accessed URL mappings. The vast majority of redirect requests will be served directly from this cache in milliseconds, protecting the database and ensuring minimal latency.

  * Message Queue (SQS): To keep redirects fast, analytics are handled asynchronously. When a link is clicked, a message is instantly placed on the queue. This decouples the critical redirect path from the slower analytics processing.

  * Analytics Service: A separate pool of workers consumes messages from the queue in batches to update click counts in the database, ensuring that user-facing performance is never impacted by analytics writes.

  * Global NoSQL Database (DynamoDB): A NoSQL database is chosen for its immense horizontal scalability, which is essential for storing billions of URLs. A solution like DynamoDB Global Tables provides a main write region and low-latency read replicas in all other regions, enabling fast database lookups for users anywhere in the world.

-----

## 4\. API and Data Model

### API Endpoints

The service will expose a simple RESTful API.

  * `POST /api/v1/urls` - Creates a new short URL.
  * `GET /{short_code}` - Redirects to the original long URL.
  * `GET /api/v1/urls/{short_code}/stats` - Retrieves click count and other statistics.

### Data Model

A NoSQL data model is used for simplicity and scalability.

**Table Name**: `url_mappings`

  * **Primary Key / Partition Key**: `short_code` (String)
  * **Attributes**:
      * `original_url` (String)
      * `creation_date` (Timestamp)
      * `user_id` (String, Optional)
      * `click_count` (Number)

-----

## 5\. Architectural Trade-offs

  * **NoSQL vs. SQL**: We chose a NoSQL database for its superior horizontal scalability and predictable low-latency performance at scale, which is critical for this read-heavy application. This comes at the cost of the rich querying capabilities and transactional consistency offered by traditional RDBMS.

  * **Eventual Consistency for Analytics**: Click counts are updated asynchronously. This means the stats may have a slight delay, but it's a necessary trade-off to guarantee the primary function—URL redirection—remains extremely fast and reliable.

