# Q03: Guardrails — Preventive vs. Detective

## Question

A company using AWS Control Tower needs to implement the following governance requirements:
1. No AWS account should be able to delete CloudTrail logs from the centralized log archive S3 bucket
2. All EBS volumes must be encrypted at rest — violations should be detected and reported
3. EC2 instances should only be launched in approved Regions (us-east-1 and eu-west-1)
4. S3 buckets with public access should be flagged for remediation, not blocked

Which combination of Control Tower guardrail types should the solutions architect apply for each requirement?

## Options

- **A.** Requirement 1: Preventive guardrail. Requirement 2: Detective guardrail. Requirement 3: Preventive guardrail. Requirement 4: Detective guardrail.
- **B.** Requirement 1: Detective guardrail. Requirement 2: Preventive guardrail. Requirement 3: Detective guardrail. Requirement 4: Preventive guardrail.
- **C.** All four requirements should use preventive guardrails (SCPs) to enforce compliance proactively.
- **D.** All four requirements should use detective guardrails (AWS Config rules) to monitor compliance and generate reports.

## Answers

### A. Preventive (1, 3) + Detective (2, 4) — ✅ Correct

The correct mapping aligns the guardrail type with the enforcement model:

**Requirement 1 — Preventive (SCP)**: Preventing CloudTrail log deletion is a hard security boundary. An SCP denying `s3:DeleteObject` and `s3:DeleteBucket` on the log archive bucket ensures no account — regardless of IAM permissions — can delete logs. This must be proactively blocked.

**Requirement 2 — Detective (Config rule)**: "All EBS volumes must be encrypted" is best monitored via the `encrypted-volumes` AWS Config rule. Detective guardrails flag non-compliant volumes for remediation. While you could use an SCP to prevent creating unencrypted volumes, the question says "violations should be detected and reported" — a detective guardrail.

**Requirement 3 — Preventive (SCP)**: Region restriction is a preventive control. An SCP with `aws:RequestedRegion` condition key denies API calls outside us-east-1 and eu-west-1. This proactively prevents any resource creation in unapproved Regions.

**Requirement 4 — Detective (Config rule)**: The question explicitly says "flagged for remediation, not blocked." This is a detective control — the `s3-bucket-public-read-prohibited` Config rule detects public buckets and reports them. If blocking were required, a preventive SCP would be appropriate.

### B. Reversed mapping — ❌ Incorrect

This reverses the guardrail types for each requirement. Detecting CloudTrail deletion after it happens defeats the purpose — logs would already be gone. Making EBS encryption a preventive control could work but doesn't match the "detected and reported" language. Region restriction as detective-only would allow resources to be created in unauthorized Regions and only flag them afterward.

### C. All preventive — ❌ Incorrect

While preventive controls are strongest, requirement 2 specifically asks for detection/reporting, and requirement 4 explicitly asks for flagging, not blocking. Using preventive controls where detective is appropriate adds unnecessary rigidity and doesn't meet the stated requirements.

### D. All detective — ❌ Incorrect

Detective controls for requirements 1 and 3 are insufficient. Detecting log deletion after it occurs means evidence may already be destroyed. Detecting resources in unauthorized Regions after creation means data may already be in a non-compliant location, requiring costly cleanup.

## Recommendations

- **Preventive guardrails** (SCPs) are best for: hard security boundaries, irreversible actions (deletion), and compliance-critical restrictions (Region, service).
- **Detective guardrails** (Config rules) are best for: configuration compliance monitoring, remediation workflows, and "observe-then-fix" scenarios.
- Control Tower provides **mandatory guardrails** that are always enabled (e.g., disallow log archive deletion) — these cannot be disabled.
- **Proactive guardrails** (newer category) use CloudFormation hooks to check resources before provisioning — a middle ground between preventive and detective.
- Layer guardrails: use preventive SCPs for hard limits, detective Config rules for visibility, and proactive hooks for deployment-time validation.

## Relevant Links

- [Control Tower Guardrails](https://docs.aws.amazon.com/controltower/latest/userguide/guardrails.html)
- [Preventive Guardrails](https://docs.aws.amazon.com/controltower/latest/userguide/preventive-guardrails.html)
- [Detective Guardrails](https://docs.aws.amazon.com/controltower/latest/userguide/detective-guardrails.html)
- [Proactive Guardrails](https://docs.aws.amazon.com/controltower/latest/userguide/proactive-guardrails.html)
