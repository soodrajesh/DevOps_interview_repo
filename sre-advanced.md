# SRE Advanced Guide
## Troubleshooting Scenarios, Monitoring & Disaster Recovery

---

# 1. TROUBLESHOOTING SCENARIOS

---

## 1.1 Application Layer Issues

### Scenario: Memory Leak in Java Application

```
Symptoms:
- Memory usage grows over time
- Eventual OutOfMemoryError
- GC pauses increasing
- Application becomes unresponsive

Investigation Flow:
```

```bash
# Step 1: Confirm memory growth pattern
# Check container/pod memory
kubectl top pods -n production
docker stats <container>

# Check JVM heap
jstat -gc <pid> 1000  # GC stats every second

# Output interpretation:
# S0C, S1C: Survivor space capacity
# EC: Eden space capacity
# OC: Old gen capacity
# OU: Old gen used ← Watch this grow

# Step 2: Check GC behavior
jstat -gcutil <pid> 1000

# If Old Gen (O) keeps growing and GC can't reclaim → leak

# Step 3: Get heap dump
# Option A: JVM flag (dumps on OOM)
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/

# Option B: Manual dump
jmap -dump:format=b,file=/tmp/heap.hprof <pid>

# Option C: In Kubernetes
kubectl exec -it <pod> -- jmap -dump:format=b,file=/tmp/heap.hprof 1
kubectl cp <pod>:/tmp/heap.hprof ./heap.hprof

# Step 4: Analyze heap dump
# Use Eclipse MAT, VisualVM, or YourKit
# Look for:
# - Dominator tree (largest objects)
# - Leak suspects report
# - Histogram (object counts)

# Step 5: Common leak sources
# - Unbounded caches
# - Static collections growing
# - Listeners not unregistered
# - ThreadLocal not cleaned
# - Connection pools not releasing
```

```java
// Common leak patterns to look for:

// 1. Static collection growing
public class LeakyCache {
    // BAD: Never cleared
    private static Map<String, Object> cache = new HashMap<>();
    
    public void addToCache(String key, Object value) {
        cache.put(key, value);  // Grows forever
    }
}

// FIX: Use bounded cache
private static Map<String, Object> cache = new LinkedHashMap<>() {
    @Override
    protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > MAX_SIZE;
    }
};

// Or use Caffeine/Guava cache with TTL
Cache<String, Object> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .build();

// 2. Listener not removed
public void subscribe() {
    eventBus.register(this);  // Registered
    // But never: eventBus.unregister(this);
}

// 3. ThreadLocal leak
private static ThreadLocal<Connection> connHolder = new ThreadLocal<>();

public void process() {
    connHolder.set(getConnection());
    // Missing: finally { connHolder.remove(); }
}
```

### Scenario: Deadlock Detection

```
Symptoms:
- Application hangs
- Requests time out
- CPU low but threads blocked
- No errors in logs

Investigation:
```

```bash
# Step 1: Get thread dump
# Java
jstack <pid> > threads.txt
# Or send SIGQUIT
kill -3 <pid>  # Dumps to stdout

# In Kubernetes
kubectl exec <pod> -- jstack 1

# Step 2: Look for BLOCKED threads
grep -A 20 "BLOCKED" threads.txt

# Deadlock pattern in thread dump:
# "Thread-1" - waiting to lock <0x000000076bf62208>
#   which is held by "Thread-2"
# "Thread-2" - waiting to lock <0x000000076bf62108>
#   which is held by "Thread-1"

# JVM will actually detect and report:
# "Found one Java-level deadlock:"

# Step 3: Analyze lock ordering
# Thread 1: Lock A → trying Lock B
# Thread 2: Lock B → trying Lock A
# Solution: Always acquire locks in same order
```

```python
# Python deadlock detection
import threading
import sys

def dump_threads():
    """Dump all thread stacks"""
    for thread_id, frame in sys._current_frames().items():
        print(f"\nThread {thread_id}:")
        traceback.print_stack(frame)

# In production, expose via endpoint or signal handler
import signal
signal.signal(signal.SIGUSR1, lambda s, f: dump_threads())
# Then: kill -USR1 <pid>
```

### Scenario: Connection Pool Exhaustion

```
Symptoms:
- "Cannot acquire connection" errors
- Requests queuing/timing out
- Database shows many connections
- Some connections idle for long time

Investigation:
```

```bash
# Step 1: Check current connections
# PostgreSQL
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
SELECT * FROM pg_stat_activity WHERE state != 'idle' ORDER BY query_start;

# MySQL
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads_connected';

# Step 2: Check connection pool metrics
# HikariCP (Java) - expose via JMX or Prometheus
hikaricp_connections_active
hikaricp_connections_idle
hikaricp_connections_pending
hikaricp_connections_timeout_total

# Step 3: Find connection leak
# Enable leak detection (HikariCP)
spring.datasource.hikari.leak-detection-threshold=30000  # 30 sec

# Will log:
# "Connection leak detection triggered, stack trace follows"
# Shows where connection was acquired but not returned

# Step 4: Common causes
# - Transaction not committed/rolled back
# - Connection not closed in finally block
# - Long-running queries holding connections
# - N+1 query problem using many connections
```

```java
// BAD: Connection leak
public void fetchData() {
    Connection conn = dataSource.getConnection();
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery("SELECT * FROM users");
    // If exception here, connection never closed!
    processResults(rs);
    conn.close();
}

// GOOD: Try-with-resources
public void fetchData() {
    try (Connection conn = dataSource.getConnection();
         Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
        processResults(rs);
    }  // Auto-closed even on exception
}
```

---

## 1.2 Infrastructure Layer Issues

### Scenario: Kubernetes Pod Evictions

```
Symptoms:
- Pods restarting unexpectedly
- "Evicted" pod status
- OOMKilled in pod events
- Node pressure events

Investigation:
```

```bash
# Step 1: Check pod status
kubectl get pods -o wide
kubectl describe pod <evicted-pod>

# Look for:
# Status: Failed
# Reason: Evicted
# Message: "Pod was evicted due to memory pressure"

# Step 2: Check node conditions
kubectl describe node <node-name>

# Conditions:
# MemoryPressure   True/False
# DiskPressure     True/False
# PIDPressure      True/False

# Step 3: Check node resource usage
kubectl top nodes
kubectl top pods --all-namespaces --sort-by=memory

# Step 4: Check for OOMKilled
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.containerStatuses[*].lastState.terminated.reason}{"\n"}{end}'

# Step 5: Check kubelet logs on node
journalctl -u kubelet | grep -i evict
journalctl -u kubelet | grep -i oom

# Step 6: Eviction thresholds (kubelet config)
# --eviction-hard=memory.available<100Mi
# --eviction-hard=nodefs.available<10%
# --eviction-soft=memory.available<300Mi

# Prevention:
# 1. Set appropriate resource requests/limits
resources:
  requests:
    memory: "256Mi"  # Scheduler uses this
    cpu: "100m"
  limits:
    memory: "512Mi"  # OOM killed if exceeded
    cpu: "500m"      # Throttled if exceeded

# 2. Use PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2  # Or use maxUnavailable
  selector:
    matchLabels:
      app: myapp

# 3. Set priority classes for critical pods
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical production workloads"
```

### Scenario: Network Connectivity Issues in Kubernetes

```
Symptoms:
- Service-to-service calls failing
- DNS resolution failing
- Intermittent connection timeouts
- "Connection refused" or "No route to host"

Investigation:
```

```bash
# Step 1: Test from affected pod
kubectl exec -it <pod> -- /bin/sh

# DNS resolution
nslookup kubernetes.default
nslookup <service-name>
nslookup <service-name>.<namespace>.svc.cluster.local

# If DNS fails, check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# Step 2: Test connectivity
# To service
curl -v http://<service-name>:<port>/health
wget -O- --timeout=5 http://<service-name>:<port>/health

# To pod directly
curl -v http://<pod-ip>:<container-port>/health

# Step 3: Check service and endpoints
kubectl get svc <service-name> -o wide
kubectl get endpoints <service-name>

# If endpoints empty:
# - Check selector matches pod labels
# - Check pod is Ready
# - Check targetPort matches container port

# Step 4: Check network policies
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy-name>

# Test if network policy blocking:
# Temporarily delete policy and test
# Or check if pod labels match policy selectors

# Step 5: Debug with netshoot
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash
# Inside netshoot:
nslookup service-name
curl -v http://service-name:port
tcpdump -i any host <pod-ip>
mtr <destination>

# Step 6: Check kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Check iptables rules (on node)
iptables -t nat -L KUBE-SERVICES

# Step 7: CNI issues (Calico, Cilium, etc.)
# Calico
kubectl get pods -n kube-system -l k8s-app=calico-node
calicoctl node status

# Cilium
kubectl get pods -n kube-system -l k8s-app=cilium
cilium status
```

### Scenario: Disk I/O Performance Issues

```
Symptoms:
- High I/O wait (wa in top)
- Slow database queries
- Application timeouts
- High disk latency metrics

Investigation:
```

```bash
# Step 1: Identify I/O bottleneck
iostat -xz 1 5

# Key metrics:
# %util    - Device utilization (>80% concerning)
# await    - Average wait time (ms)
# r_await  - Read wait time
# w_await  - Write wait time
# avgqu-sz - Average queue length

# High await + low %util = slow storage
# High %util + high avgqu-sz = saturated device

# Step 2: Find processes doing I/O
iotop -oP
# Or
pidstat -d 1 5

# Step 3: Check for specific file I/O
# Using fatrace (file access trace)
fatrace -f W  # Write operations only

# Using strace on specific process
strace -p <pid> -e trace=read,write,fsync -T

# Step 4: Check filesystem issues
df -h              # Space
df -i              # Inodes
dmesg | grep -i "i/o error"
smartctl -a /dev/sda  # Disk health

# Step 5: Check for noisy neighbors (cloud)
# Burst credits (AWS EBS)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name BurstBalance \
  --dimensions Name=VolumeId,Value=vol-xxx \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Average

# Step 6: Common fixes
# - Upgrade to provisioned IOPS (cloud)
# - Add read replicas for database
# - Implement caching layer
# - Optimize queries to reduce I/O
# - Use faster storage class (SSD vs HDD)
# - Spread I/O across multiple volumes
```

---

## 1.3 Database Issues

### Scenario: Database Replication Lag

```
Symptoms:
- Read replicas showing stale data
- Replication lag increasing
- Replica falling behind during peak hours

Investigation:
```

```bash
# PostgreSQL
# On primary
SELECT * FROM pg_stat_replication;
# Check: sent_lsn vs write_lsn vs replay_lsn

# Calculate lag in bytes
SELECT 
    client_addr,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) / 1024 / 1024 AS lag_mb
FROM pg_stat_replication;

# On replica
SELECT 
    now() - pg_last_xact_replay_timestamp() AS replication_lag;

# MySQL
SHOW SLAVE STATUS\G
# Check: Seconds_Behind_Master

# If lag increasing:
# 1. Check replica resources (CPU, I/O, memory)
# 2. Check for long-running queries on replica
SHOW PROCESSLIST;

# 3. Check for large transactions on primary
# Single large transaction = replica must apply serially

# 4. Parallel replication (MySQL 5.7+)
slave_parallel_workers = 4
slave_parallel_type = LOGICAL_CLOCK

# 5. Skip problematic statement (DANGEROUS)
# SET GLOBAL sql_slave_skip_counter = 1;
# START SLAVE;
```

### Scenario: Lock Contention

```
Symptoms:
- Queries waiting/blocking
- "Lock wait timeout exceeded"
- Deadlocks in error log
- Throughput drops suddenly

Investigation:
```

```sql
-- PostgreSQL: Find blocking queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity 
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks 
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity 
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- MySQL: Find blocking transactions
SELECT 
    r.trx_id waiting_trx_id,
    r.trx_mysql_thread_id waiting_thread,
    r.trx_query waiting_query,
    b.trx_id blocking_trx_id,
    b.trx_mysql_thread_id blocking_thread,
    b.trx_query blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b 
    ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r 
    ON r.trx_id = w.requesting_trx_id;

-- Kill blocking query (carefully!)
-- PostgreSQL
SELECT pg_terminate_backend(<pid>);

-- MySQL
KILL <thread_id>;
```

### Scenario: Query Performance Degradation

```
Symptoms:
- Queries suddenly slower
- CPU/I/O spikes
- Execution plans changed

Investigation:
```

```sql
-- PostgreSQL: Analyze slow query
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE customer_id = 123;

-- Look for:
-- Seq Scan on large tables (missing index?)
-- High "actual rows" vs "estimated rows" (stale stats?)
-- Nested Loop with high row counts

-- Check if statistics are stale
SELECT 
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- Update statistics
ANALYZE orders;

-- Check index usage
SELECT 
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE relname = 'orders';

-- Check for index bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) as table_size,
    pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) as index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;

-- MySQL: Check query execution
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE customer_id = 123;

-- Check if using index
SHOW INDEX FROM orders;

-- Force index hint (testing)
SELECT * FROM orders FORCE INDEX (idx_customer_id) WHERE customer_id = 123;
```

---

## 1.4 CI/CD Pipeline Issues

### Scenario: Flaky Tests

```
Symptoms:
- Tests pass/fail randomly
- Same code, different results
- Tests pass locally, fail in CI

Investigation:
```

```bash
# Step 1: Identify flaky tests
# Run tests multiple times
for i in {1..10}; do
    pytest tests/ --tb=no -q 2>&1 | tee run_$i.txt
done

# Compare results
diff run_1.txt run_2.txt

# Step 2: Common causes and fixes

# 1. Test order dependency
# Run in random order to detect
pytest tests/ --random-order

# 2. Shared state between tests
# Bad: Modifying global/class state
# Fix: Use fixtures with proper scope, clean up in teardown

# 3. Timing issues
# Bad: sleep(1); assert something_happened
# Fix: Use polling with timeout
def wait_for(condition, timeout=10):
    start = time.time()
    while time.time() - start < timeout:
        if condition():
            return True
        time.sleep(0.1)
    return False

# 4. Resource contention
# Bad: All tests use same database
# Fix: Use separate database per test or transactions

@pytest.fixture
def db_session():
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    yield session
    session.close()
    transaction.rollback()
    connection.close()

# 5. External dependencies
# Bad: Tests call real APIs
# Fix: Mock external calls
@patch('app.services.external_api.call')
def test_something(mock_api):
    mock_api.return_value = {'status': 'ok'}
    result = my_function()
    assert result == expected

# 6. Environment differences
# Bad: Relies on local file paths, env vars
# Fix: Use consistent CI environment, containerize tests
```

### Scenario: Docker Build Failures

```
Symptoms:
- "No space left on device"
- "Cannot connect to Docker daemon"
- Layer caching not working
- Build takes too long

Investigation:
```

```bash
# Step 1: Check disk space
df -h
docker system df

# Clean up
docker system prune -a --volumes
docker builder prune

# Step 2: Check Docker daemon
systemctl status docker
journalctl -u docker -n 100

# Step 3: Layer caching issues
# Bad: COPY everything early (busts cache)
COPY . /app
RUN pip install -r requirements.txt

# Good: Copy dependencies first
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app

# Step 4: Multi-stage builds for smaller images
# Build stage
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/main

# Runtime stage
FROM alpine:3.18
RUN apk --no-cache add ca-certificates
COPY --from=builder /app/main /main
ENTRYPOINT ["/main"]

# Step 5: BuildKit for faster builds
DOCKER_BUILDKIT=1 docker build .

# Or in daemon.json
{ "features": { "buildkit": true } }

# Step 6: Use .dockerignore
.git
node_modules
*.log
.env
__pycache__
```

---

## 1.5 Cloud Infrastructure Issues

### Scenario: AWS Lambda Cold Starts

```
Symptoms:
- First request very slow (seconds)
- Latency spikes after idle period
- p99 latency much higher than p50

Investigation and Fixes:
```

```python
# Step 1: Identify cold starts in logs
# Look for "Init Duration" in CloudWatch

# Step 2: Reduce package size
# Before: 50MB deployment package
# After: Use Lambda layers for dependencies

# Step 3: Increase memory (also increases CPU)
# More memory = faster init
# Test with Lambda Power Tuning tool

# Step 4: Initialize outside handler
import boto3

# GOOD: Initialized once per container
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('my-table')

def handler(event, context):
    # Reuses existing connection
    return table.get_item(Key={'id': event['id']})

# BAD: Initialized every invocation
def handler(event, context):
    dynamodb = boto3.resource('dynamodb')  # Slow!
    table = dynamodb.Table('my-table')
    return table.get_item(Key={'id': event['id']})

# Step 5: Use Provisioned Concurrency
aws lambda put-provisioned-concurrency-config \
    --function-name my-function \
    --qualifier prod \
    --provisioned-concurrent-executions 10

# Step 6: Keep warm with scheduled events
# CloudWatch Events rule: rate(5 minutes)
# Invoke with {"warm": true}

def handler(event, context):
    if event.get('warm'):
        return {'statusCode': 200, 'body': 'warm'}
    # Normal processing
```

### Scenario: Auto Scaling Not Working

```
Symptoms:
- High CPU but no new instances
- Scaling too slow
- Scaling too aggressively (thrashing)

Investigation:
```

```bash
# Step 1: Check ASG configuration
aws autoscaling describe-auto-scaling-groups \
    --auto-scaling-group-names my-asg

# Check:
# - MinSize, MaxSize, DesiredCapacity
# - Cooldown period
# - Health check type and grace period

# Step 2: Check scaling policies
aws autoscaling describe-policies \
    --auto-scaling-group-name my-asg

# Step 3: Check scaling activities
aws autoscaling describe-scaling-activities \
    --auto-scaling-group-name my-asg \
    --max-items 20

# Look for:
# - "Failed" activities and why
# - Timestamps (is cooldown blocking?)

# Step 4: Check CloudWatch alarms
aws cloudwatch describe-alarms \
    --alarm-names my-asg-cpu-high

# Step 5: Common issues

# Issue: Cooldown too long
# Default 300s, might need shorter
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name my-asg \
    --default-cooldown 60

# Issue: Health check grace period too short
# Instances killed before app starts
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name my-asg \
    --health-check-grace-period 300

# Issue: Wrong metric
# CPU might not reflect actual load
# Use custom metrics (queue depth, request count)

# Step 6: Target Tracking (simpler, recommended)
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name my-asg \
    --policy-name cpu-target-tracking \
    --policy-type TargetTrackingScaling \
    --target-tracking-configuration '{
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ASGAverageCPUUtilization"
        },
        "TargetValue": 70.0
    }'
```

---

# 2. PROMETHEUS & GRAFANA

---

## 2.1 Prometheus Configuration

### Complete Prometheus Setup

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    region: 'us-east-1'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Rule files
rule_files:
  - /etc/prometheus/rules/*.yml

# Scrape configurations
scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node1:9100', 'node2:9100', 'node3:9100']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):\d+'
        replacement: '${1}'

  # Kubernetes API server
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

  # Kubernetes nodes
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)

  # Kubernetes pods (auto-discovery)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Use custom port if specified
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}
      # Use custom path if specified
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      # Add pod labels
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod

  # Service monitors (for services)
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    metrics_path: /metrics
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: ${1}:${2}

  # Application with custom metrics
  - job_name: 'myapp'
    static_configs:
      - targets: ['myapp:8080']
    metrics_path: /metrics
    scrape_interval: 10s
```

### Recording Rules

```yaml
# /etc/prometheus/rules/recording_rules.yml
groups:
  - name: node_recording_rules
    interval: 30s
    rules:
      # CPU utilization (averaged over 5m)
      - record: instance:node_cpu_utilization:rate5m
        expr: |
          100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      # Memory utilization
      - record: instance:node_memory_utilization:ratio
        expr: |
          1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

      # Disk utilization
      - record: instance:node_disk_utilization:ratio
        expr: |
          1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})

  - name: http_recording_rules
    rules:
      # Request rate by service
      - record: service:http_requests:rate5m
        expr: |
          sum by (service) (rate(http_requests_total[5m]))

      # Error rate by service
      - record: service:http_errors:rate5m
        expr: |
          sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))

      # Error percentage
      - record: service:http_error_percentage:rate5m
        expr: |
          service:http_errors:rate5m / service:http_requests:rate5m * 100

      # Latency percentiles (pre-calculated for dashboards)
      - record: service:http_request_duration_seconds:p50
        expr: |
          histogram_quantile(0.50, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))

      - record: service:http_request_duration_seconds:p95
        expr: |
          histogram_quantile(0.95, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))

      - record: service:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99, sum by (service, le) (rate(http_request_duration_seconds_bucket[5m])))
```

### Alerting Rules

```yaml
# /etc/prometheus/rules/alerting_rules.yml
groups:
  - name: infrastructure_alerts
    rules:
      # High CPU
      - alert: HighCPUUsage
        expr: instance:node_cpu_utilization:rate5m > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}% on {{ $labels.instance }}"
          runbook_url: "https://wiki.example.com/runbooks/high-cpu"

      - alert: HighCPUUsageCritical
        expr: instance:node_cpu_utilization:rate5m > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}% on {{ $labels.instance }}"

      # High Memory
      - alert: HighMemoryUsage
        expr: instance:node_memory_utilization:ratio > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

      # Disk Space
      - alert: DiskSpaceLow
        expr: instance:node_disk_utilization:ratio > 0.80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}"

      - alert: DiskSpaceCritical
        expr: instance:node_disk_utilization:ratio > 0.90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space critical on {{ $labels.instance }}"

      # Node down
      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.instance }} is down"
          description: "Node exporter on {{ $labels.instance }} is not responding"

  - name: application_alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: service:http_error_percentage:rate5m > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate for {{ $labels.service }}"
          description: "Error rate is {{ $value | printf \"%.1f\" }}% for {{ $labels.service }}"

      - alert: HighErrorRateCritical
        expr: service:http_error_percentage:rate5m > 10
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Critical error rate for {{ $labels.service }}"

      # High latency
      - alert: HighLatency
        expr: service:http_request_duration_seconds:p99 > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency for {{ $labels.service }}"
          description: "p99 latency is {{ $value | printf \"%.2f\" }}s for {{ $labels.service }}"

      # Low traffic (potential issue)
      - alert: LowTraffic
        expr: service:http_requests:rate5m < 10 and service:http_requests:rate5m > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Unusually low traffic for {{ $labels.service }}"

      # No traffic (likely down)
      - alert: NoTraffic
        expr: service:http_requests:rate5m == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "No traffic for {{ $labels.service }}"

  - name: kubernetes_alerts
    rules:
      # Pod not ready
      - alert: PodNotReady
        expr: |
          sum by (namespace, pod) (kube_pod_status_phase{phase=~"Pending|Unknown"}) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"

      # Pod restarting
      - alert: PodRestarting
        expr: |
          increase(kube_pod_container_status_restarts_total[1h]) > 5
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} restarting frequently"
          description: "Pod has restarted {{ $value }} times in the last hour"

      # Deployment replicas mismatch
      - alert: DeploymentReplicasMismatch
        expr: |
          kube_deployment_spec_replicas != kube_deployment_status_available_replicas
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} replica mismatch"

      # PVC almost full
      - alert: PVCAlmostFull
        expr: |
          kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PVC {{ $labels.persistentvolumeclaim }} almost full"

  - name: slo_alerts
    rules:
      # Error budget burn rate (fast burn)
      - alert: ErrorBudgetFastBurn
        expr: |
          (
            1 - (sum(rate(http_requests_total{status!~"5.."}[1h])) / sum(rate(http_requests_total[1h])))
          ) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error budget burning fast"
          description: "At current rate, 30-day error budget will be exhausted in 5 hours"

      # Error budget burn rate (slow burn)
      - alert: ErrorBudgetSlowBurn
        expr: |
          (
            1 - (sum(rate(http_requests_total{status!~"5.."}[6h])) / sum(rate(http_requests_total[6h])))
          ) > (1 * 0.001)
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Error budget burning steadily"
          description: "At current rate, 30-day error budget will be exhausted"
```

## 2.2 Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'password'

  slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'
  
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  # Default receiver
  receiver: 'slack-notifications'
  
  # Group alerts by these labels
  group_by: ['alertname', 'cluster', 'service']
  
  # Wait before sending first notification
  group_wait: 30s
  
  # Wait before sending updated notification
  group_interval: 5m
  
  # Wait before resending
  repeat_interval: 4h

  # Child routes
  routes:
    # Critical alerts go to PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true  # Also send to Slack
    
    - match:
        severity: critical
      receiver: 'slack-critical'
    
    # Warning alerts to Slack
    - match:
        severity: warning
      receiver: 'slack-warnings'
      group_wait: 1m
      repeat_interval: 12h
    
    # Database alerts to DBA team
    - match_re:
        alertname: ^(PostgreSQL|MySQL|Database).*
      receiver: 'slack-dba'
    
    # Specific team routing
    - match:
        team: platform
      receiver: 'slack-platform-team'

# Inhibition rules
inhibit_rules:
  # If critical, suppress warning
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
  
  # If node down, suppress all alerts from that node
  - source_match:
      alertname: 'NodeDown'
    target_match_re:
      alertname: '.+'
    equal: ['instance']

# Receivers
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '🚨 {{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
        actions:
          - type: button
            text: 'Runbook'
            url: '{{ .CommonAnnotations.runbook_url }}'
          - type: button
            text: 'Dashboard'
            url: 'https://grafana.example.com/d/xxx'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warnings'
        send_resolved: true

  - name: 'slack-dba'
    slack_configs:
      - channel: '#dba-alerts'
        send_resolved: true

  - name: 'slack-platform-team'
    slack_configs:
      - channel: '#platform-alerts'
        send_resolved: true

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'your-pagerduty-service-key'
        severity: critical
        description: '{{ .CommonAnnotations.summary }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'

  - name: 'email-oncall'
    email_configs:
      - to: 'oncall@example.com'
        send_resolved: true
```

## 2.3 Grafana Dashboards

### Dashboard JSON (Infrastructure Overview)

```json
{
  "dashboard": {
    "title": "Infrastructure Overview",
    "tags": ["infrastructure", "overview"],
    "timezone": "browser",
    "refresh": "30s",
    "panels": [
      {
        "title": "CPU Usage by Node",
        "type": "timeseries",
        "gridPos": {"x": 0, "y": 0, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "instance:node_cpu_utilization:rate5m",
            "legendFormat": "{{ instance }}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"value": 0, "color": "green"},
                {"value": 70, "color": "yellow"},
                {"value": 90, "color": "red"}
              ]
            }
          }
        }
      },
      {
        "title": "Memory Usage by Node",
        "type": "timeseries",
        "gridPos": {"x": 12, "y": 0, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "instance:node_memory_utilization:ratio * 100",
            "legendFormat": "{{ instance }}"
          }
        ]
      },
      {
        "title": "Request Rate",
        "type": "stat",
        "gridPos": {"x": 0, "y": 8, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "sum(service:http_requests:rate5m)",
            "legendFormat": "req/s"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps"
          }
        }
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "gridPos": {"x": 6, "y": 8, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "sum(service:http_error_percentage:rate5m)",
            "legendFormat": "error %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"value": 0, "color": "green"},
                {"value": 1, "color": "yellow"},
                {"value": 5, "color": "red"}
              ]
            }
          }
        }
      },
      {
        "title": "P99 Latency",
        "type": "stat",
        "gridPos": {"x": 12, "y": 8, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "max(service:http_request_duration_seconds:p99)",
            "legendFormat": "p99"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s"
          }
        }
      }
    ]
  }
}
```

### PromQL Quick Reference

```promql
# Rate of increase (for counters)
rate(http_requests_total[5m])

# Instant rate (last two samples)
irate(http_requests_total[5m])

# Increase (total increase over time range)
increase(http_requests_total[1h])

# Percentiles from histogram
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Aggregation operators
sum(rate(http_requests_total[5m])) by (service)
avg(node_cpu_seconds_total) by (instance)
max(memory_usage_bytes) by (pod)
count(up == 1) by (job)
topk(5, rate(http_requests_total[5m]))

# Comparison operators
http_requests_total > 100
up == 1
rate(errors_total[5m]) > 0.1

# Label matching
http_requests_total{status="500"}
http_requests_total{status=~"5.."}  # Regex match
http_requests_total{status!="200"}  # Not equal

# Offset (look back in time)
rate(http_requests_total[5m] offset 1h)

# Absent (alert when metric missing)
absent(up{job="myapp"})

# Changes (how many times value changed)
changes(process_start_time_seconds[1h])

# Predict linear (forecasting)
predict_linear(node_filesystem_avail_bytes[1h], 4*3600)  # Predict 4h ahead

# Common patterns
# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# Availability
1 - (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))

# Saturation (queue depth)
avg_over_time(http_request_queue_length[5m])

# Pod memory percentage
container_memory_usage_bytes / container_spec_memory_limit_bytes * 100
```

---

# 3. DISASTER RECOVERY

---

## 3.1 DR Fundamentals

### RPO and RTO

```
RPO (Recovery Point Objective):
- Maximum acceptable data loss
- "How much data can we afford to lose?"
- Determines backup frequency
- Example: RPO of 1 hour = backup every hour

RTO (Recovery Time Objective):
- Maximum acceptable downtime
- "How quickly must we recover?"
- Determines DR architecture
- Example: RTO of 4 hours = must be back online in 4 hours

                    Disaster
                       │
                       ▼
    ◄────── RPO ──────►│◄────── RTO ──────►
                       │
    Last good          │                Service
    backup             │                restored
                       │
    Data lost in       │        Downtime duration
    this window        │
```

### DR Tiers

```
Tier 1: Backup & Restore (RTO: 24+ hours, RPO: 24 hours)
┌─────────────────┐         ┌─────────────────┐
│   Production    │ backup  │    S3 / GCS     │
│   Environment   │ ─────── │    (another     │
│                 │         │    region)      │
└─────────────────┘         └─────────────────┘
- Cheapest
- Longest recovery time
- Good for: Non-critical systems

Tier 2: Pilot Light (RTO: 4-8 hours, RPO: minutes)
┌─────────────────┐         ┌─────────────────┐
│   Production    │  sync   │    DR Region    │
│   (us-east-1)   │ ─────── │   (us-west-2)   │
│                 │         │                 │
│ ┌─────────────┐ │         │ ┌─────────────┐ │
│ │ App (active)│ │         │ │ App (off)   │ │
│ └─────────────┘ │         │ └─────────────┘ │
│ ┌─────────────┐ │         │ ┌─────────────┐ │
│ │ DB (primary)│ │ ──────► │ │ DB (replica)│ │
│ └─────────────┘ │  async  │ └─────────────┘ │
└─────────────────┘  replic └─────────────────┘
- Core infra running
- App servers stopped (launch on failover)
- Database replicated

Tier 3: Warm Standby (RTO: 1-4 hours, RPO: minutes)
┌─────────────────┐         ┌─────────────────┐
│   Production    │         │    DR Region    │
│   (us-east-1)   │         │   (us-west-2)   │
│                 │         │                 │
│ ┌─────────────┐ │         │ ┌─────────────┐ │
│ │ App (3x)    │ │         │ │ App (1x)    │ │
│ └─────────────┘ │         │ └─────────────┘ │
│ ┌─────────────┐ │         │ ┌─────────────┐ │
│ │ DB (primary)│ │ ──────► │ │ DB (replica)│ │
│ └─────────────┘ │         │ └─────────────┘ │
└─────────────────┘         └─────────────────┘
- Scaled-down copy running
- Scale up on failover
- Database replicated

Tier 4: Hot Standby / Active-Active (RTO: minutes, RPO: near-zero)
┌─────────────────┐         ┌─────────────────┐
│   Region A      │◄───────►│    Region B     │
│   (us-east-1)   │  GSLB   │   (us-west-2)   │
│                 │         │                 │
│ ┌─────────────┐ │         │ ┌─────────────┐ │
│ │ App (3x)    │ │         │ │ App (3x)    │ │
│ └─────────────┘ │         │ └─────────────┘ │
│ ┌─────────────┐ │         │ ┌─────────────┐ │
│ │ DB (multi-  │ │◄───────►│ │ DB (multi-  │ │
│ │   master)   │ │  sync   │ │   master)   │ │
│ └─────────────┘ │         │ └─────────────┘ │
└─────────────────┘         └─────────────────┘
- Both regions active
- Global load balancer routes traffic
- Most complex, most expensive
```

## 3.2 Backup Strategies

### Database Backup Configuration

```bash
# PostgreSQL - Continuous Archiving (WAL)
# postgresql.conf
archive_mode = on
archive_command = 'aws s3 cp %p s3://backups/wal/%f'
wal_level = replica

# Base backup script
#!/bin/bash
set -euo pipefail

BACKUP_NAME="basebackup_$(date +%Y%m%d_%H%M%S)"
BACKUP_DIR="/tmp/${BACKUP_NAME}"
S3_BUCKET="s3://my-backups/postgresql"

# Create base backup
pg_basebackup -D ${BACKUP_DIR} -Ft -z -P

# Upload to S3
aws s3 cp ${BACKUP_DIR}/base.tar.gz ${S3_BUCKET}/${BACKUP_NAME}/
aws s3 cp ${BACKUP_DIR}/pg_wal.tar.gz ${S3_BUCKET}/${BACKUP_NAME}/

# Cleanup
rm -rf ${BACKUP_DIR}

# Retention (keep 30 days)
aws s3 ls ${S3_BUCKET}/ | while read -r line; do
    createDate=$(echo $line | awk '{print $1}')
    if [[ $(date -d "$createDate" +%s) -lt $(date -d "30 days ago" +%s) ]]; then
        fileName=$(echo $line | awk '{print $4}')
        aws s3 rm ${S3_BUCKET}/${fileName} --recursive
    fi
done
```

```yaml
# Kubernetes backup with Velero
# Install Velero
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.5.0 \
    --bucket velero-backups \
    --backup-location-config region=us-east-1 \
    --snapshot-location-config region=us-east-1 \
    --secret-file ./credentials-velero

# Schedule backups
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  template:
    includedNamespaces:
      - production
      - staging
    excludedResources:
      - events
      - pods
    snapshotVolumes: true
    ttl: 720h  # 30 days

# Restore
velero restore create --from-backup daily-backup-20240115
```

### Application State Backup

```python
#!/usr/bin/env python3
"""
Backup critical application data to S3 with versioning and encryption.
"""

import boto3
import json
import gzip
from datetime import datetime
from pathlib import Path
import hashlib

class BackupManager:
    def __init__(self, bucket: str, prefix: str):
        self.s3 = boto3.client('s3')
        self.bucket = bucket
        self.prefix = prefix
    
    def backup_data(self, data: dict, backup_type: str) -> str:
        """Backup data with compression and integrity check."""
        
        timestamp = datetime.utcnow().strftime('%Y%m%d_%H%M%S')
        key = f"{self.prefix}/{backup_type}/{timestamp}.json.gz"
        
        # Serialize and compress
        json_data = json.dumps(data, default=str)
        compressed = gzip.compress(json_data.encode('utf-8'))
        
        # Calculate checksum
        checksum = hashlib.sha256(compressed).hexdigest()
        
        # Upload with metadata
        self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=compressed,
            ContentType='application/gzip',
            Metadata={
                'checksum': checksum,
                'backup_type': backup_type,
                'timestamp': timestamp
            },
            ServerSideEncryption='aws:kms',
            StorageClass='STANDARD_IA'  # Cheaper for backups
        )
        
        return key
    
    def restore_data(self, key: str) -> dict:
        """Restore and verify data integrity."""
        
        response = self.s3.get_object(
            Bucket=self.bucket,
            Key=key
        )
        
        compressed = response['Body'].read()
        
        # Verify checksum
        expected_checksum = response['Metadata'].get('checksum')
        actual_checksum = hashlib.sha256(compressed).hexdigest()
        
        if expected_checksum and expected_checksum != actual_checksum:
            raise ValueError(f"Checksum mismatch for {key}")
        
        # Decompress and deserialize
        json_data = gzip.decompress(compressed).decode('utf-8')
        return json.loads(json_data)
    
    def list_backups(self, backup_type: str, limit: int = 10) -> list:
        """List available backups."""
        
        prefix = f"{self.prefix}/{backup_type}/"
        
        response = self.s3.list_objects_v2(
            Bucket=self.bucket,
            Prefix=prefix,
            MaxKeys=limit
        )
        
        backups = []
        for obj in response.get('Contents', []):
            backups.append({
                'key': obj['Key'],
                'size': obj['Size'],
                'last_modified': obj['LastModified']
            })
        
        return sorted(backups, key=lambda x: x['last_modified'], reverse=True)


# Usage
backup_mgr = BackupManager(
    bucket='my-backups',
    prefix='app-data'
)

# Backup
user_data = fetch_users_from_db()
backup_mgr.backup_data(user_data, 'users')

# Restore
latest = backup_mgr.list_backups('users', limit=1)[0]
restored_data = backup_mgr.restore_data(latest['key'])
```

## 3.3 Failover Automation

### DNS-Based Failover

```hcl
# Terraform - Route53 health check and failover
resource "aws_route53_health_check" "primary" {
  fqdn              = "primary.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  tags = {
    Name = "primary-health-check"
  }
}

resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "A"

  # Primary record
  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "www_secondary" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "A"

  # Secondary record (no health check - always available)
  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "secondary"

  alias {
    name                   = aws_lb.secondary.dns_name
    zone_id                = aws_lb.secondary.zone_id
    evaluate_target_health = true
  }
}
```

### Database Failover Script

```python
#!/usr/bin/env python3
"""
Automated database failover for PostgreSQL.
"""

import boto3
import psycopg2
import time
import logging
from dataclasses import dataclass

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@dataclass
class DatabaseConfig:
    primary_host: str
    replica_host: str
    port: int
    database: str
    user: str
    password: str


class DatabaseFailover:
    def __init__(self, config: DatabaseConfig):
        self.config = config
        self.rds = boto3.client('rds')
    
    def check_primary_health(self) -> bool:
        """Check if primary database is healthy."""
        try:
            conn = psycopg2.connect(
                host=self.config.primary_host,
                port=self.config.port,
                database=self.config.database,
                user=self.config.user,
                password=self.config.password,
                connect_timeout=5
            )
            
            with conn.cursor() as cur:
                cur.execute("SELECT 1")
                cur.fetchone()
            
            conn.close()
            return True
            
        except Exception as e:
            logger.error(f"Primary health check failed: {e}")
            return False
    
    def check_replica_lag(self) -> int:
        """Get replication lag in seconds."""
        try:
            conn = psycopg2.connect(
                host=self.config.replica_host,
                port=self.config.port,
                database=self.config.database,
                user=self.config.user,
                password=self.config.password,
                connect_timeout=5
            )
            
            with conn.cursor() as cur:
                cur.execute("""
                    SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))::INT
                """)
                lag = cur.fetchone()[0]
            
            conn.close()
            return lag or 0
            
        except Exception as e:
            logger.error(f"Failed to check replica lag: {e}")
            return -1
    
    def promote_replica(self) -> bool:
        """Promote replica to primary (AWS RDS)."""
        try:
            # For RDS
            response = self.rds.promote_read_replica(
                DBInstanceIdentifier=self.config.replica_host.split('.')[0]
            )
            logger.info(f"Replica promotion initiated: {response}")
            return True
            
        except Exception as e:
            logger.error(f"Failed to promote replica: {e}")
            return False
    
    def execute_failover(self, max_lag_seconds: int = 60) -> bool:
        """Execute failover with safety checks."""
        
        logger.info("Starting failover process...")
        
        # Step 1: Verify primary is actually down
        for i in range(3):
            if self.check_primary_health():
                logger.info("Primary is healthy, aborting failover")
                return False
            time.sleep(5)
        
        logger.info("Primary confirmed unhealthy")
        
        # Step 2: Check replica lag
        lag = self.check_replica_lag()
        if lag > max_lag_seconds:
            logger.error(f"Replica lag ({lag}s) exceeds threshold ({max_lag_seconds}s)")
            logger.error("Manual intervention required")
            return False
        
        logger.info(f"Replica lag acceptable: {lag}s")
        
        # Step 3: Promote replica
        if not self.promote_replica():
            return False
        
        # Step 4: Wait for promotion
        logger.info("Waiting for replica promotion...")
        for i in range(60):
            time.sleep(10)
            # Check if replica is now accepting writes
            try:
                conn = psycopg2.connect(
                    host=self.config.replica_host,
                    port=self.config.port,
                    database=self.config.database,
                    user=self.config.user,
                    password=self.config.password
                )
                with conn.cursor() as cur:
                    cur.execute("SELECT pg_is_in_recovery()")
                    is_recovery = cur.fetchone()[0]
                conn.close()
                
                if not is_recovery:
                    logger.info("Replica successfully promoted!")
                    return True
                    
            except Exception as e:
                logger.warning(f"Waiting... ({e})")
        
        logger.error("Failover timeout")
        return False


# Automated monitoring and failover
def monitor_and_failover(config: DatabaseConfig, check_interval: int = 30):
    """Continuous monitoring with automatic failover."""
    
    failover = DatabaseFailover(config)
    consecutive_failures = 0
    failover_threshold = 3
    
    while True:
        if failover.check_primary_health():
            consecutive_failures = 0
            logger.debug("Primary healthy")
        else:
            consecutive_failures += 1
            logger.warning(f"Primary unhealthy ({consecutive_failures}/{failover_threshold})")
            
            if consecutive_failures >= failover_threshold:
                logger.critical("Initiating automatic failover!")
                
                if failover.execute_failover():
                    logger.info("Failover completed successfully")
                    # Update config to point to new primary
                    # Send notifications
                    break
                else:
                    logger.error("Failover failed, manual intervention required")
                    break
        
        time.sleep(check_interval)
```

## 3.4 Multi-Region Architecture

### Active-Passive Multi-Region Setup

```hcl
# Terraform - Multi-region infrastructure

# Primary region
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

# DR region
provider "aws" {
  alias  = "dr"
  region = "us-west-2"
}

# Global Accelerator for fast failover
resource "aws_globalaccelerator_accelerator" "main" {
  name            = "main-accelerator"
  ip_address_type = "IPV4"
  enabled         = true
}

resource "aws_globalaccelerator_listener" "main" {
  accelerator_arn = aws_globalaccelerator_accelerator.main.id
  protocol        = "TCP"

  port_range {
    from_port = 443
    to_port   = 443
  }
}

resource "aws_globalaccelerator_endpoint_group" "primary" {
  listener_arn = aws_globalaccelerator_listener.main.id
  
  endpoint_group_region = "us-east-1"
  
  endpoint_configuration {
    endpoint_id = aws_lb.primary.arn
    weight      = 100
  }
  
  health_check_path             = "/health"
  health_check_interval_seconds = 10
  threshold_count               = 3
}

resource "aws_globalaccelerator_endpoint_group" "dr" {
  listener_arn = aws_globalaccelerator_listener.main.id
  
  endpoint_group_region = "us-west-2"
  
  endpoint_configuration {
    endpoint_id = aws_lb.dr.arn
    weight      = 0  # No traffic until failover
  }
  
  health_check_path             = "/health"
  health_check_interval_seconds = 10
  threshold_count               = 3
}

# Cross-region database replication
resource "aws_db_instance" "primary" {
  provider = aws.primary
  
  identifier     = "mydb-primary"
  engine         = "postgres"
  engine_version = "14"
  instance_class = "db.r5.xlarge"
  
  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  
  # Enable replication
  storage_encrypted = true
}

resource "aws_db_instance" "replica" {
  provider = aws.dr
  
  identifier          = "mydb-replica"
  replicate_source_db = aws_db_instance.primary.arn
  instance_class      = "db.r5.xlarge"
  
  # Can be promoted to standalone
  storage_encrypted = true
}

# S3 cross-region replication
resource "aws_s3_bucket" "primary" {
  provider = aws.primary
  bucket   = "myapp-assets-primary"
}

resource "aws_s3_bucket" "replica" {
  provider = aws.dr
  bucket   = "myapp-assets-replica"
}

resource "aws_s3_bucket_replication_configuration" "replication" {
  provider = aws.primary
  bucket   = aws_s3_bucket.primary.id
  role     = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.replica.arn
      storage_class = "STANDARD"
    }
  }
}
```

### DR Runbook Template

```markdown
# Disaster Recovery Runbook

## Overview
- **Service**: MyApp Production
- **Primary Region**: us-east-1
- **DR Region**: us-west-2
- **RTO**: 4 hours
- **RPO**: 1 hour

## Pre-requisites
- [ ] AWS CLI configured with appropriate credentials
- [ ] Access to AWS Console for both regions
- [ ] PagerDuty/Slack access for communications
- [ ] Database credentials available

## Decision Criteria for Failover
Initiate DR if ANY of the following:
1. Primary region completely unavailable for >30 minutes
2. Data corruption in primary requiring restore
3. AWS declares regional outage
4. Security incident requiring isolation

## Failover Procedure

### Phase 1: Assessment (15 minutes)
1. [ ] Confirm outage is not localized
   ```bash
   aws ec2 describe-availability-zones --region us-east-1
   aws health describe-events --region us-east-1
   ```
2. [ ] Check AWS Health Dashboard
3. [ ] Notify stakeholders via Slack #incident channel
4. [ ] Page DR team lead

### Phase 2: Pre-Failover Checks (15 minutes)
1. [ ] Check database replication lag
   ```bash
   aws rds describe-db-instances --db-instance-identifier mydb-replica \
       --query 'DBInstances[0].StatusInfos'
   ```
2. [ ] Verify DR infrastructure is healthy
   ```bash
   kubectl --context dr-cluster get nodes
   kubectl --context dr-cluster get pods -n production
   ```
3. [ ] Document current replica lag for RPO calculation

### Phase 3: Database Failover (30 minutes)
1. [ ] Stop writes to primary (if accessible)
   ```sql
   -- Run on primary if accessible
   ALTER SYSTEM SET default_transaction_read_only = on;
   SELECT pg_reload_conf();
   ```
2. [ ] Promote read replica
   ```bash
   aws rds promote-read-replica --db-instance-identifier mydb-replica
   ```
3. [ ] Wait for promotion (monitor in console)
4. [ ] Verify new primary accepts writes
   ```sql
   SELECT pg_is_in_recovery();  -- Should return false
   ```

### Phase 4: Application Failover (30 minutes)
1. [ ] Update application config to use new database
   ```bash
   kubectl --context dr-cluster set env deployment/myapp \
       DATABASE_HOST=mydb-replica.xxxxx.us-west-2.rds.amazonaws.com
   ```
2. [ ] Scale up DR application
   ```bash
   kubectl --context dr-cluster scale deployment/myapp --replicas=10
   ```
3. [ ] Verify application health
   ```bash
   kubectl --context dr-cluster get pods -n production
   curl https://dr.example.com/health
   ```

### Phase 5: DNS/Traffic Failover (15 minutes)
1. [ ] Update Global Accelerator weights
   ```bash
   aws globalaccelerator update-endpoint-group \
       --endpoint-group-arn arn:aws:globalaccelerator::xxx:accelerator/xxx/listener/xxx/endpoint-group/primary \
       --endpoint-configurations "EndpointId=arn:aws:elasticloadbalancing:us-east-1:xxx:loadbalancer/app/xxx,Weight=0"
   
   aws globalaccelerator update-endpoint-group \
       --endpoint-group-arn arn:aws:globalaccelerator::xxx:accelerator/xxx/listener/xxx/endpoint-group/dr \
       --endpoint-configurations "EndpointId=arn:aws:elasticloadbalancing:us-west-2:xxx:loadbalancer/app/xxx,Weight=100"
   ```
2. [ ] Verify traffic flowing to DR
3. [ ] Monitor error rates

### Phase 6: Validation (30 minutes)
1. [ ] Run smoke tests
   ```bash
   ./scripts/smoke-tests.sh --env dr
   ```
2. [ ] Verify critical user journeys
3. [ ] Check monitoring dashboards
4. [ ] Confirm alerting is working

### Phase 7: Communication
1. [ ] Update status page
2. [ ] Send stakeholder update
3. [ ] Document timeline and decisions

## Failback Procedure
(To be executed after primary region recovery)

1. Assess primary region health
2. Sync data from DR to primary
3. Set up replication DR → Primary
4. Gradually shift traffic back
5. Re-establish Primary → DR replication

## Contacts
| Role | Name | Phone | Slack |
|------|------|-------|-------|
| DR Lead | John Smith | +1-xxx | @john |
| DBA | Jane Doe | +1-xxx | @jane |
| Platform | Bob Wilson | +1-xxx | @bob |
```

## 3.5 Testing DR

### Chaos Engineering for DR

```yaml
# Chaos Mesh - Kubernetes chaos engineering
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: simulate-region-partition
spec:
  action: partition
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      "app": "myapp"
  direction: both
  target:
    selector:
      namespaces:
        - production
      labelSelectors:
        "tier": "database"
    mode: all
  duration: "5m"

---
# Pod failure
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-failure
spec:
  action: pod-failure
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      "app": "myapp"
  duration: "60s"
```

```python
#!/usr/bin/env python3
"""
DR Testing Framework
"""

import boto3
import requests
import time
from datetime import datetime


class DRTest:
    def __init__(self, primary_url: str, dr_url: str):
        self.primary_url = primary_url
        self.dr_url = dr_url
        self.results = []
    
    def test_failover_time(self) -> dict:
        """Measure actual failover time."""
        
        result = {
            'test': 'failover_time',
            'start_time': datetime.utcnow().isoformat(),
            'steps': []
        }
        
        # Record start
        start_time = time.time()
        
        # Simulate primary failure (in real test, actually break something)
        result['steps'].append({
            'action': 'primary_failure_simulated',
            'timestamp': time.time() - start_time
        })
        
        # Poll DR until healthy
        dr_healthy = False
        while not dr_healthy and (time.time() - start_time) < 3600:
            try:
                response = requests.get(f"{self.dr_url}/health", timeout=5)
                if response.status_code == 200:
                    dr_healthy = True
                    result['steps'].append({
                        'action': 'dr_healthy',
                        'timestamp': time.time() - start_time
                    })
            except:
                pass
            time.sleep(5)
        
        result['total_time_seconds'] = time.time() - start_time
        result['success'] = dr_healthy
        
        self.results.append(result)
        return result
    
    def test_data_integrity(self) -> dict:
        """Verify data integrity after failover."""
        
        result = {
            'test': 'data_integrity',
            'checks': []
        }
        
        # Write test data to primary before failover
        test_id = f"dr_test_{int(time.time())}"
        
        # After failover, verify data exists in DR
        # Implementation depends on your data layer
        
        return result
    
    def test_rpo(self) -> dict:
        """Measure actual RPO (data loss)."""
        
        # Write continuous data to primary
        # Trigger failover
        # Check what data made it to DR
        # Calculate gap
        
        pass
    
    def generate_report(self) -> str:
        """Generate DR test report."""
        
        report = f"""
# DR Test Report
Generated: {datetime.utcnow().isoformat()}

## Summary
- Tests Run: {len(self.results)}
- Passed: {sum(1 for r in self.results if r.get('success'))}
- Failed: {sum(1 for r in self.results if not r.get('success'))}

## Detailed Results
"""
        for result in self.results:
            report += f"\n### {result['test']}\n"
            report += f"- Success: {result.get('success')}\n"
            report += f"- Duration: {result.get('total_time_seconds', 'N/A')}s\n"
        
        return report
```

### DR Test Checklist

```markdown
## Quarterly DR Test Checklist

### Pre-Test (1 week before)
- [ ] Schedule maintenance window
- [ ] Notify stakeholders
- [ ] Review and update runbooks
- [ ] Verify backups are current
- [ ] Check DR infrastructure capacity
- [ ] Prepare rollback plan

### Test Execution
- [ ] Take baseline metrics
- [ ] Document start time
- [ ] Execute failover procedure
- [ ] Measure RTO (time to recover)
- [ ] Verify application functionality
- [ ] Verify data integrity
- [ ] Measure RPO (data loss)
- [ ] Test critical business processes
- [ ] Document any issues

### Post-Test
- [ ] Execute failback
- [ ] Verify normal operations
- [ ] Calculate actual RTO/RPO vs targets
- [ ] Document lessons learned
- [ ] Update runbooks with findings
- [ ] Create action items for gaps
- [ ] Present results to stakeholders

### Metrics to Record
| Metric | Target | Actual | Pass/Fail |
|--------|--------|--------|-----------|
| RTO | 4 hours | | |
| RPO | 1 hour | | |
| Data integrity | 100% | | |
| Feature parity | 100% | | |
| Failback time | 2 hours | | |
```

---

# 4. QUICK REFERENCE CARDS

---

## Troubleshooting Decision Tree

```
Issue Reported
     │
     ├── Is it affecting all users?
     │      │
     │      ├── Yes → Check global: DNS, LB, CDN
     │      │
     │      └── No → Check specific: Region, network path, user segment
     │
     ├── When did it start?
     │      │
     │      ├── Sudden → Check: Deploy, config change, dependency
     │      │
     │      └── Gradual → Check: Resource exhaustion, leak, traffic growth
     │
     ├── What's the error?
     │      │
     │      ├── 5xx → Check: App logs, dependencies, resources
     │      │
     │      ├── Timeout → Check: Network, DB, downstream services
     │      │
     │      └── Connection refused → Check: Service running, port, firewall
     │
     └── Recent changes?
            │
            ├── Yes → Rollback and verify
            │
            └── No → Deep investigation needed
```

## Key Prometheus Alerts

| Alert | Query | Threshold |
|-------|-------|-----------|
| High Error Rate | `rate(http_errors[5m])/rate(http_total[5m])` | > 5% |
| High Latency | `histogram_quantile(0.99, rate(duration_bucket[5m]))` | > 1s |
| Low Traffic | `rate(http_total[5m])` | < 10 req/s |
| Pod Restarts | `increase(restarts_total[1h])` | > 5 |
| CPU High | `avg(cpu_usage)` | > 80% |
| Memory High | `memory_used/memory_total` | > 85% |
| Disk Low | `disk_free/disk_total` | < 20% |
| Replication Lag | `replication_lag_seconds` | > 60s |

## DR Quick Reference

| Tier | RTO | RPO | Cost | Complexity |
|------|-----|-----|------|------------|
| Backup/Restore | 24h+ | 24h | $ | Low |
| Pilot Light | 4-8h | Minutes | $ | Medium |
| Warm Standby | 1-4h | Minutes | $$ | Medium |
| Hot/Active-Active | Minutes | Near-zero | $$ | High |

---

*This completes the advanced SRE guide covering troubleshooting scenarios, monitoring configuration, and disaster recovery patterns.*