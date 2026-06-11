# EKS Hybrid Nodes on Latitude.sh Bare Metal via Megaport

End-to-end deployment guide for running Amazon EKS hybrid nodes on Latitude.sh bare metal infrastructure with private connectivity through Megaport Direct Connect.

## Architecture Overview

The deployment consists of three independent layers that need to be set up in order:

1. AWS layer (VPC, EKS cluster, IAM, Direct Connect hosted VIF acceptance)
2. Megaport layer (MCR, VXC to AWS, VXC to Latitude server, BGP)
3. Latitude bare metal layer (server provisioning, VLAN config, nodeadm registration)

The control plane runs in AWS, the worker node runs on Latitude bare metal, and Megaport provides the private network fabric between them.

## Prerequisites

The following are required before starting:

- AWS account with admin permissions
- Megaport account with at least one provisioned MCR
- Latitude.sh account with a bare metal server provisioned
- Local tools installed: AWS CLI v2, eksctl, kubectl, terraform 1.5 or later, cilium CLI 0.16 or later, Session Manager plugin for AWS CLI

## Critical CIDR Design Principle

The EKS RemoteNodeNetwork CIDR, the MCR A-End interface IP, and the node IP must all be in the same /24 subnet. This ensures layer 2 ARP works correctly between the MCR and the node. Mismatched subnets cause silent failures in kubectl exec, kubectl logs, and pod-to-control-plane communication.

The working design is:

```
MCR A-End interface IP:   10.10.0.1/24
Node IP:                  10.10.0.10/24
EKS RemoteNodeNetwork:    10.10.0.0/24
```

All three live in the same layer 2 segment so the MCR can ARP for the node and vice versa.

## Configuration Variables

Set these once before running any of the steps below. Save them as `terraform.tfvars` and reference them throughout.

```hcl
# AWS Configuration
aws_region              = "us-east-1"
vpc_cidr                = "10.0.0.0/16"
vpc_subnet_a_cidr       = "10.0.1.0/24"
vpc_subnet_b_cidr       = "10.0.2.0/24"
availability_zones      = ["us-east-1a", "us-east-1b"]

# EKS Cluster Configuration
cluster_name            = "lsh-hybrid-demo"
kubernetes_version      = "1.35"

# Hybrid Node Network Configuration
node_cidr               = "10.10.0.0/24"
node_ip                 = "10.10.0.10"
mcr_gateway_ip          = "10.10.0.1"
pod_network_cidr        = "192.168.0.0/16"

# Direct Connect BGP Configuration
customer_asn            = 4200000001
amazon_asn              = 64512

# Latitude Server Configuration
latitude_location       = "Dallas"
```

## Part 1: AWS Foundation with Terraform

Save the following as `main.tf` in your terraform directory.

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Variables
variable "aws_region" { type = string }
variable "vpc_cidr" { type = string }
variable "vpc_subnet_a_cidr" { type = string }
variable "vpc_subnet_b_cidr" { type = string }
variable "availability_zones" { type = list(string) }
variable "cluster_name" { type = string }
variable "kubernetes_version" { type = string }
variable "node_cidr" { type = string }
variable "pod_network_cidr" { type = string }

# VPC with required DNS settings for private EKS endpoint
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.cluster_name}-vpc"
  }
}

# Private subnets in two AZs
resource "aws_subnet" "private_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.vpc_subnet_a_cidr
  availability_zone = var.availability_zones[0]

  tags = {
    Name = "${var.cluster_name}-private-a"
  }
}

resource "aws_subnet" "private_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.vpc_subnet_b_cidr
  availability_zone = var.availability_zones[1]

  tags = {
    Name = "${var.cluster_name}-private-b"
  }
}

# Virtual Private Gateway for Direct Connect termination
resource "aws_vpn_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.cluster_name}-vgw"
  }
}

# Explicit route tables for each subnet (eksctl requires explicit, not main)
resource "aws_route_table" "private_a" {
  vpc_id = aws_vpc.main.id

  propagating_vgws = [aws_vpn_gateway.main.id]

  route {
    cidr_block = var.node_cidr
    gateway_id = aws_vpn_gateway.main.id
  }

  route {
    cidr_block = var.pod_network_cidr
    gateway_id = aws_vpn_gateway.main.id
  }

  tags = {
    Name = "${var.cluster_name}-private-rt-a"
  }
}

resource "aws_route_table" "private_b" {
  vpc_id = aws_vpc.main.id

  propagating_vgws = [aws_vpn_gateway.main.id]

  route {
    cidr_block = var.node_cidr
    gateway_id = aws_vpn_gateway.main.id
  }

  route {
    cidr_block = var.pod_network_cidr
    gateway_id = aws_vpn_gateway.main.id
  }

  tags = {
    Name = "${var.cluster_name}-private-rt-b"
  }
}

resource "aws_route_table_association" "private_a" {
  subnet_id      = aws_subnet.private_a.id
  route_table_id = aws_route_table.private_a.id
}

resource "aws_route_table_association" "private_b" {
  subnet_id      = aws_subnet.private_b.id
  route_table_id = aws_route_table.private_b.id
}

# Security group for hybrid nodes ingress
resource "aws_security_group" "hybrid_nodes" {
  name        = "${var.cluster_name}-hybrid-nodes-sg"
  description = "Allow hybrid nodes to reach EKS control plane"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTPS from hybrid nodes"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.node_cidr]
  }

  ingress {
    description = "HTTPS from pod network"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.pod_network_cidr]
  }

  ingress {
    description = "Kubelet API from VPC"
    from_port   = 10250
    to_port     = 10250
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  tags = {
    Name = "${var.cluster_name}-hybrid-nodes-sg"
  }
}

# IAM role for hybrid nodes
resource "aws_iam_role" "hybrid_nodes" {
  name = "${var.cluster_name}-hybrid-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "ssm.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "hybrid_worker_node" {
  role       = aws_iam_role.hybrid_nodes.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodeMinimalPolicy"
}

resource "aws_iam_role_policy_attachment" "hybrid_ecr_pull" {
  role       = aws_iam_role.hybrid_nodes.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly"
}

resource "aws_iam_role_policy" "hybrid_eks_describe" {
  name = "eks-hybrid-permissions"
  role = aws_iam_role.hybrid_nodes.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "eks:DescribeCluster",
          "eks:ListAccessEntries",
          "eks:DescribeAccessEntry"
        ]
        Resource = "*"
      },
      {
        Effect = "Allow"
        Action = [
          "ssmmessages:CreateControlChannel",
          "ssmmessages:CreateDataChannel",
          "ssmmessages:OpenControlChannel",
          "ssmmessages:OpenDataChannel",
          "ssm:UpdateInstanceInformation",
          "ssm:ListInstanceAssociations"
        ]
        Resource = "*"
      }
    ]
  })
}

# Outputs for use in subsequent steps
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = [aws_subnet.private_a.id, aws_subnet.private_b.id]
}

output "vgw_id" {
  value = aws_vpn_gateway.main.id
}

output "hybrid_nodes_sg_id" {
  value = aws_security_group.hybrid_nodes.id
}

output "hybrid_role_arn" {
  value = aws_iam_role.hybrid_nodes.arn
}
```

Apply the Terraform:

```bash
terraform init
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars
```

Note the outputs for VPC ID, subnet IDs, VGW ID, security group ID, and hybrid role ARN.

## Part 2: Megaport Configuration

The Megaport side is configured through the portal as their Terraform provider has limited support for VXC modifications. Follow these steps in order.

### Step 2.1: Create the AWS-facing VXC

1. Log into the Megaport portal
2. Navigate to your MCR and click Add Connection
3. Select Cloud then AWS
4. Configure as follows:
   - Service: Hosted VIF (not hosted connection, to avoid Direct Connect port charges)
   - AWS Account ID: Your AWS account ID
   - AWS Region: us-east-1
   - VIF Type: Private
   - BGP Customer ASN: 4200000001 (or your chosen customer ASN)
   - VLAN: Auto-assign or specify
   - BGP Auth Key: Generate a strong key, save it securely

5. Submit and wait for the order to provision (typically 15 minutes)

### Step 2.2: Accept the hosted VIF in AWS

```bash
# List pending VIFs
aws directconnect describe-virtual-interfaces \
  --region us-east-1 \
  --query "virtualInterfaces[?virtualInterfaceState=='confirming']"

# Accept the VIF and attach to VGW
aws directconnect confirm-private-virtual-interface \
  --virtual-interface-id dxvif-xxxxxxxx \
  --virtual-gateway-id vgw-xxxxxxxx \
  --region us-east-1
```

Wait for BGP status to show as `up` before proceeding.

### Step 2.3: Configure BGP on the MCR

In the Megaport portal, edit the BGP connection settings for the AWS-facing VXC. Confirm:

- Local IP matches the customer-side IP shown in AWS (typically a /30 in 169.254.x.x range)
- Peer IP matches the AWS-side IP
- Peer ASN: 64512
- Local ASN: 4200000001
- BGP Auth Key: matches what you set during VXC creation

Also configure the static route to advertise the node CIDR to AWS:

- Prefix: `10.10.0.0/24`
- Next Hop: Must match an IP in the same subnet as the MCR interface

### Step 2.4: Create the VXC to the Latitude server

This is the critical step where the CIDR design pays off. Configure the MCR A-End with a single clean interface in the same subnet as the EKS RemoteNodeNetwork.

1. In Megaport portal, navigate to your MCR
2. Click Add Connection then Latitude.sh
3. Select your target Latitude server port
4. Configure the MCR A-End:
   - Description: e.g., "EKS hybrid node VXC"
   - Interface IP: `10.10.0.1/24` (the MCR gateway IP for the node CIDR)
   - VLAN ID: Note the value assigned, you will need this on the Latitude node side

5. Add a static route on this VXC:
   - Prefix: `10.10.0.0/24`
   - Next Hop: `10.10.0.10` (the node IP)
   - Description: Identifies this is for the EKS node

6. Submit and wait for provisioning

The key principle here is that the MCR A-End interface (10.10.0.1) and the node IP (10.10.0.10) live in the same /24 subnet, so ARP works naturally and the MCR can forward traffic from AWS back to the node without any layer 2 issues.

## Part 3: Create EKS Cluster

Save the following as `cluster.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: lsh-hybrid-demo
  region: us-east-1
  version: "1.35"

vpc:
  id: vpc-xxxxxxxx  # Replace with VPC ID from Terraform output
  subnets:
    private:
      us-east-1a:
        id: subnet-xxxxxxxx  # Replace with subnet ID from Terraform output
      us-east-1b:
        id: subnet-xxxxxxxx  # Replace with subnet ID from Terraform output

privateCluster:
  enabled: true

remoteNetworkConfig:
  remoteNodeNetworks:
    - cidrs:
        - 10.10.0.0/24
  remotePodNetworks:
    - cidrs:
        - 192.168.0.0/16

addons:
  - name: vpc-cni
  - name: kube-proxy
  - name: coredns
```

Create the cluster:

```bash
eksctl create cluster -f cluster.yaml
```

This takes 10 to 15 minutes. Once complete, attach the hybrid nodes security group:

```bash
# Get existing subnet IDs and security group from cluster
aws eks describe-cluster \
  --name lsh-hybrid-demo \
  --region us-east-1 \
  --query "cluster.resourcesVpcConfig"

# Add the hybrid nodes SG alongside the existing cluster SG
aws eks update-cluster-config \
  --name lsh-hybrid-demo \
  --region us-east-1 \
  --resources-vpc-config \
"subnetIds=SUBNET_A_ID,SUBNET_B_ID,securityGroupIds=EXISTING_SG_ID,HYBRID_NODES_SG_ID"
```

## Part 4: SSM Hybrid Activation

Create the SSM activation with sufficient registration limit to handle reinstalls:

```bash
aws ssm create-activation \
  --iam-role lsh-hybrid-demo-hybrid-role \
  --registration-limit 10 \
  --region us-east-1 \
  --description "lsh-hybrid-demo-node"
```

Note the ActivationCode and ActivationId from the output. These will be passed to nodeadm on the Latitude server.

## Part 5: EKS Access Entries

Create access entries for the hybrid role and your admin user:

```bash
# Access entry for hybrid role (allows nodes to join)
aws eks create-access-entry \
  --cluster-name lsh-hybrid-demo \
  --principal-arn arn:aws:iam::ACCOUNT_ID:role/lsh-hybrid-demo-hybrid-role \
  --type HYBRID_LINUX \
  --region us-east-1

# Access entry for admin user (for kubectl)
aws eks create-access-entry \
  --cluster-name lsh-hybrid-demo \
  --principal-arn arn:aws:iam::ACCOUNT_ID:user/YOUR_ADMIN_USERNAME \
  --region us-east-1

aws eks associate-access-policy \
  --cluster-name lsh-hybrid-demo \
  --principal-arn arn:aws:iam::ACCOUNT_ID:user/YOUR_ADMIN_USERNAME \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region us-east-1
```

## Part 6: Latitude Server Network Configuration

Run this on the Latitude bare metal server.

### Step 6.1: Configure VLAN interface

Identify your network interface:

```bash
ip link show
```

Create the netplan configuration. Save as `/etc/netplan/99-mcr-vlan.yaml`:

```yaml
network:
  version: 2
  vlans:
    vlan.VLAN_ID:
      id: VLAN_ID  # The VLAN ID from the Megaport MCR A-End config
      link: eno2  # Or whatever your interface is from ip link show
      addresses:
        - 10.10.0.10/24
      routes:
        - to: 10.0.0.0/16
          via: 10.10.0.1
        - to: 192.168.0.0/16
          via: 10.10.0.1
```

Apply the configuration:

```bash
sudo netplan apply
```

Test connectivity to the EKS control plane:

```bash
# Test reachability to MCR gateway
ping -c 3 10.10.0.1

# Test reachability to VPC
ping -c 3 10.0.0.2

# Test EKS control plane (should return 401 Unauthorized)
curl -k https://YOUR_EKS_ENDPOINT.gr7.us-east-1.eks.amazonaws.com
```

A 401 Unauthorized response confirms the private path is working end to end.

### Step 6.2: Install SSM agent

```bash
curl -Lo /tmp/amazon-ssm-agent.deb \
  https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/debian_amd64/amazon-ssm-agent.deb
sudo dpkg -i /tmp/amazon-ssm-agent.deb
```

Create the snap-named service workaround that nodeadm expects:

```bash
sudo tee /etc/systemd/system/snap.amazon-ssm-agent.amazon-ssm-agent.service << 'EOF'
[Unit]
Description=amazon-ssm-agent
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/amazon-ssm-agent
KillMode=process
Restart=on-failure
RestartSec=15min

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl stop amazon-ssm-agent
sudo rm -f /var/lib/amazon/ssm/ipc/*
sudo systemctl start amazon-ssm-agent
sleep 20
```

### Step 6.3: Install nodeadm

```bash
curl -Lo nodeadm \
  https://hybrid-assets.eks.amazonaws.com/releases/latest/bin/linux/amd64/nodeadm
chmod +x nodeadm
sudo mv nodeadm /usr/local/bin/

sudo nodeadm install 1.35 --credential-provider ssm
```

### Step 6.4: Configure and initialise the node

Save the nodeadm config:

```bash
sudo mkdir -p /etc/eks
sudo tee /etc/eks/nodeadm-config.yaml << 'EOF'
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    name: lsh-hybrid-demo
    region: us-east-1
  hybrid:
    ssm:
      activationCode: "ACTIVATION_CODE"
      activationId: "ACTIVATION_ID"
  kubelet:
    flags:
      - "--node-ip=10.10.0.10"
EOF
```

Initialise the node:

```bash
sudo nodeadm init -c file:///etc/eks/nodeadm-config.yaml
```

## Part 7: Install Cilium CNI

```bash
cilium install \
  --version 1.19.4 \
  --set nodeinit.enabled=true \
  --set nodeinit.reconfigureKubelet=true \
  --set nodeinit.removeCbrBridge=false \
  --set cni.binPath=/opt/cni/bin \
  --set operator.replicas=1 \
  --set ipam.mode=kubernetes \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan \
  --set operator.image.override=quay.io/cilium/operator-generic:v1.19.4

# Wait for Cilium to be ready
cilium status --wait
```

## Part 8: Label the Node

After the node joins the cluster:

```bash
kubectl label node NODE_INSTANCE_ID \
  topology.kubernetes.io/zone=dal-zone

kubectl label node NODE_INSTANCE_ID \
  node.latitude.sh/location=dal
```

## Part 9: Validation

Verify the deployment is healthy:

```bash
# Node should show Ready
kubectl get nodes -o wide

# All cluster-essential pods should be Running
kubectl get pods -A

# Cilium should report no errors
cilium status

# Deploy a test pod
kubectl run test-pod \
  --image=busybox \
  --restart=Never \
  --overrides='{
    "spec": {
      "containers": [{
        "name": "test-pod",
        "image": "busybox",
        "command": ["sleep", "86400"]
      }]
    }
  }'

kubectl get pod test-pod -o wide

# Exec into the pod to verify networking
kubectl exec -it test-pod -- sh
# From inside: ping 10.0.0.2 to verify VPC connectivity
```

## Teardown

To tear down the entire environment for cost savings:

```bash
# Deregister hybrid instances first
aws ssm describe-instance-information \
  --filters "Key=ResourceType,Values=ManagedInstance" \
  --region us-east-1 \
  --query "InstanceInformationList[*].InstanceId" \
  --output text | tr '\t' '\n' | while read id; do
    aws ssm deregister-managed-instance --instance-id $id --region us-east-1
done

# Delete SSM activations
aws ssm describe-activations --region us-east-1 \
  --query "ActivationList[*].ActivationId" --output text | tr '\t' '\n' | while read id; do
    aws ssm delete-activation --activation-id $id --region us-east-1
done

# Set SSM tier back to standard
aws ssm update-service-setting \
  --setting-id arn:aws:ssm:us-east-1:ACCOUNT_ID:servicesetting/ssm/managed-instance/activation-tier \
  --setting-value standard \
  --region us-east-1

# Delete EKS cluster (also removes VPC endpoints and most associated resources)
eksctl delete cluster --name lsh-hybrid-demo --region us-east-1

# Delete any remaining VPC endpoints
aws ec2 describe-vpc-endpoints \
  --region us-east-1 \
  --query "VpcEndpoints[*].VpcEndpointId" \
  --output text | tr '\t' '\n' | while read id; do
    aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $id --region us-east-1
done

# Delete Direct Connect VIFs
aws directconnect describe-virtual-interfaces \
  --region us-east-1 \
  --query "virtualInterfaces[*].virtualInterfaceId" \
  --output text | tr '\t' '\n' | while read id; do
    aws directconnect delete-virtual-interface --virtual-interface-id $id --region us-east-1
done

# Detach and delete VGW (do this after VIFs are deleted)
aws ec2 detach-vpn-gateway --vpn-gateway-id VGW_ID --vpc-id VPC_ID --region us-east-1
sleep 30
aws ec2 delete-vpn-gateway --vpn-gateway-id VGW_ID --region us-east-1

# Destroy Terraform-managed resources
terraform destroy -var-file=terraform.tfvars

# Cancel Megaport VXCs in the portal
# Navigate to each VXC and click Terminate

# Latitude server can be deprovisioned in the Latitude dashboard
```

## Cost Optimisation Notes

For demo environments that get spun up and down regularly:

- Use 50 Mbps VXCs on Megaport (the minimum tier) rather than higher bandwidths
- Use hosted VIFs not hosted connections to avoid AWS Direct Connect port charges
- Skip Transit Gateway, use VGW directly to avoid attachment hour and data processing charges
- Tear down the EKS cluster between demos as the control plane charges $0.10/hour even idle
- Latitude bare metal can be decommissioned and reprovisioned rather than kept running
- Remember to switch SSM tier back to standard, the Advanced Instances tier charges $0.00695 per instance per hour even when not actively used

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| nodeadm fails with "snap service does not exist" | nodeadm expects snap-named SSM service | Create the systemd service file as shown in Step 6.2 |
| Node registers but stays NotReady | CNI not installed | Install Cilium as shown in Part 7 |
| kubectl exec times out | Security group missing kubelet port 10250 | Verify hybrid nodes SG allows TCP 10250 from VPC CIDR |
| Pod stuck in ContainerCreating | Cilium operator crashed | Check operator logs, ensure ipam.mode=kubernetes |
| BGP session down on Direct Connect | ASN or auth key mismatch | Verify both sides match exactly, especially the auth key |
| Node IP shows as public IP not private | kubelet picking wrong interface | Add `--node-ip` flag in nodeadm config kubelet section |
| SSM activation limit exceeded | Reinstalls consumed all registrations | Create new activation with higher limit |
| ARP timeouts between MCR and node | Node IP and MCR interface in different subnets | Ensure node IP and MCR A-End interface are in the same /24 |
| kubectl exec times out despite SG allowing 10250 | MCR cannot ARP for node IP | Verify MCR A-End interface IP is in same subnet as node IP |

## Key Design Decisions

These are the choices that prevent the most common issues:

The node IP, MCR A-End interface IP, and EKS RemoteNodeNetwork CIDR all share the same /24 subnet. This is essential for layer 2 ARP to work between the MCR and the node, which is what enables kubectl exec, kubectl logs, and pod-to-control-plane streaming.

Private EKS endpoint only, no public access. All control plane traffic stays on the private Direct Connect path.

Hosted VIFs not hosted connections. This keeps AWS Direct Connect port charges off your bill since Megaport owns the underlying connection.

VGW not Direct Connect Gateway or Transit Gateway. For a single VPC deployment this is the simplest and cheapest option.

Cilium with kubernetes IPAM and VXLAN tunnel mode. This is what works reliably with EKS hybrid nodes and the operator-generic image variant.

SSM hybrid activations with high registration limit. Set to 10 or more so reinstalls during troubleshooting do not exhaust the limit.

The snap-named SSM service workaround. nodeadm explicitly checks for `snap.amazon-ssm-agent.amazon-ssm-agent.service` even on systems where SSM was installed via deb package, so the service file is required.

## References

- [EKS Hybrid Nodes documentation](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-overview.html)
- [Latitude.sh networking documentation](https://www.latitude.sh/docs/networking/private-networks)
- [Megaport MCR documentation](https://docs.megaport.com/cloud/mcr/)
- [Cilium installation for EKS](https://docs.cilium.io/en/stable/installation/k8s-install-helm/)
