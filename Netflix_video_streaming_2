# Netflix Video Streaming — System Design (Redesigned)

## 1. Functional Requirements

- User registration, login, profile management (multi-profile per account)
- Browse catalog: M genres, N titles per genre
- Search titles by name, actor, genre, director
- Stream video with adaptive bitrate (ABR)
- Track watch history and resume position
- Show trending titles per genre and globally
- Personalized recommendations
- A/B testing for UI and algorithm experiments

### Out of Scope

- Payments & subscription billing (separate bounded context)
- Social features (sharing, comments)
- Content upload by creators (admin-only ingestion)
- Offline downloads
- Parental controls
- Live streaming (sports, events)

---

## 2. Non-Functional Requirements

| NFR | Target | Linked FR | Reasoning |
|-----|--------|-----------|-----------|
| Availability | 99.99% | All | Revenue loss at scale ~$1M/min downtime |
| Catalog Latency | < 100ms p99 | Browse, Search | Users abandon after 3s, feed must be instant |
| Stream Start | < 2 seconds | Streaming | Industry benchmark, buffering = churn |
| Write Throughput | 115K writes/sec peak | Watch History | 100M DAU × 20 events/session, must not drop |
| Read Throughput | 18K QPS peak | Catalog Browse | Home feed loaded 3x/day per user |
| Scalability | Horizontal | All | 100M DAU today, design for 500M |
| Consistency — Auth | Strong | Login, Profile | Cannot serve wrong user's data |
| Consistency — Analytics | Eventual (≤5s) | Trending, History | 5s staleness acceptable for view counts |
| Durability | Zero data loss | Watch History, User Data | History is ML training input, cannot lose |
| Fault Tolerance | No SPOF | All | Region failure should not cause global outage |
| Security | DRM protected | Streaming | Content protection required by studios |
| Global Latency | < 50ms to edge | Streaming | CDN edge must be close to users |

---

## 3. Back of Envelope

### Users & Traffic

```
Total Users        = 300M
DAU                = 100M
Concurrent Users   = 10% of DAU = 10M (peak hour)
Avg Sessions/Day   = 2
Avg Session Length = 45 min
Regions            = 5 (NA, EU, APAC, LATAM, MEA)
```

### Read / Write QPS

```
--- Watch History Writes ---
Events per session       = 20 (progress update every 30s for 45 min ≈ 90, but batched to ~20)
Total events/day         = 100M users × 20 events = 2B events/day
Avg Write QPS            = 2,000,000,000 / 86,400 = ~23,148 QPS
Peak Write QPS (5x avg)  = 23,148 × 5 = ~115,000 QPS

--- Catalog Reads ---
Feed loads per user/day  = 3
Total reads/day          = 100M × 3 = 300M
Avg Read QPS             = 300,000,000 / 86,400 = ~3,472 QPS
Peak Read QPS (5x avg)   = 3,472 × 5 = ~17,360 QPS

--- Search ---
Searches per user/day    = 5
Total searches/day       = 100M × 5 = 500M
Avg Search QPS           = 500,000,000 / 86,400 = ~5,787 QPS
Peak Search QPS (5x avg) = 5,787 × 5 = ~28,935 QPS

--- API Gateway ---
Total Peak QPS           = 115K + 17K + 29K + misc = ~200K QPS
```

### Bandwidth

```
--- Streaming Egress ---
Concurrent streams       = 10M
Avg bitrate              = 5 Mbps (1080p adaptive average)
Total egress             = 10,000,000 × 5 Mbps = 50,000,000 Mbps = 50 Tbps

--- Chunk Serving ---
Chunk duration           = 4 seconds
Chunk size at 5 Mbps     = 5 × 4 / 8 = 2.5 MB per chunk
Chunks served per sec    = 10,000,000 / 4 = 2,500,000 chunks/sec

--- API Bandwidth ---
Avg API response         = 5 KB
Peak API QPS             = 200K QPS
API egress               = 200,000 × 5 KB = ~1 GB/sec = ~8 Gbps (negligible vs streaming)
```

### Storage

```
--- Catalog ---
Unique titles            = 50,000
Metadata per title       = 10 KB
Catalog total            = 50,000 × 10 KB = 500 MB (fits entirely in memory)

--- Video Files ---
Avg video length         = 1.5 hours
Resolutions              = 4 (480p, 720p, 1080p, 4K)
Codecs                   = 2 (H.264, H.265/HEVC)
Audio tracks             = 3 (stereo, 5.1, Atmos)
Total variants per title = 4 × 2 × 3 = 24
Avg size per variant     = 3 GB
Storage per title        = 24 × 3 GB = 72 GB
Total video storage      = 50,000 × 72 GB = 3,600,000 GB = 3.6 PB

--- Watch History ---
Record size              = 100 bytes
Records per day          = 2B
Retention                = 2 years (ML training requires long history)
Total history storage    = 2B × 730 × 100 bytes = 146 TB

--- User Data ---
Users                    = 300M
User record size         = 2 KB (profile, preferences, settings)
User storage             = 300M × 2 KB = 600 GB
```

### Cache Sizing (Corrected)

```
--- Catalog Cache ---
Working set              = 500 MB full catalog
Cache overhead           = 1.5x
Catalog cache            = 500 MB × 1.5 = 750 MB

--- Session Cache ---
Concurrent sessions      = 10M
Session size             = 500 bytes
Session cache            = 10M × 500 bytes = 5 GB

--- Trending Cache ---
Sorted sets              = 25 genres + 1 global = 26
Members per set          = 50,000 titles
Per member               = 50 bytes (titleId + score)
Trending cache           = 26 × 50,000 × 50 = 65 MB

--- Home Feed Cache (NEW - previously missing) ---
Concurrent users         = 10M
Feed cache hit rate      = 30% (rest served from sorted sets)
Cached feeds             = 3M feeds
Avg feed size            = 50 KB (25 genres × 20 titles × 100 bytes)
Home feed cache          = 3M × 50 KB = 150 GB

--- Resume Position Cache ---
Active users with resume = 50M
Positions per user       = 5 titles avg
Entry size               = 100 bytes
Resume cache             = 50M × 5 × 100 = 25 GB

--- Manifest Cache ---
Popular titles           = 5,000 (10% of catalog)
Manifest size            = 10 KB
Manifest cache           = 5,000 × 10 KB = 50 MB

Total Redis Memory       ≈ 750 MB + 5 GB + 65 MB + 150 GB + 25 GB + 50 MB
                        ≈ 181 GB (cluster across multiple nodes)
```

### Summary

| Metric | Value |
|--------|-------|
| Concurrent Streams | 10M |
| Streaming Egress | 50 Tbps |
| Chunks/sec | 2.5M |
| Watch History Write QPS | 23K avg / 115K peak |
| Search QPS | 5.8K avg / 29K peak |
| Catalog Read QPS | 3.5K avg / 17K peak |
| API Gateway QPS | ~200K peak |
| Video Storage | 3.6 PB |
| Watch History Storage | 146 TB (2 years) |
| Catalog Size | 500 MB |
| Redis Cluster Total | ~181 GB |

---

## 4. API Design

### Versioning & Common Headers

```
Base URL: https://api.netflix.com/v1

Common Headers:
  Authorization: Bearer {token}
  X-Request-ID: {uuid}           # Distributed tracing
  X-Client-Version: {version}    # For gradual rollouts
  X-Device-Type: {mobile|web|tv} # Device-specific responses
  X-Experiment-IDs: {id1,id2}    # A/B test assignments
```

### Auth

```
POST /v1/auth/signup
  Request:  { email, password, displayName }
  Response: { userId, accessToken, refreshToken, expiresIn }
  Status:   201 Created
  Rate Limit: 10/min per IP

POST /v1/auth/login
  Request:  { email, password, deviceId }
  Response: { accessToken, refreshToken, expiresIn, userId, profiles: [...] }
  Status:   200 OK
  Rate Limit: 5/min per email

POST /v1/auth/refresh
  Request:  { refreshToken }
  Response: { accessToken, expiresIn }
  Status:   200 OK

POST /v1/auth/logout
  Request:  { refreshToken, allDevices: boolean }
  Response: { success: true }
  Status:   200 OK
```

### User & Profiles

```
GET /v1/users/{userId}/profiles
  Response: { 
    profiles: [{ profileId, name, avatar, isKids, preferences }]
  }
  Status:   200 OK

GET /v1/profiles/{profileId}
  Response: { profileId, name, avatar, isKids, preferences, maturityRating }
  Status:   200 OK

PUT /v1/profiles/{profileId}/preferences
  Request:  { genres: [{genreId, weight}], maturityRating, audioLanguage, subtitleLanguage }
  Response: { updated: true }
  Status:   200 OK
```

### Catalog

```
GET /v1/catalog/home?profileId={profileId}
  Response: {
    rows: [
      {
        rowId, rowType, title,    # rowType: genre | continue_watching | trending | because_you_watched
        titles: [{ titleId, title, rating, thumbnailUrl, contentType, matchScore }]
      }
    ],
    experimentId: "home_layout_v3"   # A/B test tracking
  }
  Status: 200 OK
  Cache: CDN 60s, Browser 30s

GET /v1/catalog/genres/{genreId}?page=1&size=20&sort=popular
  Response: {
    genreId, genreName,
    titles: [{ titleId, title, rating, thumbnailUrl, releaseYear }],
    pagination: { page, size, totalPages, totalItems, nextCursor }
  }
  Status: 200 OK

GET /v1/titles/{titleId}
  Response: {
    titleId, title, description, releaseYear, rating, durationSec, contentType,
    maturityRating, genres: [{ genreId, name }],
    cast: [{ personId, name, role }],
    episodes: [{ episodeId, seasonNum, episodeNum, title, durationSec, thumbnailUrl }],
    similar: [{ titleId, title, matchScore }]
  }
  Status: 200 OK
```

### Search

```
GET /v1/search?q={query}&genre={genreId}&year={year}&rating={min}&page=1&size=20
  Response: {
    results: [{ titleId, title, rating, thumbnailUrl, genres, matchScore, highlightedTitle }],
    suggestions: ["stranger things", "stranger"],   # Autocomplete
    pagination: { page, size, totalPages, totalItems },
    spellCheck: { original: "straner", corrected: "stranger" }
  }
  Status: 200 OK

GET /v1/search/suggestions?q={prefix}&limit=10
  Response: {
    suggestions: [{ text, type: "title|person|genre" }]
  }
  Status: 200 OK
```

### Streaming

```
GET /v1/stream/{titleId}/manifest?profileId={profileId}&episodeId={episodeId}
  Response: {
    playbackId: "uuid",          # For analytics correlation
    manifestUrl: "https://oc.netflix.com/{signed-path}/master.m3u8",
    licenseUrl: "https://license.netflix.com/widevine",
    drmConfig: {
      widevine: { licenseUrl, certificateUrl },
      fairplay: { licenseUrl, certificateUrl },
      playready: { licenseUrl }
    },
    resumePositionSec: 1234,
    expiresAt: "2026-02-26T12:00:00Z",
    availableResolutions: ["480p", "720p", "1080p", "4K"],
    audioTracks: [{ language, format }],
    subtitles: [{ language, url }]
  }
  Status: 200 OK

POST /v1/stream/{playbackId}/heartbeat
  Request:  { currentPositionSec, bitrateKbps, bufferHealthSec, quality }
  Response: { continue: true }
  Status:   200 OK
  Note:     Sent every 30s for session validation and analytics
```

### Watch History

```
POST /v1/profiles/{profileId}/history
  Request:  { titleId, episodeId?, progressSec, durationSec, timestamp, completed }
  Response: { recorded: true }
  Status:   202 Accepted (async processing)
  Note:     Batched client-side, sent every 30s or on pause/exit

GET /v1/profiles/{profileId}/history?page=1&size=50
  Response: {
    items: [{ titleId, title, thumbnailUrl, watchedAt, progressSec, durationSec, completed }],
    pagination: { page, size, totalPages, nextCursor }
  }
  Status: 200 OK

GET /v1/profiles/{profileId}/continue-watching
  Response: {
    items: [{ 
      titleId, title, thumbnailUrl, 
      episodeId?, seasonNum?, episodeNum?, episodeTitle?,
      progressSec, durationSec, progressPercent 
    }]
  }
  Status: 200 OK
```

### Trending & Recommendations

```
GET /v1/trending?genre={genreId}&window=24h&region={region}
  Response: {
    window: "24h",
    genre: "action",
    region: "US",
    titles: [{ titleId, title, thumbnailUrl, viewCount, rank, movement }]
  }
  Status: 200 OK
  Cache: 5 min

GET /v1/profiles/{profileId}/recommendations
  Response: {
    forYou: [{ titleId, title, thumbnailUrl, reason, score }],
    becauseYouWatched: [
      { basedOn: { titleId, title }, titles: [{ titleId, title, score }] }
    ],
    topPicks: [{ titleId, title, percentMatch }],
    modelVersion: "rec_v4.2.1"   # For debugging/experiments
  }
  Status: 200 OK
```

---

## 5. Data Modeling & Indexing

### PostgreSQL — Users & Profiles (Sharded by user_id)

```sql
-- Shard key: user_id (hash)
users (
  user_id         UUID PRIMARY KEY,
  email           VARCHAR(255) UNIQUE NOT NULL,
  password_hash   VARCHAR(255) NOT NULL,
  created_at      TIMESTAMP DEFAULT NOW(),
  updated_at      TIMESTAMP DEFAULT NOW(),
  account_status  VARCHAR(20) DEFAULT 'ACTIVE',
  region          VARCHAR(10) NOT NULL
)

profiles (
  profile_id      UUID PRIMARY KEY,
  user_id         UUID NOT NULL,  -- Shard key (co-located with users)
  name            VARCHAR(100) NOT NULL,
  avatar_url      VARCHAR(500),
  is_kids         BOOLEAN DEFAULT FALSE,
  maturity_rating VARCHAR(10) DEFAULT 'TV-MA',
  created_at      TIMESTAMP DEFAULT NOW(),
  FOREIGN KEY (user_id) REFERENCES users(user_id)
)

profile_preferences (
  profile_id    UUID REFERENCES profiles(profile_id),
  genre_id      UUID,
  weight        FLOAT DEFAULT 1.0,
  PRIMARY KEY (profile_id, genre_id)
)

-- Indexes
CREATE INDEX idx_users_email ON users(email);                    -- Login lookup
CREATE INDEX idx_profiles_user ON profiles(user_id);             -- Fetch all profiles for user
CREATE INDEX idx_profile_prefs ON profile_preferences(profile_id);
```

### PostgreSQL — Catalog (Read Replicas, No Sharding)

```sql
-- No sharding needed: 500 MB fits in memory, use read replicas for scale
genres (
  genre_id       UUID PRIMARY KEY,
  name           VARCHAR(100) UNIQUE NOT NULL,
  display_order  INT,
  parent_genre_id UUID REFERENCES genres(genre_id)  -- Hierarchy support
)

titles (
  title_id       UUID PRIMARY KEY,
  title          VARCHAR(500) NOT NULL,
  description    TEXT,
  release_year   INT,
  rating         DECIMAL(3,1),
  duration_sec   INT,
  content_type   VARCHAR(20),      -- MOVIE | SERIES
  maturity_rating VARCHAR(10),
  thumbnail_url  VARCHAR(1000),
  backdrop_url   VARCHAR(1000),
  manifest_key   VARCHAR(500),
  popularity_score FLOAT DEFAULT 0,
  created_at     TIMESTAMP DEFAULT NOW(),
  updated_at     TIMESTAMP DEFAULT NOW()
)

title_genres (
  title_id  UUID REFERENCES titles(title_id),
  genre_id  UUID REFERENCES genres(genre_id),
  is_primary BOOLEAN DEFAULT FALSE,
  PRIMARY KEY (title_id, genre_id)
)

episodes (
  episode_id     UUID PRIMARY KEY,
  title_id       UUID REFERENCES titles(title_id),
  season_num     INT,
  episode_num    INT,
  episode_title  VARCHAR(500),
  description    TEXT,
  duration_sec   INT,
  thumbnail_url  VARCHAR(500),
  manifest_key   VARCHAR(500),
  release_date   DATE,
  UNIQUE (title_id, season_num, episode_num)
)

persons (
  person_id      UUID PRIMARY KEY,
  name           VARCHAR(255) NOT NULL,
  bio            TEXT,
  image_url      VARCHAR(500)
)

title_cast (
  title_id       UUID REFERENCES titles(title_id),
  person_id      UUID REFERENCES persons(person_id),
  role           VARCHAR(50),      -- ACTOR | DIRECTOR | WRITER
  character_name VARCHAR(255),
  billing_order  INT,
  PRIMARY KEY (title_id, person_id, role)
)

-- Indexes
CREATE INDEX idx_title_genres_genre ON title_genres(genre_id);
CREATE INDEX idx_titles_release ON titles(release_year DESC);
CREATE INDEX idx_titles_rating ON titles(rating DESC);
CREATE INDEX idx_titles_popularity ON titles(popularity_score DESC);
CREATE INDEX idx_episodes_title ON episodes(title_id, season_num, episode_num);
CREATE INDEX idx_title_cast_person ON title_cast(person_id);
```

### Cassandra — Watch History (Partitioned by profile_id + time bucket)

```sql
-- Partition: (profile_id, year_month) to prevent hot partitions from power users
-- Clustering: watched_at DESC for time-ordered retrieval
CREATE TABLE watch_history (
  profile_id    UUID,
  year_month    TEXT,             -- '2026-02' for time bucketing
  watched_at    TIMESTAMP,
  title_id      UUID,
  episode_id    UUID,
  progress_sec  INT,
  duration_sec  INT,
  completed     BOOLEAN,
  device_type   TEXT,
  PRIMARY KEY ((profile_id, year_month), watched_at)
) WITH CLUSTERING ORDER BY (watched_at DESC)
  AND default_time_to_live = 63072000    -- 2 years TTL
  AND compaction = {'class': 'TimeWindowCompactionStrategy', 'compaction_window_size': 7, 'compaction_window_unit': 'DAYS'};

-- Materialized view for resume position (latest per title)
CREATE TABLE watch_progress (
  profile_id    UUID,
  title_id      UUID,
  episode_id    UUID,
  progress_sec  INT,
  duration_sec  INT,
  updated_at    TIMESTAMP,
  PRIMARY KEY (profile_id, title_id)
);

-- Analytics table for ML training (separate keyspace, higher retention)
CREATE TABLE watch_events_analytics (
  event_date    DATE,
  profile_id    UUID,
  title_id      UUID,
  event_type    TEXT,             -- START | PROGRESS | COMPLETE | PAUSE | SEEK
  position_sec  INT,
  timestamp     TIMESTAMP,
  PRIMARY KEY ((event_date), timestamp, profile_id)
) WITH default_time_to_live = 94608000;  -- 3 years for ML
```

### Redis Key Patterns & Data Structures

```
=== Session & Auth ===
session:{sessionId}                    → JSON {userId, profileId, roles, deviceId}     TTL 1h
refresh:{refreshToken}                 → JSON {userId, deviceId, createdAt}            TTL 30d
user:sessions:{userId}                 → SET {sessionId1, sessionId2, ...}             TTL 30d

=== Catalog Read Path (CQRS) ===
catalog:genre:{genreId}                → SORTED SET {titleId → popularityScore}        No TTL
title:meta:{titleId}                   → HASH {title, rating, thumbnailUrl, contentType, releaseYear, maturityRating}
title:detail:{titleId}                 → JSON {full title details including cast}      TTL 1h
genre:list                             → LIST [genreId1, genreId2, ...] in display order

=== Personalized Feeds ===
home_feed:{profileId}                  → JSON {serialized personalized feed}           TTL 5m
profile:prefs:{profileId}              → HASH {genreId → weight}                       TTL 7d
recs:{profileId}                       → JSON {recommendation results}                 TTL 1h

=== Watch Progress ===
resume:{profileId}                     → HASH {titleId → JSON{episodeId, progressSec, updatedAt}}  TTL 30d
continue:{profileId}                   → LIST [titleId1, titleId2, ...] ordered by recency         TTL 7d

=== Streaming ===
manifest:{titleId}:{resolution}        → JSON {signed URLs, DRM config}                TTL 5m
playback:{playbackId}                  → HASH {profileId, titleId, startedAt, lastHeartbeat}       TTL 4h

=== Trending ===
trending:global:24h                    → SORTED SET {titleId → viewCount}              TTL 25h
trending:global:7d                     → SORTED SET {titleId → viewCount}              TTL 8d
trending:{genreId}:24h                 → SORTED SET {titleId → viewCount}              TTL 25h
trending:{region}:24h                  → SORTED SET {titleId → viewCount}              TTL 25h

=== Rate Limiting ===
ratelimit:{endpoint}:{identifier}      → COUNT                                         TTL varies
ratelimit:login:{email}                → COUNT                                         TTL 15m

=== A/B Testing ===
experiment:{experimentId}:assignments  → HASH {userId → variant}                       TTL 30d
experiment:{experimentId}:config       → JSON {variants, weights, metrics}             No TTL
```

### Elasticsearch Index

```json
{
  "index": "titles",
  "settings": {
    "number_of_shards": 10,
    "number_of_replicas": 2,
    "analysis": {
      "analyzer": {
        "title_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding", "edge_ngram_filter"]
        },
        "search_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding"]
        }
      },
      "filter": {
        "edge_ngram_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 20
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title_id":       { "type": "keyword" },
      "title":          { "type": "text", "analyzer": "title_analyzer", "search_analyzer": "search_analyzer" },
      "title_exact":    { "type": "keyword" },
      "description":    { "type": "text", "analyzer": "standard" },
      "genres":         { "type": "keyword" },
      "actors":         { "type": "text", "fields": { "raw": { "type": "keyword" } } },
      "directors":      { "type": "text", "fields": { "raw": { "type": "keyword" } } },
      "release_year":   { "type": "integer" },
      "rating":         { "type": "float" },
      "maturity_rating": { "type": "keyword" },
      "content_type":   { "type": "keyword" },
      "tags":           { "type": "keyword" },
      "popularity":     { "type": "float" },
      "region_availability": { "type": "keyword" },
      "suggest": {
        "type": "completion",
        "analyzer": "simple",
        "contexts": [
          { "name": "region", "type": "category" }
        ]
      }
    }
  }
}
```

---

## 6. Sharding & Partitioning Strategy

| Component | Strategy | Key | Rationale | Hot Spot Risk | Mitigation |
|-----------|----------|-----|-----------|---------------|------------|
| Users DB (PG) | Hash shard | user_id | Even distribution, all queries by user_id | Low — UUIDs distribute evenly | None needed |
| Profiles | Co-located | user_id | Profiles always fetched with user | None — same shard as user | None needed |
| Catalog DB (PG) | No shard | — | 500 MB fits in memory | None — small dataset | 10+ read replicas |
| Cassandra History | Composite partition | (profile_id, year_month) | Bounds partition size, enables TTL | Power users in same month | Monthly bucket limits growth |
| Cassandra Progress | Simple partition | profile_id | Single row per title, small | None — bounded by catalog size | None needed |
| Redis Cluster | Hash slots | Key prefix | Auto-distributed across 16384 slots | Trending sorted sets | Read replicas for hot keys |
| Elasticsearch | Default + Routing | — (no routing) | Cross-genre searches hit all shards anyway | Popular searches | Query caching, replicas |
| Kafka | Topic-based | Varies by topic | Different access patterns need different partitioning | Viral content | See Kafka section |
| S3 / Open Connect | Prefix-based | {hash}/{titleId}/ | S3 partitions by prefix | New releases | Random prefix hash |

### Kafka Topic Partitioning

```
Topic: watch-events
  Partitions: 256
  Key: profileId
  Rationale: Ordering per user, consumer groups can parallelize
  Consumers: History writers, Analytics

Topic: catalog-updates  
  Partitions: 16
  Key: titleId
  Rationale: Low volume, ordering per title
  Consumers: Cache invalidators, Search indexers

Topic: trending-aggregation
  Partitions: 64
  Key: titleId
  Rationale: Aggregation by title, enables parallel counting
  Consumers: Trending workers

Topic: recommendation-events
  Partitions: 128
  Key: profileId
  Rationale: ML feature updates per user
  Consumers: Feature store updaters
```

---

## 7. High-Level Design

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                         CLIENTS                                          │
│                              Mobile · Web · Smart TV · Gaming                            │
└───────────────────────────────────────┬─────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    │                                       │
                    ▼                                       ▼
        ┌───────────────────┐                   ┌───────────────────────┐
        │     GeoDNS        │                   │   Open Connect CDN    │
        │   (Route 53)      │                   │   (Edge Appliances)   │
        └─────────┬─────────┘                   │                       │
                  │                             │  ┌─────────────────┐  │
                  │                             │  │ ISP PoP Cache   │  │
                  │                             │  │ (Video Chunks)  │  │
                  │                             │  └────────┬────────┘  │
                  │                             │           │           │
                  │                             │  ┌────────▼────────┐  │
                  │                             │  │ Regional Cache  │  │
                  │                             │  │ (Origin Shield) │  │
                  │                             │  └────────┬────────┘  │
                  │                             └───────────┼───────────┘
                  │                                         │
                  ▼                                         │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    API LAYER                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │   WAF / DDoS    │  │  Rate Limiter   │  │   API Gateway   │  │  Load Balancer  │    │
│  │  (CloudFlare)   │  │    (Redis)      │  │   (Kong/Envoy)  │  │    (Regional)   │    │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │
│           └────────────────────┴────────────────────┴────────────────────┘              │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┴───────────────────────────────┐
                    │                   SERVICE MESH                     │
                    │              (Istio / Consul Connect)              │
                    └───────────────────┬───────────────────────────────┘
                                        │
        ┌───────────┬───────────┬───────┴───────┬───────────┬───────────┐
        ▼           ▼           ▼               ▼           ▼           ▼
┌─────────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌─────────┐ ┌─────────────┐
│    Auth     │ │  User   │ │ Catalog │ │  Search   │ │Streaming│ │   Watch     │
│   Service   │ │ Service │ │ Service │ │  Service  │ │ Service │ │  History    │
└──────┬──────┘ └────┬────┘ └────┬────┘ └─────┬─────┘ └────┬────┘ └──────┬──────┘
       │             │           │             │            │             │
       │             │           │             │            │             │
       ▼             ▼           ▼             ▼            ▼             ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              DATA LAYER                                       │
│                                                                               │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐     │
│  │   Redis Cluster    │  │   Redis Cluster    │  │   Redis Cluster    │     │
│  │    (Sessions)      │  │  (Catalog Cache)   │  │    (Trending)      │     │
│  └────────────────────┘  └────────────────────┘  └────────────────────┘     │
│                                                                               │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐     │
│  │ PostgreSQL Cluster │  │ PostgreSQL Cluster │  │    Cassandra       │     │
│  │  (Users - Sharded) │  │ (Catalog - Replica)│  │  (Watch History)   │     │
│  └────────────────────┘  └────────────────────┘  └────────────────────┘     │
│                                                                               │
│  ┌────────────────────┐  ┌────────────────────┐                              │
│  │   Elasticsearch    │  │        S3          │                              │
│  │  (Search Index)    │  │  (Video Storage)   │                              │
│  └────────────────────┘  └────────────────────┘                              │
└──────────────────────────────────────────────────────────────────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    │              KAFKA CLUSTER            │
                    │         (Event Streaming Bus)         │
                    └───────────────────┬───────────────────┘
                                        │
        ┌───────────────┬───────────────┼───────────────┬───────────────┐
        ▼               ▼               ▼               ▼               ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Trending   │ │ Analytics   │ │    ML       │ │   Cache     │ │  Search     │
│  Workers    │ │  Pipeline   │ │  Training   │ │ Invalidator │ │  Indexer    │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
                        │               │
                        ▼               ▼
                ┌─────────────┐ ┌─────────────────────────────────┐
                │  Data Lake  │ │     ML PLATFORM                 │
                │ (Spark/S3)  │ │  ┌───────────┐ ┌─────────────┐  │
                └─────────────┘ │  │  Feature  │ │   Model     │  │
                                │  │   Store   │ │   Serving   │  │
                                │  └───────────┘ └─────────────┘  │
                                └─────────────────────────────────┘
                                            │
                                            ▼
                                ┌─────────────────────┐
                                │  Recommendation     │
                                │     Service         │
                                └─────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                           OBSERVABILITY LAYER                                 │
│                                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Jaeger    │  │ Prometheus  │  │     ELK     │  │  PagerDuty  │         │
│  │  (Tracing)  │  │  (Metrics)  │  │  (Logging)  │  │  (Alerting) │         │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                              DRM LAYER                                        │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                       License Server Cluster                         │    │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────────────┐ │    │
│  │  │ Widevine  │  │ FairPlay  │  │ PlayReady │  │   Key Management  │ │    │
│  │  │  (Android,│  │  (Apple)  │  │ (Windows) │  │   (HSM Backed)    │ │    │
│  │  │  Chrome)  │  │           │  │           │  │                   │ │    │
│  │  └───────────┘  └───────────┘  └───────────┘  └───────────────────┘ │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Detailed Component Design

### 8.1 Open Connect CDN Architecture

Netflix's custom CDN with edge appliances placed inside ISPs:

```
┌─────────────────────────────────────────────────────────────────┐
│                     OPEN CONNECT ARCHITECTURE                    │
└─────────────────────────────────────────────────────────────────┘

User Request Flow:
                                                    
  ┌────────┐    1. DNS     ┌─────────────┐         
  │ Client │──────────────▶│   GeoDNS    │         
  └────────┘               └──────┬──────┘         
       │                          │                 
       │                          │ 2. Return nearest OCA IP
       │                          ▼                 
       │                   ┌─────────────┐         
       │                   │   Steering  │  (Real-time health + load)
       │                   │   Service   │         
       │                   └─────────────┘         
       │                                           
       │  3. Request chunk from assigned OCA       
       │                                           
       ▼                                           
┌─────────────────────────────────────────────────────────────────┐
│                         ISP NETWORK                              │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                   Open Connect Appliance (OCA)               │ │
│  │  ┌─────────┐  ┌─────────────┐  ┌──────────────────────────┐ │ │
│  │  │  nginx  │  │ Flash-based │  │     Cache Population     │ │ │
│  │  │ (HTTP)  │  │   Storage   │  │   (Overnight proactive)  │ │ │
│  │  └─────────┘  │   100TB+    │  └──────────────────────────┘ │ │
│  │               └─────────────┘                                │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              │                                    │
│                              │ Cache Miss (rare)                  │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                      Regional OCA Hub                        │ │
│  │                    (Larger cache, IXP)                       │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Cache Miss (very rare)
                              ▼
                    ┌─────────────────┐
                    │   S3 Origin     │
                    │  (AWS Region)   │
                    └─────────────────┘

Key Design Points:
- OCAs pre-populated overnight with predicted popular content
- Steering service routes users based on: geography, OCA health, ISP peering, load
- 95%+ cache hit rate at ISP-level OCAs
- Sub-10ms latency from OCA to user (same ISP network)
```

### 8.2 Streaming Service & DRM

```
┌─────────────────────────────────────────────────────────────────┐
│                    STREAMING SERVICE DESIGN                      │
└─────────────────────────────────────────────────────────────────┘

Play Request Flow:

┌────────┐  1. GET /stream/{titleId}/manifest
│ Client │─────────────────────────────────────────┐
└────────┘                                         │
                                                   ▼
                                          ┌─────────────────┐
                                          │   Streaming     │
                                          │    Service      │
                                          └────────┬────────┘
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    │                              │                              │
                    ▼                              ▼                              ▼
           ┌─────────────────┐          ┌─────────────────┐           ┌─────────────────┐
           │  Redis: Check   │          │  Entitlement    │           │   Resume        │
           │ manifest cache  │          │    Check        │           │   Position      │
           └────────┬────────┘          │ (subscription)  │           │   (Redis)       │
                    │                   └─────────────────┘           └─────────────────┘
                    │                              
           Cache Miss?                             
                    │                              
                    ▼                              
           ┌─────────────────┐                    
           │  Build Manifest │                    
           │  - Get S3 paths │                    
           │  - Sign URLs    │                    
           │  - Get DRM keys │                    
           └────────┬────────┘                    
                    │                              
                    ▼                              
           ┌─────────────────┐                    
           │  Cache in Redis │                    
           │    TTL: 5min    │                    
           └────────┬────────┘                    
                    │                              
                    ▼                              
           Return to Client:
           {
             manifestUrl: "signed HLS URL",
             drmConfig: {
               widevine: { licenseUrl, cert },
               fairplay: { licenseUrl, cert }
             },
             resumePositionSec: 1234
           }


DRM License Flow:

┌────────┐  1. License Request (with challenge)
│ Client │───────────────────────────────────────────────────────┐
└────────┘                                                       │
                                                                 ▼
                                                    ┌────────────────────┐
                                                    │   License Server   │
                                                    │    (Per DRM)       │
                                                    └──────────┬─────────┘
                                                               │
                                    ┌──────────────────────────┼─────────────────────────┐
                                    │                          │                         │
                                    ▼                          ▼                         ▼
                           ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
                           │ Validate Token  │      │  Check Device   │      │  Key Management │
                           │ (JWT + Session) │      │   Trust Level   │      │  System (HSM)   │
                           └─────────────────┘      └─────────────────┘      └─────────────────┘
                                                                                      │
                                                                                      ▼
                                                                              Return License
                                                                              (encrypted keys)

Security Levels:
- L1 (Hardware): 4K/HDR allowed
- L3 (Software): Max 720p
- Device attestation validates security level
```

### 8.3 Watch Progress Consistency

```
┌─────────────────────────────────────────────────────────────────┐
│                 WATCH PROGRESS WRITE FLOW                        │
│              (Ensuring Consistency Across Stores)                │
└─────────────────────────────────────────────────────────────────┘

Client sends progress every 30 seconds:

┌────────┐  POST /history { progressSec: 1234 }
│ Client │────────────────────────────────────────┐
└────────┘                                        │
                                                  ▼
                                        ┌─────────────────┐
                                        │  Watch History  │
                                        │    Service      │
                                        └────────┬────────┘
                                                 │
                                                 │
┌────────────────────────────────────────────────┴────────────────────────────────────────┐
│                                WRITE STRATEGY                                            │
│                                                                                          │
│   Step 1: Write to Cassandra FIRST (Source of Truth)                                    │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │  INSERT INTO watch_progress (profile_id, title_id, progress_sec, ...)           │   │
│   │  WITH consistency = QUORUM                                                       │   │
│   │                                                                                  │   │
│   │  IF write fails → Return 503, client retries with exponential backoff           │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                                │
│                                         │ Success                                        │
│                                         ▼                                                │
│   Step 2: Update Redis (Best Effort, Async)                                             │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │  HSET resume:{profileId} {titleId} "{progressSec, episodeId, ts}"               │   │
│   │  LPUSH continue:{profileId} {titleId} (with dedup + trim)                       │   │
│   │                                                                                  │   │
│   │  IF write fails → Log error, DO NOT fail the request                            │   │
│   │                → Background job reconciles from Cassandra                       │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                                │
│                                         │                                                │
│                                         ▼                                                │
│   Step 3: Publish to Kafka (Fire and Forget)                                            │
│   ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│   │  Produce to 'watch-events' topic                                                 │   │
│   │  Key: profileId (ensures ordering per user)                                      │   │
│   │  Acks: 1 (leader only — acceptable for analytics)                               │   │
│   │                                                                                  │   │
│   │  IF publish fails → Log, continue (analytics can tolerate gaps)                 │   │
│   └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                          │
└──────────────────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
                              Return 202 Accepted


READ FLOW (Continue Watching):

┌────────┐  GET /continue-watching
│ Client │────────────────────────────────────────┐
└────────┘                                        │
                                                  ▼
                           ┌──────────────────────────────────────┐
                           │            Try Redis First           │
                           │   LRANGE continue:{profileId} 0 9    │
                           │   HGETALL resume:{profileId}         │
                           └───────────────────┬──────────────────┘
                                               │
                              ┌────────────────┴────────────────┐
                              │                                 │
                          Cache Hit                         Cache Miss
                              │                                 │
                              ▼                                 ▼
                      Return immediately          ┌─────────────────────────┐
                                                  │ Query Cassandra         │
                                                  │ (watch_progress table)  │
                                                  └────────────┬────────────┘
                                                               │
                                                               ▼
                                                  ┌─────────────────────────┐
                                                  │ Hydrate Redis Cache     │
                                                  │ (async, non-blocking)   │
                                                  └────────────┬────────────┘
                                                               │
                                                               ▼
                                                         Return to client
```

### 8.4 Recommendation System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   RECOMMENDATION SYSTEM                          │
└─────────────────────────────────────────────────────────────────┘

                         OFFLINE TRAINING PIPELINE
                         ─────────────────────────
                         
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Kafka     │───▶│  Spark      │───▶│   Model     │───▶│   Model     │
│ (watch      │    │  Streaming  │    │  Training   │    │  Registry   │
│  events)    │    │  + Batch    │    │  (Daily)    │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                          │                                     │
                          ▼                                     │
                   ┌─────────────┐                              │
                   │  Feature    │                              │
                   │   Store     │◀─────────────────────────────┘
                   │  (Redis +   │        Deploy new model
                   │   Feast)    │
                   └─────────────┘


                         ONLINE SERVING PIPELINE
                         ──────────────────────
                         
┌────────┐    ┌─────────────┐    ┌─────────────────────────────────────┐
│ Client │───▶│   Rec       │───▶│           FEATURE RETRIEVAL         │
│        │    │  Service    │    │                                     │
└────────┘    └─────────────┘    │  ┌─────────────┐  ┌─────────────┐  │
                    │            │  │ User        │  │ Title       │  │
                    │            │  │ Features    │  │ Features    │  │
                    │            │  │ - history   │  │ - genre     │  │
                    │            │  │ - prefs     │  │ - actors    │  │
                    │            │  │ - demo      │  │ - ratings   │  │
                    │            │  └─────────────┘  └─────────────┘  │
                    │            └─────────────────────────────────────┘
                    │                           │
                    │                           ▼
                    │            ┌─────────────────────────────────────┐
                    │            │          MODEL INFERENCE            │
                    │            │                                     │
                    │            │  ┌─────────────┐  ┌─────────────┐  │
                    │            │  │Collaborative│  │ Content     │  │
                    │            │  │ Filtering   │  │ Based       │  │
                    │            │  │ (ALS)       │  │ (Deep NN)   │  │
                    │            │  └─────────────┘  └─────────────┘  │
                    │            │         │               │          │
                    │            │         └───────┬───────┘          │
                    │            │                 ▼                   │
                    │            │         ┌─────────────┐            │
                    │            │         │  Ensemble   │            │
                    │            │         │  Ranker     │            │
                    │            │         └─────────────┘            │
                    │            └─────────────────────────────────────┘
                    │                           │
                    │                           ▼
                    │            ┌─────────────────────────────────────┐
                    │            │        POST-PROCESSING              │
                    │            │  - Diversity injection              │
                    │            │  - Business rules (new releases)    │
                    │            │  - A/B test variant selection       │
                    │            │  - Region availability filter       │
                    │            └─────────────────────────────────────┘
                    │                           │
                    ▼                           ▼
              ┌─────────────┐           Return ranked titles
              │ Cache result│           with explanations
              │ in Redis    │
              │ TTL: 1h     │
              └─────────────┘


Models Used:
─────────────
1. Collaborative Filtering (ALS)
   - User-item interaction matrix
   - Finds users with similar taste
   
2. Content-Based Neural Network
   - Title embeddings from description, cast, genre
   - User taste profile from watch history
   
3. Sequence Model (Transformer)
   - Predicts next title based on watch sequence
   - "Because you watched X" recommendations
   
4. Two-Tower Model
   - User tower: encodes user features
   - Item tower: encodes title features  
   - Fast ANN lookup for candidates

Feature Store Contents:
───────────────────────
User Features:
- watch_history_embedding (128 dims)
- genre_affinity_vector (25 dims)
- avg_session_length
- preferred_content_type
- time_of_day_pattern

Title Features:
- content_embedding (256 dims)
- genre_vector
- popularity_score
- recency_score
- avg_completion_rate
```

### 8.5 A/B Testing Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                   A/B TESTING SYSTEM                             │
└─────────────────────────────────────────────────────────────────┘

Experiment Configuration (stored in Redis + PostgreSQL):

{
  "experimentId": "home_row_order_v3",
  "status": "RUNNING",
  "startDate": "2026-02-01",
  "endDate": "2026-03-01",
  "targetPercent": 10,
  "variants": [
    { "id": "control", "weight": 50, "config": { "rowOrder": "default" } },
    { "id": "treatment_a", "weight": 25, "config": { "rowOrder": "trending_first" } },
    { "id": "treatment_b", "weight": 25, "config": { "rowOrder": "personalized_first" } }
  ],
  "metrics": ["ctr", "watch_time", "session_length"],
  "segmentation": {
    "regions": ["US", "CA"],
    "platforms": ["web", "mobile"]
  }
}


Assignment Flow:
────────────────

┌────────┐    Request with userId
│ Client │────────────────────────────────────────┐
└────────┘                                        │
                                                  ▼
                                   ┌─────────────────────────┐
                                   │   Experiment Service    │
                                   └────────────┬────────────┘
                                                │
                         ┌──────────────────────┼──────────────────────┐
                         │                      │                      │
                         ▼                      ▼                      ▼
              ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
              │ Check existing  │    │ Eligibility     │    │ Assignment      │
              │ assignment      │    │ Check           │    │ (Deterministic) │
              │ (Redis)         │    │ (segments,      │    │                 │
              │                 │    │  exclusions)    │    │ hash(userId +   │
              └─────────────────┘    └─────────────────┘    │ experimentId)   │
                      │                      │              │ % 100           │
                      │                      │              └─────────────────┘
                      │                      │                      │
                      └──────────────────────┴──────────────────────┘
                                             │
                                             ▼
                              ┌─────────────────────────┐
                              │  Store Assignment       │
                              │  Redis: experiment:{id}:│
                              │         assignments     │
                              │  (HASH: userId → variant)│
                              └─────────────────────────┘
                                             │
                                             ▼
                              Return variant config to service


Metrics Collection:
───────────────────

All events tagged with experimentId + variant:

┌────────────────────┐    ┌────────────────────┐    ┌────────────────────┐
│   Watch Events     │    │   Click Events     │    │   Search Events    │
│   + experiment     │───▶│   + experiment     │───▶│   + experiment     │
│     context        │    │     context        │    │     context        │
└────────────────────┘    └────────────────────┘    └────────────────────┘
           │                        │                        │
           └────────────────────────┼────────────────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │       Kafka         │
                         │  (experiment-events)│
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    Spark Streaming  │
                         │    Aggregation      │
                         └──────────┬──────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
         ┌─────────────────┐             ┌─────────────────┐
         │  Real-time      │             │  Statistical    │
         │  Dashboard      │             │  Analysis       │
         │  (Grafana)      │             │  (p-values,     │
         │                 │             │   confidence)   │
         └─────────────────┘             └─────────────────┘
```

---

## 9. Multi-Region Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           MULTI-REGION ACTIVE-ACTIVE DESIGN                              │
└─────────────────────────────────────────────────────────────────────────────────────────┘

                                    ┌─────────────┐
                                    │   GeoDNS    │
                                    │ (Route 53)  │
                                    └──────┬──────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
              ▼                            ▼                            ▼
    ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
    │   US-EAST       │          │   EU-WEST       │          │   APAC          │
    │   Region        │          │   Region        │          │   Region        │
    └────────┬────────┘          └────────┬────────┘          └────────┬────────┘
             │                            │                            │
    ┌────────┴────────┐          ┌────────┴────────┐          ┌────────┴────────┐
    │                 │          │                 │          │                 │
    │  ┌───────────┐  │          │  ┌───────────┐  │          │  ┌───────────┐  │
    │  │ Services  │  │          │  │ Services  │  │          │  │ Services  │  │
    │  │ (Full     │  │          │  │ (Full     │  │          │  │ (Full     │  │
    │  │  Stack)   │  │          │  │  Stack)   │  │          │  │  Stack)   │  │
    │  └───────────┘  │          │  └───────────┘  │          │  └───────────┘  │
    │                 │          │                 │          │                 │
    │  ┌───────────┐  │          │  ┌───────────┐  │          │  ┌───────────┐  │
    │  │ Redis     │  │◀────────▶│  │ Redis     │  │◀────────▶│  │ Redis     │  │
    │  │ Cluster   │  │  Async   │  │ Cluster   │  │  Async   │  │ Cluster   │  │
    │  └───────────┘  │  Sync    │  └───────────┘  │  Sync    │  └───────────┘  │
    │                 │          │                 │          │                 │
    │  ┌───────────┐  │          │  ┌───────────┐  │          │  ┌───────────┐  │
    │  │PostgreSQL │  │          │  │PostgreSQL │  │          │  │PostgreSQL │  │
    │  │ (Primary) │──┼─────────▶│  │ (Replica) │  │◀─────────┼──│ (Replica) │  │
    │  └───────────┘  │  Async   │  └───────────┘  │  Async   │  └───────────┘  │
    │                 │  Repl    │                 │  Repl    │                 │
    │  ┌───────────┐  │          │  ┌───────────┐  │          │  ┌───────────┐  │
    │  │Cassandra  │  │◀────────▶│  │Cassandra  │  │◀────────▶│  │Cassandra  │  │
    │  │ DC        │  │  Multi-  │  │ DC        │  │  Multi-  │  │ DC        │  │
    │  └───────────┘  │  DC      │  └───────────┘  │  DC      │  └───────────┘  │
    │                 │          │                 │          │                 │
    └─────────────────┘          └─────────────────┘          └─────────────────┘


Data Replication Strategy:
──────────────────────────

┌─────────────────────────────────────────────────────────────────┐
│                    DATA TYPE ROUTING                             │
├─────────────────┬───────────────────┬───────────────────────────┤
│ Data Type       │ Consistency       │ Replication               │
├─────────────────┼───────────────────┼───────────────────────────┤
│ User Auth       │ Strong            │ Write to home region,     │
│                 │                   │ sync read to all          │
├─────────────────┼───────────────────┼───────────────────────────┤
│ User Profile    │ Strong            │ Single primary region     │
│                 │                   │ per user (sticky)         │
├─────────────────┼───────────────────┼───────────────────────────┤
│ Catalog         │ Eventual (1s)     │ Primary in US-EAST,       │
│                 │                   │ replicate to all          │
├─────────────────┼───────────────────┼───────────────────────────┤
│ Watch History   │ Eventual (5s)     │ Cassandra multi-DC        │
│                 │                   │ LOCAL_QUORUM writes       │
├─────────────────┼───────────────────┼───────────────────────────┤
│ Session         │ Regional          │ No cross-region repl      │
│                 │                   │ (re-auth on failover)     │
├─────────────────┼───────────────────┼───────────────────────────┤
│ Trending        │ Regional          │ Computed per region       │
│                 │                   │ (different trends)        │
└─────────────────┴───────────────────┴───────────────────────────┘


Conflict Resolution:
────────────────────

For User Profile (Last-Writer-Wins with Vector Clock):

┌─────────────────────────────────────────────────────────────────┐
│  User updates profile in US-EAST at T1                          │
│  {name: "John", vclock: {us: 1, eu: 0}}                         │
│                                                                  │
│  Same user updates in EU-WEST at T2 (before sync)               │
│  {name: "Johnny", vclock: {us: 0, eu: 1}}                       │
│                                                                  │
│  Conflict detected during sync:                                  │
│  - Compare vector clocks: neither dominates                      │
│  - Apply LWW using physical timestamp                            │
│  - T2 > T1, so "Johnny" wins                                    │
│  - Merged: {name: "Johnny", vclock: {us: 1, eu: 1}}            │
└─────────────────────────────────────────────────────────────────┘


Failover Flow:
──────────────

1. Health checks detect US-EAST is down
2. Route 53 health check fails (3 consecutive)
3. DNS TTL expires (60s)
4. Traffic routes to EU-WEST
5. Users re-authenticate (session not shared)
6. Watch history available (Cassandra multi-DC)
7. Some profile updates may be lost (async replication lag)

RTO: ~2 minutes (DNS propagation)
RPO: ~5 seconds (replication lag)
```

---

## 10. HLD Walkthrough

### Step 1 — User Opens App (Login)

1. Client resolves `api.netflix.com` via GeoDNS → nearest regional endpoint
2. Request passes through WAF (DDoS protection) → Rate Limiter → API Gateway
3. `POST /v1/auth/login` → Auth Service
4. Auth Service:
   - Rate limit check: `INCR ratelimit:login:{email}` in Redis
   - Verify credentials against PostgreSQL (user's home region)
   - Generate JWT (access token: 1h, refresh token: 30d)
   - Create session: `SET session:{sessionId}` in Redis
   - Return tokens + profile list
5. Client stores tokens securely

### Step 2 — Home Feed Loads (CQRS Read Path)

**Key: Read path NEVER touches PostgreSQL**

1. `GET /v1/catalog/home?profileId={id}` → Catalog Service
2. **L1 Cache Check:** `GET home_feed:{profileId}` (TTL 5 min)
   - **HIT** → Return immediately (~1ms)
   - **MISS** → Continue to L2
3. **L2 — Redis Sorted Sets:**
   ```
   PIPELINE:
     ZREVRANGE catalog:genre:{genreId1} 0 19
     ZREVRANGE catalog:genre:{genreId2} 0 19
     ... (25 genres in parallel)
   ```
4. **Hydrate Metadata:**
   ```
   PIPELINE:
     HGETALL title:meta:{titleId1}
     HGETALL title:meta:{titleId2}
     ... (500 titles in one batch)
   ```
5. **Get User Preferences:** `HGETALL profile:prefs:{profileId}`
6. **Personalization:**
   - Call Recommendation Service for personalized ranking
   - Apply A/B test variant config
   - Inject "Continue Watching" row from `LRANGE continue:{profileId}`
7. **Cache Result:** `SET home_feed:{profileId}` with TTL 5 min
8. Return assembled feed with experimentId for tracking

### Step 3 — User Searches

1. `GET /v1/search?q=stranger` → Search Service
2. Query Elasticsearch:
   ```json
   {
     "query": {
       "bool": {
         "should": [
           { "match": { "title": { "query": "stranger", "boost": 3 } } },
           { "match": { "actors": "stranger" } },
           { "match": { "description": "stranger" } }
         ]
       }
     },
     "suggest": {
       "title-suggest": {
         "prefix": "stranger",
         "completion": { "field": "suggest" }
       }
     }
   }
   ```
3. Return ranked results with fuzzy matching and suggestions

### Step 4 — User Clicks Play

**4a. Manifest Request:**
1. `GET /v1/stream/{titleId}/manifest` → Streaming Service
2. Check entitlement (subscription active?)
3. Check Redis: `GET manifest:{titleId}:{resolution}`
   - **HIT** → Return cached manifest
   - **MISS** → Build manifest:
     - Fetch manifest template from S3
     - Generate signed URLs for Open Connect CDN (5 min expiry)
     - Get DRM license URLs for all supported systems
     - Cache in Redis with TTL 5 min
4. Get resume position: `HGET resume:{profileId} {titleId}`
5. Return manifest with DRM config and resume position

**4b. Video Playback:**
1. Client parses HLS manifest
2. Client requests license from License Server (Widevine/FairPlay/PlayReady)
3. License Server validates session + device, returns decryption keys
4. Client fetches video chunks from Open Connect CDN:
   - OCA in user's ISP serves chunk (95% hit rate)
   - On miss, OCA fetches from regional hub or S3 origin
5. Client uses ABR to switch quality based on bandwidth

**4c. Heartbeat (Every 30s):**
```
POST /v1/stream/{playbackId}/heartbeat
{ currentPositionSec, bitrateKbps, bufferHealthSec }
```
- Validates session still active
- Updates `playback:{playbackId}` TTL in Redis
- Feeds real-time analytics

### Step 5 — Progress Tracking

Every 30 seconds (and on pause/exit):

1. `POST /v1/profiles/{profileId}/history` → Watch History Service
2. **Write to Cassandra FIRST** (CL=LOCAL_QUORUM):
   ```sql
   INSERT INTO watch_progress (profile_id, title_id, progress_sec, ...)
   INSERT INTO watch_history (profile_id, year_month, watched_at, ...)
   ```
3. **Update Redis** (best effort):
   ```
   HSET resume:{profileId} {titleId} {progressJson}
   LPUSH continue:{profileId} {titleId}
   LTRIM continue:{profileId} 0 19
   ```
4. **Publish to Kafka** (fire-and-forget):
   - Topic: `watch-events`
   - Key: `profileId`
5. Return `202 Accepted`

### Step 6 — Trending Computation (Async)

1. Trending Workers consume from `watch-events` Kafka topic
2. For each event:
   ```
   ZINCRBY trending:global:24h 1 {titleId}
   ZINCRBY trending:{genreId}:24h 1 {titleId}
   ZINCRBY trending:{region}:24h 1 {titleId}
   ```
3. Sliding window: Separate sorted sets for each time window
4. When user requests `GET /v1/trending`:
   ```
   ZREVRANGE trending:global:24h 0 49 WITHSCORES
   ```

### Step 7 — Content Ingestion (Admin)

1. Admin uploads video to S3 ingest bucket
2. Transcoding pipeline triggered:
   - Encode to multiple resolutions (480p, 720p, 1080p, 4K)
   - Multiple codecs (H.264, H.265)
   - Multiple audio tracks (stereo, 5.1, Atmos)
   - Generate HLS manifests
3. Store encoded chunks in S3 with predictable paths
4. Publish `content-ready` event to Kafka
5. Workers:
   - Update Elasticsearch index
   - Invalidate relevant caches
   - Trigger Open Connect proactive fill (popular content)
   - Notify Recommendation Service for embedding generation

### Step 8 — Catalog Update (CQRS Write Path)

1. Admin updates title metadata via internal API
2. Catalog Service writes to PostgreSQL (source of truth)
3. Publish `catalog-updated` event to Kafka
4. Cache Invalidator Worker:
   - Rebuild `catalog:genre:{genreId}` sorted sets
   - Update `title:meta:{titleId}` hash
   - Delete affected `home_feed:*` keys
   - Update Elasticsearch index
5. Propagation time: ~1-2 seconds (eventual consistency)

---

## 11. Trade-offs

| Decision | Chose | Over | Why |
|----------|-------|------|-----|
| CDN | Open Connect (own CDN) | Third-party (CloudFront) | 50 Tbps egress, cost control, ISP placement |
| Watch History DB | Cassandra | PostgreSQL | 115K peak writes/sec, linear write scalability, multi-DC |
| Search | Elasticsearch | PostgreSQL LIKE | Fuzzy, ranked, faceted, autocomplete at 30K QPS |
| Trending | Redis ZINCRBY | DB aggregation | O(log N) increment, instant top-K reads |
| Catalog Read Path | Redis CQRS | Direct PostgreSQL | 1ms vs 200ms, zero DB load on reads |
| Consistency | Eventual for analytics | Strong everywhere | Throughput over precision for trending/history |
| Event Bus | Kafka | RabbitMQ | Replay, ordering, partitioning, 100K+ msg/sec |
| Feed Strategy | Pull + cache | Pre-compute (fan-out on write) | 50K titles × 100M users = 5T entries impossible |
| Multi-region | Active-Active | Active-Passive | Sub-100ms latency globally, no failover delay |
| Session Storage | Regional Redis | Global replication | Latency; re-auth on failover acceptable |
| DRM | Multi-DRM | Single provider | Device coverage (Widevine, FairPlay, PlayReady) |
| ABR Protocol | HLS | DASH | Apple ecosystem support, wider device compatibility |
| Progress Sync | Cassandra-first | Redis-first | Durability over latency; Redis is cache |

---

## 12. Fault Tolerance & Recovery

| Component | Failure Mode | Impact | Detection | Mitigation | RTO |
|-----------|-------------|--------|-----------|------------|-----|
| Auth Service | Service down | No new logins | Health check | Existing JWTs valid, circuit breaker, multi-instance | 30s |
| PostgreSQL Primary | Node crash | No writes | Replication lag spike | Auto-failover to sync replica, read path unaffected | 30s |
| Redis Cluster | Node failure | Cache miss spike | Cluster health | Redis Cluster auto-failover, fallback to DB | 10s |
| Cassandra Node | Node down | Reduced capacity | Gossip protocol | RF=3, LOCAL_QUORUM still serves | 0s |
| Kafka Broker | Broker crash | Event delay | ISR shrinkage | RF=3, min.insync=2, producer retries | 5s |
| Open Connect OCA | Appliance failure | Stream interruption | Health probe | Steering redirects to next OCA | 2s |
| Elasticsearch | Node failure | Search degraded | Cluster health | 2 replicas, fallback to basic search | 30s |
| License Server | Service down | No new playback | Health check | Multi-AZ, cached licenses still valid | 30s |
| Recommendation | Service down | Generic feed | Circuit breaker | Fallback to trending/popular from Redis | 0s |
| Full Region | Region outage | Regional users affected | Route 53 health | GeoDNS failover to nearest region | 2 min |

### Circuit Breaker Configuration

```yaml
circuitBreaker:
  catalog-service:
    failureRateThreshold: 50        # Open after 50% failures
    waitDurationInOpenState: 30s    # Wait before half-open
    permittedCallsInHalfOpen: 10    # Test calls
    slidingWindowSize: 100          # Sample size
    
  recommendation-service:
    failureRateThreshold: 30        # Lower threshold (non-critical)
    waitDurationInOpenState: 60s
    fallback: trending-service      # Explicit fallback
```

### Graceful Degradation Hierarchy

```
Level 0 (Normal):     Full personalization + recommendations
Level 1 (Rec down):   Personal history + trending
Level 2 (Redis down): Database fallback + basic search
Level 3 (Regional):   Route to alternate region
Level 4 (Global):     Static emergency page with status
```

---

## 13. Observability

### Distributed Tracing (Jaeger)

```
Every request gets trace context:
  X-Request-ID: {traceId}
  
Trace spans:
  api-gateway → auth-service → redis
                            → postgresql
  api-gateway → catalog-service → redis
                               → recommendation-service → feature-store
                                                       → model-server
```

### Metrics (Prometheus + Grafana)

```yaml
Key Metrics:

# Latency
http_request_duration_seconds{service, endpoint, status}
cassandra_query_latency_seconds{keyspace, table, operation}
redis_command_duration_seconds{command}

# Throughput  
http_requests_total{service, endpoint, status}
kafka_messages_produced_total{topic}
kafka_consumer_lag{topic, consumer_group}

# Saturation
redis_memory_used_bytes
cassandra_pending_compactions
postgresql_connections_active

# Errors
http_errors_total{service, endpoint, error_type}
circuit_breaker_state{service, state}

# Business
active_streams_total{region, quality}
playback_starts_total{device_type}
search_queries_total{result_count_bucket}
```

### Alerting Rules

```yaml
alerts:
  - name: HighLatencyP99
    condition: histogram_quantile(0.99, http_request_duration_seconds) > 0.5
    for: 5m
    severity: warning
    
  - name: ErrorRateSpike
    condition: rate(http_errors_total[5m]) / rate(http_requests_total[5m]) > 0.01
    for: 2m
    severity: critical
    
  - name: CassandraWriteLatency
    condition: cassandra_query_latency_seconds{operation="write"} > 0.1
    for: 5m
    severity: warning
    
  - name: KafkaConsumerLag
    condition: kafka_consumer_lag > 100000
    for: 10m
    severity: warning
    
  - name: StreamingEgressDrop
    condition: rate(streaming_bytes_served_total[1m]) < 40000000000000  # 40 Tbps
    for: 5m
    severity: critical
```

### Log Aggregation (ELK)

```json
{
  "timestamp": "2026-02-26T10:30:00Z",
  "level": "INFO",
  "service": "streaming-service",
  "traceId": "abc123",
  "spanId": "def456",
  "userId": "user-789",
  "profileId": "profile-012",
  "action": "manifest_request",
  "titleId": "title-345",
  "latencyMs": 12,
  "cacheHit": true,
  "region": "us-east-1"
}
```

---

## 14. Security Considerations

### Authentication & Authorization

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                               │
├─────────────────────────────────────────────────────────────────┤
│ Layer 1: Edge Security                                          │
│   - WAF rules (OWASP Top 10)                                    │
│   - DDoS protection (CloudFlare/AWS Shield)                     │
│   - TLS 1.3 termination                                         │
│   - Certificate pinning for mobile apps                         │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: API Gateway                                            │
│   - JWT validation                                              │
│   - Rate limiting per user/IP                                   │
│   - Request validation (schema)                                 │
│   - API key management for partners                             │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Service Mesh                                           │
│   - mTLS between services                                       │
│   - Service-to-service authorization                            │
│   - Secrets rotation (Vault)                                    │
├─────────────────────────────────────────────────────────────────┤
│ Layer 4: Data Security                                          │
│   - Encryption at rest (AES-256)                                │
│   - Field-level encryption for PII                              │
│   - Database audit logging                                      │
│   - Key rotation (90 days)                                      │
├─────────────────────────────────────────────────────────────────┤
│ Layer 5: Content Protection                                     │
│   - Multi-DRM (Widevine, FairPlay, PlayReady)                  │
│   - Hardware security level enforcement                         │
│   - Forensic watermarking                                       │
│   - License expiry and renewal                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Rate Limiting Strategy

```yaml
rateLimits:
  global:
    requestsPerSecond: 200000
    
  perUser:
    authenticated:
      requestsPerMinute: 300
      searchesPerMinute: 30
      
  perEndpoint:
    /auth/login:
      requestsPerMinute: 5
      perEmail: true
      
    /auth/signup:
      requestsPerMinute: 3
      perIP: true
      
    /stream/*/manifest:
      requestsPerMinute: 30
      perUser: true

  response:
    statusCode: 429
    headers:
      Retry-After: {seconds}
      X-RateLimit-Remaining: {count}
```

---

## 15. Cost Optimization

| Component | Cost Driver | Optimization |
|-----------|-------------|--------------|
| CDN Egress | 50 Tbps bandwidth | Open Connect (own CDN) vs third-party saves ~80% |
| Video Storage | 3.6 PB | S3 Intelligent Tiering, delete old codec versions |
| Transcoding | CPU/GPU compute | Spot instances for non-urgent, reserved for baseline |
| Database | IOPS, storage | Reserved capacity, archive old history to S3 |
| Redis | Memory | Eviction policies, compress values, optimize TTLs |
| Compute | EC2/containers | Auto-scaling, right-sizing, spot for stateless |
| Data Transfer | Cross-region | Minimize replication, regional processing |

### Estimated Monthly Cost (100M DAU)

```
CDN / Open Connect:     $15M (egress + hardware + ISP placement)
Compute (Services):     $8M  (EC2/EKS across regions)
Storage (S3):           $3M  (3.6 PB video + backups)
Databases:              $4M  (PostgreSQL, Cassandra, Redis clusters)
Transcoding:            $2M  (encoding pipeline)
Elasticsearch:          $1M  (search cluster)
Kafka:                  $500K (event streaming)
Observability:          $500K (logging, metrics, tracing)
Network/Other:          $1M  (VPC, load balancers, DNS)
──────────────────────────────────────────────────────
Total:                  ~$35M/month
Per User:               ~$0.35/month
```

---

## 16. Summary

This design addresses a Netflix-scale video streaming platform with:

- **50 Tbps** streaming egress via Open Connect CDN
- **115K writes/sec** peak for watch history on Cassandra
- **200K API QPS** through regional API gateways
- **< 100ms p99** catalog latency via CQRS with Redis
- **< 2s** stream start time with manifest caching
- **99.99%** availability through multi-region active-active
- **Multi-DRM** content protection with hardware security levels
- **Real-time personalization** with ML recommendation pipeline
- **Comprehensive observability** with distributed tracing and alerting

Key architectural patterns used:
1. **CQRS** for catalog reads (Redis) vs writes (PostgreSQL)
2. **Event Sourcing** via Kafka for analytics and ML training
3. **Circuit Breaker** for graceful degradation
4. **Multi-region Active-Active** with conflict resolution
5. **Edge Computing** with Open Connect appliances in ISPs
