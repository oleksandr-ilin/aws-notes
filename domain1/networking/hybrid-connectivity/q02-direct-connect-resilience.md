# Q02: Direct Connect Resilience Architectures

## Question

A healthcare company processes sensitive patient data and requires a hybrid connectivity solution rated for "maximum resilience" according to AWS recommendations. The company has TWO data centers (one primary in New York, one secondary in Chicago). The RTO for connectivity loss is **0 minutes** (no downtime tolerance). The company needs to connect to VPCs in us-east-1.

Which Direct Connect architecture provides maximum resilience?

## Options

- **A.** Provision one Direct Connect connection from the New York data center to one DX location. Provision a second Direct Connect connection from the same data center to the same DX location. Configure both connections with BGP for active/passive failover.
- **B.** Provision Direct Connect connections from the New York data center to TWO different DX locations near New York. Provision additional Direct Connect connections from the Chicago data center to TWO different DX locations near Chicago. Configure all four connections with BGP for active/active load sharing.
- **C.** Provision one Direct Connect connection from the New York data center and one from the Chicago data center, both to the same DX location in New York. Configure BGP for failover.
- **D.** Provision one Direct Connect connection from the New York data center. Set up two AWS Site-to-Site VPN connections from both data centers as backup. Configure BGP with VPN as lower priority.

## Answers

### B. Four connections across four DX locations from two data centers — ✅ Correct

This is the **AWS-recommended "maximum resilience"** architecture:
- **Two data centers**: Protects against a single-site disaster (fire, power outage).
- **Four DX locations** (two per data center): Protects against a single DX location failure (facility issue, fiber cut).
- **Four total connections**: Even if one data center AND one DX location fail simultaneously, connectivity remains via the surviving path.
- **Active/active BGP**: All connections share traffic — no capacity is wasted waiting for failover. Traffic automatically reroutes around failures without downtime.
- This meets the **0-minute RTO** requirement because there's always at least one active path.

### A. Two connections, one DX location — ❌ Incorrect

This is the **"high resilience"** model per AWS — it protects against a single connection failure but not against a DX location failure. If the DX location itself goes down (facility issue, fiber cut at the location), both connections fail. Single data center is also a single point of failure. Does not meet the 0-minute RTO under all failure scenarios.

### C. Two connections, one DX location, two data centers — ❌ Incorrect

Two data centers provides site diversity, but both connections terminating at the **same DX location** creates a single point of failure at the location level. If the DX facility in New York has an outage, both connections are affected regardless of data center diversity.

### D. DX + VPN backup — ❌ Incorrect

VPN over the internet cannot guarantee 0-minute RTO — internet paths may experience latency spikes or packet loss during regional events. VPN also has bandwidth limitations compared to Direct Connect. For zero-tolerance downtime, all connectivity paths should be dedicated (Direct Connect), not shared (internet VPN).

## Recommendations

- AWS defines three resilience levels: **Development** (1 connection), **High** (2 connections, 1 location), **Maximum** (4+ connections, 2+ locations).
- Use **AWS Direct Connect Resiliency Toolkit** in the console to provision resilient architectures with guided setup.
- For maximum resilience, each DX location should use connections with **separate devices** (single-mode vs. LAG considerations).
- **Link Aggregation Groups (LAGs)** aggregate multiple connections at a single location but don't add location diversity.
- Consider the cost: 4× DX connections is expensive. Evaluate whether **"high resilience" + VPN backup** meets actual business requirements instead of maximum resilience.

## Relevant Links

- [Direct Connect Resiliency Recommendations](https://docs.aws.amazon.com/directconnect/latest/UserGuide/resilency_toolkit.html)
- [Maximum Resiliency Model](https://docs.aws.amazon.com/directconnect/latest/UserGuide/maximum_resiliency.html)
- [High Resiliency Model](https://docs.aws.amazon.com/directconnect/latest/UserGuide/high_resiliency.html)
- [LAG on Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/lags.html)
