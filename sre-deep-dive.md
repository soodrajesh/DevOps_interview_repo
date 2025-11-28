# SRE Deep Dive: ML Infrastructure, Troubleshooting & Google SRE Principles

---

# Section 1: ML Infrastructure for SRE

---

## 1.1 Understanding TPUs vs GPUs

### What is a TPU?

TPU (Tensor Processing Unit) is Google's custom-built ASIC specifically designed for machine learning workloads.

```
TPU Architecture (Simplified):

┌─────────────────────────────────────────┐
│              TPU v4 Pod                 │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │
│  │TPU  │ │TPU  │ │TPU  │ │TPU  │       │
│  │Chip │ │Chip │ │Chip │ │Chip │       │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘       │
│     │       │       │       │          │
│  ┌──┴───────┴───────┴───────┴──┐       │
│  │    High-Bandwidth Interconnect      │
│  │         (ICI - Inter-Core)          │
│  └─────────────────────────────┘       │
└─────────────────────────────────────────┘

Key Components:
- Matrix Multiply Units (MXUs): Core compute for tensor operations
- High Bandwidth Memory (HBM): Fast memory access
- Inter-Core Interconnect (ICI): TPU-to-TPU communication
```

### TPU vs GPU Comparison

| Aspect | TPU | GPU (NVIDIA) |
|--------|-----|--------------|
| **Design** | Custom for ML | General-purpose parallel |
| **Precision** | bfloat16 optimized | FP16, FP32, TF32 |
| **Memory** | HBM2e, shared across chips | HBM per GPU |
| **Interconnect** | ICI (Google proprietary) | NVLink, InfiniBand |
| **Programming** | JAX, TensorFlow | CUDA, PyTorch native |
| **Scaling** | Pod-level (4096 chips) | Cluster of individual GPUs |
| **Cost Model** | Per TPU-hour | Per GPU-hour |
| **Failure Mode** | Chip failures affect pod | Individual GPU failures |

### SRE Implications for TPU Infrastructure

**1. Failure Handling**
```
TPU Failure Modes:
├── Chip failures (individual TPU dies)
│   └── Impact: Training job fails, needs restart from checkpoint
├── ICI link failures (interconnect)
│   └── Impact: Pod communication broken, entire pod unusable
├── Host failures (CPU/memory of TPU host)
│   └── Impact: TPU becomes unreachable
└── Thermal throttling
    └── Impact: Reduced performance, potential job slowdown

Mitigation Strategies:
1. Frequent checkpointing (every N steps)
2. Automatic job restart with checkpoint recovery
3. Health monitoring at chip level
4. Spare capacity allocation
5. Graceful degradation (run on smaller slice)
```

**2. Capacity Planning**
```python
# TPU capacity considerations
class TPUCapacityPlanner:
    def __init__(self):
        self.tpu_types = {
            'v4': {'chips_per_pod': 4096, 'tflops': 275},
            'v5e': {'chips_per_pod': 256, 'tflops': 197},
            'v5p': {'chips_per_pod': 8960, 'tflops': 459}
        }
    
    def estimate_training_capacity(self, model_params, batch_size, 
                                    target_throughput):
        """
        Estimate TPU requirements for training job
        
        model_params: Number of parameters (e.g., 175B for GPT-3 scale)
        batch_size: Global batch size
        target_throughput: Tokens per second target
        """
        # Rough heuristic: larger models need more chips for memory
        # and compute parallelism
        
        # Memory requirement (params + gradients + optimizer states)
        # ~20 bytes per parameter for Adam optimizer
        memory_gb = (model_params * 20) / 1e9
        
        # TPU v4 has ~32GB HBM per chip
        min_chips_memory = memory_gb / 32
        
        # Compute requirement based on throughput
        # More complex in reality - depends on model architecture
        
        return {
            'min_chips': max(min_chips_memory, 8),
            'recommended_pod_slice': self._nearest_pod_slice(min_chips_memory)
        }
```

**3. Monitoring TPU Health**
```yaml
# Key TPU metrics to monitor
TPU Metrics:
  Hardware:
    - tpu/chip/health_status        # Per-chip health
    - tpu/chip/temperature          # Thermal monitoring
    - tpu/chip/power_usage          # Power consumption
    - tpu/memory/usage              # HBM utilization
    
  Performance:
    - tpu/mxu/utilization           # Matrix unit usage (should be high)
    - tpu/memory/bandwidth_usage    # Memory bandwidth
    - tpu/ici/bandwidth_usage       # Interconnect bandwidth
    
  Job Level:
    - training/steps_per_second     # Training throughput
    - training/loss                 # Model convergence
    - training/checkpoint_latency   # Time to save checkpoint
    
  Alerts:
    - MXU utilization < 30% for 10 min  # Inefficient job
    - Chip health degraded              # Hardware issue
    - Checkpoint failed                 # Data loss risk
    - Step time variance > 20%          # Performance instability
```

---

## 1.2 Model Serving Architecture

### Inference vs Training: Different Beasts

```
Training Characteristics:
├── Long-running jobs (hours to months)
├── Batch processing (large batches = efficient)
├── Checkpoint frequently
├── Can retry/restart
├── Throughput optimized
└── Predictable resource usage

Inference Characteristics:
├── Request-response (milliseconds)
├── Real-time latency requirements
├── Variable traffic patterns
├── Must be highly available
├── Latency optimized
├── Unpredictable spikes
└── Cost per request matters
```

### LLM Serving Architecture

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    Load Balancer                        │
                    │            (L7, session affinity optional)              │
                    └────────────────────────┬────────────────────────────────┘
                                             │
                    ┌────────────────────────┼────────────────────────────────┐
                    │                        │                                │
           ┌────────▼────────┐    ┌──────────▼──────────┐    ┌───────────────▼┐
           │  API Gateway    │    │   API Gateway       │    │  API Gateway   │
           │  (Auth, Rate    │    │   (Region 2)        │    │  (Region 3)    │
           │   Limiting)     │    │                     │    │                │
           └────────┬────────┘    └──────────┬──────────┘    └───────────────┬┘
                    │                        │                               │
           ┌────────▼────────┐    ┌──────────▼──────────┐                   │
           │ Request Router  │    │  Request Router     │                   │
           │ (Model version, │    │                     │                   │
           │  A/B testing)   │    │                     │                   │
           └────────┬────────┘    └──────────┬──────────┘                   │
                    │                        │                               │
        ┌───────────┼───────────┐            │                               │
        │           │           │            │                               │
   ┌────▼───┐  ┌────▼───┐  ┌────▼───┐       │                               │
   │ Model  │  │ Model  │  │ Model  │       │                               │
   │Server 1│  │Server 2│  │Server 3│       │                               │
   │(TPU/GPU)│ │(TPU/GPU)│ │(TPU/GPU)│      │                               │
   └────────┘  └────────┘  └────────┘       │                               │
        │           │           │            │                               │
        └───────────┴───────────┴────────────┴───────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │    Model Registry     │
                    │  (Versions, Configs)  │
                    └───────────────────────┘
```

### Key Latency Components in LLM Serving

```
Total Latency Breakdown:
│
├── Network latency (client → server)
│   └── Typically 10-50ms depending on geography
│
├── Queue wait time
│   └── 0ms (healthy) to seconds (overloaded)
│
├── Preprocessing
│   ├── Tokenization: 1-5ms
│   └── Prompt processing: varies with length
│
├── Model inference
│   ├── Time to First Token (TTFT): 50-500ms
│   │   └── Depends on prompt length, model size
│   ├── Token generation: 20-50ms per token
│   │   └── Autoregressive, sequential
│   └── Total generation: (num_tokens × time_per_token)
│
├── Postprocessing
│   └── Detokenization, formatting: 1-5ms
│
└── Network latency (server → client)
    └── Typically 10-50ms

Example for 100-token response:
- TTFT: 200ms
- Generation: 100 × 30ms = 3000ms
- Total: ~3.2 seconds
```

### Optimizations for LLM Serving

**1. KV Cache Management**
```python
"""
KV Cache: Stores key-value pairs from attention layers
to avoid recomputation during autoregressive generation.

Problem: KV cache grows with sequence length and batch size
Memory = batch_size × seq_len × num_layers × 2 × hidden_dim × precision

For a 70B model with 4K context:
- Per request: ~8GB for KV cache alone
- Limits concurrent requests per GPU
"""

class KVCacheManager:
    def __init__(self, max_memory_gb, model_config):
        self.max_memory = max_memory_gb * 1e9
        self.cache_per_token = self._calculate_cache_per_token(model_config)
        self.active_caches = {}
    
    def can_accept_request(self, prompt_length, max_new_tokens):
        """Check if we have memory for this request's KV cache"""
        required = (prompt_length + max_new_tokens) * self.cache_per_token
        available = self.max_memory - self._current_usage()
        return required <= available
    
    def _calculate_cache_per_token(self, config):
        # 2 (K and V) × layers × hidden_dim × precision_bytes
        return 2 * config.num_layers * config.hidden_dim * 2  # bfloat16
```

**2. Batching Strategies**
```
Static Batching:
- Wait for N requests, process together
- Problem: One slow request blocks all
- Good for: Offline batch processing

Dynamic/Continuous Batching:
- Requests join/leave batch dynamically
- New requests added as old ones complete
- Much better GPU utilization
- Used by: vLLM, TensorRT-LLM, Triton

┌─────────────────────────────────────────────────────┐
│ Continuous Batching Example                         │
│                                                     │
│ Time →                                              │
│ Req1: [████████████████████]                       │
│ Req2:     [████████]                               │
│ Req3:         [██████████████████]                 │
│ Req4:              [████]                          │
│                                                     │
│ GPU processes all active requests each step        │
│ Requests exit when done, new ones join             │
└─────────────────────────────────────────────────────┘
```

**3. Speculative Decoding**
```
Problem: Autoregressive generation is slow (1 token at a time)

Solution: Use small "draft" model to guess multiple tokens,
          verify with large model in parallel

┌────────────┐     ┌────────────────┐
│ Draft Model│────▶│ Guess: "The   │
│ (Small)    │     │ cat sat on"   │
└────────────┘     └───────┬────────┘
                           │
                           ▼
┌────────────┐     ┌────────────────┐
│ Main Model │────▶│ Verify in one  │
│ (Large)    │     │ forward pass   │
└────────────┘     └───────┬────────┘
                           │
                    Accept 4/5 tokens
                    Reject 1, regenerate

Speedup: 2-3x for well-matched draft models
```

### Model Serving Reliability Patterns

**1. Graceful Degradation**
```python
class LLMServingWithDegradation:
    def __init__(self):
        self.models = {
            'primary': GeminiUltra(),      # Best quality
            'fallback1': GeminiPro(),      # Good quality, faster
            'fallback2': GeminiNano(),     # Fast, lower quality
        }
        self.circuit_breakers = {
            name: CircuitBreaker() for name in self.models
        }
    
    async def generate(self, prompt, quality_requirement='high'):
        # Try models in order based on quality requirement
        model_order = self._get_model_order(quality_requirement)
        
        for model_name in model_order:
            cb = self.circuit_breakers[model_name]
            
            if not cb.can_execute():
                continue  # Skip if circuit is open
            
            try:
                result = await asyncio.wait_for(
                    self.models[model_name].generate(prompt),
                    timeout=30.0
                )
                cb.record_success()
                return result, model_name
            
            except asyncio.TimeoutError:
                cb.record_failure()
                continue
            except Exception as e:
                cb.record_failure()
                continue
        
        # All models failed
        raise ServiceDegradedError("All models unavailable")
```

**2. Request Prioritization**
```python
class PriorityRequestQueue:
    """
    Different priority levels for different use cases
    - P0: Revenue-critical (paid API)
    - P1: User-facing products
    - P2: Internal tools
    - P3: Batch/async jobs
    """
    
    def __init__(self):
        self.queues = {
            'P0': asyncio.PriorityQueue(),
            'P1': asyncio.PriorityQueue(),
            'P2': asyncio.PriorityQueue(),
            'P3': asyncio.PriorityQueue(),
        }
        self.processing_ratio = {'P0': 0.5, 'P1': 0.3, 'P2': 0.15, 'P3': 0.05}
    
    async def get_next_request(self):
        """Weighted fair queuing across priority levels"""
        # Implementation of weighted fair queuing
        # Ensures lower priorities still get some processing
        pass
    
    def shed_load(self, current_latency_p99):
        """Drop lower priority requests when overloaded"""
        if current_latency_p99 > SLO_TARGET * 0.8:
            # Start rejecting P3
            self.queues['P3'].clear()
        if current_latency_p99 > SLO_TARGET * 0.9:
            # Also reject P2
            self.queues['P2'].clear()
```

---

## 1.3 LLM-Specific Scaling Patterns

### Why LLMs Scale Differently

```
Traditional Microservice:
- Stateless (mostly)
- Fast startup
- Linear scaling
- Memory: ~hundreds of MB
- Request time: milliseconds

LLM Service:
- Stateful (model weights, KV cache)
- Slow startup (model loading: minutes)
- Non-linear scaling (batch effects)
- Memory: tens to hundreds of GB
- Request time: seconds

Implications:
1. Can't spin up new instances quickly
2. Can't easily move requests between instances
3. Memory is the bottleneck, not CPU
4. Batch size affects efficiency non-linearly
```

### Scaling Strategies

**1. Horizontal Scaling with Constraints**
```yaml
# Kubernetes HPA for LLM - different from typical services

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gemini-serving-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gemini-serving
  minReplicas: 10          # High minimum - can't scale from 0 quickly
  maxReplicas: 100
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling up
      policies:
      - type: Pods
        value: 4                        # Add max 4 pods at a time
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 600  # Wait 10 min before scaling down
      policies:
      - type: Percent
        value: 10                       # Remove max 10% at a time
        periodSeconds: 120
  metrics:
  - type: External
    external:
      metric:
        name: queue_depth_per_replica
      target:
        type: AverageValue
        averageValue: "5"              # Target 5 requests queued per replica
```

**2. Pre-warming and Capacity Reservation**
```python
class CapacityManager:
    """
    Pre-warm instances before expected traffic spikes
    """
    
    def __init__(self, predictor, scaler):
        self.predictor = predictor  # Traffic prediction model
        self.scaler = scaler
        self.warm_up_time = 300  # 5 minutes to load model
    
    async def predictive_scaling(self):
        while True:
            # Predict traffic 10 minutes ahead
            predicted_traffic = self.predictor.predict(
                horizon_minutes=10
            )
            
            # Calculate required capacity
            required_replicas = self._calculate_replicas(predicted_traffic)
            
            # Add buffer for safety
            target_replicas = int(required_replicas * 1.3)
            
            # Scale up if needed (scale up is slow, so be proactive)
            current = self.scaler.get_current_replicas()
            if target_replicas > current:
                self.scaler.scale_to(target_replicas)
                logging.info(f"Pre-warming: {current} → {target_replicas}")
            
            await asyncio.sleep(60)  # Check every minute
```

**3. Model Sharding and Parallelism**
```
For very large models that don't fit on one accelerator:

Tensor Parallelism (TP):
- Split individual layers across devices
- Each device holds part of each layer
- All-reduce communication each layer
- Best for: inference latency

┌─────────┬─────────┬─────────┬─────────┐
│ Layer 1 │ Layer 1 │ Layer 1 │ Layer 1 │
│ Part A  │ Part B  │ Part C  │ Part D  │
├─────────┼─────────┼─────────┼─────────┤
│ Layer 2 │ Layer 2 │ Layer 2 │ Layer 2 │
│ Part A  │ Part B  │ Part C  │ Part D  │
├─────────┼─────────┼─────────┼─────────┤
│   ...   │   ...   │   ...   │   ...   │
└─────────┴─────────┴─────────┴─────────┘
  GPU 0     GPU 1     GPU 2     GPU 3


Pipeline Parallelism (PP):
- Split layers across devices
- Each device holds complete layers
- Pipeline the forward pass
- Best for: training throughput

┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Layers  │  │ Layers  │  │ Layers  │  │ Layers  │
│  1-10   │─▶│  11-20  │─▶│  21-30  │─▶│  31-40  │
│         │  │         │  │         │  │         │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
  GPU 0        GPU 1        GPU 2        GPU 3


Data Parallelism (DP):
- Full model on each device
- Split data across devices
- Gradient synchronization
- Best for: scaling batch size

┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Full    │  │ Full    │  │ Full    │  │ Full    │
│ Model   │  │ Model   │  │ Model   │  │ Model   │
│ Batch A │  │ Batch B │  │ Batch C │  │ Batch D │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
  GPU 0        GPU 1        GPU 2        GPU 3
```

### Monitoring LLM Systems

```yaml
# Comprehensive LLM monitoring dashboard

Latency Metrics:
  - time_to_first_token_p50: 150ms
  - time_to_first_token_p99: 500ms
  - token_generation_rate: 30 tokens/sec
  - end_to_end_latency_p50: 2s
  - end_to_end_latency_p99: 10s

Throughput Metrics:
  - requests_per_second: 1000
  - tokens_generated_per_second: 50000
  - batch_size_average: 32
  - gpu_utilization: 85%

Quality Metrics:
  - output_length_average: 150 tokens
  - output_length_p99: 500 tokens
  - empty_response_rate: 0.1%
  - safety_filter_trigger_rate: 0.5%

Resource Metrics:
  - gpu_memory_usage: 78%
  - kv_cache_hit_rate: 65%
  - queue_depth: 45
  - rejected_requests_rate: 0.01%

Cost Metrics:
  - cost_per_1k_tokens: $0.002
  - gpu_hours_per_day: 240
  - requests_per_gpu_hour: 500

Alerts:
  - TTFT p99 > 1s for 5 min
  - Queue depth > 100 for 2 min
  - GPU memory > 95%
  - Empty response rate > 1%
  - Error rate > 0.5%
```

---

# Section 2: Systematic Troubleshooting

---

## 2.1 The Troubleshooting Framework

### Think Out Loud Structure

When troubleshooting in an interview, use this structure:

```
1. CLARIFY (30 seconds)
   "Let me make sure I understand the problem..."
   - What's the symptom?
   - When did it start?
   - What's the impact?
   - Any recent changes?

2. HYPOTHESIZE (1 minute)
   "Based on this, my top hypotheses are..."
   - List 3-5 possible causes
   - Order by likelihood and impact
   - Explain your reasoning

3. INVESTIGATE (bulk of time)
   "I'll start by checking X because..."
   - Follow the data, not hunches
   - Eliminate hypotheses systematically
   - Adjust priorities based on findings

4. MITIGATE (as soon as possible)
   "While investigating, I'd also consider..."
   - Can we rollback?
   - Can we failover?
   - Can we shed load?

5. COMMUNICATE (throughout)
   "At this point I would update stakeholders..."
   - Regular status updates
   - Clear impact statements
   - ETA if possible
```

### The Hierarchy of Investigation

```
Start broad, narrow down:

┌─────────────────────────────────────────────────────────┐
│ 1. IS IT REALLY A PROBLEM?                              │
│    - Verify the alert/report                            │
│    - Check multiple data sources                        │
│    - Rule out false positives                           │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 2. WHAT'S THE SCOPE?                                    │
│    - All users or subset?                               │
│    - All endpoints or specific?                         │
│    - All regions or specific?                           │
│    - Started suddenly or gradually?                     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 3. WHAT CHANGED?                                        │
│    - Recent deployments?                                │
│    - Config changes?                                    │
│    - Traffic patterns?                                  │
│    - Dependency updates?                                │
│    - Infrastructure changes?                            │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 4. WHERE IS THE PROBLEM?                                │
│    - Client? Network? Load balancer?                    │
│    - Application? Database? Cache?                      │
│    - Upstream service? Downstream service?              │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│ 5. WHY IS IT HAPPENING?                                 │
│    - Resource exhaustion?                               │
│    - Bug in code?                                       │
│    - Misconfiguration?                                  │
│    - External failure?                                  │
└─────────────────────────────────────────────────────────┘
```

---

## 2.2 Practice Scenarios with Talk-Tracks

### Scenario 1: High Latency on LLM API

**Setup**: "The Gemini API latency has increased from p99 of 2s to p99 of 15s over the last 30 minutes. User complaints are coming in."

**Talk-Track**:

```
CLARIFY:
"First, let me understand the scope. Is this affecting all requests or specific 
ones? All regions or specific regions? I'd also want to know if there were any
deployments in the last hour."

INTERVIEWER: "It's affecting all regions, all request types."

INITIAL HYPOTHESES:
"Given it's global and gradual, my top hypotheses are:
1. Increased traffic beyond capacity
2. A dependency slowdown (model serving, database)
3. A bad deployment that's gradually rolling out
4. Resource exhaustion (memory, GPU)
5. Network issues at the infrastructure level"

INVESTIGATION:

"I'll start with the quickest checks:

Step 1: Check traffic volume
- 'Is traffic higher than normal?'
- If yes → capacity issue, scale up
- If no → continue

Let's say traffic is normal...

Step 2: Check for recent deployments
- 'What was deployed in the last 2 hours?'
- If yes → candidate for rollback
- Check canary metrics if available

Let's say there was a model update 45 minutes ago...

Step 3: Correlate with deployment
- 'Does the latency increase correlate with rollout percentage?'
- Check metrics before/after by server cohort
- 'Are servers with new version slower?'

MITIGATE:
"At this point, I'd recommend:
1. Halt the rollout immediately
2. Start rollback if correlation is strong
3. Keep a few instances on new version for debugging

While rollback proceeds, I'd also:
- Check what changed in the model update
- Compare inference times old vs new
- Look at memory/GPU utilization differences"

RESOLUTION:
"Let's say we confirmed the new model version is 10x slower. We complete the
rollback and latency recovers. 

Post-incident, we'd want to understand:
- Why wasn't this caught in testing?
- Was there a performance regression in the model?
- Do we need better pre-production performance testing?"
```

### Scenario 2: Cascading Failures

**Setup**: "Multiple services are reporting errors. Started with Service A, now B, C, D are also failing. Error rates climbing."

**Talk-Track**:

```
CLARIFY:
"This sounds like a cascading failure. Let me understand the dependencies.
Does A depend on B, C, D? Or do they all depend on A?"

INTERVIEWER: "B, C, D all depend on A. A is a core authentication service."

IMMEDIATE ACTION:
"First priority is stopping the cascade. I'd:
1. Check if A can be quickly recovered
2. If not, can B, C, D fail open (serve without auth temporarily)?
3. Consider emergency traffic shedding

INVESTIGATION:

Step 1: Focus on root cause - Service A
- 'What's Service A's current state?'
- Error logs, metrics, resource utilization

Let's say A is returning 503s and CPU is at 100%...

Step 2: Why is A overloaded?
- Traffic spike to A?
- A's dependencies failing?
- Resource leak?

Let's say traffic to A is 10x normal...

Step 3: Why the traffic spike?
- Is this real user traffic?
- Or is it retry storms from B, C, D?

INTERVIEWER: 'It's retry storms. B, C, D have aggressive retries.'

ROOT CAUSE IDENTIFIED:
"Classic retry storm. Here's what happened:
1. A had a brief hiccup
2. B, C, D started retrying
3. Retries overwhelmed A
4. A failed more, causing more retries
5. Cascading failure

IMMEDIATE FIXES:
1. Disable retries temporarily on B, C, D (if possible via config)
2. Add emergency rate limiting at A's load balancer
3. Scale up A aggressively
4. Once A recovers, gradually re-enable retries

LONG-TERM FIXES:
1. Implement exponential backoff with jitter on all clients
2. Add circuit breakers on B, C, D
3. Add admission control/load shedding on A
4. Set up retry budget (max % of requests that can be retries)
5. Add dependency health checks and alerts"
```

### Scenario 3: Memory Leak in Production

**Setup**: "A service's memory grows 1GB per hour until OOM after 12 hours. We've been restarting it every 10 hours as a workaround."

**Talk-Track**:

```
CLARIFY:
"Let me understand the pattern:
- Is this consistent across all instances?
- Did this start after a specific deployment?
- What's the normal memory baseline?"

INTERVIEWER: "Started 2 weeks ago, affects all instances, baseline should be 4GB"

NARROW DOWN:
"So something changed 2 weeks ago. I'd check:
- Git history: what was deployed 2 weeks ago?
- Dependency updates?
- Traffic pattern changes?
- New features enabled?

Let's say we find a feature flag was enabled 2 weeks ago...

INVESTIGATION APPROACH:

Step 1: Confirm correlation
- Disable feature flag on one instance
- Monitor memory - does leak stop?

Let's say yes, leak stops...

Step 2: Examine the feature code
- What objects does it create?
- Are there caches without eviction?
- Event listeners being added?
- Connection pools?

Step 3: Get heap dumps
- Take dumps at startup, 4 hours, 8 hours
- Compare object counts
- Identify growing objects

EXAMPLE FINDING:
'We see RequestContext objects growing from 1000 to 50000 over 8 hours.
They're not being garbage collected.'

Step 4: Code review
- Find where RequestContext is created
- Find where it should be cleaned up
- Identify the missing cleanup

EXAMPLE ROOT CAUSE:
'The new feature adds requests to a debugging cache for tracing, but
the cache has no TTL or size limit.'

FIX:
1. Short-term: Disable feature flag
2. Medium-term: Add TTL/size limit to cache
3. Long-term: 
   - Add memory leak detection tests
   - Monitor object counts in production
   - Add memory growth rate alerts"
```

### Scenario 4: Database Connection Exhaustion

**Setup**: "Application errors spiking. Logs show 'connection pool exhausted'. Database seems fine."

**Talk-Track**:

```
CLARIFY:
"Connection pool exhausted means we're using all available connections.
Let me understand:
- What's the pool size configured?
- What's the database's max connections?
- Is this a sudden spike or gradual?"

INTERVIEWER: "Pool is 100 connections per app instance, 10 instances = 1000 total.
Database max is 2000. Happened suddenly 10 minutes ago."

HYPOTHESES:
"If pool is exhausted but DB has capacity, connections aren't being returned.
Possible causes:
1. Slow queries holding connections
2. Deadlocks
3. Transactions not being committed/rolled back
4. Connection leak in code
5. Sudden traffic spike"

INVESTIGATION:

Step 1: Check query performance
- 'Are there long-running queries?'
- Check database slow query log
- Check active transactions

INTERVIEWER: 'There are queries running for 5+ minutes, normally they take 50ms'

Step 2: Identify the slow queries
- What tables? What queries?
- When did they start being slow?
- Are they blocked on locks?

INTERVIEWER: 'It's SELECT queries on the users table, waiting on table lock'

Step 3: Find the lock holder
- Who holds the lock?
- Is there a long-running transaction?

INTERVIEWER: 'There's an ALTER TABLE running on users table from 15 minutes ago'

ROOT CAUSE:
"Someone ran ALTER TABLE on a large production table. In MySQL (with certain
storage engines), this locks the table. All queries queue up, exhaust the
connection pool, and the app fails."

MITIGATION:
1. Immediate: Kill the ALTER TABLE if possible
2. If can't kill: 
   - Failover to replica for reads
   - Increase connection pool timeout to shed load
   - Return errors gracefully to users

PREVENTION:
1. Use pt-online-schema-change for schema changes
2. Restrict DDL permissions in production
3. Require schema change review process
4. Monitor for long-running transactions
5. Add connection pool exhaustion alerting"
```

### Scenario 5: Mysterious Network Timeouts

**Setup**: "Intermittent timeouts between Service A and Service B. Happens about 5% of requests. Both services show normal CPU/memory."

**Talk-Track**:

```
CLARIFY:
"Intermittent issues are tricky. I need to understand:
- Is the timeout consistent (always 30s) or variable?
- Is it random or patterns (certain times, certain requests)?
- Are A and B in the same datacenter/zone?"

INTERVIEWER: "Timeout is always 30s (the configured timeout). Seems random.
Same zone. Started happening yesterday."

HYPOTHESES:
"Intermittent network issues in same zone:
1. DNS resolution problems
2. Load balancer issues
3. Connection pooling/reuse problems
4. Packet loss at network level
5. Service B occasionally slow (but not showing in averages)
6. TCP connection state issues"

INVESTIGATION:

Step 1: Check if requests reach Service B
- Look at B's access logs
- Do 5% of requests never arrive? Or arrive and time out?

INTERVIEWER: 'Requests arrive at B, B processes them in 100ms, but A times out'

KEY INSIGHT:
"So B responds quickly but A doesn't receive it. The problem is in the
response path, not the request path."

Step 2: Network-level debugging
- Check for packet retransmissions
- Look at TCP connection states
- Check for network errors

Let's say we find high retransmission rates on certain connections...

Step 3: Identify pattern
- Which instances are affected?
- Is it specific IPs?
- Is it specific load balancer nodes?

INTERVIEWER: 'It's always requests through load balancer node LB-3'

Step 4: Investigate LB-3
- Check LB-3's metrics
- Check LB-3's network interface
- Compare config with other LB nodes

INTERVIEWER: 'LB-3 has 0.1% packet loss on its network interface'

ROOT CAUSE:
"LB-3 has a failing network interface causing packet loss. With TCP retries
and timeouts, 0.1% packet loss causes ~5% of requests to exceed timeout."

FIX:
1. Immediate: Drain LB-3 from rotation
2. Replace/repair the network interface
3. Improve monitoring:
   - Alert on per-node packet loss
   - Alert on per-node timeout rates
   - Add retransmission metrics"
```

---

## 2.3 Common Investigation Commands

### Quick Reference for Interviews

```bash
# "First, I'd check system resource utilization"
top -bn1 | head -20          # CPU, memory snapshot
vmstat 1 5                    # Virtual memory over time
iostat -x 1 5                 # Disk I/O
free -h                       # Memory summary

# "Let me check network connectivity"
ping -c 3 service-b           # Basic connectivity
traceroute service-b          # Path analysis
curl -w "@timing.txt" url     # HTTP timing breakdown
nc -zv host 443               # Port check

# "I'll look at network connections"
ss -tuln                      # Listening ports
ss -tn state established      # Established connections
ss -tn state time-wait | wc   # TIME_WAIT count
netstat -an | grep :443       # Connections to port 443

# "Let me check the application logs"
journalctl -u myservice -f    # Follow service logs
tail -f /var/log/app.log | grep -i error
dmesg -T | tail               # Kernel messages

# "I'll look at process details"
ps aux --sort=-%cpu | head    # Top CPU consumers
ps aux --sort=-%mem | head    # Top memory consumers
lsof -p <pid>                 # Open files/connections
strace -p <pid> -f -e trace=network  # Network syscalls

# "Let me check disk usage"
df -h                         # Filesystem usage
du -sh /* | sort -h           # Space by directory
lsof +D /path                 # Open files in directory
iotop -oP                     # IO by process

# "I'll capture network traffic"
tcpdump -i eth0 port 443 -c 100 -w capture.pcap
tcpdump -i any host 10.0.0.5

# "Check for connection issues"
curl -sS --connect-timeout 5 http://service-b/health
curl -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" -o /dev/null -s http://service-b/
```

---

# Section 3: Google SRE Principles Deep Dive

---

## 3.1 Core Philosophy

### The 50% Rule

```
SRE Time Allocation:

┌─────────────────────────────────────────────────────┐
│                                                     │
│  ████████████████████████  ████████████████████████│
│        ≤50% Toil               ≥50% Engineering    │
│                                                     │
│  - On-call response            - Automation        │
│  - Manual tasks                - Tool building     │
│  - Ticket work                 - System design     │
│  - Repetitive ops              - Reliability work  │
│                                                     │
└─────────────────────────────────────────────────────┘

If toil exceeds 50%:
1. Team is understaffed
2. Or automation opportunities exist
3. Or service is too unreliable

Actions:
- Redirect tickets to dev team
- Pause new service onboarding
- Focus sprint on automation
- Push back on manual requests
```

### Error Budgets in Practice

```
Example: Service with 99.9% SLO (43.2 min/month budget)

Month Timeline:
Day 1-5:   [████████████████████████████████] Budget: 100%
Day 6:     Deployment causes 10 min outage
           [████████████████████████░░░░░░░░] Budget: 77%
Day 7-15:  Small incidents total 5 min
           [██████████████████████░░░░░░░░░░] Budget: 65%
Day 16:    Major outage: 20 min
           [████████████░░░░░░░░░░░░░░░░░░░░] Budget: 19%
Day 17+:   BUDGET CRITICAL

When budget is low:
- Freeze non-critical deployments
- Require extra review for changes
- Focus on reliability improvements
- Postmortem any new incidents

When budget is healthy:
- Deploy more frequently
- Run experiments
- Take on riskier improvements
- Pay down tech debt
```

### Error Budget Policy Example

```yaml
# Error Budget Policy Document

service: gemini-api
slo: 99.9% availability
budget_period: rolling_30_days

thresholds:
  healthy:
    budget_remaining: ">50%"
    actions:
      - normal_deployment_velocity
      - can_run_experiments
      - innovation_projects_allowed
  
  caution:
    budget_remaining: "25-50%"
    actions:
      - reduce_deployment_frequency
      - require_extra_rollout_monitoring
      - pause_non_critical_experiments
      
  critical:
    budget_remaining: "10-25%"
    actions:
      - freeze_non_essential_deployments
      - require_director_approval_for_changes
      - all_hands_reliability_focus
      
  exhausted:
    budget_remaining: "<10%"
    actions:
      - complete_deployment_freeze
      - incident_review_required
      - executive_escalation
      - mandatory_postmortem

escalation:
  budget_burn_rate_alert: 
    condition: "consuming >5% budget in 1 hour"
    action: page_oncall
  
  budget_projection_alert:
    condition: "projected to exhaust in <7 days"
    action: notify_team_lead
```

---

## 3.2 Toil: Understanding and Eliminating

### Identifying Toil

```
Toil Characteristics (ALL must apply):
☑ Manual - requires human action
☑ Repetitive - done over and over
☑ Automatable - could be done by software
☑ Reactive - triggered by external events
☑ No enduring value - doesn't improve service
☑ Scales with service - grows with usage

NOT Toil:
✗ Project work (building new systems)
✗ Documentation (enduring value)
✗ Meetings (not repetitive in same way)
✗ Design reviews (adds value)
✗ Learning/training (improves capability)
```

### Toil Tracking Framework

```python
class ToilTracker:
    """
    Track and prioritize toil elimination
    """
    
    def __init__(self):
        self.toil_items = []
    
    def add_toil(self, item):
        """
        item = {
            'name': 'Manual certificate renewal',
            'frequency': 'weekly',
            'time_per_occurrence': 30,  # minutes
            'occurrences_per_month': 4,
            'people_required': 1,
            'automation_effort': 'medium',  # low/medium/high
            'risk_if_missed': 'high',       # low/medium/high
        }
        """
        item['monthly_minutes'] = (
            item['time_per_occurrence'] * 
            item['occurrences_per_month'] * 
            item['people_required']
        )
        self.toil_items.append(item)
    
    def prioritize(self):
        """
        Priority = time_saved / effort_required
        """
        effort_multiplier = {'low': 1, 'medium': 2, 'high': 4}
        
        for item in self.toil_items:
            effort = effort_multiplier[item['automation_effort']]
            annual_hours = (item['monthly_minutes'] * 12) / 60
            item['roi_score'] = annual_hours / effort
        
        return sorted(self.toil_items, 
                     key=lambda x: x['roi_score'], 
                     reverse=True)
```

### Common Toil Patterns and Solutions

```
Pattern                      Solution
─────────────────────────────────────────────────────────────
Manual deployments    →      CI/CD pipelines, GitOps
Certificate renewal   →      cert-manager, auto-rotation
Access requests       →      Self-service with approval workflow
Capacity scaling      →      Auto-scaling policies
Log analysis          →      Automated anomaly detection
Incident response     →      Runbook automation, self-healing
Config changes        →      Configuration management (Terraform)
Security patches      →      Automated patching, immutable infra
Data backups          →      Automated backup verification
Report generation     →      Scheduled jobs, dashboards
```

---

## 3.3 SLOs: Getting Them Right

### SLO Design Principles

```
Good SLOs:
✓ Measure user experience, not system internals
✓ Are achievable but ambitious
✓ Leave room for error budget
✓ Are measurable with existing tooling
✓ Are understandable by all stakeholders

Bad SLOs:
✗ "99.999% uptime" (too aggressive, no room for change)
✗ "CPU < 80%" (system metric, not user experience)
✗ "No bugs" (unmeasurable)
✗ "Fast responses" (not quantified)
```

### SLI/SLO Examples for Gemini API

```yaml
# Availability SLI/SLO
availability:
  sli: |
    The proportion of valid requests that return 
    a successful response (non-5xx)
  
  measurement: |
    sum(rate(requests{status!~"5.."}[5m])) / 
    sum(rate(requests[5m]))
  
  slo: 99.9%
  
  exclusions:
    - Requests rejected due to rate limiting (429)
    - Invalid API keys (401)
    - Malformed requests (400)

# Latency SLI/SLO  
latency:
  sli: |
    The proportion of requests that complete within 
    the latency threshold
  
  measurement: |
    histogram_quantile(0.99, 
      rate(request_duration_bucket[5m]))
  
  slos:
    - p50: 500ms   # Interactive use
    - p95: 2s
    - p99: 5s      # Long-tail acceptable
  
  notes: |
    Latency measured from request received to 
    last byte of response sent. Excludes client
    network time.

# Throughput SLI (for capacity planning)
throughput:
  sli: |
    Requests per second the system can handle 
    while meeting latency SLOs
  
  target: 10000 RPS at p99 < 5s
  
  headroom: 30% above peak traffic

# Quality SLI (LLM-specific)
quality:
  sli: |
    Proportion of responses that pass quality checks
  
  checks:
    - Non-empty response
    - Response coherence (automated eval)
    - Safety filters not triggered erroneously
  
  slo: 99.5%
```

### Multi-Window, Multi-Burn-Rate Alerts

```yaml
# Alert when burning budget too fast

# Fast burn (2% of 30-day budget in 1 hour)
# Catches major outages quickly
- alert: HighErrorRateSevere
  expr: |
    (
      1 - (sum(rate(requests_success[1h])) / 
           sum(rate(requests_total[1h])))
    ) > 14.4 * 0.001  # 14.4x burn rate
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning very fast"
    description: "At this rate, 30-day budget exhausted in 5 hours"

# Slow burn (5% of 30-day budget in 6 hours)
# Catches slow degradations
- alert: HighErrorRateSlow
  expr: |
    (
      1 - (sum(rate(requests_success[6h])) / 
           sum(rate(requests_total[6h])))
    ) > 1 * 0.001  # 1x burn rate sustained
  for: 30m
  labels:
    severity: warning
  annotations:
    summary: "Error budget burning steadily"
    description: "At this rate, 30-day budget exhausted in 30 days"
```

---

## 3.4 On-Call Best Practices

### On-Call Structure

```
Rotation Design:
├── Primary on-call (first responder)
├── Secondary on-call (backup, escalation)
└── Tertiary/Expert escalation (deep knowledge)

Shift Length:
- 12-hour shifts (day/night) or
- 24-hour shifts (full day)
- Never more than 1 week continuous

Handoff Requirements:
- Document ongoing incidents
- Note any known issues
- Share context on recent changes
- Review upcoming deployments
```

### Page Quality Standards

```
Good Pages:
✓ Actionable - on-call can do something
✓ Urgent - needs immediate attention
✓ Novel - not already known
✓ Real - represents actual user impact

Page Hygiene Metrics:
- Pages per shift: Target <2
- Actionable rate: Target >80%
- False positive rate: Target <10%
- Time to acknowledge: Target <5 min
- Time to mitigate: Track but no hard target

Page Anti-Patterns:
✗ "CPU high" - Not actionable, no user impact
✗ "Disk 80% full" - Not urgent, should be ticket
✗ Multiple pages for same issue - Dedup needed
✗ Pages for expected behavior - Alert tuning needed
```

### Incident Documentation Template

```markdown
# Incident Report: [Title]

## Summary
- **Date**: YYYY-MM-DD
- **Duration**: X hours Y minutes
- **Severity**: P1/P2/P3
- **Impact**: [User-facing impact description]

## Timeline (all times UTC)
- HH:MM - Alert fired
- HH:MM - On-call acknowledged
- HH:MM - [Investigation step]
- HH:MM - Mitigation applied
- HH:MM - Service recovered
- HH:MM - Root cause identified

## Root Cause
[Clear explanation of what caused the incident]

## Resolution
[What was done to fix it]

## Detection
- How was the incident detected?
- Could we have detected it faster?

## Action Items
| Action | Owner | Priority | Due Date |
|--------|-------|----------|----------|
| [Action 1] | @name | P1 | YYYY-MM-DD |
| [Action 2] | @name | P2 | YYYY-MM-DD |

## Lessons Learned
- What went well?
- What could have gone better?
- Where did we get lucky?

## Supporting Data
[Links to dashboards, logs, etc.]
```

---

## 3.5 Release Engineering

### Deployment Strategies

```
Rolling Update:
├── Replace instances one at a time
├── Good for: Small changes, stateless services
├── Risk: Slow rollback
└── Config: maxUnavailable=1, maxSurge=1

Blue-Green:
├── Two identical environments
├── Switch traffic instantly
├── Good for: Database migrations, risky changes
├── Risk: 2x resource cost
└── Fast rollback: Just switch back

Canary:
├── Route small % to new version
├── Gradually increase if healthy
├── Good for: Most changes, large scale
├── Risk: Need good monitoring
└── Rollback: Route traffic away

┌─────────────────────────────────────────────────────┐
│              Canary Rollout Example                 │
│                                                     │
│  Stage 1: 1% traffic for 10 min                    │
│  Stage 2: 10% traffic for 30 min                   │
│  Stage 3: 50% traffic for 1 hour                   │
│  Stage 4: 100% traffic                              │
│                                                     │
│  At each stage, check:                              │
│  - Error rate < baseline + 0.1%                     │
│  - Latency p99 < baseline * 1.1                     │
│  - No new error types                               │
│                                                     │
│  If any check fails: Auto-rollback                  │
└─────────────────────────────────────────────────────┘
```

### Deployment Checklist

```markdown
## Pre-Deployment
- [ ] Code reviewed and approved
- [ ] Tests passing (unit, integration, e2e)
- [ ] Load tested if significant change
- [ ] Rollback plan documented
- [ ] Monitoring dashboards ready
- [ ] Team aware of deployment
- [ ] Error budget checked (sufficient headroom?)

## During Deployment
- [ ] Canary metrics monitored
- [ ] No unexpected errors
- [ ] Latency within bounds
- [ ] Resource usage acceptable
- [ ] Log anomalies checked

## Post-Deployment
- [ ] All instances healthy
- [ ] Metrics stable for 30 min
- [ ] No user complaints
- [ ] Deployment documented
- [ ] On-call briefed if needed
```

---

## 3.6 Capacity Planning

### Demand Forecasting

```python
class CapacityPlanner:
    """
    Basic capacity planning framework
    """
    
    def __init__(self, historical_data):
        self.data = historical_data
    
    def forecast_demand(self, days_ahead=90):
        """
        Forecast future demand based on:
        - Historical growth trends
        - Seasonality (daily, weekly, yearly)
        - Known events (launches, campaigns)
        """
        
        # Components of demand
        trend = self._calculate_growth_trend()
        seasonality = self._calculate_seasonality()
        events = self._get_planned_events(days_ahead)
        
        return {
            'baseline': trend * (1 + seasonality),
            'peak': trend * self._peak_to_average_ratio(),
            'with_events': self._add_event_spikes(trend, events)
        }
    
    def recommend_capacity(self, forecast):
        """
        Recommend capacity with buffer for:
        - Normal variance: 20%
        - Failure headroom: 33% (N+1, N+2)
        - Burst capacity: 2x normal peak
        """
        
        peak_demand = forecast['peak']
        
        return {
            'minimum': peak_demand * 1.2,           # 20% buffer
            'recommended': peak_demand * 1.5,       # 50% buffer
            'n_plus_one': peak_demand * 1.5 * 1.33, # Can lose 1 zone
            'burst': peak_demand * 2.0              # Handle 2x spike
        }
```

### Load Testing Guidelines

```yaml
# Load Test Requirements

test_types:
  baseline:
    description: "Current production load"
    duration: 30 minutes
    success_criteria:
      - latency_p99 < slo_target
      - error_rate < 0.1%
      - no resource exhaustion
  
  peak:
    description: "Expected peak load"
    duration: 1 hour
    load: 2x baseline
    success_criteria:
      - latency_p99 < slo_target * 1.2
      - error_rate < 0.5%
  
  stress:
    description: "Find breaking point"
    type: ramp up until failure
    success_criteria:
      - identify failure mode
      - graceful degradation (not crash)
      - recovery when load decreases
  
  soak:
    description: "Long-running stability"
    duration: 24+ hours
    load: baseline
    success_criteria:
      - no memory leaks
      - no connection leaks
      - stable latency over time

before_production:
  - Run all test types on staging
  - Test with production-like data
  - Include dependency failures
  - Test rollback procedure
```

---

## 3.7 Postmortem Culture

### Blameless Postmortem Principles

```
The Goal:
Learn from incidents, not punish individuals

Key Principles:
1. Assume good intentions
   - People made the best decisions with available info
   
2. Focus on systems, not individuals
   - "The process allowed this" not "Person X did this"
   
3. Multiple contributing factors
   - Incidents rarely have single root cause
   
4. Counterfactual thinking
   - "What would need to be different?" not "Who should have..."

5. Forward-looking
   - Action items prevent future incidents
   - No action item = wasted postmortem

Language Matters:
Instead of: "John deployed bad code"
Say: "The deployment pipeline allowed code that caused..."

Instead of: "Nobody noticed the alert"
Say: "The alert wasn't surfaced effectively to responders"
```

### 5 Whys Example

```
Incident: API returned errors for 30 minutes

Why 1: Why did the API return errors?
→ The database connection pool was exhausted

Why 2: Why was the connection pool exhausted?
→ Queries were taking 10x longer than normal

Why 3: Why were queries slow?
→ A new query pattern caused full table scans

Why 4: Why did the query cause table scans?
→ Missing index for the new query pattern

Why 5: Why was the index missing?
→ No process to review query plans for new features

Root Cause: Missing process for query plan review

Action Items:
1. Add the missing index (immediate)
2. Implement query plan review in code review (prevent)
3. Add slow query alerting (detect)
4. Add connection pool usage alerting (detect)
```

---

# Quick Reference Cards

## Card 1: Incident Severity

| Sev | User Impact | Response | Examples |
|-----|-------------|----------|----------|
| 1 | Complete outage | Page everyone, all hands | API down, data loss |
| 2 | Major degradation | Page on-call + backup | 50% errors, major feature broken |
| 3 | Minor impact | On-call during hours | Slow responses, minor feature broken |
| 4 | Minimal | Normal ticket | Cosmetic issues |

## Card 2: Availability Math

| SLO | Annual Downtime | Monthly | Weekly |
|-----|-----------------|---------|--------|
| 99% | 3.65 days | 7.3 hours | 1.68 hours |
| 99.9% | 8.76 hours | 43.8 min | 10.1 min |
| 99.95% | 4.38 hours | 21.9 min | 5 min |
| 99.99% | 52.6 min | 4.38 min | 1 min |
| 99.999% | 5.26 min | 26.3 sec | 6 sec |

## Card 3: Troubleshooting Flow

```
Alert → Verify → Scope → Change? → Mitigate → Debug → Fix → Postmortem
  │        │       │        │          │         │      │        │
  v        v       v        v          v         v      v        v
Real?   Impact  Broad/   Deploy/   Rollback/  Root   Patch  Prevent
        users   narrow   config?   failover   cause         future
```

## Card 4: Golden Signals

| Signal | What to Measure | Alert When |
|--------|-----------------|------------|
| Latency | Request duration (success & error) | p99 > SLO |
| Traffic | Requests/sec, bytes/sec | Unusual patterns |
| Errors | Error rate, error types | Rate > threshold |
| Saturation | Resource usage, queue depth | > 80% capacity |

---

*Good luck with your interview! Remember: think out loud, be systematic, and show your reasoning.*
