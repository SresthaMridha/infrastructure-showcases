# Infrastructure Showcases

Architecture write-ups for infrastructure projects I've built privately.

These repositories are kept private to protect the implementation details — but the engineering decisions, architecture, and debugging process are documented here in full. No code. Just the thinking.

---

## Projects

### [DevSecOps Platform on AWS EKS](./devsecops-platform/)
> `Terraform` · `EKS` · `ArgoCD` · `Kaniko` · `IRSA` · `ECR` · `VPC Endpoints`

A cloud-native DevSecOps platform built from scratch on AWS — fully automated GitOps pipeline where pushing to Git is the only action a developer takes. EKS cluster provisioned entirely via Terraform, daemonless image builds inside the cluster with Kaniko, keyless AWS authentication via IRSA, and private networking through VPC Interface Endpoints with no NAT Gateway.

Includes full architecture breakdown, key design decisions, and a raw engineering journal documenting what broke and how it was debugged.

---

### [Serverless AWS Portfolio Infrastructure](./serverless-portfolio/)
> `Terraform` · `Lambda` · `ALB` · `CloudFront` · `SNS` · `SQS` · `DynamoDB` · `WAF` · `HashiCorp Vault` · `SSM`

Production-grade, fully event-driven serverless infrastructure on AWS — built to the same standards I'd apply in a real production environment. Private subnets only, 10 VPC endpoints replacing NAT Gateway, self-hosted Vault on EC2, zero SSH access, zero hardcoded credentials. Deployed and validated end-to-end, then destroyed to avoid ongoing costs.

Includes full architecture diagram, VPC design, stack rationale, and 21 documented engineering challenges from the build.

---

## Why Private Code

The infrastructure code in these projects represents original design work — specific architectural patterns, security constraints, and engineering decisions that took real time to figure out. Keeping the code private isn't about hiding basics. It's about not handing over work that took genuine effort to produce.

If you're a recruiter or hiring engineer and want to see the full implementation, reach out via [LinkedIn](https://www.linkedin.com/in/srestha-mridha/) or [email](mailto:sresthamridha@gmail.com).

---

**Srestha Mridha** · Cloud & DevOps Engineer · Kolkata, India
