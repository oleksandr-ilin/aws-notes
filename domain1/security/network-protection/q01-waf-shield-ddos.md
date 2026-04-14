# Q01: WAF, Shield, and DDoS Protection Strategy

## Question

A company runs a public-facing e-commerce application behind an Application Load Balancer with Amazon CloudFront. During peak sales events, the application has experienced:
- HTTP flood attacks (thousands of requests/second from bot networks)
- SQL injection attempts targeting the search functionality
- Volumetric DDoS attacks (UDP reflection, SYN floods)

The company needs a comprehensive protection strategy that: automatically mitigates infrastructure-layer DDoS attacks, provides application-layer DDoS protection with 24/7 access to the AWS DDoS Response Team (DRT), blocks common web exploits (SQLi, XSS), and provides cost protection against DDoS-related scaling charges.

Which combination should the solutions architect recommend? **(Select TWO.)**

## Options

- **A.** Subscribe to AWS Shield Advanced on the CloudFront distribution and ALB. Enable AWS Shield Advanced's DDoS cost protection. Configure proactive engagement with the DDoS Response Team.
- **B.** Deploy AWS Web Application Firewall (WAF) on CloudFront and the ALB. Configure managed rule groups for SQL injection and XSS protection. Create rate-based rules to limit request rates from individual IP addresses.
- **C.** Enable AWS Shield Standard (default, free) and rely on it for all DDoS protection tiers. Deploy a network firewall for application-layer protection.
- **D.** Deploy Amazon GuardDuty to detect DDoS attacks and automatically block malicious IPs using security groups.
- **E.** Use Amazon CloudFront's built-in caching as the sole DDoS mitigation strategy.

## Answers

### A. AWS Shield Advanced — ✅ Correct

Shield Advanced provides comprehensive DDoS protection:
- **Infrastructure-layer protection**: Automatic mitigation of volumetric and state-exhaustion attacks (Layer 3/4) — same as Shield Standard but with enhanced detection.
- **Application-layer protection**: Detects application-layer DDoS patterns when combined with WAF. Shield Advanced can automatically create WAF rules to mitigate detected attacks.
- **DDoS Response Team (DRT) access**: 24/7 access to AWS DDoS experts who can assist during active attacks and proactively place mitigations.
- **Cost protection**: Refunds for scaling charges (EC2, ELB, CloudFront, Route 53) incurred due to DDoS attacks.
- **Proactive engagement**: DRT contacts you during detected events, not just when you escalate.

### B. AWS WAF with managed rules + rate-based rules — ✅ Correct

AWS WAF provides application-layer (Layer 7) protection:
- **Managed rule groups**: Pre-built rules from AWS and Marketplace partners that detect SQLi, XSS, and other OWASP Top 10 attacks.
- **Rate-based rules**: Automatically block IP addresses that exceed a configured request rate (e.g., >2,000 requests per 5 minutes). Essential for HTTP flood attacks.
- **Custom rules**: Match specific patterns (headers, query strings, geographic origin) to block suspicious traffic.
- **Deployed on CloudFront and ALB**: WAF evaluates requests before they reach the application.

### C. Shield Standard only — ❌ Incorrect

Shield Standard is automatically enabled for all AWS accounts at no cost and protects against common infrastructure-layer DDoS attacks. However, it does **not** provide: application-layer DDoS protection, DRT access, cost protection, or visibility into attack metrics. For an e-commerce application with active DDoS threats, Standard is insufficient.

### D. GuardDuty for DDoS — ❌ Incorrect

GuardDuty detects threats by analyzing flow logs, DNS queries, and CloudTrail events, but it is not a DDoS mitigation service. It cannot block traffic in real-time or create WAF rules. Its findings are detected after the fact and require separate automation (Lambda, Step Functions) for response. Shield + WAF provide purpose-built DDoS mitigation.

### E. CloudFront caching only — ❌ Incorrect

CloudFront caching absorbs some traffic by serving cached responses, but it doesn't protect against: cache-busting attacks (unique URLs to bypass cache), POST floods, or application-layer attacks targeting dynamic endpoints. Caching is a complementary measure, not a DDoS mitigation strategy.

## Recommendations

- **Shield Standard** (free) provides baseline Layer 3/4 protection for all AWS customers.
- **Shield Advanced** ($3,000/month + data transfer) adds: Layer 7 detection, DRT access, cost protection, and enhanced visibility.
- **WAF** provides request-level inspection and filtering. Essential for application-layer protection regardless of Shield tier.
- **Shield Advanced + WAF integration**: Shield Advanced can automatically create WAF rules to mitigate detected application-layer attacks.
- Deploy WAF on **CloudFront** (edge-level filtering) AND **ALB** (defense in depth, in case traffic bypasses CloudFront).
- Use WAF **Bot Control** managed rule group for sophisticated bot detection beyond simple rate limiting.

## Relevant Links

- [AWS Shield](https://docs.aws.amazon.com/waf/latest/developerguide/shield-chapter.html)
- [AWS Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced.html)
- [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html)
- [WAF Managed Rule Groups](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups.html)
- [DDoS Best Practices](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/welcome.html)
