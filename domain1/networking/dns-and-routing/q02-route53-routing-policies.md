# Q02: Route 53 Routing Policies for Multi-Region Applications

## Question

A company runs an application in us-east-1 (primary) and eu-west-1 (secondary). The requirements are:
- Normal operation: Route all traffic to us-east-1
- If us-east-1 health checks fail: Automatically route all traffic to eu-west-1
- During planned maintenance on us-east-1: Route 10% of traffic to eu-west-1 for pre-warming, then shift to 100%
- For a new API version being tested: Route users from European IP ranges to eu-west-1 and all others to us-east-1

Which Route 53 routing configuration should the solutions architect use for each scenario?

## Options

- **A.** Failover routing for automatic failover, weighted routing for gradual traffic shifting, geolocation routing for European user routing.
- **B.** Failover routing for automatic failover, latency-based routing for gradual traffic shifting, geoproximity routing for European user routing.
- **C.** Simple routing for all scenarios with manual DNS record updates during failover and maintenance.
- **D.** Latency-based routing for automatic failover with health checks, weighted routing for gradual traffic shifting, geolocation routing for European user routing.

## Answers

### A. Failover + Weighted + Geolocation — ✅ Correct

Each scenario maps to a different Route 53 routing policy:

**Failover routing** (automatic failover):
- **Primary** record points to us-east-1. **Secondary** record points to eu-west-1.
- Route 53 health checks monitor the us-east-1 endpoint. When health checks fail, Route 53 automatically returns the secondary record.
- This is the correct policy for active/passive failover scenarios.

**Weighted routing** (gradual traffic shifting):
- Create two weighted records: us-east-1 with weight 90 and eu-west-1 with weight 10.
- Route 53 distributes traffic proportionally (90%/10%). Adjust weights to shift traffic gradually (80/20 → 50/50 → 0/100).
- Ideal for blue/green deployments and maintenance pre-warming.

**Geolocation routing** (geographic routing):
- Create a geolocation record for "Europe" pointing to eu-west-1.
- Create a geolocation record for "Default" pointing to us-east-1.
- Route 53 routes European users to eu-west-1 and all others to us-east-1 based on the source IP's geographic location.

### B. Failover + Latency + Geoproximity — ❌ Incorrect

Latency-based routing sends users to the lowest-latency Region, which doesn't allow explicit percentage-based traffic splitting. You can't say "send 10% to eu-west-1" with latency routing. Geoproximity routing uses geographic coordinates and bias values — while it can approximate geographic routing, geolocation routing provides exact continent/country-based control.

### C. Simple routing — ❌ Incorrect

Simple routing returns a single value (or multiple random values) with no intelligence. Manual DNS changes for failover introduce downtime (TTL propagation delay) and human error. It doesn't support health checks, weighted distribution, or geographic routing.

### D. Latency-based for failover — ❌ Incorrect

Latency-based routing with health checks can provide failover behavior, but it routes based on latency, not explicit primary/secondary designation. A user near eu-west-1 might be routed there even when us-east-1 is healthy. Failover routing provides deterministic primary/secondary behavior.

## Recommendations

- **Combine routing policies** using alias records for complex scenarios (e.g., failover wrapping weighted records).
- **Health checks** work with all routing policies except Simple routing. Always configure health checks for production DNS records.
- **Geolocation vs. Geoproximity**: Geolocation = hard boundaries by continent/country. Geoproximity = soft boundaries with adjustable bias values for fine-tuning.
- **Weighted routing** with weight 0 excludes a Region entirely — useful for directing all traffic away during maintenance.
- **Multivalue answer routing** returns up to 8 healthy records — a simple load distribution mechanism, not a replacement for ELB.

## Relevant Links

- [Route 53 Routing Policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [Failover Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
- [Weighted Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html)
- [Geolocation Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html)
- [Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-creating.html)
