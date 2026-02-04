# Tap2 Personalize - System Architecture

**Version 1.0 | February 2026**

This document describes the system architecture, technical decisions, and implementation details for Tap2 Personalize.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture Principles](#architecture-principles)
3. [System Components](#system-components)
4. [Data Architecture](#data-architecture)
5. [API Design](#api-design)
6. [Infrastructure](#infrastructure)
7. [Security](#security)
8. [Scalability & Performance](#scalability--performance)
9. [Deployment](#deployment)
10. [Monitoring & Observability](#monitoring--observability)

---

## Overview

Tap2 Personalize is a **serverless, edge-native** payment personalization platform built on Cloudflare's global network. The system delivers personalized payment experiences through ML-powered recommendations while maintaining strict privacy controls.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                 │
│  (Merchant Websites, Mobile Apps, SDKs)                         │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS / API
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CLOUDFLARE WORKERS                           │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐    │
│  │   API       │  │ Personaliz.  │  │  Rate Limiting     │    │
│  │  Gateway    │  │   Engine     │  │  & Auth            │    │
│  └─────────────┘  └──────────────┘  └────────────────────┘    │
└────────────────────────┬────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   D1 Database│  │  KV Cache    │  │  Durable     │
│  (Prefs,     │  │  (Real-time  │  │  Objects     │
│   Events)    │  │   Sessions)  │  │  (Exports)   │
└──────────────┘  └──────────────┘  └──────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CLOUDFLARE QUEUES                            │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │  Feedback Queue │  │  Analytics Queue│                      │
│  └─────────────────┘  └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architecture Principles

### 1. Serverless-First
- **No server management**: Use Cloudflare Workers for compute
- **Auto-scaling**: Automatic horizontal scaling at the edge
- **Pay-per-use**: Cost efficiency with usage-based pricing

### 2. Edge-Native
- **Global distribution**: Compute close to users worldwide
- **Low latency**: Sub-100ms response times for personalization
- **Regional data**: Data residency compliance

### 3. Privacy-First
- **Data minimization**: Collect only necessary data
- **Consent management**: Explicit opt-in for data usage
- **Right to deletion**: GDPR-compliant data deletion
- **Encryption**: AES-256 at rest, TLS 1.3 in transit

### 4. API-First
- **RESTful design**: Standard HTTP methods and status codes
- **Versioned APIs**: `/v1/` namespace with backward compatibility
- **SDK support**: JavaScript, Python, PHP client libraries

### 5. Observability
- **Structured logging**: JSON logs for all services
- **Metrics**: Request latency, error rates, business metrics
- **Tracing**: Distributed tracing for request flows

---

## System Components

### 1. API Gateway (Cloudflare Worker)

**Responsibility**: Request routing, authentication, rate limiting

**Features**:
- API key validation using Cloudflare KV
- Rate limiting per merchant (configurable)
- Request/response logging
- CORS handling
- Request validation

**Technology**: Cloudflare Workers, Hono framework

**Endpoints**:
```
POST /v1/personalize/feedback
GET  /v1/personalize/methods
GET  /v1/personalize/offers
GET  /v1/personalize/preferences
PUT  /v1/personalize/preferences
```

---

### 2. Personalization Engine (Cloudflare Worker)

**Responsibility**: Generate personalized recommendations

**Features**:
- Payment method ordering algorithm
- Context-aware ranking (device, location, time)
- Preference retrieval and scoring
- Fallback to default ordering
- Confidence scoring

**Algorithms** (MVP - Rules-Based):

```typescript
interface MethodScore {
  method_id: string;
  score: number;
  confidence: number;
}

function scoreMethods(
  customerId: string,
  merchantId: string,
  context: Context
): MethodScore[] {
  // 1. Retrieve customer preferences from D1
  const prefs = await getPreferences(customerId, merchantId);

  // 2. Retrieve historical usage from D1
  const history = await getUsageHistory(customerId, merchantId);

  // 3. Apply context boosts
  const contextBoost = calculateContextBoost(context);

  // 4. Calculate scores
  const scores = methods.map(method => ({
    method_id: method.id,
    score: (
      (prefs.default_method === method.id ? 100 : 0) +
      (history.usage_count[method.id] || 0) * 10 +
      contextBoost[method.id] || 0
    ),
    confidence: calculateConfidence(history)
  }));

  // 5. Sort by score descending
  return scores.sort((a, b) => b.score - a.score);
}
```

**Technology**: TypeScript, Cloudflare Workers, D1 database

---

### 3. Preference Store (Cloudflare D1)

**Responsibility**: Persistent storage of customer preferences and usage data

**Schema**:

```sql
-- Customer preferences
CREATE TABLE preferences (
  customer_id TEXT NOT NULL,
  merchant_id TEXT NOT NULL,
  default_method TEXT,
  personalization_enabled BOOLEAN DEFAULT true,
  cross_merchant_consent BOOLEAN DEFAULT false,
  excluded_methods TEXT, -- JSON array
  favorite_methods TEXT, -- JSON array
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_used_at DATETIME,
  PRIMARY KEY (customer_id, merchant_id)
);

-- Payment method usage tracking
CREATE TABLE usage_events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  customer_id TEXT NOT NULL,
  merchant_id TEXT NOT NULL,
  method_id TEXT NOT NULL,
  amount INTEGER, -- cents
  currency TEXT DEFAULT 'USD',
  device_type TEXT,
  location_country TEXT,
  timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_customer_merchant (customer_id, merchant_id),
  INDEX idx_timestamp (timestamp)
);

-- Cross-merchant consent log
CREATE TABLE consent_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  customer_id TEXT NOT NULL,
  consent_type TEXT NOT NULL, -- 'cross_merchant', 'personalization'
  action TEXT NOT NULL, -- 'granted', 'revoked'
  merchant_id TEXT,
  timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_customer (customer_id)
);
```

**Technology**: Cloudflare D1 (SQLite)

---

### 4. KV Cache Layer (Cloudflare KV)

**Responsibility**: Low-latency access to frequently accessed data

**Use Cases**:
- API key validation (TTL: 3600s)
- Real-time preference cache (TTL: 300s)
- Merchant configuration (TTL: 600s)
- Session data (TTL: 1800s)

**Key Patterns**:
```
api_key:{key} -> {merchant_id, tier, rate_limit}
pref:{customer_id}:{merchant_id} -> {preferences JSON}
merchant:{merchant_id} -> {config JSON}
session:{session_id} -> {session_data JSON}
```

**Technology**: Cloudflare KV (global key-value store)

---

### 5. Feedback Processing Queue

**Responsibility**: Async processing of interaction feedback

**Flow**:
1. Client sends feedback to API
2. API Gateway validates and pushes to Queue
3. Consumer Worker processes events
4. Batch insert to D1 (100 events per batch)
5. Update preference cache

**Technology**: Cloudflare Queues

---

### 6. Analytics Pipeline

**Responsibility**: Calculate performance metrics and merchant insights

**Jobs**:
- Hourly aggregation of usage events
- Daily calculation of conversion metrics
- Weekly reporting generation

**Metrics Calculated**:
- Conversion rate (personalized vs. baseline)
- Average order value
- Payment method distribution
- Offer redemption rate (Enhanced phase)
- Checkout time

**Storage**: Aggregated data in D1, raw events in Durable Objects for longer retention

---

## Data Architecture

### Data Flow

```
┌────────────────────────────────────────────────────────────────┐
│ 1. MERCHANT REQUEST                                             │
│    GET /v1/personalize/methods?customer_id=123&merchant_id=456 │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│ 2. AUTH & RATE LIMIT                                            │
│    - Validate API key from KV                                   │
│    - Check rate limit (per merchant)                            │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│ 3. RETRIEVE PREFERENCES                                         │
│    - Check KV cache for preferences                             │
│    - If miss, query D1 database                                │
│    - Cache result in KV                                         │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│ 4. CONTEXT AWARENESS                                            │
│    - Extract device type from User-Agent                        │
│    - Get country from CF-IPCountry header                       │
│    - Apply context-based boosting                               │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│ 5. PERSONALIZATION ALGORITHM                                    │
│    - Score methods based on preferences                         │
│    - Apply context weights                                      │
│    - Filter unavailable methods                                │
│    - Sort by score descending                                   │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────┐
│ 6. RETURN RESPONSE                                              │
│    {                                                             │
│      "methods": [                                                │
│        {"method_id": "apple_pay", "confidence": 0.92},         │
│        {"method_id": "visa", "confidence": 0.85}               │
│      ],                                                          │
│      "personalized": true                                       │
│    }                                                             │
└────────────────────────────────────────────────────────────────┘
```

### Data Models

#### Preference Object
```typescript
interface Preference {
  customer_id: string;
  merchant_id: string;
  default_method?: string;
  favorite_methods: string[];
  excluded_methods: string[];
  personalization_enabled: boolean;
  cross_merchant_consent: boolean;
  created_at: string;
  updated_at: string;
  last_used_at: string;
}
```

#### Usage Event Object
```typescript
interface UsageEvent {
  customer_id: string;
  merchant_id: string;
  method_id: string;
  amount?: number;
  currency?: string;
  device_type?: 'mobile' | 'desktop' | 'tablet';
  location_country?: string;
  timestamp: string;
}
```

#### Context Object
```typescript
interface Context {
  device_type: 'mobile' | 'desktop' | 'tablet';
  country: string;
  time_of_day: 'morning' | 'afternoon' | 'evening' | 'night';
  day_of_week: number;
}
```

#### Personalization Response
```typescript
interface PersonalizationResponse {
  methods: Array<{
    method_id: string;
    confidence: number; // 0-1
    score?: number;
  }>;
  personalized: boolean;
  fallback_reason?: string;
}
```

---

## API Design

### Base URL
```
Production: https://api.tap2-personalize.com
Staging: https://api-staging.tap2-personalize.com
```

### Authentication
- **API Key**: Passed via `X-API-Key` header or `?api_key=` query param
- **OAuth 2.0**: For dashboard access (merchant accounts)

### Common Headers
```
X-API-Key: {api_key}
Content-Type: application/json
Accept: application/json
```

### Endpoints

#### 1. Get Personalized Methods
```http
GET /v1/personalize/methods?customer_id={id}&merchant_id={id}

Response:
{
  "methods": [
    {"method_id": "apple_pay", "confidence": 0.92},
    {"method_id": "visa", "confidence": 0.85},
    {"method_id": "mastercard", "confidence": 0.78}
  ],
  "personalized": true
}
```

#### 2. Submit Feedback
```http
POST /v1/personalize/feedback
Content-Type: application/json

{
  "customer_id": "123",
  "merchant_id": "456",
  "method_id": "visa",
  "amount": 9900,
  "currency": "USD",
  "device_type": "mobile",
  "location_country": "US"
}

Response: 204 No Content
```

#### 3. Get Preferences
```http
GET /v1/personalize/preferences?customer_id={id}&merchant_id={id}

Response:
{
  "customer_id": "123",
  "merchant_id": "456",
  "default_method": "visa",
  "favorite_methods": ["visa", "apple_pay"],
  "excluded_methods": ["paypal"],
  "personalization_enabled": true,
  "cross_merchant_consent": false,
  "created_at": "2026-02-01T10:00:00Z",
  "updated_at": "2026-02-03T15:30:00Z"
}
```

#### 4. Update Preferences
```http
PUT /v1/personalize/preferences
Content-Type: application/json

{
  "customer_id": "123",
  "merchant_id": "456",
  "default_method": "apple_pay",
  "favorite_methods": ["apple_pay", "visa"],
  "excluded_methods": [],
  "personalization_enabled": true,
  "cross_merchant_consent": true
}

Response:
{
  "updated": true,
  "preferences": { /* ... */ }
}
```

### Error Responses

All errors follow this format:
```json
{
  "error": {
    "code": "INVALID_API_KEY",
    "message": "The provided API key is invalid or expired",
    "request_id": "req_abc123",
    "documentation_url": "https://docs.tap2.com/errors"
  }
}
```

**HTTP Status Codes**:
- `200` - Success
- `204` - Success (no content)
- `400` - Bad Request
- `401` - Unauthorized (invalid API key)
- `429` - Rate limit exceeded
- `500` - Internal server error

---

## Infrastructure

### Cloudflare Stack

| Service | Purpose | Configuration |
|---------|---------|---------------|
| **Workers** | API compute | 10ms CPU time limit, 128MB memory |
| **D1** | Primary database | 5GB storage, 5,000 reads/sec |
| **KV** | Caching layer | 1GB storage, 100M reads/day |
| **Queues** | Async processing | 100 messages/second |
| **R2** | File storage (future) | Exports, reports |

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GIT REPOSITORY                            │
│  (main, develop, feature branches)                           │
└──────────────────────────┬──────────────────────────────────┘
                           │ git push
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    GITHUB ACTIONS                            │
│  - Run tests (Vitest)                                        │
│  - Build TypeScript                                          │
│  - Run linter (ESLint)                                       │
│  - Create deployment artifact                                │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  WRANGLER CLI                                │
│  - Deploy Workers to Cloudflare                              │
│  - Run D1 migrations                                         │
│  - Update KV namespace                                       │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Production  │  │   Staging    │  │  Dev         │
│  .tap2.com   │  │ -staging.com │  │  .dev        │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Environment Configuration

**Production** (`wrangler.production.toml`):
```toml
name = "tap2-personalize-api"
vars = { ENVIRONMENT = "production" }
routes = [
  { pattern = "https://api.tap2-personalize.com/*", zone_name = "tap2-personalize.com" }
]

[[d1_databases]]
binding = "DB"
database_name = "tap2-personalize-prod"
database_id = "prod-database-id"

[[kv_namespaces]]
binding = "CACHE"
id = "prod-cache-id"

[[queues.producers]]
binding = "FEEDBACK_QUEUE"
queue = "feedback-events-prod"
```

**Staging** (`wrangler.staging.toml`):
```toml
name = "tap2-personalize-api-staging"
vars = { ENVIRONMENT = "staging" }
routes = [
  { pattern = "https://api-staging.tap2-personalize.com/*", zone_name = "tap2-personalize.com" }
]
# ... similar structure with staging IDs
```

---

## Security

### Authentication & Authorization

1. **API Key Authentication**:
   - Merchant-specific API keys
   - Stored in KV with metadata (tier, rate limit, merchant_id)
   - Rotatable via dashboard
   - Key prefix: `tp2_` for identification

2. **OAuth 2.0** (Dashboard):
   - Authorization code flow for merchant login
   - PKCE extension for security
   - Scope-based permissions

### Data Encryption

| Layer | Encryption Method |
|-------|-------------------|
| At Rest (D1) | AES-256 (managed by Cloudflare) |
| In Transit | TLS 1.3 |
| API Keys | SHA-256 hashed before storage |
| PII | Encrypted with application-level key |

### Privacy Controls

1. **Consent Management**:
   - Explicit opt-in for cross-merchant personalization
   - Granular controls (per merchant, global)
   - Audit log for consent changes

2. **Data Retention**:
   - Usage events: 90 days
   - Preferences: 18 months after last use
   - Consent logs: 3 years (GDPR requirement)

3. **Right to Deletion**:
   - API endpoint for customer data deletion
   - Cascading delete across all tables
   - 30-day grace period for recovery

### Rate Limiting

| Tier | Limit | Burst |
|------|-------|-------|
| Starter | 1,000 req/hour | 100 |
| Growth | 5,000 req/hour | 500 |
| Scale | 10,000 req/hour | 1,000 |

Implementation: Cloudflare Workers with KV counter

---

## Scalability & Performance

### Performance Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| API p50 Latency | < 50ms | Cloudflare Analytics |
| API p95 Latency | < 100ms | Cloudflare Analytics |
| API p99 Latency | < 200ms | Cloudflare Analytics |
| Cache Hit Rate | > 85% | KV metrics |
| DB Query Time | < 50ms | D1 metrics |

### Scaling Strategy

1. **Horizontal Scaling**:
   - Cloudflare Workers auto-scale globally
   - No capacity planning needed
   - Edge locations: 300+ cities worldwide

2. **Database Scaling** (Future):
   - D1 read replicas (when available)
   - Partitioning by merchant_id for large merchants
   - Archive old data to object storage

3. **Caching Strategy**:
   - L1: Edge cache (static responses, TTL: 60s)
   - L2: KV cache (preferences, TTL: 300s)
   - L3: D1 database (persistent storage)

### Optimization Techniques

1. **Batch Processing**:
   - Aggregate feedback events in batches of 100
   - Bulk insert to D1
   - Reduce round trips

2. **Query Optimization**:
   - Indexed columns on frequently queried fields
   - Use prepared statements
   - Limit result sets with pagination

3. **CDN Caching**:
   - Cache GET requests with low variability
   - Cache key includes customer_id, merchant_id
   - Short TTL for personalized data

---

## Deployment

### CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm test
      - run: npm run lint

  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: cloudflare/wrangler-action@v2
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          command: deploy --env staging

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: cloudflare/wrangler-action@v2
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          command: deploy --env production
```

### Database Migrations

```sql
-- migrations/001_initial_schema.sql
-- Run: wrangler d1 execute tap2-personalize-prod --file=migrations/001_initial_schema.sql

CREATE TABLE preferences (
  -- ... (see schema above)
);

CREATE TABLE usage_events (
  -- ... (see schema above)
);
```

### Feature Flags

Use environment variables and KV for feature toggling:

```typescript
const FEATURE_FLAGS = {
  TARGETED_OFFERS: env.ENABLE_TARGETED_OFFERS === 'true',
  AB_TESTING: env.ENABLE_AB_TESTING === 'true',
  CROSS_MERCHANT: await KV.get('feature:cross_merchant') === 'true'
};
```

---

## Monitoring & Observability

### Logging

**Structured Logging Format**:
```json
{
  "timestamp": "2026-02-04T10:30:00Z",
  "level": "info",
  "request_id": "req_abc123",
  "merchant_id": "456",
  "customer_id": "123",
  "endpoint": "/v1/personalize/methods",
  "method": "GET",
  "status_code": 200,
  "latency_ms": 45,
  "user_agent": "Mozilla/5.0...",
  "cf_country": "US"
}
```

**Log Destinations**:
- Cloudflare Workers Analytics (built-in)
- External: Datadog, New Relic (via Workers Logpush)

### Metrics

**Business Metrics**:
- Personalization rate (% of requests with preferences)
- Method selection distribution
- Conversion lift (personalized vs. baseline)
- Cross-merchant opt-in rate

**Technical Metrics**:
- Request latency (p50, p95, p99)
- Error rate (%)
- Cache hit rate
- Rate limit rejections

### Monitoring Stack

| Tool | Purpose |
|------|---------|
| **Cloudflare Analytics** | Request metrics, latency, errors |
| **Sentry** | Error tracking, alerting |
| **Datadog** (future) | APM, dashboards |
| **Grafana** (self-hosted) | Custom dashboards |

### Alerting

**Alert Rules**:
- Error rate > 1% for 5 minutes
- p95 latency > 200ms for 5 minutes
- Cache hit rate < 70% for 15 minutes
- Database query failures > 0.1%

**On-Call**:
- PagerDuty integration
- Severity levels: P1 (critical), P2 (high), P3 (medium)

---

## Technology Stack Summary

| Component | Technology | Version |
|-----------|------------|---------|
| **Runtime** | Cloudflare Workers | - |
| **Language** | TypeScript | 5.x |
| **Framework** | Hono | 4.x |
| **Database** | Cloudflare D1 | - |
| **Cache** | Cloudflare KV | - |
| **Queues** | Cloudflare Queues | - |
| **Testing** | Vitest | 2.x |
| **Build** | esbuild | 0.x |
| **Package Manager** | npm | 10.x |
| **CI/CD** | GitHub Actions | - |
| **Deployment** | Wrangler CLI | 3.x |

---

## Future Enhancements

### Enhanced Phase (Q1 2027)
- **ML Model Integration**: TensorFlow Lite for offer targeting
- **A/B Testing**: Experiment assignment framework
- **Advanced Analytics**: Real-time dashboards

### Advanced Phase (Q2 2027)
- **Cross-Merchant**: Aggregated preference analysis
- **Deep Learning**: Neural network models for personalization
- **Feature Store**: Real-time feature computation

### Enterprise Phase (Q3 2027)
- **Custom Models**: Merchant-specific model training
- **Webhooks**: Real-time event notifications
- **SLA Monitoring**: Uptime tracking and reporting

---

*End of Document*
