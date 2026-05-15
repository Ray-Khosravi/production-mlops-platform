# MLOps Project — Step-by-Step Roadmap

End-to-end guide: raw data on S3 → cleaning on EC2 → training with MLflow → FastAPI + HTML/CSS app → Docker → EKS via Terraform → Ingress with ALB → Route 53 custom domain → CI/CD with GitHub Actions + Argo CD.

---

## Table of Contents

- [Phase 0 — Prerequisites](#phase-0--prerequisites)
- [Phase 1 — Raw Data on S3](#phase-1--raw-data-on-s3)
- [Phase 2 — Data Cleaning EC2](#phase-2--data-cleaning-ec2)
- [Phase 3 — Training EC2 + MLflow](#phase-3--training-ec2--mlflow)
- [Phase 4 — FastAPI Backend](#phase-4--fastapi-backend)
- [Phase 5 — HTML/CSS Frontend](#phase-5--htmlcss-frontend)
- [Phase 6 — Dockerize Everything](#phase-6--dockerize-everything)
- [Phase 7 — Terraform: VPC, EKS, ECR](#phase-7--terraform-vpc-eks-ecr)
- [Phase 8 — Push Images to ECR](#phase-8--push-images-to-ecr)
- [Phase 9 — Kubernetes Manifests](#phase-9--kubernetes-manifests)
- [Phase 10 — ALB Ingress Controller + Ingress](#phase-10--alb-ingress-controller--ingress)
- [Phase 11 — Route 53 Custom Domain + ACM](#phase-11--route-53-custom-domain--acm)
- [Phase 12 — CI with GitHub Actions](#phase-12--ci-with-github-actions)
- [Phase 13 — CD with Argo CD](#phase-13--cd-with-argo-cd)
- [Execution Order & Cost Tips](#execution-order--cost-tips)

---

## Phase 0 — Prerequisites

### 0.1 Install tools on your local machine

| Tool | Purpose | Install command (macOS / Ubuntu) |
|------|---------|----------------------------------|
| AWS CLI v2 | Talk to AWS | `brew install awscli` / `apt install awscli` |
| Terraform ≥ 1.6 | Infrastructure as code | `brew install terraform` |
| kubectl | k8s CLI | `brew install kubectl` |
| eksctl | EKS helper | `brew install eksctl` |
| helm | k8s package manager | `brew install helm` |
| Docker Desktop | Build images | Download from docker.com |
| Python 3.11 | Dev | `brew install python@3.11` |
| Git | Version control | already on most systems |

### 0.2 Configure AWS

```bash
aws configure
# Enter Access Key ID, Secret Access Key, default region (e.g. us-east-1), output: json
```

Create an IAM user with `AdministratorAccess` (for learning; tighten with least-privilege policies later).

### 0.3 VS Code extensions

- **Remote - SSH** (for EC2 dev)
- **Python**
- **Docker**
- **Kubernetes**
- **HashiCorp Terraform**

### 0.4 Project folder layout

Create the skeleton up front — every phase fills in one folder:

```
cancer-detection/
├── terraform/                 # VPC, EKS, ECR, IAM, ACM, Route 53
├── data-pipeline/             # cleaning scripts (runs on EC2)
├── training/                  # training scripts + MLflow
├── backend/                   # FastAPI
├── frontend/                  # HTML/CSS + nginx
├── k8s/                       # deployment.yaml, service.yaml, ingress.yaml
├── .github/workflows/         # GitHub Actions CI
├── argocd/                    # Argo CD Application manifests
└── README.md
```

```bash
mkdir -p cancer-detection/{terraform,data-pipeline,training,backend,frontend,k8s,.github/workflows,argocd}
cd cancer-detection && git init
```

---

## Phase 1 — Raw Data on S3

### 1.1 Create the three buckets

Pick a unique suffix (e.g., your initials + date): `cancer-detection-raw-xyz0514`.

```bash
SUFFIX=xyz0514
aws s3 mb s3://cancer-detection-raw-$SUFFIX     --region us-east-1
aws s3 mb s3://cancer-detection-clean-$SUFFIX   --region us-east-1
aws s3 mb s3://cancer-detection-models-$SUFFIX  --region us-east-1
```

### 1.2 Enable versioning (so you can roll back bad cleaning runs)

```bash
for B in raw clean models; do
  aws s3api put-bucket-versioning \
    --bucket cancer-detection-$B-$SUFFIX \
    --versioning-configuration Status=Enabled
done
```

### 1.3 Block public access (security default)

```bash
for B in raw clean models; do
  aws s3api put-public-access-block \
    --bucket cancer-detection-$B-$SUFFIX \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
done
```

### 1.4 Upload raw images

```bash
aws s3 cp ./local-data s3://cancer-detection-raw-$SUFFIX/ --recursive
# OR, for resumable big uploads:
aws s3 sync ./local-data s3://cancer-detection-raw-$SUFFIX/
```

### 1.5 Verify

```bash
aws s3 ls s3://cancer-detection-raw-$SUFFIX/ --recursive --human-readable --summarize
```

---

## Phase 2 — Data Cleaning EC2

### 2.1 Create an EC2 key pair (if you don't have one)

```bash
aws ec2 create-key-pair --key-name cancer-key \
  --query 'KeyMaterial' --output text > ~/.ssh/cancer-key.pem
chmod 400 ~/.ssh/cancer-key.pem
```

### 2.2 Create an IAM role for the EC2 (so no AWS keys live on the box)

**Trust policy** (`ec2-trust.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

```bash
aws iam create-role --role-name DataEC2Role \
  --assume-role-policy-document file://ec2-trust.json
```

**S3 policy** (`s3-data-policy.json`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::cancer-detection-raw-xyz0514",
      "arn:aws:s3:::cancer-detection-raw-xyz0514/*",
      "arn:aws:s3:::cancer-detection-clean-xyz0514",
      "arn:aws:s3:::cancer-detection-clean-xyz0514/*"
    ]
  }]
}
```

```bash
aws iam put-role-policy --role-name DataEC2Role \
  --policy-name S3DataAccess \
  --policy-document file://s3-data-policy.json

aws iam create-instance-profile --instance-profile-name DataEC2Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name DataEC2Profile --role-name DataEC2Role
```

### 2.3 Launch the EC2 instance

- **Type**: `m6i.2xlarge` (8 vCPU, 32 GB RAM). Use `r6i.2xlarge` if memory-bound.
- **AMI**: Ubuntu 22.04 LTS
- **Storage**: 200 GB gp3 EBS
- **Security group**: allow SSH (port 22) **from your IP only**
- **IAM instance profile**: `DataEC2Profile`

Via console: EC2 → Launch instance → pick the options above.

Or via CLI:

```bash
# Get your IP for SG rule
MY_IP=$(curl -s ifconfig.me)

# Create SG
aws ec2 create-security-group --group-name data-ec2-sg \
  --description "Data cleaning EC2"
aws ec2 authorize-security-group-ingress --group-name data-ec2-sg \
  --protocol tcp --port 22 --cidr ${MY_IP}/32

# Launch
aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --instance-type m6i.2xlarge \
  --key-name cancer-key \
  --security-groups data-ec2-sg \
  --iam-instance-profile Name=DataEC2Profile \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=200,VolumeType=gp3}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=data-cleaning}]'
```

Grab the public IP from the console or:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=data-cleaning" \
  --query 'Reservations[].Instances[].PublicIpAddress' --output text
```

### 2.4 Connect from VS Code

Add to `~/.ssh/config`:

```
Host data-ec2
  HostName <PUBLIC_IP>
  User ubuntu
  IdentityFile ~/.ssh/cancer-key.pem
```

In VS Code: `Cmd/Ctrl+Shift+P` → **Remote-SSH: Connect to Host** → `data-ec2`. You're now editing files directly on the EC2 box.

### 2.5 Install Python environment

```bash
sudo apt update && sudo apt install -y python3-pip python3-venv awscli
python3 -m venv .venv && source .venv/bin/activate
pip install pandas numpy pillow opencv-python boto3 pyarrow tqdm
```

### 2.6 Cleaning script

`data-pipeline/clean.py`:

```python
import boto3, os
from pathlib import Path
from PIL import Image
from tqdm import tqdm

s3 = boto3.client("s3")
RAW_BUCKET = "cancer-detection-raw-xyz0514"
CLEAN_BUCKET = "cancer-detection-clean-xyz0514"

def list_keys(bucket, prefix=""):
    paginator = s3.get_paginator("list_objects_v2")
    for page in paginator.paginate(Bucket=bucket, Prefix=prefix):
        for obj in page.get("Contents", []):
            yield obj["Key"]

def clean_image(local_path, out_path, size=(224, 224)):
    img = Image.open(local_path).convert("RGB")
    img = img.resize(size)
    img.save(out_path, "JPEG", quality=90)

Path("/tmp/raw").mkdir(exist_ok=True)
Path("/tmp/clean").mkdir(exist_ok=True)

for key in tqdm(list(list_keys(RAW_BUCKET))):
    raw_local = f"/tmp/raw/{Path(key).name}"
    clean_local = f"/tmp/clean/{Path(key).name}"
    try:
        s3.download_file(RAW_BUCKET, key, raw_local)
        clean_image(raw_local, clean_local)
        s3.upload_file(clean_local, CLEAN_BUCKET, key)
    except Exception as e:
        print(f"skip {key}: {e}")
    finally:
        for p in (raw_local, clean_local):
            if os.path.exists(p):
                os.remove(p)
```

### 2.7 Run it in the background

```bash
nohup python clean.py > clean.log 2>&1 &
tail -f clean.log
```

### 2.8 Stop the EC2 to save money

When cleaning is done:

```bash
aws ec2 stop-instances --instance-ids <i-id>
```

(You can restart later if you need to re-clean. Don't terminate unless you're sure.)

---

## Phase 3 — Training EC2 + MLflow

### 3.1 Launch a GPU EC2

- **Type**: `g5.xlarge` (1× NVIDIA A10G, ~$1/hr) or `g4dn.xlarge` for cheaper
- **AMI**: **Deep Learning AMI (Ubuntu 22.04)** — comes with CUDA, PyTorch, conda preinstalled
- **Storage**: 300 GB gp3
- **Security group**: SSH (22) from your IP, plus **port 5000 (MLflow) from your IP only**
- **IAM role**: similar to Phase 2 but include the `models` bucket

> 💡 If your AWS account is new, you may need to request a vCPU quota increase for GPU instance types (Service Quotas console → EC2 → "Running On-Demand G and VT instances").

### 3.2 SSH + clone your repo

```bash
ssh -i ~/.ssh/cancer-key.pem ubuntu@<gpu-ip>
git clone https://github.com/<you>/cancer-detection.git
cd cancer-detection
```

### 3.3 Install MLflow + ML libraries

```bash
pip install mlflow boto3 scikit-learn torch torchvision
```

### 3.4 Start the MLflow tracking server

Backend store = SQLite, artifact store = S3:

```bash
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root s3://cancer-detection-models-xyz0514/mlflow-artifacts/ \
  --host 0.0.0.0 --port 5000 &
```

Visit `http://<gpu-ip>:5000` in your browser.

> 🔒 **Don't open port 5000 to the world.** Restrict it to your IP in the SG. For production, put it behind a VPN or an authenticated reverse proxy.

### 3.5 Sync cleaned data down to the EC2

```bash
aws s3 sync s3://cancer-detection-clean-xyz0514 ./data
# expects data/<class_name>/*.jpg structure (ImageFolder format)
```

### 3.6 Training script

`training/train.py`:

```python
import mlflow, mlflow.pytorch
import torch, torch.nn as nn
from torch.utils.data import DataLoader
from torchvision import datasets, transforms, models
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, f1_score
import numpy as np

mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("cancer-detection")

transform = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

train_ds = datasets.ImageFolder("./data/train", transform=transform)
val_ds   = datasets.ImageFolder("./data/val",   transform=transform)
train_loader = DataLoader(train_ds, batch_size=32, shuffle=True, num_workers=4)
val_loader   = DataLoader(val_ds,   batch_size=32, num_workers=4)

# ---------- Neural network run ----------
with mlflow.start_run(run_name="resnet50"):
    mlflow.log_params({"lr": 1e-4, "epochs": 10, "batch_size": 32, "arch": "resnet50"})

    device = "cuda"
    model = models.resnet50(weights="IMAGENET1K_V2")
    model.fc = nn.Linear(model.fc.in_features, len(train_ds.classes))
    model = model.to(device)
    opt = torch.optim.Adam(model.parameters(), lr=1e-4)
    loss_fn = nn.CrossEntropyLoss()

    for epoch in range(10):
        model.train()
        for x, y in train_loader:
            x, y = x.to(device), y.to(device)
            opt.zero_grad()
            loss = loss_fn(model(x), y)
            loss.backward()
            opt.step()
        # validation
        model.eval()
        preds, gts = [], []
        with torch.no_grad():
            for x, y in val_loader:
                p = model(x.to(device)).argmax(1).cpu()
                preds.extend(p.tolist()); gts.extend(y.tolist())
        acc = accuracy_score(gts, preds)
        f1  = f1_score(gts, preds, average="macro")
        mlflow.log_metrics({"val_acc": acc, "val_f1": f1}, step=epoch)

    mlflow.pytorch.log_model(model, "model")

# ---------- Random Forest run (on flattened features) ----------
def flatten(ds):
    X, y = [], []
    for img, label in ds:
        X.append(img.numpy().flatten()); y.append(label)
    return np.array(X), np.array(y)

with mlflow.start_run(run_name="random_forest"):
    mlflow.log_params({"n_estimators": 200, "max_depth": 20})
    X_tr, y_tr = flatten(train_ds)
    X_v,  y_v  = flatten(val_ds)
    rf = RandomForestClassifier(n_estimators=200, max_depth=20, n_jobs=-1)
    rf.fit(X_tr, y_tr)
    preds = rf.predict(X_v)
    mlflow.log_metrics({
        "val_acc": accuracy_score(y_v, preds),
        "val_f1":  f1_score(y_v, preds, average="macro"),
    })
    mlflow.sklearn.log_model(rf, "model")
```

### 3.7 Run training

```bash
python training/train.py
```

Watch metrics in real-time in the MLflow UI.

### 3.8 Register the best model

In the MLflow UI:

1. **Experiments** → `cancer-detection` → pick the run with the best `val_f1`.
2. Click **Register Model** → name it `cancer-detection-prod`.
3. Go to the **Models** tab → click your model → set the version to **Production** stage.

This is the version your FastAPI will load.

### 3.9 Stop the GPU EC2

```bash
aws ec2 stop-instances --instance-ids <i-id>
```

> 💸 A `g5.xlarge` left running 24/7 costs ~$24/day. Always stop it when not training.

---

## Phase 4 — FastAPI Backend

### 4.1 Files

`backend/app.py`:

```python
from fastapi import FastAPI, UploadFile, File
from fastapi.middleware.cors import CORSMiddleware
import torch, io, os
from PIL import Image
from torchvision import transforms
import mlflow.pytorch

app = FastAPI(title="Cancer Detection API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# In k8s, this will be the cluster-internal service DNS
MLFLOW_URI = os.getenv("MLFLOW_TRACKING_URI", "http://localhost:5000")
mlflow.set_tracking_uri(MLFLOW_URI)

model = mlflow.pytorch.load_model("models:/cancer-detection-prod/Production")
model.eval()
classes = ["benign", "malignant"]

tfm = transforms.Compose([
    transforms.Resize((224, 224)),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/predict")
async def predict(file: UploadFile = File(...)):
    img = Image.open(io.BytesIO(await file.read())).convert("RGB")
    x = tfm(img).unsqueeze(0)
    with torch.no_grad():
        probs = torch.softmax(model(x), dim=1)[0]
    idx = probs.argmax().item()
    return {
        "label": classes[idx],
        "confidence": float(probs[idx]),
        "probabilities": {c: float(probs[i]) for i, c in enumerate(classes)},
    }
```

`backend/requirements.txt`:

```
fastapi==0.115.0
uvicorn[standard]==0.32.0
python-multipart==0.0.12
torch==2.4.1
torchvision==0.19.1
mlflow==2.17.0
pillow==10.4.0
boto3==1.35.0
```

### 4.2 Run locally

```bash
cd backend
pip install -r requirements.txt

# For local testing, point at your GPU EC2's MLflow (or download the model artifact)
export MLFLOW_TRACKING_URI=http://<gpu-ip>:5000
export AWS_REGION=us-east-1

uvicorn app:app --reload --port 8000
```

Test: `curl -F "file=@sample.jpg" http://localhost:8000/predict`

---

## Phase 5 — HTML/CSS Frontend

### 5.1 Files

`frontend/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Cancer Detection</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div class="card">
    <h1>Welcome</h1>
    <p class="subtitle">This model is used for cancer detection. Upload a medical image and the model will tell you whether cancer is present.</p>

    <label class="upload">
      <input type="file" id="file" accept="image/*">
      <span>Choose image</span>
    </label>

    <img id="preview" alt="" />

    <button id="go" onclick="predict()">Analyze</button>

    <div id="result"></div>
  </div>

<script>
const fileInput = document.getElementById("file");
const preview   = document.getElementById("preview");

fileInput.addEventListener("change", () => {
  const f = fileInput.files[0];
  if (f) preview.src = URL.createObjectURL(f);
});

async function predict() {
  const f = fileInput.files[0];
  if (!f) { alert("Please select an image first."); return; }
  const fd = new FormData();
  fd.append("file", f);

  const btn = document.getElementById("go");
  btn.disabled = true; btn.textContent = "Analyzing…";

  try {
    const r = await fetch("/api/predict", { method: "POST", body: fd });
    if (!r.ok) throw new Error(`HTTP ${r.status}`);
    const j = await r.json();
    const cls = j.label === "malignant" ? "bad" : "good";
    document.getElementById("result").innerHTML =
      `<div class="result ${cls}">
         <strong>${j.label.toUpperCase()}</strong>
         <span>Confidence: ${(j.confidence * 100).toFixed(1)}%</span>
       </div>`;
  } catch (err) {
    document.getElementById("result").textContent = "Error: " + err.message;
  } finally {
    btn.disabled = false; btn.textContent = "Analyze";
  }
}
</script>
</body>
</html>
```

`frontend/style.css`:

```css
* { box-sizing: border-box; }
body {
  font-family: -apple-system, system-ui, "Segoe UI", Roboto, sans-serif;
  background: linear-gradient(135deg, #667eea, #764ba2);
  min-height: 100vh;
  margin: 0;
  display: grid; place-items: center;
  padding: 24px;
}
.card {
  background: #fff; border-radius: 16px; padding: 40px;
  max-width: 500px; width: 100%;
  box-shadow: 0 20px 60px rgba(0,0,0,0.3);
  text-align: center;
}
h1 { margin: 0 0 8px; color: #2d3748; }
.subtitle { color: #718096; margin-bottom: 28px; }
.upload {
  display: inline-block; padding: 12px 24px; border: 2px dashed #cbd5e0;
  border-radius: 8px; cursor: pointer; margin-bottom: 16px;
  transition: border-color 0.2s;
}
.upload:hover { border-color: #667eea; }
.upload input { display: none; }
#preview { max-width: 100%; max-height: 250px; border-radius: 8px; margin-bottom: 16px; }
#preview:not([src]) { display: none; }
button {
  background: #667eea; color: white; border: 0; padding: 12px 32px;
  border-radius: 8px; font-size: 16px; cursor: pointer; width: 100%;
}
button:disabled { opacity: 0.6; cursor: not-allowed; }
.result {
  margin-top: 20px; padding: 16px; border-radius: 8px;
  display: flex; flex-direction: column; gap: 4px;
}
.result.good { background: #f0fff4; color: #22543d; }
.result.bad  { background: #fff5f5; color: #742a2a; }
```

### 5.2 Run locally

```bash
cd frontend
python -m http.server 8080
```

For local testing with the backend, you'll need to either:
- Add CORS (already done in FastAPI), and change `fetch("/api/predict")` to `fetch("http://localhost:8000/predict")`, or
- Use a small nginx/Caddy proxy. The k8s setup in Phase 6 handles this for real.

---

## Phase 6 — Dockerize Everything

### 6.1 Backend Dockerfile

`backend/Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    libglib2.0-0 libsm6 libxext6 libxrender-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
EXPOSE 8000

# Healthcheck for orchestrators
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

`backend/.dockerignore`:

```
__pycache__
*.pyc
.venv
.env
.git
```

### 6.2 Frontend Dockerfile

`frontend/Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY index.html style.css /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

`frontend/nginx.conf` (proxies `/api/*` to the backend service in k8s):

```nginx
server {
  listen 80;

  location / {
    root /usr/share/nginx/html;
    index index.html;
  }

  location /api/ {
    proxy_pass http://backend-service:8000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    client_max_body_size 20M;
  }
}
```

### 6.3 Local test with docker-compose

`docker-compose.yml` (at the repo root):

```yaml
services:
  backend:
    build: ./backend
    environment:
      - MLFLOW_TRACKING_URI=http://<gpu-ip>:5000
      - AWS_REGION=us-east-1
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
    ports: ["8000:8000"]

  frontend:
    build: ./frontend
    ports: ["8080:80"]
    depends_on: [backend]
```

```bash
docker compose up --build
# Visit http://localhost:8080
```

Once this works end-to-end locally, you're ready to deploy.

---

## Phase 7 — Terraform: VPC, EKS, ECR

### 7.1 Terraform state bucket (one-time)

```bash
aws s3 mb s3://cancer-detection-tfstate-xyz0514 --region us-east-1
aws s3api put-bucket-versioning \
  --bucket cancer-detection-tfstate-xyz0514 \
  --versioning-configuration Status=Enabled
```

### 7.2 Main config

`terraform/main.tf`:

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket = "cancer-detection-tfstate-xyz0514"
    key    = "infra/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = "us-east-1"
}

# ----- VPC -----
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "cancer-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true   # cost-saving for dev; use one_per_az in prod
  enable_dns_hostnames = true

  # Required tags for the AWS Load Balancer Controller to discover subnets
  public_subnet_tags  = { "kubernetes.io/role/elb"          = 1 }
  private_subnet_tags = { "kubernetes.io/role/internal-elb" = 1 }
}

# ----- EKS -----
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "cancer-eks"
  cluster_version = "1.30"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets   # worker nodes in private subnets

  cluster_endpoint_public_access = true

  enable_cluster_creator_admin_permissions = true

  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.large"]
      min_size       = 2
      max_size       = 4
      desired_size   = 2
    }
  }
}

# ----- ECR repos -----
resource "aws_ecr_repository" "backend" {
  name                 = "cancer-backend"
  image_tag_mutability = "MUTABLE"
  image_scanning_configuration { scan_on_push = true }
}

resource "aws_ecr_repository" "frontend" {
  name                 = "cancer-frontend"
  image_tag_mutability = "MUTABLE"
  image_scanning_configuration { scan_on_push = true }
}

# ----- Outputs -----
output "cluster_name"     { value = module.eks.cluster_name }
output "ecr_backend_url"  { value = aws_ecr_repository.backend.repository_url }
output "ecr_frontend_url" { value = aws_ecr_repository.frontend.repository_url }
```

### 7.3 Apply

```bash
cd terraform
terraform init
terraform plan -out tf.plan
terraform apply tf.plan
```

Provisioning takes ~15 minutes (the EKS control plane is the slow part).

### 7.4 Configure kubectl

```bash
aws eks update-kubeconfig --name cancer-eks --region us-east-1
kubectl get nodes
# Should show 2 nodes Ready
```

---

## Phase 8 — Push Images to ECR

### 8.1 Login

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
REGION=us-east-1

aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin $ACCOUNT.dkr.ecr.$REGION.amazonaws.com
```

### 8.2 Build, tag, push

```bash
# Backend
docker build -t cancer-backend ./backend
docker tag cancer-backend:latest $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/cancer-backend:v1
docker push $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/cancer-backend:v1

# Frontend
docker build -t cancer-frontend ./frontend
docker tag cancer-frontend:latest $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/cancer-frontend:v1
docker push $ACCOUNT.dkr.ecr.$REGION.amazonaws.com/cancer-frontend:v1
```

### 8.3 (Important) Give EKS pods access to S3 / MLflow artifacts

The backend will load the model from S3 via MLflow. The cleanest way is **IRSA** (IAM Roles for Service Accounts):

```bash
eksctl utils associate-iam-oidc-provider --cluster cancer-eks --approve

# Policy that allows reading the models bucket
cat > backend-s3-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::cancer-detection-models-xyz0514",
      "arn:aws:s3:::cancer-detection-models-xyz0514/*"
    ]
  }]
}
EOF

aws iam create-policy --policy-name BackendS3Read \
  --policy-document file://backend-s3-policy.json

eksctl create iamserviceaccount \
  --cluster=cancer-eks \
  --namespace=default \
  --name=backend-sa \
  --attach-policy-arn=arn:aws:iam::$ACCOUNT:policy/BackendS3Read \
  --approve
```

You'll reference `serviceAccountName: backend-sa` in the backend Deployment.

---

## Phase 9 — Kubernetes Manifests

### 9.1 Backend

`k8s/backend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels: { app: backend }
spec:
  replicas: 2
  selector:
    matchLabels: { app: backend }
  template:
    metadata:
      labels: { app: backend }
    spec:
      serviceAccountName: backend-sa
      containers:
      - name: backend
        image: <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/cancer-backend:v1
        ports: [{ containerPort: 8000 }]
        env:
        - name: MLFLOW_TRACKING_URI
          value: "http://mlflow-service:5000"   # if you also deploy MLflow in-cluster
        - name: AWS_REGION
          value: "us-east-1"
        resources:
          requests: { cpu: "500m", memory: "1Gi" }
          limits:   { cpu: "2",    memory: "4Gi" }
        readinessProbe:
          httpGet: { path: /health, port: 8000 }
          initialDelaySeconds: 15
          periodSeconds: 10
        livenessProbe:
          httpGet: { path: /health, port: 8000 }
          initialDelaySeconds: 30
          periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector: { app: backend }
  ports: [{ port: 8000, targetPort: 8000 }]
  type: ClusterIP
```

> 💡 If you don't want to run MLflow in-cluster, set `MLFLOW_TRACKING_URI` to a `file:`-style URI and bake the model into the image, or download it from S3 directly at startup.

### 9.2 Frontend

`k8s/frontend-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels: { app: frontend }
spec:
  replicas: 2
  selector:
    matchLabels: { app: frontend }
  template:
    metadata:
      labels: { app: frontend }
    spec:
      containers:
      - name: frontend
        image: <ACCOUNT>.dkr.ecr.us-east-1.amazonaws.com/cancer-frontend:v1
        ports: [{ containerPort: 80 }]
        resources:
          requests: { cpu: "50m",  memory: "64Mi" }
          limits:   { cpu: "200m", memory: "128Mi" }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector: { app: frontend }
  ports: [{ port: 80, targetPort: 80 }]
  type: ClusterIP
```

### 9.3 Apply

```bash
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml
kubectl get pods -w
```

---

## Phase 10 — ALB Ingress Controller + Ingress

### 10.1 Install the AWS Load Balancer Controller

```bash
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

# IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Service account with the IAM policy
eksctl create iamserviceaccount \
  --cluster=cancer-eks \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::$ACCOUNT:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Helm install
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=cancer-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify
kubectl -n kube-system get deployment aws-load-balancer-controller
```

### 10.2 Ingress

`k8s/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cancer-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    # Set after Phase 11:
    # alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:<ACCOUNT>:certificate/<cert-id>
spec:
  rules:
  # While testing, omit host: rule so the ALB serves on its raw DNS name.
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port: { number: 8000 }
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port: { number: 80 }
```

```bash
kubectl apply -f k8s/ingress.yaml

# Wait ~3 min, then:
kubectl get ingress cancer-ingress
# ADDRESS column will show: k8s-default-cancering-xxxx.us-east-1.elb.amazonaws.com
```

Open that ALB DNS in your browser — you should see your app. 🎉

---

## Phase 11 — Route 53 Custom Domain + ACM

### 11.1 Buy / use a domain

In Route 53 → **Registered domains** → Register or Transfer. Or use a domain you already own (point its NS records at the Route 53 hosted zone).

### 11.2 Request an ACM certificate

In ACM (same region as your ALB, e.g. `us-east-1`):

1. Request a public certificate for `app.yourdomain.com` (and optionally `*.yourdomain.com`).
2. Validation method: **DNS**.
3. Route 53 offers a "Create records in Route 53" button — click it.
4. Wait ~5 min for status to become **Issued**.

### 11.3 Add cert ARN to the Ingress

Edit `k8s/ingress.yaml` and uncomment:

```yaml
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:<ACCOUNT>:certificate/<cert-id>
```

Add the host rule:

```yaml
spec:
  rules:
  - host: app.yourdomain.com
    http:
      paths: ...
```

```bash
kubectl apply -f k8s/ingress.yaml
```

### 11.4 Route 53 alias to ALB

In Route 53 → Hosted zone → Create record:

- Name: `app`
- Type: `A`
- **Alias**: Yes
- Route traffic to: Alias to Application and Classic Load Balancer → pick your ALB.
- Save.

After DNS propagates, `https://app.yourdomain.com` serves your app.

### 11.5 (Optional) Manage with Terraform

`terraform/dns.tf`:

```hcl
data "aws_route53_zone" "main" {
  name = "yourdomain.com."
}

resource "aws_acm_certificate" "app" {
  domain_name       = "app.yourdomain.com"
  validation_method = "DNS"
  lifecycle { create_before_destroy = true }
}

resource "aws_route53_record" "cert_validation" {
  for_each = {
    for d in aws_acm_certificate.app.domain_validation_options : d.domain_name => {
      name   = d.resource_record_name
      type   = d.resource_record_type
      record = d.resource_record_value
    }
  }
  zone_id = data.aws_route53_zone.main.zone_id
  name    = each.value.name
  type    = each.value.type
  records = [each.value.record]
  ttl     = 60
}

resource "aws_acm_certificate_validation" "app" {
  certificate_arn         = aws_acm_certificate.app.arn
  validation_record_fqdns = [for r in aws_route53_record.cert_validation : r.fqdn]
}

# The ALB record gets created after the Ingress controller provisions the ALB.
# Look it up by name/tag or use data sources after kubectl apply.
```

---

## Phase 12 — CI with GitHub Actions

### 12.1 IAM role for GitHub Actions (OIDC, no static keys)

Create an OIDC provider for GitHub (one-time per account):

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Trust policy `gha-trust.json` (replace `<OWNER>/<REPO>`):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::<ACCOUNT>:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"},
      "StringLike":   {"token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:*"}
    }
  }]
}
```

```bash
aws iam create-role --role-name github-actions-deploy \
  --assume-role-policy-document file://gha-trust.json

# Attach an ECR push policy
aws iam attach-role-policy --role-name github-actions-deploy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
```

### 12.2 Workflow

`.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [main]
    paths:
      - 'backend/**'
      - 'frontend/**'
      - 'k8s/**'
      - '.github/workflows/**'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # for OIDC
      contents: write   # for committing back manifest updates
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT>:role/github-actions-deploy
          aws-region: us-east-1

      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr

      - name: Build & push backend
        run: |
          IMG=${{ steps.ecr.outputs.registry }}/cancer-backend:${{ github.sha }}
          docker build -t $IMG ./backend
          docker push $IMG

      - name: Build & push frontend
        run: |
          IMG=${{ steps.ecr.outputs.registry }}/cancer-frontend:${{ github.sha }}
          docker build -t $IMG ./frontend
          docker push $IMG

      - name: Update k8s manifests with new image tags
        run: |
          REG=${{ steps.ecr.outputs.registry }}
          sed -i "s|image: .*cancer-backend:.*|image: ${REG}/cancer-backend:${{ github.sha }}|"   k8s/backend-deployment.yaml
          sed -i "s|image: .*cancer-frontend:.*|image: ${REG}/cancer-frontend:${{ github.sha }}|" k8s/frontend-deployment.yaml

      - name: Commit manifest updates
        run: |
          git config user.name  github-actions
          git config user.email actions@github.com
          git add k8s/
          git commit -m "ci: bump images to ${{ github.sha }}" || exit 0
          git push
```

Push to `main` → images build → manifests get bumped → Argo CD picks it up.

---

## Phase 13 — CD with Argo CD

### 13.1 Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 13.2 Access the UI

```bash
# port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d ; echo
```

Open `https://localhost:8080` → user `admin`, password from above.

### 13.3 Application manifest

`argocd/application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cancer-detection
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<OWNER>/<REPO>.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f argocd/application.yaml
```

In the Argo CD UI, you'll see the app, its sync status, and a topology graph of all the k8s resources.

### 13.4 Full loop

1. You push code → GitHub Actions builds new images, pushes to ECR, updates image tags in `k8s/*.yaml`, commits back to main.
2. Argo CD detects the manifest change and rolls it out to EKS.
3. Old pods are replaced by new ones with zero downtime (rolling update).

---

## Execution Order & Cost Tips

### Recommended execution order

1. **Phase 0–1**: AWS setup, S3 buckets, raw upload.
2. **Phase 2**: Cleaning EC2 end-to-end.
3. **Phase 3**: Training + MLflow + register best model.
4. **Phase 4–5**: Backend + frontend locally.
5. **Phase 6**: Dockerize, test with docker-compose.
6. **Phase 7**: `terraform apply` for VPC + EKS + ECR.
7. **Phase 8**: Push images.
8. **Phase 9–10**: k8s manifests + ALB Ingress (test with the ALB DNS first).
9. **Phase 11**: Route 53 + ACM (custom domain + HTTPS).
10. **Phase 12**: CI.
11. **Phase 13**: Argo CD.

### Cost-saving habits

| Resource | Idle cost | What to do |
|----------|-----------|------------|
| Data EC2 (m6i.2xlarge) | ~$0.38/hr | **Stop** when not cleaning |
| Training EC2 (g5.xlarge) | ~$1.00/hr | **Stop** when not training |
| EKS control plane | ~$2.40/day | `terraform destroy` between work sessions |
| NAT Gateway | ~$1.10/day + data | Comes down with `terraform destroy` |
| ALB | ~$0.55/day + LCU | Tied to Ingress; deletes with the cluster |
| S3 | ~$0.023/GB/mo | Keep |

A full `terraform destroy` brings the EKS-related charges back to zero. S3 buckets and stopped EC2 EBS volumes persist (cheap).

### Quick teardown

```bash
# k8s side first (deletes ALB and target groups cleanly)
kubectl delete -f k8s/

# Then infra
cd terraform
terraform destroy
```

---

## What runs where (cheat sheet)

| Concern | Location |
|---------|----------|
| Raw / cleaned / model artifacts | S3 (`raw`, `clean`, `models` buckets) |
| Data cleaning code | EC2 (Phase 2), VS Code Remote-SSH |
| Training + MLflow | GPU EC2 (Phase 3) |
| Experiment tracking | MLflow server, artifacts in S3 |
| Model registry | MLflow Models tab, `cancer-detection-prod` |
| Containers | ECR |
| Compute | EKS (worker nodes in private subnets) |
| Networking | VPC: public subnets (IGW) + private subnets (NAT GW) |
| External traffic | ALB created by AWS Load Balancer Controller (from Ingress) |
| Domain + TLS | Route 53 + ACM |
| CI | GitHub Actions (build, push, bump manifest tags) |
| CD | Argo CD (watches repo, syncs to EKS) |

---

**Happy shipping.** When you finish a phase, commit and tag it (`git tag phase-3-done`) so you can roll back if a later phase breaks something.
