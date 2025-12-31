# 19 - USE CASES THỰC TẾ

## 🎯 Mục Tiêu
Tìm hiểu các use cases thực tế của Datadog trong nhiều scenarios khác nhau, từ startup đến enterprise.

---

## 🛒 Use Case 1: E-Commerce Platform

### **Scenario**
```
Company: Online retail store
Traffic: 100,000 users/day
Tech stack: 
  - Frontend: React
  - Backend: Node.js microservices
  - Database: PostgreSQL, Redis
  - Infrastructure: AWS (EC2, RDS, ElastiCache)
```

### **Pain Points**
```
❌ Không biết service nào causing slowness
❌ Khách hàng complain checkout chậm
❌ Không track được business metrics
❌ Khó troubleshoot issues across services
❌ Alert đến muộn, mất revenue
```

### **Datadog Solution**

#### **1. Infrastructure Monitoring**
```yaml
Monitor:
  - EC2 instances: CPU, Memory, Disk
  - RDS: Connections, slow queries, replication lag
  - ElastiCache: Cache hit rate, evictions
  - ALB: Request count, target response time

Dashboard: "Infrastructure Health"
Widgets:
  - Host map colored by CPU
  - Database connections timeline
  - Cache hit ratio
  - Load balancer latency
```

#### **2. APM Setup**
```javascript
// tracer.js
require('dd-trace').init({
  service: 'checkout-api',
  env: 'production',
  version: process.env.APP_VERSION,
  logInjection: true
});

// Key services to trace:
const services = [
  'frontend',
  'product-api',
  'cart-api',
  'checkout-api',
  'payment-api',
  'inventory-api',
  'notification-api'
];

// Service map shows dependencies
```

**Result:**
```
Discovered bottleneck:
  checkout-api → payment-api: 2.5s (SLOW!)
    → payment-api → Stripe API: 2.3s
  
Issue: Synchronous call to Stripe
Solution: Move to async queue
Result: Checkout time: 2.5s → 300ms ✅
```

#### **3. Business Metrics**
```python
from datadog import statsd

class BusinessMetrics:
    @staticmethod
    def track_order(order):
        # Order placed
        statsd.increment(
            'orders.placed',
            tags=[
                f'country:{order.country}',
                f'payment_method:{order.payment_method}',
                f'user_tier:{order.user.tier}'
            ]
        )
        
        # Revenue
        statsd.histogram(
            'revenue.amount',
            order.total,
            tags=[
                f'currency:{order.currency}',
                f'country:{order.country}'
            ]
        )
        
        # Items sold
        statsd.increment(
            'items.sold',
            len(order.items),
            tags=[f'category:{item.category}' for item in order.items]
        )
    
    @staticmethod
    def track_cart_abandonment(user_id, cart):
        statsd.increment(
            'cart.abandoned',
            tags=[
                f'cart_value:{cart.get_value_bucket()}',
                f'items_count:{len(cart.items)}'
            ]
        )
```

**Dashboard: "Business Metrics"**
```yaml
Widgets:
  - Orders per minute (timeseries)
  - Revenue today vs yesterday (query value)
  - Top selling products (top list)
  - Conversion rate % (query value with formula):
      (orders / sessions) * 100
  - Cart abandonment rate
  - Average order value
```

#### **4. Critical Alerts**
```yaml
Alert 1: "Checkout Success Rate Low"
  Query: (sum:orders.successful / sum:checkout.attempts) * 100 < 95
  Threshold: < 95%
  Notification: @pagerduty-critical @slack-engineering
  Impact: Direct revenue loss

Alert 2: "Payment Processing Failures"
  Query: sum:payment.failed{*} > 5 in last 5 minutes
  Notification: @pagerduty-payments
  
Alert 3: "Inventory Service Down"
  Query: No data from inventory-api
  Notification: @slack-ops
  Impact: Can't show product availability

Alert 4: "Slow Checkout (p95 latency)"
  Query: p95:trace.checkout.duration{*} > 1000ms
  Notification: @slack-engineering

Alert 5: "Database Connection Pool Exhausted"
  Query: avg:postgresql.connections.max{*} > 90
  Notification: @slack-dba @pagerduty-ops
```

#### **5. Real User Monitoring (RUM)**
```javascript
// Frontend
import { datadogRum } from '@datadog/browser-rum';

datadogRum.init({
  applicationId: '<APP_ID>',
  clientToken: '<CLIENT_TOKEN>',
  site: 'datadoghq.com',
  service: 'ecommerce-web',
  env: 'production',
  version: '1.0.0',
  sampleRate: 100,
  trackInteractions: true,
  trackResources: true,
  trackLongTasks: true
});

// Track business events
function trackPurchase(order) {
  datadogRum.addAction('purchase_completed', {
    order_id: order.id,
    amount: order.total,
    items: order.items.length
  });
}

function trackError(error) {
  datadogRum.addError(error);
}
```

**RUM Insights:**
```
Discovered:
- Mobile users: 40% slower than desktop
- Chrome: Fastest browser
- Safari on iOS 14: JavaScript errors → Fixed!
- Page load time spike at 8 PM (peak traffic)
```

### **Results After 3 Months**

```
✅ Checkout success rate: 92% → 98%
✅ p95 latency: 2.5s → 300ms
✅ MTTR (Mean Time To Recovery): 45 min → 8 min
✅ Revenue increase: 15% (faster, more reliable checkout)
✅ Customer satisfaction: Up 22%
✅ Prevented 4 major outages with proactive alerts
✅ Saved $50k/month with infrastructure optimization
```

---

## 📱 Use Case 2: Mobile Banking App

### **Scenario**
```
Company: Digital bank
Users: 2 million active users
Tech stack:
  - Mobile: iOS + Android apps
  - Backend: Microservices (Java/Spring Boot)
  - Database: Oracle, MongoDB
  - Message Queue: Kafka
  - Infrastructure: On-premise + Cloud hybrid
```

### **Requirements**
```
🔒 High security
💰 Zero tolerance for transaction errors
⚡ Low latency (< 200ms API response)
📊 Regulatory compliance (audit logs)
🌍 24/7 availability
```

### **Datadog Implementation**

#### **1. Critical Path Monitoring**
```
Transaction Flow:
  Mobile App
    ↓
  API Gateway (Kong)
    ↓
  Auth Service → Verify Token
    ↓
  Account Service → Check Balance
    ↓
  Transaction Service → Process Transfer
    ↓
  Notification Service → Send SMS/Push
    ↓
  Audit Service → Log Transaction
```

**APM Tracing:**
```java
import datadog.trace.api.Trace;

@Trace(operationName = "transaction.transfer", resourceName = "POST /transfer")
public TransferResponse processTransfer(TransferRequest request) {
    Span span = GlobalTracer.get().activeSpan();
    
    // Add business context
    span.setTag("transaction.amount", request.getAmount());
    span.setTag("transaction.type", request.getType());
    span.setTag("user.tier", request.getUser().getTier());
    
    try {
        // Validate
        validateTransfer(request);
        
        // Check balance
        Balance balance = accountService.getBalance(request.getFromAccount());
        
        // Process
        TransferResult result = executeTransfer(request);
        
        // Audit
        auditService.logTransaction(result);
        
        // Notify
        notificationService.sendConfirmation(result);
        
        return new TransferResponse(result);
        
    } catch (InsufficientFundsException e) {
        span.setTag("error", true);
        span.setTag("error.type", "insufficient_funds");
        throw e;
    }
}
```

#### **2. Security Monitoring**
```python
from datadog import statsd
import logging

class SecurityMonitor:
    
    @staticmethod
    def track_login_attempt(username, success, ip):
        """Track login attempts for anomaly detection"""
        statsd.increment(
            'auth.login.attempt',
            tags=[
                f'success:{success}',
                f'ip:{ip}',
                'service:auth'
            ]
        )
        
        if not success:
            # Count failed attempts
            statsd.increment(
                'auth.login.failed',
                tags=[f'username:{hash(username)}']  # Hashed for privacy
            )
            
            # Log for audit
            logging.warning(
                'Failed login attempt',
                extra={
                    'username_hash': hash(username),
                    'ip': ip,
                    'timestamp': datetime.utcnow(),
                    'event.type': 'security.login_failed'
                }
            )
    
    @staticmethod
    def track_suspicious_transaction(transaction):
        """Flag suspicious transactions"""
        if transaction.amount > 10000 or transaction.is_high_risk():
            statsd.increment(
                'security.suspicious_transaction',
                tags=[
                    f'amount_bracket:{get_bracket(transaction.amount)}',
                    f'risk_level:{transaction.risk_level}'
                ]
            )
            
            logging.warning(
                'Suspicious transaction detected',
                extra={
                    'transaction_id': transaction.id,
                    'amount': transaction.amount,
                    'from_account': transaction.from_account,
                    'to_account': transaction.to_account,
                    'risk_score': transaction.risk_score,
                    'event.type': 'security.suspicious_tx'
                }
            )
```

**Security Alerts:**
```yaml
Alert 1: "Brute Force Attack"
  Query: sum:auth.login.failed{*} by {ip} > 10 in 5 minutes
  Action: 
    - Block IP automatically
    - Alert security team
    - Page on-call if > 100 attempts

Alert 2: "Unusual Transaction Pattern"
  Query: anomalies(sum:transactions.amount{*}, 'agile', 3)
  Notification: @security-team
  
Alert 3: "Multiple Failed Transactions"
  Query: sum:transaction.failed{*} by {user_id} > 5 in 10 min
  Action: Lock account, notify user

Alert 4: "API Abuse"
  Query: sum:api.requests{*} by {api_key} > 1000 in 1 minute
  Action: Rate limit, alert
```

#### **3. Compliance & Audit**
```python
# Structured logging for compliance
import json
import logging

class AuditLogger:
    """Compliance audit logging"""
    
    @staticmethod
    def log_transaction(transaction, user):
        audit_log = {
            'timestamp': datetime.utcnow().isoformat(),
            'event_type': 'transaction',
            'transaction_id': transaction.id,
            'user_id': user.id,
            'amount': transaction.amount,
            'currency': transaction.currency,
            'from_account': transaction.from_account,
            'to_account': transaction.to_account,
            'status': transaction.status,
            'ip_address': request.remote_addr,
            'user_agent': request.user_agent,
            'trace_id': get_trace_id(),  # Link to APM trace
            'compliance': {
                'regulation': 'PSD2',
                'retention_period': '7_years',
                'category': 'financial_transaction'
            }
        }
        
        logging.info('Transaction processed', extra=audit_log)
```

**Log Retention:**
```yaml
# Index for 30 days (fast search)
Index Filter: @event_type:transaction

# Archive to S3 for 7 years (compliance)
Archive: All logs → S3 bucket
  - Encrypted at rest
  - Access logged
  - Immutable

# Rehydrate when needed for investigations
```

#### **4. Performance SLOs**
```yaml
SLO 1: "API Availability"
  Target: 99.95% uptime
  Time window: 30 days
  Metric: sum:api.requests.success / sum:api.requests.total
  Error budget: 21.6 minutes/month
  
SLO 2: "Transaction Latency"
  Target: 95% of transactions < 200ms
  Metric: p95:trace.transaction.duration < 200
  
SLO 3: "Transaction Success Rate"
  Target: 99.9% success
  Metric: sum:transaction.success / sum:transaction.total
  
SLO 4: "Mobile App Crash Rate"
  Target: < 0.1% crash rate
  Metric: (rum.errors / rum.sessions) * 100 < 0.1
```

#### **5. Cost Optimization**
```yaml
# High-volume, low-value metrics: Sample
auth.token.validated: 10% sample rate
api.health_check: Don't index logs
session.keep_alive: Aggregate per minute

# Critical metrics: 100%
transaction.*: 100% (revenue critical)
security.*: 100% (compliance)
error.*: 100% (troubleshooting)

# Logs
Index only:
  - Errors
  - Transactions
  - Security events
  - Audit logs
  
Don't index:
  - Debug logs
  - Health checks
  - Info logs (archive only)

Result: 70% cost reduction while maintaining visibility
```

### **Results**

```
✅ Transaction success rate: 99.92% (Target: 99.9%)
✅ API p95 latency: 180ms (Target: < 200ms)
✅ MTTD (Mean Time To Detect): 45s
✅ MTTR: 12 minutes
✅ Zero compliance violations
✅ Prevented 3 security breaches (blocked before damage)
✅ Mobile app crash rate: 0.08%
✅ Customer trust score: Increased 18%
```

---

## 🎮 Use Case 3: Gaming Platform

### **Scenario**
```
Company: Multiplayer online game
Players: 5 million MAU
Tech stack:
  - Game servers: C++ (custom engine)
  - Matchmaking: Go microservices
  - Player data: Cassandra
  - Real-time: WebSocket
  - Infrastructure: AWS (EC2, ECS, DynamoDB)
```

### **Challenges**
```
⚡ Real-time performance critical (<50ms latency)
📊 Massive metrics volume (millions/second)
🌍 Global player base (multi-region)
🎯 Match quality depends on fast matchmaking
💰 Revenue from in-game purchases
```

### **Solution Highlights**

#### **1. Custom Metrics via DogStatsD**
```cpp
// Game server C++ code
#include "dogstatsd.hpp"

DogStatsD statsd("localhost", 8125);

class GameServer {
public:
    void on_player_join(Player player) {
        // Track active players
        statsd.gauge("game.players.active", active_players);
        statsd.increment("game.players.joined", {
            "region:" + player.region,
            "tier:" + player.tier
        });
    }
    
    void on_match_complete(Match match) {
        // Match duration
        statsd.histogram("game.match.duration", match.duration_ms);
        
        // Match quality score
        statsd.histogram("game.match.quality", match.quality_score);
        
        // Players per match
        statsd.histogram("game.match.players", match.player_count);
    }
    
    void on_tick() {
        // Server tick rate
        statsd.histogram("game.server.tick", tick_duration_ms);
        
        // Player latency
        for (auto player : players) {
            statsd.histogram("game.player.latency", player.latency, {
                "region:" + player.region
            });
        }
    }
};
```

#### **2. Matchmaking Performance**
```go
// Matchmaking service
package matchmaking

import "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"

func FindMatch(player Player) (*Match, error) {
    span := tracer.StartSpan("matchmaking.find")
    defer span.Finish()
    
    span.SetTag("player.rank", player.Rank)
    span.SetTag("player.region", player.Region)
    
    // Search for players
    candidates := searchPlayers(player)
    span.SetTag("candidates.found", len(candidates))
    
    // Create match
    match := createMatch(candidates)
    span.SetTag("match.id", match.ID)
    span.SetTag("match.quality", match.QualityScore)
    
    // Metrics
    statsd.Histogram("matchmaking.duration", span.Duration())
    statsd.Histogram("matchmaking.quality", match.QualityScore)
    
    return match, nil
}
```

**Discovered Issue:**
```
Problem: Matchmaking taking > 30 seconds at peak times
APM trace showed:
  - searchPlayers: 100ms ✅
  - createMatch: 5ms ✅
  - validatePlayers: 25 seconds ⚠️ (N+1 query to Cassandra!)

Solution: Batch validation
Result: 30s → 500ms
Player satisfaction: +35%
```

#### **3. Regional Performance Dashboard**
```yaml
Dashboard: "Global Game Performance"

Template Variables:
  - $region: us-east, eu-west, ap-southeast
  - $game_mode: battle_royale, team_deathmatch

Widgets:
  Row 1: Overview
    - Active players by region (map widget)
    - Current CCU (Concurrent Users)
    - Matches in progress
    
  Row 2: Performance
    - Server tick rate by region
    - Player latency (p50, p95, p99) by region
    - Matchmaking duration by region
    
  Row 3: Quality
    - Match quality score
    - Player dropoff rate
    - Reconnection rate
    
  Row 4: Revenue
    - In-game purchases per minute
    - Revenue by region
    - ARPU (Average Revenue Per User)
```

#### **4. Player Experience Monitoring**
```javascript
// Client-side (Unity/Unreal Engine integration)
using Datadog.Unity;

public class PlayerMonitoring : MonoBehaviour {
    void Start() {
        DatadogSDK.Instance.SetUserInfo(
            id: playerId,
            attributes: new Dictionary<string, object> {
                {"tier", playerTier},
                {"region", playerRegion},
                {"device", SystemInfo.deviceModel}
            }
        );
    }
    
    void Update() {
        // Frame rate
        float fps = 1.0f / Time.deltaTime;
        DatadogSDK.Instance.SendMetric("game.client.fps", fps);
        
        // Latency
        DatadogSDK.Instance.SendMetric("game.client.latency", networkLatency);
    }
    
    void OnApplicationError(string message, string stackTrace) {
        DatadogSDK.Instance.AddError(
            message: message,
            source: "Unity",
            stackTrace: stackTrace
        );
    }
}
```

---

## 📝 Tóm Tắt

### **Patterns Chung Across Use Cases**

```
1. START WITH BUSINESS METRICS
   - What matters to users?
   - What drives revenue?
   - What indicates health?

2. INSTRUMENT CRITICAL PATHS
   - User journeys
   - Transaction flows
   - Key operations

3. MULTI-LAYER MONITORING
   - Infrastructure
   - Application
   - User experience
   - Business

4. PROACTIVE ALERTING
   - SLO-based alerts
   - Anomaly detection
   - Composite monitors

5. CORRELATION
   - Metrics ↔ Logs ↔ Traces
   - Infrastructure ↔ Application
   - Technical ↔ Business

6. COST OPTIMIZATION
   - Sample high-volume
   - Index strategically
   - Archive for compliance

7. CONTINUOUS IMPROVEMENT
   - Review weekly
   - Optimize monthly
   - Iterate based on incidents
```

---

## ➡️ Bước Tiếp Theo

Áp dụng những patterns này vào use case của bạn!

**Quay lại**: [README](README.md) để xem tổng quan khóa học

---

**📌 Ghi Chú Của Bạn**
```
(Use case của công ty bạn, ý tưởng áp dụng)








```

