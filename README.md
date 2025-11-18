# DevOps-Design-Take-Home-Assignment
Design a UI, API, and Database system using AWS


---

## Summary
This repository contains a design-only solution for deploying a secure, scalable UI, API, and relational database on AWS. The target architecture uses modern managed services (S3 + CloudFront, ECS Fargate, RDS for PostgreSQL) and includes CI/CD, secrets management, monitoring, and high-availability patterns.

Files included:
- `diagram/architecture.drawio` — draw.io source 
- `diagram/architecture.pdf` — exported PDF 
- `README.md` — this file

---

## UI (Frontend) — design details
**Hosting & routing**
- Build artifact (static files) uploaded to S3 (origin access via OAC or Origin Access Identity).
- CloudFront (edge caching) in front of S3. CloudFront behaviors:
  - `/*` serve static assets from S3
  - `/api/*` forward to ALB origin (no caching) with necessary headers forwarded

**Scaling & availability**
- Static content scales via CDN, infinite edge capacity.
- For SSR or server-side needs, place that behind ALB + Fargate and scale with ECS autoscaling.

**Security & performance**
- Use ACM TLS cert attached to CloudFront; enforce HTTPS.
- Use CloudFront + WAF to protect against OWASP Top 10.
- Cache-control headers, versioned assets, GZIP/Brotli at build.

---

## API (Backend) — design details
**Where it runs**
- ECS Fargate tasks (private subnets) with ECR-hosted images. Task role grants Secrets Manager and CloudWatch permissions.

**Networking**
- ALB in public subnets terminates TLS and routes `/api/*` to ECS target groups in private subnets across 2–3 AZs.
- Security groups: ALB (443 public), ECS tasks (only accept traffic from ALB), RDS (only accept from ECS SG and admin IP).

**Scaling**
- Target-tracking autoscaling using CPU or custom metrics (requests per target); scale-out/in policies defined in ECS Service Auto Scaling.
- Use CloudWatch metrics for scaling decisions (latency, CPU, custom app metrics).

**Secrets**
- Store DB credentials and third-party keys in Secrets Manager. Inject secrets into tasks via the ECS Secrets integration.

**Observability**
- Application logs → CloudWatch Logs.
- Distributed tracing using X-Ray (instrument API).
- Alarms for 5xx error rate, latency, CPU, DB replica lag.

---

## Database (Postgres) — design details
**High availability**
- Amazon RDS for PostgreSQL with **Multi-AZ** deployment (synchronous standby).
- Read replicas for read scaling (asynchronous).

**Backups & PITR**
- Automated backups enabled; PITR configured with daily retention per requirements.
- Snapshots copied to another region for DR (optionally scheduled).

**Scaling**
- Vertical: change instance class for heavier workloads.
- Horizontal (read): add read replicas, aggregate reads through a read pool or use a proxy (PgBouncer or RDS Proxy).

**Migration & schema changes**
- Use migration tooling (Flyway / Liquibase / Alembic) with versioned scripts and run in CI staging first.
- Apply backward-compatible changes; use feature flags for rollout.

---

## CI/CD Design (GitHub Actions) — summary
- Branch model: `feature/*` → `staging` → `main`.
- Triggers:
  - Push to `feature/*`: lint & unit tests.
  - PR to `staging`: full test suite & security scans.
  - Merge to `staging`: auto-deploy to staging.
  - Promotion to `main`: manual approval + deploy to prod.
- Pipeline steps:
  1. Build frontend (if changed) and produce static artifacts.
  2. Build backend image and run unit tests.
  3. Security scans (Trivy/Snyk) in CI.
  4. Push container image to ECR.
  5. Update ECS task definition & service; use ALB health checks.
  6. Post-deploy smoke tests; rollback on failures.

---
