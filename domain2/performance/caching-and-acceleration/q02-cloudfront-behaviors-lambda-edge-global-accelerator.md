# Q02: CloudFront Behaviors, Lambda@Edge, and Global Accelerator

## Question

A company operates a platform with three traffic types served from the same domain (app.example.com):

1. **Static assets** (CSS, JS, images) — served from S3, cacheable for 7 days, ~500 GB total
2. **Dynamic API** (/api/*) — served from ALB → ECS, personalized per user (not cacheable), requires < 100 ms global latency
3. **WebSocket connections** (/ws) — long-lived real-time connections for live notifications, served by an NLB → EC2 fleet

Additional requirements:
- Serve all traffic over HTTPS from a single domain
- Redirect HTTP to HTTPS at the edge
- Add security headers (Content-Security-Policy, Strict-Transport-Security) to all responses
- Bot traffic (~30% of total) must be identified and blocked at the edge before reaching origin
- Users in China must be blocked due to licensing restrictions

Which architecture serves all three traffic types efficiently?

## Options

- **A.** A single CloudFront distribution with three cache behaviors: (1) `/static/*` → S3 origin, TTL 7 days, (2) `/api/*` → ALB origin, TTL 0 (no caching), (3) Default (`/*`) → ALB origin. CloudFront Functions for HTTP→HTTPS redirect and security headers (viewer-request/viewer-response). AWS WAF on CloudFront for bot detection and geo-restriction (block CN). For WebSocket, use a SEPARATE Global Accelerator → NLB endpoint on a subdomain (ws.example.com) — CloudFront doesn't support WebSocket long-lived connections efficiently via cache behaviors.
- **B.** A single CloudFront distribution with cache behaviors for all three traffic types including WebSocket. `/ws` behavior with WebSocket protocol support → NLB origin. CloudFront handles WebSocket natively. Lambda@Edge for security headers, bot detection, and geo-blocking. AWS WAF for additional bot rules.
- **C.** Three separate CloudFront distributions (one per traffic type) sharing the same domain via Route 53 weighted routing. WAF on each distribution. Lambda@Edge on each for security headers.
- **D.** Global Accelerator for ALL traffic types. Three endpoint groups: S3 (static), ALB (API), NLB (WebSocket). Global Accelerator handles routing based on URL path. CloudFront only for S3 static asset caching on a separate subdomain.

## Answers

### B. Single CloudFront distribution with WebSocket support + Lambda@Edge + WAF — ✅ Correct

CloudFront natively supports WebSocket connections, and a single distribution handles all three traffic types:

- **Single CloudFront distribution with multiple cache behaviors**:
  - CloudFront evaluates path patterns in order and routes to the matching origin:
    - `/static/*` → S3 origin, cache policy with TTL 7 days, compress (gzip/brotli)
    - `/api/*` → ALB origin (custom origin), cache policy with TTL 0 (forward all headers, query strings, cookies — no caching), origin request policy to forward Authorization header
    - `/ws` → NLB or ALB origin with WebSocket support
    - Default `/*` → ALB origin (catch-all)
  - **One domain, one SSL certificate**: A single CloudFront distribution uses one custom SSL certificate (ACM in us-east-1). All traffic types share the same HTTPS termination.

- **CloudFront WebSocket support**:
  - CloudFront has supported WebSocket connections since 2018. When a client sends an HTTP request with `Upgrade: websocket`, CloudFront forwards it to the origin and establishes the WebSocket connection through CloudFront.
  - **Configuration**: The cache behavior for `/ws` must be configured with: `AllowedMethods: GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE` (to allow the WebSocket upgrade), `CachePolicyId: CachingDisabled`, and protocol policy `HTTPS Only` or `Match Viewer`.
  - CloudFront maintains the persistent WebSocket connection — it does NOT cache WebSocket frames. The connection passes through CloudFront to the origin transparently.

- **Lambda@Edge for security headers and customization**:
  - **Viewer-response trigger**: Lambda@Edge function adds security headers to ALL responses regardless of cache behavior:
    ```javascript
    headers['strict-transport-security'] = [{value: 'max-age=63072000; includeSubdomains; preload'}];
    headers['content-security-policy'] = [{value: "default-src 'self'"}];
    headers['x-content-type-options'] = [{value: 'nosniff'}];
    ```
  - Lambda@Edge runs at edge locations — headers are added without a round trip to the origin.
  - **HTTP → HTTPS redirect**: A viewer-request Lambda@Edge function checks if the protocol is HTTP and returns a 301 redirect to HTTPS. Alternatively, CloudFront's built-in "Redirect HTTP to HTTPS" viewer protocol policy handles this without Lambda.

- **AWS WAF on CloudFront**:
  - WAF web ACL attached to the CloudFront distribution inspects all requests before they reach any origin.
  - **Bot detection**: AWS WAF Bot Control managed rule group identifies and blocks bots using browser fingerprinting, CAPTCHA challenges, and behavioral analysis. The 30% bot traffic is blocked at the edge — reducing origin load by 30%.
  - **Geo-restriction**: CloudFront's built-in geo-restriction can block requests from China (CN). Alternatively, WAF geo-match condition provides the same functionality with more granular control (can block specific IPs within allowed countries).

- **Why Lambda@Edge over CloudFront Functions**:
  - CloudFront Functions are lighter-weight (< 1 ms execution, 10 KB max code, limited to viewer-request/viewer-response) and cheaper. For simple security headers and redirects, CloudFront Functions suffice.
  - Lambda@Edge supports origin-request/origin-response triggers, has 128-10,240 MB memory, 5 second (viewer) / 30 second (origin) timeout, and can make external network calls. For complex bot detection logic that requires database lookups, Lambda@Edge is needed.
  - For this scenario, WAF handles bot detection and CloudFront Functions could handle headers/redirects. Lambda@Edge provides additional flexibility if needed.

### A. CloudFront + separate Global Accelerator for WebSocket — ❌ Incorrect

- **CloudFront supports WebSocket natively**: The premise that CloudFront "doesn't support WebSocket long-lived connections efficiently" is incorrect. CloudFront has supported WebSocket since November 2018. Using a separate Global Accelerator and subdomain for WebSocket adds unnecessary complexity (two endpoints, two SSL certificates, cross-origin issues).
- **Separate subdomain (ws.example.com)**: Requires a separate DNS entry, separate SSL certificate, and the client application must connect to a different domain for WebSocket vs. HTTP — adding client-side complexity and potential CORS issues.
- The CloudFront configuration for static and API is correct, but separating WebSocket is unnecessary.

### C. Three separate CloudFront distributions — ❌ Incorrect

- **Multiple distributions sharing one domain**: A CloudFront distribution is associated with a set of domain names (CNAMEs). Two distributions cannot have the same CNAME — this is a CloudFront constraint. Each distribution would need a separate subdomain (static.example.com, api.example.com, ws.example.com).
- **Route 53 weighted routing**: Route 53's weighted routing distributes traffic based on weights (percentages), not URL paths. You can't route `/static/*` to one distribution and `/api/*` to another using DNS alone — DNS resolves at the domain level, not the path level.
- **Three WAF web ACLs + three Lambda@Edge deployments**: Tripling the security infrastructure for no architectural benefit. A single distribution with cache behaviors handles path-based routing natively.

### D. Global Accelerator for everything — ❌ Incorrect

- **Global Accelerator doesn't cache**: Global Accelerator routes TCP/UDP traffic to endpoints — it doesn't cache content. Static assets (500 GB) would be fetched from S3 on every request — no CDN benefit. CloudFront caches at 400+ edge locations, dramatically reducing origin requests and latency for static content.
- **Global Accelerator doesn't support path-based routing**: Global Accelerator routes traffic to endpoint groups (ALB, NLB, EC2, EIP) based on geography and health — not URL paths. You can't route `/static/*` to S3 and `/api/*` to ALB using Global Accelerator alone.
- **Global Accelerator doesn't support WAF**: WAF web ACLs can be attached to CloudFront, ALB, and API Gateway — but NOT Global Accelerator. Bot detection must happen at the ALB level (after traffic traverses the network), not at the edge.
- Global Accelerator is designed for dynamic API and TCP/UDP traffic where caching isn't possible — it's complementary to CloudFront, not a replacement.

## Recommendations

- **Single CloudFront distribution with multiple cache behaviors** is the recommended pattern for serving different traffic types from one domain.
- **Cache behavior order matters**: CloudFront evaluates path patterns in order (from most specific to least specific). Place specific patterns first (`/api/v1/users/*`) before general ones (`/api/*`).
- **CloudFront Functions vs. Lambda@Edge**: Use CloudFront Functions for simple request/response manipulation (headers, redirects, URL rewrites — < 1 ms, ~1/6th the cost). Use Lambda@Edge for complex logic requiring network calls, larger code, or origin-level triggers.
- **WAF on CloudFront** is the most cost-effective protection point — it blocks bad traffic before it reaches any origin, reducing origin compute costs.
- **WebSocket through CloudFront**: Use cache behavior with CachingDisabled policy. CloudFront passes through the WebSocket connection transparently. Idle timeout is 10 seconds by default (configurable up to 180 seconds via origin keep-alive timeout).

## Relevant Links

- [CloudFront Cache Behaviors](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-web-values-specify.html#DownloadDistValuesCacheBehavior)
- [CloudFront WebSocket Support](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-working-with.websockets.html)
- [CloudFront Functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html)
- [Lambda@Edge](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-the-edge.html)
- [AWS WAF Bot Control](https://docs.aws.amazon.com/waf/latest/developerguide/waf-bot-control.html)
- [Global Accelerator vs CloudFront](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html)
