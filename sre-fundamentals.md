# SRE/DevOps Fundamentals & Interview FAQ
## Complete Reference for Senior Engineers

---

# 1. NETWORKING

---

## 1.1 OSI Model - Practical Understanding

```
Layer 7 - Application   HTTP, HTTPS, DNS, SMTP, gRPC
                        "What protocol is the app using?"
                        
Layer 4 - Transport     TCP, UDP
                        "How is data delivered reliably?"
                        
Layer 3 - Network       IP, ICMP, routing
                        "How do packets find their destination?"
                        
Layer 2 - Data Link     Ethernet, MAC addresses, switches
                        "How do devices talk on same network?"
                        
Layer 1 - Physical      Cables, signals
                        "Is the cable plugged in?"

Troubleshooting tip: Start from Layer 1 up, or Layer 7 down
- Can't connect at all? Check L1-L3
- Connects but app fails? Check L4-L7
```

## 1.2 TCP Deep Dive

### Three-Way Handshake
```
Client                    Server
   |                         |
   |------- SYN (seq=x) ---->|
   |                         |
   |<-- SYN-ACK (seq=y, -----| 
   |       ack=x+1)          |
   |                         |
   |---- ACK (ack=y+1) ----->|
   |                         |
   [Connection Established]
```

### Connection Termination (Four-Way)
```
Client                    Server
   |                         |
   |------- FIN ------------>|  Client done sending
   |<------ ACK -------------|
   |                         |
   |<------ FIN -------------|  Server done sending
   |------- ACK ------------>|
   |                         |
   [TIME_WAIT on client]         2*MSL (60-120 sec)
```

### TCP States You'll See in Production

| State | Meaning | Common Issues |
|-------|---------|---------------|
| ESTABLISHED | Active connection | Normal |
| TIME_WAIT | Waiting after close | Too many = port exhaustion |
| CLOSE_WAIT | Waiting for app to close | App not closing connections (leak) |
| FIN_WAIT_2 | Waiting for server FIN | Server not closing properly |
| SYN_RECV | Got SYN, sent SYN-ACK | Too many = SYN flood attack |

### FAQ: Why do I have thousands of TIME_WAIT connections?

```
Answer: TIME_WAIT is normal - it prevents old packets from corrupting new 
connections on the same port. High counts usually mean:

1. High connection churn (many short-lived connections)
   Fix: Use connection pooling, keep-alive

2. Client-side closes (client sends FIN first)
   Fix: Let server close when possible

3. Tune kernel parameters (use cautiously):
   net.ipv4.tcp_tw_reuse = 1        # Reuse TIME_WAIT for outgoing
   net.ipv4.tcp_fin_timeout = 30    # Reduce FIN_WAIT timeout

Check with: ss -tan state time-wait | wc -l
```

### FAQ: What's the difference between TCP and UDP?

```
TCP:
- Connection-oriented (handshake required)
- Reliable (guaranteed delivery, ordering)
- Flow control, congestion control
- Higher overhead, higher latency
- Use for: HTTP, database connections, file transfer

UDP:
- Connectionless (fire and forget)
- Unreliable (no guarantee)
- No flow control
- Lower overhead, lower latency
- Use for: DNS, video streaming, gaming, metrics

Modern trend: QUIC (HTTP/3) = UDP + reliability at application layer
```

## 1.3 DNS

### Resolution Flow
```
Browser → Local Cache → OS Cache → Resolver → Root NS → TLD NS → Auth NS
                                      ↓
                              Returns IP address
```

### Record Types

| Type | Purpose | Example |
|------|---------|---------|
| A | IPv4 address | example.com → 93.184.216.34 |
| AAAA | IPv6 address | example.com → 2606:2800:220:1:... |
| CNAME | Alias to another name | www → example.com |
| MX | Mail server | priority 10 mail.example.com |
| TXT | Text data | SPF, DKIM, verification |
| NS | Name servers | ns1.example.com |
| PTR | Reverse lookup | IP → hostname |
| SRV | Service location | _http._tcp.example.com |

### FAQ: DNS is returning old IP after I changed it

```
Causes:
1. TTL hasn't expired - Records cached for TTL duration
2. Negative caching - "Not found" responses also cached
3. Client/OS caching - Beyond DNS TTL

Debugging:
# Check TTL
dig example.com +noall +answer

# Query specific nameserver (bypass cache)
dig @8.8.8.8 example.com

# Check propagation
dig @ns1.example.com example.com  # Authoritative
dig @8.8.8.8 example.com          # Google
dig @1.1.1.1 example.com          # Cloudflare

Prevention:
- Lower TTL before planned changes (24h before)
- After change verified, raise TTL back
```

### FAQ: How does DNS-based load balancing work?

```
Methods:
1. Round-robin DNS: Return multiple A records, client picks
   Pros: Simple
   Cons: No health checks, caching breaks distribution

2. Weighted DNS: Return records with different frequencies
   Use: Gradual traffic shifting

3. GeoDNS: Return different IPs based on client location
   Use: Global load balancing, latency reduction

4. Health-checked DNS (Route53, Cloud DNS):
   - Monitor endpoints
   - Remove unhealthy from responses
   - Failover to backup

Limitation: DNS TTL means slow failover (minutes, not seconds)
Solution: Combine with L4/L7 load balancers for fast failover
```

## 1.4 HTTP/HTTPS

### HTTP/1.1 vs HTTP/2 vs HTTP/3

```
HTTP/1.1:
- One request per connection (or pipelining, rarely used)
- Head-of-line blocking
- Text-based headers (verbose)
- Multiple TCP connections for parallelism

HTTP/2:
- Multiplexed streams over single connection
- Binary framing (efficient)
- Header compression (HPACK)
- Server push
- Still TCP, so TCP head-of-line blocking remains

HTTP/3:
- Based on QUIC (UDP)
- No TCP head-of-line blocking
- Built-in TLS 1.3
- Connection migration (survives IP changes)
- Faster connection establishment
```

### TLS Handshake (Simplified)

```
TLS 1.2 (2 round trips):
Client                         Server
   |-- ClientHello ------------>|
   |<- ServerHello, Cert, Done -|
   |-- KeyExchange, Finished -->|
   |<-------- Finished ---------|

TLS 1.3 (1 round trip):
Client                         Server
   |-- ClientHello + KeyShare ->|
   |<- ServerHello + KeyShare --|
   |<- {Cert, Finished} --------|
   |-- {Finished} ------------->|

0-RTT (resumption): Send data with first packet
Risk: Replay attacks, use carefully
```

### FAQ: How do I debug SSL/TLS issues?

```bash
# Test SSL connection
openssl s_client -connect example.com:443 -servername example.com

# Check certificate details
openssl s_client -connect example.com:443 | openssl x509 -text

# Check certificate chain
openssl s_client -connect example.com:443 -showcerts

# Test specific TLS version
openssl s_client -connect example.com:443 -tls1_2
openssl s_client -connect example.com:443 -tls1_3

# Common issues:
# - Certificate expired
# - Wrong hostname (SNI mismatch)
# - Incomplete chain (missing intermediate)
# - Protocol/cipher mismatch
```

## 1.5 Network Debugging Commands

```bash
# Connectivity
ping host                    # ICMP echo (basic connectivity)
traceroute host              # Path to destination
mtr host                     # Continuous traceroute
nc -zv host port             # TCP port check
telnet host port             # Interactive TCP

# DNS
dig domain                   # DNS lookup
dig +trace domain            # Full resolution path
nslookup domain              # Simple lookup
host domain                  # Simple lookup

# Connections
ss -tuln                     # Listening ports
ss -tan                      # All TCP connections
ss -tan state established    # Established only
netstat -an                  # All connections (older)
lsof -i :80                  # What's using port 80

# Traffic
tcpdump -i eth0 port 443     # Capture traffic
tcpdump -i any host 10.0.0.5 # Traffic to/from host
tcpdump -w file.pcap         # Save for Wireshark

# HTTP
curl -I url                  # Headers only
curl -v url                  # Verbose (shows TLS, headers)
curl -w "@format.txt" url    # Custom output (timing)
curl --resolve host:port:ip  # Override DNS

# Timing breakdown with curl
curl -w "DNS: %{time_namelookup}s
Connect: %{time_connect}s
TLS: %{time_appconnect}s
TTFB: %{time_starttransfer}s
Total: %{time_total}s\n" -o /dev/null -s https://example.com
```

---

# 2. LINUX SYSTEMS

---

## 2.1 Process Management

### Process States
```
R - Running/Runnable      Actively running or ready to run
S - Sleeping (Interruptible)  Waiting for event (normal)
D - Sleeping (Uninterruptible) Waiting for I/O (can't be killed)
Z - Zombie                Finished but parent hasn't read exit status
T - Stopped               Suspended (Ctrl+Z or debugger)
```

### FAQ: How do I find and kill a runaway process?

```bash
# Find high CPU processes
top -o %CPU                  # Interactive, sorted by CPU
ps aux --sort=-%cpu | head   # One-shot top CPU

# Find high memory processes
top -o %MEM
ps aux --sort=-%mem | head

# Find by name
pgrep -la nginx              # PID and command line
pidof nginx                  # Just PIDs

# Kill gracefully (SIGTERM)
kill <pid>
pkill -f "pattern"

# Force kill (SIGKILL) - last resort
kill -9 <pid>
pkill -9 -f "pattern"

# Kill all by name
killall nginx

Why not always use kill -9?
- Process can't clean up (temp files, locks, connections)
- May corrupt data
- Child processes may become orphans
Always try SIGTERM first, wait, then SIGKILL
```

### FAQ: What's a zombie process and how do I fix it?

```
Zombie: Process finished but parent hasn't called wait() to read exit status

Why bad: Consumes PID (limited resource), indicates buggy parent

Identify:
ps aux | grep 'Z'
ps -eo pid,ppid,stat,cmd | grep 'Z'

Fix:
1. Can't kill zombie directly (already dead)
2. Kill the parent process - orphaned zombies adopted by init and cleaned up
3. If parent is critical, fix the code to handle SIGCHLD properly

Prevention (in code):
signal(SIGCHLD, SIG_IGN);  // Auto-reap children
// Or handle SIGCHLD and call wait()
```

## 2.2 Memory Management

### Memory Types
```
Physical Memory: Actual RAM
Virtual Memory: Address space seen by process (can exceed physical)
Resident Memory (RSS): Physical RAM actually used
Shared Memory: Memory shared between processes (libs, etc.)
Swap: Disk space used when RAM is full
```

### Reading `free -h` Output
```
              total        used        free      shared  buff/cache   available
Mem:           32Gi        12Gi       1.5Gi       256Mi        18Gi        19Gi
Swap:          8Gi         0.5Gi      7.5Gi

Key insight: "available" is what matters, not "free"
- Linux uses free RAM for disk cache (buff/cache)
- This is released when apps need it
- "available" = free + reclaimable cache
```

### FAQ: Memory usage is high but no single process is using much?

```
Possible causes:

1. Disk cache (normal, good)
   Check: free -h shows high buff/cache but good "available"
   Action: None needed, it's optimization

2. Shared memory / tmpfs
   Check: df -h /dev/shm; ls -la /dev/shm
   Action: Clean up if needed

3. Slab cache (kernel data structures)
   Check: cat /proc/meminfo | grep Slab
   Check: slabtop
   Common culprit: dentry/inode cache from many files
   Action: echo 3 > /proc/sys/vm/drop_caches (temporary)

4. Memory fragmentation
   Check: cat /proc/buddyinfo
   Action: May need restart

5. Hugepages reserved
   Check: cat /proc/meminfo | grep Huge
   Action: Reduce vm.nr_hugepages if not needed
```

### FAQ: OOM Killer killed my process. Why and how to prevent?

```
Why OOM Killer exists:
- Kernel runs out of memory
- Must kill something to survive
- Picks victim by oom_score

Check who was killed:
dmesg | grep -i "killed process"
journalctl -k | grep -i oom

Prevent specific process from being killed:
# Lower oom_score_adj (-1000 to 1000, lower = less likely)
echo -500 > /proc/<pid>/oom_score_adj

# In systemd:
[Service]
OOMScoreAdjust=-500

Better solutions:
1. Add more RAM
2. Fix memory leaks
3. Set proper memory limits (cgroups/containers)
4. Enable swap (controversial for prod)
5. Use memory limits that trigger app restart before OOM
```

## 2.3 Disk & I/O

### Filesystem Basics
```bash
# Disk space
df -h                        # Filesystem usage
df -i                        # Inode usage (can run out with many small files)

# Directory sizes
du -sh /var/*                # Size of each item
du -h --max-depth=1 /        # Top level breakdown
ncdu /                       # Interactive (install ncdu)

# Find large files
find / -type f -size +100M -exec ls -lh {} \;
find / -xdev -type f -size +100M 2>/dev/null | head

# Find what's using space in deleted files
lsof +L1                     # Open files that are deleted
```

### FAQ: Disk is full but I can't find what's using space

```
Common causes:

1. Deleted files still open
   lsof +L1
   # Shows files deleted but held open by process
   # Fix: Restart the process, or truncate with:
   : > /proc/<pid>/fd/<fd_number>

2. Files in mounted-over directory
   # Something was written to /var before /var was mounted
   umount /var; ls /var; mount /var

3. Reserved space (ext4 reserves 5% for root)
   tune2fs -l /dev/sda1 | grep "Reserved"
   # Reduce if needed (carefully):
   tune2fs -m 1 /dev/sda1

4. Sparse files appearing larger
   du -h file    # Actual blocks used
   ls -lh file   # Apparent size
```

### I/O Performance

```bash
# I/O stats
iostat -x 1 5                # Extended stats every second
iotop -oP                    # I/O by process

# Key iostat metrics:
%util     # How busy the device is (>80% concerning)
await     # Average wait time (ms) - includes queue
r_await   # Read wait
w_await   # Write wait
avgqu-sz  # Average queue length

# High await + low %util = storage is slow
# High %util + high avgqu-sz = device saturated
```

### FAQ: What's causing high I/O wait?

```bash
# Find processes doing I/O
iotop -oP

# Check which files being accessed
# For specific process:
strace -p <pid> -e trace=read,write,open

# System-wide file access (if iotop not available)
perf record -e block:block_rq_issue -a sleep 10
perf report

# Common causes:
1. Swap thrashing (low memory)
   Check: vmstat 1 - look at si/so columns
   
2. Logging too much
   Check: iotop - is rsyslogd/journald high?
   
3. Database without enough RAM for working set
   Check: DB-specific metrics
   
4. Failing disk (high latency)
   Check: dmesg for disk errors
   
5. RAID rebuild
   Check: cat /proc/mdstat
```

## 2.4 System Performance Quick Reference

```bash
# One-liner health check
uptime; free -h; df -h; iostat -x 1 1

# CPU
top -bn1 | head -20          # Quick snapshot
mpstat -P ALL 1 5            # Per-CPU stats
pidstat 1 5                  # Per-process CPU

# Memory
free -h
vmstat 1 5                   # Virtual memory stats
cat /proc/meminfo

# Network
ss -s                        # Socket summary
sar -n DEV 1 5               # Network throughput

# Combined views
dstat                        # CPU, disk, net, paging
glances                      # Everything (install)
htop                         # Better top (install)
```

---

# 3. COMPUTE & CONTAINERS

---

## 3.1 Virtualization vs Containers

```
Virtual Machines:
┌─────────────────────────────────────┐
│            Application              │
├─────────────────────────────────────┤
│           Guest OS (full)           │
├─────────────────────────────────────┤
│            Hypervisor               │
├─────────────────────────────────────┤
│            Host OS/Hardware         │
└─────────────────────────────────────┘
- Full OS isolation
- Heavy (GBs), slow to start (minutes)
- Strong security boundary
- Use for: Different OS, strong isolation needed

Containers:
┌─────────────────────────────────────┐
│    App A    │    App B    │  App C  │
├─────────────┴─────────────┴─────────┤
│         Container Runtime           │
├─────────────────────────────────────┤
│            Host OS Kernel           │
└─────────────────────────────────────┘
- Shared kernel, isolated userspace
- Light (MBs), fast to start (seconds)
- Weaker isolation than VMs
- Use for: Microservices, same OS, density
```

### Container Isolation Mechanisms

```
Namespaces (what container sees):
- PID: Isolated process tree
- Network: Separate network stack
- Mount: Isolated filesystem view
- User: Mapped UID/GID
- UTS: Separate hostname
- IPC: Isolated inter-process communication

Cgroups (resource limits):
- CPU: cpu.shares, cpu.cfs_quota
- Memory: memory.limit_in_bytes
- I/O: blkio.*
- PIDs: pids.max

Together: Isolated view + resource limits = container
```

### FAQ: Container vs Pod?

```
Container:
- Single isolated process (or process tree)
- Has its own filesystem, network (if not shared)
- Basic unit of packaging

Pod (Kubernetes):
- Group of containers sharing:
  - Network namespace (same IP, localhost works)
  - Storage volumes
  - IPC namespace
- Scheduled together on same node
- Basic unit of deployment in K8s

When to use multi-container pods:
- Sidecar pattern (logging, proxy)
- Tightly coupled processes
- Shared data via emptyDir volume
```

## 3.2 Docker Essentials

```bash
# Lifecycle
docker run -d --name app nginx:latest
docker stop app
docker start app
docker rm app

# Debugging
docker ps -a                 # All containers
docker logs app              # Stdout/stderr
docker logs -f app           # Follow
docker exec -it app sh       # Shell into container
docker inspect app           # Full details

# Resources
docker stats                 # Live resource usage
docker top app               # Processes in container

# Images
docker images                # List images
docker pull nginx:1.25
docker build -t myapp:v1 .
docker push registry/myapp:v1

# Cleanup
docker system prune          # Remove unused data
docker volume prune          # Remove unused volumes
docker image prune -a        # Remove unused images
```

### FAQ: Container keeps restarting. How to debug?

```bash
# Check status
docker ps -a                 # See exit code

# Check logs
docker logs --tail 100 container_name
docker logs container_name 2>&1 | tail

# If container exits too fast, run interactively
docker run -it --entrypoint sh image_name

# Check events
docker events --since 1h

# Common causes:
1. Application error (check logs)
2. Missing environment variables
3. Missing config files/volumes
4. Health check failing
5. OOM killed (docker inspect, check OOMKilled)
6. Entrypoint/CMD issue

# For OOM:
docker inspect container_name | grep -i oom
# Increase memory limit or fix app
```

## 3.3 Kubernetes Essentials

### Architecture
```
Control Plane:
├── API Server: All requests go through here
├── etcd: Cluster state storage (critical!)
├── Scheduler: Assigns pods to nodes
├── Controller Manager: Reconciliation loops
└── Cloud Controller: Cloud provider integration

Worker Node:
├── kubelet: Pod lifecycle on node
├── kube-proxy: Network rules (Services)
└── Container Runtime: Docker/containerd
```

### Essential Objects

```yaml
# Pod - smallest deployable unit
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"

# Deployment - manages ReplicaSets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx:1.25

# Service - stable network endpoint
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP  # or NodePort, LoadBalancer
```

### Debugging Commands

```bash
# Get resources
kubectl get pods -o wide
kubectl get pods -A               # All namespaces
kubectl get events --sort-by='.lastTimestamp'

# Describe (detailed info + events)
kubectl describe pod myapp
kubectl describe node node1

# Logs
kubectl logs myapp
kubectl logs myapp -c sidecar     # Specific container
kubectl logs myapp --previous     # Previous instance
kubectl logs -l app=myapp         # By label

# Shell into pod
kubectl exec -it myapp -- /bin/sh

# Resource usage
kubectl top pods
kubectl top nodes

# Debug scheduling issues
kubectl describe pod <pending-pod>  # Check events
kubectl get events | grep <pod>

# Debug networking
kubectl run debug --image=busybox -it --rm -- sh
# Then: wget, nslookup, ping, etc.
```

### FAQ: Pod stuck in Pending state

```
Check why:
kubectl describe pod <pod-name>

Common causes:

1. Insufficient resources
   Events: "Insufficient cpu" or "Insufficient memory"
   Fix: Add nodes, reduce requests, check resource hogs

2. Node selector/affinity not matched
   Events: "0/3 nodes match"
   Fix: Check nodeSelector, add correct labels to nodes

3. Taints not tolerated
   Fix: Add tolerations or remove taints

4. PVC not bound
   Events: "persistentvolumeclaim not found"
   Fix: Create PV, check storage class

5. Image pull issues
   Fix: Check imagePullSecrets, image name
```

### FAQ: Pod stuck in CrashLoopBackOff

```bash
# CrashLoopBackOff = container keeps crashing and K8s backs off restarts

# Check logs
kubectl logs <pod> --previous

# Check events
kubectl describe pod <pod>

# Common causes:
1. Application error (check logs!)
2. Missing config/secrets
3. Health check failing immediately
4. Wrong command/args
5. Insufficient memory (OOMKilled)

# Check for OOM
kubectl describe pod <pod> | grep -i oom

# Debug interactively
kubectl run debug --image=<your-image> -it --rm --command -- sh
```

---

# 4. HIGH AVAILABILITY

---

## 4.1 HA Fundamentals

### Availability Math
```
Single component: A = MTBF / (MTBF + MTTR)

Series (all must work): A_total = A1 × A2 × A3
Example: 99.9% × 99.9% × 99.9% = 99.7%

Parallel (any can work): A_total = 1 - (1-A1) × (1-A2)
Example: Two 99% systems = 1 - (0.01 × 0.01) = 99.99%

Key insight: Add parallel components to increase availability
```

### FAQ: How do I design for 99.99% availability?

```
99.99% = 52 minutes downtime per year

Requirements:
1. No single points of failure (SPOF)
   - Every component has backup
   - Database: Primary + replica
   - App servers: Multiple instances
   - Load balancers: Active-passive or active-active

2. Multiple availability zones
   - Survive zone failure
   - Data replicated across zones
   - Automatic failover

3. Fast detection and recovery
   - Health checks every 5-10 seconds
   - Automatic failover < 30 seconds
   - No human in the loop for common failures

4. Deployment safety
   - Canary deployments
   - Automatic rollback on failure
   - No big-bang changes

5. Operational excellence
   - Runbooks for all alerts
   - Regular failover testing
   - Chaos engineering
```

## 4.2 Failover Patterns

### Active-Passive (Hot Standby)
```
┌─────────┐         ┌─────────┐
│ Primary │ ──────> │ Standby │  (replication)
│ (active)│         │(passive)│
└────┬────┘         └────┬────┘
     │                   │
     └───────┬───────────┘
             │
     ┌───────▼───────┐
     │  Health Check │
     │  / Failover   │
     └───────────────┘

Failover: Promote standby to primary when primary fails

Pros: Simple, standby has full data
Cons: Standby resources idle, failover can be slow
Use for: Databases, stateful services
```

### Active-Active
```
┌─────────┐         ┌─────────┐
│ Node A  │ <─────> │ Node B  │  (sync or async)
│(active) │         │(active) │
└────┬────┘         └────┬────┘
     │                   │
     └───────┬───────────┘
             │
     ┌───────▼───────┐
     │ Load Balancer │
     └───────────────┘

Both nodes handle traffic. If one fails, other continues.

Pros: Better resource utilization, faster failover
Cons: Data consistency challenges, conflict resolution needed
Use for: Stateless services, read-heavy workloads
```

### FAQ: Database failover - synchronous vs asynchronous replication?

```
Synchronous Replication:
┌─────────┐  write  ┌─────────┐
│ Primary │ ──────> │ Replica │
└─────────┘         └─────────┘
     │                   │
     └───── ack ─────────┘
     │
Response to client only after replica confirms

Pros: No data loss on failover (RPO = 0)
Cons: Higher latency, primary blocked if replica slow

Asynchronous Replication:
┌─────────┐  write  ┌─────────┐
│ Primary │ ──────> │ Replica │ (applied later)
└─────────┘         └─────────┘
     │
Response immediately

Pros: Lower latency, primary not blocked
Cons: Potential data loss on failover (RPO > 0)

Trade-off: Consistency vs Availability vs Latency
- Financial transactions: Sync (no data loss)
- Social media posts: Async (speed matters more)
```

## 4.3 Health Checks

### Types of Health Checks

```yaml
# Kubernetes example with all three types

livenessProbe:           # Is the process alive?
  httpGet:               # If fails, container restarted
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:          # Is it ready for traffic?
  httpGet:               # If fails, removed from service
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

startupProbe:            # Has it started yet?
  httpGet:               # Disables liveness until success
    path: /healthz       # For slow-starting apps
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

### FAQ: What should health check endpoints actually check?

```python
# Bad: Always returns 200
@app.route('/health')
def health():
    return 'OK', 200

# Good: Actually checks dependencies
@app.route('/health')
def health():
    checks = {
        'database': check_db_connection(),
        'cache': check_redis_connection(),
        'disk': check_disk_space(),
    }
    
    if all(checks.values()):
        return jsonify(checks), 200
    else:
        return jsonify(checks), 503

# Best practice: Separate endpoints
/health   - Basic liveness (process running)
/ready    - Can handle traffic (dependencies OK)
/health/detailed - Full status (for debugging, not LB)

Important considerations:
- Don't make health checks too slow
- Don't fail health check for non-critical dependencies
- Cache dependency check results briefly
- Timeout health checks (don't hang forever)
```

---

# 5. DATABASES

---

## 5.1 SQL vs NoSQL Decision

```
Choose SQL (PostgreSQL, MySQL) when:
- Complex queries, joins, aggregations
- ACID transactions required
- Data model is relational
- Strong consistency needed
- Mature tooling, known patterns

Choose NoSQL when:
- Massive scale (millions of ops/sec)
- Flexible schema needed
- Specific access patterns (key-value, document, graph)
- Eventually consistent is acceptable
- Horizontal scaling is priority

NoSQL Types:
- Key-Value (Redis, DynamoDB): Simple lookups, caching
- Document (MongoDB): Flexible documents, nested data
- Column-family (Cassandra): Time-series, write-heavy
- Graph (Neo4j): Relationship-heavy queries
```

## 5.2 Database Performance

### Key Metrics to Monitor

```
Query Performance:
- Queries per second
- Query latency (p50, p95, p99)
- Slow query count
- Lock wait time

Connections:
- Active connections
- Connection pool utilization
- Connection errors

Replication:
- Replication lag
- Replication status

Resources:
- CPU utilization
- Memory usage (buffer pool)
- Disk I/O (reads, writes)
- Disk space
```

### FAQ: Database queries are slow. How do I debug?

```sql
-- Step 1: Find slow queries

-- MySQL
SHOW PROCESSLIST;
-- Check slow_query_log

-- PostgreSQL
SELECT * FROM pg_stat_activity WHERE state = 'active';
-- Check pg_stat_statements extension

-- Step 2: Analyze specific query
EXPLAIN ANALYZE SELECT ...;

-- Look for:
-- - Full table scans (bad for large tables)
-- - Missing index usage
-- - High row estimates vs actuals
-- - Sort operations on large sets
-- - Nested loop joins on large tables

-- Step 3: Common fixes

-- Add index
CREATE INDEX idx_users_email ON users(email);
-- Compound index for multiple columns
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- Optimize query
-- Bad: SELECT * (fetches all columns)
-- Good: SELECT id, name (only needed columns)

-- Bad: SELECT ... WHERE YEAR(created_at) = 2024
-- Good: SELECT ... WHERE created_at >= '2024-01-01' (can use index)

-- Step 4: Check for locks
-- MySQL
SHOW ENGINE INNODB STATUS;
-- PostgreSQL
SELECT * FROM pg_locks WHERE granted = false;
```

### FAQ: Database connections are exhausted

```
Symptoms:
- "too many connections" errors
- Application timeouts
- Connection pool full

Debugging:
-- MySQL
SHOW STATUS LIKE 'Threads_connected';
SHOW VARIABLES LIKE 'max_connections';
SHOW PROCESSLIST;

-- PostgreSQL
SELECT count(*) FROM pg_stat_activity;
SHOW max_connections;
SELECT * FROM pg_stat_activity;

Common causes:
1. Connection leak (app not closing connections)
   - Check app code for connection handling
   - Ensure connections returned to pool

2. Too many app instances
   - 10 app instances × 20 pool size = 200 connections
   - Use PgBouncer or ProxySQL for connection pooling

3. Long-running queries holding connections
   - Find and optimize slow queries
   - Set query timeout

4. Pool size too large
   - More connections ≠ better
   - Diminishing returns, context switching overhead
   - Start with: connections = (cores × 2) + effective_spindle_count

Solutions:
1. Connection pooler (PgBouncer, ProxySQL)
2. Reduce pool size per app
3. Fix connection leaks
4. Kill idle connections
5. Increase max_connections (last resort)
```

## 5.3 Replication & Consistency

### Replication Patterns

```
Primary-Replica (Read Replicas):
┌─────────┐    async    ┌─────────┐
│ Primary │ ──────────> │ Replica │
│ (R/W)   │             │ (R only)│
└─────────┘             └─────────┘

Use: Read scaling, reporting queries, backups
Caveat: Replica lag means stale reads

Primary-Primary (Multi-Master):
┌─────────┐  <────────> ┌─────────┐
│ Primary │  conflict   │ Primary │
│   A     │  resolution │   B     │
└─────────┘             └─────────┘

Use: Geo-distributed writes, HA
Caveat: Conflict resolution complexity
```

### FAQ: How do I handle replication lag?

```
Causes:
1. Replica under-resourced (CPU/IO/network)
2. Large transactions on primary
3. Heavy read load on replica
4. Network issues

Monitoring:
-- MySQL
SHOW SLAVE STATUS\G  -- Seconds_Behind_Master

-- PostgreSQL
SELECT pg_last_wal_receive_lsn() - pg_last_wal_replay_lsn();

Handling in application:
1. Read-your-writes consistency
   - After write, read from primary briefly
   - Or track last write timestamp, route to primary if recent

2. Session consistency
   - Route user session to same replica
   - Sticky sessions at load balancer

3. Accept eventual consistency
   - For non-critical reads, stale data acceptable
   - Show "data may be delayed" in UI

4. Causal consistency
   - Pass version/timestamp from write to read
   - Only read from replica if caught up
```

---

# 6. APPLICATION & APIs

---

## 6.1 REST API Best Practices

### HTTP Methods

```
GET     - Read resource (idempotent, cacheable)
POST    - Create resource (not idempotent)
PUT     - Replace resource (idempotent)
PATCH   - Partial update (may not be idempotent)
DELETE  - Remove resource (idempotent)

Idempotent: Multiple identical requests = same result
```

### Status Codes

```
2xx Success:
200 OK              - Success
201 Created         - Resource created
204 No Content      - Success, no body (DELETE)

3xx Redirection:
301 Moved Permanently
302 Found (temporary redirect)
304 Not Modified (cache valid)

4xx Client Error:
400 Bad Request     - Invalid input
401 Unauthorized    - Not authenticated
403 Forbidden       - Authenticated but not allowed
404 Not Found       - Resource doesn't exist
409 Conflict        - State conflict
422 Unprocessable   - Validation error
429 Too Many Requests - Rate limited

5xx Server Error:
500 Internal Error  - Unexpected error
502 Bad Gateway     - Upstream error
503 Service Unavailable - Overloaded/maintenance
504 Gateway Timeout - Upstream timeout
```

### FAQ: API is returning 502/504 errors. What's happening?

```
502 Bad Gateway:
- Upstream server sent invalid response
- Upstream server crashed
- Connection refused by upstream

504 Gateway Timeout:
- Upstream server took too long to respond
- Request timed out at proxy/LB

Debugging:
1. Check upstream service health
2. Check upstream service logs
3. Check network between proxy and upstream
4. Check timeout configurations

Load balancer → [502/504] → App server

Common causes:
- App server crashed/not running (502)
- App server overloaded, slow responses (504)
- App server health check passing but app failing (502)
- Timeout too short for slow operations (504)
- Connection limits exhausted (502)

Fixes:
- Increase timeouts (carefully)
- Scale app servers
- Fix slow endpoints
- Add circuit breaker
- Improve health checks
```

## 6.2 API Performance & Reliability

### Timeouts - Get Them Right

```
Client → LB → API → Service → Database

Each hop needs timeout < previous hop:
Client timeout: 30s
LB timeout: 25s
API to Service: 20s
Service to DB: 15s

Why? Avoid:
- Client retries while server still processing
- Wasted resources on requests client abandoned
- Cascading timeouts

Rule: timeout_upstream < timeout_downstream
Include time for retries in your timeout budget
```

### Retry Best Practices

```python
# Bad: Immediate retries
for i in range(3):
    try:
        return call_service()
    except:
        pass  # Retry immediately

# Good: Exponential backoff with jitter
import random
import time

def call_with_retry(func, max_retries=3, base_delay=1):
    for attempt in range(max_retries):
        try:
            return func()
        except RetryableError as e:
            if attempt == max_retries - 1:
                raise
            
            # Exponential backoff
            delay = base_delay * (2 ** attempt)
            
            # Add jitter (±25%)
            jitter = delay * 0.25 * (random.random() * 2 - 1)
            
            time.sleep(delay + jitter)
    
# Why jitter?
# Without: All clients retry at same time → thundering herd
# With: Retries spread out → smoother load
```

### Circuit Breaker Pattern

```
States:
CLOSED → (failures exceed threshold) → OPEN
OPEN → (timeout expires) → HALF_OPEN
HALF_OPEN → (success) → CLOSED
HALF_OPEN → (failure) → OPEN

Implementation:
┌────────────────────────────────────────────────────┐
│                  Circuit Breaker                   │
│                                                    │
│  Closed: Normal operation, count failures         │
│          If failures > threshold → Open           │
│                                                    │
│  Open: Fail fast, don't call downstream           │
│        After timeout → Half-Open                  │
│                                                    │
│  Half-Open: Allow one request through             │
│             If success → Closed                   │
│             If failure → Open                     │
└────────────────────────────────────────────────────┘

Benefits:
- Fail fast when downstream is down
- Give downstream time to recover
- Prevent cascade failures
```

## 6.3 API Security

### Authentication Methods

```
API Keys:
- Simple, in header or query param
- No expiration (unless rotated)
- Use for: Server-to-server, public APIs with rate limits

JWT (JSON Web Tokens):
- Self-contained, signed token
- Contains claims (user ID, roles, expiry)
- Stateless verification (no DB lookup)
- Use for: User authentication, microservices

OAuth 2.0:
- Authorization framework
- Access tokens + refresh tokens
- Scopes for granular permissions
- Use for: Third-party access, user delegation

mTLS (Mutual TLS):
- Both client and server have certificates
- Very strong authentication
- Use for: Internal services, high security
```

### FAQ: JWT vs session cookies for authentication?

```
JWT Pros:
- Stateless (no session store needed)
- Works across domains
- Good for microservices
- Mobile-friendly

JWT Cons:
- Can't revoke individual tokens easily
- Token size can be large
- If stolen, valid until expiry

Session Pros:
- Easy to revoke (delete from store)
- Smaller cookie size
- Server controls session data

Session Cons:
- Requires session store (Redis, DB)
- Sticky sessions or shared store needed
- Cookie domain restrictions

Recommendation:
- Traditional web app: Sessions
- SPA + API: JWT with short expiry + refresh tokens
- Microservices: JWT (internal), mTLS
- Sensitive ops: JWT + additional verification
```

---

# 7. LOAD BALANCING

---

## 7.1 Load Balancer Types

### Layer 4 (Transport) vs Layer 7 (Application)

```
Layer 4 LB:
- Routes based on IP and port
- Doesn't inspect HTTP content
- Lower latency, higher throughput
- No URL routing, no SSL termination (usually)
- Examples: AWS NLB, HAProxy (TCP mode)

Use when:
- High performance needed
- Simple round-robin is fine
- Non-HTTP protocols (TCP, UDP)

Layer 7 LB:
- Inspects HTTP headers, URLs, cookies
- Can route based on path, host, headers
- SSL termination
- Can modify requests/responses
- Examples: AWS ALB, nginx, HAProxy (HTTP mode)

Use when:
- Need URL-based routing (/api → service A)
- Need host-based routing (a.com vs b.com)
- Need sticky sessions
- SSL termination at LB
```

### Load Balancing Algorithms

```
Round Robin:
- Requests distributed evenly in order
- Simple, fair if servers are equal
- Problem: Doesn't account for server load

Weighted Round Robin:
- Servers get proportional traffic by weight
- Server A (weight 3): 3 requests
- Server B (weight 1): 1 request
- Use for: Different server capacities

Least Connections:
- Route to server with fewest active connections
- Good for: Varying request duration
- Problem: New server gets all traffic initially

Least Response Time:
- Route to fastest responding server
- Considers both connections and latency
- Good for: Mixed server performance

IP Hash:
- Hash client IP to select server
- Same client → same server
- Use for: Sticky sessions without cookies
- Problem: Uneven if client IP distribution skewed

Consistent Hashing:
- Minimizes redistribution when servers change
- Good for: Caching, stateful services
- Used by: Many distributed systems
```

### FAQ: How do I implement sticky sessions?

```
Methods:

1. Cookie-based (L7 LB)
   - LB inserts cookie with server ID
   - Subsequent requests routed by cookie
   - Most flexible, survives IP changes
   
   nginx example:
   upstream backend {
       server 10.0.0.1:8080;
       server 10.0.0.2:8080;
       sticky cookie srv_id expires=1h;
   }

2. IP-based (L4 or L7)
   - Hash client IP
   - Simple but breaks with NAT, proxies
   
   nginx example:
   upstream backend {
       ip_hash;
       server 10.0.0.1:8080;
       server 10.0.0.2:8080;
   }

3. Application-managed
   - Store session in shared store (Redis)
   - Any server can handle any request
   - Most scalable, recommended

When to use sticky sessions:
- WebSocket connections
- File upload in progress
- Legacy apps without shared session

Better: Design for statelessness
- Shared session store
- JWT tokens
- Pass state with request
```

## 7.2 Health Checks

```yaml
# nginx health check
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    
    # Passive health check (mark failed after errors)
    server 10.0.0.3:8080 max_fails=3 fail_timeout=30s;
}

# Active health check (nginx plus or alternatives)
# - Regularly probe /health endpoint
# - Remove from pool if unhealthy
# - Add back when healthy

# HAProxy health check
backend webservers
    option httpchk GET /health
    http-check expect status 200
    server web1 10.0.0.1:8080 check inter 5s fall 3 rise 2
    server web2 10.0.0.2:8080 check inter 5s fall 3 rise 2

# inter 5s: Check every 5 seconds
# fall 3: Mark down after 3 failures
# rise 2: Mark up after 2 successes
```

### FAQ: Load balancer shows all backends unhealthy but app works directly

```
Common causes:

1. Health check endpoint wrong
   - LB checks /health but app has /healthz
   - Check LB configuration

2. Health check from wrong network
   - App allows 10.0.0.0/24, LB is in 10.1.0.0/24
   - Check security groups/firewall

3. Different ports
   - LB checks 8080, app on 8081
   - Verify port configuration

4. Host header required
   - App needs Host: example.com
   - LB sends IP or wrong hostname
   - Configure health check headers

5. Health check timeout too short
   - App takes 5s to respond, LB timeout 2s
   - Increase timeout or optimize health endpoint

6. SSL/TLS mismatch
   - LB sends HTTP, app expects HTTPS
   - Or certificate validation failing

Debug steps:
1. Curl from LB to backend (exact health check request)
2. Check backend logs for health check requests
3. Compare working direct request vs failed LB request
4. Check timing (is it timing out?)
```

---

# 8. TROUBLESHOOTING METHODOLOGY

---

## 8.1 Systematic Approach

```
┌─────────────────────────────────────────────────────┐
│              Troubleshooting Flow                   │
│                                                     │
│  1. OBSERVE                                         │
│     What's the symptom? Who's affected?             │
│     When did it start? Is it getting worse?         │
│                                                     │
│  2. CORRELATE                                       │
│     What changed? (deploy, config, traffic)         │
│     Check timeline of changes vs issue start        │
│                                                     │
│  3. HYPOTHESIZE                                     │
│     List possible causes                            │
│     Rank by likelihood and ease of testing          │
│                                                     │
│  4. TEST                                            │
│     Test hypothesis with minimal impact             │
│     Eliminate or confirm each theory                │
│                                                     │
│  5. MITIGATE                                        │
│     Fix the immediate issue                         │
│     Rollback, restart, scale, failover              │
│                                                     │
│  6. RESOLVE                                         │
│     Fix root cause                                  │
│     Document and prevent recurrence                 │
└─────────────────────────────────────────────────────┘
```

## 8.2 Common Scenarios Quick Reference

### Scenario: High Latency

```
Check order:
1. Is it all requests or specific ones?
   → Specific: Check that endpoint's dependencies
   → All: Check shared resource (DB, network)

2. When did it start?
   → Correlate with deploys, traffic, changes

3. Resource saturation?
   → CPU: top, check GC pauses
   → Memory: free, check swap usage
   → Disk: iostat, check I/O wait
   → Network: check bandwidth, packet loss

4. External dependencies?
   → Database: slow queries, connection pool
   → Cache: hit rate dropped, evictions
   → Downstream services: their latency up?

5. Application level?
   → Logs: errors, slow operations
   → Profiling: where is time spent?
   → Threads: deadlocks, thread pool exhausted

Quick mitigations:
- Scale out (if resource bottleneck)
- Restart (if memory leak / stale state)
- Rollback (if recent deploy)
- Shed load (return 503 early)
```

### Scenario: Service Unavailable (5xx errors)

```
Check order:
1. Is the service running?
   → Process up? kubectl get pods / docker ps
   → Health check passing?

2. Can it accept connections?
   → netcat/telnet to port
   → Check connection limits

3. Recent changes?
   → Deploy in progress?
   → Config change?
   → Dependency changed?

4. Resource exhaustion?
   → Out of memory (OOM killed)?
   → Disk full?
   → File descriptors exhausted?
   → Threads/goroutines maxed?

5. Dependency failures?
   → Database down?
   → Cache down?
   → Network partition?

Quick mitigations:
- Rollback deployment
- Restart service
- Scale up
- Failover to backup
- Enable degraded mode
```

### Scenario: Connection Issues

```
"Cannot connect to service X"

Check order:
1. Is service running?
   → ps, docker ps, kubectl get pods

2. Is it listening?
   → ss -tuln | grep PORT
   → netstat -an | grep LISTEN

3. Network path?
   → ping (ICMP might be blocked)
   → telnet host port
   → traceroute

4. Firewall/security groups?
   → iptables -L
   → Cloud console SG rules
   → Network ACLs

5. DNS?
   → nslookup/dig service
   → Is it resolving correctly?

6. Load balancer?
   → Backend health status
   → LB configuration

Common gotchas:
- Binding to 127.0.0.1 instead of 0.0.0.0
- Firewall rule order (deny before allow)
- Security group missing
- Service not in DNS yet
- Wrong port in configuration
```

### Scenario: Disk Space Full

```bash
# Find what's using space
df -h                        # Which filesystem is full
du -sh /* | sort -h          # Top-level breakdown
du -sh /var/* | sort -h      # Drill down

# Find large files
find / -xdev -type f -size +100M -exec ls -lh {} \; 2>/dev/null

# Check for deleted files still open
lsof +L1

# If /var/log full
# Check log sizes
du -sh /var/log/*

# Truncate large log (don't delete if process has handle)
cat /dev/null > /var/log/large.log

# Fix: Set up log rotation
# /etc/logrotate.d/myapp

# If Docker eating space
docker system df             # What's using space
docker system prune -a       # Clean up (careful!)

# If /tmp full
find /tmp -type f -atime +7 -delete
```

## 8.3 Quick Diagnostic Commands

```bash
# System overview
uptime                       # Load averages
top -bn1 | head -20          # Quick process snapshot
free -h                      # Memory
df -h                        # Disk space

# Process issues
ps aux --sort=-%cpu | head   # Top CPU
ps aux --sort=-%mem | head   # Top memory
pgrep -af "process"          # Find process
lsof -p PID                  # Open files/connections
strace -p PID -f             # System calls

# Memory
vmstat 1 5                   # Virtual memory
cat /proc/meminfo            # Detailed memory
slabtop                      # Kernel slab cache

# Disk I/O
iostat -x 1 5                # Disk stats
iotop -oP                    # I/O by process
lsof +D /path                # Files open in directory

# Network
ss -tuln                     # Listening ports
ss -tan state established    # Connections
ss -tan state time-wait | wc # TIME_WAIT count
netstat -s                   # Network stats
tcpdump -i eth0 port 80      # Traffic capture

# Logs
tail -f /var/log/syslog      # System log
journalctl -u service -f     # Service log
dmesg -T | tail              # Kernel messages
grep -r "error" /var/log/    # Find errors

# Kubernetes
kubectl get pods -o wide
kubectl describe pod NAME
kubectl logs NAME --previous
kubectl top pods
kubectl get events --sort-by='.lastTimestamp'
```

---

# 9. MONITORING & OBSERVABILITY

---

## 9.1 The Three Pillars

```
Metrics:
- Numeric measurements over time
- Aggregatable (counters, gauges, histograms)
- Good for: Alerting, dashboards, capacity planning
- Tools: Prometheus, Datadog, CloudWatch

Logs:
- Discrete events with context
- High cardinality, detailed
- Good for: Debugging, audit trails
- Tools: ELK, Splunk, CloudWatch Logs

Traces:
- Request flow across services
- Causality and timing
- Good for: Distributed debugging, latency analysis
- Tools: Jaeger, Zipkin, X-Ray
```

## 9.2 Key Metrics (Golden Signals)

```
Rate (Traffic):
- Requests per second
- Measure: How busy are we?

Errors:
- Error rate, error types
- Measure: How correct are we?

Duration (Latency):
- Response time percentiles (p50, p95, p99)
- Measure: How fast are we?

Saturation:
- Resource utilization, queue depth
- Measure: How full are we?
```

### USE Method (for Resources)

```
For each resource (CPU, memory, disk, network):

Utilization: % of resource used
- CPU: time spent busy
- Memory: % used
- Disk: % capacity used
- Network: % bandwidth used

Saturation: Work waiting for resource
- CPU: run queue length
- Memory: swap usage, OOM events
- Disk: I/O queue depth
- Network: dropped packets

Errors: Error count
- CPU: (rare)
- Memory: ECC errors
- Disk: read/write errors
- Network: CRC errors, drops
```

### RED Method (for Services)

```
For each service:

Rate: Requests per second
Errors: Failed requests per second
Duration: Time per request

This captures user experience directly.
```

## 9.3 Alerting Best Practices

```
Alert on Symptoms, Not Causes:
✗ CPU > 80%                  (cause)
✓ Request latency > 500ms    (symptom users feel)

Alert on What's Actionable:
✗ GC time increasing         (not actionable)
✓ Error rate > 1%            (need to investigate)

Good Alert Properties:
- Actionable: Someone can do something
- Urgent: Needs attention now
- Unique: Not duplicate of another alert
- Documented: Runbook linked
- Tested: Has fired correctly before

Alert Structure:
Title: Clear description
Severity: P1/P2/P3/P4
Impact: Who is affected
Runbook: Link to response steps
Dashboard: Link to relevant metrics
```

---

# 10. QUICK INTERVIEW ANSWERS

---

## Networking
**Q: How does HTTPS work?**
TLS handshake establishes encrypted channel: client hello, server responds with certificate, key exchange creates session keys, all traffic encrypted. TLS 1.3 reduced to 1 round trip, 0-RTT possible for resumption.

**Q: What's the difference between TCP and UDP?**
TCP: connection-oriented, reliable delivery, ordering guaranteed, flow control. UDP: connectionless, fire-and-forget, no ordering guarantees, lower latency. Use TCP for HTTP, databases. Use UDP for DNS, streaming, gaming.

**Q: Explain DNS resolution.**
Recursive: Client → Local resolver → Root NS → TLD NS → Authoritative NS → Returns IP. Results cached at each level based on TTL.

## Linux
**Q: Process is taking too much CPU. How do you investigate?**
`top/htop` to identify, `strace -p PID` for system calls, `perf top` for CPU profiling, check if it's actually doing work or stuck in loop. Consider if it's expected behavior under load.

**Q: Explain Linux permissions.**
rwx for user, group, others. chmod numeric (755) or symbolic (u+x). SUID/SGID for elevated execution. Directories need x to traverse. Use 755 for dirs, 644 for files typically.

**Q: What happens when you run a command?**
Shell forks, child execs the command, parent waits. exec replaces process image. PATH searched if not absolute path. Shell built-ins don't fork.

## Databases
**Q: How do you scale a database?**
Read scaling: read replicas. Write scaling: sharding (application routes to correct shard). Caching reduces load. Vertical scaling (bigger instance) as first step. Connection pooling reduces connection overhead.

**Q: Explain database indexing.**
B-tree indexes allow O(log n) lookups instead of O(n) table scans. Index columns used in WHERE, JOIN, ORDER BY. Trade-off: faster reads, slower writes, storage overhead. Composite indexes for multi-column queries, column order matters.

## High Availability
**Q: How do you achieve 99.99% uptime?**
Multi-AZ deployment, no single points of failure, automated failover under 30 seconds, canary deployments, error budgets for controlled risk, chaos engineering to verify resilience.

**Q: Explain CAP theorem.**
Distributed system can't guarantee all three: Consistency, Availability, Partition tolerance. During network partition, must choose C or A. Most systems choose availability with eventual consistency.

## Load Balancing
**Q: L4 vs L7 load balancer?**
L4: routes by IP/port, faster, can't inspect HTTP. L7: routes by URL/headers/cookies, can do SSL termination, can modify requests. Use L4 for raw performance, L7 for intelligent routing.

**Q: How does consistent hashing work?**
Nodes and keys mapped to ring via hash. Key served by next node clockwise. When node added/removed, only adjacent keys move. Used for distributed caches, databases to minimize data movement.

## Troubleshooting
**Q: Service is slow. How do you approach?**
Clarify scope (all users? all endpoints?), check recent changes, look at golden signals (latency, errors, traffic, saturation), check dependencies (DB, cache, downstream), narrow down with logs and traces, mitigate (scale, rollback, restart) while investigating.

**Q: Application OOM killed. What do you do?**
Check dmesg/journalctl for OOM messages, identify which process was killed and why (oom_score). Investigate: memory leak? undersized instance? spike in traffic? Fix: increase limits, fix leak, add horizontal scaling, tune JVM/GC settings.

---

*Last updated: 2025. Use this as a quick reference before interviews or for day-to-day troubleshooting.*
