# Amazon CloudFront

## Service Overview
Amazon CloudFront is a global CDN that caches and delivers content (static and dynamic) from edge locations to reduce latency.

## Tasks this service can solve
- Global static asset delivery (web, images, JS/CSS)
- Dynamic site acceleration and API caching
- Protect origin with WAF and origin access controls

## Alternatives
| Name | Type | Short description | Pro vs CloudFront | Cons vs CloudFront | Price comparison |
|---|---|---:|---|---|---|
| Cloudflare / Fastly | Commercial | Global CDN and edge compute | Rich edge features and routing | External vendor | Comparable; depends on traffic and feature set
| API Gateway + Regional Cache | AWS | Regional API caching | Simpler per-API caching | Not global edge caching | Often cheaper for small API traffic

## Limitations
- Cache invalidation costs and TTL management required; large dynamic payloads reduce caching effectiveness.

## Price info
- Charged for data transfer out, requests at edge, and optional features like field-level encryption.

## Network & Multi-region considerations
- Global by design; configure regional origins for failover and origin replication. Use origin groups and signed URLs for origin protection.

## When Not to Use This Service
- Not needed for purely internal low-latency intra-VPC traffic; use internal load balancing or regional caching.

## DR strategy
- Use multi-origin configurations and replicate origin data to alternate regions; maintain IaC for distributions to recreate quickly.

## Popular use cases
- Website static asset hosting, video streaming, API acceleration, and edge security with WAF integration.
