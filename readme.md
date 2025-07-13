# 🚀 Kubernetes EKS Project: Deploy 2048 Game with Ingress & ALB

This project demonstrates how to:

* ✅ Install necessary tools: `kubectl`, `eksctl`, and `AWS CLI`
* ✅ Create an EKS Cluster on Fargate
* ✅ Deploy the 2048 Game App
* ✅ Set up Application Load Balancer (ALB) with Ingress Controller

---

## ✅ 1. Prerequisites Installation

### 🔧 Install `kubectl`

#### 📍 Linux / EC2:

```bash
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-04-19/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin
kubectl version --client
```
---

### 🔧 Install `eksctl`

#### 📍 Linux / EC2:

```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
---

### 🔧 Install AWS CLI

#### 📍 Linux / EC2:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

---

### 🔐 Configure AWS CLI (All OS):

```bash
aws configure
# Enter Access Key ID, Secret Key, Region (e.g., us-east-1), Output format (e.g., json)
```

---

## 🏗️ 2. Create EKS Cluster (Fargate)

```bash
eksctl create cluster \
  --name eks-game2048-cluster \
  --region us-east-1 \
  --fargate
```

---

## 📛 3. Create Fargate Profile

```bash
eksctl create fargateprofile \
  --cluster eks-game2048-cluster \
  --region us-east-1 \
  --name alb-sample-app \
  --namespace game-2048
```

---

## 🎮 4. Deploy 2048 Game App

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml
```

---

## 🔗 5. Configure IAM for OIDC

Check if OIDC is already associated:

```bash
aws iam list-open-id-connect-providers
```

If not associated:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster eks-game2048-cluster \
  --approve
```

---

## 🔐 6. Set Up IAM Policy for ALB Controller

### 📥 Download IAM Policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### 🛡️ Create IAM Policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### 👤 Create IAM Role & Service Account

Replace `<your-aws-account-id>` with your actual AWS account ID.

```bash
eksctl create iamserviceaccount \
  --cluster=eks-game2048-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

## ⚙️ 7. Install ALB Ingress Controller (via Helm)

### ➕ Add Helm Repo

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

### 📦 Install Controller

Replace `<your-vpc-id>` with your actual VPC ID.

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks-game2048-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<your-vpc-id>
```

### ✅ Verify Installation

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## 📂 8. `2048_full.yaml` Reference

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  replicas: 5
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
      - name: app-2048
        image: public.ecr.aws/l6m2t8p7/docker-2048:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-2048
            port:
              number: 80
```

---

## ✅ Final Output

Access the **2048 game** via the **DNS name** of the ALB once it's provisioned. Get the ALB URL using:

```bash
kubectl get ingress -n game-2048
```

---
