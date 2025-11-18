# DevOps-Design-Take-Home-Assignment
Design a UI, API, and Database system using AWS


---

## Summary
This repository contains a design-only solution for deploying a secure, scalable UI, API, and relational database on AWS. The target architecture uses modern managed services (S3 + CloudFront, ECS Fargate, RDS for PostgreSQL) and includes CI/CD, secrets management, monitoring, and high-availability patterns.

Files included:
- `diagram/architecture.pdf` — exported PDF 
- `README.md` — this file

---

## UI (Frontend) Architecture

**Hosting & routing**
- Host built SPA on **S3** (public-read disabled; use CloudFront signed URL or origin access identity for secure origin access).
- Use **CloudFront** as CDN in front of S3 for global caching, TLS termination (recommended: ACM certificate in us-east-1 for CloudFront), and route-based forwarding.
- Use **Route 53** for DNS and health checks.
- Use **WAF** attached to CloudFront to block OWASP Top 10, rate-limit, or IP blacklists.

**Horizontal scaling**
- Static frontends are stateless: scale is effectively infinite via CDN — no application servers required.
- For assets that require server logic (SSR), use Lambda@Edge or an autoscaled ALB+ECS layer; but for this assignment we use static SPA.

**Traffic routing**
- Client → DNS → CloudFront
- CloudFront cache miss → S3 origin (or ALB for API endpoints)
- Configure CloudFront behaviors: `/static/*` origin S3, `/api/*` or `/graphql` origin ALB (forward headers/cookies as needed)

**Caching / performance**
- CloudFront caches static assets at edge (TTL based on Cache-Control headers). Use versioned filenames for cache invalidation.
- Use gzip/brotli compression at build time.
- Enable HTTP/2 and OCSP stapling via CloudFront.

**Availability zones**
- CloudFront is global; S3 is regional with cross-region replication if multi-region desired.
- For origin infrastructure (ALB/ECS/RDS), deploy across at least **2–3 AZs** for high availability.

**Security**
- Enforce HTTPS with TLS certs (ACM).
- Set strict CSP headers and HSTS in CloudFront.
- Use origin access identity or OAC so S3 objects are not public.

---

## API (Backend) Architecture

**Where it runs**
- **EC2 Auto Scaling Group (ASG)** running backend application instances.
- Launch Template with latest AMI baked via CI/CD (Packer optional).
- EC2 instances deployed across **private subnets** in 2–3 AZs.

**Traffic entry & routing**
- **Application Load Balancer (ALB)** in public subnets.
- Routes `/api/*` to target groups containing EC2 instances.
- ALB performs **health checks** on `/health` endpoint.

**Horizontal scaling**
- Auto Scaling Group scaling policies:
  - **Target Tracking** on CPU usage (e.g., 60%)
  - **Step Scaling** based on request count or ALB metrics
- Cooldown periods to prevent rapid scale in/out.
- Minimum 2 instances across multiple AZs.

**Secrets storage**
- AWS **Secrets Manager** for DB credentials & sensitive API keys.
- Inject via EC2 SSM Agent or environment at boot using `aws secretsmanager get-secret-value`.
- Use **EC2 IAM Role** with least-privilege permissions.

**Service-to-database communication**
- EC2 SG allows outbound only to RDS SG on port 5432.
- RDS SG allows inbound only from EC2 SG and bastion/admin IPs.

**Observability**
- CloudWatch Agent installed on EC2 for logs & metrics.
- X-Ray optional for distributed tracing.
- CloudWatch Alarms for CPU, 5xx errors, ALB latency, ASG instance health.

**Security**
- ALB terminates TLS using ACM.
- EC2 instances in private subnets have no public IP.
- NAT Gateway handles outbound internet for patch updates.
---

## Database Architecture (Postgres)

**Service**: Amazon RDS for PostgreSQL (managed)

**High Availability**
- **Multi-AZ deployment** for automatic failover (synchronous replication to standby in different AZ).
- **Read replicas** (asynchronous) for read-scaling. Place read replicas in different AZs and optionally in different regions if cross-region reads are required.

**Scaling**
- **Vertical scaling**: scale instance class up for CPU/memory when needed (requires brief downtime for Single-AZ; Multi-AZ switch may take longer).
- **Horizontal read-scaling**: Add read replicas and route read-only traffic via a separate connection pool or a read-proxy solution.
- Consider **Amazon Aurora** if you need faster auto-scaling and serverless options in future.

**Backups & DR**
- Enable **automated backups** with retention (e.g., 7–35 days depending on RTO/RPO requirements).
- Enable **Point-in-Time Recovery (PITR)** using continuous WAL archiving (RDS native PITR).
- Daily snapshots to S3 and lifecycle rules to move to Glacier for long-term.
- For cross-region disaster recovery, take regular snapshots and copy to another region; optionally deploy read replica in another region and do a manual failover.

**Migration strategy (schema changes)**
- Use migration tools (Flyway, Liquibase, Alembic) with versioned migration scripts tracked in git.
- Perform migrations in CI with a `migrate` stage; run in staging first.
- Use online-schema-change patterns where possible (backfill new columns with defaults, create new tables and gradually switch reads/writes).
- For backward compatibility: deploy API that supports both old and new schema during migration window; do zero-downtime deploys by rolling updates and feature flags.

**Backups test & restore**
- Periodically test snapshot restores to a restore cluster.
- Document RTO (time to recover) and RPO (data loss tolerance) and run DR drills.

---

## CI/CD Pipeline (Design Only)

**Tooling**: GitHub Actions (recommended)

**Branches & triggers**
- `main` → production
- `staging` → staging
- `feature/*` → PRs

**Pipeline steps**
1. Build & test backend
2. Build AMI via **Packer** (if using baked images)
3. Update Launch Template / Launch Configuration
4. Trigger **EC2 Auto Scaling Group rolling update**
5. ALB health checks ensure zero-downtime rotation
6. Post-deploy smoke tests
7. Manual approval for production

# Operational Considerations

**Monitoring & Alerts**
- CloudWatch metrics and logs for ALB/ECS/RDS.
- X-Ray for tracing requests across API and DB.
- Create alarms for:
  - High error rates (5xx > threshold)
  - Latency SLO breaches
  - CPU/Memory high on EC2 instances
  - DB replica lag exceeding threshold
- Route alerts to SNS → PagerDuty/Slack/Email.

**Cost control**
- Use Fargate spot for non-critical workloads to save cost.
- Use reserved instances or saving plans for RDS if long-term predictable usage.
- Autoscale down at night for non-prod environments.

**Testing & staging**
- Keep an environment parity between staging and production but with scaled-down resources.
- Use database seeding scripts and anonymized production snapshots for realistic testing.

---

# Diagram

**Diagram layout (suggested layers, left→right / top→bottom):**
1. Client (Browser, Mobile)
2. Route 53 (DNS) -> CloudFront -> WAF
3. CloudFront edge cache -> S3 (UI assets) as origin
4. CloudFront forwarding / ALB for API routes (`/api/*`) → ALB in public subnets
5. ALB → EC2 Auto Scaling Group (API Instances) (private subnets, across AZs) → AutoScaling (target tracking on CPU/RPS)
6. ECS Tasks ((optional ECR if using AMI-builder flow; otherwise handled by Packer/AMIs) image) → connect to RDS (Postgres) in private subnets
7. RDS primary (Multi-AZ) with read replicas in same region
8. Secrets Manager / Parameter Store accessed by EC2 instances via IAM role
9. CI/CD (GitHub Actions) -> pushes to ECR and triggers ECS deployment via AWS Deploy / ECS API
10. Monitoring: CloudWatch, X-Ray, SNS Pager alerts
11. Optional: Redis (ElastiCache) for caching sessions and DB offload
12. Optional: S3 lifecycle + backups -> Glacier for long-term retention

---
