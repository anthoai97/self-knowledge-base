# LLM Inference Cost for a Normal Agentic Application

Based on NVIDIA's article: [LLM Inference Benchmarking: How Much Does Your LLM Inference Cost?](https://developer.nvidia.com/blog/llm-inference-benchmarking-how-much-does-your-llm-inference-cost/)

This example follows the article's general sizing approach:

1. Benchmark latency and throughput first.
2. Pick the highest-throughput deployment that still meets latency requirements.
3. Size the number of model instances from peak request volume.
4. Convert required instances into GPUs and servers.
5. Calculate annual infrastructure cost and cost per request/token.

The article discusses metrics such as:

- **TTFT**: Time to first token
- **ITL**: Inter-token latency
- **TPS**: Tokens per second
- **RPS**: Requests per second

It also notes that peak **requests per second** is not the same as concurrent users, because not all users are actively sending a request at the same time.

---

## 1. Application

Assume an enterprise has an internal customer-support agent that can:

- Search documentation
- Call tools
- Retrieve relevant ticket/customer context
- Summarize findings
- Draft a final answer

This is a typical agentic application because one user-facing request may trigger multiple LLM calls behind the scenes.

---

## 2. Normal Agentic Traffic Pattern

| Traffic assumption | Example value |
|---|---:|
| Daily active users | 10,000 |
| Agent runs per user per day | 3 |
| Total agent runs per day | 30,000 |
| LLM calls per agent run | 4 |
| Total LLM requests per day | 120,000 |
| Peak-hour share of daily requests | 20% |
| Peak LLM request rate | 6.7 requests/sec |

A single **agent run** may look like this:

| Step | LLM calls | Avg input tokens | Avg output tokens |
|---|---:|---:|---:|
| Plan task | 1 | 1,200 | 150 |
| Tool/RAG reasoning | 2 | 3,500 each | 250 each |
| Final answer | 1 | 4,500 | 650 |
| **Total per agent run** | **4** | **12,700** | **1,300** |

So each agent run uses about:

```text
12,700 input tokens + 1,300 output tokens = 14,000 combined tokens
```

Daily volume:

| Metric | Value |
|---|---:|
| Agent runs/day | 30,000 |
| LLM requests/day | 120,000 |
| Combined tokens/day | 420 million |
| Combined tokens/year | 153.3 billion |
| LLM requests/year | 43.8 million |

---

## 3. Benchmark Result to Plug Into NVIDIA's Method

The NVIDIA article recommends using benchmark data to find the best latency-throughput point.

For example, you might choose the highest-throughput deployment that still satisfies a latency target such as:

```text
TTFT <= 250 ms
```

Assume your benchmark result for the chosen model and sequence length is:

| Benchmark result | Example value |
|---|---:|
| Latency target | TTFT <= 250 ms |
| Throughput per model instance | 2.5 requests/sec |
| GPUs per model instance | 2 GPUs |
| GPUs per server | 8 GPUs |

Now size for peak traffic:

```text
Peak request rate = 120,000 requests/day × 20% / 3,600 seconds
                  = 6.7 requests/second

Required instances = ceil(6.7 / 2.5)
                   = 3 model instances

Required GPUs = 3 instances × 2 GPUs
              = 6 GPUs

Required servers = ceil(6 GPUs / 8 GPUs per server)
                 = 1 server
```

So this example workload requires:

| Capacity item | Required amount |
|---|---:|
| Model instances | 3 |
| GPUs | 6 |
| Servers | 1 |

---

## 4. TCO Using the Article's Example Cost Structure

The NVIDIA article's example cost inputs include:

| Cost item | Example value |
|---|---:|
| Server cost | $320,000 |
| GPUs per server | 8 |
| Depreciation period | 4 years |
| Yearly hosting cost | $3,000 |
| Yearly software licensing | $4,500 |

Annual server cost:

```text
Yearly server cost
= initial server cost / depreciation period
  + yearly hosting cost
  + yearly software cost

= $320,000 / 4 + $3,000 + $4,500
= $87,500/year
```

Since the example workload needs **1 server**, the annual infrastructure cost is:

```text
Total yearly cost = 1 × $87,500
                  = $87,500/year
```

---

## 5. Cost per Agent Run, Request, and Token

Using the expected yearly traffic:

```text
Agent runs/year = 30,000 × 365
                = 10.95 million agent runs/year

LLM requests/year = 120,000 × 365
                  = 43.8 million LLM requests/year

Combined tokens/year = 420 million × 365
                     = 153.3 billion tokens/year
```

Cost per 1,000 LLM requests:

```text
Cost per 1,000 LLM requests
= $87,500 / 43.8M requests × 1,000
≈ $2.00 per 1,000 LLM requests
```

Cost per agent run:

```text
Cost per agent run
= $87,500 / 10.95M agent runs
≈ $0.008 per agent run
```

Cost per 1 million combined tokens:

```text
Cost per 1M combined tokens
= $87,500 / 153.3B tokens × 1,000,000
≈ $0.57 per 1M combined input + output tokens
```

Summary:

| Unit | Estimated cost |
|---|---:|
| 1,000 LLM requests | ~$2.00 |
| 1 agent run | ~$0.008 |
| 1M combined tokens | ~$0.57 |
| Annual infrastructure cost | ~$87,500 |

---

## 6. Input and Output Token Cost Split

The NVIDIA article also discusses separating input-token and output-token costs because output tokens are usually more expensive to generate.

It gives a reference ratio of:

```text
$1 input : $3 output
```

In this example:

```text
Input tokens per agent run  = 12,700
Output tokens per agent run = 1,300

Input tokens/year  = 12,700 × 10.95M
                   = 139.1B

Output tokens/year = 1,300 × 10.95M
                   = 14.2B
```

Using the 1:3 input/output cost ratio, the effective internal cost becomes approximately:

| Token type | Effective cost |
|---|---:|
| 1M input tokens | ~$0.48 |
| 1M output tokens | ~$1.44 |

---

## 7. Important Caveat

This is a **capacity-based self-hosting example**, not an API-provider billing example.

Because the server cost is mostly fixed, the cost per token depends heavily on utilization.

In this example, the app provisions for a **6.7 requests/sec peak**, but the average daily traffic is much lower. If the same hardware is shared across more applications or kept busier, the cost per token drops. If the workload is more idle, the cost per token rises.

---

## 8. Final Takeaway

For a normal internal agentic application with:

- 10,000 daily active users
- 3 agent runs per user per day
- 4 LLM calls per agent run
- 120,000 LLM requests per day
- 420 million combined tokens per day
- 6.7 peak requests/sec

The estimated self-hosted inference cost is:

| Metric | Estimate |
|---|---:|
| Required servers | 1 |
| Annual infrastructure cost | ~$87,500 |
| Cost per 1,000 LLM requests | ~$2.00 |
| Cost per agent run | ~$0.008 |
| Cost per 1M combined tokens | ~$0.57 |
