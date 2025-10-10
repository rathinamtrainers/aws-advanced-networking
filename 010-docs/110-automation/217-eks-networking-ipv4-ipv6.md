# Lab Guide: Amazon EKS Networking Deep Dive - IPv4 and IPv6

## Topic: Container Networking with EKS - Complete Guide

### Prerequisites
- AWS account with EKS permissions
- kubectl installed locally
- AWS CLI configured
- eksctl installed (optional but recommended)
- Basic understanding of Kubernetes concepts
- Familiarity with VPC networking (Topics 3-10)
- Knowledge of IPv6 basics (Topics 76-85)

### Learning Objectives
By the end of this lab, you will be able to:
- Understand EKS networking architecture and AWS VPC CNI
- Deploy EKS clusters with IPv4 networking
- Configure EKS clusters with IPv6 networking
- Understand pod-to-pod, pod-to-service, and external communication
- Configure security groups for pods
- Implement network policies
- Troubleshoot EKS networking issues
- Automate EKS deployment with IaC

### Architecture Overview
This lab covers:
- **VPC CNI Plugin**: How EKS assigns IP addresses to pods
- **IPv4 Mode**: Traditional pod networking using secondary IPs
- **IPv6 Mode**: Modern IPv6-only and dual-stack configurations
- **Service Networking**: ClusterIP, NodePort, LoadBalancer
- **Security Groups for Pods**: Granular security control
- **Network Policies**: Calico/Cilium integration

### Lab Duration
Estimated time: 120-150 minutes

---

## Part 1: EKS Networking Fundamentals

### Understanding AWS VPC CNI

**Key Concepts**:
- **VPC CNI Plugin**: AWS-native CNI that assigns VPC IP addresses to pods
- **ENI**: Each worker node gets Elastic Network Interfaces
- **Secondary IPs**: Pods receive secondary IP addresses from the VPC
- **WARM Pool**: Pre-allocated IPs for faster pod scheduling
- **SNAT**: Source NAT for pod-to-external communication

**IP Address Assignment**:
```
Worker Node (t3.medium example):
├── Primary ENI
│   ├── Primary IP: 10.0.1.50 (node IP)
│   └── Secondary IPs: 10.0.1.51-10.0.1.57 (7 IPs for pods)
├── Secondary ENI (if needed)
│   └── Secondary IPs: 10.0.1.58-10.0.1.64 (7 more IPs)
└── Tertiary ENI (if needed)
    └── Secondary IPs: 10.0.1.65-10.0.1.71
```

**ENI Limits by Instance Type**:
| Instance Type | Max ENIs | IPs per ENI | Max Pods |
|---------------|----------|-------------|----------|
| t3.small      | 3        | 4           | 11       |
| t3.medium     | 3        | 6           | 17       |
| t3.large      | 3        | 12          | 35       |
| m5.large      | 3        | 10          | 29       |
| m5.xlarge     | 4        | 15          | 58       |

**Formula**: Max Pods = (ENIs × IPs per ENI) - 1 (node IP)

---

## Part 2: EKS Cluster with IPv4 Networking

### Step 1: Create VPC for EKS (IPv4)

**Objective**: Set up VPC infrastructure for EKS cluster

**Theory**: EKS requires specific VPC configuration with public and private subnets across at least 2 AZs.

**AWS Console Steps**:
1. Navigate to **VPC Console** → **Create VPC**
2. Select **VPC and more** (creates subnets, route tables, gateways)
3. Configure:
   - **Name tag**: `eks-ipv4-vpc`
   - **IPv4 CIDR**: `10.0.0.0/16`
   - **IPv6 CIDR**: None (IPv4 only for now)
   - **Number of AZs**: 2
   - **Number of public subnets**: 2
   - **Number of private subnets**: 2
   - **NAT gateways**: 1 per AZ
   - **VPC endpoints**: S3 Gateway
4. Click **Create VPC**

**Required Tags for EKS**:

After VPC creation, add tags to subnets:

**Public Subnets**:
```
kubernetes.io/role/elb = 1
kubernetes.io/cluster/eks-ipv4-cluster = shared
```

**Private Subnets**:
```
kubernetes.io/role/internal-elb = 1
kubernetes.io/cluster/eks-ipv4-cluster = shared
```

**Add tags via Console**:
1. Go to **Subnets** → Select each public subnet
2. **Tags** tab → **Manage tags**
3. Add the tags above
4. Repeat for private subnets

**Verification via AWS CLI**:
```bash
# Set VPC ID
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=eks-ipv4-vpc" --query 'Vpcs[0].VpcId' --output text)

# Verify subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`kubernetes.io/role/elb`].Value | [0]]' --output table
```

---

### Step 2: Create IAM Role for EKS Cluster

**Objective**: Create IAM role that EKS control plane will assume

**AWS Console Steps**:
1. Navigate to **IAM Console** → **Roles** → **Create role**
2. **Trusted entity type**: AWS service
3. **Use case**: EKS → **EKS - Cluster**
4. Click **Next**
5. **Permissions** (auto-attached):
   - `AmazonEKSClusterPolicy`
6. **Role name**: `eks-cluster-role`
7. Click **Create role**

**Verification via AWS CLI**:
```bash
# Verify role exists
aws iam get-role --role-name eks-cluster-role --query 'Role.[RoleName,Arn]' --output table

# Verify attached policies
aws iam list-attached-role-policies --role-name eks-cluster-role --output table
```

---

### Step 3: Create EKS Cluster (IPv4)

**Objective**: Deploy the EKS control plane

**Theory**: EKS control plane runs in AWS-managed VPC. Worker nodes run in your VPC and register to control plane.

**AWS Console Steps**:
1. Navigate to **EKS Console** → **Clusters** → **Create cluster**
2. **Name**: `eks-ipv4-cluster`
3. **Kubernetes version**: 1.28 (latest)
4. **Cluster service role**: `eks-cluster-role`
5. Click **Next**

**Networking Configuration**:
6. **VPC**: Select `eks-ipv4-vpc`
7. **Subnets**: Select **both public and both private** subnets
8. **Security groups**: Leave default (auto-created)
9. **Cluster endpoint access**: Public and private
10. **Networking add-ons**:
    - **Amazon VPC CNI**: v1.15.x (latest)
    - **kube-proxy**: v1.28.x
    - **CoreDNS**: v1.10.x
11. Click **Next**

**Observability**:
12. **Control plane logging**: Enable all log types (API, Audit, Authenticator, Controller Manager, Scheduler)
13. Click **Next**

**Add-ons**:
14. Keep default versions
15. Click **Next**

**Review and Create**:
16. Review configuration
17. Click **Create**

**Wait for cluster creation** (10-15 minutes)

**Verification via AWS CLI**:
```bash
# Check cluster status
aws eks describe-cluster --name eks-ipv4-cluster --query 'cluster.[name,status,version,endpoint]' --output table

# Get cluster details
aws eks describe-cluster --name eks-ipv4-cluster > eks-cluster-config.json
```

**Configure kubectl**:
```bash
# Update kubeconfig
aws eks update-kubeconfig --name eks-ipv4-cluster --region us-east-1

# Verify connection
kubectl get svc
# Should show kubernetes ClusterIP service
```

---

### Step 4: Create IAM Role for Worker Nodes

**Objective**: Create IAM role for EC2 worker nodes

**AWS Console Steps**:
1. **IAM Console** → **Roles** → **Create role**
2. **Trusted entity**: AWS service → **EC2**
3. Click **Next**
4. Attach policies:
   - `AmazonEKSWorkerNodePolicy`
   - `AmazonEKS_CNI_Policy`
   - `AmazonEC2ContainerRegistryReadOnly`
5. **Role name**: `eks-node-role`
6. Click **Create role**

**Verification via AWS CLI**:
```bash
# Verify role and policies
aws iam get-role --role-name eks-node-role --query 'Role.Arn'
aws iam list-attached-role-policies --role-name eks-node-role --output table
```

---

### Step 5: Create EKS Node Group (IPv4)

**Objective**: Add worker nodes to the cluster

**Theory**: Node groups are auto-scaling groups of EC2 instances running as Kubernetes workers.

**AWS Console Steps**:
1. Go to **EKS Console** → **Clusters** → **eks-ipv4-cluster** → **Compute** tab
2. Click **Add node group**
3. **Name**: `eks-ipv4-ng`
4. **Node IAM role**: `eks-node-role`
5. Click **Next**

**Compute Configuration**:
6. **AMI type**: Amazon Linux 2 (AL2_x86_64)
7. **Capacity type**: On-Demand
8. **Instance types**: t3.medium
9. **Disk size**: 20 GB
10. Click **Next**

**Scaling Configuration**:
11. **Desired size**: 2
12. **Minimum size**: 2
13. **Maximum size**: 4
14. Click **Next**

**Networking**:
15. **Subnets**: Select **private subnets only**
16. **Configure remote access**: Optional (enable SSH if needed)
17. Click **Next**

**Review and Create**:
18. Review → **Create**

**Wait for node group creation** (5-10 minutes)

**Verification via AWS CLI**:
```bash
# Check node group status
aws eks describe-nodegroup --cluster-name eks-ipv4-cluster --nodegroup-name eks-ipv4-ng --query 'nodegroup.[nodegroupName,status,scalingConfig]' --output table

# Verify nodes in Kubernetes
kubectl get nodes -o wide

# Expected output: 2 nodes in Ready state
```

**Check VPC CNI Configuration**:
```bash
# Get VPC CNI daemonset
kubectl get daemonset -n kube-system aws-node

# Check CNI configuration
kubectl describe daemonset -n kube-system aws-node | grep -A 5 "Environment"

# View CNI logs
kubectl logs -n kube-system -l k8s-app=aws-node --tail=50
```

---

### Step 6: Deploy Sample Application (IPv4)

**Objective**: Test pod networking and service exposure

**Create deployment YAML**:

**File: `nginx-deployment-ipv4.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ipv4
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**Deploy application**:
```bash
# Apply deployment
kubectl apply -f nginx-deployment-ipv4.yaml

# Wait for pods
kubectl get pods -l app=nginx -o wide

# Check pod IPs (should be from VPC CIDR)
kubectl get pods -l app=nginx -o custom-columns=NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName
```

**Verification**:
```bash
# Get service details
kubectl get svc nginx-service

# Get LoadBalancer DNS
export LB_DNS=$(kubectl get svc nginx-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test external access (wait 2-3 minutes for LB provisioning)
curl http://$LB_DNS

# Should return nginx welcome page
```

**Check pod-to-pod communication**:
```bash
# Get pod names
POD1=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
POD2=$(kubectl get pods -l app=nginx -o jsonpath='{.items[1].metadata.name}')
POD2_IP=$(kubectl get pod $POD2 -o jsonpath='{.status.podIP}')

# Test connectivity from pod1 to pod2
kubectl exec -it $POD1 -- curl http://$POD2_IP

# Should succeed (pods in same VPC can communicate directly)
```

---

### Step 7: Verify IPv4 Networking

**Check ENI allocation**:
```bash
# Get node instance IDs
NODE_INSTANCE_IDS=$(kubectl get nodes -o jsonpath='{.items[*].spec.providerID}' | sed 's/aws:\/\/\/us-east-1[a-z]\///g')

# Check ENIs for first node
FIRST_INSTANCE=$(echo $NODE_INSTANCE_IDS | awk '{print $1}')
aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=$FIRST_INSTANCE" --query 'NetworkInterfaces[].[NetworkInterfaceId,PrivateIpAddresses[].PrivateIpAddress,Description]' --output table
```

**Check IP allocation**:
```bash
# View VPC CNI IPAM
kubectl get pods -n kube-system -l k8s-app=aws-node -o name | xargs -I {} kubectl exec -n kube-system {} -- sh -c 'aws-k8s-agent introspect eni' 2>/dev/null | head -50

# Alternative: Check using AWS CNI metrics helper
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/cni-metrics-helper.yaml

# Get pod IPs and verify they're from VPC CIDR
kubectl get pods --all-namespaces -o wide | grep -v "kube-system\|kube-public"
```

---

## Part 3: EKS Cluster with IPv6 Networking

### Step 8: Create VPC for EKS (IPv6)

**Objective**: Set up dual-stack VPC for IPv6-enabled EKS

**Theory**: EKS IPv6 mode assigns IPv6 addresses to pods, solving IP exhaustion issues in large clusters.

**AWS Console Steps**:
1. **VPC Console** → **Create VPC**
2. Select **VPC and more**
3. Configure:
   - **Name**: `eks-ipv6-vpc`
   - **IPv4 CIDR**: `10.1.0.0/16`
   - **IPv6 CIDR**: Amazon-provided IPv6 CIDR
   - **Number of AZs**: 2
   - **Public subnets**: 2
   - **Private subnets**: 2
   - **NAT gateways**: None (IPv6 doesn't need NAT)
   - **Egress-only internet gateway**: Yes
4. Click **Create VPC**

**Configure IPv6 for Subnets**:

After creation, edit each subnet:
1. Select subnet → **Actions** → **Edit subnet settings**
2. **Auto-assign IPv6 address**: Enable
3. Save

**Add EKS tags** (same as IPv4 section):
```bash
# Get VPC ID
export VPC_ID_V6=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=eks-ipv6-vpc" --query 'Vpcs[0].VpcId' --output text)

# Get subnet IDs
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID_V6" --query 'Subnets[].[SubnetId,Tags[?Key==`Name`].Value | [0],CidrBlock,Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock]' --output table
```

**Tag subnets via CLI**:
```bash
# Get public subnet IDs
PUBLIC_SUBNET_1=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID_V6" "Name=tag:Name,Values=*public*" --query 'Subnets[0].SubnetId' --output text)
PUBLIC_SUBNET_2=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID_V6" "Name=tag:Name,Values=*public*" --query 'Subnets[1].SubnetId' --output text)

# Tag public subnets
aws ec2 create-tags --resources $PUBLIC_SUBNET_1 $PUBLIC_SUBNET_2 --tags Key=kubernetes.io/role/elb,Value=1 Key=kubernetes.io/cluster/eks-ipv6-cluster,Value=shared

# Get and tag private subnets
PRIVATE_SUBNET_1=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID_V6" "Name=tag:Name,Values=*private*" --query 'Subnets[0].SubnetId' --output text)
PRIVATE_SUBNET_2=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID_V6" "Name=tag:Name,Values=*private*" --query 'Subnets[1].SubnetId' --output text)

aws ec2 create-tags --resources $PRIVATE_SUBNET_1 $PRIVATE_SUBNET_2 --tags Key=kubernetes.io/role/internal-elb,Value=1 Key=kubernetes.io/cluster/eks-ipv6-cluster,Value=shared
```

---

### Step 9: Create EKS Cluster (IPv6)

**AWS Console Steps**:
1. **EKS Console** → **Create cluster**
2. **Name**: `eks-ipv6-cluster`
3. **Kubernetes version**: 1.28
4. **Cluster service role**: `eks-cluster-role` (reuse from Step 2)
5. Click **Next**

**Networking**:
6. **VPC**: `eks-ipv6-vpc`
7. **Subnets**: Select all 4 subnets
8. **IP family**: **IPv6** (Important!)
9. **Security groups**: Default
10. **Cluster endpoint access**: Public and private
11. Click **Next**
12. Enable control plane logging → **Next**
13. Review add-ons → **Next**
14. **Create**

**Wait for cluster creation** (10-15 minutes)

**Configure kubectl**:
```bash
# Update kubeconfig
aws eks update-kubeconfig --name eks-ipv6-cluster --region us-east-1

# Switch context
kubectl config use-context arn:aws:eks:us-east-1:ACCOUNT_ID:cluster/eks-ipv6-cluster

# Verify
kubectl get svc
```

---

### Step 10: Create Node Group (IPv6)

**AWS Console Steps**:
1. **eks-ipv6-cluster** → **Compute** → **Add node group**
2. **Name**: `eks-ipv6-ng`
3. **Node IAM role**: `eks-node-role`
4. **Next**

**Compute**:
5. **AMI**: Amazon Linux 2
6. **Instance type**: t3.medium
7. **Disk size**: 20 GB
8. **Next**

**Scaling**:
9. **Desired**: 2, **Min**: 2, **Max**: 4
10. **Next**

**Networking**:
11. **Subnets**: Private subnets
12. **Next** → **Create**

**Verification**:
```bash
# Wait for nodes
kubectl get nodes -o wide

# Check node IPv6 addresses
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
```

---

### Step 11: Deploy Application (IPv6)

**File: `nginx-deployment-ipv6.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ipv6
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-ipv6
  template:
    metadata:
      labels:
        app: nginx-ipv6
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ipv6-service
spec:
  type: LoadBalancer
  ipFamilies:
  - IPv6
  ipFamilyPolicy: SingleStack
  selector:
    app: nginx-ipv6
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

**Deploy**:
```bash
kubectl apply -f nginx-deployment-ipv6.yaml

# Get pods with IPv6 addresses
kubectl get pods -l app=nginx-ipv6 -o custom-columns=NAME:.metadata.name,IPv6:.status.podIP
```

**Test connectivity**:
```bash
# Get service
kubectl get svc nginx-ipv6-service

# Test from pod
POD_NAME=$(kubectl get pods -l app=nginx-ipv6 -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -- curl -6 -v http://nginx-ipv6-service
```

---

## Part 4: Advanced VPC CNI Configuration

### Step 12: Configure Custom Networking

**Objective**: Use different subnets for pods than nodes (useful for IP conservation)

**Theory**: Custom networking allows pods to use secondary CIDR ranges or different subnets.

**Enable custom networking**:
```bash
# Set ENI config
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true

# Disable SNAT (if using custom networking with public subnets)
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_EXTERNALSNAT=true
```

**Create ENIConfig for each AZ**:

**File: `eniconfig-us-east-1a.yaml`**
```yaml
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1a
spec:
  subnet: subnet-xxxxxxxxx  # Secondary subnet in AZ 1a
  securityGroups:
  - sg-xxxxxxxxx  # Pod security group
```

**File: `eniconfig-us-east-1b.yaml`**
```yaml
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: us-east-1b
spec:
  subnet: subnet-yyyyyyyyy  # Secondary subnet in AZ 1b
  securityGroups:
  - sg-xxxxxxxxx
```

**Apply ENIConfigs**:
```bash
kubectl apply -f eniconfig-us-east-1a.yaml
kubectl apply -f eniconfig-us-east-1b.yaml

# Verify
kubectl get eniconfigs
```

---

### Step 13: Security Groups for Pods

**Objective**: Assign specific security groups to pods for granular control

**Theory**: Instead of node-level security groups, assign SGs directly to pods for micro-segmentation.

**Enable SG for Pods**:
```bash
# Enable feature
kubectl set env daemonset aws-node -n kube-system ENABLE_POD_ENI=true

# Install VPC Resource Controller
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-resource-controller-k8s/master/config/crd/bases/vpcresources.k8s.aws_securitygrouppolicies.yaml
```

**Create Security Group**:
```bash
# Create SG for database pods
aws ec2 create-security-group \
  --group-name eks-database-pods \
  --description "Security group for database pods" \
  --vpc-id $VPC_ID

# Get SG ID
DB_SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=eks-database-pods" --query 'SecurityGroups[0].GroupId' --output text)

# Add ingress rule (allow MySQL from app pods)
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG_ID \
  --protocol tcp \
  --port 3306 \
  --source-group $APP_SG_ID
```

**Create SecurityGroupPolicy**:

**File: `sg-policy-database.yaml`**
```yaml
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: database-pods-sg
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: database
  securityGroups:
    groupIds:
    - sg-xxxxxxxxx  # Your database SG ID
```

**Apply and test**:
```bash
kubectl apply -f sg-policy-database.yaml

# Deploy pod with label
kubectl run mysql --image=mysql:8.0 --labels="role=database" --env="MYSQL_ROOT_PASSWORD=password"

# Verify pod has security group
kubectl describe pod mysql | grep "vpc.amazonaws.com/pod-eni"
```

---

### Step 14: Configure Prefix Delegation

**Objective**: Increase pod density by using /28 prefix delegation instead of individual IPs

**Theory**: Prefix delegation assigns /28 blocks (16 IPs) per ENI instead of individual secondary IPs, drastically increasing pod capacity.

**Enable prefix delegation**:
```bash
# Enable prefix mode
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# Optionally set warm prefix target
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1

# Restart daemonset
kubectl rollout restart daemonset aws-node -n kube-system
```

**Verification**:
```bash
# Check node capacity (should increase significantly)
kubectl get nodes -o custom-columns=NAME:.metadata.name,CAPACITY:.status.capacity.pods,ALLOCATABLE:.status.allocatable.pods

# Example output for t3.medium:
# Before: 17 pods
# After: 110 pods (with prefix delegation)
```

**Check ENI prefixes**:
```bash
# Get node instance
INSTANCE_ID=$(kubectl get nodes -o jsonpath='{.items[0].spec.providerID}' | cut -d'/' -f5)

# Check assigned prefixes
aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=$INSTANCE_ID" --query 'NetworkInterfaces[].Ipv4Prefixes' --output table
```

---

## Part 5: Network Policies

### Step 15: Install Calico for Network Policies

**Objective**: Implement pod-to-pod traffic controls

**Theory**: EKS doesn't enforce NetworkPolicy by default. Calico provides network policy enforcement.

**Install Calico**:
```bash
# Install Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Install Calico CRDs
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

# Wait for Calico pods
kubectl get pods -n calico-system --watch
```

**Create Network Policy**:

**File: `network-policy-deny-all.yaml`**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

**File: `network-policy-allow-nginx.yaml`**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 80
```

**Test network policy**:
```bash
# Apply deny-all
kubectl apply -f network-policy-deny-all.yaml

# Test connectivity (should fail)
kubectl run test-pod --image=busybox --rm -it -- wget -O- http://nginx-service --timeout=5

# Apply allow policy
kubectl apply -f network-policy-allow-nginx.yaml

# Add label to test pod
kubectl run test-pod --image=busybox --labels="role=frontend" --rm -it -- wget -O- http://nginx-service

# Should succeed
```

---

## Part 6: Monitoring and Troubleshooting

### Step 16: Monitor CNI Metrics

**Deploy CNI metrics helper**:
```bash
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/cni-metrics-helper.yaml

# View metrics
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq .
```

**Check CloudWatch metrics**:
```bash
# Get CNI metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EKS \
  --metric-name pod_eni_count \
  --dimensions Name=ClusterName,Value=eks-ipv4-cluster \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

### Step 17: Troubleshooting Commands

**Check pod networking issues**:
```bash
# Get pod details
kubectl describe pod <pod-name>

# Check CNI logs
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100

# Check IP allocation
kubectl exec -n kube-system <aws-node-pod> -- /app/aws-k8s-agent introspect eni

# Check ipamd logs
kubectl logs -n kube-system <aws-node-pod> -c aws-node | grep -i error

# Verify ENIs on node
NODE_INSTANCE=$(kubectl get node <node-name> -o jsonpath='{.spec.providerID}' | cut -d'/' -f5)
aws ec2 describe-instances --instance-ids $NODE_INSTANCE --query 'Reservations[].Instances[].NetworkInterfaces[].[NetworkInterfaceId,PrivateIpAddresses]'
```

**Common issues**:
```bash
# Issue 1: Insufficient IPs in subnet
# Solution: Add secondary CIDR or use custom networking

# Issue 2: Pod stuck in ContainerCreating
kubectl describe pod <pod-name> | grep -A 10 Events
# Check if CNI failed to allocate IP

# Issue 3: Pods can't reach internet
# Check NAT Gateway or egress-only gateway (IPv6)
kubectl run test --image=busybox --rm -it -- ping -c 3 8.8.8.8
```

---

## Part 7: Infrastructure as Code (Terraform)

### Terraform EKS with IPv4

**File: `eks-ipv4.tf`**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "eks-ipv4-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = false
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }

  tags = {
    Environment = "dev"
    Terraform   = "true"
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "eks-ipv4-cluster"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  # Enable IRSA
  enable_irsa = true

  # Cluster addons
  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
      configuration_values = jsonencode({
        env = {
          ENABLE_PREFIX_DELEGATION = "true"
          WARM_PREFIX_TARGET       = "1"
        }
      })
    }
  }

  # Managed node group
  eks_managed_node_groups = {
    default = {
      min_size     = 2
      max_size     = 4
      desired_size = 2

      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"

      labels = {
        Environment = "dev"
      }

      tags = {
        ExtraTag = "EKS-managed-node-group"
      }
    }
  }

  tags = {
    Environment = "dev"
    Terraform   = "true"
  }
}
```

**Deploy**:
```bash
terraform init
terraform plan
terraform apply

# Configure kubectl
aws eks update-kubeconfig --name eks-ipv4-cluster --region us-east-1
```

---

### Terraform EKS with IPv6

**File: `eks-ipv6.tf`**
```hcl
# VPC with IPv6
resource "aws_vpc" "eks_ipv6" {
  cidr_block                       = "10.1.0.0/16"
  assign_generated_ipv6_cidr_block = true
  enable_dns_support               = true
  enable_dns_hostnames             = true

  tags = {
    Name = "eks-ipv6-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.eks_ipv6.id
}

# Egress-only gateway for IPv6
resource "aws_egress_only_internet_gateway" "main" {
  vpc_id = aws_vpc.eks_ipv6.id
}

# Public Subnets (dual-stack)
resource "aws_subnet" "public" {
  count = 2

  vpc_id                          = aws_vpc.eks_ipv6.id
  cidr_block                      = cidrsubnet(aws_vpc.eks_ipv6.cidr_block, 8, count.index)
  ipv6_cidr_block                 = cidrsubnet(aws_vpc.eks_ipv6.ipv6_cidr_block, 8, count.index)
  availability_zone               = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch         = true
  assign_ipv6_address_on_creation = true

  tags = {
    Name                     = "eks-ipv6-public-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"
  }
}

# Private Subnets (dual-stack)
resource "aws_subnet" "private" {
  count = 2

  vpc_id                          = aws_vpc.eks_ipv6.id
  cidr_block                      = cidrsubnet(aws_vpc.eks_ipv6.cidr_block, 8, count.index + 10)
  ipv6_cidr_block                 = cidrsubnet(aws_vpc.eks_ipv6.ipv6_cidr_block, 8, count.index + 10)
  availability_zone               = data.aws_availability_zones.available.names[count.index]
  assign_ipv6_address_on_creation = true

  tags = {
    Name                              = "eks-ipv6-private-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"
  }
}

# EKS Cluster with IPv6
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "eks-ipv6-cluster"
  cluster_version = "1.28"

  vpc_id     = aws_vpc.eks_ipv6.id
  subnet_ids = concat(aws_subnet.public[*].id, aws_subnet.private[*].id)

  cluster_ip_family = "ipv6"

  cluster_addons = {
    vpc-cni = {
      most_recent = true
      configuration_values = jsonencode({
        env = {
          ENABLE_IPv6 = "true"
        }
      })
    }
  }

  eks_managed_node_groups = {
    ipv6_ng = {
      min_size     = 2
      max_size     = 4
      desired_size = 2

      instance_types = ["t3.medium"]
      subnet_ids     = aws_subnet.private[*].id
    }
  }
}
```

---

## Part 8: AWS CDK (Python) for EKS

**File: `eks_stack.py`**
```python
from aws_cdk import (
    Stack,
    aws_ec2 as ec2,
    aws_eks as eks,
    aws_iam as iam,
)
from constructs import Construct

class EksStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        # VPC
        vpc = ec2.Vpc(
            self,
            "EksVpc",
            vpc_name="eks-cdk-vpc",
            ip_addresses=ec2.IpAddresses.cidr("10.0.0.0/16"),
            max_azs=2,
            nat_gateways=2,
            subnet_configuration=[
                ec2.SubnetConfiguration(
                    name="Public",
                    subnet_type=ec2.SubnetType.PUBLIC,
                    cidr_mask=24,
                ),
                ec2.SubnetConfiguration(
                    name="Private",
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS,
                    cidr_mask=24,
                ),
            ],
        )

        # EKS Cluster
        cluster = eks.Cluster(
            self,
            "EksCluster",
            cluster_name="eks-cdk-cluster",
            version=eks.KubernetesVersion.V1_28,
            vpc=vpc,
            vpc_subnets=[ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS)],
            default_capacity=0,  # We'll add managed node group separately
        )

        # Managed Node Group
        cluster.add_nodegroup_capacity(
            "ManagedNodeGroup",
            instance_types=[ec2.InstanceType("t3.medium")],
            min_size=2,
            max_size=4,
            desired_size=2,
            disk_size=20,
            subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PRIVATE_WITH_EGRESS),
        )

        # Enable prefix delegation
        cluster.aws_auth.add_account(self.account)
```

**Deploy**:
```bash
cdk bootstrap
cdk deploy

# Configure kubectl
aws eks update-kubeconfig --name eks-cdk-cluster
```

---

## Cleanup

### Delete EKS Resources

```bash
# Delete Kubernetes resources
kubectl delete svc nginx-service nginx-ipv6-service
kubectl delete deployment nginx-ipv4 nginx-ipv6

# Delete node groups (via Console or CLI)
aws eks delete-nodegroup --cluster-name eks-ipv4-cluster --nodegroup-name eks-ipv4-ng
aws eks delete-nodegroup --cluster-name eks-ipv6-cluster --nodegroup-name eks-ipv6-ng

# Wait for node groups to delete (5-10 minutes)
aws eks wait nodegroup-deleted --cluster-name eks-ipv4-cluster --nodegroup-name eks-ipv4-ng

# Delete clusters
aws eks delete-cluster --name eks-ipv4-cluster
aws eks delete-cluster --name eks-ipv6-cluster

# Delete VPCs (via Console or manually delete dependencies)

# If using Terraform
terraform destroy

# If using CDK
cdk destroy
```

---

## Key Takeaways

### IPv4 vs IPv6 in EKS

| Feature | IPv4 | IPv6 |
|---------|------|------|
| **IP Assignment** | Secondary IPs from VPC | IPv6 addresses from /64 block |
| **Scalability** | Limited by VPC CIDR size | Virtually unlimited |
| **NAT Required** | Yes (NAT Gateway for private subnets) | No (egress-only gateway) |
| **Cost** | NAT Gateway costs | Lower (no NAT) |
| **Prefix Delegation** | /28 blocks (16 IPs) | /80 blocks (many IPs) |
| **Pod Density** | Limited by ENI/IP limits | Much higher |
| **Internet Access** | Via NAT or IGW | Direct via egress-only gateway |
| **LoadBalancer Type** | NLB/CLB (IPv4) | NLB (dualstack or IPv6) |

### VPC CNI Best Practices

1. **Use prefix delegation** for high pod density
2. **Enable custom networking** to separate pod and node subnets
3. **Implement security groups for pods** for micro-segmentation
4. **Monitor IP exhaustion** using CNI metrics
5. **Use IPv6** for large clusters to avoid IP exhaustion
6. **Configure warm pool settings** based on scaling patterns
7. **Enable network policies** for pod-to-pod security

### Common Pitfalls

1. **Subnet IP exhaustion**: Provision large enough subnets
2. **ENI limits**: Choose instance types with sufficient ENI capacity
3. **Security group limits**: Consolidate rules, use SG for pods
4. **LoadBalancer costs**: Use Ingress controllers for HTTP workloads
5. **Network policies**: Don't forget to install Calico/Cilium

---

## Additional Resources

- [AWS VPC CNI Plugin](https://github.com/aws/amazon-vpc-cni-k8s)
- [EKS Best Practices - Networking](https://aws.github.io/aws-eks-best-practices/networking/)
- [Prefix Delegation](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)
- [Security Groups for Pods](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html)
- [EKS IPv6](https://docs.aws.amazon.com/eks/latest/userguide/cni-ipv6.html)

---

## Exam Tips

1. **VPC CNI is AWS-specific**: Uses actual VPC IPs for pods
2. **IP planning crucial**: Calculate max pods based on instance type and ENI limits
3. **Prefix delegation**: Dramatically increases pod density on supported instances
4. **IPv6 benefits**: Solves IP exhaustion, no NAT costs
5. **Security groups for pods**: Enables application-level security policies
6. **Custom networking**: Use when you need pods in different subnets than nodes
7. **Network policies**: Require Calico/Cilium, not enabled by default
