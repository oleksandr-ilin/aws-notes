# Q01: VPC Flow Logs and Traffic Mirroring

## Question

A company suspects data exfiltration from an EC2 instance in a production VPC. The security team needs to: identify which IP addresses the instance communicated with over the past week, determine the volume of data transferred to each destination, perform deep packet inspection on the instance's network traffic going forward, and all data must be stored for forensic analysis.

Which combination of services should the solutions architect recommend? **(Select TWO.)**

## Options

- **A.** Enable VPC Flow Logs on the instance's ENI, publishing to Amazon S3. Use Amazon Athena to query the flow logs for source/destination IP addresses and byte counts over the past week.
- **B.** Enable Amazon VPC Traffic Mirroring on the instance's ENI. Configure a traffic mirror target pointing to a Network Load Balancer fronting EC2 instances running network analysis tools (e.g., Zeek, Suricata). Store captured packets in Amazon S3.
- **C.** Enable AWS CloudTrail data events for the instance to capture all network communications and packet payloads.
- **D.** Install Amazon Inspector on the instance to monitor real-time network traffic and detect data exfiltration patterns.
- **E.** Enable Amazon GuardDuty to perform deep packet inspection and identify the exfiltration destination.

## Answers

### A. VPC Flow Logs + Athena — ✅ Correct

VPC Flow Logs capture metadata about network traffic (source IP, destination IP, port, protocol, bytes transferred, action). They are ideal for:
- **Retrospective analysis**: Flow logs for the past week can identify which external IPs the instance communicated with.
- **Data volume**: Byte counts show how much data was transferred to each destination.
- **Athena querying**: Flow logs published to S3 can be queried with SQL for ad-hoc forensic analysis.
- Flow logs capture **metadata only** (not packet contents), which is sufficient for identifying communication patterns.

### B. VPC Traffic Mirroring — ✅ Correct

Traffic Mirroring provides **deep packet inspection** by copying actual network traffic:
- Mirrors inbound and/or outbound traffic from the instance's ENI.
- Traffic is sent to a mirror target (NLB, ENI, or Gateway Load Balancer).
- Analysis tools (Zeek, Suricata, Wireshark) can inspect packet payloads for content analysis.
- This goes beyond flow logs — you can see the actual data being exfiltrated, not just the IP addresses.
- Traffic Mirroring must be configured going forward (it doesn't capture historical traffic).

### C. CloudTrail for network traffic — ❌ Incorrect

CloudTrail captures **AWS API calls** (e.g., RunInstances, PutObject), not network traffic. Data events capture S3 object-level and Lambda invocation events, not network packets or IP communications. CloudTrail is not a network monitoring tool.

### D. Amazon Inspector — ❌ Incorrect

Inspector is a vulnerability scanner that detects CVEs, exposed ports, and software vulnerabilities. It does not monitor real-time network traffic or detect data exfiltration. It performs periodic scans, not continuous traffic analysis.

### E. GuardDuty for deep packet inspection — ❌ Incorrect

GuardDuty analyzes VPC Flow Logs, DNS logs, and CloudTrail events to detect threats (including data exfiltration patterns) using machine learning. However, it does **not** perform deep packet inspection — it works on metadata. It can detect anomalous traffic patterns but cannot inspect packet payloads.

## Recommendations

- **VPC Flow Logs** = metadata (who talked to whom, how much). Use for initial investigation and historical analysis.
- **Traffic Mirroring** = full packets (what was said). Use for deep forensic investigation and ongoing threat detection.
- Traffic Mirroring is supported on **Nitro-based instances** only. Verify instance type compatibility.
- For cost optimization, use **Traffic Mirroring filter rules** to capture only the traffic you need (specific ports, protocols, or directions).
- Combine with **GuardDuty** for automated threat detection based on flow log analysis — it can identify potential exfiltration without manual investigation.
- Store all forensic data (flow logs, mirrored packets) in an immutable S3 bucket with Object Lock.

## Relevant Links

- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [VPC Traffic Mirroring](https://docs.aws.amazon.com/vpc/latest/mirroring/what-is-traffic-mirroring.html)
- [Querying Flow Logs with Athena](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-athena.html)
- [Traffic Mirroring Targets](https://docs.aws.amazon.com/vpc/latest/mirroring/traffic-mirroring-targets.html)
