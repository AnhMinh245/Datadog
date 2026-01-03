# 13 - SERVICE LEVEL OBJECTIVES (SLO)

## 🎯 Mục Tiêu
Hiểu sâu về SLO, SLA, SLI - bản chất, khi nào dùng, và cách áp dụng cho banking systems.

---

## 📚 Bản Chất: SLI, SLO, SLA

### **Ba Concepts Liên Quan**

```
SLI (Service Level Indicator)
  ↓ Đo lường
SLO (Service Level Objective)  
  ↓ Cam kết
SLA (Service Level Agreement)
```

### **SLI - Service Level Indicator**

**Định nghĩa:**
> Metric đo lường chất lượng service từ góc nhìn user

**Ví dụ:**
```
Request Success Rate:
  SLI = (Successful Requests / Total Requests) × 100

API Latency:
  SLI = p95 response time

Availability:
  SLI = (Uptime / Total Time) × 100
```

**Good SLI Characteristics:**
```
✓ Measurable - Có thể đo được chính xác
✓ Meaningful - Quan trọng với users
✓ Actionable - Team có thể improve
✓ Simple - Dễ hiểu, dễ communicate

Example Banking SLIs:
- Transaction success rate
- ATM availability
- Mobile app login success rate
- Payment processing time
```

---

### **SLO - Service Level Objective**

**Định nghĩa:**
> Target value cho SLI - "Chúng ta muốn đạt X%"

**Format:**
```
[SLI] should be [≥/≤] [Target] over [Time Window]

Examples:
- API success rate ≥ 99.9% over 30 days
- p95 latency ≤ 200ms over 7 days
- Availability ≥ 99.95% over 30 days
```

**Tại Sao Cần SLO?**
```
Without SLO:
❌ "Service nhanh là được" - Vague
❌ Không có measurement of success
❌ Team không biết prioritize gì

With SLO:
✅ "p95 latency ≤ 200ms" - Concrete
✅ Clear success criteria
✅ Data-driven decisions
✅ Balance reliability vs velocity
```

---

### **SLA - Service Level Agreement**

**Định nghĩa:**
> Contract với customers - "Nếu không đạt, sẽ có consequences"

**SLO vs SLA:**
```
SLO (Internal):
- Team's target
- Can be tighter than SLA
- Failure = Internal incident
- Example: 99.95% uptime

SLA (External):
- Customer commitment
- Usually looser than SLO
- Failure = Refund/penalty
- Example: 99.9% uptime

Why looser? Buffer zone (error budget)
```

**Banking Example:**
```
Service: Payment Processing API

SLI:
  - Success rate = successful payments / total attempts
  
SLO (Internal):
  - Success rate ≥ 99.95% over 30 days
  - p95 latency ≤ 150ms
  
SLA (Customer-facing):
  - Success rate ≥ 99.9% over 30 days
  - p95 latency ≤ 200ms
  - If violated: 10% monthly credit

Buffer: 0.05% for deployments, incidents
```

---

## 🎯 Error Budget

### **Concept:**

**Định nghĩa:**
> Lượng "failure được phép" trong time window

**Calculation:**
```
Error Budget = 100% - SLO

Example:
SLO: 99.9% availability over 30 days
Error Budget: 0.1% = 43.2 minutes downtime allowed

30 days = 43,200 minutes
0.1% of 43,200 = 43.2 minutes
```

### **Tại Sao Quan Trọng?**

**Balance Reliability vs Innovation:**
```
Scenario 1: Error Budget Remaining
  Status: SLO đang đạt, còn budget
  Decision: ✅ Deploy new features
           ✅ Experiment
           ✅ Move fast
  
Scenario 2: Error Budget Exhausted
  Status: SLO bị vi phạm, hết budget
  Decision: ⛔ Freeze deployments
           ⛔ Focus on reliability
           ⛔ Fix issues first
           
Result: Data-driven balance
```

**Banking Example:**
```
Core Banking API
SLO: 99.95% availability (21.6 min/month error budget)

Month Progress:
Week 1: 1 incident = 5 min downtime (23% budget used)
Week 2: Smooth (23% budget used)
Week 3: 2 incidents = 10 min (69% budget used)
Week 4 Decision:
  ✅ Still have 31% budget → Continue normal operations
  
If budget exhausted:
  ⛔ Freeze all changes
  ⛔ Only emergency fixes
  ⛔ Root cause analysis mandatory
```

---

## 🏦 Banking SLO Patterns

### **Availability SLOs**

**Transaction Processing:**
```
Target: 99.95% (21.6 min downtime/month)

Why 99.95% not 99.99%?
- Balance cost vs benefit
- 99.99% = 4x more expensive infrastructure
- 99.95% acceptable for non-critical hours
- Higher during business hours (time-based SLO)

Measurement:
SLI = (Successful requests / Total requests) × 100

Exclude from SLI:
- Scheduled maintenance (pre-announced)
- User errors (400 status codes)
- DDoS attacks (external factors)
```

**Time-Based SLO (Advanced):**
```
Business Hours (8 AM - 8 PM):
  SLO: 99.99% availability (52 seconds downtime/month)
  
Non-Business Hours (8 PM - 8 AM):
  SLO: 99.9% availability (5.2 min downtime/month)
  
Rationale: Higher impact during business hours
```

### **Latency SLOs**

**API Response Time:**
```
Why use percentiles, not average?

❌ Average: 100ms
   - Doesn't show user experience
   - 99% requests could be fast, 1% very slow
   
✅ p95: 200ms
   - 95% of users experience ≤ 200ms
   - Captures long tail
   
✅ p99: 500ms
   - 99% of users experience ≤ 500ms
   - Catches worst cases

Banking Standard:
- p95 ≤ 200ms (Good user experience)
- p99 ≤ 1000ms (Acceptable for complex transactions)
```

**Multi-Tier SLO:**
```
Payment API:

Tier 1 (Critical - Real-time payments):
  p95 ≤ 100ms
  p99 ≤ 200ms
  
Tier 2 (Standard - ACH transfers):
  p95 ≤ 500ms
  p99 ≤ 1000ms
  
Tier 3 (Batch - End-of-day):
  p95 ≤ 5000ms
  p99 ≤ 10000ms

Different SLOs for different use cases
```

### **Throughput SLOs**

**Transaction Capacity:**
```
SLO: Support 1000 TPS (Transactions Per Second) with p95 < 200ms

Why important for banking:
- Peak load planning (salary day, month-end)
- Capacity planning
- Scalability validation

Measurement:
- Synthetic tests: Can system handle 1000 TPS?
- Real load: Current TPS with latency tracking
```

### **Correctness SLOs**

**Data Accuracy:**
```
Banking Critical: 100% correctness required

SLO: 99.9999% transaction accuracy (Six Sigma)

What this means:
- 1 error per million transactions allowed
- For 100K transactions/day = 1 error per 10 days
- Any error triggers immediate investigation

Types of correctness:
- Amount accuracy (no rounding errors)
- Account balance consistency
- Transaction atomicity (no partial updates)
```

---

## 📊 SLO Design Framework

### **Step 1: Choose SLIs**

**Questions:**
```
1. What matters to users?
   Banking: Speed, reliability, correctness
   
2. What can we measure?
   Metrics, logs, synthetic tests
   
3. What can we control?
   Our infrastructure, not third-party APIs (separate SLO)
```

**Banking SLI Selection:**
```
✅ Good SLIs:
- Transaction success rate (we control)
- API response time (we control)
- ATM availability (we control)

⚠️ Questionable SLIs:
- Payment gateway success rate (third-party)
  → Track separately, don't include in SLO
  
❌ Bad SLIs:
- User satisfaction score (subjective)
- Feature count (not user-facing)
```

### **Step 2: Set Targets**

**Approach:**
```
1. Historical Baseline
   - What are we achieving now?
   - Example: Currently 99.85% availability
   
2. User Expectations
   - What do users need?
   - Banking: Very high (99.9%+)
   
3. Cost Analysis
   - Cost of each nine
   - 99.9% vs 99.99% = 10x infrastructure cost
   
4. Competition Benchmark
   - What do others offer?
   - Industry standard for banking: 99.95%
   
5. Set Target
   - Slightly better than current
   - Meet user needs
   - Economically viable
   - Example: 99.9% → reasonable target
```

**The Nines:**
```
Availability  | Downtime/Month | Downtime/Year | Cost Multiple
--------------|----------------|---------------|---------------
99%           | 7.2 hours      | 3.65 days     | 1x (baseline)
99.9%         | 43.2 minutes   | 8.76 hours    | 3-5x
99.95%        | 21.6 minutes   | 4.38 hours    | 5-10x
99.99%        | 4.3 minutes    | 52.6 minutes  | 10-100x
99.999%       | 26 seconds     | 5.26 minutes  | 100-1000x

Banking typical: 99.9% - 99.95%
Critical systems: 99.99% (payment switches)
```

### **Step 3: Define Time Window**

**Options:**
```
Rolling Window:
- Last 30 days
- Last 7 days
- Continuous evaluation
- Better for SRE teams

Calendar Window:
- This month
- This quarter
- Easier for business reporting
- Better for SLAs

Banking typically uses: 30-day rolling window
```

### **Step 4: Exclusions**

**What to Exclude:**
```
✅ Exclude:
- Scheduled maintenance (announced 48h advance)
- User errors (400 status codes)
- External attacks (DDoS)
- Third-party service failures (clearly marked)

❌ Don't Exclude:
- Unscheduled downtime
- Our bugs
- Performance issues
- Capacity problems

Transparency principle: Only exclude what's truly outside control
```

---

## 🚨 SLO Monitoring & Alerting

### **Error Budget Alerts**

**Multi-Level Alerting:**
```
Level 1: Warning (50% budget consumed)
  Notify: Team Slack channel
  Action: Awareness, start reviewing
  
Level 2: Critical (80% budget consumed)
  Notify: Team + Manager
  Action: Freeze non-critical changes
  
Level 3: Emergency (100% budget consumed)
  Notify: All stakeholders + Exec
  Action: Full deployment freeze, war room
```

**Alert Example:**
```yaml
Alert: "Payment API Error Budget Critical"

Trigger: 
  Error budget consumed ≥ 80% in current 30-day window
  
Message:
  "⚠️ Payment API Error Budget: 80% consumed
   
   Current SLO: 99.9% (43.2 min error budget)
   Used: 34.5 minutes
   Remaining: 8.7 minutes
   
   Actions:
   - Review recent incidents
   - Freeze non-critical deployments
   - Focus on reliability improvements
   
   Dashboard: [link]
   Postmortem: [link]"
```

### **Burn Rate Alerts**

**Concept:**
> How fast are we consuming error budget?

**Fast Burn (Critical):**
```
Consuming error budget at 10x rate
Example: Using 1 day budget in 2.4 hours

Alert: Immediate
Action: Stop everything, fix now
```

**Slow Burn (Warning):**
```
Consuming at 2x rate
Example: Using 1 day budget in 12 hours

Alert: Within 1 hour
Action: Investigate, plan fix
```

---

## 💡 SLO Best Practices

### **1. Start Simple**

```
❌ Don't start with:
- 10 different SLIs
- Complex calculations
- Aggressive targets (99.99%)

✅ Start with:
- 1-2 key SLIs (availability, latency)
- Simple measurements
- Achievable targets (current baseline + 10%)

Iterate and refine over time
```

### **2. User-Centric**

```
✅ Good: "95% of login attempts succeed within 2 seconds"
   Why: User-facing, clear impact

❌ Bad: "Database CPU usage < 70%"
   Why: Internal metric, no direct user impact
   
Rule: If users don't notice, it's not an SLI
```

### **3. Realistic Targets**

```
Banking Trap: "We need 100% uptime!"

Reality:
- 100% is impossible
- Trying achieves 99.9% at 100x cost
- Better: 99.95% well-engineered system

Philosophy: "Reliability has diminishing returns"
```

### **4. Document Everything**

```
SLO Document Should Include:
- SLI definition and calculation
- Target and rationale
- Time window
- Exclusions policy
- Measurement method
- Owner team
- Review cadence (quarterly)
```

### **5. Regular Review**

```
Quarterly Review Questions:
- Is SLO still relevant?
- Too easy to achieve? (raise target)
- Always missing? (lower target or improve system)
- User expectations changed?
- Business requirements evolved?

SLOs should evolve with the service
```

---

## 🏦 Banking SLO Examples

### **Example 1: Core Banking System**

```yaml
Service: Account Management API
Owner: Core Banking Team

SLI 1: Availability
  Measurement: (Successful requests / Total requests) × 100
  Target: ≥ 99.95%
  Window: 30-day rolling
  Exclusions: Scheduled maintenance, user errors (4xx)
  
SLI 2: Latency  
  Measurement: p95 response time
  Target: ≤ 200ms
  Window: 30-day rolling
  
Error Budget: 21.6 minutes/month
  
Alert Thresholds:
  - 50% budget consumed: Warning
  - 80% budget consumed: Critical
  - 100% budget consumed: Emergency freeze
```

### **Example 2: Payment Processing**

```yaml
Service: Payment Gateway
Owner: Payments Team

SLI 1: Transaction Success Rate
  Measurement: (Approved + Declined) / (Total - System Errors) × 100
  Target: ≥ 99.99%
  Window: 7-day rolling (more stringent)
  Note: Declined by issuer = Success (correct behavior)
  
SLI 2: Processing Time
  Measurement: p99 end-to-end time
  Target: ≤ 3 seconds
  Window: 7-day rolling
  
SLI 3: Throughput
  Measurement: Sustained TPS
  Target: ≥ 1000 TPS with p95 < 500ms
  Window: Peak hours (10 AM - 4 PM)
  
Error Budget: 1 minute/week
```

### **Example 3: Mobile Banking App**

```yaml
Service: Mobile API Backend
Owner: Digital Banking Team

SLI 1: Login Success Rate
  Measurement: (Successful logins / Attempted logins) × 100
  Target: ≥ 99.9%
  Window: 30-day rolling
  Exclusions: Wrong password (user error)
  
SLI 2: API Availability
  Measurement: Synthetic test success rate
  Target: ≥ 99.95%
  Window: 30-day rolling
  Tests: Every 1 minute from 5 global locations
  
SLI 3: Session Reliability
  Measurement: Sessions completed / Sessions started
  Target: ≥ 99%
  Window: 7-day rolling
  
Time-Based SLO:
  Business hours (6 AM - 11 PM): All SLIs apply
  Night hours (11 PM - 6 AM): Relaxed to 99.5%
```

---

## 📊 Implementing in Datadog

### **SLO Feature in Datadog**

**Create SLO:**
```
1. Navigate: Service Management → SLO → New SLO

2. Choose Type:
   - Metric-based (availability, latency)
   - Monitor-based (existing monitors)
   
3. Define SLI:
   Good events: success requests
   Total events: all requests
   
4. Set Target: 99.9% over 30 days

5. Alerts:
   - Warning: 50% error budget consumed
   - Critical: 80% consumed
```

**SLO Query Examples:**
```
Availability:
  SLI = sum:http.requests{status:2xx}.as_count() / 
        sum:http.requests{*}.as_count()
  Target: ≥ 99.9%

Latency:
  SLI = p95:trace.http.request.duration{*}
  Target: ≤ 200ms
  
Success Rate:
  SLI = (sum:payments.success + sum:payments.declined) / 
        sum:payments.total
  Target: ≥ 99.99%
```

### **SLO Dashboard**

**Recommended Widgets:**
```
1. SLO Status Overview
   - Current SLO % (Query Value)
   - Error budget remaining (Query Value)
   - Color: Green/Yellow/Red
   
2. Burn Rate Graph
   - How fast consuming budget (Timeseries)
   
3. Historical Performance
   - SLO over time (Timeseries)
   - Show target line
   
4. Error Budget Trend
   - Budget remaining over time
   - Project when will exhaust
   
5. Top Contributing Errors
   - What's causing SLO misses (Top List)
```

---

## 🤔 Decision Framework

### **Khi Nào Define SLO?**

```
✅ Define SLO when:
- Service is user-facing
- Service is critical to business
- Need to balance reliability vs velocity
- Want data-driven decisions
- Multiple teams depend on service

⚠️ Maybe not needed:
- Internal tools with 5 users
- Services in alpha/beta
- Experimental features
```

### **How Many SLOs Per Service?**

```
✅ Sweet spot: 2-3 SLOs per service

Example:
1. Availability (always)
2. Latency (for sync services)
3. Throughput or Correctness (as needed)

❌ Too many: Hard to manage, dilutes focus
✅ Start with 1 (availability), add as mature
```

### **Internal vs External SLO?**

```
Pattern:
- Internal SLO: Tighter (99.95%)
- External SLA: Looser (99.9%)
- Buffer: 0.05% for safety margin

Rationale:
- Hit internal SLO = Customers happy
- Miss internal but hit SLA = Warning sign
- Miss SLA = Contract violation
```

---

## 📝 Tóm Tắt

### **Key Takeaways**

```
1. SLI = Measurement
   - What to measure (success rate, latency)
   - User-centric, actionable
   
2. SLO = Target
   - What to achieve (99.9%, <200ms)
   - Internal goal
   
3. SLA = Contract
   - What to commit (99.5%)
   - External agreement
   
4. Error Budget = Innovation Fuel
   - Balance reliability vs velocity
   - Data-driven deployment decisions
   
5. Banking Standard
   - 99.9% - 99.95% typical
   - 99.99% for critical systems
   - Time-based SLOs for peak hours
```

### **Must Remember**

```
✓ Perfect reliability is impossible
✓ Higher nines = exponentially higher cost
✓ SLOs should evolve with service
✓ User-facing metrics only
✓ Error budget enables innovation
✓ Regular review and adjustment
```

---

## ➡️ Bước Tiếp Theo

**Related Topics:**
- [12 - Monitors & Alerts](12-MONITORS-ALERTS.md) - Implement SLO alerts
- [16 - Best Practices](16-BEST-PRACTICES.md) - SLO best practices
- [21 - Banking Implementation](21-BANKING-IMPLEMENTATION.md) - Banking SLO requirements

---

**📌 Ghi Chú Của Bạn**
```
(SLO definitions, targets, calculations cho projects của bạn)








```

