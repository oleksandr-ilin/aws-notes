# Q06: DR Strategy Selection Based on Cost and Requirements

## Question

A company is evaluating disaster recovery strategies for four different workloads. Match each workload to the MOST cost-effective DR strategy that meets its requirements.

**Workload A:** Internal HR portal — RTO: 24 hours, RPO: 24 hours, Budget: Minimal
**Workload B:** Customer-facing API — RTO: 4 hours, RPO: 15 minutes, Budget: Moderate
**Workload C:** Real-time payment processing — RTO: 1 minute, RPO: 0, Budget: High
**Workload D:** Order management system — RTO: 30 minutes, RPO: 5 minutes, Budget: Moderate-High

Which mapping is correct?

## Options

- **A.** A: Backup & Restore, B: Pilot Light, C: Multi-site Active/Active, D: Warm Standby
- **B.** A: Pilot Light, B: Warm Standby, C: Multi-site Active/Active, D: Pilot Light
- **C.** A: Backup & Restore, B: Warm Standby, C: Warm Standby, D: Multi-site Active/Active
- **D.** A: Backup & Restore, B: Pilot Light, C: Warm Standby, D: Multi-site Active/Active

## Answers

### A. Correct mapping — ✅ Correct

| Workload | Strategy | Reasoning |
|----------|----------|-----------|
| A (HR portal) | **Backup & Restore** | 24h RTO/RPO with minimal budget — daily backups + restore on demand is cheapest |
| B (Customer API) | **Pilot Light** | 4h RTO gives time to launch infrastructure; 15m RPO achieved via DB replication; moderate cost |
| C (Payments) | **Multi-site Active/Active** | 1-minute RTO and zero RPO requires both Regions fully active; high budget supports this |
| D (Orders) | **Warm Standby** | 30m RTO needs pre-running infrastructure; 5m RPO needs continuous DB replication; moderate-high budget fits |

### B. — ❌ Incorrect

Puts the HR portal (24h RTO) in pilot light — unnecessarily expensive. Also puts the order management system in pilot light, which can't reliably meet a 30-minute RTO (launching instances + health checks + LB warm-up takes longer).

### C. — ❌ Incorrect

Assigns warm standby to the payment system (1-minute RTO). Warm standby requires scaling up during failover, which takes minutes — not achievable in 1 minute. Assigns multi-site to orders (30m RTO), which is unnecessarily expensive for a 30-minute RTO tolerance.

### D. — ❌ Incorrect

Assigns warm standby to payments (1-minute RTO) — same issue as option C. Also assigns multi-site to orders, which is over-provisioned for the 30-minute RTO requirement.

## Recommendations

- **DR strategy cost spectrum** (cheapest → most expensive): Backup & Restore → Pilot Light → Warm Standby → Multi-site Active/Active
- **RTO mapping** (approximate):
  - Backup & Restore: hours to days
  - Pilot Light: 1–4 hours
  - Warm Standby: 10–30 minutes
  - Multi-site: near-zero to minutes
- Always select the **cheapest strategy that meets requirements** — the exam often tests whether you over-provision DR.
- Remember: budget is always a factor on the SAP exam. "Most cost-effective" means the least expensive option that still meets all requirements.

## Relevant Links

- [DR Strategies Comparison](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)
- [AWS Well-Architected — Reliability Pillar — Plan for DR](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/plan-for-disaster-recovery-dr.html)
- [AWS Disaster Recovery Blog](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-i-strategies-for-recovery-in-the-cloud/)
