# Q01: Auto Scaling Policies and Database Failover

## Question

A SaaS platform experiences a predictable traffic pattern: low traffic 11 PM – 7 AM, steady traffic 7 AM – 5 PM, and 3x traffic spikes 5 PM – 11 PM (evening peak). On Black Friday, traffic is 10x normal. The platform runs on an ASG with ALB, Aurora PostgreSQL, and ElastiCache Redis. Requirements:
- Evening peak scaling must complete before the spike (no reactive scaling lag)
- Black Friday scaling must be handled without pre-provisioning for weeks in advance
- If Aurora writer fails, the application must reconnect automatically without code changes
- If an AZ experiences degradation (not full failure), unhealthy instances must be replaced
- Cost must be minimized during low-traffic hours

Which auto scaling and failover configuration should the solutions architect implement?

## Options

- **A.** Combine 3 ASG scaling policies: (1) scheduled scaling to set min capacity high at 4:45 PM and low at 11 PM daily, (2) predictive scaling enabled with forecast-based pre-scaling, and (3) target tracking on ALB request count per target as a safety net for unexpected spikes. For Black Friday, update the scheduled policy temporarily. Configure Aurora with a cluster endpoint (writer) and reader endpoint. Use Aurora's built-in failover and configure the application to use the cluster endpoint DNS (CNAME that auto-updates on failover). Enable ECS/EC2 health checks in the ASG that include ALB health check status so AZ-degraded instances are replaced.
- **B.** Use only target tracking scaling. Set a low target to be aggressive on scaling. Over-provision the ASG minimum to handle evening peaks. For Black Friday, manually set desired capacity the night before. Connect the application to the Aurora writer instance endpoint directly. Use default EC2 health checks.
- **C.** Use step scaling policies with CloudWatch CPU alarms at 50%, 70%, and 90%. Schedule a Lambda to increase ASG capacity at 4:45 PM daily. For Black Friday, create a separate Auto Scaling group with higher capacity. Use Route 53 failover routing between the primary Aurora instance and a read replica. Enable ALB health checks.
- **D.** Use predictive scaling only — it will learn all patterns including Black Friday from historical data. Connect to Aurora using the instance endpoint. Use EC2 status checks for health monitoring.

## Answers

### A. Scheduled + predictive + target tracking + Aurora cluster endpoint — ✅ Correct

This addresses every requirement:
- **Scheduled scaling (4:45 PM pre-scaling)**: Sets the ASG minimum capacity high 15 minutes before the known evening spike. Instances launch and warm up before traffic arrives — no reactive lag.
- **Predictive scaling**: Uses ML to forecast traffic patterns from 14 days of history and pre-scales up to 15 minutes before predicted demand. This handles daily variations and gradual pattern changes automatically.
- **Target tracking as safety net**: If actual traffic exceeds predictions (unexpected spike), target tracking reacts to the current ALB request count. This combination of proactive (scheduled + predictive) and reactive (target tracking) ensures coverage.
- **Black Friday**: Update the scheduled policy a few days before to set a higher minimum during the sale period. Predictive scaling adapts if this year's traffic differs from last year's. After Black Friday, revert the schedule — no permanent over-provisioning.
- **Aurora cluster endpoint**: The cluster endpoint (`*.cluster-*.region.rds.amazonaws.com`) is a DNS CNAME that always points to the current writer. On failover, Aurora updates the CNAME to point to the promoted replica. The application reconnects using the same DNS name — no code change, no manual DNS update. Connection poolers (like RDS Proxy) further smooth reconnection.
- **ALB health checks in ASG**: ALB marks targets unhealthy based on HTTP response codes. ASG uses ALB health status to terminate and replace instances — even if the EC2 instance itself is running but the application is failing due to AZ degradation.

### B. Target tracking only + over-provisioning + direct instance endpoint — ❌ Incorrect

- **Target tracking only**: Reactive — scaling happens AFTER load increases, causing 2-5 minute lag while new instances launch and warm up. Evening peak users experience degradation.
- **Over-provisioning minimum**: Expensive — paying for peak capacity 24/7 when it's only needed 6 hours/day.
- **Manual Black Friday scaling**: Risk of human error (wrong capacity, wrong time). Requires someone to remember the night before.
- **Aurora instance endpoint**: Points to a specific instance. On failover, the endpoint doesn't change — the application still connects to the old (now reader or defunct) instance. Connection fails until the app is reconfigured.
- **Default EC2 health checks**: Only detect hard EC2 failures (instance stopped/terminated). An instance with a crashed application process still appears healthy to EC2 status checks — it stays in the ASG serving errors.

### C. Step scaling + Lambda scheduler + separate ASG — ❌ Incorrect

- **Step scaling**: Better than simple scaling but still reactive. CPU-based alarms may not correlate with request load (a single slow query can spike CPU without increased traffic).
- **Lambda for scheduling**: Functional but reinvents what ASG scheduled actions provide natively. Custom code to maintain.
- **Separate ASG for Black Friday**: Requires maintaining two ASG configurations, migration between them, and potential DNS/ALB changes. Unnecessary complexity.
- **Route 53 failover for Aurora**: Route 53 is DNS-level — it has TTL propagation delays (seconds to minutes). Aurora's built-in failover via cluster endpoint is faster and requires no external DNS configuration.

### D. Predictive scaling only — ❌ Insufficient

- Predictive scaling needs **14+ days of consistent historical data** to forecast accurately. One-off events like Black Friday may not be in the forecast window if it only looks at recent weeks.
- No scheduled override means no guaranteed pre-scaling for known events.
- No target tracking safety net means unexpected traffic patterns (above prediction) cause under-provisioning.
- Instance endpoints and EC2-only health checks have the same problems as described in option B.

## Recommendations

- **Combine scaling policies**: AWS supports applying scheduled, predictive, and target tracking simultaneously. The effective desired capacity is the maximum of all policy outputs — this ensures the most demanding signal wins.
- **Predictive scaling mode**: Use `ForecastAndScale` (actually scales based on forecast) not just `ForecastOnly` (generates forecast but doesn't act on it).
- **Aurora cluster endpoints**: ALWAYS use the cluster endpoint (writer) and reader endpoint (reader) — never connect to instance endpoints in production.
- **RDS Proxy**: Consider adding RDS Proxy between the application and Aurora — it maintains a persistent connection pool and handles failover reconnection transparently, reducing application-side connection storms.
- **Health check grace period**: Set appropriately (e.g., 300 seconds) so new instances have time to warm up before health checks mark them unhealthy.

## Relevant Links

- [Predictive Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-predictive-scaling.html)
- [Target Tracking Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- [Aurora Endpoints](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html)
- [Aurora Failover](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [ASG Health Checks](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-health-checks.html)
