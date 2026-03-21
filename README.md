# Assignment 1 — K3s on AWS

## Full Name: Esethu Elsie Mbizweni

## Student Number: 222458968

## Repository: assignment-1-EsethuMbizweni

## Date: 23 March 2026

---

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Architecture Explanation](#architecture-explanation)
3. [Environment Setup](#environment-setup)
4. [Installation Steps](#installation-steps)
5. [Evidence of Deployment](#evidence-of-deployment)
6. [NGINX Ingress Controller](#nginx-ingress-controller)
7. [Storage Configuration](#storage-configuration)
8. [Uninstalling K3s](#uninstalling-k3s)
9. [Troubleshooting](#troubleshooting)
10. [Reflection](#reflection)

---

## System Requirements

| Component | Specification |
| --- | --- |
| Cloud Provider | AWS (us-east-1 / N. Virginia) |
| Instance Type | t3.large |
| vCPUs | 2 per node |
| RAM | 8 GB per node |
| Storage | 20 GiB gp3 per node |
| OS | Ubuntu Server 22.04 LTS (64-bit x86) |
| Nodes | 3 × EC2 instances (all as control plane masters) |
| K3s Version | v1.34.5+k3s1 |

---

## Architecture Explanation

### What is K3s?

K3s is a lightweight, CNCF-certified Kubernetes distribution developed by Rancher Labs (now SUSE). It packages the entire Kubernetes control plane into a single binary under 100MB, making it ideal for edge computing, IoT, CI/CD pipelines, and resource-constrained environments. Unlike standard Kubernetes (kubeadm), K3s removes legacy and alpha features, replaces etcd with SQLite by default (or embedded etcd for HA), and bundles everything needed — container runtime, CNI, ingress controller, and storage provisioner — out of the box.

### Why K3s on AWS?

- **Simplicity:** Single binary install via a one-line curl command
- **HA Support:** Embedded etcd enables multi-master HA without external dependencies
- **Cost-effective:** Runs on smaller instance types than full Kubernetes
- **Production-ready:** CNCF certified, suitable for 5G edge and cloud-native workloads
- **Built-in components:** Includes Flannel CNI and local-path storage provisioner

### Cluster Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS VPC (172.31.0.0/16)              │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │   k3s-master-1   │  │   k3s-master-2   │                │
│  │  172.31.32.92    │  │  172.31.95.163   │                │
│  │                  │  │                  │                │
│  │  Control Plane   │  │  Control Plane   │                │
│  │  etcd (embedded) │  │  etcd (embedded) │                │
│  │  API Server      │  │  API Server      │                │
│  │  Scheduler       │  │  Scheduler       │                │
│  └────────┬─────────┘  └────────┬─────────┘                │
│           │                     │                           │
│           └──────────┬──────────┘                           │
│                      │ K3s Cluster                          │
│           ┌──────────┴──────────┐                           │
│           │    k3s-master-3     │                           │
│           │   172.31.85.34      │                           │
│           │                     │                           │
│           │  Control Plane      │                           │
│           │  etcd (embedded)    │                           │
│           └─────────────────────┘                           │
│                                                             │
│  Security Group: k3s-ha-sg                                  │
│  Ports: 22, 6443, 2379-2380, 8472/UDP, 10250, 30000-32767  │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Implementation | Purpose |
| --- | --- | --- |
| Control Plane | `k3s-master-1`, `k3s-master-2`, `k3s-master-3` | Runs Kubernetes API server, scheduler, and controller manager |
| etcd | Embedded (per master) | Stores cluster state and provides high availability across all 3 members |
| Container Runtime | `containerd` | Responsible for running containers on each node |
| CNI | Flannel (VXLAN) | Provides pod networking across nodes (uses UDP port 8472) |
| Ingress | NGINX Ingress Controller | Handles HTTP/HTTPS routing into the cluster |
| Storage (Dev/Test) | local-path-provisioner | Provides dynamic hostPath-based volume provisioning |
| Storage (Production) | AWS EBS CSI Driver | Provides durable, network-attached block storage |

---

## Environment Setup

### Step 1 — Set Variables and Create the Security Group

Set shell variables once and reuse them throughout the setup:

```bash
export AWS_REGION="us-east-1"
export KEY_NAME="my-k3s-key"
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text \
  --region $AWS_REGION)
export SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text \
  --region $AWS_REGION)
```

Create the security group:

```bash
export SG_ID=$(aws ec2 create-security-group \
  --group-name k3s-ha-sg \
  --description "K3s HA cluster security group" \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query GroupId --output text)
```

Add inbound rules:

```bash
# SSH
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 22 --cidr 0.0.0.0/0 --region $AWS_REGION

# Kubernetes API server
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 6443 --cidr 0.0.0.0/0 --region $AWS_REGION

# etcd (inter-node only — restrict to the SG itself)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 2379-2380 --source-group $SG_ID --region $AWS_REGION

# Kubelet
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 10250 --source-group $SG_ID --region $AWS_REGION

# Flannel VXLAN
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol udp --port 8472 --source-group $SG_ID --region $AWS_REGION

# NodePort range
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 30000-32767 --cidr 0.0.0.0/0 --region $AWS_REGION
```

Inbound rules summary:

| Type | Protocol | Port Range | Source | Purpose |
| --- | --- | --- | --- | --- |
| SSH | TCP | 22 | 0.0.0.0/0 | Remote access |
| Custom TCP | TCP | 6443 | 0.0.0.0/0 | Kubernetes API server |
| Custom TCP | TCP | 2379-2380 | k3s-ha-sg (self) | etcd cluster communication |
| Custom UDP | UDP | 8472 | k3s-ha-sg (self) | Flannel VXLAN overlay network |
| Custom TCP | TCP | 10250 | k3s-ha-sg (self) | Kubelet API |
| Custom TCP | TCP | 30000-32767 | 0.0.0.0/0 | NodePort services |

> **Important:** Create the security group **before** launching instances so that self-referencing rules (etcd, Flannel, Kubelet) can be added immediately. AWS does not allow a security group to reference itself if it does not yet exist.

---

### Step 2 — Launch 3 × t3.large EC2 Instances

```bash
export AMI_ID="ami-0c7217cdde317cfec"   # Ubuntu 22.04 LTS, us-east-1

for i in 1 2 3; do
  aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.large \
    --key-name $KEY_NAME \
    --security-group-ids $SG_ID \
    --subnet-id $SUBNET_ID \
    --associate-public-ip-address \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=k3s-master-$i}]" \
    --region $AWS_REGION \
    --query "Instances[0].InstanceId" --output text
done
```

### Step 3 — Record IP Addresses

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=k3s-master-*" "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].[Tags[?Key=='Name']|[0].Value,PrivateIpAddress,PublicIpAddress]" \
  --output table --region $AWS_REGION
```

| Hostname | Private IP | Public IP |
| --- | --- | --- |
| k3s-master-1 | 172.31.32.92 | 13.217.107.13 |
| k3s-master-2 | 172.31.95.163 | 54.205.89.127 |
| k3s-master-3 | 172.31.85.34 | 98.93.234.105 |

> **Note:** AWS Learner Lab assigns dynamic public IPs that change on restart.

---

## Installation Steps

### Step 4 — Prepare All Nodes

SSH into each node and run the following (adjusting the hostname per node):

```bash
# Set hostname (change per node)
sudo hostnamectl set-hostname k3s-master-1   # or k3s-master-2 / k3s-master-3

# Update packages
sudo apt-get update && sudo apt-get upgrade -y

# Set timezone
sudo timedatectl set-timezone UTC

# Update /etc/hosts on ALL nodes (same content on each)
sudo tee -a /etc/hosts <<EOF
172.31.32.92   k3s-master-1
172.31.95.163  k3s-master-2
172.31.85.34   k3s-master-3
EOF

# Disable swap (recommended for predictable performance)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

---

### Step 5 — Install K3s on Master-1 (Cluster Bootstrap)

SSH into `k3s-master-1` and create the K3s configuration file:

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
cluster-init: true
node-ip: 172.31.32.92
advertise-address: 172.31.32.92
tls-san:
  - 172.31.32.92
  - 13.217.107.13
  - k3s-master-1
disable: [servicelb, traefik]
EOF
```

> **Why `disable: [servicelb, traefik]`?**
> `servicelb` (Klipper) is replaced by the AWS NLB provisioned by the NGINX Ingress Controller.
> `traefik` is replaced by NGINX Ingress in Step 7. Using list syntax avoids the YAML duplicate-key bug where only the last `disable:` entry takes effect.

Install K3s:

```bash
curl -sfL https://get.k3s.io | sh -
```

Verify the installation:

```bash
sudo systemctl status k3s
sudo kubectl get nodes
sudo kubectl get pods -A
```

Retrieve the cluster join token (save this — you will need it for masters 2 and 3):

```bash
sudo cat /var/lib/rancher/k3s/server/token
```

---

### Step 6 — Join Master-2 and Master-3 as Server Nodes

Run the following on `k3s-master-2` (adjust IPs and token for `k3s-master-3`):

```bash
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
server: https://172.31.32.92:6443
token: <token-from-master-1>
node-ip: 172.31.95.163
advertise-address: 172.31.95.163
tls-san:
  - 172.31.95.163
  - 54.205.89.127
  - k3s-master-2
disable: [servicelb, traefik]
EOF

curl -sfL https://get.k3s.io | sh -s - server
```

Verify all 3 nodes after joining:

```bash
sudo kubectl get nodes -o wide
```

Expected output:

```
NAME           STATUS   ROLES                       AGE   VERSION
k3s-master-1   Ready    control-plane,etcd,master   5m    v1.34.5+k3s1
k3s-master-2   Ready    control-plane,etcd,master   2m    v1.34.5+k3s1
k3s-master-3   Ready    control-plane,etcd,master   1m    v1.34.5+k3s1
```

---

### Step 7 — Configure kubectl Remotely (Optional)

```bash
# Copy kubeconfig from master-1 to your local machine
scp -i ~/.ssh/my-k3s-key.pem ubuntu@13.217.107.13:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s.yaml

# Fix the server address
sed -i 's|https://127.0.0.1:6443|https://13.217.107.13:6443|' ~/.kube/k3s.yaml

# Use this kubeconfig
export KUBECONFIG=~/.kube/k3s.yaml

kubectl get nodes
```

---

### Step 8 — Deploy a Test Application

```bash
kubectl apply -f web-app.yml

# Verify pods and service
kubectl get pods,svc

# Access the app via any master node's public IP and the NodePort
curl http://13.217.107.13:30080
```

Expected output: `welcome to my web app!`

---

## Evidence of Deployment

> **Screenshots below show the actual deployment. Terminal prompts and instance metadata bars confirm node hostnames and IPs, proving originality.**

---

### 1. GitHub Classroom — Assignment Accepted

<img width="1355" height="618" alt="image" src="https://github.com/user-attachments/assets/1a0a5cfd-b4e5-4fad-a378-d80b08a0d89b" />



GitHub Classroom confirmation showing the assignment `open5g on AWS EC2` accepted under the repository `open5g-on-aws-ec2-EsethuElsie` in the `cput-it-advdip` organisation.

---

### 2. AWS Academy Profile — Student Identity Confirmed

<img width="1358" height="671" alt="image" src="https://github.com/user-attachments/assets/8804f578-2aac-4f4c-939d-834dd35ec142" />


AWS Academy (Canvas) profile page showing student number `222458968@mycput.ac.za`, confirming the account used throughout this assignment belongs to Esethu Elsie Mbizweni.

---

### 3. AWS EC2 Console — All 3 Instances Running


<img width="1363" height="646" alt="image" src="https://github.com/user-attachments/assets/fafb5f39-7d34-472d-ba96-c7ee70fa9688" />


AWS EC2 console showing all three instances — `k3s-master-1`, `k3s-master-2`, and `k3s-master-3` — with instance state **Running** and instance type **t3.large** in the us-east-1 (N. Virginia) region. Account ID `4626-2943-6342` is visible in the top-right corner.

---

### 4. K3s Installation on Master-1 — Service Active

<img width="1362" height="627" alt="image" src="https://github.com/user-attachments/assets/ea163301-d813-4148-9a05-3059f786542b" />


Terminal output on `k3s-master-1` (PublicIP: `54.85.138.74`, PrivateIP: `172.31.32.92`) showing the K3s installer completing successfully. The `systemctl status k3s` output confirms the service is **active (running)** since `Sat 2026-03-21 08:30:52 UTC`.

---

### 5. Master-1 — kubectl get nodes & Token Retrieved

<img width="1362" height="627" alt="image" src="https://github.com/user-attachments/assets/d336b630-3052-44aa-adc4-c6d27a99df26" />


Terminal on `k3s-master-1` showing:
- `kubectl get pods -A` — all system pods running (CoreDNS, local-path-provisioner, metrics-server, Traefik, svclb-traefik)
- `kubectl cluster-info` — control plane running at `https://127.0.0.1:6443`
- `kubectl get nodes -o wide` — k3s-master-1 `Ready`, `control-plane`, v1.34.5+k3s1, internal IP `172.31.32.92`
- `cat /var/lib/rancher/k3s/server/token` — cluster join token retrieved for use by master-2 and master-3

---

### 6. K3s Installation on Master-2 — Node Ready

<img width="1361" height="558" alt="image" src="https://github.com/user-attachments/assets/c63fd1fb-d086-4009-8fce-de49d5336c34" />


Terminal on `k3s-master-2` (PublicIP: `3.91.161.94`, PrivateIP: `172.31.95.163`) showing the K3s installer completing and `kubectl get nodes -o wide` confirming `k3s-master-2` is **Ready** with role `control-plane` running v1.34.5+k3s1.

---

### 7. Test Application Deployed — Pods and Services

<img width="1364" height="581" alt="image" src="https://github.com/user-attachments/assets/2cc60928-2e4a-483e-94a6-fd90c4795abd" />


Terminal on `k3s-master-1` showing:
- `kubectl apply -f ~/web-app.yml` — deployment and service created
- `kubectl get pods,svc` — nginx pod and web-app pods all **Running**
- Services: `service/nginx` on NodePort `30899`, `service/web-app` on NodePort `30080`

---

### 8. NGINX Welcome Page — NodePort Access Confirmed

<img width="1364" height="551" alt="image" src="https://github.com/user-attachments/assets/db058cb2-9c76-4ebd-9f21-c20a8ed528f8" />


Browser accessing `http://54.85.138.74:30080` — the **Welcome to nginx!** page confirms the deployed service is reachable via NodePort from the public internet.

---

### 9. NGINX Welcome Page — curl Confirmed

<img width="1358" height="340" alt="image" src="https://github.com/user-attachments/assets/70ec2f5b-de69-4877-8106-cfda4bd3eee7" />


Terminal on `k3s-master-1` showing the raw HTML response from `curl`, confirming the nginx deployment is serving correctly and matching the browser output above.

---

### 10. NGINX Ingress Controller — Pods and Services

<img width="1358" height="536" alt="image" src="https://github.com/user-attachments/assets/2434b0e7-3b16-49a7-9500-1cfb2bb82faf" />


Terminal on `k3s-master-1` (PublicIP: `54.85.138.74`) showing:
- All NGINX Ingress resources created (configmap, service, deployment, jobs, ingressclass, webhook)
- `kubectl -n ingress-nginx get pods -w` — admission jobs Completed, controller pod **Running 1/1**
- `kubectl -n ingress-nginx get svc` — `ingress-nginx-controller` service of type `LoadBalancer` on ports `80:30886` and `443:32090`, with EXTERNAL-IP pending (NLB provisioning)

---

### 11. K3s Uninstall — Clean Removal on Master-1
<img width="1365" height="623" alt="image" src="https://github.com/user-attachments/assets/c00e8cf1-1943-454d-adc9-e1304a407c7f" />



Terminal on `k3s-master-1` (PublicIP: `98.91.247.188`, PrivateIP: `172.31.32.92`) showing the output of `k3s-uninstall.sh` — removing systemd service files, symlinks, Flannel state, rancher data directories, kubelet, and the K3s binary itself. The script exits with `return 0`, confirming a clean uninstall.

---

### 12. Troubleshooting — K3s Service Failure (Earlier Attempt)

<img width="1360" height="546" alt="image" src="https://github.com/user-attachments/assets/39e71711-7285-4a01-a47a-5fedb28e680a" />


Terminal on an earlier `k3s-master-1` instance (PublicIP: `3.84.136.60`, PrivateIP: `172.31.38.97`) showing a K3s installation that **failed** with the error:

```
Job for k3s.service failed because the control process exited with error code.
See "systemctl status k3s.service" and "journalctl -xeu k3s.service" for details.
```

This failure was caused by a misconfigured `config.yaml` using placeholder IPs and an incorrect token. The node was fully uninstalled, the config corrected, and K3s reinstalled successfully on the replacement instance (as shown in screenshots 4–5 above).

---

## NGINX Ingress Controller

K3s deploys Helm charts automatically via manifests placed in `/var/lib/rancher/k3s/server/manifests/`.

```bash
# On k3s-master-1
sudo cp nginx-ingress.yml /var/lib/rancher/k3s/server/manifests/nginx-ingress.yaml

# Verify the controller starts
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx get svc
```

The `nginx-ingress.yml` manifest includes the following AWS NLB annotations:

```yaml
service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
```

The LoadBalancer service automatically provisions an AWS NLB. The NLB DNS name appears in the `EXTERNAL-IP` column of `kubectl -n ingress-nginx get svc`.

---

## Storage Configuration

### 8.1 — Default Storage (local-path-provisioner)

K3s ships with `local-path-provisioner` built-in. It dynamically provisions `hostPath` volumes on the node where the pod is scheduled.

Change the default storage path (optional):

```bash
# Add to /etc/rancher/k3s/config.yaml on all master nodes
default-local-storage-path: /mnt/disk1

sudo systemctl restart k3s

# Restart the provisioner to pick up the new path
kubectl -n kube-system rollout restart deploy local-path-provisioner
```

Test local-path storage:

```bash
kubectl create -f pvc.yaml
kubectl create -f pod.yaml

kubectl get pvc
kubectl get pv
kubectl get pod volume-test
```

### 8.2 — AWS EBS CSI Driver (Production)

For production workloads requiring durable, network-attached block storage:

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.35"
```

Create a StorageClass backed by EBS gp3:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
```

Verify:

```bash
kubectl get storageclass
kubectl -n kube-system get pods | grep ebs
```

---

## Uninstalling K3s

```bash
# On server (master) nodes — run on each master
sudo /usr/local/bin/k3s-uninstall.sh

# On agent (worker) nodes — if any were added
sudo /usr/local/bin/k3s-agent-uninstall.sh

# Verify removal
sudo systemctl status k3s
# Expected: Unit k3s.service could not be found.
```

---

## Troubleshooting

**Check K3s service logs**

```bash
journalctl -u k3s -f
```

**Check etcd health**

```bash
sudo k3s etcd-snapshot list

# Detailed etcd member list via etcdctl
sudo k3s kubectl -n kube-system exec -it \
  $(sudo k3s kubectl -n kube-system get pod -l component=etcd -o jsonpath='{.items[0].metadata.name}') \
  -- etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
     --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
     --key=/var/lib/rancher/k3s/server/tls/etcd/client.key \
     member list
```

**Nodes stay in NotReady**
- Check security group rules — ensure ports 8472/UDP (Flannel VXLAN) and 10250/TCP (Kubelet) are open between nodes
- Verify hostname resolution: `ping k3s-master-2` from k3s-master-1

**API server unreachable from kubectl**
- Ensure port 6443/TCP is open in the security group for your local IP
- Verify the `server:` URL in your local kubeconfig points to the correct public IP

**Token CA hash mismatch on join**
- Re-copy the token from master-1: `sudo cat /var/lib/rancher/k3s/server/token | tr -d '\n'`
- Uninstall K3s on the failing node first: `sudo /usr/local/bin/k3s-uninstall.sh`
- Reinstall with the corrected token and private IP in `config.yaml`

**Permission errors with k3s.yaml**

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

**Instance Metadata Service (IMDS)**

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id <instance-id> \
  --http-put-response-hop-limit 2 \
  --http-endpoint enabled \
  --region us-east-1
```

---

## Reflection

### What I Learned

Through this project, I gained hands-on experience deploying a highly available Kubernetes cluster using K3s across multiple AWS EC2 instances. I learned how to configure control plane nodes with embedded etcd clustering, manage node joining via tokens, and expose services using NodePort and Ingress controllers.

I also deepened my understanding of how Kubernetes components like the API server, scheduler, and etcd work together to maintain cluster state. Working with `kubectl` to deploy, expose, and debug workloads gave me practical skills that go beyond theoretical knowledge. Before this assignment I understood concepts such as control planes, pods, and services at a conceptual level — but provisioning a real multi-node HA cluster from scratch made those concepts concrete.

I also learned that infrastructure configuration is just as important as the software itself. For example, using self-referencing security group rules for inter-node communication — rather than opening internal ports to the public internet — is a critical security practice that I had not fully appreciated before.

---

### Challenges I Faced and How I Resolved Them

**Masters 2 and 3 failing to join the cluster**
The most significant challenge was that `k3s-master-2` and `k3s-master-3` kept failing with a "connection refused" error on port 6443. After examining the logs with:

```bash
journalctl -xeu k3s.service
```

I identified that the AWS Security Group was blocking inter-node communication on that port. I resolved this by adding an inbound rule to allow TCP traffic on port 6443 between the nodes within the security group.

**Misconfigured IP address in config.yaml**
A second issue arose because I had used a placeholder IP (`10.0.1.10`) in `config.yaml` instead of the actual private IP of master-1 (`172.31.32.92`). This caused the token validation to fail when master-2 attempted to join. I resolved this by completely uninstalling K3s on the affected node and performing a clean reinstall with the correct private IP values.

**Permission errors with k3s.yaml**
After copying the kubeconfig to my local machine, `kubectl` commands were failing due to file permission errors. This was fixed by setting the correct permissions:

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

**Security group creation ordering**
I initially tried to add self-referencing rules to the security group during instance launch, which AWS does not allow because the group does not yet exist at that point. The solution was to create the security group first using the CLI, add all rules immediately, and then select it when launching the instances.

---

### How K3s Relates to Production Kubernetes and 5G Cloud-Native Concepts

K3s is a lightweight, CNCF-certified Kubernetes distribution designed for resource-constrained environments. In production, full Kubernetes distributions such as EKS or GKE are used for large-scale workloads, while K3s is increasingly adopted in edge computing and 5G Multi-Access Edge Computing (MEC) deployments.

In 5G networks, cloud-native Network Functions (CNFs) replace traditional hardware-based Virtual Network Functions (VNFs). These CNFs are containerized and orchestrated using Kubernetes, making K3s a viable platform for deploying them at the network edge where resources are limited but low latency is critical. Concepts such as service meshes, ingress controllers, and high-availability clusters — all practised in this project — are directly applicable to 5G core network deployments.

---

### How Virtualization and Containerization Enable Scalable Services

Virtualization allows physical hardware to be divided into multiple isolated Virtual Machines (VMs), each running its own OS. This improves resource utilization and enables rapid provisioning of infrastructure, as demonstrated by using AWS EC2 instances in this project.

Containerization takes this further by packaging applications and their dependencies into lightweight, portable containers that share the host OS kernel. This results in faster startup times, lower overhead, and greater density compared to VMs.

Together, virtualization and containerization form the foundation of modern scalable services — VMs provide the isolated infrastructure layer while containers provide the application layer. Kubernetes orchestrates these containers at scale, enabling automatic scheduling, self-healing, and rolling updates. These capabilities are essential for maintaining the availability and performance of services in both cloud and 5G environments.

---

### Future Improvements

- Deploy infrastructure using Infrastructure as Code (Terraform)
- Implement monitoring and observability with Prometheus and Grafana
- Configure automated etcd snapshots for disaster recovery
- Apply Kubernetes RBAC policies for improved cluster security
- Deploy applications using Helm charts or a GitOps workflow (e.g. ArgoCD)

---

*Repository: assignment-1-EsethuMbizweni | Student: 222458968*
