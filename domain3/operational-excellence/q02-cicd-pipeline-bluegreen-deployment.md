# Q02: Improving CI/CD Pipeline and Deployment Strategies

## Question

A company deploys a customer-facing API (ECS Fargate behind ALB) with the following current deployment process:

**Current state (problems identified in post-mortem):**
1. **All-at-once deployments**: The team uses `ecs update-service --force-new-deployment` which replaces all tasks simultaneously. During the 3-minute deployment window, some requests hit old tasks and some hit new tasks — no consistent version. Last month, a bad deployment caused 100% of requests to fail for 2 minutes before anyone noticed.
2. **No automated testing in pipeline**: Code goes from GitHub → CodeBuild (build + push image to ECR) → manual `ecs update-service`. No integration tests, no canary checks, no automated rollback.
3. **Environment drift**: The staging environment diverged from production — different task definitions, different environment variables, different IAM roles. A deployment that worked in staging failed in production because production had a stricter security group.
4. **Rollback takes 15 minutes**: After detecting the bad deployment, rolling back required: finding the previous task definition revision, manually updating the service, waiting for new tasks to start. No one-click rollback.

The team wants: (a) zero-downtime deployments with automatic rollback on errors, (b) production validation before full traffic shift, (c) infrastructure-as-code to prevent environment drift, and (d) rollback in under 2 minutes.

Which improvement strategy addresses all requirements?

## Options

- **A.** Implement **CodePipeline** with stages: Source (GitHub) → Build (CodeBuild: build image, push to ECR, run unit tests) → Deploy to Staging (CodeDeploy ECS blue/green to staging cluster, run integration tests) → Approval Gate (manual or automated) → Deploy to Production (CodeDeploy ECS **blue/green deployment** with traffic shifting). CodeDeploy blue/green: launches a new task set (green) alongside the existing task set (blue). Shifts traffic using ALB listener rules: start with 10% to green → run **CodeDeploy lifecycle hooks** (Lambda functions that test the green deployment: health checks, smoke tests, error rate validation via CloudWatch). If tests pass, shift 100% to green. If any test fails OR CloudWatch alarm triggers (5xx rate > 1%), **automatic rollback** re-routes 100% traffic back to blue in < 1 minute. Use **CloudFormation** (or CDK) for infrastructure — task definitions, security groups, IAM roles, and environment variables defined in templates. Same template deploys to staging and production with parameter overrides. Prevents environment drift.
- **B.** Switch to rolling deployments with `minimumHealthyPercent = 50` and `maximumPercent = 200`. Add a CloudWatch alarm for 5xx errors. Manual rollback by reverting the task definition.
- **C.** Use AWS Proton for all deployments. Proton manages environment templates and service templates. CodePipeline triggers Proton deployments.
- **D.** Deploy with Kubernetes (EKS) using Argo Rollouts for canary deployments. Migrate from ECS Fargate to EKS for better deployment control.

## Answers

### A. CodePipeline + CodeDeploy ECS blue/green + CloudFormation — ✅ Correct

**Requirement (a) — Zero-downtime with automatic rollback:**

- **CodeDeploy ECS blue/green deployment**:
  - ECS blue/green deployments managed by CodeDeploy maintain TWO task sets simultaneously:
    - **Blue (original)**: Current production tasks, serving 100% of traffic
    - **Green (replacement)**: New version tasks, initially serving 0% traffic
  - The ALB has two target groups: one for blue, one for green. CodeDeploy updates the ALB listener rules to shift traffic between target groups.

- **Traffic shifting options**:
  - **AllAtOnce**: 100% to green immediately (no validation window — not recommended)
  - **Linear10PercentEvery1Minute**: Shift 10% at a time, every minute. Full shift in 10 minutes. Errors at any stage trigger rollback.
  - **Canary10Percent5Minutes**: Shift 10% to green, wait 5 minutes (validation window), then shift remaining 90%. Best balance of speed and safety.
  - For this scenario, `Canary10Percent5Minutes` is recommended — 10% of traffic validates the green deployment. If the 10% shows errors, only 10% of users were affected, and rollback is instant.

- **Automatic rollback triggers**:
  - CodeDeploy monitors a configured CloudWatch alarm (e.g., `HTTPCode_Target_5XX_Count > 10`).
  - If the alarm enters ALARM state during deployment, CodeDeploy automatically:
    1. Stops the traffic shift
    2. Routes 100% traffic back to blue (original) target group
    3. Terminates green tasks
    4. Marks the deployment as FAILED
  - **Rollback time**: < 1 minute (ALB listener rule update is near-instant). No new tasks need to launch — blue tasks are still running.

- **Zero-downtime guarantee**: Blue tasks continue serving traffic until green is fully validated. At no point are there zero healthy tasks. Even during rollback, blue tasks handle 100% of traffic immediately.

**Requirement (b) — Production validation before full traffic shift:**

- **CodeDeploy lifecycle hooks**: Lambda functions that run at specific stages of the deployment:
  - **BeforeAllowTraffic**: Runs BEFORE any traffic goes to green. Validates: green tasks are healthy, task count matches desired, containers started without errors, health check endpoints respond.
  - **AfterAllowTraffic**: Runs AFTER the initial traffic shift (e.g., after 10% shifts to green). Validates: error rate is within threshold, latency is normal, business metrics are correct (e.g., order success rate > 99%).
  - If any hook returns FAILURE, CodeDeploy triggers automatic rollback.

- **Test endpoint**: CodeDeploy can configure a separate ALB listener (port 8443) that always routes to the green target group. QA can test the green deployment on this port BEFORE any production traffic shifts. This is a "test before promote" pattern.

**Requirement (c) — Infrastructure-as-code preventing drift:**

- **CloudFormation templates**:
  - Define the entire stack in a CloudFormation template: ECS task definition (including image URI, environment variables, IAM task role, log configuration), ECS service (deployment configuration, load balancer), security groups, IAM roles, ALB target groups.
  - **Staging and production use the same template** with parameter overrides:
    ```yaml
    Parameters:
      Environment:
        Type: String
        AllowedValues: [staging, production]
      DesiredCount:
        Type: Number
        Default: 2
    ```
  - Staging: `Environment=staging, DesiredCount=2`. Production: `Environment=production, DesiredCount=20`.
  - **Drift detection**: CloudFormation drift detection identifies when someone manually changes a resource outside the template (e.g., modifying a security group in the console). Run drift detection weekly to catch unauthorized changes.

- **Preventing the staging/production divergence**: Because both environments are deployed from the same template, they CANNOT diverge on infrastructure configuration. If a security group rule exists in production, it exists in staging (same template). The environment divergence problem is eliminated.

**Requirement (d) — Rollback in under 2 minutes:**

- Blue/green rollback is a listener rule update — CodeDeploy modifies the ALB listener to route 100% back to the blue target group. This takes < 30 seconds. Blue tasks are already running and healthy — no startup delay.
- Compare to the current process: find previous task definition → update service → wait for new tasks to start → old tasks drain = 15+ minutes. Blue/green eliminates all of this because the rollback target (blue) is already running.

### B. Rolling deployment with 50% minimum — ❌ Incorrect

- **Rolling deployment**: ECS rolling updates replace tasks gradually — stop some old tasks, start new tasks, repeat. With `minimumHealthyPercent = 50`, only 50% of tasks are guaranteed healthy during deployment. A 50% capacity reduction may cause performance degradation under load.
- **No traffic isolation**: Unlike blue/green, rolling deployments serve traffic from BOTH old and new tasks simultaneously. There's no way to route "only 10% to new version" — tasks are randomly mixed in the same target group. You can't validate the new version in isolation.
- **Rollback is NOT instant**: To roll back a rolling deployment, you must deploy the previous version — another rolling update. This takes the same time as the original deployment (3-5 minutes minimum). During this period, some tasks run the bad version.
- **No lifecycle hooks**: Rolling ECS deployments don't support CodeDeploy lifecycle hooks. You can't run validation Lambda functions during the deployment. The only validation is the ALB health check (which checks "/health" responds with 200, not whether the application logic is correct).

### C. AWS Proton — ❌ Incorrect

- **AWS Proton**: Proton is a managed service for platform teams to create and manage standardized deployment templates for development teams. It's designed for organizations where a platform team defines infrastructure patterns and development teams consume them.
- Proton solves the "environment standardization" problem but adds significant complexity for a team that just needs CI/CD with blue/green deployments. Proton requires: defining environment templates, service templates, managing template versions, and Proton-specific workflows.
- For a single application team improving their deployment pipeline, CodePipeline + CodeDeploy + CloudFormation is the standard, well-documented approach. Proton is for platform teams managing dozens of development teams with standardized patterns.
- Proton still uses CodePipeline internally for deployments — it's an abstraction layer on top of the same tools. Adding Proton doesn't provide blue/green deployments or automatic rollback that CodeDeploy doesn't already provide.

### D. Migrate to EKS + Argo Rollouts — ❌ Incorrect

- **Re-platforming to EKS**: Migrating from ECS Fargate to EKS is a significant undertaking: Kubernetes cluster management, pod definitions, Helm charts, RBAC, service mesh, and ops team Kubernetes expertise. The team's problem is deployment process, not container orchestration.
- **Argo Rollouts**: Third-party Kubernetes controller for canary/blue-green deployments. Powerful but requires: EKS cluster, Argo installation, CRD definitions, and Kubernetes expertise. CodeDeploy provides native blue/green for ECS without Kubernetes.
- **Principle of least change**: The existing ECS Fargate architecture works. Adding CodeDeploy blue/green to ECS requires configuration changes, not re-architecture. Migrating to EKS introduces weeks of migration work, new operational skills, and new failure modes — disproportionate to the deployment improvement needed.

## Recommendations

- **Blue/green is the recommended deployment type for production ECS services**: It provides zero-downtime, instant rollback, and pre-deployment validation. Use rolling updates only for non-critical services where deployment speed matters more than safety.
- **CodeDeploy alarm configuration**: The CloudWatch alarm used for automatic rollback should measure user-visible impact (5xx errors, latency P99) — not infrastructure metrics (CPU). A new deployment might increase CPU (expected) while maintaining low error rates (acceptable).
- **Pipeline stages for production**: Source → Build (unit tests) → Deploy Staging (blue/green) → Integration Tests → Manual Approval → Deploy Production (blue/green with canary). The manual approval gate is optional but recommended for regulated environments.
- **CloudFormation vs. CDK**: Both achieve the same outcome (infrastructure-as-code). CDK generates CloudFormation templates from TypeScript/Python code — preferred for teams comfortable with programming. CloudFormation YAML/JSON is preferred for teams wanting declarative configuration.
- **Tag container images with git SHA**: Use the git commit SHA as the ECR image tag (e.g., `myapp:a1b2c3d`). This creates traceability from deployed image → git commit → code changes. Never use `:latest` in production — it makes rollback ambiguous.

## Relevant Links

- [CodeDeploy ECS Blue/Green](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-groups-create-blue-green-ecs.html)
- [CodeDeploy Traffic Shifting](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html#deployment-configuration-ecs)
- [CodeDeploy Automatic Rollback](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployments-rollback-and-redeploy.html)
- [CodePipeline Tutorial ECS](https://docs.aws.amazon.com/codepipeline/latest/userguide/ecs-cd-pipeline.html)
- [CloudFormation Drift Detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/detect-drift-stack.html)
