# Q05: Scale-Up vs. Scale-Out Architecture Decision

## Question

A data processing application runs on a single **r6g.16xlarge** EC2 instance (64 vCPUs, 512 GB RAM) performing in-memory analytics on large datasets. The instance is currently at 85% memory utilization during peak processing. The company expects dataset sizes to **double within 6 months**. The application is currently a single monolithic process.

The solutions architect must recommend an approach to handle the growth. Which approach is BEST?

## Options

- **A.** Scale up to **r6g.metal** (64 vCPUs, 512 GB RAM) and then to **x2idn.metal** (128 vCPUs, 2 TB RAM) when needed. Continue running the monolithic application on a single larger instance.
- **B.** Re-architect the application to run as a distributed processing framework using Amazon EMR with Apache Spark. Partition the data and process across a cluster of instances that can scale out horizontally.
- **C.** Move the application to AWS Lambda with provisioned concurrency to leverage serverless scaling.
- **D.** Add a second r6g.16xlarge instance and use NFS (Amazon EFS) to share the dataset between both instances, with application logic unchanged.

## Answers

### B. Re-architect with EMR/Spark — ✅ Correct

With data expected to double (approaching 1 TB in memory), a single-instance scale-up approach has a ceiling — the largest memory-optimized instance (x2idn.metal) has 2 TB RAM but represents a hard limit. Re-architecting for horizontal scale-out using a distributed framework like Spark on EMR allows the application to partition data across many nodes and scale **linearly** with data growth. While this requires upfront engineering effort, it's the only approach that provides long-term, unbounded scalability.

### A. Scale up — ❌ Incorrect

Scaling up works short-term (moving to x2idn.metal with 2 TB RAM handles the immediate doubling), but it has fundamental limitations:
1. There's a **maximum instance size ceiling** — you can't scale up indefinitely.
2. Single point of failure — one instance means one AZ, one host.
3. Cost-inefficiency — memory-optimized very large instances are expensive even during low-usage periods.
4. The question implies continued growth beyond 6 months, making scale-up a temporary fix.

### C. Lambda — ❌ Incorrect

Lambda has a **10 GB memory limit** per function, which is far below the 512 GB currently needed. Lambda functions have a 15-minute execution timeout. In-memory analytics on large datasets is fundamentally incompatible with Lambda's execution model and resource limits.

### D. Two instances with EFS — ❌ Incorrect

Simply adding a second instance and sharing data over NFS doesn't help because the **application is monolithic** — it's a single process that needs all data in memory. The application can't magically split its processing across two instances without re-architecture. EFS also has higher latency than local memory, so it would significantly degrade in-memory analytics performance.

## Recommendations

- **Scale-up vs. Scale-out decision tree:**
  - Scale-up when: workload is monolithic, growth is bounded, and larger instance types are available
  - Scale-out when: growth is unbounded, you need fault tolerance, or you're approaching instance size limits
- **Memory-optimized instance families** to know: R-series (general memory), X-series (extreme memory, up to 2–4 TB), High Memory instances (up to 24 TB for SAP HANA)
- For analytics workloads, the exam favors **distributed architectures** (EMR, Glue, Redshift) over single-instance scale-up.
- Watch for the **"dataset is growing" cue** — this almost always points to a scale-out solution.
- **EFS** is not a substitute for distributed processing. Shared storage ≠ distributed compute.

## Relevant Links

- [Amazon EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [Amazon EMR](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-what-is-emr.html)
- [Scale-Up vs. Scale-Out](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/design-your-workload-to-adapt-to-changes-in-demand.html)
- [Apache Spark on EMR](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-spark.html)
