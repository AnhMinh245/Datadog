# 03 - KIẾN TRÚC DATADOG

## 🎯 Mục Tiêu Bài Học
Sau bài học này, bạn sẽ hiểu:
- Kiến trúc tổng thể của Datadog
- Data flow từ application đến dashboard
- Các components và vai trò
- Cách Datadog scale và đảm bảo reliability

---

## 🏗️ Kiến Trúc Tổng Quan

### Big Picture

```
┌─────────────────────────────────────────────────────────┐
│                   YOUR INFRASTRUCTURE                   │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │  Host 1  │  │  Host 2  │  │  Host N  │             │
│  │          │  │          │  │          │             │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │             │
│  │ │Agent │ │  │ │Agent │ │  │ │Agent │ │             │
│  │ └───┬──┘ │  │ └───┬──┘ │  │ └───┬──┘ │             │
│  └─────┼────┘  └─────┼────┘  └─────┼────┘             │
│        │             │             │                   │
└────────┼─────────────┼─────────────┼───────────────────┘
         │             │             │
         └─────────────┴─────────────┘
                       │
              HTTPS (Encrypted)
                       │
         ┌─────────────▼─────────────┐
         │   DATADOG CLOUD BACKEND   │
         │   (SaaS Platform)         │
         │                           │
         │  ┌──────────────────┐    │
         │  │  Data Ingestion  │    │
         │  └────────┬─────────┘    │
         │           │               │
         │  ┌────────▼─────────┐    │
         │  │   Data Storage   │    │
         │  │  - Metrics DB    │    │
         │  │  - Logs Index    │    │
         │  │  - Traces Store  │    │
         │  └────────┬─────────┘    │
         │           │               │
         │  ┌────────▼─────────┐    │
         │  │  Query Engine    │    │
         │  └────────┬─────────┘    │
         │           │               │
         │  ┌────────▼─────────┐    │
         │  │  Alert Engine    │    │
         │  └────────┬─────────┘    │
         │           │               │
         └───────────┼───────────────┘
                     │
         ┌───────────▼───────────┐
         │    WEB INTERFACE      │
         │  - Dashboards         │
         │  - Monitors           │
         │  - Logs Explorer      │
         │  - APM                │
         └───────────────────────┘
                     │
                     ▼
              👤 YOU (User)
```

---

## 🔍 Components Chi Tiết

### 1. Datadog Agent

**Vai trò**: Data collector chạy trên infrastructure của bạn

**Architecture của Agent:**

```
┌─────────────────────────────────────────┐
│         DATADOG AGENT                   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │      Agent Core                 │   │
│  │  - Configuration Manager        │   │
│  │  - Scheduler                    │   │
│  │  - Forwarder (Gửi data)         │   │
│  └──────────┬──────────────────────┘   │
│             │                           │
│  ┌──────────▼──────────────────────┐   │
│  │   Metrics Collection            │   │
│  │  - System metrics (CPU, RAM)    │   │
│  │  - DogStatsD (Custom metrics)   │   │
│  │  - Checks (Integrations)        │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │   Logs Collection               │   │
│  │  - Tail log files               │   │
│  │  - TCP/UDP listeners            │   │
│  │  - Container logs               │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │   Traces Collection             │   │
│  │  - APM traces receiver          │   │
│  │  - Span processing              │   │
│  │  - Sampling                     │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │   Process Collection            │   │
│  │  - Process list                 │   │
│  │  - Process metrics              │   │
│  │  - Network connections          │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Agent Config Files:**
```
/etc/datadog-agent/
├── datadog.yaml          # Main config
├── conf.d/               # Integrations config
│   ├── postgres.d/
│   │   └── conf.yaml
│   ├── redis.d/
│   │   └── conf.yaml
│   └── nginx.d/
│       └── conf.yaml
└── checks.d/             # Custom checks
    └── my_check.py
```

**Data Flow trong Agent:**

```
1. COLLECT
   ↓
[System Metrics] → Collector Thread
[Logs] → Log Agent
[Traces] → Trace Agent
[Custom Metrics] → DogStatsD
   ↓
2. AGGREGATE
   ↓
Combine metrics over flush interval (15s default)
Calculate: avg, min, max, count, rate
   ↓
3. SERIALIZE
   ↓
Convert to Protocol Buffers format
Compress data
   ↓
4. QUEUE
   ↓
Store in memory buffer
(Disk buffer if memory full)
   ↓
5. FORWARD
   ↓
Send to Datadog via HTTPS
Retry if failed (exponential backoff)
```

---

### 2. Datadog Backend (Cloud)

**Multi-Region Architecture:**

```
┌──────────────────────────────────────────────────────────┐
│              DATADOG GLOBAL INFRASTRUCTURE               │
│                                                          │
│  ┌─────────────────┐      ┌─────────────────┐          │
│  │   US1 Region    │      │   EU1 Region    │          │
│  │  (us1.datadoghq │      │  (eu.datadoghq  │          │
│  │   .com)         │      │   .com)         │          │
│  │                 │      │                 │          │
│  │ ┌─────────────┐ │      │ ┌─────────────┐ │          │
│  │ │   Intake    │ │      │ │   Intake    │ │          │
│  │ │   Servers   │ │      │ │   Servers   │ │          │
│  │ └──────┬──────┘ │      │ └──────┬──────┘ │          │
│  │        │        │      │        │        │          │
│  │ ┌──────▼──────┐ │      │ ┌──────▼──────┐ │          │
│  │ │  Processing │ │      │ │  Processing │ │          │
│  │ │   Pipeline  │ │      │ │   Pipeline  │ │          │
│  │ └──────┬──────┘ │      │ └──────┬──────┘ │          │
│  │        │        │      │        │        │          │
│  │ ┌──────▼──────┐ │      │ ┌──────▼──────┐ │          │
│  │ │   Storage   │ │      │ │   Storage   │ │          │
│  │ │  (Time-     │ │      │ │  (Time-     │ │          │
│  │ │   series DB)│ │      │ │   series DB)│ │          │
│  │ └─────────────┘ │      │ └─────────────┘ │          │
│  └─────────────────┘      └─────────────────┘          │
│                                                          │
│  ┌───────────────────────────────────────────┐          │
│  │      Global Load Balancer / CDN           │          │
│  └───────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────┘
```

**Available Sites:**
```
US1: app.datadoghq.com (Default)
US3: us3.datadoghq.com
US5: us5.datadoghq.com
EU1: app.datadoghq.eu (GDPR compliant)
```

---

### 3. Data Processing Pipeline

**Metrics Pipeline:**

```
┌──────────────────────────────────────────────────────────┐
│                  METRICS PIPELINE                        │
└──────────────────────────────────────────────────────────┘

1. INGESTION
   │
   ▼
[Agent sends metrics every 15s]
   │
   ▼
API Endpoint: https://api.datadoghq.com/api/v1/series
   │
   ▼
2. VALIDATION
   │
   ├─ Check API key
   ├─ Validate format
   ├─ Rate limiting check
   └─ Cardinality check
   │
   ▼
3. ENRICHMENT
   │
   ├─ Add host tags
   ├─ Add integration tags
   └─ Resolve hostnames
   │
   ▼
4. AGGREGATION
   │
   ├─ Rollup (1min, 5min, 1hour)
   ├─ Calculate percentiles
   └─ Downsample old data
   │
   ▼
5. STORAGE
   │
   ├─ Hot storage (Recent data, fast)
   ├─ Warm storage (30 days)
   └─ Cold storage (15 months)
   │
   ▼
6. INDEXING
   │
   └─ Index by: metric name, tags, host
   │
   ▼
7. QUERY READY ✅
```

**Logs Pipeline:**

```
┌──────────────────────────────────────────────────────────┐
│                   LOGS PIPELINE                          │
└──────────────────────────────────────────────────────────┘

1. COLLECTION
   │
   ▼
[Agent tails log files]
   │
   ▼
2. PARSING
   │
   ├─ Auto-detect log format
   ├─ Extract timestamp
   ├─ Detect status (info/error)
   └─ Parse JSON/Syslog/Custom
   │
   ▼
3. FILTERING (Optional)
   │
   ├─ Exclusion filters
   │   Example: Drop debug logs
   │
   └─ Sampling
       Example: Keep 10% of info logs
   │
   ▼
4. ENRICHMENT
   │
   ├─ Add tags
   ├─ GeoIP lookup
   └─ Correlate with traces
   │
   ▼
5. INDEXING DECISION
   │
   ├─ Index Filters
   │   ↓
   │   Indexed → Query & Alert ✅
   │
   └─ Not Indexed → Archive only 📦
   │
   ▼
6. ARCHIVE (All logs)
   │
   └─ Send to S3/GCS
       (For compliance, long-term storage)
```

**Traces Pipeline:**

```
┌──────────────────────────────────────────────────────────┐
│                  TRACES PIPELINE                         │
└──────────────────────────────────────────────────────────┘

1. INSTRUMENTATION
   │
   ▼
[App sends spans to Agent]
   │
   ▼
2. AGENT PROCESSING
   │
   ├─ Collect spans
   ├─ Build trace from spans
   ├─ Calculate trace metrics
   │   (latency, errors, hits)
   └─ Apply sampling
   │
   ▼
3. SAMPLING
   │
   ├─ Priority Sampling
   │   ├─ Always keep errors
   │   ├─ Keep slow traces
   │   └─ Sample normal traces
   │
   └─ Default: 100% trace metrics
                1% indexed traces
   │
   ▼
4. BACKEND PROCESSING
   │
   ├─ Service Map generation
   ├─ Dependency graph
   └─ Performance analytics
   │
   ▼
5. STORAGE
   │
   ├─ Indexed Traces (15 days)
   │   → Full search capability
   │
   └─ Trace Metrics (15 months)
       → Aggregated statistics
```

---

## 📊 Data Flow Examples

### Example 1: Web Request Journey

```
USER ACTION: Click "Buy Now" button
│
▼
┌─────────────────────────────────────────────────────────┐
│ FRONTEND (Browser)                                      │
│ ├─ RUM collects:                                        │
│ │  - Page load time: 250ms                             │
│ │  - User action: button_click                         │
│ │  - Session ID: abc123                                │
│ └─ Send to Datadog ──────────────────────────┐         │
└───────────────────────────────────────────────┼─────────┘
                                                │
                                                ▼
┌─────────────────────────────────────────────────────────┐
│ BACKEND API (Python)                                    │
│ ├─ APM Tracer creates:                                  │
│ │  - Trace ID: trace_xyz                               │
│ │  - Span: POST /api/checkout                          │
│ │  - Duration: 300ms                                   │
│ │                                                       │
│ ├─ Logger writes:                                       │
│ │  - "Order created: order_id=789"                     │
│ │  - trace_id=trace_xyz (injected)                     │
│ │                                                       │
│ ├─ StatsD sends:                                        │
│ │  - Counter: orders.created                           │
│ │  - Gauge: active_orders = 150                        │
│ │                                                       │
│ └─ All sent to Agent ────────────────────────┐         │
└───────────────────────────────────────────────┼─────────┘
                                                │
                                                ▼
┌─────────────────────────────────────────────────────────┐
│ DATADOG AGENT                                           │
│ ├─ Receives:                                            │
│ │  - Traces from APM                                   │
│ │  - Logs from file                                    │
│ │  - Metrics from StatsD                               │
│ │                                                       │
│ ├─ Adds tags:                                           │
│ │  - host:web-server-01                                │
│ │  - env:production                                    │
│ │  - service:checkout-api                              │
│ │                                                       │
│ └─ Forwards to Datadog Cloud ─────────────┐            │
└────────────────────────────────────────────┼────────────┘
                                             │
                                             ▼
┌──────────────────────────────────────────────────────────┐
│ DATADOG BACKEND                                          │
│                                                          │
│ CORRELATES DATA:                                         │
│ ├─ Trace trace_xyz:                                      │
│ │  ├─ 300ms total                                       │
│ │  ├─ Service: checkout-api                            │
│ │  └─ Status: OK                                        │
│ │                                                       │
│ ├─ Logs with trace_id=trace_xyz:                        │
│ │  └─ "Order created: order_id=789"                    │
│ │                                                       │
│ ├─ Metrics at timestamp:                                │
│ │  ├─ orders.created +1                                │
│ │  └─ active_orders = 150                              │
│ │                                                       │
│ └─ RUM Session abc123:                                  │
│    └─ Button click → Success                           │
│                                                          │
└──────────────────────────────────────────────────────────┘
                         │
                         ▼
             ┌───────────────────────┐
             │   YOUR DASHBOARD      │
             │                       │
             │ Shows:                │
             │ - Trace timeline      │
             │ - Related logs        │
             │ - Metrics graph       │
             │ - User session        │
             └───────────────────────┘
```

---

## 🔐 Security Architecture

### Data Encryption

```
┌──────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                       │
└──────────────────────────────────────────────────────────┘

1. IN TRANSIT
   │
   ├─ Agent → Datadog Backend
   │  └─ TLS 1.2+ encryption
   │     (HTTPS with cert pinning)
   │
   └─ Browser → Datadog Web
      └─ TLS 1.2+ encryption

2. AT REST
   │
   ├─ Data in Datadog storage
   │  └─ AES-256 encryption
   │
   └─ Archives (S3/GCS)
      └─ Server-side encryption

3. ACCESS CONTROL
   │
   ├─ API Keys
   │  ├─ API Key (Write only)
   │  └─ Application Key (Read/Write)
   │
   ├─ RBAC (Role-Based Access Control)
   │  ├─ Admin
   │  ├─ Standard
   │  └─ Read-only
   │
   └─ SSO Integration
      ├─ SAML
      ├─ OAuth
      └─ Google/Microsoft SSO
```

### API Keys

```python
# Two types of keys

1. API Key (dd-api-key)
   - Used by Agent to send data
   - Write-only
   - Per organization
   - Example: 1234567890abcdef1234567890abcdef

2. Application Key (dd-app-key)
   - Used for API queries
   - Read/Write access
   - Per user
   - Example: abcdef1234567890abcdef1234567890
```

**Best Practices:**
```
✅ Rotate keys regularly
✅ Use different keys per environment
✅ Store in secrets manager (Vault, AWS Secrets)
✅ Never commit keys to git
✅ Restrict application key scope
```

---

## 🌐 Network Architecture

### Endpoints

```
┌──────────────────────────────────────────────────────────┐
│               DATADOG API ENDPOINTS (US1)                │
├──────────────────────────────────────────────────────────┤
│ Metrics:                                                 │
│   https://api.datadoghq.com/api/v1/series               │
│                                                          │
│ Logs:                                                    │
│   https://http-intake.logs.datadoghq.com/v1/input       │
│   TCP: intake.logs.datadoghq.com:10516 (TLS)            │
│                                                          │
│ Traces:                                                  │
│   https://trace.agent.datadoghq.com                     │
│                                                          │
│ Process:                                                 │
│   https://process.datadoghq.com                         │
│                                                          │
│ Orchestrator:                                            │
│   https://orchestrator.datadoghq.com                    │
└──────────────────────────────────────────────────────────┘
```

### Firewall Rules

```
OUTBOUND (Allow từ Agent):
─────────────────────────────
Port 443 (HTTPS)
  → *.datadoghq.com
  → *.datadoghq.eu (if EU)

Port 10516 (TCP Logs - Optional)
  → intake.logs.datadoghq.com

INBOUND (Agent không cần):
─────────────────────────────
Không cần mở inbound ports
Agent chỉ tạo outbound connections
```

### Proxy Configuration

```yaml
# datadog.yaml - Cấu hình proxy
proxy:
  https: http://proxy.company.com:3128
  no_proxy:
    - 169.254.169.254  # AWS metadata
    - localhost
```

---

## 📈 Scalability & Performance

### How Datadog Scales

```
┌──────────────────────────────────────────────────────────┐
│              DATADOG SCALABILITY MODEL                   │
└──────────────────────────────────────────────────────────┘

YOUR SIDE:
──────────
Agent per host
  ├─ Collects data locally
  ├─ Aggregates before sending
  ├─ Uses local buffer
  └─ Auto-throttles if backend slow

DATADOG SIDE:
─────────────
1. Horizontal Scaling
   ├─ Add more intake servers
   ├─ Add more processing workers
   └─ Add more storage nodes

2. Sharding
   ├─ Partition by customer
   ├─ Partition by time range
   └─ Partition by metric namespace

3. Caching
   ├─ Query result cache
   ├─ CDN for static assets
   └─ Tag metadata cache

4. Auto-scaling
   └─ Scale based on load
       (Managed by Datadog, transparent to you)
```

### Performance Numbers

```
TYPICAL LATENCY:
────────────────
Metrics ingestion → visible in UI: 5-10 seconds
Logs ingestion → searchable: 10-30 seconds
Traces ingestion → visible: 10-20 seconds
Dashboard queries: 200-500ms
Alert evaluation: 1-2 minutes

THROUGHPUT (per account):
─────────────────────────
Metrics: Millions of points/second
Logs: Terabytes/day
Traces: Billions of spans/day
```

---

## 🔧 High Availability

### Agent HA

```
Agent Level:
────────────
✅ Local buffering (memory + disk)
✅ Automatic retry with backoff
✅ Health checks
✅ Graceful shutdown

If Datadog is down:
───────────────────
1. Agent buffers data (up to 5 minutes)
2. Retries with exponential backoff
3. Drops oldest data if buffer full
4. Logs warning locally

When Datadog is back:
─────────────────────
✅ Agent automatically reconnects
✅ Sends buffered data
✅ Normal operation resumes
```

### Datadog Backend HA

```
Infrastructure:
───────────────
✅ Multi-AZ deployment
✅ Auto-failover
✅ Data replication
✅ 99.9% SLA uptime

Monitoring their own system:
────────────────────────────
✅ Datadog monitors Datadog!
✅ Status page: status.datadoghq.com
✅ Incident notifications
```

---

## 📝 Tóm Tắt

### Architecture Key Points

```
1. AGENT-BASED ARCHITECTURE
   - Agent runs on your infrastructure
   - Collects and forwards data
   - Lightweight, ~50MB RAM

2. SAAS BACKEND
   - Fully managed by Datadog
   - Multi-region
   - Auto-scaling

3. DATA FLOW
   App → Agent → Datadog Backend → You
   
4. THREE PIPELINES
   - Metrics: 15s flush, 15 months retention
   - Logs: Realtime, configurable retention
   - Traces: Sampled, 15 days + metrics

5. SECURITY
   - TLS encryption in transit
   - AES-256 at rest
   - API key authentication
   - RBAC for access control
```

### Components Summary

| Component | Location | Role |
|-----------|----------|------|
| **Agent** | Your infra | Collector |
| **DogStatsD** | Within Agent | Custom metrics receiver |
| **Trace Agent** | Within Agent | APM traces receiver |
| **Log Agent** | Within Agent | Logs collector |
| **Backend** | Datadog Cloud | Processing & Storage |
| **Web UI** | Browser | Visualization & Control |

---

## 🤔 Kiểm Tra Hiểu Biết

### Phân tích data flow:

Vẽ data flow cho scenario:
```
Một user click button "Login"
→ Tạo API request
→ Database query
→ Return response

Trace qua:
1. Có bao nhiêu spans?
2. Logs nào được generate?
3. Metrics nào được collect?
```

### Thiết kế architecture:

Thiết kế monitoring cho:
```
Infrastructure:
- 10 web servers
- 5 API servers
- 2 database servers
- 1 Redis server

Questions:
1. Cần bao nhiêu agents?
2. Network requirements?
3. Estimated data volume?
```

### Đánh giá security:

Review security:
```
1. API keys được store ở đâu?
2. Firewall rules cần gì?
3. Data có encrypted không?
4. Compliance requirements? (GDPR, HIPAA)
```

---

## ➡️ Bước Tiếp Theo

Bạn đã hiểu kiến trúc của Datadog!

**Bài tiếp theo**: [04 - Cài Đặt Datadog Agent](04-CAI-DAT-AGENT.md)

Trong bài tiếp theo, chúng ta sẽ thực hành cài đặt Agent và gửi data đầu tiên lên Datadog.

---

**📌 Ghi Chú Của Bạn**
```
(Viết ghi chú, câu hỏi, hoặc ý tưởng của bạn ở đây)








```

