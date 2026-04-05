# From Git Push to Running in Production on EKS

### A complete, hands-on walkthrough of building a production-grade AWS CI/CD pipeline using CodeCommit, CodeArtifact, CodeBuild, CodePipeline, CodeDeploy, Amazon EKS, and ArgoCD — wired together into one automated delivery system.

---

Modern software delivery is no longer about running a script or clicking a deploy button. It's a disciplined, automated chain — where every code change flows through security checks, automated tests, container builds, and policy-gated deployments before a single pod restarts in production.

This post walks through building that exact chain using AWS-native tools. We'll deploy a simple Flask REST API called **TaskFlow** — but the architecture and every command here applies to any real production service.

> *"Every artifact is scanned, validated, and deployed in a controlled, auditable, and secure manner — that's the standard this pipeline holds itself to."*

---

## What You'll Build

One `git push` triggers a pipeline that:

1. Pulls Python dependencies from a private **CodeArtifact** package registry — not public PyPI
2. Runs all tests and builds a Docker image in **CodeBuild**
3. Pushes the image to **ECR** tagged with the exact git commit SHA
4. Waits for a **manual approval** in CodePipeline before deploying
5. Uses **CodeDeploy lifecycle hooks** to safely roll out to EKS with auto-rollback on failure
6. Meanwhile, **ArgoCD** watches the `k8s/` manifests in Git and auto-syncs any infrastructure changes

**The full pipeline flow:**

```
CodeCommit → CodePipeline → CodeArtifact → CodeBuild → Manual Approve → CodeDeploy → EKS
                                                ↕
                              ArgoCD (GitOps) watches k8s/ → auto-syncs to EKS
```

---

## The Application: TaskFlow API

The app itself is intentionally simple — a task manager with CRUD endpoints plus `/health`, `/ready`, and `/live` endpoints. Those last three are what Kubernetes uses to decide whether to send traffic to a pod and whether to restart it.

```
aws-devops-lab/
├── app/app.py            ← Flask API: /health /ready /live + CRUD
├── Dockerfile            ← BUILD_NUMBER + IMAGE_TAG injected by CodeBuild
├── requirements.txt      ← Pulled from CodeArtifact, not public PyPI
├── buildspec.yml         ← Full CodeBuild spec (4 phases)
├── appspec.yml           ← CodeDeploy lifecycle hook config
├── scripts/              ← 4 hook scripts (before/after/validate)
├── k8s/                  ← Namespace, ConfigMap, Deployment, Service, HPA
├── argocd/               ← Application CRD + CodeCommit repo secret
└── pipeline/             ← CodePipeline JSON + IAM policies
```

> **Setup Order Matters:** Follow this sequence — EKS → ECR → CodeArtifact → CodeCommit → CodeBuild → CodePipeline → CodeDeploy → ArgoCD. Each service depends on the one before it being ready.

---

## 1. AWS CodeArtifact — Private Package Registry

Your builds should never depend on the public internet. CodeArtifact proxies PyPI and caches every package your project uses — giving you full control, auditability, and the ability to block vulnerable versions.

A **domain** is the top-level namespace. Inside it, you create repositories. The pattern here: one upstream `pypi-store` that talks to public PyPI and caches everything, and one `taskflow-repo` that your projects actually pull from.

### Create Domain and Repositories

```bash
# 1. Create the domain (one per AWS account/org)
aws codeartifact create-domain \
  --domain taskflow-domain \
  --region ap-south-1

# 2. Create the upstream PyPI proxy store
aws codeartifact create-repository \
  --domain     taskflow-domain \
  --repository pypi-store

aws codeartifact associate-external-connection \
  --domain              taskflow-domain \
  --repository          pypi-store \
  --external-connection public:pypi

# 3. Create your project repo with pypi-store as upstream
aws codeartifact create-repository \
  --domain     taskflow-domain \
  --repository taskflow-repo \
  --upstreams  repositoryName=pypi-store
```

### Test pip Login Locally

Before wiring this into CodeBuild, verify it works end-to-end from your local machine. The auth token is short-lived (12 hours) — in CI, CodeBuild generates it fresh at the start of every build.

```bash
# Configure pip to point at CodeArtifact
aws codeartifact login \
  --tool         pip \
  --domain       taskflow-domain \
  --domain-owner $(aws sts get-caller-identity --query Account --output text) \
  --repository   taskflow-repo

# Install — packages now come from CodeArtifact (cached from PyPI)
pip install flask==3.0.3 gunicorn==22.0.0

# Confirm packages were cached in your repo
aws codeartifact list-packages \
  --domain     taskflow-domain \
  --repository taskflow-repo \
  --format     pypi
```

**Why this matters:** If PyPI goes down — or if a malicious package gets published — your builds are unaffected because they're hitting your cached registry. You can also pin exact approved versions per project and block packages that fail your security review.

---

## 2. AWS CodeBuild — Build and Test

Serverless build runner. No infrastructure to manage. Every build starts in a fresh container, runs your tests, builds and pushes the Docker image, and writes deployment metadata for downstream stages.

### Understanding buildspec.yml

The `buildspec.yml` defines four phases. Failure in any command fails the entire build — which is exactly what you want.

| Phase | What Happens |
|-------|-------------|
| `install` | Sets up Python 3.12 runtime, upgrades pip |
| `pre_build` | Authenticates to ECR, gets CodeArtifact token, installs deps, sets `IMAGE_TAG` = first 8 chars of git commit SHA |
| `build` | Runs `pytest` with coverage, then `docker build` tagged with git SHA and `latest` |
| `post_build` | Pushes both tags to ECR, updates `k8s/deployment.yaml` image tag via `sed`, writes `imagedefinitions.json` for CodeDeploy |

```yaml
phases:
  pre_build:
    commands:
      # Authenticate to ECR
      - aws ecr get-login-password --region $AWS_REGION |
          docker login --username AWS --password-stdin
          $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

      # Authenticate to CodeArtifact and configure pip
      - aws codeartifact login --tool pip
          --domain $DOMAIN_NAME --repository $REPO_NAME

      - pip install -r requirements.txt

      # Tag = first 8 chars of git commit SHA
      - export IMAGE_TAG=${CODEBUILD_RESOLVED_SOURCE_VERSION:0:8}

  build:
    commands:
      - pytest tests/ -v --cov=app --cov-report=xml
      - docker build --platform linux/amd64
          --build-arg BUILD_NUMBER=$CODEBUILD_BUILD_NUMBER
          --build-arg IMAGE_TAG=$IMAGE_TAG
          -t $ECR_URI:$IMAGE_TAG -t $ECR_URI:latest .

  post_build:
    commands:
      - docker push $ECR_URI:$IMAGE_TAG
      - docker push $ECR_URI:latest
      # Update deployment.yaml with new image tag
      - sed -i "s|image:.*taskflow-api.*|image: $ECR_URI:$IMAGE_TAG|g"
          k8s/deployment.yaml
      # Required by CodeDeploy
      - printf '[{"name":"taskflow-api","imageUri":"%s"}]'
          $ECR_URI:$IMAGE_TAG > imagedefinitions.json
```

### Create the CodeBuild Project

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create ECR repository
aws ecr create-repository --repository-name taskflow-api

# Create the CodeBuild project
aws codebuild create-project \
  --name        taskflow-api-build \
  --source      '{"type":"CODECOMMIT","location":"https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/taskflow-api"}' \
  --artifacts   '{"type":"NO_ARTIFACTS"}' \
  --environment '{
    "type":"LINUX_CONTAINER",
    "image":"aws/codebuild/standard:7.0",
    "computeType":"BUILD_GENERAL1_SMALL",
    "privilegedMode":true,
    "environmentVariables":[
      {"name":"AWS_ACCOUNT_ID","value":"'$ACCOUNT_ID'"},
      {"name":"AWS_REGION","value":"ap-south-1"},
      {"name":"DOMAIN_NAME","value":"taskflow-domain"},
      {"name":"REPO_NAME","value":"taskflow-repo"}
    ]
  }' \
  --service-role arn:aws:iam::${ACCOUNT_ID}:role/CodeBuildTaskflowRole

# Trigger a test build and follow logs
BUILD_ID=$(aws codebuild start-build \
  --project-name taskflow-api-build --query build.id --output text)

aws logs tail /aws/codebuild/taskflow-api-build --follow
```

> **Note on `privilegedMode: true`** — CodeBuild needs this flag to run `docker build` inside a container (Docker-in-Docker). Without it, the Docker daemon is unavailable and the build fails. Only enable this when you actually need Docker.

---

## 3. AWS CodePipeline — Orchestration

The conductor. Listens for every push to main on CodeCommit, triggers CodeBuild, enforces a human approval gate, then hands the artifact to CodeDeploy. All automated — except the approval you choose to keep human.

### The Four Stages

| Stage | Provider | What It Does |
|-------|----------|-------------|
| Source | CodeCommit | Detects push to main via EventBridge. Stores repo as S3 artifact. |
| Build | CodeBuild | Runs `buildspec.yml`, produces `imagedefinitions.json` + updated k8s manifests. |
| Approve | Manual | Sends SNS email. Pipeline pauses until you approve or reject. |
| Deploy | CodeDeploy | Runs lifecycle hooks, validates health, auto-rolls back on failure. |

```bash
# Create artifact bucket (versioning required by CodePipeline)
BUCKET=taskflow-pipeline-artifacts-$(aws sts get-caller-identity --query Account --output text)
aws s3 mb s3://$BUCKET --region ap-south-1
aws s3api put-bucket-versioning --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# Create SNS approval topic + subscribe your email
TOPIC_ARN=$(aws sns create-topic --name taskflow-approvals \
  --query TopicArn --output text)
aws sns subscribe --topic-arn $TOPIC_ARN \
  --protocol email --notification-endpoint you@email.com

# Create the pipeline
aws codepipeline create-pipeline \
  --cli-input-json file://pipeline/pipeline.json
```

### Approving the Manual Gate via CLI

```bash
# Get the approval token
TOKEN=$(aws codepipeline get-pipeline-state \
  --name taskflow-api-pipeline \
  --query "stageStates[?stageName=='Approve'].actionStates[0].latestExecution.token" \
  --output text)

# Approve
aws codepipeline put-approval-result \
  --pipeline-name taskflow-api-pipeline \
  --stage-name    Approve \
  --action-name   ManualApproval \
  --token         $TOKEN \
  --result        '{"status":"Approved","summary":"Tests passed, LGTM"}'

# Or reject (Deploy stage never runs)
aws codepipeline put-approval-result \
  --pipeline-name taskflow-api-pipeline \
  --stage-name    Approve \
  --action-name   ManualApproval \
  --token         $TOKEN \
  --result        '{"status":"Rejected","summary":"Performance regression found"}'
```

---

## 4. AWS CodeDeploy — Safe Deployment with Auto Rollback

Controlled deployment execution with lifecycle hook scripts, health validation, and automatic rollback — so a failed deployment never stays failed.

### The Four Lifecycle Hooks

CodeDeploy calls these shell scripts at specific points in the deployment. If any exits with a non-zero code, CodeDeploy marks the deployment as failed and rolls back automatically.

**1. BeforeInstall** (`before_install.sh`) — Verifies `kubectl` is installed and the EKS cluster is reachable before any manifests are applied. A fast early exit if the cluster is unhealthy.

**2. AfterInstall** (`after_install.sh`) — Runs `kubectl apply` on the namespace, ConfigMap, Deployment, and Service. The image tag has already been updated by CodeBuild.

**3. BeforeAllowTraffic** (`validate_service.sh`) — Waits for `kubectl rollout status` to confirm all pods are healthy. If it times out, automatically runs `kubectl rollout undo` and exits with code 1 — triggering CodeDeploy's rollback.

**4. AfterAllowTraffic** (`after_allow_traffic.sh`) — Traffic is live on the new version. Sends a success notification.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create CodeDeploy application for EKS
aws deploy create-application \
  --application-name taskflow-api \
  --compute-platform EKS

# Create deployment group with auto-rollback on failure
aws deploy create-deployment-group \
  --application-name       taskflow-api \
  --deployment-group-name  taskflow-api-eks-dg \
  --service-role-arn       arn:aws:iam::${ACCOUNT_ID}:role/CodeDeployTaskflowRole \
  --deployment-config-name CodeDeployDefault.EKSLinear10PercentEvery1Minutes \
  --auto-rollback-configuration '{"enabled":true,"events":["DEPLOYMENT_FAILURE"]}'

# Practice auto-rollback: break the validate script and push
echo "exit 1  # simulated failure" >> scripts/validate_service.sh
git add scripts/validate_service.sh
git commit -m "test: simulate deployment validation failure"
git push origin main
# CodeDeploy fails the deployment and rolls back to previous revision automatically
```

**Deployment Strategies available:**
- `EKSLinear10PercentEvery1Minutes` — shifts 10% of pods every minute. Slow and safe for production.
- `EKSCanary10Percent5Minutes` — 10% for 5 minutes, then 100%. Good middle ground.
- `EKSAllAtOnce` — replaces all at once. Fast, no incremental safety net.

---

## 5. Amazon EKS — Managed Kubernetes

AWS owns the control plane. You manage the nodes. The app runs as 2 pods across 2 Availability Zones with topology spread constraints, HPA autoscaling, and liveness/readiness probes.

### Cluster Setup

```bash
# Create cluster — 2 nodes, 2 AZs, managed node group (~15 mins)
eksctl create cluster \
  --name           taskflow-eks \
  --region         ap-south-1 \
  --version        1.32 \
  --nodegroup-name taskflow-ng \
  --node-type      t3.small \
  --nodes          2 \
  --nodes-min      2 \
  --nodes-max      4 \
  --managed        \
  --with-oidc      \
  --asg-access

# Confirm nodes are in different AZs
kubectl get nodes --show-labels | grep topology.kubernetes.io/zone

# Deploy the application
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa.yaml

# Verify
kubectl get all -n taskflow
kubectl get pods -n taskflow -o wide
```

### Least-Privilege RBAC

The CodeDeploy role needs to call `kubectl apply` — but it should never read secrets, list all nodes, or modify RBAC rules. We map the IAM role in `aws-auth` with only `system:authenticated`, then create a targeted ClusterRole.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: codedeploy-deployer
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs:     ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "services", "namespaces", "configmaps"]
  verbs:     ["get", "list", "watch", "create", "update", "patch"]
# Notably absent: secrets, nodes, clusterroles
```

> ⚠️ **Never use `system:masters` for CI/CD.** It is full cluster-admin with no restrictions. If a CodeDeploy role or build agent is ever compromised, an attacker with `system:masters` owns your entire cluster. Always use a scoped ClusterRole.

---

## 6. ArgoCD — GitOps Pull-Based Deployment

Pull-based GitOps. ArgoCD watches the `k8s/` directory in CodeCommit and continuously syncs the cluster to match. Change a YAML, push it — the cluster updates itself. No `kubectl` from CI needed.

### CodeDeploy vs ArgoCD — Two Philosophies

**CodeDeploy (Push-Based)**
- CI pipeline drives deployment
- Explicit approval gates
- Lifecycle hooks for custom logic
- Built-in canary/linear strategies
- Pipeline holds cluster credentials

**ArgoCD (Pull-Based / GitOps)**
- Cluster watches Git continuously
- Automatic drift correction
- No cluster credentials in CI
- Rollback = revert a git commit
- Git is single source of truth

These aren't mutually exclusive. Many teams use both: CodeDeploy for application releases (pipeline-driven) and ArgoCD for infrastructure changes (ConfigMaps, HPA settings, Kubernetes config).

### Install ArgoCD on EKS

```bash
# Install ArgoCD into its own namespace
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=Ready pods --all -n argocd --timeout=180s

# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Expose the UI
kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"LoadBalancer"}}'

# Install argocd CLI and login
ARGOCD_HOST=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

argocd login $ARGOCD_HOST --username admin --insecure \
  --password $(kubectl get secret argocd-initial-admin-secret \
    -n argocd -o jsonpath="{.data.password}" | base64 -d)
```

### The GitOps Workflow

Once the Application is created, the entire workflow is: **edit YAML → git push → ArgoCD syncs**. No pipeline step. No manual kubectl.

```bash
# Connect CodeCommit repo to ArgoCD
argocd repo add \
  https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/taskflow-api \
  --username YOUR_IAM_USER --password YOUR_CODECOMMIT_HTTPS_PASS

# Create the Application (auto-sync + self-heal enabled)
argocd app create taskflow-api \
  --repo           https://git-codecommit.ap-south-1.amazonaws.com/v1/repos/taskflow-api \
  --path           k8s \
  --dest-server    https://kubernetes.default.svc \
  --dest-namespace taskflow \
  --revision       main \
  --sync-policy    automated \
  --auto-prune \
  --self-heal

# === Practice: GitOps Update ===
# Change replica count → push → ArgoCD auto-applies in ~3 minutes
sed -i 's/replicas: 2/replicas: 3/' k8s/deployment.yaml
git add k8s/deployment.yaml && git commit -m "scale: 3 replicas" && git push origin main

# Force an immediate sync
argocd app sync taskflow-api

# === Practice: Drift Detection ===
# Manually scale to 1 — ArgoCD self-heals back to 3 within seconds
kubectl scale deployment taskflow-api -n taskflow --replicas=1
kubectl get pods -n taskflow --watch  # watch it snap back

# === Practice: Rollback ===
argocd app history taskflow-api
argocd app rollback taskflow-api <REVISION_ID>
```

---

## Putting It All Together

The first end-to-end run is always the most satisfying. You push one commit to CodeCommit, and over the next few minutes you watch: CodePipeline light up, CodeBuild run your tests, an image appear in ECR with a git SHA tag, an approval email hit your inbox, CodeDeploy lifecycle hooks fire one by one, and finally `kubectl get pods -n taskflow` showing the new version running.

ArgoCD adds a separate layer — watching the same repo and ensuring the cluster never drifts from what Git says. Try manually editing a replica count with `kubectl` and watch ArgoCD snap it back within seconds. That's self-healing infrastructure.

### What Each Service Does in This Pipeline

| Service | Role |
|---------|------|
| **CodeCommit** | Source of truth. Triggers the entire pipeline on every push to main. |
| **CodeArtifact** | Private PyPI proxy. Controlled, auditable, network-independent dependencies. |
| **CodeBuild** | Tests pass, image built, pushed to ECR with an immutable git-SHA tag. |
| **CodePipeline** | Orchestrates every stage with a human approval gate before deploy. |
| **CodeDeploy** | Lifecycle hooks, health validation, auto-rollback on failure. |
| **Amazon EKS** | Multi-AZ, autoscaling, least-privilege RBAC, self-healing pods. |
| **ArgoCD** | GitOps drift detection. Cluster always matches Git. Instant rollback. |

---

> ⚠️ **Cost Reminder:** EKS control plane costs ~$0.10/hr. Two t3.small nodes add ~$0.04/hr each. When done: `eksctl delete cluster --name taskflow-eks` and delete ECR images, CodePipeline, S3 bucket, and SNS topic.

---

*AWS DevOps Complete Lab · CodeCommit · CodeArtifact · CodeBuild · CodePipeline · CodeDeploy · EKS · ArgoCD*
