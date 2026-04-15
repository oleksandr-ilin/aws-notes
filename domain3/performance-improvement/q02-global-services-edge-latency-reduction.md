# Q02: Adopting Global Services and Edge Computing for Latency Improvement

## Question

A company runs a customer-facing application serving users across North America, Europe, and Asia-Pacific. Current architecture: all resources in us-east-1, no edge services.

**Current performance complaints:**
- US East users: 120 ms P95 latency (acceptable)
- US West users: 200 ms P95 (borderline)
- EU users: 350 ms P95 (**unacceptable** — target < 200 ms)
- APAC users: 500 ms P95 (**unacceptable** — target < 250 ms)

**Architecture breakdown (latency analysis from X-Ray):**
- DNS resolution: 5 ms (Route 53 — fast globally)
- TCP/TLS handshake to ALB in us-east-1: EU = 80 ms, APAC = 150 ms (cross-ocean round trips)
- ALB → EC2 processing: 40 ms (same Region — constant)
- EC2 → RDS query: 25 ms (same Region — constant)
- EC2 → response to client: EU = 80 ms, APAC = 150 ms (return trip)
- **Total EU**: 5 + 80 + 40 + 25 + 80 = 230 ms (server-side) + client rendering
- **Total APAC**: 5 + 150 + 40 + 25 + 150 = 370 ms (server-side)

The network round trips (TCP handshake + response) dominate latency for distant users. The application team **cannot** deploy the full application stack in multiple Regions (too complex, too expensive for current scale).

Which improvements reduce global latency to meet targets without multi-Region deployment?

## Options

- **A.** Deploy **CloudFront** in front of the ALB. CloudFront edge locations in EU and APAC terminate TCP/TLS at the nearest edge (EU user → Frankfurt edge = 10 ms handshake instead of 80 ms). CloudFront maintains persistent connections to the origin (ALB in us-east-1) — eliminating per-request TCP handshake overhead to the origin. For cacheable responses (product pages, images, API responses with TTL): served directly from edge cache — 0 ms origin fetch. For dynamic requests: CloudFront routes over AWS backbone (faster than public internet) — EU-to-us-east-1 backbone latency ≈ 40 ms vs. 80 ms public internet. Use **Lambda@Edge** (origin-request trigger) to personalize responses at the edge without hitting the origin for every request. Add **CloudFront Origin Shield** in us-east-1 to consolidate cache fills from all edge locations. Combined: EU P95 drops from 350 ms to ~150 ms, APAC from 500 ms to ~220 ms. For additional APAC improvement, add **Global Accelerator** for the non-CloudFront traffic path (WebSocket, TCP) — Accelerator routes traffic over AWS backbone from the nearest edge to us-east-1, improving TCP latency by 30-40%.
- **B.** Deploy a full application stack (ALB + EC2 + RDS read replica) in eu-west-1 and ap-southeast-1. Use Route 53 latency-based routing. EU and APAC users connect to their local Region.
- **C.** Increase EC2 instance sizes in us-east-1 to process requests faster. Reduce RDS query time with read replicas. This reduces the 40 ms + 25 ms server processing time.
- **D.** Use Route 53 latency-based routing with health checks. Move the DNS TTL to 0 for fastest resolution. Enable TCP keepalive on all clients.

## Answers

### A. CloudFront + Lambda@Edge + Origin Shield + Global Accelerator — ✅ Correct

**CloudFront — Eliminating Distance Penalty:**

- **Edge termination**: CloudFront has 400+ Points of Presence (PoP) globally. A user in Frankfurt connects to the Frankfurt edge (10 ms round trip) instead of us-east-1 (80 ms). The TCP and TLS handshakes happen at the edge — 3 round trips saved at 10 ms instead of 80 ms = 210 ms saved (3 × 70 ms improvement).

- **Persistent origin connections**: CloudFront maintains a pool of persistent TCP connections to the origin (ALB). When a user request arrives at the edge, CloudFront doesn't establish a new TCP connection to the origin — it reuses an existing one. This saves another 80-150 ms per request (the TCP/TLS handshake to origin).

- **Cacheable responses (cache hit)**:
  - Request hits CloudFront edge → edge has cached response → returns immediately (< 10 ms round trip)
  - No origin fetch at all. For a 60% cache hit rate, 60% of requests see < 20 ms total latency
  - Product catalog, images, CSS/JS, API responses with `Cache-Control: public, max-age=300` — all cacheable

- **Dynamic responses (cache miss)**:
  - Request hits CloudFront edge → edge forwards to origin over AWS private backbone
  - AWS backbone latency: Frankfurt to us-east-1 ≈ 40 ms (vs. 80 ms public internet)
  - Response returns over backbone: another 40 ms
  - Total dynamic request (EU): 10 ms (edge) + 40 ms (backbone) + 40 ms (server processing) + 25 ms (RDS) + 40 ms (return backbone) = **155 ms** (vs. 350 ms current)

- **Latency improvement summary**:

  | User Location | Current P95 | With CloudFront | Target |
  |---|---|---|---|
  | US East | 120 ms | 80 ms | < 200 ms ✅ |
  | US West | 200 ms | 120 ms | < 200 ms ✅ |
  | EU | 350 ms | 155 ms | < 200 ms ✅ |
  | APAC | 500 ms | 220 ms | < 250 ms ✅ |

**Lambda@Edge — Edge-Side Personalization:**

- Some requests that appear dynamic (personalized greetings, locale-specific content) can be handled at the edge without hitting the origin:
  - **Viewer-request trigger**: Inspect the `Accept-Language` header, rewrite the URL to serve the locale-specific cached version: `/products` → `/products/de-DE` (cached at edge)
  - **Origin-request trigger**: Run lightweight personalization logic (user segment from JWT → select cached variant). This turns "dynamic" requests into cache hits
  - **Origin-response trigger**: Add security headers, modify `Cache-Control` based on content type

- Lambda@Edge limitations: 5-second timeout (viewer triggers), 30-second timeout (origin triggers), 128 MB max (viewer), 10 GB max (origin). For heavy computation, use the origin (EC2/Lambda in us-east-1).

**Origin Shield — Cache Layer Consolidation:**

- Without Origin Shield: 400+ edge locations independently fetch from the origin. A popular resource fetched from 50 edges = 50 origin requests
- With Origin Shield: All edges fetch from a single Origin Shield location (e.g., us-east-1 edge). Origin Shield caches the response. Only 1 origin request for 50 edges.
- **Impact**: Further increases cache hit rate (at the Origin Shield layer) and reduces origin load by 50-80%.

**Global Accelerator — For Non-HTTP Traffic:**

- CloudFront handles HTTP/HTTPS and WebSocket. For raw TCP/UDP workloads (game servers, IoT device connections, legacy protocols), **Global Accelerator** provides similar routing benefits:
  - Anycast IP addresses → traffic enters AWS backbone at the nearest edge
  - Health check-based failover between endpoints
  - Static IP addresses that don't change with endpoint changes

- For this application's API traffic, CloudFront is sufficient. Global Accelerator adds value for WebSocket connections or if the application has non-HTTP components.

### B. Full multi-Region deployment — ❌ Incorrect

- **Technically correct but over-engineered**: Deploying full stacks in 3 Regions provides the best possible latency (local resources for each Region). However:
  - The team explicitly stated they can't do multi-Region ("too complex, too expensive for current scale")
  - Multi-Region requires: database replication (RDS read replicas or Aurora Global Database), data consistency management, deployment pipeline for 3 Regions, cross-Region monitoring, and 3× infrastructure cost
  - CloudFront achieves the latency targets (< 200 ms EU, < 250 ms APAC) without multi-Region complexity
  - Multi-Region is the right answer when CloudFront alone can't meet latency requirements (e.g., < 50 ms for APAC — only local compute can achieve this)

### C. Increase instance sizes — ❌ Incorrect

- **Server processing (40 ms + 25 ms = 65 ms) is NOT the bottleneck**: Even reducing server processing to 0 ms, the EU latency would be: 80 ms (handshake) + 0 ms + 80 ms (return) = 160 ms. For APAC: 150 + 0 + 150 = 300 ms — still exceeds the target.
- The bottleneck is **network round trips** (80-150 ms × 2-3 handshakes), not server processing. Faster servers don't fix network latency — edge services do.
- Read replicas reduce RDS query time but don't reduce network distance to the user. A query that takes 15 ms instead of 25 ms saves 10 ms — irrelevant when the network penalty is 300 ms.

### D. Route 53 optimization — ❌ Incorrect

- **Route 53 latency-based routing**: Routes users to the nearest endpoint. But there's only one endpoint (us-east-1) — latency-based routing with a single Region returns the same result as simple routing. You need endpoints in multiple Regions for latency-based routing to be useful.
- **DNS TTL = 0**: Most DNS resolvers ignore TTL of 0 and cache for 30-60 seconds anyway. Even if TTL worked at 0, DNS resolution is 5 ms of the 500 ms total — optimizing DNS saves trivial time compared to the 300 ms network penalty.
- **TCP keepalive on clients**: Keepalive maintains persistent connections between the client and server, avoiding repeated TCP handshakes. But:
  - Mobile clients and browsers already use HTTP/2 multiplexing (persistent connections). Additional keepalive settings have minimal effect.
  - The initial connection still requires TCP + TLS handshakes. For users who haven't connected recently, the full handshake penalty remains.
  - CloudFront edge termination is more effective — it moves the handshake target from us-east-1 to the local edge (10 ms vs. 150 ms).

## Recommendations

- **CloudFront is the first tool for global latency reduction**: Before considering multi-Region, evaluate whether CloudFront + edge services meet your latency targets. CloudFront is simpler, cheaper, and solves 80% of global latency problems.
- **Cache everything possible**: Every cache hit at the edge eliminates the origin round trip entirely. Set appropriate `Cache-Control` headers on all responses. Use cache policies to normalize cache keys (ignore tracking query strings).
- **HTTP/3 (QUIC) on CloudFront**: CloudFront supports HTTP/3 with QUIC — reduces handshake latency further (0-RTT connection establishment). Enable it for all distributions.
- **CloudFront Functions vs. Lambda@Edge**: CloudFront Functions run at all 400+ PoPs with sub-millisecond overhead (~1 ms) — use for simple manipulations (header rewrites, URL rewrites, redirects). Lambda@Edge runs at Regional edge caches (13 locations) with higher latency (~5-50 ms) — use for complex logic (authentication, personalization, origin selection).
- **Monitor Regional latency**: Create CloudWatch Metrics with a `Region` dimension. Track P95 latency per Region/country to detect degradation before users complain.
- **Evaluate multi-Region when edge isn't enough**: If the application requires < 50 ms for global users, edge services can't achieve this for dynamic content. Multi-Region deployment is then necessary.

## Relevant Links

- [CloudFront Performance](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/performance.html)
- [Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html)
- [CloudFront Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html)
- [Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html)
- [CloudFront vs Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-comparison.html)
- [CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html)
