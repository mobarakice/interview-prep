# AWS Infrastructure Cheatsheet

> Quick-reference guide for AWS networking, load balancing, IAM, database clustering, and security groups

---

## 1. VPC Network Architecture

- **Subnet Allocation Rules**:
  - **Public**: Web load balancers, NAT Gateways, Bastion hosts. Direct internet route (`0.0.0.0/0` -> Internet Gateway).
  - **Private**: Application servers, EKS Nodes. Outbound route via NAT Gateway (`0.0.0.0/0` -> NAT).
  - **Isolated**: Databases, RDS clusters. No external routes.
- **VPC Peering vs Transit Gateway**:
  - Peering: Best for simple point-to-point connections. Non-transitive routing.
  - Transit Gateway: Hub-and-spoke model. Best for routing across multiple VPCs and VPN connections.

---

## 2. Load Balancing & Edge Delivery

| Service | Target Layer | Best For | Key Feature |
|---|---|---|---|
| **ALB (Application)** | Layer 7 (HTTP/HTTPS) | HTTP path routing, headers | WAF integration, OIDC authentication |
| **NLB (Network)** | Layer 4 (TCP/UDP) | High-speed low-latency TCP traffic | Dynamic IP static assignment |
| **CloudFront** | Global Edge (CDN) | Static asset cache, global delivery | Origin Shield caching, Edge certificate |

---

## 3. Database Replication & Storage Options

- **Amazon RDS Multi-AZ**: High-availability synchronous replication to a secondary subnet in another Availability Zone. Automated failover in < 60 seconds.
- **Amazon RDS Read Replicas**: Asynchronous replication for read-scaling. Can be promoted to primary if disaster recovery occurs.
- **S3 Storage Classes**:
  - Standard: Hot data access.
  - Standard-IA (Infrequent Access): Data accessed less than once a month, retrieval fee.
  - Glacier Flexible / Deep Archive: Cold storage, long retrieval latency (minutes to hours).
