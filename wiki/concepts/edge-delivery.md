---
title: Edge Delivery, Caching, and Global Network Acceleration
type: concept
tags: [edge, cdn, caching, anycast, cloudfront, global-accelerator, tls, authorization]
certs: [aws-saa]
updated: 2026-11-10
---

# Edge Delivery, Caching, and Global Network Acceleration

An edge architecture moves part of an application's delivery path close to users while the main application stays in one or more Regions. [CloudFront](../services/cloudfront.md) moves reusable HTTP content closer through caching; [Global Accelerator](../services/global-accelerator.md) brings TCP/UDP connections onto the AWS backbone earlier. These complement the [global/regional architecture](../patterns/global-regional-architecture.md) model.

## Choose the Mechanism by the Requirement

| Requirement | Mechanism |
| --- | --- |
| Reuse a popular image without repeatedly reading its origin | CloudFront caches |
| Handle distinct application paths under one web hostname | CloudFront behaviors and origins |
| Transform a request or choose content at an edge trigger | [Lambda@Edge](../services/lambda-edge.md) |
| Deliver HTTP private content after application authorization | CloudFront signed URLs/cookies plus protected origin |
| Accelerate a global TCP/UDP application with stable entry IPs | Global Accelerator |
| Discover a hostname's destination | [DNS](dns.md), often [Route 53](../services/route53.md) |
| Prove HTTPS server identity | [TLS](tls-https.md) certificate, often managed by [ACM](../services/acm.md) |

Both edge services can help web applications. “Dynamic” alone is not enough to choose Global Accelerator: consider whether the application needs HTTP routing/transformation or general network acceleration. DNS discovery, application traffic forwarding, and authorization are independent responsibilities.

## Keep Four Boundaries Separate

**Cache correctness:** a response can be shared only across requests for which it is valid. Partition the cache key by meaningful variants or disable caching. A forwarded header does not necessarily participate in that key. TTL controls freshness; invalidation changes cached availability; versioned filenames change object identity.

**Viewer authorization:** the application decides entitlement, and CloudFront checks signed access on protected behaviors. Authenticate before granting signatures; possession of a URL, hostname, or session cookie is not proof of entitlement.

**Origin protection:** restrict the backend so users cannot bypass edge authorization. For S3, prefer OAC with suitable bucket/KMS permissions. Network filtering and custom-header checks serve a different role from a viewer signature.

**Transport security:** CloudFront has separate viewer and origin connections. A browser's secure connection to the edge does not prove that the edge-to-origin connection is encrypted. Regional certificate associations and CloudFront's us-east-1 viewer association are distinct.

## Anycast Entry Versus Regional Destination

Anycast exposes the same entry address at multiple network locations. It does not copy an application into every location. Global Accelerator accepts traffic at an edge and selects a configured endpoint behind that entry point. Regional load balancers and instances keep their own addressing. The [Global Accelerator setup walkthrough](../services/global-accelerator.md#what-happens-after-i-receive-the-ips) connects assigned IPs, listeners, endpoint groups, DNS, and HTTPS testing.

## References

- [CloudFront delivery model](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html)
- [Global Accelerator operation](https://docs.aws.amazon.com/global-accelerator/latest/dg/introduction-how-it-works.html)
- [Lambda@Edge examples](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-examples.html#lambda-examples-redirecting-examples)
- [CloudFront cache key](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/understanding-the-cache-key.html)
