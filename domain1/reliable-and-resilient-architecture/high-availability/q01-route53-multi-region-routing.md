# Q01: Route 53 Routing Policies for Multi-Region HA

## Question

A company operates a web application in **three AWS Regions**: us-east-1, eu-west-1, and ap-southeast-1. Each Region runs an identical deployment. The company wants to route users to the **nearest healthy Region** to minimize latency. If a Region becomes unhealthy, traffic should automatically shift to the next closest Region. The application serves a global user base.

Which Route 53 configuration should the solutions architect implement?

## Options

- **A.** Use Route 53 **weighted routing** with equal weights for all three Regions. Associate health checks with each record. If a Region fails its health check, Route 53 removes it from rotation.
- **B.** Use Route 53 **latency-based routing** with health checks enabled for each Region's endpoint. Configure the "Evaluate Target Health" option. When a Region fails, Route 53 automatically routes to the next closest Region.
- **C.** Use Route 53 **failover routing** with us-east-1 as primary and eu-west-1 as secondary. Configure a third CNAME for ap-southeast-1.
- **D.** Use Route 53 **geolocation routing** to map continents to Regions (North America → us-east-1, Europe → eu-west-1, Asia → ap-southeast-1) with health checks.
- **E.** Use AWS Global Accelerator with endpoint groups in each Region. Configure health checks to remove unhealthy Regions.

## Answers

### B. Latency-based routing with health checks — ✅ Correct

Latency-based routing directs users to the Region with the **lowest network latency** from their location. With health checks enabled, if a Region becomes unhealthy, Route 53 automatically excludes it and routes to the next-lowest-latency Region. This precisely matches the requirements: nearest healthy Region, automatic failover, global users.

### E. Global Accelerator — ❌ Incorrect (partially valid)

Global Accelerator uses anycast IPs to route traffic to the nearest healthy endpoint and can handle Region-level failover. While it would technically work, the question specifically asks about Route 53 configuration (DNS-level routing). Global Accelerator operates at the network layer and adds additional cost. It's a better solution when you need deterministic IP addresses or TCP/UDP optimization, not just DNS routing.

### A. Weighted routing — ❌ Incorrect

Weighted routing distributes traffic based on **assigned weights**, not proximity. Equal weights would send roughly equal traffic to all Regions regardless of user location. A user in Tokyo would be equally likely to hit us-east-1 as ap-southeast-1, resulting in poor latency. Weighted routing doesn't consider geographic or network proximity.

### C. Failover routing — ❌ Incorrect

Failover routing supports **only two records** (one primary, one secondary) per record name. You cannot configure three-way failover directly. To handle three Regions, you'd need nested routing policies (failover → latency), which adds unnecessary complexity when latency-based routing with health checks handles this natively.

### D. Geolocation routing — ❌ Incorrect

Geolocation routing maps users by **geographic location** (continent/country), not by network latency. A user in the UK would always hit eu-west-1, even if us-east-1 happened to have lower latency at that moment. Geolocation also requires a **default record** for locations not explicitly mapped — missing this causes DNS failures. Geolocation is correct when you need location-based content (e.g., language) or compliance, not performance optimization.

## Recommendations

- **Route 53 routing policy cheat sheet:**
  - **Latency-based:** Route to lowest-latency Region → best for performance
  - **Failover:** Primary/secondary → best for simple two-Region DR
  - **Weighted:** Percentage-based distribution → best for blue/green deployments, canary releases
  - **Geolocation:** Location-based → best for compliance or localized content
  - **Geoproximity:** Tunable proximity → best when you want to shift traffic proportions between Regions
  - **Multivalue answer:** Multiple healthy IPs → simple load distribution
- **Always enable health checks** with Route 53 routing policies in multi-Region setups.
- Know the difference between **latency-based** (network performance) vs. **geolocation** (physical location) — the exam frequently tests this distinction.

## Relevant Links

- [Route 53 Latency-Based Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html)
- [Route 53 Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-types.html)
- [Route 53 Routing Policy Comparison](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [AWS Global Accelerator vs Route 53](https://aws.amazon.com/global-accelerator/faqs/)
