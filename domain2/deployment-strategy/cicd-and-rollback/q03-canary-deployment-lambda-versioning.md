# Q03: API Gateway Canary Deployments and Lambda Versioning

## Question

A company runs a serverless API with API Gateway (REST API) → Lambda → DynamoDB. The API serves 10 million requests/day. Recent deployments have introduced regressions that weren't caught in pre-production testing. The team needs:

- **Canary releases**: Route 5% of production traffic to the new version for 30 minutes before full rollout
- **Instant rollback**: If error rate exceeds 1% during canary, automatically revert within 60 seconds
- **Zero-downtime deployments**: No request failures during version transitions
- **Version history**: Ability to roll back to any of the last 10 deployments

Which deployment strategy achieves all requirements?

## Options

- **A.** Use API Gateway stage canary settings to route 5% of traffic to a canary deployment. Lambda functions use versioned aliases — the `prod` alias points to the current version, and the canary deployment updates the alias to point to the new version. After 30-minute validation, promote the canary to 100%. Use CloudWatch Alarms on Lambda error rate + API Gateway 5xx count to trigger a CodeDeploy automatic rollback that shifts the Lambda alias back to the previous version.
- **B.** Deploy two separate API Gateway stages (prod and canary). Use Route 53 weighted routing to send 5% to the canary stage. Monitor with CloudWatch. Manual rollback by updating Route 53 weights to 0% canary.
- **C.** Use Lambda function URLs with CloudFront. Create a CloudFront distribution with two origins (current and canary Lambda URLs) and use origin groups with 95/5 weight. Monitor CloudFront error rates. Rollback by removing the canary origin.
- **D.** Use API Gateway stages with stage variables pointing to Lambda aliases. Deploy new version by updating the stage variable to point to a new Lambda alias. Use Lambda provisioned concurrency to pre-warm the new version. Rollback by changing the stage variable back.

## Answers

### A. API Gateway canary settings + Lambda versioned aliases + CodeDeploy auto-rollback — ✅ Correct

This uses the native API Gateway canary deployment feature combined with Lambda's version/alias system:

- **API Gateway stage canary settings**:
  - REST APIs support native canary deployments at the stage level. When you create a canary on a stage, API Gateway splits traffic between the current stage deployment and the canary deployment at the specified percentage (5%).
  - Canary traffic uses the same stage URL — clients don't need to change endpoints. API Gateway handles the traffic split internally using weighted random routing.
  - After validation, "promote canary" makes the canary deployment the new stage deployment (100% traffic). "Delete canary" reverts to the original deployment.

- **Lambda versioned aliases**:
  - Published Lambda versions are immutable snapshots of function code + configuration. The `prod` alias is a pointer to a specific version (e.g., version 15).
  - During canary: the alias is updated to use weighted alias routing — 95% → version 15 (current), 5% → version 16 (new). This provides Lambda-level traffic splitting independent of API Gateway.
  - **Version history**: Published versions are retained. The last 10 versions can be rolled back to by simply updating the alias pointer. Each version is immutable and independently invocable.

- **CodeDeploy for automated alias shifting**:
  - AWS CodeDeploy integrates with Lambda to manage alias traffic shifting. Deployment configurations include:
    - `CodeDeployDefault.LambdaCanary10Percent5Minutes` — 10% canary for 5 minutes
    - Custom configurations — 5% for 30 minutes (matching the requirement)
  - **Alarm-based rollback**: CodeDeploy monitors specified CloudWatch alarms during the deployment. If the Lambda error rate or API Gateway 5xx alarm triggers, CodeDeploy automatically shifts 100% traffic back to the previous version — within seconds, not minutes.
  - **Zero-downtime**: Traffic shifting is gradual — no requests are dropped. Both versions run simultaneously during the canary window.

### B. Two stages + Route 53 weighted routing — ❌ Incorrect

- **Two API Gateway stages**: Each stage has a separate URL (e.g., `prod.api.example.com`, `canary.api.example.com`). Route 53 weighted routing can split DNS between them, but DNS-based traffic splitting is imprecise — TTL caching means some clients keep resolving to the old IP for minutes after a change. DNS-based canary has 5-15 minute convergence, not the precise 5% split needed.
- **Manual rollback**: Updating Route 53 weights requires human intervention, then DNS propagation delay. The requirement is automatic rollback within 60 seconds — DNS TTL makes this impossible even with manual intervention.
- **Two stages also mean doubling API Gateway configuration** — stage-specific settings, throttling limits, and logging must be maintained in both places.

### C. Lambda function URLs + CloudFront origin groups — ❌ Incorrect

- **CloudFront origin groups** are for failover, not weighted traffic splitting. An origin group has a primary and secondary origin — CloudFront routes 100% to primary and fails over to secondary only on error (4xx/5xx). There's no 95/5 weighted distribution between origins.
- **Lambda function URLs** bypass API Gateway entirely — losing API Gateway features (request validation, throttling, API keys, usage plans, WAF integration, caching).
- **Removing canary origin for rollback**: CloudFront distribution updates take 5-15 minutes to propagate globally. This far exceeds the 60-second rollback requirement.

### D. Stage variables + Lambda aliases + provisioned concurrency — ❌ Incorrect

- **Stage variables pointing to aliases**: This works for directing a stage to a specific Lambda alias, but it's an all-or-nothing switch. Updating the stage variable instantly routes 100% of traffic to the new version — no canary split. This violates the 5% gradual rollout requirement.
- **Provisioned concurrency**: Pre-warms Lambda execution environments (good practice) but doesn't address traffic splitting, canary validation, or automated rollback. It's complementary, not a deployment strategy.
- **Manual rollback by changing stage variable**: Requires human intervention and an API call. No automated alarm-based rollback. Also, stage variable updates trigger a new deployment (takes seconds), during which some requests may see temporary inconsistency.

## Recommendations

- **API Gateway canary deployments** are the native mechanism for gradual rollouts — use them instead of DNS-based or external traffic splitting.
- **Lambda alias traffic shifting with CodeDeploy** provides automated, alarm-gated deployments with instant rollback — this is the AWS-recommended pattern for serverless CI/CD.
- **Always publish Lambda versions** — never deploy directly to `$LATEST` in production. Aliases should point to published versions for rollback capability.
- **Retain at least 10 published versions** for rollback history. Use a lifecycle policy (custom Lambda function via CloudFormation custom resource) to clean up old versions beyond your retention window.
- **Combine API Gateway canary + Lambda alias shifting** for defense in depth — API Gateway splits traffic at the API level, Lambda aliases split at the function level.

## Relevant Links

- [API Gateway Canary Deployments](https://docs.aws.amazon.com/apigateway/latest/developerguide/canary-release.html)
- [Lambda Versions and Aliases](https://docs.aws.amazon.com/lambda/latest/dg/configuration-aliases.html)
- [Lambda Traffic Shifting with CodeDeploy](https://docs.aws.amazon.com/lambda/latest/dg/lambda-rolling-deployments.html)
- [CodeDeploy Lambda Deployments](https://docs.aws.amazon.com/codedeploy/latest/userguide/applications-create-lambda.html)
