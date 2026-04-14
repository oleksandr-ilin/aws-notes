# Q04: Global Accelerator vs. CloudFront

## Question

A company operates a real-time bidding platform for digital advertising. The platform handles TCP-based WebSocket connections and UDP traffic for bid requests. Users connect globally, and the company needs to reduce latency for connections from Asia-Pacific and Europe to the application running in us-east-1. The application is NOT HTTP-based — it uses custom TCP and UDP protocols on specific ports.

Which service should the solutions architect recommend for global performance optimization?

## Options

- **A.** Deploy Amazon CloudFront with the application's ALB as the origin. Configure CloudFront to cache bid responses at edge locations worldwide.
- **B.** Deploy AWS Global Accelerator with the application's Network Load Balancer as the endpoint. Use static Anycast IP addresses for client connections. Global Accelerator routes traffic over the AWS global network to the nearest edge location.
- **C.** Deploy the application in multiple Regions and use Route 53 latency-based routing to direct users to the nearest Region.
- **D.** Deploy Amazon CloudFront with a custom origin using TCP on the required ports. Enable CloudFront's WebSocket support.

## Answers

### B. AWS Global Accelerator — ✅ Correct

Global Accelerator is designed for non-HTTP, latency-sensitive applications:
- **TCP and UDP support**: Handles any TCP or UDP traffic on any port — not limited to HTTP/HTTPS. Ideal for WebSocket and custom protocols.
- **AWS global network**: Traffic enters the AWS network at the nearest edge location (via Anycast) and travels over optimized AWS backbone instead of the public internet. Reduces jitter and packet loss for real-time applications.
- **Static Anycast IPs**: Two static IP addresses that clients connect to globally. No DNS TTL issues — routing changes happen at the network level instantly.
- **Health checking and failover**: Built-in endpoint health checks with automatic failover to healthy endpoints.
- **No caching**: Global Accelerator is a network-layer accelerator, not a CDN — every request reaches the origin. Ideal for dynamic, real-time traffic.

### A. CloudFront as CDN — ❌ Incorrect

CloudFront is a CDN designed for HTTP/HTTPS content delivery with caching. It does **not** support UDP traffic. While CloudFront supports WebSocket connections, it's optimized for cacheable HTTP content. A real-time bidding platform with unique bid requests for every connection wouldn't benefit from caching. CloudFront's TCP support is limited to HTTP/HTTPS and WebSocket — not arbitrary TCP protocols.

### C. Multi-Region with latency routing — ❌ Incorrect

Multi-Region deployment would reduce latency but requires replicating the entire application stack, data layer, and state management across Regions. For a real-time bidding platform, state synchronization across Regions adds complexity and potential inconsistency. Global Accelerator achieves latency reduction with a single-Region deployment by optimizing the network path.

### D. CloudFront custom TCP origin — ❌ Incorrect

CloudFront only supports HTTP, HTTPS, and WebSocket protocols. It cannot proxy arbitrary TCP connections on custom ports, and it does not support UDP at all. Custom origin configuration allows setting the origin port and protocol, but the viewer-facing side must be HTTP/HTTPS.

## Recommendations

- **CloudFront** = HTTP/HTTPS content delivery with caching. Best for: web applications, static assets, video streaming (HLS/DASH), API acceleration.
- **Global Accelerator** = TCP/UDP network acceleration without caching. Best for: gaming, IoT, VoIP, real-time bidding, financial trading, non-HTTP protocols.
- Global Accelerator integrates with ALB, NLB, EC2 instances, and Elastic IPs as endpoints.
- Consider **Global Accelerator custom routing** for applications that need deterministic port/IP mapping (e.g., multiplayer gaming).
- Both services use the AWS global edge network — the difference is protocol support and caching behavior.

## Relevant Links

- [AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html)
- [Global Accelerator vs. CloudFront](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html)
- [CloudFront Supported Protocols](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-web-values-specify.html)
- [Global Accelerator Custom Routing](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-custom-routing-accelerators.html)
