# Q02: Auto Scaling with Predictive and Dynamic Scaling

## Question

An online ticketing platform experiences **predictable traffic spikes** every weekday at 10 AM when tickets go on sale, with traffic increasing from 1,000 RPS to 15,000 RPS within 2 minutes. The platform runs on an Auto Scaling group with EC2 instances behind an Application Load Balancer. Currently, the team uses target tracking scaling (CPU at 60%), but instances take **4 minutes to launch and pass health checks**. Users experience errors during the first few minutes of every sale.

Which approach BEST solves this problem with the LEAST operational overhead?

## Options

- **A.** Increase the minimum capacity of the Auto Scaling group to handle peak traffic at all times.
- **B.** Create a scheduled scaling action to increase capacity 10 minutes before each sale. Return to normal capacity after sales settle.
- **C.** Enable **predictive scaling** on the Auto Scaling group in addition to the existing target tracking policy. Predictive scaling analyzes historical patterns and pre-provisions capacity before predicted spikes.
- **D.** Replace target tracking with step scaling policies that use more aggressive thresholds (e.g., scale out at 30% CPU) to react faster.
- **E.** Use AWS Lambda with provisioned concurrency instead of EC2 to handle the spike.

## Answers

### C. Predictive scaling — ✅ Correct

Predictive scaling uses **machine learning** to analyze historical Auto Scaling group metrics and forecast future demand. It automatically pre-provisions instances **before** predicted traffic spikes. Since the traffic pattern is predictable (daily at 10 AM), predictive scaling will learn this pattern and scale proactively. Combined with the existing target tracking policy (which handles unexpected variations), this provides comprehensive scaling with **zero manual scheduling effort**.

### B. Scheduled scaling — ❌ Incorrect (partially valid)

Scheduled scaling would work and is a reasonable solution. However, it requires **manual configuration and maintenance** — the team must define the schedule, update it if sale times change, and manage holiday/weekend exceptions. The question asks for "LEAST operational overhead," and predictive scaling automates what scheduled scaling does manually.

### A. Over-provision minimum — ❌ Incorrect

Running enough instances to handle 15,000 RPS at all times is extremely wasteful. At 1,000 RPS baseline, you'd be running 15x the needed capacity for most of the day. This solves the problem but at unnecessary cost.

### D. Aggressive step scaling — ❌ Incorrect

Even with a 30% CPU threshold, step scaling is **reactive** — it triggers after CPU rises, then waits for instances to launch (4 minutes). The spike happens in 2 minutes, but instances take 4 minutes to be ready. More aggressive thresholds make reactive scaling faster, but they can't solve a 4-minute bootstrap when the spike arrives in 2 minutes.

### E. Lambda with provisioned concurrency — ❌ Incorrect

Migrating from EC2 to Lambda is a significant re-architecture effort, not "least operational overhead." Additionally, Lambda has limitations (15-minute timeout, connection limits, payload sizes) that may not suit a ticketing platform. This might be valid long-term but doesn't address the immediate scaling issue.

## Recommendations

- **Predictive scaling** is the modern answer for predictable, recurring traffic patterns. It requires minimal setup and learns automatically.
- **Scaling strategy stack** (combine these):
  1. **Predictive scaling** — proactive, ML-driven, handles known patterns
  2. **Target tracking** — reactive, handles unexpected demand
  3. **Scheduled scaling** — manual override for known events not in historical data
- If the question says "predictable pattern" + "least overhead," think predictive scaling.
- If the question says "one-time event" (e.g., product launch), scheduled scaling is better since predictive scaling needs historical data.
- **Instance warm-up time** is critical: if instances take N minutes to warm up, reactive scaling needs to trigger N minutes before the load arrives.

## Relevant Links

- [Predictive Scaling for EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-predictive-scaling.html)
- [Auto Scaling Policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scale-based-on-demand.html)
- [Scheduled Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-scheduled-scaling.html)
- [Target Tracking Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
