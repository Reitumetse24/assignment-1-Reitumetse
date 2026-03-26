[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/w_4Lm1Qk)
# K3s High-Availability Cluster on AWS EC2

This guide walks you through deploying a **K3s HA cluster with 3 master (server) nodes** on AWS EC2 `t3.large` instances using embedded etcd. All examples are AWS-native — no external datastore or load balancer is required to bootstrap the cluster.

> For the full K3s documentation see <https://docs.k3s.io/>.

---

## Architecture

| Role | Count | Instance type | OS |
|------|-------|---------------|----|
| K3s server (master) | 3 | t3.large (2 vCPU / 8 GiB RAM) | Ubuntu 22.04 LTS |

`t3.large` provides enough CPU and RAM to run the K3s control plane (etcd, API server, scheduler, controller-manager) alongside application workloads. All 3 nodes form an HA control plane backed by embedded etcd. Worker nodes can be added later (see [Step 9](#step-9-optional-adding-worker-nodes)).

---

## Prerequisites

- AWS account with permissions to create EC2 instances, VPCs, and security groups
- AWS CLI v2 installed and configured (`aws configure`)
- An EC2 SSH key pair created in the target region
- `kubectl` installed on your local machine

---

## Step 1: AWS Infrastructure Setup

### 1.1 — Variables (set once, reuse throughout)

```sh
export AWS_REGION="us-east-1"
export KEY_NAME="my-k3s-key"        # existing EC2 key pair name
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text \
  --region $AWS_REGION)
export SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text \
  --region $AWS_REGION)
```

### 1.2 — Create a Security Group

```sh
export SG_ID=$(aws ec2 create-security-group \
  --group-name k3s-ha-sg \
  --description "K3s HA cluster security group" \
  --vpc-id $VPC_ID \
  --region $AWS_REGION \
  --query GroupId --output text)

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

# NodePort range (for test applications)
aws ec2 authorize-security-group-ingress --group-id $SG_ID \
  --protocol tcp --port 30000-32767 --cidr 0.0.0.0/0 --region $AWS_REGION

echo "Security group: $SG_ID"
```

### 1.3 — Launch 3 x t3.large Instances

```sh
# Ubuntu 22.04 LTS AMI (update the ami-* ID for your region)
export AMI_ID="ami-0c7217cdde317cfec"   # us-east-1 Ubuntu 22.04 LTS

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

> **Note:** Replace `ami-0c7217cdde317cfec` with the latest Ubuntu 22.04 LTS AMI for your region. Find it with:
> ```sh
> aws ec2 describe-images --owners 099720109477 \
>   --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
>   --query "sort_by(Images,&CreationDate)[-1].ImageId" \
>   --output text --region $AWS_REGION
> ```

### 1.4 — Note the Private and Public IPs

```sh
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=k3s-master-*" "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].[Tags[?Key=='Name']|[0].Value,PrivateIpAddress,PublicIpAddress]" \
  --output table --region $AWS_REGION
```

Record the values — you will need them throughout this guide:

| Hostname | Private IP | Public IP |
|----------|------------|-----------|
| k3s-master-1 | 172.31.86.199 | 44.211.196.217 |
| k3s-master-2 | 172.31.86.122 | 34.205.129.216 |
| k3s-master-3 | 172.31.80.243 | 54.164.13.239 |

---

## Step 2: Prepare All Nodes

Run the following on **each** of the 3 instances.

### 2.1 — SSH into the node

```sh
ssh -i ~/.ssh/$KEY_NAME.pem ubuntu@<public-ip>
```

### 2.2 — Set the hostname (run separately on each node)

```sh
# On k3s-master-1
sudo hostnamectl set-hostname k3s-master-1

# On k3s-master-2
sudo hostnamectl set-hostname k3s-master-2

# On k3s-master-3
sudo hostnamectl set-hostname k3s-master-3
```

### 2.3 — Update packages and set timezone

```sh
sudo apt-get update && sudo apt-get upgrade -y
sudo timedatectl set-timezone UTC
```

### 2.4 — Update `/etc/hosts` on every node

Add an entry for each node so they can resolve each other by hostname. Replace the IPs with your **private** IPs.

```sh
sudo tee -a /etc/hosts <<EOF
10.0.1.10  k3s-master-1
10.0.1.11  k3s-master-2
10.0.1.12  k3s-master-3
EOF
```

> K3s does not require swap to be disabled, but it is recommended for predictable performance.
> ```sh
> sudo swapoff -a
> sudo sed -i '/ swap / s/^/#/' /etc/fstab
> ```

---

## Step 3: Install K3s on the First Master Node

SSH into **k3s-master-1**.

### 3.1 — Create the K3s configuration file

```sh
sudo mkdir -p /etc/rancher/k3s

# Replace 10.0.1.10 with the private IP of k3s-master-1
# Replace 1.2.3.4  with the public IP / Elastic IP of k3s-master-1
sudo tee /etc/rancher/k3s/config.yaml <<EOF
cluster-init: true
node-ip: 10.0.1.10
advertise-address: 10.0.1.10
tls-san:
  - 10.0.1.10
  - 1.2.3.4
  - k3s-master-1
disable: [servicelb, traefik]
EOF
```

> **Why `disable: [servicelb, traefik]`?**
> - `servicelb` (Klipper) is replaced by the AWS cloud controller or an NLB.
> - `traefik` is replaced by the NGINX Ingress Controller in Step 7.
> Using the list syntax avoids the YAML duplicate-key bug where only the last `disable:` entry would take effect.

### 3.2 — Install K3s

```sh
curl -sfL https://get.k3s.io | sh -
```

### 3.3 — Verify the installation

```sh
sudo kubectl get nodes
sudo kubectl get pods -A
```

### 3.4 — Retrieve the cluster join token

```sh
sudo cat /var/lib/rancher/k3s/server/token
```

Save this token — you will need it in the next step.

---

## Step 4: Join Master Nodes 2 and 3

Run the following on **k3s-master-2** and **k3s-master-3** (adjust IPs accordingly).

### 4.1 — Create the K3s configuration file

```sh
sudo mkdir -p /etc/rancher/k3s

# Example for k3s-master-2. Replace IPs and token with your values.
sudo tee /etc/rancher/k3s/config.yaml <<EOF
server: https://10.0.1.10:6443
token: <token-from-master-1>
node-ip: 10.0.1.11
advertise-address: 10.0.1.11
tls-san:
  - 10.0.1.11
  - 1.2.3.5
  - k3s-master-2
disable: [servicelb, traefik]
EOF
```

### 4.2 — Install K3s as a server node

```sh
curl -sfL https://get.k3s.io | sh -s - server
```

### 4.3 — Verify cluster membership (run on any master node)

```sh
sudo kubectl get nodes -o wide
```

All 3 nodes should appear with status `Ready` and role `control-plane,master`.

```
NAME           STATUS   ROLES                       AGE   VERSION
k3s-master-1   Ready    control-plane,etcd,master   5m    v1.30.x+k3s1
k3s-master-2   Ready    control-plane,etcd,master   2m    v1.30.x+k3s1
k3s-master-3   Ready    control-plane,etcd,master   1m    v1.30.x+k3s1
```

---

## Step 5: Configure kubectl Remotely (Optional)

```sh
# Copy the kubeconfig from k3s-master-1 to your local machine
scp -i ~/.ssh/$KEY_NAME.pem ubuntu@<master-1-public-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s.yaml

# Update the server address to the public IP of a master node
sed -i 's|https://127.0.0.1:6443|https://<master-1-public-ip>:6443|' ~/.kube/k3s.yaml

# Use this kubeconfig
export KUBECONFIG=~/.kube/k3s.yaml

kubectl get nodes
```

---

## Step 6: Deploy a Test Application

```sh
kubectl apply -f web-app.yml

# Verify pods and service
kubectl get pods,svc

# Access the app — use the public IP of any master node and the NodePort
curl http://<master-public-ip>:30080
```

Expected output: `welcome to my web app!`

---

## Step 7: Configure NGINX Ingress Controller

K3s can deploy Helm charts automatically by placing a manifest in `/var/lib/rancher/k3s/server/manifests/`.

```sh
# On k3s-master-1
sudo cp nginx-ingress.yml /var/lib/rancher/k3s/server/manifests/nginx-ingress.yaml

# Verify the controller starts
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx get svc
```

The `LoadBalancer` service automatically provisions an AWS NLB. The NLB DNS name is shown in the `EXTERNAL-IP` column of `kubectl -n ingress-nginx get svc`.

The `nginx-ingress.yml` manifest in this repository already includes the NLB annotations:
```yaml
service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
```

---

## Step 8: Configure Default Storage

K3s ships with **local-path-provisioner** out-of-the-box. It dynamically provisions `hostPath` volumes on the node where the pod is scheduled. Refer to the [local-path-provisioner docs](https://github.com/rancher/local-path-provisioner/blob/master/README.md#usage) for more configuration options.

### 8.1 — Change the default storage path (optional)

Add the following to `/etc/rancher/k3s/config.yaml` on all master nodes and restart K3s:

```yaml
default-local-storage-path: /mnt/disk1
```

```sh
sudo systemctl restart k3s

# Restart the provisioner to pick up the new path
kubectl -n kube-system rollout restart deploy local-path-provisioner
```

### 8.2 — Test local-path storage

```sh
kubectl create -f pvc.yaml
kubectl create -f pod.yaml

kubectl get pvc
kubectl get pv
kubectl get pod volume-test
```

### 8.3 — EBS CSI driver (production alternative)

For production workloads that need durable, network-attached block storage, install the [AWS EBS CSI driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver):

```sh
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.35"
```

Create a StorageClass backed by EBS:

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

---

## Step 9: (Optional) Adding Worker Nodes

To add a dedicated worker (agent) node, launch an additional EC2 instance (any type) in the same security group and run:

```sh
# On the new worker node
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
server: https://10.0.1.10:6443
token: <token-from-master-1>
node-ip: <worker-private-ip>
EOF

curl -sfL https://get.k3s.io | sh -s - agent
```

Verify on a master node:

```sh
kubectl get nodes -o wide
```

---

## Step 10: Uninstalling K3s

```sh
# On server (master) nodes
/usr/local/bin/k3s-uninstall.sh

# On agent (worker) nodes
/usr/local/bin/k3s-agent-uninstall.sh
```

---

## Troubleshooting

### Check K3s service logs

```sh
journalctl -u k3s -f
```

### Check etcd health

```sh
sudo k3s etcd-snapshot list

# Detailed etcd member list
sudo k3s etcd-snapshot ls

# etcdctl (available inside the K3s binary)
sudo k3s kubectl -n kube-system exec -it \
  $(sudo k3s kubectl -n kube-system get pod -l component=etcd -o jsonpath='{.items[0].metadata.name}') \
  -- etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
     --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
     --key=/var/lib/rancher/k3s/server/tls/etcd/client.key \
     member list
```

### Nodes stay in `NotReady`

- Check security group rules — ensure ports 8472/UDP (Flannel VXLAN) and 10250/TCP (Kubelet) are open between nodes.
- Verify all nodes can resolve each other's hostnames: `ping k3s-master-2` from k3s-master-1.

### API server unreachable from kubectl

- Ensure port 6443/TCP is open in the security group for your local IP.
- Verify the `server:` URL in your local kubeconfig points to the correct public IP.

### Instance Metadata Service (IMDS)

K3s uses IMDSv2 on EC2 for the node's ProviderID. If you see errors related to instance metadata, ensure IMDSv2 is enabled and the hop limit is at least 2:

```sh
aws ec2 modify-instance-metadata-options \
  --instance-id <instance-id> \
  --http-put-response-hop-limit 2 \
  --http-endpoint enabled \
  --region $AWS_REGION
```

### Private Registry (Amazon ECR)

See `registries.yml` for the ECR configuration. The recommended approach on EC2 is to attach the `AmazonEC2ContainerRegistryReadOnly` IAM policy to the instance profile — no static credentials are needed.





### Name: Reitumetse Modise
### Student number: 222910038



#### What is K3s and Why is it Used?

K3s is a lightweight, fully compliant Kubernetes distribution developed by Rancher Labs. It is designed to run efficiently in resource-constrained environments such as edge computing, IoT devices, and small cloud instances. Unlike standard Kubernetes, K3s reduces complexity by removing non-essential components and packaging everything into a single binary, making it easier to install, manage, and operate.

K3s is widely used because it requires less memory and CPU, starts quickly, and simplifies cluster setup while still maintaining core Kubernetes functionality. This makes it ideal for learning environments, development, and production scenarios where lightweight orchestration is needed, such as in distributed and 5G edge networks.

#### Key Components of K3s
1. Control Plane

The control plane is responsible for managing the entire cluster. It includes components such as the API server, scheduler, and controller manager. These components ensure that the desired state of the cluster is maintained, workloads are scheduled correctly, and the system operates reliably.

2. Agents (Worker Nodes)

Agents, also known as worker nodes, are responsible for running application workloads. They communicate with the control plane and execute tasks such as running containers and reporting node status. In a K3s cluster, agents are lightweight and easy to join using a cluster token.

3. Container Runtime

K3s uses a container runtime (by default, containerd) to run containers. The container runtime is responsible for pulling container images, starting and stopping containers, and managing their lifecycle. This is the core layer that actually executes applications within the cluster.

4. CNI (Container Network Interface)

The CNI handles networking between pods and services within the cluster. It ensures that containers can communicate with each other across different nodes. K3s includes a default networking solution (Flannel), which simplifies cluster networking setup.

5. Ingress / Load Balancer

Ingress controllers manage external access to services within the cluster, typically via HTTP or HTTPS. K3s includes Traefik by default, but in this project it was disabled and replaced with an NGINX Ingress Controller. Load balancing ensures traffic is distributed efficiently across multiple pods, improving availability and performance.

6. Storage

Storage in K3s allows applications to persist data beyond the lifecycle of individual containers. K3s includes a simple local storage provisioner by default, but it can also integrate with external storage systems such as cloud-based volumes. Persistent storage is essential for stateful applications like databases.

<img width="1366" height="720" alt="Ree" src="https://github.com/user-attachments/assets/f0b2e50a-bdac-4652-bef9-b94fba0ae814" />
<img width="1366" height="720" alt="reee" src="https://github.com/user-attachments/assets/03d1371e-6142-4058-8efe-9abeceac4261" />
<img width="1366" height="720" alt="reeeee" src="https://github.com/user-attachments/assets/f1abfd99-6f9d-48c6-8f6c-f41c60f2e847" />
<img width="1366" height="720" alt="reitu" src="https://github.com/user-attachments/assets/61e634e7-233c-45b3-9c3f-12821086e9fa" />

#### Reflection

This project provided valuable hands-on experience in deploying a lightweight Kubernetes cluster using K3s on AWS. Through the process, I gained a deeper understanding of cluster architecture, node configuration, and the importance of proper networking between nodes. Setting up multiple master nodes helped me understand high availability concepts and how distributed systems maintain reliability. I also learned how to use SSH securely, configure Linux systems, and manage services using command-line tools, which strengthened my practical system administration skills.

One of the main challenges I faced was correctly configuring communication between nodes. Initially, there were issues with nodes not resolving each other’s hostnames, which prevented the cluster from forming properly. I resolved this by carefully updating the /etc/hosts file on each node with the correct private IP addresses. Another challenge was handling SSH access using a .pem key in Windows PowerShell, as permission and path issues caused connection failures. This was resolved by ensuring the correct file path and command syntax were used. Additionally, YAML configuration errors, such as incorrect indentation, caused failures during K3s installation. I addressed this by carefully reviewing and correcting the formatting.

K3s is a lightweight distribution of Kubernetes designed for edge, IoT, and resource-constrained environments, but it still retains the core functionality of standard Kubernetes. This makes it highly relevant to production environments, especially in modern cloud-native and 5G architectures. In 5G networks, services are designed using microservices and container-based approaches, which require orchestration platforms like Kubernetes. K3s demonstrates how Kubernetes can be optimized for edge computing, which is a critical component of 5G infrastructure where low latency and distributed processing are essential.

Virtualization and containerization play a key role in enabling scalable services. Virtualization allows multiple virtual machines to run on a single physical server, improving resource utilization and flexibility. Containerization, on the other hand, allows applications to run in isolated environments with all their dependencies, making them portable and consistent across different environments. Kubernetes builds on containerization by automating deployment, scaling, and management of these containers. Together, these technologies enable organizations to deploy highly scalable, resilient, and efficient systems that can adapt to changing workloads and user demands.
