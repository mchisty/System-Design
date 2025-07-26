# Capacity Estimation

Before diving into the architecture, let's establish baseline requirements to understand the scale we're designing for:

## User Scale
- **Daily Active Users (DAU)**: 100 million users
- **Monthly Active Users (MAU)**: 2 billion users
- **Peak concurrent users**: 20 million (during prime time)

## Video Upload Requirements
- **Videos uploaded per day**: 500,000 videos
- **Average video size**: 100 MB (after compression)
- **Peak upload rate**: 1,000 videos/minute during busy hours
- **Video formats supported**: Multiple resolutions (360p, 720p, 1080p, 4K)

## Storage Requirements
- **Daily storage needs**: 500,000 videos × 100 MB = 50 TB/day
- **Annual storage growth**: ~18 PB/year
- **Total storage (5 years)**: ~90 PB
- **Redundancy factor**: 3x (considering backups and CDN replication)
- **Total storage with redundancy**: ~270 PB

## Bandwidth Requirements
- **Average concurrent viewers**: 10 million
- **Average video bitrate**: 2 Mbps (mixed resolutions)
- **Peak bandwidth**: 10M users × 2 Mbps = 20 Tbps
- **Monthly data transfer**: ~500 PB

## Database Scale
- **Video metadata records**: Growing by 500K/day = 180M/year
- **Comments per day**: ~50 million
- **User interactions**: ~500 million/day (likes, views, subscriptions)

These numbers justify our architectural decisions around horizontal scaling, CDN usage, and distributed storage systems.

---

# Trade-offs and Design Decisions

Every system design involves critical trade-offs. Here are the key decisions made in this architecture and their implications:

## Consistency vs. Availability

**Decision**: Eventual consistency for most operations
- **Trade-off**: We prioritize availability over strong consistency
- **Rationale**: Video streaming can tolerate slight delays in metadata updates (view counts, likes) but cannot afford service downtime
- **Implementation**: 
  - Video uploads use eventual consistency between metadata and actual video files
  - View counts may be slightly delayed but videos remain watchable
  - Critical user data (authentication) maintains stronger consistency

## Storage Strategy

**Decision**: Object storage over traditional databases for video files
- **Trade-off**: Lose ACID properties but gain massive scalability and cost efficiency
- **Rationale**: Video files are immutable and don't require complex queries
- **Implementation**: 
  - Videos stored in distributed object storage (AWS S3, Google Cloud Storage)
  - Metadata kept in traditional databases for complex queries
  - Hybrid approach balances performance and functionality

## Caching Strategy

**Decision**: Multi-layer caching (CDN + Application cache)
- **Trade-off**: Increased complexity and potential cache invalidation issues
- **Rationale**: Video streaming is read-heavy with high geographic distribution
- **Implementation**:
  - CDN caches popular videos globally
  - Application cache stores frequently accessed metadata
  - Cache invalidation strategy accepts brief inconsistency windows

## Microservices vs. Monolith

**Decision**: Microservices architecture with API gateways
- **Trade-off**: Higher operational complexity vs. better scalability and team autonomy
- **Rationale**: Different services have vastly different scaling requirements
- **Implementation**:
  - Upload service needs different scaling than recommendation engine
  - Independent deployments reduce blast radius of failures
  - Each team can optimize their service independently

## Synchronous vs. Asynchronous Processing

**Decision**: Hybrid approach - sync for user-facing, async for background tasks
- **Trade-off**: Complexity in handling both patterns vs. optimal user experience
- **Rationale**: Users need immediate feedback but heavy processing can be deferred
- **Implementation**:
  - Video uploads return immediately after initial validation
  - Transcoding happens asynchronously with status updates
  - Real-time features (comments, likes) use synchronous APIs

## Data Partitioning Strategy

**Decision**: Horizontal partitioning by video ID and user ID
- **Trade-off**: Cross-shard queries become complex but individual operations scale linearly
- **Rationale**: Most queries are isolated to single videos or users
- **Implementation**:
  - Video data partitioned by video ID hash
  - User data partitioned by user ID
  - Recommendation engine handles cross-partition aggregations

## Search Implementation

**Decision**: Dedicated search service (Elasticsearch) vs. database queries
- **Trade-off**: Additional infrastructure complexity vs. search performance and flexibility
- **Rationale**: Video search requires full-text, fuzzy matching, and ranking algorithms
- **Implementation**:
  - Elasticsearch indexes video metadata asynchronously
  - Database handles exact matches and simple queries
  - Search service provides advanced features like autocomplete and personalized results

These trade-offs reflect the reality that no single solution fits all requirements. The key is making conscious decisions that align with your specific use case, scale, and team capabilities.