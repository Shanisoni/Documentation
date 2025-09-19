# GEO Score Integration System: Complete Technical Guide

## Executive Summary

This document provides a **100x detailed** technical and theoretical explanation of how to integrate three separate GEO scoring backends (Crawler, Prompt Engine, and Server Logs) into a unified scoring system with a single React dashboard. The system uses a **coordinator pattern** to aggregate individual scores using dynamic weighting based on service availability.

---

## Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Data Flow and Integration Patterns](#2-data-flow-and-integration-patterns)
3. [Scoring Algorithm Deep Dive](#3-scoring-algorithm-deep-dive)
4. [API Design and Implementation](#4-api-design-and-implementation)
5. [Database Schema and Data Management](#5-database-schema-and-data-management)
6. [Frontend Integration and UI Components](#6-frontend-integration-and-ui-components)
7. [Error Handling and Resilience](#7-error-handling-and-resilience)
8. [Performance Optimization and Caching](#8-performance-optimization-and-caching)
9. [Security and Compliance](#9-security-and-compliance)
10. [Deployment and Infrastructure](#10-deployment-and-infrastructure)

---

## 1. System Architecture Overview

### High-Level Architecture Diagram

```mermaid
graph TB
    subgraph "Client Layer"
        A[React Dashboard] --> B[GEO Score Display]
        A --> C[Score Breakdown]
        A --> D[Service Status]
    end
    
    subgraph "API Gateway Layer"
        E[Coordinator API] --> F[Authentication]
        E --> G[Rate Limiting]
        E --> H[Request Routing]
    end
    
    subgraph "Service Layer"
        I[Crawler Service] --> J[Crawler GEO API]
        K[Prompt Engine] --> L[Prompt GEO API]
        M[Server Logs] --> N[Logs GEO API]
    end
    
    subgraph "Data Layer"
        O[(PostgreSQL)] --> P[Tenant Features]
        O --> Q[Score Cache]
        R[(Redis Cache)] --> S[Session Data]
        R --> T[API Response Cache]
    end
    
    subgraph "External Services"
        U[Upstash Redis]
        V[Supabase Database]
        W[Vercel Hosting]
    end
    
    A --> E
    E --> I
    E --> K
    E --> M
    E --> O
    E --> R
    
    style A fill:#e1f5fe
    style E fill:#f3e5f5
    style O fill:#e8f5e8
    style R fill:#fff3e0
```

### Component Responsibilities

#### React Dashboard (Frontend)
- **Primary Function:** Display unified GEO scores and breakdowns
- **Key Features:** Real-time updates, service status indicators, score history
- **Technology Stack:** React 18+, TypeScript, Tailwind CSS, React Query

#### Coordinator API (Backend)
- **Primary Function:** Aggregate scores from multiple services
- **Key Features:** Dynamic weighting, error handling, caching, rate limiting
- **Technology Stack:** Node.js, Express.js, TypeScript, BullMQ

#### Individual Service APIs
- **Crawler Service:** Provides content quality and crawl coverage scores
- **Prompt Engine:** Provides AI response quality and relevance scores
- **Server Logs:** Provides AI crawler presence and activity scores

---

## 2. Data Flow and Integration Patterns

### Complete Data Flow Sequence

```mermaid
sequenceDiagram
    participant U as User
    participant R as React Dashboard
    participant C as Coordinator API
    participant CR as Crawler API
    participant P as Prompt API
    participant L as Logs API
    participant DB as Database
    participant CACHE as Redis Cache

    U->>R: Opens dashboard
    R->>C: GET /api/sites/:siteId/geo
    C->>DB: Check tenant features
    DB-->>C: Return logs_enabled status
    
    par Parallel API Calls
        C->>CR: GET /geo/:siteId
        CR-->>C: {score: 72, computed_at: "2024-01-15T10:00:00Z"}
    and
        C->>P: GET /geo/:siteId
        P-->>C: {score: 60, computed_at: "2024-01-15T10:00:00Z"}
    and
        alt Logs Enabled
            C->>L: GET /geo/:siteId
            L-->>C: {score: 80, computed_at: "2024-01-15T10:00:00Z"}
        else Logs Disabled
            C->>C: Skip logs API call
        end
    end
    
    C->>C: Calculate weighted average
    C->>CACHE: Store computed score
    C-->>R: Return unified GEO score
    R->>U: Display score and breakdown
```

### Integration Patterns

#### 1. Coordinator Pattern
**Purpose:** Centralized orchestration of multiple service calls
**Benefits:** 
- Single point of failure management
- Consistent error handling
- Unified response format
- Centralized caching

#### 2. Circuit Breaker Pattern
**Purpose:** Prevent cascade failures when individual services are down
**Implementation:**
```typescript
class CircuitBreaker {
  private failureCount = 0;
  private lastFailureTime = 0;
  private readonly threshold = 5;
  private readonly timeout = 60000; // 1 minute

  async call<T>(fn: () => Promise<T>): Promise<T | null> {
    if (this.isOpen()) {
      return null; // Circuit is open, don't call
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
}
```

#### 3. Retry Pattern with Exponential Backoff
**Purpose:** Handle transient failures gracefully
**Implementation:**
```typescript
async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  baseDelay = 1000
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;
      
      const delay = baseDelay * Math.pow(2, attempt);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

---

## 3. Scoring Algorithm Deep Dive

### Mathematical Foundation

The unified GEO score is calculated using a **dynamic weighted average** approach:

```
GEO_Score = Σ(Score_i × Weight_i) / Σ(Weight_i)
```

Where:
- `Score_i` = Individual service score (0-100)
- `Weight_i` = Dynamic weight based on service availability
- `n` = Number of available services

### Weight Calculation Logic

```mermaid
flowchart TD
    A[Start Weight Calculation] --> B[Count Available Services]
    B --> C{How Many Services?}
    
    C -->|3 Services| D[Weight = 33.33% each]
    C -->|2 Services| E[Weight = 50% each]
    C -->|1 Service| F[Weight = 100%]
    C -->|0 Services| G[Return Error]
    
    D --> H[Apply Weights]
    E --> H
    F --> H
    G --> I[End with Error]
    H --> J[Calculate Final Score]
    J --> K[End]
    
    style A fill:#e1f5fe
    style K fill:#e8f5e8
    style G fill:#ffebee
```

### Detailed Scoring Examples

#### Example 1: All Three Services Available
```typescript
const scores = {
  crawler: { score: 72, computed_at: "2024-01-15T10:00:00Z" },
  prompt: { score: 60, computed_at: "2024-01-15T10:00:00Z" },
  logs: { score: 80, computed_at: "2024-01-15T10:00:00Z" }
};

// Weight calculation
const availableServices = 3;
const weight = 100 / availableServices; // 33.33%

// Score calculation
const geoScore = (72 * 0.3333) + (60 * 0.3333) + (80 * 0.3333);
// Result: 70.67
```

#### Example 2: Two Services Available (Logs Disabled)
```typescript
const scores = {
  crawler: { score: 72, computed_at: "2024-01-15T10:00:00Z" },
  prompt: { score: 60, computed_at: "2024-01-15T10:00:00Z" },
  logs: null // Disabled
};

// Weight calculation
const availableServices = 2;
const weight = 100 / availableServices; // 50%

// Score calculation
const geoScore = (72 * 0.5) + (60 * 0.5);
// Result: 66
```

#### Example 3: Single Service Available (Others Failed)
```typescript
const scores = {
  crawler: { score: 72, computed_at: "2024-01-15T10:00:00Z" },
  prompt: null, // API failed
  logs: null    // API failed
};

// Weight calculation
const availableServices = 1;
const weight = 100 / availableServices; // 100%

// Score calculation
const geoScore = 72 * 1.0;
// Result: 72
```

### Score Quality Assessment

```mermaid
graph LR
    A[Raw Scores] --> B[Data Validation]
    B --> C[Staleness Check]
    C --> D[Quality Filtering]
    D --> E[Weight Application]
    E --> F[Final Calculation]
    
    B --> B1[Score Range: 0-100]
    B --> B2[Valid Timestamp]
    B --> B3[Non-null Values]
    
    C --> C1[Age < 24 hours]
    C --> C2[Mark as Stale if > 24h]
    
    D --> D1[Remove Outliers]
    D --> D2[Apply Confidence Scores]
```

---

## 4. API Design and Implementation

### Coordinator API Endpoints

#### Primary Endpoint: Get Unified GEO Score
```typescript
GET /api/sites/:siteId/geo
```

**Request Parameters:**
- `siteId` (path): Unique identifier for the site
- `includeBreakdown` (query, optional): Include detailed breakdown (default: true)
- `forceRefresh` (query, optional): Bypass cache and force recalculation

**Response Format:**
```typescript
interface GeoScoreResponse {
  siteId: string;
  geo: number; // 0-100
  breakdown: {
    crawler?: {
      score: number;
      weight: number;
      computed_at: string;
      status: 'active' | 'stale' | 'error';
    };
    prompt?: {
      score: number;
      weight: number;
      computed_at: string;
      status: 'active' | 'stale' | 'error';
    };
    logs?: {
      score: number;
      weight: number;
      computed_at: string;
      status: 'active' | 'stale' | 'error' | 'disabled';
    };
  };
  computed_at: string;
  cache_ttl: number;
  services_available: number;
  services_total: number;
}
```

### Implementation Architecture

```mermaid
classDiagram
    class CoordinatorAPI {
        +getGeoScore(siteId: string): Promise<GeoScoreResponse>
        +refreshScore(siteId: string): Promise<GeoScoreResponse>
        +getScoreHistory(siteId: string): Promise<ScoreHistory[]>
    }
    
    class ServiceClient {
        +crawler: CrawlerClient
        +prompt: PromptClient
        +logs: LogsClient
        +callAllServices(siteId: string): Promise<ServiceScores>
    }
    
    class ScoreCalculator {
        +calculateWeightedScore(scores: ServiceScores): number
        +validateScore(score: ServiceScore): boolean
        +checkStaleness(computed_at: string): boolean
    }
    
    class CacheManager {
        +get(key: string): Promise<any>
        +set(key: string, value: any, ttl: number): Promise<void>
        +invalidate(pattern: string): Promise<void>
    }
    
    class DatabaseManager {
        +getTenantFeatures(tenantId: string): Promise<TenantFeatures>
        +saveScore(siteId: string, score: GeoScoreResponse): Promise<void>
        +getScoreHistory(siteId: string): Promise<ScoreHistory[]>
    }
    
    CoordinatorAPI --> ServiceClient
    CoordinatorAPI --> ScoreCalculator
    CoordinatorAPI --> CacheManager
    CoordinatorAPI --> DatabaseManager
```

### Service Client Implementation

```typescript
class ServiceClient {
  private readonly crawlerUrl: string;
  private readonly promptUrl: string;
  private readonly logsUrl: string;
  private readonly circuitBreakers: Map<string, CircuitBreaker>;

  constructor(config: ServiceConfig) {
    this.crawlerUrl = config.crawlerUrl;
    this.promptUrl = config.promptUrl;
    this.logsUrl = config.logsUrl;
    this.circuitBreakers = new Map();
  }

  async callAllServices(siteId: string, tenantFeatures: TenantFeatures): Promise<ServiceScores> {
    const promises = [
      this.callCrawlerService(siteId),
      this.callPromptService(siteId),
      tenantFeatures.logs_enabled ? this.callLogsService(siteId) : Promise.resolve(null)
    ];

    const [crawler, prompt, logs] = await Promise.allSettled(promises);

    return {
      crawler: this.extractResult(crawler),
      prompt: this.extractResult(prompt),
      logs: this.extractResult(logs)
    };
  }

  private async callCrawlerService(siteId: string): Promise<ServiceScore | null> {
    const breaker = this.getCircuitBreaker('crawler');
    return breaker.call(async () => {
      const response = await fetch(`${this.crawlerUrl}/geo/${siteId}`);
      if (!response.ok) throw new Error(`Crawler API error: ${response.status}`);
      return response.json();
    });
  }

  private extractResult(promiseResult: PromiseSettledResult<any>): ServiceScore | null {
    if (promiseResult.status === 'fulfilled') {
      return promiseResult.value;
    }
    return null;
  }
}
```

---

## 5. Database Schema and Data Management

### Core Tables

#### Tenants Table
```sql
CREATE TABLE tenants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  domain VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### Tenant Features Table
```sql
CREATE TABLE tenant_features (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
  logs_enabled BOOLEAN DEFAULT FALSE,
  crawler_enabled BOOLEAN DEFAULT TRUE,
  prompt_enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(tenant_id)
);
```

#### GEO Scores Cache Table
```sql
CREATE TABLE geo_scores_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id VARCHAR(255) NOT NULL,
  tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
  geo_score DECIMAL(5,2) NOT NULL,
  breakdown JSONB NOT NULL,
  services_available INTEGER NOT NULL,
  services_total INTEGER NOT NULL,
  computed_at TIMESTAMP WITH TIME ZONE NOT NULL,
  cache_expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  UNIQUE(site_id, tenant_id)
);
```

#### Score History Table
```sql
CREATE TABLE geo_score_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  site_id VARCHAR(255) NOT NULL,
  tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
  geo_score DECIMAL(5,2) NOT NULL,
  breakdown JSONB NOT NULL,
  services_available INTEGER NOT NULL,
  computed_at TIMESTAMP WITH TIME ZONE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### Database Relationships

```mermaid
erDiagram
    TENANTS ||--o{ TENANT_FEATURES : "has"
    TENANTS ||--o{ GEO_SCORES_CACHE : "owns"
    TENANTS ||--o{ GEO_SCORE_HISTORY : "tracks"
    
    TENANTS {
        uuid id PK
        varchar name
        varchar domain
        timestamp created_at
        timestamp updated_at
    }
    
    TENANT_FEATURES {
        uuid id PK
        uuid tenant_id FK
        boolean logs_enabled
        boolean crawler_enabled
        boolean prompt_enabled
        timestamp created_at
        timestamp updated_at
    }
    
    GEO_SCORES_CACHE {
        uuid id PK
        varchar site_id
        uuid tenant_id FK
        decimal geo_score
        jsonb breakdown
        integer services_available
        integer services_total
        timestamp computed_at
        timestamp cache_expires_at
        timestamp created_at
    }
    
    GEO_SCORE_HISTORY {
        uuid id PK
        varchar site_id
        uuid tenant_id FK
        decimal geo_score
        jsonb breakdown
        integer services_available
        timestamp computed_at
        timestamp created_at
    }
```

### Data Management Strategies

#### Caching Strategy
```typescript
interface CacheStrategy {
  ttl: number; // Time to live in seconds
  refreshThreshold: number; // Refresh when this close to expiry
  maxRetries: number; // Max retry attempts
}

const CACHE_STRATEGIES: Record<string, CacheStrategy> = {
  geo_score: {
    ttl: 300, // 5 minutes
    refreshThreshold: 60, // Refresh when 1 minute left
    maxRetries: 3
  },
  tenant_features: {
    ttl: 3600, // 1 hour
    refreshThreshold: 300, // Refresh when 5 minutes left
    maxRetries: 2
  }
};
```

#### Data Retention Policy
```typescript
class DataRetentionManager {
  async cleanupOldData(): Promise<void> {
    // Keep score history for 1 year
    await this.db.query(`
      DELETE FROM geo_score_history 
      WHERE created_at < NOW() - INTERVAL '1 year'
    `);
    
    // Keep cache for 7 days max
    await this.db.query(`
      DELETE FROM geo_scores_cache 
      WHERE cache_expires_at < NOW()
    `);
  }
}
```

---

## 6. Frontend Integration and UI Components

### React Component Architecture

```mermaid
graph TD
    A[GEO Dashboard] --> B[Score Display]
    A --> C[Service Status]
    A --> D[Score History]
    A --> E[Settings Panel]
    
    B --> B1[Main Score Card]
    B --> B2[Score Breakdown]
    B --> B3[Trend Indicators]
    
    C --> C1[Service Health]
    C --> C2[Last Updated]
    C --> C3[Error States]
    
    D --> D1[Historical Chart]
    D --> D2[Score Timeline]
    D --> D3[Export Options]
    
    E --> E1[Feature Toggles]
    E --> E2[Refresh Settings]
    E --> E3[Cache Management]
```

### Core React Components

#### Main GEO Score Component
```typescript
interface GeoScoreProps {
  siteId: string;
  autoRefresh?: boolean;
  refreshInterval?: number;
}

const GeoScore: React.FC<GeoScoreProps> = ({ 
  siteId, 
  autoRefresh = true, 
  refreshInterval = 30000 
}) => {
  const { data, error, isLoading, refetch } = useGeoScore(siteId, {
    refetchInterval: autoRefresh ? refreshInterval : false
  });

  if (isLoading) return <ScoreSkeleton />;
  if (error) return <ScoreError error={error} onRetry={refetch} />;
  if (!data) return <NoDataMessage />;

  return (
    <div className="geo-score-container">
      <MainScoreCard score={data.geo} />
      <ScoreBreakdown breakdown={data.breakdown} />
      <ServiceStatus services={data.breakdown} />
    </div>
  );
};
```

#### Score Breakdown Component
```typescript
const ScoreBreakdown: React.FC<{ breakdown: ScoreBreakdown }> = ({ breakdown }) => {
  const services = Object.entries(breakdown).map(([key, service]) => ({
    name: key,
    ...service
  }));

  return (
    <div className="score-breakdown">
      <h3>Score Breakdown</h3>
      <div className="breakdown-grid">
        {services.map(service => (
          <ServiceCard key={service.name} service={service} />
        ))}
      </div>
    </div>
  );
};

const ServiceCard: React.FC<{ service: ServiceInfo }> = ({ service }) => {
  const getStatusIcon = (status: string) => {
    switch (status) {
      case 'active': return '✅';
      case 'stale': return '⚠️';
      case 'error': return '❌';
      case 'disabled': return '🔒';
      default: return '❓';
    }
  };

  return (
    <div className={`service-card ${service.status}`}>
      <div className="service-header">
        <span className="service-name">{service.name}</span>
        <span className="status-icon">{getStatusIcon(service.status)}</span>
      </div>
      <div className="service-score">
        {service.score !== null ? `${service.score}` : '—'}
      </div>
      <div className="service-weight">
        Weight: {service.weight}%
      </div>
      <div className="service-timestamp">
        {service.computed_at ? formatDate(service.computed_at) : 'Never'}
      </div>
    </div>
  );
};
```

### Custom Hooks

#### useGeoScore Hook
```typescript
interface UseGeoScoreOptions {
  refetchInterval?: number | false;
  staleTime?: number;
  retryOnError?: boolean;
}

export const useGeoScore = (
  siteId: string, 
  options: UseGeoScoreOptions = {}
) => {
  const {
    refetchInterval = 30000,
    staleTime = 5 * 60 * 1000, // 5 minutes
    retryOnError = true
  } = options;

  return useQuery({
    queryKey: ['geoScore', siteId],
    queryFn: () => fetchGeoScore(siteId),
    refetchInterval,
    staleTime,
    retry: retryOnError ? 3 : false,
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000)
  });
};
```

#### useScoreHistory Hook
```typescript
export const useScoreHistory = (siteId: string, days: number = 30) => {
  return useQuery({
    queryKey: ['scoreHistory', siteId, days],
    queryFn: () => fetchScoreHistory(siteId, days),
    staleTime: 10 * 60 * 1000, // 10 minutes
  });
};
```

### UI State Management

```mermaid
stateDiagram-v2
    [*] --> Loading
    Loading --> Success: Data loaded
    Loading --> Error: API failed
    Success --> Refreshing: User refresh
    Success --> Success: Auto refresh
    Error --> Loading: Retry
    Refreshing --> Success: Refresh complete
    Refreshing --> Error: Refresh failed
    Success --> [*]: Component unmount
    Error --> [*]: Component unmount
```

---

## 7. Error Handling and Resilience

### Error Classification System

```typescript
enum ErrorType {
  NETWORK_ERROR = 'NETWORK_ERROR',
  API_ERROR = 'API_ERROR',
  TIMEOUT_ERROR = 'TIMEOUT_ERROR',
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  AUTHENTICATION_ERROR = 'AUTHENTICATION_ERROR',
  RATE_LIMIT_ERROR = 'RATE_LIMIT_ERROR',
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE'
}

interface ErrorContext {
  type: ErrorType;
  service: string;
  siteId: string;
  timestamp: string;
  retryCount: number;
  originalError?: Error;
}
```

### Circuit Breaker Implementation

```typescript
class CircuitBreaker {
  private failureCount = 0;
  private lastFailureTime = 0;
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED';
  
  constructor(
    private threshold = 5,
    private timeout = 60000,
    private resetTimeout = 30000
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.resetTimeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}
```

### Retry Strategy with Exponential Backoff

```typescript
class RetryManager {
  async executeWithRetry<T>(
    fn: () => Promise<T>,
    maxRetries = 3,
    baseDelay = 1000,
    maxDelay = 30000
  ): Promise<T> {
    let lastError: Error;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error as Error;
        
        if (attempt === maxRetries) {
          throw lastError;
        }

        const delay = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);
        const jitter = Math.random() * 0.1 * delay; // Add jitter
        
        await new Promise(resolve => 
          setTimeout(resolve, delay + jitter)
        );
      }
    }

    throw lastError!;
  }
}
```

### Error Recovery Strategies

```mermaid
flowchart TD
    A[Error Occurs] --> B{Error Type?}
    
    B -->|Network| C[Retry with Backoff]
    B -->|Timeout| D[Increase Timeout & Retry]
    B -->|Rate Limit| E[Wait & Retry]
    B -->|Service Down| F[Use Circuit Breaker]
    B -->|Auth Error| G[Refresh Token & Retry]
    B -->|Validation| H[Log & Return Error]
    
    C --> I{Retry Success?}
    D --> I
    E --> I
    F --> J[Return Cached Data]
    G --> I
    H --> K[Return Error to User]
    
    I -->|Yes| L[Return Success]
    I -->|No| M{Max Retries?}
    
    M -->|No| C
    M -->|Yes| N[Return Partial Data]
    
    J --> O[End]
    K --> O
    L --> O
    N --> O
```

---

## 8. Performance Optimization and Caching

### Multi-Layer Caching Strategy

```mermaid
graph TB
    A[Client Request] --> B[CDN Cache]
    B --> C{CDN Hit?}
    C -->|Yes| D[Return Cached Response]
    C -->|No| E[API Gateway]
    
    E --> F[Redis Cache]
    F --> G{Redis Hit?}
    G -->|Yes| H[Return Cached Data]
    G -->|No| I[Service Layer]
    
    I --> J[Database Cache]
    J --> K{DB Cache Hit?}
    K -->|Yes| L[Return Cached Score]
    K -->|No| M[Calculate New Score]
    
    M --> N[Store in All Caches]
    N --> O[Return Response]
    
    style B fill:#e1f5fe
    style F fill:#f3e5f5
    style J fill:#e8f5e8
```

### Cache Implementation

```typescript
class CacheManager {
  constructor(
    private redis: Redis,
    private db: Database,
    private cdn: CDNClient
  ) {}

  async get<T>(key: string, ttl: number = 300): Promise<T | null> {
    // Try CDN first
    const cdnData = await this.cdn.get(key);
    if (cdnData) return cdnData;

    // Try Redis
    const redisData = await this.redis.get(key);
    if (redisData) {
      // Update CDN
      await this.cdn.set(key, redisData, ttl);
      return JSON.parse(redisData);
    }

    // Try Database
    const dbData = await this.db.getCachedScore(key);
    if (dbData) {
      // Update Redis and CDN
      await this.redis.setex(key, ttl, JSON.stringify(dbData));
      await this.cdn.set(key, dbData, ttl);
      return dbData;
    }

    return null;
  }

  async set<T>(key: string, data: T, ttl: number = 300): Promise<void> {
    const serialized = JSON.stringify(data);
    
    // Store in all layers
    await Promise.all([
      this.redis.setex(key, ttl, serialized),
      this.db.setCachedScore(key, data, ttl),
      this.cdn.set(key, data, ttl)
    ]);
  }
}
```

### Performance Monitoring

```typescript
class PerformanceMonitor {
  private metrics: Map<string, number[]> = new Map();

  recordTiming(operation: string, duration: number): void {
    if (!this.metrics.has(operation)) {
      this.metrics.set(operation, []);
    }
    
    const timings = this.metrics.get(operation)!;
    timings.push(duration);
    
    // Keep only last 100 measurements
    if (timings.length > 100) {
      timings.shift();
    }
  }

  getAverageTiming(operation: string): number {
    const timings = this.metrics.get(operation) || [];
    if (timings.length === 0) return 0;
    
    return timings.reduce((sum, time) => sum + time, 0) / timings.length;
  }

  getP95Timing(operation: string): number {
    const timings = this.metrics.get(operation) || [];
    if (timings.length === 0) return 0;
    
    const sorted = [...timings].sort((a, b) => a - b);
    const index = Math.ceil(sorted.length * 0.95) - 1;
    return sorted[index];
  }
}
```

---

## 9. Security and Compliance

### Authentication and Authorization

```typescript
interface AuthContext {
  tenantId: string;
  userId: string;
  permissions: string[];
  features: TenantFeatures;
}

class AuthMiddleware {
  async authenticate(req: Request): Promise<AuthContext> {
    const token = this.extractToken(req);
    const payload = await this.verifyToken(token);
    
    const tenant = await this.getTenant(payload.tenantId);
    const features = await this.getTenantFeatures(payload.tenantId);
    
    return {
      tenantId: payload.tenantId,
      userId: payload.userId,
      permissions: payload.permissions,
      features
    };
  }

  private extractToken(req: Request): string {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      throw new Error('Invalid authorization header');
    }
    return authHeader.substring(7);
  }
}
```

### Data Privacy and PII Protection

```typescript
class PIIRedactor {
  private patterns = {
    email: /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g,
    phone: /\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g,
    ssn: /\b\d{3}-\d{2}-\d{4}\b/g,
    creditCard: /\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b/g
  };

  redact(text: string): string {
    let redacted = text;
    
    Object.entries(this.patterns).forEach(([type, pattern]) => {
      redacted = redacted.replace(pattern, `[${type.toUpperCase()}]`);
    });
    
    return redacted;
  }
}
```

### Rate Limiting

```typescript
class RateLimiter {
  constructor(private redis: Redis) {}

  async checkLimit(
    key: string, 
    limit: number, 
    window: number
  ): Promise<{ allowed: boolean; remaining: number; resetTime: number }> {
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, window);
    }
    
    const ttl = await this.redis.ttl(key);
    const remaining = Math.max(0, limit - current);
    
    return {
      allowed: current <= limit,
      remaining,
      resetTime: Date.now() + (ttl * 1000)
    };
  }
}
```

---

## 10. Deployment and Infrastructure

### Infrastructure Architecture

```mermaid
graph TB
    subgraph "Frontend (Vercel)"
        A[React App]
        B[Static Assets]
        C[Edge Functions]
    end
    
    subgraph "API Layer (Cloud Run)"
        D[Coordinator API]
        E[Load Balancer]
        F[Auto Scaling]
    end
    
    subgraph "Data Layer"
        G[(Supabase PostgreSQL)]
        H[(Upstash Redis)]
        I[S3 Storage]
    end
    
    subgraph "External Services"
        J[Crawler API]
        K[Prompt API]
        L[Logs API]
    end
    
    A --> E
    E --> D
    D --> G
    D --> H
    D --> I
    D --> J
    D --> K
    D --> L
    
    style A fill:#e1f5fe
    style D fill:#f3e5f5
    style G fill:#e8f5e8
    style H fill:#fff3e0
```

### Docker Configuration

#### Coordinator API Dockerfile
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY src/ ./src/
COPY tsconfig.json ./

RUN npm run build

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

#### Docker Compose for Development
```yaml
version: '3.8'

services:
  coordinator-api:
    build: ./coordinator-api
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://user:pass@postgres:5432/geo_scores
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=geo_scores
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Kubernetes Deployment

#### Coordinator API Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coordinator-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: coordinator-api
  template:
    metadata:
      labels:
        app: coordinator-api
    spec:
      containers:
      - name: coordinator-api
        image: geo-coordinator-api:latest
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: geo-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: geo-secrets
              key: redis-url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Monitoring and Observability

```typescript
class MonitoringService {
  constructor(
    private prometheus: PrometheusClient,
    private logger: Logger
  ) {}

  recordApiCall(service: string, duration: number, success: boolean): void {
    this.prometheus.histogram('api_call_duration_seconds', duration, {
      service,
      success: success.toString()
    });
  }

  recordScoreCalculation(siteId: string, score: number, servicesUsed: number): void {
    this.prometheus.gauge('geo_score', score, {
      site_id: siteId,
      services_used: servicesUsed.toString()
    });
  }

  recordError(error: Error, context: ErrorContext): void {
    this.logger.error('GEO Score Error', {
      error: error.message,
      stack: error.stack,
      context
    });
  }
}
```

---

## Conclusion

This comprehensive guide provides a complete technical and theoretical foundation for implementing a unified GEO score integration system. The architecture leverages modern patterns like the Coordinator pattern, Circuit Breaker, and multi-layer caching to create a robust, scalable solution.

Key benefits of this approach:
- **Unified Interface:** Single API endpoint for all GEO score data
- **Dynamic Weighting:** Automatic adjustment based on service availability
- **High Availability:** Graceful degradation when services are unavailable
- **Performance Optimized:** Multi-layer caching and efficient data management
- **Scalable:** Cloud-native architecture with auto-scaling capabilities
- **Maintainable:** Clear separation of concerns and comprehensive error handling

The system is designed to be startup-friendly while providing enterprise-grade reliability and performance.
