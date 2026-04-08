# AWS VPN

## Service Overview
AWS VPN provides encrypted site-to-site and client VPN connectivity between on-premises networks and AWS.

## Tasks this service can solve
- Secure connectivity for remote sites and users to AWS resources
- Quick, encrypted connectivity for hybrid scenarios

## Alternatives
| Name | Type | Short description | Pro vs VPN | Cons vs VPN | Price comparison |
|---|---|---:|---|---|---|
| Direct Connect | AWS | Dedicated connectivity | More predictable performance | Higher setup/cost | Direct Connect more expensive but consistent
| Third-party VPN appliances | Commercial | Advanced VPN features | Richer features | License and ops cost | Varies by vendor

## Limitations
- Internet-based tunnels can be less predictable; throughput and latency constrained by ISP links.

## Price info
- Charged per VPN connection and data transfer; client VPN has per-hour and per-connection charges.

## Network & Multi-region considerations
- Use redundant tunnels and BGP for failover; combine with Direct Connect for hybrid redundancy.

## When Not to Use This Service
- For very high-throughput, low-latency connections prefer Direct Connect.

## DR strategy
- Configure redundant VPN tunnels to multiple endpoints and automate BGP failover; have playbooks to switch to alternative connections.

## Popular use cases
- Office-to-cloud connectivity, remote-access VPN for admins, and quick hybrid connectivity during migrations.
