# Q03: Reserved Instances for Database Services — RDS, Redshift, ElastiCache, OpenSearch

## Question

A company runs a data platform with four database services that have been running continuously for 6 months with stable, predictable usage:

| Service | Instance/Node Type | Count | Monthly On-Demand Cost | Usage Pattern |
|---|---|---|---|---|
| RDS PostgreSQL Multi-AZ | db.r6g.2xlarge | 2 (primary + standby) | $4,200 | 24/7, steady for 2+ years |
| Amazon Redshift | ra3.4xlarge | 6-node cluster | $9,400 | 24/7, expanding to 8 nodes in 6 months |
| ElastiCache Redis Multi-AZ | r6g.xlarge | 3 (1 primary + 2 replicas) | $1,800 | 24/7, stable cluster size |
| Amazon OpenSearch | r6g.2xlarge.search | 3 data nodes + 3 UltraWarm | $3,600 | 24/7, log analytics, growing 20%/year |

Total monthly On-Demand cost: $19,000 ($228,000/year). The CFO wants to reduce database costs by at least 35% over a 3-year commitment. The data team plans two changes in the next 3 years:
1. Redshift cluster may grow from 6 to 10 nodes as data volume grows
2. RDS might migrate from PostgreSQL to Aurora PostgreSQL within 18 months (under evaluation)

Which pricing strategy meets the 35% savings target while accommodating the planned changes?

## Options

- **A.** RDS: **Convertible Reserved Instances** (3-year, Partial Upfront) — Convertible RIs can be exchanged for a different instance type, size, or engine (e.g., exchange db.r6g.2xlarge PostgreSQL RI for db.r6g.2xlarge Aurora PostgreSQL RI when migrating). Redshift: **Standard Reserved Instances** (1-year, Partial Upfront) for the current 6 nodes only — renew after Year 1 and add RIs for additional nodes. Redshift RIs are specific to the node type. ElastiCache: **Standard Reserved Instances** (3-year, All Upfront) — the cluster is stable, maximum discount. ElastiCache RIs cover both primary and replica nodes. OpenSearch: **Standard Reserved Instances** (3-year, Partial Upfront) for the 3 data nodes — UltraWarm nodes have separate Reserved pricing.
- **B.** Apply a single **Compute Savings Plan** to cover all four services. Compute Savings Plans cover EC2, Lambda, and Fargate across all Regions and families.
- **C.** Purchase **Standard Reserved Instances** (3-year, All Upfront) for ALL services at their current sizes. Maximum discount, fixed commitment.
- **D.** Use **On-Demand** for all services and apply **AWS Budgets** alerts to control spending. Negotiate an Enterprise Discount Program (EDP) with AWS for volume discounts.

## Answers

### A. Convertible RI (RDS) + 1-year Standard RI (Redshift) + 3-year Standard RI (ElastiCache) + 3-year Standard RI (OpenSearch) — ✅ Correct

This strategy applies the optimal RI type per service based on change likelihood:

**RDS PostgreSQL — Convertible RIs (3-year):**

- **Why Convertible, not Standard**: The team may migrate from RDS PostgreSQL to Aurora PostgreSQL within 18 months. Standard RIs are locked to the specific engine — a PostgreSQL Standard RI cannot be applied to Aurora PostgreSQL. Convertible RIs can be **exchanged** for a different:
  - Engine (PostgreSQL → Aurora PostgreSQL)
  - Instance family (r6g → r7g)
  - Instance size (2xlarge → 4xlarge)
  - The exchange must be for an RI of equal or greater value (you can "upgrade" but not "downgrade").

- **Convertible RI savings**: 3-year Convertible Partial Upfront saves ~36-40% vs. On-Demand (compared to ~42-47% for Standard 3-year). The flexibility to exchange engines justifies the slightly lower discount.

- **Multi-AZ coverage**: An RDS RI for a Multi-AZ deployment covers both the primary and standby instances. You need 1 RI for a Multi-AZ pair (not 2). The RI pricing for Multi-AZ is approximately 2x the Single-AZ RI price (it covers both instances).

- **Savings calculation**: $4,200/month × 40% = $1,680/month savings = $20,160/year.

**Redshift — 1-year Standard RIs (renewing):**

- **Why 1-year, not 3-year**: The cluster is growing from 6 to 10 nodes in 6 months. A 3-year RI for 6 nodes locks you into a fixed commitment — you can't convert Redshift Standard RIs (Redshift doesn't offer Convertible RIs). By purchasing 1-year RIs:
  - Year 1: 6 RIs (cover current cluster). After 6 months, add 2-4 On-Demand nodes for the expansion.
  - Year 2: Renew 6 RIs + purchase 4 more RIs (cover expanded 10-node cluster).
  - This gives flexibility to match RI count to actual cluster size at each renewal.

- **Redshift RI specifics**: Redshift RIs are per-node-type (not convertible). ra3.4xlarge RI covers ra3.4xlarge nodes only. If you upgrade to ra3.16xlarge, existing RIs don't apply.

- **Savings**: 1-year Partial Upfront Redshift RI saves ~23-25% vs. On-Demand. 3-year would save ~40%, but inflexibility risk outweighs the additional discount given the planned growth.

- **Note**: Redshift Serverless is an alternative that eliminates RI planning entirely — pay per RPU-hour. For predictable, always-on clusters, provisioned + RIs is typically cheaper.

**ElastiCache Redis — 3-year Standard RIs (All Upfront):**

- **Why 3-year All Upfront**: The cluster is stable (no planned changes). 3-year All Upfront provides the maximum discount (~50-55% vs. On-Demand for ElastiCache).

- **ElastiCache RI coverage**: ElastiCache RIs are per cache node. A 3-node cluster (1 primary + 2 replicas) requires 3 RIs. Each RI covers one r6g.xlarge node regardless of whether it's primary or replica.

- **Region-specific**: ElastiCache RIs apply to a specific Region only. If you add a Redis Global Datastore replica in another Region, you need separate RIs for that Region.

- **Savings**: $1,800/month × 55% = $990/month savings = $11,880/year.

**OpenSearch — 3-year Standard RIs (Partial Upfront):**

- **Data nodes vs. UltraWarm nodes**: OpenSearch RIs are separate for data nodes (r6g.2xlarge.search) and UltraWarm nodes (ultrawarm1.medium.search). Purchase RIs for each node type independently.

- **Why Partial Upfront (not All Upfront)**: The domain is growing 20%/year. In 2-3 years, you may need larger or additional nodes. Partial Upfront reduces the cash outlay while still providing ~45-50% savings. If the domain needs to be resized, the financial exposure is lower than All Upfront.

- **Savings**: $3,600/month × 48% = $1,728/month savings = $20,736/year.

**Total savings**:
- RDS: ~$20,160/year
- Redshift: ~$27,000/year (blended 1-year RIs + partial On-Demand for growth)
- ElastiCache: ~$11,880/year
- OpenSearch: ~$20,736/year
- Total: ~$79,776/year vs. $228,000 On-Demand = **35% savings** ✅

**Key RI concepts for the exam:**

| Feature | Standard RI | Convertible RI |
|---|---|---|
| Discount | Higher (42-47% for 3-year) | Lower (36-40% for 3-year) |
| Exchange | Same family only (size flexibility) | Any family, engine, size, OS |
| Available for | RDS, Redshift, ElastiCache, OpenSearch | RDS only (among database services) |
| Marketplace resale | ✅ Can sell unused | ❌ Cannot resell |
| Term | 1-year or 3-year | 1-year or 3-year |

### B. Compute Savings Plan for all services — ❌ Incorrect

- **Compute Savings Plans do NOT cover database services**: Compute Savings Plans apply to: EC2 instances, Lambda, and Fargate. They do NOT apply to: RDS, Aurora, Redshift, ElastiCache, or OpenSearch.
- **No Savings Plan type covers databases**: There is no RDS Savings Plan, Redshift Savings Plan, or ElastiCache Savings Plan. Database services use Reserved Instances (a different pricing model), not Savings Plans.
- **SageMaker Savings Plan** exists for SageMaker ML instances but does not cover databases either.
- This is a common exam distractor — candidates who know Savings Plans are "more flexible than RIs" may incorrectly apply them to databases.

### C. Standard RIs (3-year, All Upfront) for everything at current sizes — ❌ Incorrect

- **RDS Standard RI problem**: If the team migrates to Aurora PostgreSQL in 18 months, the Standard PostgreSQL RI cannot be exchanged for Aurora. The RI continues to charge (or goes unused) for the remaining 18 months — potentially losing savings instead of gaining them. Convertible RIs protect against this engine change.

- **Redshift Standard RI (3-year) problem**: Purchasing 6 RIs for 3 years when the cluster grows to 10 nodes means 4 nodes run at On-Demand prices. If 10 RIs were purchased initially and the cluster doesn't grow as planned, 4 RIs are wasted. 1-year terms allow adjusting RI count to match actual cluster size.

- **All Upfront cash flow**: All Upfront for all services requires a massive upfront payment. For Redshift alone: 6 × ra3.4xlarge × 3-year All Upfront ≈ $180,000 upfront. Total across all services: potentially $300,000-$400,000 upfront. This ties up capital and creates financial risk if any service is downsized or migrated.

- **Maximum discount ≠ optimal strategy**: The highest discount percentage doesn't guarantee the best outcome if requirements change. The RI strategy must balance discount depth with flexibility based on change likelihood per service.

### D. On-Demand + EDP — ❌ Incorrect

- **EDP (Enterprise Discount Program)**: AWS offers volume discounts through EDP for large customers committing to a minimum annual spend (typically $1M+/year). EDP provides 5-15% discounts on overall spend. However:
  - The company's total spend is $228,000/year — likely below EDP eligibility thresholds.
  - Even with 10% EDP discount: $228,000 × 10% = $22,800/year savings (10%). Target is 35% ($79,800) — EDP alone doesn't come close.
  - EDP and RIs/Savings Plans can be combined — EDP discount applies to the remaining On-Demand portion. But using On-Demand + EDP alone is far less effective than RIs for predictable workloads.

- **AWS Budgets**: Budgets alert on spending but don't reduce costs. Budgets tell you "you spent $19,000 this month" — they don't make the instances cheaper. RIs and Savings Plans reduce the unit price; Budgets monitor expenditure.

## Recommendations

- **Database RI decision tree**:
  - Engine change planned → **Convertible RI** (RDS only)
  - Stable, no changes → **Standard RI** (3-year, All Upfront for max discount)
  - Growth expected → **1-year Standard RI** (renew and adjust count annually)
  - Unpredictable usage → **On-Demand** (no commitment)
- **Savings Plans vs. RIs**: Use Savings Plans for EC2/Lambda/Fargate. Use RIs for RDS, Redshift, ElastiCache, OpenSearch. They don't overlap — different services use different pricing models.
- **RI utilization monitoring**: Use Cost Explorer RI Utilization report to track whether purchased RIs are being used. Unused RIs are wasted Money. Standard RIs can be sold on the Reserved Instance Marketplace.
- **Convertible RI exchange rules**: The new RI must be of equal or greater value. You cannot exchange to reduce your commitment — only exchange to a different configuration at the same or higher cost.
- **Size flexibility**: Standard RIs for Linux/Unix in the same instance family have size flexibility within the family. A db.r6g.2xlarge RI can cover 2 × db.r6g.xlarge or 4 × db.r6g.large in the same Region.

## Relevant Links

- [RDS Reserved Instances](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithReservedDBInstances.html)
- [Redshift Reserved Nodes](https://docs.aws.amazon.com/redshift/latest/mgmt/purchase-reserved-node-instance.html)
- [ElastiCache Reserved Nodes](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/reserved-nodes.html)
- [OpenSearch Reserved Instances](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ri.html)
- [Savings Plans vs Reserved Instances](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
- [RI Marketplace](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ri-market-general.html)
