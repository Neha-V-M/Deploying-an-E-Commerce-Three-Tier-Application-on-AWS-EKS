# Deploy an E-Commerce Three-Tier Application on AWS EKS

A hands-on DevOps project that deploys the **Robot Shop** e-commerce microservices application on **Amazon EKS** (Elastic Kubernetes Service), exposes it to the internet through an **AWS Application Load Balancer**, and persists data with the **EBS CSI driver**.

Everything is provisioned from the command line with `eksctl`, `kubectl`, `helm` and the AWS CLI.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Tech Stack](#3-tech-stack)
4. [Repository Structure](#4-repository-structure)
5. [Prerequisites](#5-prerequisites)
6. [Step-by-Step Deployment](#6-step-by-step-deployment)
7. [Verification](#7-verification)
8. [Troubleshooting (Issues I Hit and Fixed)](#8-troubleshooting-issues-i-hit-and-fixed)
9. [Cost Management](#9-cost-management)
10. [Cleanup](#10-cleanup)
11. [Key Learnings](#11-key-learnings)
12. [Future Improvements](#12-future-improvements)

---

## 1. Project Overview

| Item | Value |
|---|---|
| Application | Robot Shop (microservices e-commerce demo) |
| Platform | Amazon EKS (Kubernetes 1.34) |
| Region | `us-east-1` |
| Cluster name | `demo-cluster-three-tier` |
| Namespace | `robot-shop` |
| Packaging | Helm chart |
| Ingress | AWS Load Balancer Controller → internet-facing ALB |
| Storage | Amazon EBS via the EBS CSI driver |
| Provisioning | `eksctl` (CloudFormation under the hood) |

**Goals**

- Provision a production-style EKS cluster with managed node groups.
- Configure IAM Roles for Service Accounts (IRSA) using an OIDC provider.
- Install the AWS Load Balancer Controller and expose the app through an ALB.
- Enable persistent volumes with the EBS CSI driver.
- Deploy a multi-service application with Helm and verify it end to end.

---

## 2. Architecture

The application follows a classic **three-tier** design:

| Tier | Components | Role |
|---|---|---|
| **Presentation** | `web` | Nginx front end serving the UI and proxying API calls |
| **Application** | `catalogue`, `user`, `cart`, `shipping`, `payment`, `ratings`, `dispatch` | Business logic microservices |
| **Data** | `mongodb`, `mysql`, `redis`, `rabbitmq` | Databases, cache and message queue |

```
                         Internet
                             |
                             v
                +---------------------------+
                |  AWS Application Load     |
                |  Balancer (internet-facing)|
                +-------------+-------------+
                              |  (created by Ingress via
                              |   AWS Load Balancer Controller)
                              v
        +---------------------------------------------------+
        |              Amazon EKS cluster (us-east-1)       |
        |                 namespace: robot-shop             |
        |                                                   |
        |   [ Presentation ]                                |
        |        web  (port 8080)                           |
        |          |                                        |
        |   [ Application ]                                 |
        |   catalogue | user | cart | shipping | payment     |
        |   ratings | dispatch                              |
        |          |                                        |
        |   [ Data ]                                        |
        |   mongodb | mysql | redis | rabbitmq              |
        |                                                   |
        |   EBS CSI driver --> Amazon EBS volumes           |
        +---------------------------------------------------+
              2 managed worker nodes across 2 Availability Zones
```

---

## 3. Tech Stack

- **AWS:** EKS, EC2, VPC, IAM, Elastic Load Balancing (ALB), EBS, CloudFormation
- **Kubernetes tooling:** `kubectl`, `helm`, `eksctl`
- **Controllers and add-ons:** AWS Load Balancer Controller, EBS CSI driver, CoreDNS, kube-proxy, VPC CNI, metrics-server
- **OS / shell:** Ubuntu on WSL2

---

## 4. Repository Structure

```
three-tier-architecture-demo/
├── cluster.yaml                 # eksctl cluster configuration
└── EKS/
    ├── 01-prerequisites.md      # Tools and AWS CLI setup
    ├── 02-eks-cluster-setup.md  # Cluster creation
    ├── 03-oidc-IAM.md           # OIDC provider and IAM setup
    ├── 04-alb-configuration.md  # AWS Load Balancer Controller
    ├── 05-ebs-csi-driver.md     # EBS CSI driver
    └── helm/
        ├── Chart.yaml
        ├── values.yaml
        ├── ingress.yaml         # ALB Ingress (applied separately)
        ├── README.md
        └── templates/           # Kubernetes manifests for all services
```

---

## 5. Prerequisites

- An AWS account with an IAM user that has permissions for EKS, EC2, IAM, CloudFormation and ELB
- Installed and configured tools:

| Tool | Check |
|---|---|
| AWS CLI v2 | `aws --version` |
| `eksctl` | `eksctl version` |
| `kubectl` | `kubectl version --client` |
| `helm` (v3) | `helm version` |

- AWS CLI configured:

```bash
aws configure
aws sts get-caller-identity
```

> Replace `<ACCOUNT_ID>` and `<VPC_ID>` in the commands below with your own values.

---

## 6. Step-by-Step Deployment

### Step 1: Create the EKS cluster

**Check which instance types your account may launch.** Accounts on the AWS free plan are limited to free-tier-eligible instance types. If you use an ineligible one, the node group never comes up and `eksctl` times out.

```bash
aws ec2 describe-instance-types --region us-east-1 \
  --filters Name=free-tier-eligible,Values=true \
  --query "InstanceTypes[*].InstanceType" --output text
```

Pick a type with at least 2 vCPU and 4 GB RAM (for example `m7i-flex.large`). Tiny types such as `t3.micro` cannot hold the EKS system pods plus the application.

**`cluster.yaml`**

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: demo-cluster-three-tier
  region: us-east-1
managedNodeGroups:
  - name: ng-1
    instanceTypes: ["m7i-flex.large", "c7i-flex.large"]
    desiredCapacity: 2
    minSize: 1
    maxSize: 2
    volumeSize: 20
```

```bash
eksctl create cluster -f cluster.yaml
```

This takes about 15 to 20 minutes. It creates the VPC, subnets, the EKS control plane, the managed node group and the default add-ons (CoreDNS, kube-proxy, VPC CNI, metrics-server).

```bash
kubectl get nodes
```

### Step 2: Associate an IAM OIDC provider

IRSA lets Kubernetes service accounts assume IAM roles. It requires an OIDC provider for the cluster.

```bash
export cluster_name=demo-cluster-three-tier

oidc_id=$(aws eks describe-cluster --name $cluster_name \
  --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

# Is a provider already associated? (empty output = no)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

### Step 3: Install the AWS Load Balancer Controller

**3.1 Create the IAM policy**

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

> If this returns `EntityAlreadyExists`, the policy was created earlier. IAM policies are account-wide and survive cluster deletion, so you can reuse it.

**3.2 Create the IAM role and Kubernetes service account**

```bash
eksctl create iamserviceaccount \
  --cluster=demo-cluster-three-tier \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

**3.3 Install the controller with Helm**

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster-three-tier \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<VPC_ID>
```

Get the VPC ID with:

```bash
aws eks describe-cluster --name demo-cluster-three-tier \
  --query "cluster.resourcesVpcConfig.vpcId" --output text
```

**3.4 Verify**

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Expected: `READY 2/2` (two controller replicas, which is the chart default).

### Step 4: Install the EBS CSI driver

```bash
# IAM role for the driver (role only; the add-on creates the service account)
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster demo-cluster-three-tier \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

# Install the add-on
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster demo-cluster-three-tier \
  --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

### Step 5: Deploy the application with Helm

```bash
cd EKS/helm
kubectl create namespace robot-shop
helm install robot-shop --namespace robot-shop .
```

> Helm 3 takes the release name as a positional argument. The old `--name` flag from Helm 2 no longer exists.

```bash
kubectl get pods -n robot-shop
```

All 12 pods should reach `Running`: `web`, `cart`, `catalogue`, `user`, `payment`, `shipping`, `ratings`, `dispatch`, `mongodb`, `mysql`, `redis`, `rabbitmq`.

### Step 6: Expose the application with an Ingress (ALB)

`ingress.yaml` sits next to `Chart.yaml`, outside `templates/`, so `helm install` does **not** apply it. It must be applied separately:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: robot-shop
  name: robot-shop
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress -n robot-shop -w
```

After 2 to 3 minutes the `ADDRESS` column shows the ALB DNS name, for example `k8s-robotsho-robotsho-xxxx.us-east-1.elb.amazonaws.com`.

| Annotation | Purpose |
|---|---|
| `scheme: internet-facing` | Makes the ALB reachable from the public internet |
| `target-type: ip` | ALB sends traffic straight to pod IPs |
| `ingress.class: alb` | Tells the AWS Load Balancer Controller to handle this Ingress |

---

## 7. Verification

```bash
kubectl get nodes
kubectl get pods -A
kubectl get svc -n robot-shop
kubectl get ingress -n robot-shop
```

Open the ALB DNS name from the Ingress in a browser. The Robot Shop storefront should load, and you should be able to browse products, register or log in, add items to the cart and check out.

> A `502` or `503` in the first minute is normal while the ALB targets pass their health checks.

---

## 8. Troubleshooting (Issues I Hit and Fixed)

| # | Problem | Cause | Fix |
|---|---|---|---|
| 1 | `eksctl` failed with `exceeded max wait time for StackCreateComplete`; the control plane was created but the node group stack timed out | The default instance type (`m5.large`) is not free-tier eligible on a free-plan account, so EC2 could not launch workers | List eligible types with `describe-instance-types`, then set `instanceTypes` in `cluster.yaml` |
| 2 | Nodes still not created with `t3.small` | 2 GB RAM is too small for EKS system pods; capacity can also be unavailable in a particular AZ | Use a 2 vCPU / 8 GB type such as `m7i-flex.large`, with a second type as a fallback |
| 3 | `Error: unknown flag: --name` on `helm install` | `--name` was removed in Helm 3 | `helm install robot-shop --namespace robot-shop .` |
| 4 | `helm install` failed with `-n: command not found` | A line break inside a multi-line command split it into two commands | Keep each `\` continuation intact, or run the command on one line |
| 5 | `create-policy` returned `EntityAlreadyExists` | IAM policies are account-wide and outlive the cluster | Reuse the existing policy ARN |
| 6 | `kubectl get svc` and `kubectl get ingress` showed nothing | They only search the `default` namespace | Add `-n robot-shop` or `-A` |
| 7 | Only one load balancer appeared in the console | The ALB is created by the Ingress, which had not been applied yet | `kubectl apply -f ingress.yaml` |
| 8 | A load balancer existed but the site did not open | The `web` Service of `type: LoadBalancer` produced an **internal** NLB, which is not reachable from the internet | Use the internet-facing ALB from the Ingress |

### Why do I see two load balancers?

- **NLB (internal):** created automatically because the chart's `web` Service is `type: LoadBalancer`. It is not reachable from the internet.
- **ALB (internet-facing):** created by the Ingress. This is the one used to open the site.

Both point at the same `web` pods, so only the ALB is needed.

**Optional optimization:** set the `web` Service type to `ClusterIP` in `values.yaml` or the Service template, then run `helm upgrade robot-shop . -n robot-shop`. The ALB keeps working (it targets pod IPs) and the extra NLB, with its hourly cost, is removed.

### Useful debugging commands

```bash
kubectl describe ingress robot-shop -n robot-shop
kubectl logs -n kube-system deployment/aws-load-balancer-controller
kubectl describe pod <pod-name> -n robot-shop
eksctl utils describe-stacks --region=us-east-1 --cluster=demo-cluster-three-tier
```

For node group failures, check the CloudFormation Events tab and the EC2 Auto Scaling group Activity tab.

---

## 9. Cost Management

EKS is **not** covered by free-tier limits, so everything below draws from account credits. Approximate us-east-1 on-demand prices (check the AWS pricing pages for exact numbers):

| Resource | Approx. cost |
|---|---|
| EKS control plane | ~$0.10 / hour |
| Worker node (`m7i-flex.large`) | ~$0.095 / hour each |
| NAT gateway | ~$0.045 / hour plus data |
| Application / Network Load Balancer | ~$0.02 / hour each plus usage |
| EBS volumes | small per-GB monthly charge |

Tips:

- **Delete the cluster after every session.** Recreating takes about 20 minutes.
- Use **1 node** for learning if the pods fit; scale to 2 only when pods are `Pending`.
- Set a **budget alert** in the Billing console.
- Remove load balancers **before** deleting the cluster; they bill hourly.

---

## 10. Cleanup

Order matters. Load balancers live inside the VPC but are not part of the CloudFormation stacks, so deleting the cluster first can hang the VPC deletion.

```bash
# 1. Remove the Ingress (deletes the ALB)
kubectl delete -f EKS/helm/ingress.yaml

# 2. Remove the application (deletes the NLB if the Service is type LoadBalancer)
helm uninstall robot-shop -n robot-shop

# 3. Wait until both load balancers are gone from EC2 -> Load Balancers

# 4. Delete the cluster, node group and VPC
eksctl delete cluster -f cluster.yaml
```

**Verify nothing is left behind:**

```bash
eksctl get cluster --region us-east-1
```

Then, in the console, check CloudFormation (no `eksctl-demo-cluster-three-tier-*` stacks), EC2 Load Balancers, EBS Volumes, VPC NAT Gateways and Elastic IPs.

---

## 11. Key Learnings

- Free-plan accounts restrict EC2 instance types, and an unsuitable choice appears as a node group timeout, not a clear error. Check CloudFormation Events and the Auto Scaling group Activity tab.
- IRSA (OIDC provider plus IAM role plus service account) is how EKS add-ons and controllers get AWS permissions.
- The AWS Load Balancer Controller turns Kubernetes `Service` (NLB) and `Ingress` (ALB) objects into real AWS load balancers.
- Helm 3 differs from Helm 2 (`--name` is gone), and files outside `templates/` are not deployed by Helm.
- Always delete Kubernetes-created load balancers before deleting the cluster.
- Namespaces matter: `kubectl` commands default to `default`.

---

## 12. Future Improvements

- Add a custom domain with Route 53 and an HTTPS certificate from ACM
- Move IAM permissions from IRSA to EKS Pod Identity (as recommended by AWS)
- Replace in-cluster databases with managed services (RDS, ElastiCache, Amazon MQ)
- Enable CloudWatch Container Insights, or Prometheus and Grafana, for monitoring
- Add a Horizontal Pod Autoscaler and Cluster Autoscaler (or Karpenter)
- Automate infrastructure with Terraform and add a CI/CD pipeline (GitHub Actions)
- Add network policies and private subnets for the nodes

---

## Acknowledgements

Application: [Robot Shop](https://github.com/instana/robot-shop), a sample microservices store by Instana.


# 👩‍💻 Author

**Neha V M**

- GitHub: https://github.com/Neha-V-M
- LinkedIn: https://www.linkedin.com/in/neha-v-m/

---

⭐ If you found this project useful, consider giving it a **star**.
