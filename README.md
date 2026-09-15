# Portfolio Site — AWS Auto-Scaling Deployment with Jenkins CI/CD

A self-hosted portfolio website deployed on AWS with a fully automated CI/CD pipeline, running behind a load balancer with CPU-based auto scaling.
Every push to `main` on GitHub triggers Jenkins to sync the latest site files to S3 and roll them out to every running EC2 instance via AWS Systems Manager with zero manual intervention and zero downtime.

---

## Architecture

```
                       ┌─────────────────┐
   git push ─────────▶ │     GitHub       │
                       │ (portfolio-site) │
                       └────────┬─────────┘
                                │ webhook (on push)
                                ▼
                       ┌─────────────────┐
                       │  Jenkins (EC2)   │
                       │  Pipeline job    │
                       └────────┬─────────┘
                                │
                  ┌─────────────┴─────────────┐
                  ▼                           ▼
          ┌───────────────┐         ┌──────────────────┐
          │   S3 bucket    │         │   AWS SSM         │
          │ (site source   │────────▶│ Run Command       │
          │  of truth)     │  sync   │ (push to fleet)   │
          └───────────────┘         └─────────┬─────────┘
                                                │
                     ┌──────────────────────────┼──────────────────────────┐
                     ▼                          ▼                          ▼
             ┌───────────────┐         ┌───────────────┐          ┌───────────────┐
             │  EC2 instance  │  ...    │  EC2 instance  │   ...    │  EC2 instance  │
             │ (Nginx, ASG)   │         │ (Nginx, ASG)   │          │ (Nginx, ASG)   │
             └───────┬───────┘         └───────┬───────┘          └───────┬───────┘
                     └──────────────────────────┼──────────────────────────┘
                                                 ▼
                                    ┌─────────────────────────┐
                                    │  Application Load        │
                                    │  Balancer (public entry)  │
                                    └─────────────────────────┘
                                                 ▲
                                          End users (HTTP)
```

The Auto Scaling Group sits behind the ALB and always keeps **at least one instance running**. CloudWatch alarms watch average CPU utilization across the fleet and trigger step-scaling policies:

- **CPU > 70%** for 2 consecutive minutes → **add 1 instance**
- **CPU < 30%** for 2 consecutive minutes → **remove 1 instance**
- Bounded between **min: 1** and **max: 4** instances

---

## Tech Stack

| Layer               | Tool / Service                          |
|---------------------|------------------------------------------|
| Source control      | GitHub                                   |
| CI/CD               | Jenkins (self-hosted on EC2)             |
| Artifact storage    | Amazon S3                                |
| Fleet deployment    | AWS Systems Manager (Run Command)        |
| Compute             | EC2 (Amazon Linux 2023, `t2.micro`)      |
| Scaling             | Auto Scaling Group + CloudWatch alarms   |
| Load balancing      | Application Load Balancer                |
| Web server          | Nginx                                    |
| IAM                 | Scoped instance roles for EC2 & Jenkins  |

---

## Repository Structure

```
portfolio-site/
├── Jenkinsfile              # CI/CD pipeline definition
├── index.html                # portfolio page
├── screenshots/                  # (Realtime Screenshots)
└── README.md
```

---

## How the Pipeline Works

1. A commit is pushed to `main` on GitHub.
2. A GitHub webhook notifies Jenkins instantly.
3. Jenkins checks out the repo and runs the pipeline defined in `Jenkinsfile`:
   - **Stage 1 — Sync to S3:** uploads the current site files to a dedicated S3 bucket, acting as the deployment's single source of truth.
   - **Stage 2 — Deploy to fleet:** looks up every `InService` instance in the Auto Scaling Group and sends an SSM Run Command that pulls the latest files from S3 into `/usr/share/nginx/html` and restarts Nginx.
4. New instances launched by the ASG (during scale-out events) come up with Nginx and the SSM agent pre-installed via the launch template's user-data script, so they're immediately reachable for the next deployment.

---

## Infrastructure Setup Summary

| Component            | Notes                                                         |
|-----------------------|----------------------------------------------------------------|
| IAM — `PortfolioEC2Role`   | S3 read + SSM managed-instance core, attached to portfolio instances |
| IAM — `JenkinsDeployRole`  | S3 full access + SSM full access, attached to the Jenkins instance |
| Security group — ALB       | Inbound HTTP (80) from anywhere |
| Security group — EC2 fleet | Inbound HTTP (80) only from the ALB's security group; SSH (22) restricted to admin IP |
| Security group — Jenkins   | Inbound 8080 (Jenkins UI + GitHub webhook), SSH (22) restricted to admin IP |
| Launch template            | Amazon Linux 2023, installs Nginx + SSM agent readiness via user data |
| Auto Scaling Group         | Min 1 / Desired 1 / Max 4, registered to the ALB target group |

---

## Notes on Design Decisions

- **GitHub instead of CodeCommit:** AWS closed CodeCommit to new customers in July 2024, so this project uses GitHub as the source repository.
- **Jenkins instead of CodePipeline/CodeDeploy:** the initial design used CodePipeline + CodeDeploy, but the project pivoted to a self-hosted Jenkins pipeline for the CI/CD layer.
- **S3 + SSM instead of CodeDeploy in-place deployment:** since CodeDeploy was dropped, deployments to the Auto Scaling fleet are handled by syncing the built site to S3 and pushing it out to all running instances with SSM Run Command achieving the same "deploy to every instance in the ASG" outcome without CodeDeploy.

---

## Possible Next Steps

- Rebuild this entire infrastructure as Terraform for one-command provisioning and teardown.
- Move Jenkins itself onto Docker or a managed service to reduce operational overhead.
- Add HTTPS via an ACM certificate on the ALB listener.
- Tighten IAM policies from `FullAccess` to least-privilege, scoped to the specific bucket and ASG.

---

## Author

**Bhaskar Kushwah** — Cloud & DevOps Engineer
