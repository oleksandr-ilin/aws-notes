# Amazon Route 53

## Service Overview
Amazon Route 53 is a highly-available DNS service providing authoritative DNS, routing policies, and health checks.

## Tasks this service can solve
- Authoritative DNS for domains and routing policies (latency, geolocation)
- DNS-based failover and health checking

## Alternatives
| Name | Type | Short description | Pro vs Route 53 | Cons vs Route 53 | Price comparison |
|---|---|---:|---|---|---|
| Cloudflare / DNS providers | Commercial | DNS and performance features | Additional DDoS and edge features | External vendor | Pricing varies; Route 53 per-query costs apply
| Self-hosted DNS | OSS | Custom DNS operations | Full control | Availability and ops burden | Ops cost vs managed Route 53

## Limitations
- Per-query charges can add up for very high query volumes; TTL and DNS propagation constraints apply.

## Price info
- Charged per hosted zone and per DNS query; additional charges for health checks and domain registration.

## Network & Multi-region considerations
- Route 53 is global; manage health checks to route traffic to healthy regional endpoints and use failover or latency-based routing.

## When Not to Use This Service
- Not suitable if you rely on advanced CDN + DNS integrations from third-party providers with extra edge features; use provider-based DNS.

## DR strategy
- Export DNS zone files and keep domain registrar details and credentials in secure vaults; maintain secondary DNS providers if needed.

## Popular use cases
- Global traffic steering, DNS-based failover, and domain management for public services.
