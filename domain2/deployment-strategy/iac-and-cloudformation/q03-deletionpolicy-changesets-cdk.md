# Q03: CloudFormation DeletionPolicy, Change Sets, and CDK Integration

## Question

A company manages 60 CloudFormation stacks. Three recent incidents have exposed process gaps:

1. **Data loss**: A developer deleted a CloudFormation stack during cleanup. The stack included an RDS PostgreSQL database with production data and an S3 bucket with customer uploads. Both resources were destroyed — the database data was unrecoverable because no final snapshot was taken.
2. **Unintended replacement**: A stack update changed the `DBInstanceIdentifier` property on an RDS instance. CloudFormation determined this required replacement — it deleted the old instance and created a new one, causing 45 minutes of downtime and data loss.
3. **Template sprawl**: The team writes CloudFormation YAML by hand. Templates are 2,000+ lines, error-prone, and lack type safety. A recent deployment failed because a parameter type was `String` instead of `Number`, caught only at deploy time.

The team wants to: (a) prevent accidental resource deletion, (b) preview dangerous changes before deploying, and (c) modernize their IaC authoring workflow.

Which combination of CloudFormation features and tools addresses all three issues?

## Options

- **A.** (1) Set `DeletionPolicy: Retain` on the RDS instance and S3 bucket. Set `DeletionPolicy: Snapshot` on the RDS instance as an alternative to automatically create a final snapshot before deletion. Add `UpdateReplacePolicy: Snapshot` on the RDS instance to create a snapshot before replacement during updates. (2) Use CloudFormation Change Sets before every stack update — review the change set to identify `Replacement: True` actions (resource will be deleted and recreated) before executing. Enable stack termination protection on production stacks. (3) Migrate from hand-written YAML to AWS CDK (TypeScript) — CDK provides type-safe constructs, IDE auto-completion, unit testing with assertions, and synthesizes CloudFormation templates. Use `cdk diff` to preview infrastructure changes before deployment.
- **B.** (1) Set `DeletionPolicy: Delete` (default) on all resources — rely on AWS Backup for data recovery after accidental deletion. (2) Always use `--no-execute-changeset` flag and review in the console. (3) Use Terraform instead of CloudFormation for type safety.
- **C.** (1) Set `DeletionPolicy: Retain` on ALL resources in every stack. (2) Use CloudFormation drift detection instead of change sets to preview updates. (3) Use SAM (Serverless Application Model) for all infrastructure, including non-serverless resources.
- **D.** (1) Set `DeletionPolicy: Snapshot` on ALL resources. (2) Use CloudFormation StackSets to deploy changes across environments — StackSets validate changes automatically. (3) Use AWS Proton templates for type-safe infrastructure definitions.

## Answers

### A. DeletionPolicy + UpdateReplacePolicy + Change Sets + Termination Protection + CDK — ✅ Correct

Each feature addresses a specific incident:

- **(1) DeletionPolicy and UpdateReplacePolicy — prevent data loss**:

  **DeletionPolicy options**:
  - `Delete` (default): CloudFormation deletes the resource when the stack is deleted. This caused incident #1 — the RDS and S3 were destroyed.
  - `Retain`: CloudFormation removes the resource from the stack but does NOT delete it from AWS. The database and S3 bucket continue to exist as orphaned resources. Use this when you want to preserve data regardless of stack lifecycle.
  - `Snapshot`: Only supported for RDS, ElastiCache, Neptune, Redshift, and EBS volumes. CloudFormation creates a final snapshot before deleting the resource. For RDS, this means a DB snapshot is created — data can be restored from the snapshot later.

  **Best practice for production stacks**:
  ```yaml
  MyDatabase:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot        # Final snapshot on stack deletion
    UpdateReplacePolicy: Snapshot   # Snapshot before replacement during updates
    Properties:
      DBInstanceIdentifier: prod-db
      ...
  
  MyBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain          # Never delete the bucket (Snapshot not supported for S3)
    Properties:
      BucketName: customer-uploads
  ```

  **UpdateReplacePolicy** (incident #2):
  - When a stack update requires replacing a resource (e.g., changing `DBInstanceIdentifier`), CloudFormation creates the new resource, then deletes the old one.
  - `UpdateReplacePolicy: Snapshot` creates a snapshot of the OLD resource before deleting it during replacement. Without this policy, the old RDS instance is deleted without a snapshot.
  - `UpdateReplacePolicy: Retain` keeps the old resource alive (orphaned) instead of deleting it.

- **(2) Change Sets + Termination Protection — preview dangerous changes**:

  **Change Sets**:
  - A change set shows what CloudFormation WOULD do if you execute a stack update: which resources are `Added`, `Modified`, or `Removed`. Critically, modifications show `Replacement: True` or `Replacement: Conditional` — warning that the resource will be deleted and recreated.
  - Incident #2 would have been caught: the change set would show `DBInstanceIdentifier` modification with `Replacement: True`. The team reviews this BEFORE executing and decides to either accept or modify the template.
  - **Workflow**: `create-change-set` → review in console/CLI → `execute-change-set` or `delete-change-set`. Never use `update-stack` directly in production — always go through change sets.

  **Stack Termination Protection**:
  - Prevents `delete-stack` API calls from succeeding. The developer in incident #1 would have received an error: "Stack termination protection is enabled." Must be explicitly disabled before deletion — adding a deliberate step.
  - Enable on all production stacks: `aws cloudformation update-termination-protection --enable-termination-protection --stack-name prod-stack`.

- **(3) AWS CDK (Cloud Development Kit) — type-safe IaC**:

  **CDK benefits over raw YAML**:
  - **Type safety**: CDK constructs (TypeScript/Python/Java/C#/Go) have typed properties. Passing a `string` to a `number` parameter causes a compile error — caught before synthesis, not at deploy time.
  - **IDE support**: Auto-completion, inline documentation, go-to-definition for construct properties. Writing `new rds.DatabaseInstance(this, 'DB', { ... })` shows all valid properties with types.
  - **Abstractions**: CDK L2/L3 constructs set secure defaults (e.g., `rds.DatabaseInstance` enables encryption, multi-AZ, and DeletionPolicy by default). Reduces 2,000-line YAML to 200-line TypeScript.
  - **`cdk diff`**: Shows a human-readable diff of infrastructure changes before deployment — similar to change sets but with CDK-level construct names (e.g., "MyDatabase will be replaced" instead of raw CloudFormation logical IDs).
  - **Unit testing**: `@aws-cdk/assertions` allows writing unit tests: "assert that the RDS instance has DeletionPolicy: Snapshot" — enforced in CI/CD before deployment.
  - **CDK synthesizes CloudFormation**: CDK generates CloudFormation templates (`cdk synth`). You still get CloudFormation's change set and rollback capabilities — CDK is an authoring tool, not a deployment engine replacement.

### B. DeletionPolicy: Delete + Terraform — ❌ Incorrect

- **DeletionPolicy: Delete is the default** — it's what caused the incident. Relying on AWS Backup for recovery after deletion adds latency (restore takes minutes-hours), requires backup to be configured in advance, and doesn't protect S3 objects (AWS Backup for S3 is limited to restoring point-in-time versions, not recreating a deleted bucket).
- **`--no-execute-changeset`**: This is the same as creating a change set and not executing it — functionally correct but doesn't address the data loss issue. Also, this is a CLI flag, not a structural fix — developers can forget to use it.
- **Terraform instead of CloudFormation**: Terraform is a valid IaC tool with type checking (HCL type system), but migrating 60 existing CloudFormation stacks to Terraform is a massive effort. CDK generates CloudFormation templates — no migration needed. The question asks to modernize the workflow, not re-platform.

### C. Retain on ALL resources + drift detection — ❌ Incorrect

- **DeletionPolicy: Retain on ALL resources**: Over-broad. If every resource has `Retain`, deleting a stack leaves ALL resources orphaned (running and costing money). You can never cleanly delete a test stack — manual cleanup of every resource is required. `Retain` should be selectively applied to stateful resources (databases, S3 buckets, encryption keys), not stateless ones (Lambda functions, IAM roles, security groups).
- **Drift detection for previewing updates**: Drift detection shows differences between the ACTUAL resource and the TEMPLATE — changes made outside CloudFormation. It does NOT preview what will happen during an update. Change sets are for previewing updates; drift detection is for detecting out-of-band changes. Different tools for different problems.
- **SAM for all infrastructure**: SAM (Serverless Application Model) is designed for serverless workloads (Lambda, API Gateway, DynamoDB, Step Functions). It doesn't support many non-serverless resources (VPCs, EC2, RDS, etc.) — or where it does, it falls through to raw CloudFormation. SAM is not a general-purpose replacement for CloudFormation YAML authoring.

### D. Snapshot on all resources + StackSets for validation — ❌ Incorrect

- **DeletionPolicy: Snapshot on ALL resources**: `Snapshot` is only supported for specific resource types (RDS, ElastiCache, Neptune, Redshift, EBS). Setting `Snapshot` on unsupported resources (Lambda, S3, IAM, VPC, etc.) causes a stack creation error. This option would fail at deployment time.
- **StackSets for change validation**: StackSets deploy stacks across multiple accounts and Regions — they don't validate individual stack changes. StackSets use the same update mechanism as regular stacks. Change sets are the validation tool.
- **AWS Proton for type safety**: Proton manages deployment templates for platform teams, but it doesn't provide type-safe authoring (auto-completion, compile-time checks) like CDK. Proton templates are written in CloudFormation YAML/JSON and Jinja — the same untypes template language.

## Recommendations

- **Production resource checklist**: Every stateful resource (databases, S3 buckets, EFS, encryption keys) should have `DeletionPolicy: Snapshot` or `DeletionPolicy: Retain`. Add `UpdateReplacePolicy: Snapshot` on databases.
- **Always use change sets** — never call `update-stack` directly on production stacks. Integrate change set review into your CI/CD pipeline as a manual approval gate.
- **Enable termination protection** on every production stack immediately — it's a single API call with zero impact on operations.
- **CDK adoption path**: Start with new stacks in CDK. Import existing CloudFormation templates into CDK using `cdk import` or `cloudformation-include` (L1 constructs). Gradually refactor to L2 constructs.
- **CDK best practices**: Use `cdk diff` in CI/CD (fail if unexpected replacements), write assertions for DeletionPolicy, and use `cdk.RemovalPolicy.RETAIN` (CDK's equivalent of DeletionPolicy).
- **DeletionPolicy vs. RemovalPolicy**: In CloudFormation YAML, use `DeletionPolicy`. In CDK, use `.applyRemovalPolicy(cdk.RemovalPolicy.RETAIN)` — CDK translates this to the correct CloudFormation DeletionPolicy.

## Relevant Links

- [CloudFormation DeletionPolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html)
- [CloudFormation UpdateReplacePolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatereplacepolicy.html)
- [CloudFormation Change Sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)
- [Stack Termination Protection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-protect-stacks.html)
- [AWS CDK Getting Started](https://docs.aws.amazon.com/cdk/v2/guide/getting_started.html)
- [CDK Diff and Deploy](https://docs.aws.amazon.com/cdk/v2/guide/cli.html)
