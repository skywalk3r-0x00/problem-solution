## System Architecture Diagram

> Source: `/Problem_2`

![Architecture Diagram](./Solution_architect.svg)

# System Architecture Overview

## 1. Overview Diagram (Logical Table)

| Layer                     | Components                                      | Responsibility                                                                 | HA Strategy                                                                 |
|--------------------------|------------------------------------------------|--------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| Edge + Load Balancing    | Route 53, CloudFront, WAF, Shield              | DNS failover, CDN, DDoS protection, TLS termination                           | Global, no single region dependency                                         |
|                          | ALB + NLB                                      | L7 routing, sticky session, health checks                                     | Multi-AZ by default, cross-zone LB                                          |
| Application (EKS)        | Order Svc, Market Data, WS Gateway, Wallet Svc | Business logic, stateless microservices                                       | 3 replicas minimum per AZ; HPA on CPU + RPS                                 |
| Event Streaming          | MSK (Kafka) 3-broker                           | Decouple producers/consumers, event replay, audit                             | RF=3, min.insync=2; broker spread across AZs                                |
| Matching Engine          | EC2 c6i.2xlarge (primary + hot standby)        | Price-time priority matching (stateful, single-threaded core)                 | Primary/standby; auto-failover via health check                             |
| Data                     | Aurora PG, Redis (A, B), DynamoDB, S3/Athena   | Persistence, caching, analytics                                               | Multi-AZ Aurora; Redis cluster; DynamoDB global                             |
| Observability            | CloudWatch, X-Ray, Grafana, OpenSearch/ELK     | Metrics, distributed tracing, log aggregation                                 | Separate stack, doesn't affect production path                              |
| CI/CD                    | CodePipeline, ECR, Terraform, ArgoCD           | Build, test, deploy with zero-downtime                                        | Blue/green + canary releases on EKS                                         |

---

## 2. Service Choices & Rationale

| No | Choices                     | Why                                                                 |
|----|----------------------------|----------------------------------------------------------------------|
| 1  | Route 53                   | Health check failover: monitors ALB and NLB endpoints               |
| 2  | CloudFront + WAF           | Static assets delivery, WAF blocks SQLi/XSS/bots, edge rate limiting|
| 3  | ALB + NLB                  | ALB for REST (L7 routing), NLB for WebSocket (low latency, sticky)  |
| 4  | EKS (Fargate Mode)         | No node management, sidecar auth pattern, independent scaling       |
| 5  | Amazon MSK (Kafka)         | Replayable events, managed brokers by AWS                           |
| 6  | Matching Engine (EC2)      | Single-thread for correctness, primary/standby for HA               |
| 7  | Aurora PostgreSQL          | Fast failover (<30s), minimal downtime                              |
| 8  | ElastiCache Redis          | Reduce workload, support sticky sessions                            |
| 9  | DynamoDB                   | On-demand scaling, PITR for compliance                              |
| 10 | AWS Global Accelerator     | Instant failover across AZs                                         |

---

## 3. Scaling Plan

### Baseline (Current ~500 RPS)

- 3 EKS pods per service per AZ  
- 1 MSK cluster (3 brokers, 9 partitions/topic)  
- Aurora: db.r6g.2xlarge (primary + 2 replicas)  
- Redis: r6g.large  
- Matching Engine: c6i.2xlarge  
- Estimated cost: **~$8,000–12,000/month**

---

### Scale-Out (Growth ~5000 RPS)

- HPA: 3 → 30 pods per service  
- MSK brokers: 3 → 9  
- Partitions: 9 → 90  
- Aurora read replicas: 2 → 10  
- Matching Engine: c6i.8xlarge  

---

## 4. Key Design Principles

- Stateless microservices for horizontal scalability  
- Event-driven architecture for decoupling  
- Strong consistency in matching engine (single-threaded)  
- Multi-AZ high availability across all critical layers  
- Zero-downtime deployment with blue/green & canary  

---
