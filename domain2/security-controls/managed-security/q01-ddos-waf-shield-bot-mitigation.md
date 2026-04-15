# Q01: DDoS and Application-Layer Attack Mitigation at Scale

## Question

A global e-commerce company serves 50 million monthly users via a web application behind CloudFront and an ALB. The security team has received intelligence that the company may be targeted by:
- Volumetric DDoS attacks (UDP floods, DNS amplification)
- Application-layer attacks (HTTP floods, slowloris, credential stuffing via the login endpoint)
- Bot scraping of product pricing pages

Requirements:
- Protection must be always-on (not reactive)
- Legitimate traffic must not be impacted
- The company needs DDoS response team support during active attacks
- Attack mitigation costs (scaling from traffic spikes) must be covered
- Bot traffic must be detected and managed separately from DDoS

Which combination of services should the solutions architect implement?

## Options

- **A.** Enable AWS Shield Advanced on the CloudFront distribution, ALB, and Route 53 hosted zone. Deploy AWS WAF on CloudFront with rules: rate-based rule on the login endpoint (limit 100 req/5min per IP), AWS Managed Rules common rule set (SQLi, XSS), and Bot Control managed rule group for bot detection and management. Configure Shield Advanced DDoS Response Team (DRT) authorization. Enable Shield Advanced cost protection for auto-scaling resources.
- **B.** Use AWS Shield Standard (free, automatic) for DDoS. Deploy a third-party WAF appliance on EC2 behind the ALB for application-layer filtering. Use CloudFront geo-restriction to block traffic from countries where attacks originate.
- **C.** Deploy AWS Network Firewall in the VPC to inspect all inbound traffic. Use Network Firewall's IPS rules for HTTP flood detection. Enable GuardDuty for threat detection and configure it to automatically block attacking IPs via Lambda + WAF.
- **D.** Over-provision infrastructure (10x normal capacity) to absorb attacks. Use CloudFront with no WAF — rely on CloudFront's built-in protections. Set up CloudWatch alarms to alert the team when request volume exceeds thresholds.

## Answers

### A. Shield Advanced + WAF with managed rules + Bot Control — ✅ Correct

This is the comprehensive, layered defense:
- **Shield Advanced on CloudFront**: Protects against L3/L4 volumetric attacks (SYN floods, UDP reflection) at the edge — before traffic reaches the ALB. CloudFront's global edge network absorbs attack volume. Shield Advanced provides enhanced detection and automatic mitigation for L7 attacks on CloudFront/ALB.
- **Shield Advanced DRT**: The DDoS Response Team actively assists during attacks, writes custom WAF rules, and proactively monitors protected resources. Requires authorization to access WAF/Shield configs.
- **Cost protection**: Shield Advanced provides cost protection credits for scaling caused by DDoS attacks — if an attack causes CloudFront or ALB billing to spike, AWS credits the cost.
- **WAF rate-based rules**: Limits requests per source IP to the login endpoint — mitigates credential stuffing and HTTP floods without blocking legitimate users.
- **AWS Managed Rules**: Pre-built rule sets for common attacks (SQLi, XSS, known bad inputs) maintained by AWS. No rule authoring needed.
- **Bot Control**: Classifies requests as verified bots (Googlebot), self-identified bots, and unverified bots. Can CAPTCHA challenge, block, or rate-limit by bot category — handles scraping separately from DDoS.

### B. Shield Standard + third-party WAF + geo-restriction — ❌ Incorrect

- **Shield Standard** provides basic L3/L4 protection but no L7 mitigation, no DRT access, and no cost protection. For a company facing targeted attacks, this is insufficient.
- **Third-party WAF on EC2** behind the ALB means attack traffic has already traversed CloudFront and the ALB — the WAF only sees traffic after it's consumed resources. It also introduces a scaling bottleneck and operational overhead (patching, licensing, HA).
- **Geo-restriction** blocks entire countries — legitimate customers in those countries are impacted. Sophisticated attackers use distributed botnets across many countries, making geo-blocking ineffective.

### C. Network Firewall + GuardDuty — ❌ Incorrect

- **Network Firewall** operates at the VPC level — by the time traffic reaches the VPC, it has already passed through CloudFront and the ALB. DDoS mitigation must happen at the edge, not inside the VPC.
- Network Firewall's IPS rules are designed for network-level inspection (Suricata rules), not HTTP-layer flood detection. It doesn't understand HTTP semantics like login endpoint rate limiting.
- **GuardDuty** detects threats asynchronously (minutes to hours after events) — it's a detection service, not a real-time mitigation service. Lambda → WAF integration adds latency between detection and blocking.
- No bot management capability.

### D. Over-provisioning — ❌ Incorrect

Over-provisioning 10x is extremely expensive and still insufficient for large-scale DDoS (attackers can generate traffic exceeding any reasonable over-provision). CloudFront's built-in protections are equivalent to Shield Standard — no L7 mitigation, no DRT, no cost protection. CloudWatch alerting is reactive — the team is notified after users are impacted. No bot management or application-layer protection.

## Recommendations

- **Shield Advanced** should protect internet-facing resources: CloudFront, ALB, NLB, Elastic IP, Route 53. Subscription is per-organization (not per-resource) at $3,000/month.
- **WAF placement**: On CloudFront (blocks at edge before reaching origin) AND on ALB (defense in depth). CloudFront WAF rules are evaluated at edge locations closest to the user.
- **Rate-based rules**: Use per-IP rate limiting plus IP reputation lists. Consider aggregate rate-based rules for distributed attacks (same login attempt from many IPs).
- **Bot Control** has two levels: Common (classification) and Targeted (ML-based, higher cost). Start with Common and upgrade if scraping persists.
- **Proactive engagement**: Shield Advanced proactive engagement automatically engages the DRT when Route 53 health checks detect issues — no manual escalation needed.
- Test WAF rules with **Count** mode before switching to **Block** — avoids false positives on production traffic.

## Relevant Links

- [AWS Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/shield-chapter.html)
- [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-chapter.html)
- [WAF Bot Control](https://docs.aws.amazon.com/waf/latest/developerguide/waf-bot-control.html)
- [Shield DRT](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-srt-support.html)
- [Shield Cost Protection](https://docs.aws.amazon.com/waf/latest/developerguide/request-refund.html)
