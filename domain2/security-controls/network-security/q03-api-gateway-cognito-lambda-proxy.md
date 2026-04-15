# Q03: API Gateway with Cognito Authorization, Lambda Proxy, and DynamoDB Session Store

## Question

A company is building a customer-facing REST API for a mobile banking application. The API serves 500,000 active users with peak traffic of 15,000 requests/second. The architecture requirements are:

1. **Authentication**: Users authenticate via a mobile app using username/password and MFA. The API must validate JWT tokens on every request without calling a custom authorizer Lambda for each invocation
2. **Authorization**: Different API endpoints require different permissions — `/accounts` requires `read:accounts` scope, `/transfers` requires `write:transfers` scope. Scope enforcement must happen at the API layer, not inside the backend Lambda
3. **Backend integration**: Lambda functions process API requests. The Lambda must receive the full HTTP request context (headers, path parameters, query strings, body) without manual mapping templates
4. **Session metadata**: Each authenticated user session must store metadata (device ID, IP geolocation, last activity timestamp) in a low-latency datastore for fraud detection. Read latency must be < 5 ms at 15,000 TPS
5. **Auto-scaling**: The API must handle traffic spikes (12x during salary deposits at month-end) without pre-provisioning or capacity management

Which architecture meets all requirements?

## Options

- **A.** API Gateway REST API with a Cognito User Pool authorizer. Cognito User Pool handles user registration, authentication, and MFA. The authorizer validates JWT tokens (ID token or access token) at the API Gateway layer — no Lambda invocation for auth. Configure OAuth 2.0 scopes on each API method: `/accounts GET` requires `read:accounts`, `/transfers POST` requires `write:transfers`. Use Lambda **proxy integration** — API Gateway passes the entire HTTP request as a single event object to Lambda (headers, body, path/query parameters, and Cognito claims). Lambda writes session metadata to DynamoDB with DAX (DynamoDB Accelerator) for < 5 ms reads at 15,000 TPS. API Gateway auto-scales automatically with no capacity configuration.
- **B.** API Gateway REST API with a Lambda authorizer (token-based). The Lambda authorizer validates JWTs by calling Cognito's `/oauth2/userInfo` endpoint on every request. Lambda non-proxy integration with mapping templates for request/response transformation. Session metadata stored in ElastiCache (Redis). API Gateway with usage plans for throttling.
- **C.** Application Load Balancer with OIDC authentication action. ALB validates tokens via Cognito. Lambda targets behind the ALB. Session metadata in Aurora Serverless. ALB auto-scaling via target tracking.
- **D.** API Gateway HTTP API with IAM authorization. Users sign requests with SigV4 using temporary credentials from Cognito Identity Pool. Lambda proxy integration. Session metadata in DynamoDB. API Gateway auto-scaling.

## Answers

### A. Cognito User Pool authorizer + Lambda proxy + DynamoDB/DAX — ✅ Correct

This architecture addresses every requirement with purpose-built services:

**Authentication — Cognito User Pool Authorizer:**

- **Cognito User Pool**: A fully managed user directory. Users sign up with email/phone, authenticate with username/password, and configure MFA (SMS, TOTP app). Cognito handles:
  - User registration and email/phone verification
  - Secure password policies (minimum length, complexity)
  - MFA enforcement (required, optional, or off)
  - Token issuance: ID token (user identity), access token (scopes/permissions), refresh token

- **Cognito User Pool authorizer on API Gateway**:
  - The authorizer is configured at the API Gateway level — not a Lambda function.
  - On each request, the client sends the JWT (ID or access token) in the `Authorization` header.
  - API Gateway validates the token locally: checks signature (using Cognito's JWKS public keys), issuer (`iss`), expiration (`exp`), audience (`aud`), and token use (`token_use`).
  - **No Lambda invocation for auth**: Token validation is performed BY API Gateway — not by a Lambda authorizer. This eliminates: Lambda cold starts, execution costs, and a potential single point of failure.
  - **Validated claims are passed downstream**: API Gateway extracts user claims (sub, email, custom attributes) from the token and passes them to the backend Lambda via `event.requestContext.authorizer.claims`.

- **Why not a Lambda authorizer?**: Lambda authorizers are custom code —you write a function that validates the token and returns an IAM policy. For standard JWT validation, this is unnecessary complexity. Cognito User Pool authorizers do the same validation natively with zero custom code. Use Lambda authorizers only when: (a) you need custom validation logic beyond JWT (e.g., check a deny-list, verify device fingerprint), or (b) your identity provider is not Cognito (e.g., Okta, Auth0 — though API Gateway HTTP APIs support generic JWT authorizers).

**Authorization — OAuth 2.0 Scopes:**

- **Cognito resource server + scopes**: Define a resource server in Cognito (e.g., `banking-api`) with custom scopes: `read:accounts`, `write:transfers`, `read:statements`.
- **Per-method scope enforcement**: On each API Gateway method, specify required scopes:
  - `GET /accounts` → requires scope `banking-api/read:accounts`
  - `POST /transfers` → requires scope `banking-api/write:transfers`
  - API Gateway checks the access token's `scope` claim against the required scopes. If the token lacks the required scope, API Gateway returns 403 Forbidden — the Lambda function is never invoked.
- **Scope assignment**: Cognito app clients are configured with allowed scopes. The mobile app requests specific scopes during the OAuth 2.0 authorization flow. Cognito issues an access token containing only the granted scopes.
- **Authorization at the API layer**: Scope enforcement happens at API Gateway, not inside Lambda. This is defense in depth — even if Lambda code has a bug that skips authorization checks, API Gateway blocks unauthorized requests.

**Backend — Lambda Proxy Integration:**

- **Proxy integration**: API Gateway sends the ENTIRE HTTP request as a single JSON event to Lambda:
  ```json
  {
    "httpMethod": "POST",
    "path": "/transfers",
    "headers": {"Authorization": "Bearer eyJ...", "Content-Type": "application/json"},
    "queryStringParameters": {"currency": "USD"},
    "pathParameters": {"accountId": "12345"},
    "body": "{\"amount\": 500, \"to\": \"67890\"}",
    "requestContext": {
      "authorizer": {
        "claims": {"sub": "user-uuid", "email": "user@example.com", "cognito:groups": "premium"}
      }
    }
  }
  ```
- **No mapping templates**: With proxy integration, you don't write VTL (Velocity Template Language) mapping templates to transform the request. Lambda receives the raw HTTP context and returns a response object with `statusCode`, `headers`, and `body`.
- **Lambda response format**: Lambda must return:
  ```json
  {
    "statusCode": 200,
    "headers": {"Content-Type": "application/json"},
    "body": "{\"transferId\": \"txn-789\"}"
  }
  ```
- **Non-proxy (custom) integration** would require defining mapping templates in VTL for request transformation and response mapping — additional complexity with no benefit when Lambda handles the raw HTTP event directly.

**Session Metadata — DynamoDB + DAX:**

- **DynamoDB**: The session metadata table stores:
  - Partition key: `userId` (Cognito `sub` UUID)
  - Attributes: `deviceId`, `ipGeo`, `lastActivity`, `sessionStart`, `mfaVerifiedAt`
  - TTL: expire sessions after 24 hours of inactivity
- **DynamoDB scaling**: DynamoDB on-demand mode handles 15,000 WCU and 15,000 RCU without capacity management. Scales to 12x (180,000 TPS) during month-end spikes without pre-provisioning.
- **DAX (DynamoDB Accelerator)**: In-memory cache for DynamoDB providing:
  - **< 1 ms read latency** for cache hits (vs. single-digit ms for DynamoDB direct reads)
  - Write-through cache — writes go to DynamoDB AND update the cache
  - The fraud detection service reads session metadata from DAX for < 5 ms response at high TPS
- **Why not ElastiCache Redis?**: Redis is a powerful general-purpose cache, but using it for DynamoDB caching requires: custom cache invalidation logic, separate provisioning/scaling, and VPC configuration. DAX is a purpose-built DynamoDB cache — same API, automatic cache management, no application-level cache invalidation code.

**Auto-scaling — API Gateway:**

- **API Gateway auto-scales automatically**: No capacity configuration, no scaling policies, no target tracking. API Gateway handles up to 10,000 requests/second per account per Region (soft limit, increaseable to 100,000+).
- **Lambda concurrency**: With 15,000 TPS, Lambda runs up to 15,000 concurrent executions (if each takes ~1 second). Default account concurrency limit is 1,000 — request an increase to 15,000+. Use provisioned concurrency for consistent cold start performance.
- **Month-end spikes**: Both API Gateway and DynamoDB on-demand scale automatically. Lambda concurrent execution limit should be set above peak (e.g., 20,000). No pre-provisioning or manual intervention.

### B. Lambda authorizer + non-proxy + ElastiCache — ❌ Incorrect

- **Lambda authorizer calling `/oauth2/userInfo`**: A Lambda authorizer that calls Cognito's UserInfo endpoint on EVERY request adds: Lambda invocation latency (100-500 ms cold start), an HTTP call to Cognito (50-100 ms), and Lambda execution cost. At 15,000 TPS, this is 15,000 Lambda invocations per second JUST for authentication — before the backend Lambda runs. The Cognito User Pool authorizer does the same thing natively with zero Lambda overhead.
  - API Gateway caches Lambda authorizer results (up to 3600 seconds), which reduces invocations but introduces stale authorization — a revoked token remains valid until the cache expires.

- **Non-proxy integration with mapping templates**: VTL mapping templates are complex, hard to debug, and require per-method configuration. Proxy integration passes everything to Lambda in a standard format — simpler, more maintainable, and sufficient for most use cases. Non-proxy integration is justified only when API Gateway must transform the request before Lambda receives it (e.g., XML-to-JSON conversion for legacy backends).

- **ElastiCache Redis for session metadata**: Redis works but requires:
  - Separate VPC configuration, subnet groups, security groups
  - Cluster sizing and scaling management (nodes, shards, replicas)
  - Custom serialization/deserialization code
  - No automatic scaling — you must add/remove nodes manually or use ElastiCache Auto Scaling (limited to replica auto scaling)
  - DynamoDB + DAX is simpler for key-value session lookups with automatic scaling.

### C. ALB with OIDC action — ❌ Incorrect

- **ALB OIDC authentication**: ALB supports OIDC-compliant authentication as a listener rule action. It can validate tokens via Cognito. However:
  - ALB authentication is designed for web applications (browser-based flows with redirects). For a mobile API, the ALB initiates a browser redirect to Cognito's hosted UI — this doesn't work for API clients sending JSON requests.
  - ALB doesn't support per-path scope enforcement. You can't require different OAuth scopes for `/accounts` vs. `/transfers` at the ALB level. API Gateway's method-level scope configuration is purpose-built for this.
  - ALB doesn't auto-scale in the same way as API Gateway — you must register targets (Lambda functions or EC2/ECS) and manage target groups. For Lambda targets, ALB invokes Lambda synchronously — similar to API Gateway but without API management features (throttling, API keys, usage plans, caching, request validation).

- **Aurora Serverless for session metadata**: Aurora Serverless v2 can handle the throughput, but:
  - Read latency: 5-10 ms (comparable to DynamoDB but higher than DAX < 1 ms)
  - Connection management: At 15,000 TPS, you need RDS Proxy to manage connection pooling (Lambda creates new connections per invocation). Adds complexity and cost.
  - DynamoDB is simpler for key-value session lookups — no connection management, no SQL schema, no RDS Proxy.

### D. HTTP API with IAM + SigV4 — ❌ Incorrect

- **IAM authorization with SigV4**: This requires the mobile app to sign every request with AWS SigV4 using temporary credentials from Cognito Identity Pool. The flow:
  1. User authenticates with Cognito User Pool → gets JWT tokens
  2. Exchange JWT for temporary AWS credentials via Cognito Identity Pool (STS)
  3. Sign every API request with SigV4 using the temporary credentials
  - This is complex for mobile developers — SigV4 signing requires AWS SDK integration in the mobile app. JWT-based auth (send token in Authorization header) is far simpler.
  - Scope-based authorization is not available with IAM auth — IAM policies use resource-level permissions, not OAuth scopes. Fine-grained per-endpoint authorization requires custom IAM policies per user group — much harder to manage than Cognito scopes.

- **API Gateway HTTP APIs**: HTTP APIs are simpler and cheaper than REST APIs but have fewer features — no usage plans, no request caching, no WAF integration. For a banking application handling 500K users with security requirements, REST API (with WAF, request validation, and usage plans) is more appropriate.

## Recommendations

- **Cognito User Pool authorizer** is the default choice for APIs where Cognito handles authentication. Use Lambda authorizers only for non-Cognito identity providers or custom validation logic.
- **Lambda proxy integration** is the default choice for most API Gateway + Lambda architectures. Use non-proxy integration only when API Gateway must transform the request format.
- **REST API vs HTTP API**: Use REST API for production APIs needing: WAF integration, request/response validation, caching, usage plans, API keys. Use HTTP API for simple, cost-sensitive APIs (70% cheaper than REST API).
- **DynamoDB on-demand + DAX** for session stores — no capacity management, sub-millisecond reads.
- **Monitor API Gateway**: Enable CloudWatch detailed metrics, access logging, and X-Ray tracing. Set up CloudWatch alarms for 4xx/5xx error rates and latency P99.
- **API Gateway throttling**: Configure account-level and per-method throttling to protect backend services from traffic surges. This is especially important for banking APIs — prevent abuse without affecting legitimate users.

## Relevant Links

- [API Gateway Cognito User Pool Authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-integrate-with-cognito.html)
- [API Gateway Lambda Proxy Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html)
- [Cognito Resource Servers and Scopes](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html)
- [DynamoDB DAX](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DAX.html)
- [API Gateway REST vs HTTP API](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)
- [Lambda Authorizer vs Cognito Authorizer](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-to-api.html)
