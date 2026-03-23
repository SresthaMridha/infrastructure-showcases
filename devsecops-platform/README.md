# DevSecOps Platform Infrastructure

A cloud-native DevSecOps platform on AWS — fully automated from infrastructure provisioning to container deployment. Push code to GitHub. Everything else is automatic.

> Implementation code is maintained in a private repository. Available to verified recruiters and hiring engineers on request — [LinkedIn](https://www.linkedin.com/in/srestha-mridha/) or [email](mailto:sresthamridha@gmail.com).

---

## What This Does

A developer pushes to GitHub. From that point:

- ArgoCD detects the change and syncs Kubernetes manifests to the cluster
- Kaniko builds the container image inside the cluster — no Docker daemon, no privileged containers
- The image is pushed to ECR and automatically scanned for vulnerabilities
- The cluster state is continuously reconciled to match Git — manual changes are corrected automatically
- AWS authentication happens via federated identity tied to the Kubernetes ServiceAccount — no static credentials anywhere

No manual steps. No credentials to rotate. No open ports.

---

## Architecture

```
Developer
    │
    │  git push
    ▼
GitHub Repository
    │
    ├──► ArgoCD (GitOps)
    │        │
    │        │  watches + syncs
    │        ▼
    │    EKS Cluster  [private subnets]
    │        │
    │        ├── Kaniko Build Job
    │        │       │
    │        │       │  IRSA (keyless auth)
    │        │       ▼
    │        │   ECR (image push + scan)
    │        │
    │        └── Application Workloads
    │
    └── VPC Interface Endpoints (ECR, EKS, STS, S3, EC2)
              │
              ▼
          AWS Services  ← all traffic stays private, no internet
```

---

## Stack

| Layer | Technology | Decision |
|---|---|---|
| Cloud | AWS | — |
| Infrastructure as Code | Terraform | Everything provisioned as code, nothing clicked |
| Container Orchestration | Amazon EKS | Managed control plane, SPOT node group across 2 AZs |
| GitOps | ArgoCD | Git as single source of truth, continuous reconciliation |
| Image Builds | Kaniko | Daemonless, no privileged containers required |
| Image Registry | Amazon ECR | Automated vulnerability scanning on every push |
| Authentication | IRSA + OIDC | Keyless, federated, per-workload identity |
| Networking | VPC Interface Endpoints | No NAT Gateway, no internet egress |

---

## VPC Design

All workloads run in private subnets. No NAT Gateway.

```
VPC
├── Public subnet  (AZ1) — minimal, future ALB use
├── Public subnet  (AZ2) — minimal, future ALB use  
├── Private subnet (AZ1) — EKS nodes, endpoints
└── Private subnet (AZ2) — EKS nodes, endpoints
```

**VPC Endpoints:**
- Interface: ECR API, ECR DKR, EKS, STS, EC2
- Gateway (free): S3

All outbound AWS service traffic — ECR pulls, STS token exchange, EKS API calls — routes through these endpoints and never touches the public internet.

---

## Key Design Decisions

**No NAT Gateway** — NAT Gateway costs ~$0.045/hr plus per-GB data charges. Every AWS service call in this platform routes through VPC Interface Endpoints instead. The endpoint cost is significantly lower at this scale and keeps all traffic within the AWS private network.

**Daemonless builds with Kaniko** — Docker-in-Docker requires privileged containers, which is a significant security risk in a shared cluster. Kaniko builds images from a Dockerfile as an unprivileged process inside the cluster, with no Docker socket mounted. Combined with IRSA, no credentials are ever injected — the build job authenticates to ECR using its Kubernetes ServiceAccount identity.

**IRSA over static credentials** — Every workload that calls an AWS service authenticates via IAM Roles for Service Accounts. The Kubernetes ServiceAccount token is exchanged for short-lived AWS credentials via STS. The IAM trust policy is scoped to a specific namespace and ServiceAccount name — no other workload in the cluster can assume the role.

**GitOps over imperative deployments** — ArgoCD continuously watches the repository. The cluster always reflects what is in Git. There is no `kubectl apply` in a pipeline, no manual deployment step, and no drift between what was deployed and what is in source control. Any manual change to the cluster is automatically corrected.

**SPOT node group** — Multiple instance types specified across two AZs to maximise SPOT availability. Significantly reduces compute cost with no impact on workload behaviour for this use case.

---

## What Broke and How I Fixed It

### Silent node group failure — 30+ minutes

The node group sat in a creating state for over 30 minutes without completing. No error. Just waiting.

**Root cause:** EKS worker nodes in private subnets need to reach the EKS API, download bootstrap scripts, and register themselves on first boot. There was no NAT Gateway and no VPC endpoints yet. The nodes were booting, trying to register, getting no response, and silently timing out.

**Fix:** Added VPC Interface Endpoints for ECR, EKS, STS, EC2, and a Gateway endpoint for S3. Also added the `kubernetes.io/cluster/` tag to private subnets — EKS uses this for subnet discovery, and missing it causes silent issues with load balancers and internal networking later.

---

### Multi-AZ subnet error on first apply

```
InvalidParameterException: Subnets specified must be in at least two different AZs
```

Initial design only had one subnet. EKS requires subnets across at least two AZs for high availability. Added a second subnet in a different AZ and assigned both explicitly.

---

### CIDR conflict on rebuild

```
InvalidSubnet.Conflict: The CIDR '10.0.2.0/24' conflicts with another subnet
```

After a failed apply and `terraform destroy`, the old subnets hadn't fully released before the re-apply. AWS takes time to release networking resources after destroy. Waited, then re-applied clean.

---

### IRSA silently not working

Pods were authenticating as the wrong role. No error — just wrong identity when calling `aws sts get-caller-identity`.

Two things were wrong:

1. The `tls` provider wasn't declared in `required_providers`. The OIDC setup uses `tls_certificate` to fetch the endpoint thumbprint — this is from the `hashicorp/tls` provider, not AWS. Missing declaration causes a provider not found error.

2. `endpoint_private_access = true` wasn't set on the EKS cluster `vpc_config`. Without it, the OIDC token exchange via STS can't complete from inside the VPC. The node registers fine but IRSA silently fails.

---

### ArgoCD ImagePullBackOff

ArgoCD pods immediately went into `ImagePullBackOff` after installation. ArgoCD images are hosted on `quay.io` — an external registry the private subnet nodes couldn't reach.

Short-term: temporarily moved nodes to public subnets to get the platform running. Mirroring all third-party images to ECR before deploying tools that need them is the correct long-term solution — and the lesson I'd apply from day one on the next build.

---

### ArgoCD repo not found

ArgoCD was configured to watch the GitHub repo via HTTPS, but SSH credentials were registered separately. ArgoCD treats these as different remotes. The Application CRD pointed at the HTTPS URL while credentials were for the SSH URL — `Repository not found` even though access was fine.

Fixed by patching the Application to use the SSH URL consistently.

---

### Kaniko cache repo missing

```
Error uploading layer to cache: repository 'devsecops-platform-infra-app/cache' does not exist
```

The `--cache-repo` flag in Kaniko args points to a separate ECR repository for layer caching. Kaniko doesn't create it automatically. Created it manually with `aws ecr create-repository`. Subsequent builds used cached layers and ran significantly faster.

---

## What I'd Do Differently

**Design the network before writing any Terraform.** The NAT Gateway vs VPC endpoints decision affects node group subnet placement, IAM trust policies, and ArgoCD image pull strategy. Getting that clarity upfront would have prevented the 30-minute silent node failure entirely.

**Mirror third-party images to ECR before deploying tools that need them.** Any tool pulling from quay.io, ghcr.io, or Docker Hub will fail immediately in a private subnet cluster. Always mirror first.

**Test IRSA in isolation first.** A single pod running `aws sts get-caller-identity` is enough to verify the full role assumption chain before adding Kaniko on top of it.

**Initialise `.gitignore` before the first commit.** The `.terraform/` directory contains provider binaries (~674MB). It ended up in git history and required a `git filter-branch` rewrite to remove it. `terraform.tfstate` should also never be committed — it contains resource IDs and ARNs in plaintext.

---

## What I'm Still Deepening

- IRSA and OIDC internals — the full token exchange flow between Kubernetes, OIDC, and STS
- ArgoCD Image Updater — automating image tag commits back to Git to fully close the loop
- Mirroring all third-party images to ECR and moving nodes back to private subnets
- Kaniko layer caching strategy for larger, more complex builds

---

**Srestha Mridha** · Cloud & DevOps Engineer · Kolkata, India  
[LinkedIn](https://www.linkedin.com/in/srestha-mridha/) · [GitHub](https://github.com/SresthaMridha) · [sresthamridha@gmail.com](mailto:sresthamridha@gmail.com)
