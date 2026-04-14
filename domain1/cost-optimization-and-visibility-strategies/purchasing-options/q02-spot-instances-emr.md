# Q02: Spot Instances for Fault-Tolerant Workloads

## Question

A data analytics company processes large datasets using Apache Spark on Amazon EMR. The processing jobs run nightly, take approximately 3 hours, and can tolerate individual task failures with automatic retry logic. The company wants to reduce compute costs by at least 60% compared to On-Demand pricing. The jobs must complete within a 6-hour window each night.

Which approach should the solutions architect recommend?

## Options

- **A.** Use Spot Instances for all EMR nodes (master, core, and task groups). Configure the EMR cluster with a single instance fleet using the lowest-priced instance type available.
- **B.** Use On-Demand Instances for the EMR master node and a mix of On-Demand and Spot Instances for core nodes. Use Spot Instances exclusively for task nodes. Configure instance fleets with multiple instance types and enable managed scaling.
- **C.** Purchase 1-year Reserved Instances for the average nightly cluster size. Use On-Demand Instances for any additional capacity needed above the reserved baseline.
- **D.** Use Spot Instances for all EMR nodes with a single instance type. Set the maximum Spot price to 100% of the On-Demand price to minimize interruptions.

## Answers

### B. On-Demand master + mixed core + Spot task nodes — ✅ Correct

This approach balances cost savings with reliability:
- **Master node on On-Demand**: The master node runs critical cluster services (YARN ResourceManager, HDFS NameNode). Losing the master terminates the entire cluster.
- **Core nodes mixed**: Core nodes store HDFS data. A mix of On-Demand and Spot provides a stable HDFS baseline while gaining Spot savings.
- **Task nodes on Spot**: Task nodes only run compute tasks (no HDFS storage). Spot interruptions cause task retries, not data loss. This is where the biggest Spot savings come from.
- **Instance fleets with multiple types**: Using multiple instance types across multiple AZs maximizes Spot availability and reduces interruption probability.
- **Managed scaling**: Automatically adds/removes nodes based on workload, optimizing cost.

### A. All Spot — ❌ Incorrect

Using Spot for the master node is extremely risky. If the master is interrupted, the entire EMR cluster terminates and the job must restart from scratch. This could cause the job to miss the 6-hour processing window, especially during periods of high Spot interruption rates.

### C. Reserved Instances — ❌ Incorrect

The workload runs approximately 3 hours nightly (~90 hours/month out of ~730 total hours). Reserved Instances are priced for continuous usage and would only provide savings if the instances ran most of the time. For a 3-hour nightly batch, Spot (at 60-90% discount) is far more cost-effective than RIs (at 30-72% discount on 24/7 pricing).

### D. Single instance type Spot — ❌ Incorrect

Using a single Spot instance type significantly increases the risk of capacity unavailability. If that specific instance type has high demand, all nodes could be interrupted simultaneously. AWS best practice for Spot is to diversify across multiple instance types and Availability Zones. Also, the maximum Spot price doesn't reduce interruptions — Spot instances are interrupted based on capacity demand, not price.

## Recommendations

- Always protect the **EMR master node** with On-Demand or Reserved capacity.
- Use **instance fleets** (not instance groups) for Spot workloads — they support multiple instance types and Spot allocation strategies.
- Enable **EMR managed scaling** to dynamically right-size the cluster based on YARN metrics.
- For Spot, diversify across at least 5-10 instance types of similar vCPU/memory ratios.
- Spot savings are ideal for **short-running, fault-tolerant** batch workloads, not 24/7 steady-state workloads.

## Relevant Links

- [EMR Instance Fleets](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-instance-fleet.html)
- [Best Practices for Spot Instances on EMR](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-plan-spot-instances.html)
- [Amazon EC2 Spot Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-spot-instances.html)
- [EMR Managed Scaling](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-managed-scaling.html)
