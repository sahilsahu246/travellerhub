# travellerhub — DevOps Setup Guide (v2, tailored to your fork)

> `sahilsahu246/travellerhub` (branch `devops-end-to-end`): the upstream `krishnaacharyaa/wanderlust` TypeScript MERN app. You author the DevOps layer provision all AWS infra.
>
> Region: **ap-south-1 (Mumbai)**. Terminal commands are single-line (zsh-safe). The app keeps its internal name `wanderlust` (DB name, etc.) except for some small UI changes — and the repo/brand is `travellerhub`.

---

## 0. What you have vs. what you're adding


Files 

```
backend/Dockerfile              backend/.dockerignore
frontend/Dockerfile             frontend/.dockerignore   frontend/nginx.conf
docker-compose.yml
Jenkinsfile
kubernetes/namespace.yaml
kubernetes/mongodb.yaml
kubernetes/redis.yaml
kubernetes/backend.yaml
kubernetes/frontend.yaml
GitOps/argocd-application.yaml
```

Architecture and stack are identical to the Mega-Project reference: GitHub → Jenkins CI (OWASP, SonarQube, Trivy) → Docker images → GitOps tag bump → ArgoCD CD → EKS, with Redis caching and Prometheus/Grafana monitoring.

---

## 1. FIRST: confirm three values, then place the files

All values below are now reconciled against your actual `package.json`, `.env.sample`, and `redis.ts` — no remaining guesses. Key facts baked into the files:

- **Backend port is `8080`** (`backend/.env.sample: PORT=8080`), not 5000. Note 8080 is also Jenkins' port — only a problem if you run the compose stack on the Jenkins host.
- **Project is ESM** (`"type":"module"`); `build` = `tsc`, `start` = `npm run build && node dist/server.js`. The container runs `node dist/server.js` directly (no rebuild at boot).
- **No committed lockfile** → Dockerfiles use `npm install`, not `npm ci`. The frontend's `prepare` hook (`cd .. && npm install`) is skipped with `--ignore-scripts`.
- **Env keys (verbatim from `.env.sample`):** `PORT`, `MONGODB_URI`, `REDIS_URL`, `FRONTEND_URL` (this is the CORS origin), `BACKEND_URL`, `NODE_ENV`, `JWT_SECRET`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, and `ACCESS_/REFRESH_` token + cookie vars. There is **no** GitHub OAuth or session secret.
- **Frontend** uses `VITE_API_PATH`, pointing at the backend on `8080`.

The only thing left for you to supply is real **Google OAuth credentials** (and a strong `JWT_SECRET`) — without valid Google creds, everything works except Google sign-in.

Then copy the provided files into place and commit:

```
git checkout devops-end-to-end
```

```
git add backend/Dockerfile backend/.dockerignore frontend/Dockerfile frontend/.dockerignore frontend/nginx.conf docker-compose.yml Jenkinsfile kubernetes GitOps
```

```
git commit -m "Add DevOps layer: docker, compose, Jenkins CI/CD, k8s manifests, ArgoCD"
```

```
git push -u origin devops-end-to-end
```

---

## 2. Validate locally with Docker Compose (before any cloud spend)

```
cp backend/.env.sample backend/.env && cp frontend/.env.sample frontend/.env
```

```
docker compose up -d --build
```

Seed Mongo. The seed file (`backend/data/sample_posts.json`) is on your machine, but `mongoimport` runs inside the container and can't see your filesystem — so you first copy the file in, then import it:

```
docker cp backend/data/sample_posts.json mongo:/sample_posts.json
```

```
docker compose exec mongo mongoimport --db wanderlust --collection posts --file /sample_posts.json --jsonArray
```

Open `http://localhost:5173`. Posts render → the containerization is correct. Tear down: `docker compose down`.

---

## 3. Provision AWS infrastructure manually (Console, no Terraform)

Everything by hand, region **ap-south-1 (Mumbai)**. Work through these in order — each step depends on the one before it.

---

### 3.1 VPC

**VPC → Your VPCs → Create VPC**

- **Resources to create:** VPC only (not "VPC and more" — build each piece manually so you understand it)
- **Name:** `travellerhub-vpc`
- **IPv4 CIDR:** `10.0.0.0/16`
- **Tenancy:** Default

Click **Create VPC**.

---

### 3.2 Internet Gateway

A VPC is fully isolated by default. Nothing can reach the internet without an IGW attached to it.

**VPC → Internet Gateways → Create internet gateway**

- **Name:** `travellerhub-igw`

Click **Create**, then immediately **Actions → Attach to VPC** → select `travellerhub-vpc` → **Attach**.

---

### 3.3 Subnets

You need subnets in **two different Availability Zones** — EKS requires this and will reject a cluster that only has subnets in one AZ. Create 4 subnets: 2 public (EC2 instances and EKS nodes go here) and 2 private (reserved for future use, best practice to have them).

**VPC → Subnets → Create subnet** — select `travellerhub-vpc` at the top, then use **Add new subnet** to add all 4 in one go:

| Name | AZ | CIDR |
|------|----|------|
| `travellerhub-public-1` | ap-south-1a | `10.0.1.0/24` |
| `travellerhub-public-2` | ap-south-1b | `10.0.2.0/24` |
| `travellerhub-private-1` | ap-south-1a | `10.0.3.0/24` |
| `travellerhub-private-2` | ap-south-1b | `10.0.4.0/24` |

Click **Create subnet**.

**Enable auto-assign public IP on both public subnets** — without this, EC2 instances launched into these subnets won't get a public IP automatically. For each of `travellerhub-public-1` and `travellerhub-public-2`:

Select subnet → **Actions → Edit subnet settings** → tick **Enable auto-assign public IPv4 address** → **Save**.

---

### 3.4 Route Table for public subnets

By default all subnets use the VPC's main route table, which has no internet route. Create a dedicated public route table and point it at the IGW.

**VPC → Route Tables → Create route table**

- **Name:** `travellerhub-public-rt`
- **VPC:** `travellerhub-vpc`

Click **Create**.

Add the internet route: select `travellerhub-public-rt` → **Routes tab → Edit routes → Add route**

- **Destination:** `0.0.0.0/0`
- **Target:** Internet Gateway → `travellerhub-igw`

Save.

Associate both public subnets: **Subnet associations tab → Edit subnet associations** → tick `travellerhub-public-1` and `travellerhub-public-2` → **Save**.

> Your route table should now show two routes: `10.0.0.0/16 → local` (internal VPC traffic) and `0.0.0.0/0 → travellerhub-igw` (internet traffic). Both Active.

---

### 3.5 Security Group

**EC2 → Security Groups → Create security group**

- **Name:** `travellerhub-sg`
- **Description:** `Security group for travellerhub DevOps stack`
- **VPC:** `travellerhub-vpc` (important — select your VPC, not the default)

Add inbound rules:

| Port | Source | Purpose |
|------|--------|---------|
| 22 | My IP | SSH from your machine |
| 22 | `10.0.0.0/16` | SSH between instances within the VPC (required for Jenkins master to connect to worker) |
| 8080 | Anywhere IPv4 | Jenkins UI |
| 9000 | Anywhere IPv4 | SonarQube |
| 9090 | Anywhere IPv4 | Prometheus |
| 3000 | Anywhere IPv4 | Grafana |
| 31000 | Anywhere IPv4 | Frontend NodePort |
| 31100 | Anywhere IPv4 | Backend NodePort |
| 30000–32767 | Anywhere IPv4 | Full NodePort range (ArgoCD + any other service) |

Leave outbound as default (all traffic allowed). Click **Create security group**.

---

### 3.6 Two EC2 instances

Launch both with these settings:

- **AMI:** Ubuntu Server 24.04 LTS (HVM), SSD Volume Type — `ami-0388e3ada3d9812da` (64-bit x86, ap-south-1)
- **Instance type:** `t3.large`
- **Storage:** 30 GB gp3
- **Network:** `travellerhub-vpc`
- **Subnet:** `travellerhub-public-1`
- **Auto-assign public IP:** Enable
- **Security group:** `travellerhub-sg`
- **Key pair:** create `travellerhub-kp`, download the `.pem` and keep it safe

Only the **Name** differs between the two:
- `travellerhub-master` — Jenkins master + tooling (AWS CLI, kubectl, Helm, ArgoCD CLI, SonarQube container)
- `travellerhub-worker` — Jenkins agent (Docker + Trivy)

Optionally attach `AmazonSSMManagedInstanceCore` as an instance profile and use **Session Manager** instead of SSH.

Once both instances show **2/2 checks passed** in the Status check column, click on `travellerhub-master` and copy its **Public IPv4 address**. Then SSH in from your machine:

```
chmod 400 ~/Downloads/travellerhub-kp.pem
```

```
ssh -i ~/Downloads/travellerhub-kp.pem ubuntu@<master-public-ip>
```

> Adjust the path to the `.pem` file if you saved it somewhere other than Downloads.

> **Ubuntu 24.04 differences vs 22.04:** Java ships as `openjdk-21-jre` (not 17), and the `apt-key` command is deprecated so Trivy uses a different GPG key method. All commands below account for this.

**On the master — Docker:**
```
sudo apt-get update -y && sudo apt-get install docker.io -y && sudo usermod -aG docker ubuntu && newgrp docker
```

**On the master — Jenkins** (Java 21 on 24.04, using the 2026 signing key):
```
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```
```
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```
```
sudo apt-get update -y && sudo apt-get install jenkins -y && sudo systemctl enable --now jenkins
```

Verify Jenkins is running:
```
sudo systemctl status jenkins
```

> **Note on the key:** The `jenkins.io-2023.key` expired on 2026-03-26. The correct key as of 2026 is `jenkins.io-2026.key`. If you ever see `NO_PUBKEY` or `signatures couldn't be verified` errors, this is why — always use the 2026 key URL above.

Get the first-run unlock password:
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Jenkins UI is now at `http://<master-public-ip>:8080`. Paste the password there to unlock it.

Add Jenkins to the docker group so the master can run docker commands:
```
sudo usermod -aG docker jenkins && sudo systemctl restart jenkins
```

Verify Jenkins is in the docker group:
```
groups jenkins
```

You should see `jenkins : jenkins docker` in the output.

**On the master — AWS CLI, kubectl, Helm:**
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && sudo apt install unzip -y && unzip awscliv2.zip && sudo ./aws/install
```
```
aws configure
```
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && chmod +x kubectl && sudo mv kubectl /usr/local/bin/kubectl
```
```
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**On the worker — Java and Docker** (Java 21 on 24.04):
```
sudo apt update -y && sudo apt install fontconfig openjdk-21-jre docker.io -y && sudo usermod -aG docker ubuntu && newgrp docker
```

**On the worker — AWS CLI:**
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && sudo apt install unzip -y && unzip awscliv2.zip && sudo ./aws/install
```

**On the worker — Trivy** (24.04-safe method, no `apt-key`):
```
sudo apt-get install wget apt-transport-https gnupg -y
```
```
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
```
```
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
```
```
sudo apt-get update -y && sudo apt-get install trivy -y
```

Attach an IAM role to the worker (learning: `AdministratorAccess`) via EC2 → Actions → Security → Modify IAM role.

### 3.7 IAM roles

You need **four** IAM roles total. Create all four before moving to EKS.

**Role 1 — EC2 role for the worker instance**

**IAM → Roles → Create role**
- **Trusted entity type:** AWS service
- **Use case:** EC2
- **Policy:** `AdministratorAccess`
- **Role name:** `travellerhub-ec2-role`
- Click **Create role**

Then attach it to the worker: **EC2 → Instances → travellerhub-worker → Actions → Security → Modify IAM role** → select `travellerhub-ec2-role` → **Update IAM role**.

---

**Role 2 — EKS Cluster role**

**IAM → Roles → Create role**
- **Trusted entity type:** AWS service
- **Use case:** EKS → select **EKS - Cluster**
- Policy `AmazonEKSClusterPolicy` is pre-attached automatically
- **Role name:** `travellerhub-eks-cluster-role`
- Click **Create role**

---

**Role 3 — EKS Node Group role**

**IAM → Roles → Create role**
- **Trusted entity type:** AWS service
- **Use case:** EC2
- Attach these four policies:
  - `AmazonEKSWorkerNodePolicy`
  - `AmazonEC2ContainerRegistryReadOnly`
  - `AmazonEKS_CNI_Policy`
  - `AmazonEBSCSIDriverPolicy`
- **Role name:** `travellerhub-eks-node-role`
- Click **Create role**

---

**Role 4 — EBS CSI Driver role (critical for MongoDB PVC)**

The EBS CSI controller runs as a Deployment in `kube-system` and needs its own IAM role to create EBS volumes. Without this role the CSI controller pods will crash with `CrashLoopBackOff` and MongoDB's PVC will never bind.

**IAM → Roles → Create role**
- **Trusted entity type:** AWS service
- **Use case:** EKS → select **EKS - Pod Identity**
- **Policy:** `AmazonEBSCSIDriverPolicy`
- **Role name:** `travellerhub-ebs-csi-role`
- Click **Create role**

Then attach it to the EBS CSI add-on after the cluster is created:
**EKS → Clusters → travellerhub → Add-ons → Amazon EBS CSI Driver → Edit → IAM role → select `travellerhub-ebs-csi-role` → Save changes**

This will restart the EBS CSI controller pods with proper AWS permissions.

### 3.8 EKS cluster

**EKS → Add cluster → Create**

**Step 1 — Configuration options:**
- Select **Custom configuration** (NOT Quick configuration with EKS Auto Mode — that removes your ability to manage nodes manually)
- **EKS Auto Mode:** toggle **off** (the toggle appears after selecting Custom configuration)

**Step 2 — Cluster configuration:**
- **Name:** `travellerhub`
- **Kubernetes version:** `1.35` (or latest available)
- **Cluster IAM role:** `travellerhub-eks-cluster-role`
- **Node IAM role:** leave empty (assigned during node group creation, not here)

> You will see two yellow warnings about Auto Mode missing policies — ignore them. They only apply to EKS Auto Mode which you've disabled.

**Kubernetes version settings:**
- **Upgrade policy:** Standard support ✅ (no extra cost)
- **Control plane scaling tier:** leave unchecked ✅ (paid feature, not needed)

**Cluster access:**
- **Bootstrap cluster administrator access:** Allow cluster administrator access ✅
- **Cluster authentication mode:** EKS API ✅ (modern Access Entries approach, no need to edit aws-auth ConfigMap)

**Remaining fields on this page:**
- **Envelope encryption:** leave off ✅
- **ARC Zonal shift:** Disabled ✅
- **Deletion protection:** leave off ✅ (you want to be able to delete easily when done)
- **Tags:** skip

Click **Next**.

**Step 3 — Networking:**
- **VPC:** `travellerhub-vpc`
- **Subnets:** select all four (`travellerhub-public-1`, `travellerhub-public-2`, `travellerhub-private-1`, `travellerhub-private-2`)
- **Security groups:** `travellerhub-sg`
- **Cluster IP address family:** IPv4 ✅
- **Kubernetes service IP address block:** leave off ✅
- **Hybrid nodes:** leave off ✅
- **Cluster endpoint access:** Public and private ✅ (worker node traffic to the API server stays within VPC — more secure than Public only)

Click **Next**.

**Step 4 — Observability:** leave all defaults, click **Next**.

**Step 5 — Add-ons:** keep all preselected add-ons and additionally find and enable **Amazon EBS CSI Driver** (required for MongoDB's PVC to provision an EBS volume). Click **Next**.

**Step 6 — Configure add-on settings:** leave all versions as default. For any add-on that asks for a Pod Identity IAM role (VPC CNI, EBS CSI Driver, External DNS) — **leave the role field empty**. They will fall back to the node group's IAM role automatically. Click **Next**.

**Step 7 — Review and create:** click **Create**.

Status will show **Creating** — this takes 10–15 minutes. Wait until it shows **Active** before creating the node group.

### 3.9 Managed node group

Wait for the cluster to show **Active** first, then:

**EKS → Clusters → travellerhub → Compute tab → Add node group**

**Node group configuration:**
- **Name:** `travellerhub-ng`
- **Node IAM role:** `travellerhub-eks-node-role`
- **Launch template:** none (use default)
- Click **Next**

**Node group compute configuration:**
- **AMI type:** Amazon Linux 2023 (AL2023) — default ✅
- **Capacity type:** On-Demand ✅
- **Instance type:** `t3.large`
- **Disk size:** 30 GB

**Node group scaling configuration:**
- **Minimum size:** 2
- **Maximum size:** 2
- **Desired size:** 2

Click **Next**

**Node group network configuration:**
- **Subnets:** select both public subnets (`travellerhub-public-1`, `travellerhub-public-2`)
- **Configure SSH access:** optional — if you want to SSH directly into nodes, select your key pair `travellerhub-kp`

Click **Next**

**Review and create:** click **Create**

Status will show **Creating** — wait until **Active** (5–10 minutes). Once active you should see 2 nodes.

### 3.10 Wire kubectl

Run these on the **`travellerhub-master` EC2 instance** (SSH in first). This is where all `kubectl` and `aws` CLI commands are run from — `kubectl` and AWS CLI are both installed there.

```
aws eks update-kubeconfig --region ap-south-1 --name travellerhub
```

**Before running kubectl, create an EKS Access Entry first** — otherwise you'll get a credentials error even though the kubeconfig was updated. This is because the IAM user needs to be explicitly granted access to the cluster.

**Step 1 — get your IAM ARN:**
```
aws sts get-caller-identity
```
Copy the `Arn` value from the output.

**Step 2 — create the access entry in the Console:**

**EKS → Clusters → travellerhub → Access tab → Create access entry**
- **IAM principal ARN:** paste the ARN from above
- **Type:** Standard
- Click **Next**
- **Policy name:** `AmazonEKSClusterAdminPolicy` → click **Add policy**
- Click **Next** → **Create**

**Step 3 — verify:**
```
kubectl get nodes
```

You should see 2 nodes in `Ready` state. If they show `NotReady` wait a minute and try again — they may still be initializing.

> **Node group disk size tip:** When creating the node group, the console may auto-fill 230 GiB instead of 30 GiB — always verify the disk size field before clicking Create. If it shows 230, clear the field and type `30` manually. A 230 GiB node group costs ~$23/month per node extra on EBS alone.

---

## 4. Connect worker to Jenkins as an agent

### 4.1 Generate SSH keypair on the master

Jenkins uses SSH to connect to the worker. Generate a dedicated keypair on the master (not your personal `travellerhub-kp`):

```
ssh-keygen -t ed25519 -C "jenkins-agent" -f ~/.ssh/jenkins-agent -N ""
```

This creates:
- `~/.ssh/jenkins-agent` — private key (stays on master, goes into Jenkins credentials)
- `~/.ssh/jenkins-agent.pub` — public key (goes on the worker)

Print the public key:
```
cat ~/.ssh/jenkins-agent.pub
```

### 4.2 Add the public key to the worker

SSH into the worker and run (paste the public key output from above):

```
mkdir -p ~/.ssh && echo "<paste-jenkins-agent.pub-content-here>" >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys
```

### 4.3 Add the private key to Jenkins credentials

Print the private key on the master:
```
cat ~/.ssh/jenkins-agent
```

In the Jenkins UI → **Manage Jenkins → Credentials → System → Global → Add Credentials**:
- **Kind:** SSH Username with private key
- **ID:** `jenkins-worker-ssh`
- **Username:** `ubuntu`
- **Private key:** Enter directly → paste the full contents of `~/.ssh/jenkins-agent`
- Click **Create**

### 4.4 Register the worker as a node

Jenkins UI → **Manage Jenkins → Nodes → New Node**:
- **Node name:** `worker-node`
- **Type:** Permanent Agent
- Click **Create**

Fill in the node config:
- **Number of executors:** `2`
- **Remote root directory:** `/home/ubuntu/jenkins`
- **Labels:** `worker-node` (must match the Jenkinsfile `agent { label 'worker-node' }`)
- **Launch method:** Launch agents via SSH
- **Host:** worker's **private IP** (e.g. `10.0.1.171` — use private IP since both instances are in the same VPC)
- **Credentials:** select `jenkins-worker-ssh`
- **Host Key Verification Strategy:** Non verifying
- Click **Save**

The node should show as connected within a few seconds. Click on `worker-node` in the nodes list and check the logs — you should see `Agent successfully connected and online`.

---

## 5. SonarQube setup

### 5.1 Install SonarQube on the master

```
docker run -itd --name SonarQube-Server -p 9000:9000 sonarqube:community
```

> Use `sonarqube:community` (not `sonarqube:lts-community` — the lts tag pulls an outdated version that shows a "no longer active" warning).

Verify it's running:
```
docker ps
```

Open `http://<master-public-ip>:9000`. Default login is `admin` / `admin` — it will ask you to set a new password immediately.

### 5.2 Create the project

**Projects → Create a local project**
- **Project display name:** `travellerhub`
- **Project key:** `travellerhub`
- **Main branch name:** `devops-end-to-end`
- Click **Next**

On the next screen select **Follows the instance's default** → click **Create project**.

On the Analysis Method page that appears → click **With Jenkins** → select **GitHub** as the DevOps platform. This page is just for reference — you already have a Jenkinsfile so you don't need to follow their steps. Skip to generating the token below.

### 5.3 Generate the SonarQube token

**Top right "A" icon → My Account → Security → Generate Tokens**

- **Name:** `jenkins-sonar-token`
- **Type:** Global Analysis Token
- **Expires in:** No expiration

Click **Generate** and **copy the token immediately** — it is only shown once.

---

## 6. Jenkins plugins, tools, and credentials

### 6.1 Fix the built-in node

**Manage Jenkins → Nodes → Built-In Node → Configure**
- Set **Number of executors** to `0`
- Click **Save**

This forces all builds to run on the worker node only, not on the master.

### 6.2 Install plugins

**Manage Jenkins → Plugins → Available plugins** — search and tick each, then install all at once:

- `SonarQube Scanner`
- `Docker`
- `Docker Pipeline`
- `OWASP Dependency-Check`
- `Pipeline Stage View`
- `Eclipse Temurin installer`

> The SonarQube plugin is listed as **SonarQube Scanner** in Jenkins (not "Community Build Scanner" — that's just what SonarQube calls it on their side).

### 6.3 Configure SonarQube Scanner tool

**Manage Jenkins → Tools → SonarQube Scanner installations → Add SonarQube Scanner**
- **Name:** `sonar-scanner`
- **Install automatically:** leave ticked
- Leave version as default

Click **Save**.

### 6.4 Configure SonarQube server

**Manage Jenkins → System → SonarQube servers → Add SonarQube**
- **Name:** `sonar`
- **Server URL:** `http://<master-private-ip>:9000` (use private IP `10.0.1.152` — stable across restarts)
- **Server authentication token:** select `sonar-token` (add it in credentials first — see 6.5)

Click **Save**.

### 6.5 Add credentials

**Manage Jenkins → Credentials → System → Global → Add Credentials** — add these one at a time:

**SonarQube token:**
- **Kind:** Secret text
- **Scope:** Global
- **Secret:** paste the token from section 5.3
- **ID:** `sonar-token`
- **Description:** `SonarQube token`
- Click **Create**

**Docker Hub:**
- **Kind:** Username with password
- **Scope:** Global
- **Username:** `sahilsahu246`
- **Password:** your Docker Hub password
- **ID:** `dockerHubCreds`
- Click **Create**

**GitHub PAT:**

First create the PAT on GitHub: **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)**
- **Note:** `jenkins-travellerhub`
- **Expiration:** No expiration
- **Scopes:** tick `repo` only (all sub-scopes are selected automatically)
- Click **Generate token** and copy it

Then in Jenkins:
- **Kind:** Username with password
- **Scope:** Global
- **Username:** `sahilsahu246`
- **Password:** paste the GitHub PAT
- **ID:** `github`
- **Description:** `GitHub PAT`
- Click **Create**



Open `Jenkinsfile` and set:
- `DOCKERHUB_USER` (already `sahilsahu246` — change if different).
- `VITE_API_PATH` → `http://<worker-or-node-public-ip>:31100` (the backend NodePort; baked into the frontend bundle at build time).
- Confirm credential IDs / tool names / Sonar server name match section 4.

Create a **Pipeline** job → *Pipeline script from SCM* → Git → your repo → credential `github` → branch `devops-end-to-end` → script path `Jenkinsfile`. Don't run it yet — set up ArgoCD first (section 6) so the first successful CI has something to sync into.

---

---

## 7. Fill in the Jenkinsfile placeholders

### 7.1 Get the node external IP

Run this on the **master** to get the public IPs of your EKS nodes:

```
kubectl get nodes -o wide
```

Note the `EXTERNAL-IP` of either node — you'll use it for `VITE_API_PATH` and the backend URLs. Either node's IP works since the NodePort service is accessible from any node.

### 7.2 Update the Jenkinsfile

Open `Jenkinsfile` in your repo on your Mac and update:

```
VITE_API_PATH   = 'http://<node-external-ip>:31100'
```

Replace `<node-external-ip>` with the actual external IP from above (e.g. `65.0.185.152`).

Confirm these are already set correctly:
- `DOCKERHUB_USER = 'sahilsahu246'` ✅
- `RUN_TESTS = 'false'` ✅

### 7.3 Update kubernetes/backend.yaml

Open `kubernetes/backend.yaml` and replace the two placeholder lines in the ConfigMap:

```
FRONTEND_URL: "http://<node-external-ip>:31000"
BACKEND_URL: "http://<node-external-ip>:31100"
```

- `FRONTEND_URL` — the URL the browser uses to load the frontend (NodePort 31000). This is also the CORS origin the backend allows.
- `BACKEND_URL` — the URL used for OAuth callbacks (NodePort 31100).

Also replace `<DOCKERHUB_USER>` in the image field with `sahilsahu246`.

### 7.4 Update kubernetes/frontend.yaml

Replace `<DOCKERHUB_USER>` in the image field with `sahilsahu246`.

### 7.5 Update GitOps/argocd-application.yaml

Confirm the repo URL and branch are correct:
- `repoURL: https://github.com/sahilsahu246/travellerhub.git`
- `targetRevision: devops-end-to-end`
- `path: kubernetes`

### 7.6 Commit and push

```
git add Jenkinsfile kubernetes/backend.yaml kubernetes/frontend.yaml
```

```
git commit -m "Set node IP, VITE_API_PATH, and Docker Hub image names"
```

```
git push origin devops-end-to-end
```

### 7.7 Create the Jenkins pipeline job

**Jenkins → New Item**
- **Name:** `travellerhub`
- **Type:** Pipeline
- Click **OK**

In the job config:
- **Pipeline definition:** Pipeline script from SCM
- **SCM:** Git
- **Repository URL:** `https://github.com/sahilsahu246/travellerhub.git`
- **Credentials:** select `github`
- **Branch:** `*/devops-end-to-end`
- **Script path:** `Jenkinsfile`

Click **Save**. Don't run it yet — set up ArgoCD first (section 8) so the first successful CI has something to sync into.

> **Important:** The node external IPs change if nodes are terminated and recreated. If that happens, update `VITE_API_PATH` in the Jenkinsfile and `FRONTEND_URL`/`BACKEND_URL` in `kubernetes/backend.yaml`, push the changes, and rebuild the frontend image (the old baked-in URL will be wrong).

### 7.8 First pipeline run — known fixes

The first pipeline run will likely hit two issues. Fix them before or as they appear:

**Fix 1 — SonarQube scanner path not resolving**

If the pipeline fails at the SonarQube stage with `/bin/sonar-scanner: not found`, the `${SCANNER_HOME}` variable isn't resolving. Update the SonarQube stage in the Jenkinsfile to use the `tool()` function instead:

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonar') {
            script {
                def scannerHome = tool 'sonar-scanner'
                sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=travellerhub -Dsonar.projectName=travellerhub -Dsonar.sources=backend,frontend"
            }
        }
    }
}
```

Commit and push:
```
git add Jenkinsfile && git commit -m "Fix: use tool() to resolve sonar-scanner path" && git push origin devops-end-to-end
```

**Fix 2 — OWASP Dependency-Check tool not installed**

If the pipeline fails with `No installation OWASP-DepCheck found`, the tool hasn't been configured in Jenkins yet.

First, install OWASP manually on the **worker**:
```
mkdir -p /home/ubuntu/jenkins/tools/dependency-check && cd /home/ubuntu/jenkins/tools/dependency-check
```
```
wget https://github.com/jeremylong/DependencyCheck/releases/download/v10.0.3/dependency-check-10.0.3-release.zip
```
```
unzip dependency-check-10.0.3-release.zip
```
```
chmod +x /home/ubuntu/jenkins/tools/dependency-check/dependency-check/bin/dependency-check.sh
```

Then in Jenkins:
**Manage Jenkins → Tools → Dependency-Check installations → Add Dependency-Check**
- **Name:** `OWASP-DepCheck`
- **Install automatically:** untick
- **Installation directory:** `/home/ubuntu/jenkins/tools/dependency-check/dependency-check`

> The field expects a **directory**, not the full `.sh` path — the plugin appends `bin/dependency-check.sh` automatically.

Click **Save**.

**Note on NVD API key warning:**

On the first run OWASP will show:
```
[WARN] An NVD API Key was not provided — it is highly recommended to use an NVD API key
```
This is just a warning, not an error. Without a key the NVD database download (~350k records) takes 5-15 minutes on the first run. After that it caches locally and subsequent runs are fast. You can get a free NVD API key at https://nvd.nist.gov/developers/request-an-api-key and add it as a Jenkins credential if you want faster scans.

**Fix 3 — Docker Hub push fails (wrong credentials or insufficient scope)**

If the pipeline fails at Push Images with `unauthorized: incorrect username or password` or `insufficient_scope: authorization failed`:

- Docker Hub now requires an **access token** instead of your account password for CLI logins. Generate one at **hub.docker.com → Account Settings → Personal access tokens → Generate new token** with **Read & Write** permissions.
- Update the Jenkins credential: **Manage Jenkins → Credentials → `dockerHubCreds` → Update** → paste the new token as the password.
- Double-check the **username** in both the credential and `DOCKERHUB_USER` in the Jenkinsfile match your actual Docker Hub username exactly (check hub.docker.com top-right after login).

**Fix 4 — EKS nodes can't pull images (`ImagePullBackOff`)**

By default Docker Hub repositories are private. EKS nodes can't pull private images without credentials. Fix by creating an `imagePullSecret` in the cluster:

```
kubectl create secret docker-registry dockerhub-creds --docker-server=https://index.docker.io/v1/ --docker-username=<your-dockerhub-username> --docker-password=<your-dockerhub-access-token> --docker-email=<your-email> -n wanderlust
```

Then add `imagePullSecrets` to the pod spec in both `kubernetes/backend.yaml` and `kubernetes/frontend.yaml`:

```yaml
spec:
  imagePullSecrets:
    - name: dockerhub-creds
  containers:
    - name: backend   # or frontend
```

Commit and push — ArgoCD will apply the change and pods will start pulling successfully.

> Alternatively, make the Docker Hub repositories **public** (hub.docker.com → repository → Settings → Visibility → Make Public) — simpler for learning but less secure.

**Fix 5 — Merge conflict markers pushed to Git**

If you see conflict markers in a manifest file:
```
<<<<<<< HEAD
          image: sahilsahu6246/wanderlust-frontend:8
=======
          image: sahilsahu6246/wanderlust-frontend:latest
>>>>>>> commit-hash (commit message)
```

Verify with:
```
cat kubernetes/frontend.yaml | grep -A2 "image:"
```

Fix by editing the file to keep only one version (keep `:latest` since CI rewrites the tag), removing the conflict markers entirely, then:
```
git add kubernetes/frontend.yaml kubernetes/backend.yaml
```
```
git commit -m "Fix: resolve merge conflict markers in manifests"
```
```
git push origin devops-end-to-end
```

---

## 8. ArgoCD (CD via GitOps)

### 8.1 Install ArgoCD on the cluster

Run on the **master**:

```
kubectl create namespace argocd
```

```
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> Use `--server-side --force-conflicts` — the official ArgoCD docs now require this because some ArgoCD CRDs exceed the 262KB annotation size limit imposed by client-side `kubectl apply`.

Wait for all pods to come up:

```
kubectl get pods -n argocd
```

All 7 pods should show `Running` before proceeding.

### 8.2 Expose the ArgoCD UI

The correct approach is NodePort — but it requires opening the NodePort range on the **EKS node security group** first (not `travellerhub-sg`). EKS nodes use their own auto-created security group (`eks-cluster-sg-travellerhub-...`) which by default blocks external traffic on NodePorts.

**Step 1 — Open NodePort range on the EKS node security group:**

**EC2 → Security Groups → `eks-cluster-sg-travellerhub-1128442922` → Inbound rules → Edit → Add rule:**
- **Type:** Custom TCP
- **Port range:** `30000-32767`
- **Source:** `0.0.0.0/0`

Save. This one rule fixes NodePort access for ArgoCD, the app frontend, and the app backend — you only need to do it once.

**Step 2 — Expose ArgoCD as NodePort:**

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

**Step 3 — Get the assigned NodePort:**

```
kubectl get svc argocd-server -n argocd
```

Note the port after `443:` in the PORT(S) column (e.g. `443:31251/TCP` → NodePort is `31251`).

### 8.3 Get the admin password

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d ; echo
```

### 8.4 Log into ArgoCD UI

Open in browser using a node's public IP and the NodePort (accept the self-signed certificate warning):

```
https://<node-public-ip>:<argocd-nodeport>
```

Get a node's public IP: `kubectl get nodes -o wide` → use either node's `EXTERNAL-IP`.

- **Username:** `admin`
- **Password:** from step 8.3

Change the password after first login: **User Info → Update Password**.

> **Note on LoadBalancer vs NodePort:** We initially tried LoadBalancer for ArgoCD (and it worked), but NodePort is simpler, free, and now works after opening the EKS node security group. If you previously created a LoadBalancer service for ArgoCD, patching it back to NodePort automatically deprovisions the ELB — check **EC2 → Load Balancers** to confirm it's gone.

### 8.5 Connect your GitHub repo

**Settings → Repositories → Connect Repo**
- **Connection method:** HTTPS
- **Repository URL:** `https://github.com/sahilsahu246/travellerhub.git`
- **Username:** `sahilsahu246`
- **Password:** your GitHub PAT (credential `github`)
- Click **Connect**

Status should show **Successful**.

### 8.6 Apply the ArgoCD Application

This tells ArgoCD to watch your `kubernetes/` folder on the `devops-end-to-end` branch and keep the cluster in sync with it. Run on the master:

```
kubectl apply -f GitOps/argocd-application.yaml
```

> This requires the repo to be cloned on the master. If it isn't, clone it first:
> ```
> git clone https://github.com/sahilsahu246/travellerhub.git && cd travellerhub && git checkout devops-end-to-end
> ```

ArgoCD will now appear in the UI showing the `travellerhub` application. It will be **OutOfSync** initially since no images have been built yet — that's expected. The first Jenkins pipeline run will push images and update the tags, triggering ArgoCD to sync and deploy everything.

### 8.7 Fix MongoDB PVC StorageClass

After ArgoCD syncs you'll notice MongoDB stays in **Pending** because the PVC has no StorageClass set. The default `gp2` StorageClass uses the old in-tree provisioner which doesn't work properly on modern EKS. You need to create a `gp3-csi` StorageClass that uses the EBS CSI driver.

**Step 1 — Create the StorageClass on the master:**

```
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-csi
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
EOF
```

**Step 2 — Update `kubernetes/mongodb.yaml`** on your Mac to use `gp3-csi`:

```yaml
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-csi
  resources:
    requests:
      storage: 5Gi
```

**Step 3 — Commit and push:**

```
git add kubernetes/mongodb.yaml
```

```
git commit -m "Fix: use gp3-csi StorageClass for mongo PVC"
```

```
git push origin devops-end-to-end
```

**Step 4 — Delete the stuck PVC** so ArgoCD recreates it with the new StorageClass (PVC StorageClass is immutable — it can't be patched, only recreated):

```
kubectl delete pvc mongo-pvc -n wanderlust
```

**Step 5 — Force ArgoCD to sync:**

```
kubectl annotate app travellerhub -n argocd argocd.argoproj.io/refresh=hard
```

**Step 6 — Delete the old MongoDB pod** to trigger scheduling against the new PVC (`WaitForFirstConsumer` only provisions once a pod is actually scheduled):

```
kubectl delete pod -n wanderlust -l app=mongodb
```

Watch the PVC bind:
```
kubectl get pvc -n wanderlust -w
```

Once bound, MongoDB pod will start. Verify:
```
kubectl get all -n wanderlust
```

MongoDB and Redis should show `1/1 Running`. Backend and frontend will show `ImagePullBackOff` until the Jenkins pipeline builds and pushes the images — that's expected at this stage.

### 8.8 Access the app

Once all pods are Running and ArgoCD shows Healthy, open the app in your browser using a node's public IP and the frontend NodePort:

```
http://<node-public-ip>:31000
```

Get a node's public IP: `kubectl get nodes -o wide` → use either node's `EXTERNAL-IP`.

> The NodePort works because you opened `30000-32767` on the EKS node security group in step 8.2. The same rule covers the backend NodePort (31100) and ArgoCD.

### 8.9 Expose services as LoadBalancer (optional alternative)

If you want a stable DNS hostname instead of a raw IP, change the Service type in the manifests:

In `kubernetes/frontend.yaml` and `kubernetes/backend.yaml`, change:
```yaml
type: NodePort
```
to:
```yaml
type: LoadBalancer
```
And remove the `nodePort:` field. Commit and push — ArgoCD will provision ELBs automatically. The ELB DNS hostname is stable across node changes.

> LoadBalancer costs ~$0.025/hr per ELB in ap-south-1. For learning, NodePort is free and works fine.

### 8.10 Seed MongoDB after first sync

Once the pipeline runs and MongoDB pod is running, seed the database. The seed file is in the cloned repo on the master at `~/travellerhub/backend/data/sample_posts.json`.

**Step 1 — copy the file into the MongoDB pod:**
```
kubectl cp ~/travellerhub/backend/data/sample_posts.json wanderlust/$(kubectl get pod -n wanderlust -l app=mongodb -o jsonpath='{.items[0].metadata.name}'):/sample_posts.json
```

**Step 2 — import it:**
```
kubectl exec -it -n wanderlust deploy/mongodb -- mongoimport --db wanderlust --collection posts --file /sample_posts.json --jsonArray
```

You should see `N document(s) imported successfully`. Open the app and posts will now appear.

> **If posts still don't load after seeding:** Redis may have cached empty results from before the seed. Flush the cache and the backend will query MongoDB fresh:
> ```
> kubectl exec -it -n wanderlust deploy/redis -- redis-cli FLUSHALL
> ```
> Refresh the browser — posts should now load.



---

## 9. Monitoring (Helm: Prometheus + Grafana)

`kube-prometheus-stack` installs both Prometheus and Grafana in one Helm chart and pre-wires them together. It also includes Alertmanager, kube-state-metrics, and node-exporter automatically.

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
```

```
kubectl create namespace prometheus
```

```
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```

Wait for all pods to come up:

```
kubectl get pods -n prometheus
```

**Expose Grafana as NodePort:**

```
kubectl patch svc stable-grafana -n prometheus -p '{"spec": {"type": "NodePort"}}'
```

```
kubectl get svc stable-grafana -n prometheus
```

Note the NodePort (e.g. `80:30543/TCP` → NodePort is `30543`).

**Expose Prometheus as NodePort:**

```
kubectl patch svc stable-kube-prometheus-sta-prometheus -n prometheus -p '{"spec": {"type": "NodePort"}}'
```

```
kubectl get svc stable-kube-prometheus-sta-prometheus -n prometheus
```

Note the NodePort after `9090:` (e.g. `9090:32577/TCP` → NodePort is `32577`).

**Get Grafana admin password:**

```
kubectl get secret --namespace prometheus stable-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

**Access:**
- **Grafana:** `http://<node-public-ip>:<grafana-nodeport>` — login with `admin` / password above
- **Prometheus:** `http://<node-public-ip>:<prometheus-nodeport>`

> Both NodePorts are accessible because you already opened `30000-32767` on the EKS node security group in section 8.2.

In Grafana go to **Dashboards → Browse** — pre-built Kubernetes dashboards are already configured. **Kubernetes / Compute Resources / Cluster** shows CPU and memory usage across the entire cluster.

---

## 10. End-to-end test

Push a small visible frontend change to `devops-end-to-end` → Jenkins runs (install/test → Sonar → OWASP → Trivy → build → push → bump tags in `kubernetes/` → git push) → ArgoCD shows OutOfSync→Synced → the new build is live at the frontend NodePort → Grafana shows pod metrics.

---

## 11. Cleanup (stop the meter)

```
helm uninstall stable -n prometheus && kubectl delete namespace wanderlust argocd prometheus
```
Then in the Console, delete in order: node group → EKS cluster → both EC2 instances → security group → IAM roles/user → leftover EBS volumes / Elastic IPs.

---

## 12. The known wiring gotcha (worth understanding, not just copying)

Vite bakes `VITE_API_PATH` into the JS bundle at **build time**, so the browser's backend URL is fixed when the image is built — you can't change it with a runtime env var. That's why:
- the **frontend Dockerfile** takes `VITE_API_PATH` as a `--build-arg`,
- the **Jenkinsfile** passes the backend's public NodePort URL at build,
- the **backend** must allow that frontend origin via `FRONTEND_URL`.

The cleaner production fix is an **Ingress (AWS ALB)** that serves the frontend at `/` and proxies `/api` to the backend on the **same origin** — which removes the baked-URL problem and CORS entirely. Get the NodePort version working first, then that's your natural next upgrade.

### Quick reference: confirm-before-run checklist
- [ ] Real `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET` and a strong `JWT_SECRET` set in `kubernetes/backend.yaml` (Secret).
- [ ] `<DOCKERHUB_USER>` replaced in both k8s manifests + Jenkinsfile.
- [ ] `<NODE_PUBLIC_IP>` replaced in `backend.yaml` (`FRONTEND_URL` + `BACKEND_URL`) + Jenkinsfile (`VITE_API_PATH`).
- [ ] Jenkins credential IDs/tool names/Sonar server match sections 4–5.
- [ ] ArgoCD repo connected; `argocd-application.yaml` applied.