# Q03: CloudFront and Global Accelerator for Latency Reduction

## Question

A video streaming company serves content to users worldwide from a single AWS Region (us-east-1). Users in Asia and Europe report **high latency** and **buffering issues**. The application architecture consists of:

- Static content (thumbnails, metadata): served from Amazon S3
- Video streams: served via Amazon EC2 instances running a custom streaming protocol over **TCP**
- API calls: served by an ALB in front of EC2 instances

The company wants to improve global performance. Which combination of services should be used? (Select TWO)

## Options

- **A.** Place Amazon CloudFront in front of the S3 bucket and ALB to cache static content and API responses at edge locations worldwide.
- **B.** Use AWS Global Accelerator to route video streaming traffic over the AWS global network to the EC2 instances in us-east-1. Configure endpoint groups with health checks.
- **C.** Deploy additional EC2 instances in AWS Local Zones close to users in Asia and Europe.
- **D.** Increase the EC2 instance sizes in us-east-1 to handle more concurrent connections.
- **E.** Configure S3 Transfer Acceleration for the video content bucket to speed up downloads.
- **F.** Enable Route 53 geoproximity routing with bias to shift traffic toward us-east-1.

## Answers

### A. CloudFront for static content and API — ✅ Correct

CloudFront caches **static content** (thumbnails, metadata) at 400+ edge locations globally, dramatically reducing latency for repeat requests. For API calls, CloudFront can proxy requests over AWS's backbone network, reducing latency compared to public internet routing. This directly addresses the latency complaints for HTTP/HTTPS-based content.

### B. Global Accelerator for streaming — ✅ Correct

The custom streaming protocol runs over TCP, not HTTP. CloudFront is optimized for HTTP/HTTPS, not arbitrary TCP protocols. AWS Global Accelerator provides **anycast IP addresses** that route TCP/UDP traffic over AWS's private global network instead of the public internet. This reduces jitter, packet loss, and latency for the TCP-based video streams. Global Accelerator doesn't cache content — it optimizes the network path.

### C. Local Zones — ❌ Incorrect

Local Zones extend AWS infrastructure to specific metro areas but are available in limited locations. They don't provide global coverage for users across all of Asia and Europe. Additionally, deploying and managing EC2 instances in multiple Local Zones significantly increases operational complexity without addressing the core issue (network path optimization).

### D. Larger instances — ❌ Incorrect

The problem is **network latency** (distance between users and us-east-1), not server capacity. Larger instances process requests faster but don't reduce the round-trip time for packets traveling across the Pacific or Atlantic. This is a classic distractor — scaling up doesn't solve latency problems.

### E. S3 Transfer Acceleration — ❌ Incorrect

S3 Transfer Acceleration is designed for **uploads** to S3, not downloads. It uses CloudFront edge locations to accelerate data transfer *into* S3. For serving content *from* S3 to users, CloudFront (option A) is the correct service.

### F. Geoproximity routing toward us-east-1 — ❌ Incorrect

Routing more traffic toward us-east-1 would **worsen** the problem for Asian and European users, not improve it. Geoproximity with a positive bias shifts traffic toward a Region, which is the opposite of what's needed. The application only exists in one Region, so routing policies don't help here — you need edge caching (CloudFront) and network optimization (Global Accelerator).

## Recommendations

- **CloudFront vs. Global Accelerator — know the difference:**
  - **CloudFront:** HTTP/HTTPS content caching and acceleration. Best for web content, APIs, static assets.
  - **Global Accelerator:** TCP/UDP network path optimization. Best for non-HTTP protocols, gaming, IoT, VoIP, custom TCP.
  - Both use AWS's global edge network, but for different purposes.
- If the question mentions a **custom protocol** or **TCP/UDP** traffic, think Global Accelerator.
- If the question mentions **caching**, **static content**, or **HTTP APIs**, think CloudFront.
- They can be **used together** for different traffic types, as in this question.
- **S3 Transfer Acceleration ≠ CloudFront.** Transfer Acceleration = uploads. CloudFront = downloads.

## Relevant Links

- [Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html)
- [CloudFront vs Global Accelerator](https://aws.amazon.com/global-accelerator/faqs/)
- [S3 Transfer Acceleration](https://docs.aws.amazon.com/AmazonS3/latest/userguide/transfer-acceleration.html)
