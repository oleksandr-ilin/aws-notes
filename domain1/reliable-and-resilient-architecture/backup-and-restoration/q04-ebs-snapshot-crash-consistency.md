# Q04: EBS Snapshot Lifecycle and Crash-Consistent Backups

## Question

A company runs a critical Oracle database on an Amazon EC2 instance with **four Amazon EBS volumes** configured in a RAID-0 stripe set. The database writes span across all four volumes simultaneously. The operations team currently takes EBS snapshots of each volume manually at different times during the day.

After a recent restore attempt, the team discovered that the database was **corrupted** because the four snapshots were taken at different points in time, resulting in inconsistent data across the stripe set.

How should the solutions architect ensure **crash-consistent snapshots** of the RAID-0 array? (Select TWO)

## Options

- **A.** Use **Amazon Data Lifecycle Manager (DLM)** with a multi-volume snapshot policy that creates **application-consistent snapshots** of all EBS volumes attached to the instance simultaneously.
- **B.** Stop the EC2 instance before taking snapshots of all four volumes, then restart it.
- **C.** Use the `create-snapshots` API (multi-volume snapshot) to snapshot all four EBS volumes as a **single coordinated set**, ensuring point-in-time consistency across the stripe set.
- **D.** Take an AMI of the instance, which automatically snapshots all attached EBS volumes.
- **E.** Freeze the filesystem I/O using `fsfreeze` before taking snapshots, then thaw after all snapshots are initiated.
- **F.** Increase the RAID level from RAID-0 to RAID-1 for redundancy.

## Answers

### C. Multi-volume snapshots via create-snapshots API — ✅ Correct

The `create-snapshots` API creates **crash-consistent snapshots** of multiple EBS volumes simultaneously. All volumes are snapshotted at the same point in time, ensuring the stripe set data is consistent. This is specifically designed for multi-volume workloads like RAID arrays.

### E. Freeze filesystem before snapshots — ✅ Correct

Using `fsfreeze --freeze` before initiating snapshots ensures the filesystem flushes all pending writes and stops accepting new writes. This elevates the snapshot from **crash-consistent** to **application-consistent** by ensuring no partial writes are in flight. After snapshots are initiated, `fsfreeze --thaw` resumes I/O. For databases, you should also put the database in backup mode before freezing.

### A. DLM multi-volume snapshot — ❌ Incorrect (partially valid)

DLM supports multi-volume snapshot policies that coordinate snapshots across all volumes tagged for a specific instance. However, DLM creates **crash-consistent** snapshots, not application-consistent ones (as the option claims). DLM doesn't natively integrate with `fsfreeze` or database backup modes. While DLM is useful for automating the schedule, option C directly addresses the consistency problem.

### B. Stop the instance — ❌ Incorrect

Stopping the instance ensures all data is flushed to disk and provides fully consistent snapshots. However, this causes **downtime** for a critical database, which is unacceptable for ongoing backup operations. This approach is valid for non-production systems but not for production databases.

### D. Create an AMI — ❌ Incorrect

Creating an AMI does snapshot all attached volumes, but for a **running instance** with a RAID-0 array, the individual volume snapshots within the AMI may still not be perfectly synchronized at the I/O level. AMI creation with `--no-reboot` doesn't guarantee multi-volume point-in-time consistency. The `create-snapshots` API is purpose-built for this.

### F. Change RAID level — ❌ Incorrect

Changing from RAID-0 to RAID-1 adds redundancy (mirroring) but doesn't solve the snapshot consistency problem. RAID-1 protects against **disk failure**, not against **inconsistent backups**. You'd still need coordinated snapshots across the mirror volumes.

## Recommendations

- **Multi-volume snapshots** (`create-snapshots` API) are essential for RAID arrays and any workload striping data across multiple EBS volumes.
- **Consistency levels:**
  - **Crash-consistent:** All volumes captured at same point in time (multi-volume snapshot)
  - **Application-consistent:** Filesystem frozen + database in backup mode + multi-volume snapshot
- **Amazon Data Lifecycle Manager** automates snapshot scheduling and retention. Use it to automate the `create-snapshots` calls on a schedule.
- For **Oracle on EC2**, the best practice is: Put Oracle in backup mode → `fsfreeze` → multi-volume snapshot → `fsfreeze --thaw` → end Oracle backup mode.
- **RAID-0 = no redundancy.** If a single EBS volume fails, all data is lost. Consider RAID-10 for production databases.

## Relevant Links

- [EBS Multi-Volume Snapshots](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-snapshot.html#ebs-create-snapshot-multi-volume)
- [Amazon Data Lifecycle Manager](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle.html)
- [EBS RAID Configuration](https://docs.aws.amazon.com/ebs/latest/userguide/raid-config.html)
- [Oracle Backup Best Practices on AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/oracle-on-ec2/)
