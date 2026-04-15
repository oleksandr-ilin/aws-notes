# Q02: Route 53 DNS Routing Policies for Reliability

## Question

A company runs an application in us-east-1 and eu-west-1. Different user groups have different requirements:
- US users must be routed to us-east-1 with automatic failover to eu-west-1 if us-east-1 is unhealthy
- EU users must stay in eu-west-1 for GDPR data residency — they must NEVER be routed to us-east-1
- Internal testing traffic (from specific IP ranges) must be split 80/20 between regions for canary testing
- API endpoint latency must be minimized for mobile users globally

Which combination of Route 53 routing policies should the solutions architect configure?

## Options

- **A.** Use geolocation routing as the primary policy: EU geolocation → eu-west-1 (no failover target), US geolocation → a failover record set (primary: us-east-1 with health check, secondary: eu-west-1). For internal testing, create a separate hosted zone with weighted records (80% us-east-1, 20% eu-west-1) and use Route 53 Resolver rules to direct internal DNS to this zone. For mobile API, use latency-based routing on a separate subdomain (api.example.com) with health checks.
- **B.** Use latency-based routing for all users. Add geolocation restrictions in the application layer (check user IP and redirect if EU user hits us-east-1). Use Route 53 weighted routing for canary testing on the production domain. Add failover health checks to latency records.
- **C.** Use simple routing with multiple values (both Region IPs). Let clients choose randomly. Use application-level geo-detection for routing decisions. Use a separate domain for canary testing.
- **D.** Use failover routing (primary: us-east-1, secondary: eu-west-1) for all users. Use CloudFront geo-restriction to block EU users from us-east-1 origin. Use weighted routing for canary testing.

## Answers

### A. Geolocation + nested failover + weighted (internal) + latency (API) — ✅ Correct

This uses the right policy for each requirement:
- **Geolocation for EU/US split**: Geolocation routing routes based on user location (country/continent). EU record → eu-west-1 with NO secondary record — EU users **cannot** be routed to us-east-1 under any condition. This enforces GDPR residency at the DNS level.
- **Failover nested under US geolocation**: US geolocation points to a failover record set. Primary (us-east-1) has a health check — if it fails, DNS resolves to secondary (eu-west-1). US users get automatic failover while EU users are never affected.
- **Weighted routing for internal canary**: A separate hosted zone (or distinct record name) with weighted records enables 80/20 traffic splitting. Route 53 Resolver rules in the internal VPC DNS direct queries to this zone, so only internal traffic sees the canary split. Production users see the geolocation routing.
- **Latency-based routing for mobile API**: Latency routing directs each user to the Region with lowest network latency. Health checks ensure unhealthy Regions are excluded. This minimizes API response time for global mobile users regardless of geography.
- **Policy separation by subdomain**: `app.example.com` → geolocation policy (web), `api.example.com` → latency policy (mobile API). Different subdomains can use different routing policies.

### B. Latency-based for all + application geo-detection — ❌ Incorrect

- **Latency routing** may route an EU user to us-east-1 if network latency is lower (e.g., an EU user on a VPN terminating in the US). Application-level redirect adds a round trip and is not guaranteed (what if the redirect fails?).
- GDPR compliance requires DNS-level enforcement, not application-level best-effort.
- **Weighted routing on production domain** affects ALL users (not just internal testers) — 20% of production traffic would go to the canary, which is far too aggressive.
- Failover on latency records works but doesn't solve the EU residency requirement.

### C. Simple routing with multiple values — ❌ Incorrect

- **Simple routing** returns all values and client picks randomly — no geographical control, no health-based exclusion, no failover logic.
- Application-level geo-detection is unreliable (VPNs, proxies) and adds latency.
- Separate domain for canary testing means testing a different domain than production — this doesn't validate production routing behavior.

### D. Failover for all + CloudFront geo-restriction — ❌ Incorrect

- **Failover routing** for ALL users means on us-east-1 failure, everyone goes to eu-west-1 — including US users. But during normal operation, ALL users go to us-east-1 (the primary), including EU users. This violates GDPR residency.
- **CloudFront geo-restriction** blocks entire countries from accessing content — it returns a 403, not a redirect. EU users would be blocked, not routed to eu-west-1.
- Weighted routing for canary affects all production traffic (same issue as option B).

## Recommendations

- **Geolocation routing** is the only policy that guarantees users in a specific geography never leave that Region — essential for data residency compliance.
- **Failover routing** can be nested inside geolocation records (alias to a failover record set) — this is the pattern for "same-geography failover."
- **Latency routing** optimizes for performance but provides no geographic guarantees — use it when performance matters more than residency.
- **Weighted routing** is perfect for canary and A/B testing — combine with Route 53 test-specific record names to isolate test traffic from production.
- **Set TTL ≤ 60 seconds** on failover records — longer TTLs delay failover because clients cache the old DNS record.
- **Health check types**: HTTP/HTTPS health checks with string matching (verify response body) are more reliable than TCP checks for application-level health.

## Relevant Links

- [Route 53 Geolocation Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html)
- [Route 53 Failover Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-failover.html)
- [Route 53 Latency Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-latency.html)
- [Route 53 Weighted Routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html)
- [Route 53 Health Checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-creating.html)
