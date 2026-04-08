# AWS Global Accelerator

## Service Overview
Global Accelerator provides static anycast IP addresses and intelligent routing to improve availability and performance for global applications.

## Tasks this service can solve
- Improve global app performance and failover with static IPs
- Route traffic to the nearest healthy regional endpoint

## Alternatives
| Name | Type | Short description | Pro vs Global Accelerator | Cons vs Global Accelerator | Price comparison |
|---|---|---:|---|---|---|
| CloudFront (for HTTP) | AWS | CDN with edge caching | Caching at edge for HTTP content | Not a global static IP service | CloudFront cheaper for cached HTTP static content
| DNS-based routing (Route 53) | AWS | DNS-based routing and health checks | Lower cost | DNS propagation and TTL issues | Route 53 cheaper but less responsive failover

## Limitations
- Additional per-accelerator fees and data transfer charges; best for apps needing static IPs and global routing.

## Price info
- Charged per accelerator and for data transfer across the global network.

## Network & Multi-region considerations
- Deploy regional endpoints (ALB, NLB, EC2) and let Global Accelerator route to closest healthy endpoint; combine with regional failover strategies.

## When Not to Use This Service
- Not necessary for single-region apps or when static IPs are not required; prefer DNS-based routing or CloudFront for HTTP caching.

## DR strategy
- Use multiple regional endpoints and enable endpoint health checks; maintain IaC to recreate accelerators and endpoints.

## Popular use cases
- Gaming backends, global APIs requiring static IPs, and multi-region failover for corporate services.
