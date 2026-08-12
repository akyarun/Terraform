
**1.Ques** Describe production systems you have personally owned. Include details on scale, infrastructure footprint, critical services, availability requirements, on-call responsibilities, and your decision-making authority.

**Ans:**

**Strong answer structure**

Use:

**Environment → Scale → Architecture → Criticality → Responsibilities → Incident/on-call → Decisions**

"In my current environment, I work on a production microservices platform running primarily on AWS EKS. The platform consists of multiple Kubernetes clusters across environments and regions, with production clusters supporting customer-facing microservices.

The infrastructure includes EKS, EC2 worker nodes, Aurora PostgreSQL, MSK Kafka, ALB, CloudFront, WAF, IAM, CloudWatch, Prometheus, Grafana and Thanos. Infrastructure and environment configuration are managed through Terraform/Terragrunt, while application and platform deployments are automated through GitHub Actions and Helm.

From a service perspective, the critical components include the application microservices, Kafka-based communication, Aurora databases, service mesh and the observability platform. Because these are production services, availability and recovery are important considerations, so we use health checks, autoscaling, monitoring, alerting and controlled deployment strategies.

I have also worked on changes involving Istio configuration, Prometheus/Grafana, Kafka monitoring, Aurora migration activities and AWS security components.



**Ques**:"What did you personally own?"
**Ans** : "I personally owned the implementation and operational side of..."

* Observability: "I owned Prometheus/Grafana configuration and troubleshooting, including missing metrics, dashboard issues and monitoring configuration across clusters."

* CI/CD: "I worked on GitHub Actions workflows, Helm validation and deployment automation."

* Infrastructure: "I worked with Terraform/Terragrunt for AWS infrastructure and environment-specific configuration."



**2.Ques**Design a safe promotion strategy for application, infrastructure, and database changes across dev, staging, and production environments. Include details on gates, migrations, rollback procedures, and verification steps.

**Ans**:

You need to show that application, infrastructure and database deployments cannot be treated identically.

pipeline would be like this

```yaml
Developer
   |
   v
Pull Request
   |
   +---- Code Review
   +---- Unit Tests
   +---- Security Scan
   +---- Terraform Validation
   +---- Helm Validation
   |
   v
DEV
   |
   v
Integration Tests
   |
   v
STAGING
   |
   +---- Functional Tests
   +---- Performance Tests
   +---- Security Tests
   |
   v
PRODUCTION APPROVAL
   |
   v
PRODUCTION
   |
   +---- Canary / Blue-Green
   |
   v
Post-deployment Verification
```

Application changes would be like:

```yaml
PR
 ↓
Build
 ↓
Unit Test
 ↓
SAST/SCA
 ↓
Container Scan
 ↓
DEV
 ↓
Integration Test
 ↓
STAGING
 ↓
Approval
 ↓
Canary Production
 ↓
Monitor
 ↓
Full rollout
```
Infrastructure changes would be like:

```yaml
terraform fmt
      ↓
terraform validate
      ↓
security scan
      ↓
terraform plan
      ↓
PR review
      ↓
Approval
      ↓
Apply
```

*

Explain your process for promoting global reference data (e.g., service plans, default rates, vendor costs) without impacting individual tenant data. Cover classification, packaging, approval workflows, import processes, reconciliation, and auditing.
*

How would you secure tenant-configured outbound webhooks against threats like private network access, metadata service enumeration, malicious DNS changes, and unsafe redirects, while ensuring legitimate endpoints remain functional?
*

A provider is accepting requests, but callbacks are intermittent or silent. Describe how you would determine the scope, attribute the traffic, measure account health, alert stakeholders, and communicate impact.
*

Define your proposed rules for allowing AI agents to make changes to infrastructure, CI/CD pipelines, or production-adjacent code. Include requirements for automation controls, human-in-the-loop approvals, and required evidence.
*

How would you manage your first 30 days leading work split between internal employees and an outsourced partner, specifically where there is incomplete documentation and unclear delivery, without causing disruption?