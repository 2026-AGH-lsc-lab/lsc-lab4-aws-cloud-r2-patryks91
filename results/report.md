# AWS Cloud Lab Report

**Date:** 2026-04-06
**Region:** us-east-1

---

## Assignment 1: Deployment Verification

All four endpoints deployed successfully and return identical k-NN results. The results array contains indices [35859, 24682, 35397, 20160, 30454] with distances ~12.0, confirming all environments run the same workload correctly.

---

## Assignment 2: Cold Start Characterization

Lambda Zip cold start: ~1.5 seconds
Lambda Container cold start: ~1.9 seconds
Warm invocations (both): ~200-220 ms

**Why is Zip faster?** Zip deployment has a smaller package size (~2KB vs ~500MB container image) and uses pre-built Lambda runtime, requiring less initialization time. Container must start custom entrypoint and load larger image layers.

---

## Assignment 3: Warm Steady-State Throughput

Lambda Zip p50/p95/p99: 208ms / 238ms / 492ms (c=10)
Lambda Container p50/p95/p99: 205ms / 233ms / 501ms (c=10)
Fargate p50/p95/p99: 802ms / 1083ms / 1205ms (c=10), 4002ms / 4282ms / 4391ms (c=50)
EC2 p50/p95/p99: 348ms / 1296ms / 1399ms (c=10), 879ms / 3646ms / 4241ms (c=50)

**Why doesn't Lambda p50 change with concurrency?** Each Lambda request gets its own execution environment — no resource contention, true parallel processing.

**Why does Fargate/EC2 p50 increase at c=50?** Single task/instance handles all requests. With 2 vCPU and 50 concurrent requests, ~48 requests must queue while 2 are processed.

**Latency difference (server vs client):** Server-side query_time is ~70ms, client-side p50 is ~200-350ms. The difference (~130-280ms) comes from network RTT, TLS handshake, and HTTP overhead.

---

## Assignment 4: Burst from Zero

Lambda Zip p99: 1756ms (10 cold starts out of 200 requests)
Lambda Container p99: 1416ms (10 cold starts out of 200 requests)
Fargate p99: 4265ms
EC2 p99: 1383ms

**Why is Lambda burst p99 much higher than Fargate/EC2?** Lambda triggers cold starts for the first 10 requests (one per execution environment). These cold starts add 1.3-1.8 seconds to response time, creating a bimodal distribution.

**Bimodal distribution:** 190 requests at 200-350ms (warm), 10 requests at 1300-1800ms (cold).

**Does Lambda meet p99 < 500ms SLO?** No. Lambda p99 is 1756ms (Zip) and 1416ms (Container). To meet SLO, Provisioned Concurrency is needed to eliminate cold starts.

---

## Assignment 5: Cost at Zero Load

Lambda idle cost: $0.00/month (pay only for invocations)
Fargate idle cost: $17.77/month (0.5 vCPU, 1GB running 24/7)
EC2 idle cost: $14.98/month (t3.small running 24/7)

**Which environment has zero idle cost?** Lambda — you only pay when requests are processed. Fargate and EC2 run continuously and incur cost regardless of traffic.

---

## Assignment 6: Cost Model and Recommendation

**Traffic model:** Peak 100 RPS × 30min + Normal 5 RPS × 5.5h + Idle 18h = 8,370,000 requests/month

**Monthly costs:**
- Lambda: $16.11 (requests $1.67 + duration $14.44)
- Fargate: $17.77
- EC2: $14.98

**Break-even RPS:**
- Lambda = EC2 at 3.0 RPS
- Lambda = Fargate at 3.56 RPS
- Below these thresholds, Lambda is cheaper

**Recommendation:** Lambda with Provisioned Concurrency

**Justification:**
1. Cost is comparable ($16.11 vs $14.98 EC2) but Lambda has zero idle cost during 18h/day
2. Lambda auto-scales instantly to handle 100 RPS peak without capacity planning
3. With Provisioned Concurrency (10 instances, ~$52/month extra), cold starts are eliminated and p99 < 500ms SLO is achievable

**Changes needed:** Enable Provisioned Concurrency for 10 instances. Total cost with PC: ~$68/month.

**When would recommendation change:**
- If average RPS > 3.56: Switch to Fargate (Lambda becomes more expensive)
- If SLO relaxed to p99 < 2000ms: Lambda without Provisioned Concurrency is acceptable
- If traffic becomes predictable: EC2 with Reserved Instances saves ~40%

---

## Conclusion

No environment meets p99 < 500ms SLO in default configuration. Lambda with Provisioned Concurrency is recommended for sporadic, spiky traffic because it combines zero idle cost, automatic scaling, and SLO compliance when properly configured.