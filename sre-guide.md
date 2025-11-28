# SRE Interview Ready Reckoner: Google Gemini Team

## Part 1: Core Fundamentals

---

### 1. Linux/Unix Systems

**Process Management**
- Process states: Running, Sleeping (D/S), Zombie, Stopped
- `ps aux`, `top`, `htop` — reading output matters
- `/proc` filesystem: `/proc/<pid>/fd`, `/proc/<pid>/status`
- Signals: SIGTERM (15) vs SIGKILL (9) vs SIGHUP (1)
- `strace` and `ltrace` for debugging

**Memory**
- Virtual vs Physical memory
- Page faults (major vs minor)
- OOM killer: how it selects victims (`oom_score`)
- Swap: when it's good, when it's bad
- Commands: `free -h`, `vmstat`, `/proc/meminfo`

**File Systems & Storage**
- Inodes, file descriptors, hard vs soft links
- `df -h` vs `du -sh` — why they differ
- Disk I/O: `iostat`, `iotop`
- RAID levels: 0, 1, 5, 6, 10 (trade-offs)
- LVM basics

**Networking**
- OSI model (focus on L3, L4, L7)
- TCP handshake, FIN vs RST, TIME_WAIT
- `netstat`/`ss`, `tcpdump`, `wireshark`
- DNS resolution flow (recursive, iterative)
- Load balancing: L4 vs L7, algorithms (round-robin, least-conn, consistent hashing)

---

### 2. Networking Deep Dive

**Key Concepts**
```
TCP Connection States:
LISTEN → SYN_RECEIVED → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED

Why TIME_WAIT exists: Ensures delayed packets don't corrupt new connections
Duration: 2 * MSL (typically 60 seconds)
```

**HTTP/HTTPS**
- HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC)
- Keep-alive connections
- TLS handshake (simplified): ClientHello → ServerHello → Certificate → Key Exchange → Finished

**DNS**
- Record types: A, AAAA, CNAME, MX, TXT, SRV, PTR
- TTL implications for rollbacks
- DNS as a load balancer (GSLB)

**Debugging Network Issues**
```bash
# Is the port open?
nc -zv host 443
telnet host 443

# Trace the path
traceroute host
mtr host

# Packet capture
tcpdump -i eth0 port 443 -w capture.pcap

# Check connections
ss -tuln
ss -s  # summary
```

---

### 3. Distributed Systems

**CAP Theorem**
- Consistency, Availability, Partition Tolerance — pick 2
- In reality: partitions happen, so it's C vs A during partitions
- CP systems: Strong consistency, may reject writes (e.g., traditional RDBMS with sync replication)
- AP systems: Always available, eventually consistent (e.g., Cassandra, DynamoDB)

**Consensus**
- Paxos: Complex but foundational
- Raft: Understandable consensus (leader election, log replication)
- Why consensus is hard: Byzantine failures, network partitions, split-brain

**Consistency Models**
- Strong consistency: Read sees latest write
- Eventual consistency: Given enough time, all reads return latest write
- Causal consistency: Respects causality (if A→B, everyone sees A before B)

**Key Patterns**
- Sharding: Horizontal partitioning, consistent hashing
- Replication: Leader-follower, multi-leader, leaderless
- Quorum: W + R > N for strong consistency

---

### 4. Observability (The Three Pillars)

**Metrics**
- USE Method (Utilization, Saturation, Errors) — for resources
- RED Method (Rate, Errors, Duration) — for services
- Golden Signals: Latency, Traffic, Errors, Saturation

```
Example SLI definitions:
- Availability: successful requests / total requests
- Latency: % of requests < 200ms
- Throughput: requests per second
```

**Logging**
- Structured logging (JSON) vs unstructured
- Log levels: DEBUG, INFO, WARN, ERROR, FATAL
- Centralized logging: ELK stack, Google Cloud Logging
- What to log: Request ID, timestamp, user context, error details

**Tracing**
- Distributed tracing: Trace ID propagation across services
- Spans: Units of work within a trace
- Tools: Jaeger, Zipkin, Google Cloud Trace

---

### 5. SLI/SLO/SLA Framework

**Definitions**
- **SLI (Service Level Indicator)**: Quantitative measure (e.g., request latency)
- **SLO (Service Level Objective)**: Target value for SLI (e.g., p99 latency < 200ms)
- **SLA (Service Level Agreement)**: Contract with consequences (e.g., refunds if SLO missed)

**Error Budgets**
```
If SLO = 99.9% availability:
- Error budget = 0.1% = 43.2 minutes/month downtime allowed
- Use it for: deployments, experiments, maintenance
- When exhausted: freeze changes, focus on reliability
```

**Setting Good SLOs**
- Based on user experience, not system metrics
- Achievable but ambitious
- Measured over rolling windows (7 days, 30 days)

---

### 6. Incident Management

**Severity Levels (typical)**
| Level | Impact | Response |
|-------|--------|----------|
| P1/Sev1 | Major outage, revenue impact | All hands, exec visibility |
| P2/Sev2 | Partial outage, degraded service | On-call + backup |
| P3/Sev3 | Minor impact, workaround exists | On-call during business hours |
| P4/Sev4 | Low impact, cosmetic | Normal ticket queue |

**Incident Response Flow**
1. Detect (monitoring/alerts)
2. Triage (severity, who's impacted)
3. Mitigate (stop the bleeding)
4. Resolve (fix root cause)
5. Postmortem (blameless, action items)

**Postmortem Template**
- Summary: What happened?
- Impact: Who was affected? For how long?
- Timeline: Minute-by-minute
- Root cause: 5 Whys analysis
- Action items: Preventive measures with owners and deadlines

---

### 7. Kubernetes & Containers

**Container Basics**
- Namespaces: Process isolation (PID, network, mount, user)
- Cgroups: Resource limits (CPU, memory)
- Docker layers: Each instruction creates a layer

**Kubernetes Architecture**
```
Control Plane:
- API Server: All communication goes through here
- etcd: Distributed key-value store (cluster state)
- Scheduler: Assigns pods to nodes
- Controller Manager: Maintains desired state

Worker Nodes:
- kubelet: Node agent
- kube-proxy: Network rules
- Container runtime: Docker, containerd
```

**Key Resources**
- Pod: Smallest deployable unit
- Deployment: Manages ReplicaSets, rolling updates
- Service: Stable network endpoint (ClusterIP, NodePort, LoadBalancer)
- ConfigMap/Secret: Configuration management
- HPA: Horizontal Pod Autoscaler

**Debugging Kubernetes**
```bash
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs <pod> -c <container> --previous
kubectl exec -it <pod> -- /bin/sh
kubectl top pods
```

---

### 8. Cloud & Infrastructure

**Google Cloud Specifics (relevant for this role)**
- Compute: GCE, GKE, Cloud Run
- Storage: Cloud Storage, Persistent Disk, Filestore
- Networking: VPC, Cloud Load Balancing, Cloud CDN
- Data: BigQuery, Spanner, Bigtable
- ML: Vertex AI, TPUs (crucial for Gemini)

**Infrastructure as Code**
- Terraform: Declarative, provider-agnostic
- State management: Remote state, locking
- Modules: Reusable components

**Capacity Planning**
- Load testing: Identify breaking points
- Headroom: Typically 30-50% for spikes
- Scaling strategies: Vertical vs horizontal
- Cost optimization: Right-sizing, spot/preemptible instances

---

### 9. Automation & Toil Reduction

**What is Toil?**
- Manual, repetitive, automatable work
- Scales linearly with service size
- No enduring value

**Toil Examples**
- Manual deployments
- Ticket-based access requests
- Manually scaling services
- Copy-paste configuration changes

**Automation Priorities**
1. High-frequency, low-complexity → Automate first
2. Incident response automation
3. Self-healing systems
4. Capacity management

---

### 10. ML/LLM Systems (Gemini-Specific)

**ML Infrastructure Challenges**
- Training vs Inference: Different resource profiles
- GPU/TPU management: Expensive, needs efficient scheduling
- Model serving: Low latency, high throughput
- A/B testing: Model versions, traffic splitting

**LLM-Specific Concerns**
- Token limits and context windows
- Inference latency (time to first token, tokens per second)
- Memory requirements (model size, KV cache)
- Batching strategies for throughput

**Reliability for ML**
- Model rollback capabilities
- Graceful degradation (fallback models)
- Monitoring: Prediction quality, drift detection
- Feature store reliability

---

## Part 2: Common Interview Questions & Answers

---

### System Design Questions

**Q1: Design a globally distributed rate limiter**

```
Requirements clarification:
- Scale: 1M+ requests/second globally
- Latency: < 10ms added latency
- Accuracy: Soft limits acceptable

High-level design:
1. Local rate limiting (in-memory) for speed
2. Distributed state sync for accuracy
3. Token bucket or sliding window algorithm

Components:
- Edge nodes with local counters
- Redis Cluster for distributed state
- Async sync between local and global state

Trade-offs:
- Strong consistency = higher latency
- Eventual consistency = might slightly exceed limits
- Hybrid: local enforcement + periodic sync

For Google scale:
- Use consistent hashing to partition rate limit keys
- Regional clusters with cross-region replication
- Graceful degradation: if sync fails, use local limits
```

**Q2: Design a monitoring and alerting system**

```
Requirements:
- 10,000+ services
- Millions of metrics
- Sub-minute detection

Architecture:
1. Collection layer: Agents push metrics to collectors
2. Ingestion: Kafka for buffering
3. Storage: Time-series DB (InfluxDB, Prometheus, Google's Monarch)
4. Query engine: For dashboards and alerting
5. Alerting: Rules engine + notification routing

Key decisions:
- Push vs Pull: Push scales better, Pull easier to manage
- Cardinality: High-cardinality labels are expensive
- Retention: Hot/warm/cold storage tiers
- Deduplication: Alert grouping and suppression

Alerting best practices:
- Alert on symptoms, not causes
- Multiple severity levels
- Runbook links in every alert
- Escalation paths
```

**Q3: How would you handle a service with 99.99% availability requirement?**

```
99.99% = 52 minutes downtime/year = 4.3 minutes/month

Strategies:
1. Redundancy
   - Multi-zone deployment minimum
   - Multi-region for critical paths
   - No single points of failure

2. Deployment safety
   - Canary deployments (1% → 10% → 50% → 100%)
   - Automatic rollback on SLO breach
   - Feature flags for quick disable

3. Failure handling
   - Circuit breakers
   - Bulkheads (isolation)
   - Graceful degradation
   - Retry with exponential backoff + jitter

4. Operational excellence
   - Automated incident detection
   - Runbooks for common issues
   - Regular disaster recovery testing
   - Chaos engineering

5. Change management
   - Limited deployment windows
   - Mandatory rollback plans
   - Post-deployment validation
```

---

### Troubleshooting Scenarios

**Q4: A service suddenly has high latency. How do you debug?**

```
Systematic approach:

1. Scope the problem (2 min)
   - Which endpoints? All or specific?
   - Which users? All or subset?
   - When did it start? Correlate with deployments/changes

2. Check the obvious (5 min)
   - Recent deployments? Rollback candidate
   - Dependent services healthy?
   - Resource exhaustion? (CPU, memory, disk, connections)

3. Narrow down (10 min)
   - Where is time spent? (Traces)
   - Network? (tcpdump, ping times)
   - Database? (Slow query logs)
   - External dependencies? (Timeouts)

4. Common culprits:
   - Garbage collection pauses
   - Connection pool exhaustion
   - DNS resolution issues
   - Downstream service degradation
   - Lock contention
   - Cold cache after restart

5. Mitigation options:
   - Rollback if deployment-related
   - Scale out if capacity-related
   - Shed load if overloaded
   - Failover if dependency failure
```

**Q5: You're paged at 3 AM. Production is down. Walk me through your response.**

```
First 5 minutes:
1. Acknowledge alert
2. Quick assessment: What's the user impact?
3. Check monitoring dashboards
4. Look for obvious causes (recent changes, dependent outages)

Next 10 minutes:
5. If obvious fix: Apply it
6. If not: Call for help (don't be a hero at 3 AM)
7. Start incident channel/bridge
8. Communicate status to stakeholders

During incident:
- One incident commander
- Separate roles: debugging, communication, documentation
- Regular updates (every 15-30 min)
- Prioritize mitigation over root cause

After mitigation:
- Monitor for recurrence
- Document timeline while fresh
- Schedule postmortem
- Get some sleep

Key principle: Mitigate first, debug later
```

**Q6: Memory usage keeps growing until OOM. How do you investigate?**

```
Investigation steps:

1. Confirm the pattern
   - Is it a true leak or expected growth?
   - How long until OOM? (Hours? Days?)

2. Identify the process
   - `top`, `ps aux --sort=-%mem`
   - `/proc/<pid>/status` for detailed memory breakdown

3. For the application:
   - Enable heap profiling
   - Take heap dumps at intervals
   - Compare object counts over time

4. Common leak sources:
   - Caches without eviction
   - Connection pools not releasing
   - Event listeners not unregistered
   - Circular references (in languages with ref counting)
   - Thread-local storage accumulation

5. Tools by language:
   - Java: jmap, jstat, VisualVM, async-profiler
   - Go: pprof
   - Python: tracemalloc, memory_profiler
   - Node.js: --inspect, clinic.js

6. Quick mitigations:
   - Restart on schedule (temporary)
   - Reduce instance lifetime
   - Add memory limits with graceful restart
```

---

### Coding Questions

**Q7: Implement a simple rate limiter (Token Bucket)**

```python
import time
from threading import Lock

class TokenBucket:
    def __init__(self, capacity: int, refill_rate: float):
        """
        capacity: max tokens
        refill_rate: tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity
        self.last_refill = time.time()
        self.lock = Lock()
    
    def _refill(self):
        now = time.time()
        elapsed = now - self.last_refill
        new_tokens = elapsed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + new_tokens)
        self.last_refill = now
    
    def allow(self, tokens: int = 1) -> bool:
        with self.lock:
            self._refill()
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False

# Usage
limiter = TokenBucket(capacity=10, refill_rate=1)  # 10 burst, 1/sec sustained
if limiter.allow():
    process_request()
else:
    return_429_too_many_requests()
```

**Q8: Implement a circuit breaker**

```python
import time
from enum import Enum
from threading import Lock

class State(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing fast
    HALF_OPEN = "half_open"  # Testing recovery

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=30, 
                 half_open_requests=3):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.half_open_requests = half_open_requests
        
        self.state = State.CLOSED
        self.failures = 0
        self.successes = 0
        self.last_failure_time = None
        self.lock = Lock()
    
    def can_execute(self) -> bool:
        with self.lock:
            if self.state == State.CLOSED:
                return True
            
            if self.state == State.OPEN:
                if time.time() - self.last_failure_time > self.recovery_timeout:
                    self.state = State.HALF_OPEN
                    self.successes = 0
                    return True
                return False
            
            # HALF_OPEN: allow limited requests
            return True
    
    def record_success(self):
        with self.lock:
            if self.state == State.HALF_OPEN:
                self.successes += 1
                if self.successes >= self.half_open_requests:
                    self.state = State.CLOSED
                    self.failures = 0
            else:
                self.failures = 0
    
    def record_failure(self):
        with self.lock:
            self.failures += 1
            self.last_failure_time = time.time()
            
            if self.state == State.HALF_OPEN:
                self.state = State.OPEN
            elif self.failures >= self.failure_threshold:
                self.state = State.OPEN

# Usage
cb = CircuitBreaker()

def call_external_service():
    if not cb.can_execute():
        raise CircuitOpenError("Circuit is open")
    
    try:
        result = external_service.call()
        cb.record_success()
        return result
    except Exception as e:
        cb.record_failure()
        raise
```

**Q9: Parse logs and find the top 10 IPs by request count**

```python
import heapq
from collections import defaultdict
import re

def top_k_ips(log_file: str, k: int = 10) -> list:
    """
    Log format: 192.168.1.1 - - [timestamp] "GET /path HTTP/1.1" 200 1234
    """
    ip_pattern = re.compile(r'^(\d+\.\d+\.\d+\.\d+)')
    ip_counts = defaultdict(int)
    
    with open(log_file, 'r') as f:
        for line in f:
            match = ip_pattern.match(line)
            if match:
                ip_counts[match.group(1)] += 1
    
    # Use heap for efficiency with large datasets
    return heapq.nlargest(k, ip_counts.items(), key=lambda x: x[1])

# For very large files, use streaming + approximate counting (Count-Min Sketch)
```

---

### Behavioral Questions

**Q10: Tell me about a time you handled a major outage.**

```
Use STAR format:

Situation: "At [Company], our payment service went down during Black Friday 
peak traffic, affecting 50,000+ transactions per minute."

Task: "As the on-call SRE, I needed to restore service quickly while 
coordinating with multiple teams."

Action:
- "First 5 min: Assessed impact, found database connection pool exhausted"
- "10 min: Scaled database connections, but underlying issue was query timeouts"
- "20 min: Identified a new feature causing N+1 queries, disabled via feature flag"
- "30 min: Service recovered, monitored for stability"
- "Throughout: Kept stakeholders informed every 15 minutes"

Result: "Restored service in 35 minutes. Lost revenue ~$X, but prevented 
multi-hour outage. Postmortem led to mandatory load testing for new features."

Key points to emphasize:
- Calm under pressure
- Systematic debugging
- Communication
- Prevention (postmortem)
```

**Q11: Describe a time you eliminated significant toil.**

```
Situation: "Our team spent 10+ hours weekly on manual certificate renewals 
across 200+ services."

Task: "Automate certificate management to eliminate manual work."

Action:
- "Analyzed the current process, documented all edge cases"
- "Built automation using cert-manager in Kubernetes"
- "Implemented monitoring for expiring certs (30-day warning)"
- "Created runbook for exceptions"
- "Rolled out incrementally: staging → low-risk prod → all services"

Result:
- "Reduced manual work from 10 hours/week to ~30 min/month"
- "Zero certificate-related outages since implementation"
- "Team could focus on higher-value projects"

Key points:
- Measured before and after
- Risk-managed rollout
- Didn't just automate—added monitoring
```

**Q12: How do you prioritize when everything is urgent?**

```
Framework I use:

1. Impact assessment
   - User-facing? Revenue impact?
   - How many users affected?
   - Is there a workaround?

2. Urgency assessment
   - Is it getting worse?
   - Is there a deadline (SLA, compliance)?
   - What's the blast radius if delayed?

3. Effort estimation
   - Quick win vs. deep investigation
   - Can it be delegated?

4. Decision matrix:
   | High Impact + High Urgency | DO FIRST |
   | High Impact + Low Urgency  | SCHEDULE |
   | Low Impact + High Urgency  | DELEGATE |
   | Low Impact + Low Urgency   | DROP/BACKLOG |

5. Communication
   - Be transparent about trade-offs
   - Get buy-in from stakeholders
   - Document decisions

Example: "When three P2s came in simultaneously, I triaged based on user 
impact, delegated the one with a workaround, and handled the 
revenue-impacting one myself."
```

---

## Part 3: Google-Specific Preparation

---

### Google's SRE Philosophy

**Key Principles**
1. **Embrace risk**: 100% reliability is wrong target
2. **Error budgets**: Align dev and SRE incentives
3. **Reduce toil**: Automate or eliminate
4. **Monitoring**: Symptoms over causes
5. **Release engineering**: Consistent, repeatable
6. **Simplicity**: Essential complexity only

**SRE vs DevOps**
- SRE is "class implements DevOps"
- SRE is more prescriptive (error budgets, SLOs)
- SRE has explicit engineering time allocation (50% cap on toil)

### Gemini Team Specifics

**What makes ML SRE different:**
- Hardware failures more common (GPUs/TPUs)
- Training jobs are long-running and expensive
- Model serving has unique scaling patterns
- A/B testing complexity
- Data pipeline reliability matters

**Questions to expect:**
- How would you ensure Gemini API availability during traffic spikes?
- How do you handle model rollback when quality degrades?
- What metrics would you monitor for an LLM service?

### Interview Process

**Typical rounds:**
1. Recruiter screen
2. Technical phone screen (coding or troubleshooting)
3. Onsite (virtual or in-person):
   - 2x Coding (algorithms, SRE-specific)
   - 1x System design
   - 1x Troubleshooting
   - 1x Behavioral (Googleyness & Leadership)

**Googleyness traits:**
- Comfortable with ambiguity
- Collaborative
- Bias to action
- Humble, willing to learn

---

## Part 4: Quick Reference Cheatsheet

### Commands for Interview

```bash
# System performance
uptime                    # Load averages
top -bn1 | head -20       # Quick CPU/memory snapshot
vmstat 1 5                # Virtual memory stats
iostat -x 1 5             # Disk I/O
sar -n DEV 1 5            # Network throughput

# Process debugging
ps aux --sort=-%mem       # Memory hogs
ps aux --sort=-%cpu       # CPU hogs
lsof -p <pid>             # Open files/connections
strace -p <pid> -f        # System calls

# Network
ss -tuln                  # Listening ports
ss -s                     # Socket summary
netstat -an | grep ESTABLISHED | wc -l
tcpdump -i eth0 port 443 -c 100

# Disk
df -h                     # Filesystem usage
du -sh /* | sort -h       # Space by directory
lsof +D /path             # Files open in directory

# Logs
journalctl -u service -f  # Follow service logs
dmesg -T | tail           # Kernel messages
grep -r "ERROR" /var/log/ --include="*.log"
```

### Key Formulas

```
Availability = Uptime / (Uptime + Downtime)
99.9% = 8.76 hours/year = 43.8 min/month
99.99% = 52.6 min/year = 4.38 min/month
99.999% = 5.26 min/year = 26 sec/month

Error Budget = 1 - SLO
If SLO = 99.9%, Error Budget = 0.1%

MTBF = Total Uptime / Number of Failures
MTTR = Total Downtime / Number of Failures
Availability = MTBF / (MTBF + MTTR)

Little's Law: L = λW
L = average items in system
λ = arrival rate
W = average time in system
```

### Incident Severity Quick Guide

| Severity | User Impact | Response Time | Example |
|----------|-------------|---------------|---------|
| P1 | Complete outage | Immediate | API down |
| P2 | Major degradation | 15 min | 50% errors |
| P3 | Minor impact | 4 hours | Slow responses |
| P4 | Minimal | Next day | Dashboard bug |

---

## Part 5: Study Resources

### Books
- "Site Reliability Engineering" (Google's SRE Book) — FREE online
- "The Site Reliability Workbook" — FREE online
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Systems Performance" by Brendan Gregg

### Practice
- LeetCode (medium difficulty, focus on: hash maps, heaps, graphs)
- "Grokking the System Design Interview"
- Google's SRE book case studies

### Blogs & Talks
- Google SRE Blog
- Netflix Tech Blog
- Brendan Gregg's Blog (performance)
- SREcon talks (YouTube)

---

*Last updated: 2024. Good luck with your interview!*
