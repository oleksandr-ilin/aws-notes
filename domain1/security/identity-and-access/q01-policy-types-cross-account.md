# Q01: IAM Policies — Identity-Based vs. Resource-Based vs. SCPs

## Question

A company has a cross-account architecture where developers in Account A (Development) need to access an S3 bucket in Account B (Data Lake). The security team wants to ensure that:
- Only IAM roles in Account A with the tag `Project: DataAnalytics` can access the bucket
- Access is limited to the `s3:GetObject` action on a specific prefix (`/analytics/*`)
- The Data Lake account owner controls who can access their bucket independently of Account A's IAM policies
- No other accounts in the Organization should be able to access the bucket

Which combination of policies should the solutions architect configure? **(Select TWO.)**

## Options

- **A.** In Account A, create an IAM role with an identity-based policy that grants `s3:GetObject` on `arn:aws:s3:::data-lake-bucket/analytics/*`. Add a condition `aws:PrincipalTag/Project: DataAnalytics`.
- **B.** In Account B, configure an S3 bucket policy (resource-based policy) that allows `s3:GetObject` from Account A's specific IAM role ARN on the `/analytics/*` prefix. Add a condition requiring `aws:PrincipalTag/Project: DataAnalytics`.
- **C.** In the Organization management account, create an SCP that grants `s3:GetObject` on the Data Lake bucket to Account A.
- **D.** In Account B, create an IAM role that Account A developers can assume. Attach a policy granting `s3:GetObject` on the analytics prefix.
- **E.** Use AWS Resource Access Manager to share the S3 bucket from Account B to Account A.

## Answers

### A. Identity-based policy in Account A — ✅ Correct

The IAM role in Account A needs explicit permissions to perform `s3:GetObject`. Identity-based policies define what actions the principal (IAM role) is allowed to take. The condition `aws:PrincipalTag/Project: DataAnalytics` ensures only roles with the correct tag can invoke these permissions. However, for cross-account access, the identity-based policy alone isn't sufficient — the resource owner (Account B) must also grant access.

### B. Resource-based policy (bucket policy) in Account B — ✅ Correct

For **cross-account S3 access**, the resource owner must explicitly allow the access via a **bucket policy**. This gives Account B independent control over who can access their data. The bucket policy specifies:
- **Principal**: Account A's role ARN (not the entire account)
- **Action**: `s3:GetObject` only
- **Resource**: `arn:aws:s3:::data-lake-bucket/analytics/*`
- **Condition**: `aws:PrincipalTag/Project: DataAnalytics`

Both policies (A and B) must allow the action for cross-account access to work. The effective permissions are the **union** of the identity-based policy and the resource-based policy for cross-account scenarios (unlike same-account, where it's the intersection).

### C. SCP to grant access — ❌ Incorrect

SCPs **cannot grant permissions** — they only restrict (set maximum available permissions). An SCP can deny access to the bucket from other accounts, but it cannot grant cross-account S3 access. SCPs define the permission ceiling, not the floor.

### D. Cross-account IAM role assumption — ❌ Incorrect

While cross-account role assumption is a valid pattern, it requires Account A developers to **assume a role in Account B** and then access the bucket as a principal in Account B. This changes the trust model — the developer operates as an Account B entity, not an Account A entity. It doesn't meet the requirement that access control be tied to Account A's `Project: DataAnalytics` tag. Resource-based policies (bucket policy) are simpler for S3 cross-account access.

### E. RAM for S3 sharing — ❌ Incorrect

AWS RAM does not support sharing S3 buckets. RAM supports subnets, Transit Gateways, Resolver rules, License Manager configs, and other specific resource types. S3 cross-account access is controlled through bucket policies.

## Recommendations

- **Cross-account S3 access**: Use the bucket policy (resource-based) + IAM role policy (identity-based) combination. Both must allow the action.
- **Same-account access**: Effective permissions = intersection of all applicable policies (identity-based, resource-based, permissions boundary, SCP).
- **Cross-account access**: If the resource-based policy specifies the cross-account principal, the resource-based policy alone is sufficient in some cases — but best practice is to have both policies aligned.
- **SCPs**: Can only restrict — use `Deny` statements in SCPs to prevent other accounts from accessing the bucket.
- Use **S3 Access Points** for managing complex multi-account access patterns — each access point has its own access policy.

## Relevant Links

- [IAM Policy Types](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [Cross-Account S3 Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-walkthroughs-managing-access-example2.html)
- [S3 Bucket Policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html)
- [Policy Evaluation Logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)
