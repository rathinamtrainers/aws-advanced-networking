# Topic 33: Advanced VPC Design Patterns and Architectures

## Prerequisites
- Completed Topic 32 (VPC Automation and Infrastructure as Code)
- Understanding of enterprise network design principles and patterns
- Knowledge of scalability, availability, and security requirements
- Familiarity with AWS networking services and architectural best practices

## Learning Objectives
By the end of this topic, you will be able to:
- Design enterprise-grade VPC architectures using proven patterns
- Implement multi-tier, multi-region, and multi-account VPC designs
- Apply advanced networking patterns for scalability and high availability
- Optimize VPC architectures for performance, security, and cost

## Theory

### Enterprise VPC Design Principles

#### Design Fundamentals
- **Scalability**: Design for growth and changing requirements
- **Availability**: Ensure high availability across multiple AZs and regions
- **Security**: Implement defense-in-depth with network segmentation
- **Performance**: Optimize for low latency and high throughput
- **Cost Optimization**: Balance requirements with cost-effective solutions
- **Operational Excellence**: Design for maintainability and automation

#### Network Design Considerations
```
Enterprise Network Design Stack:
┌─────────────────────────────────────────────────────┐
│ Business Requirements (Compliance, Performance)    │
├─────────────────────────────────────────────────────┤
│ Application Architecture (Microservices, Monolith) │
├─────────────────────────────────────────────────────┤
│ Network Segmentation (Security Zones, Tiers)       │
├─────────────────────────────────────────────────────┤
│ Connectivity (Hybrid, Multi-Cloud, Internet)       │
├─────────────────────────────────────────────────────┤
│ Infrastructure Services (DNS, Monitoring, Backup)  │
└─────────────────────────────────────────────────────┘
```

### Pattern 1: Multi-Tier Application Architecture

#### Traditional Three-Tier Pattern
```
Internet Gateway
       │
┌──────┴──────┐
│ Web Tier    │ (Public Subnets)
│ - ALB       │ Port 80/443 from 0.0.0.0/0
│ - Web Svrs  │ 
└──────┬──────┘
       │
┌──────┴──────┐
│ App Tier    │ (Private Subnets)
│ - App Svrs  │ Port 8080 from Web Tier
│ - API Gway  │ 
└──────┬──────┘
       │
┌──────┴──────┐
│ Data Tier   │ (Database Subnets)
│ - RDS       │ Port 3306/5432 from App Tier
│ - ElastiC   │ 
└─────────────┘
```

#### Modern Microservices Pattern
```
                Internet Gateway
                       │
          ┌────────────┼────────────┐
          │                        │
    ┌─────┴─────┐            ┌─────┴─────┐
    │ Public    │            │ Public    │
    │ Subnet 1  │            │ Subnet 2  │
    │ - ALB     │            │ - ALB     │ 
    │ - NLB     │            │ - NLB     │
    └─────┬─────┘            └─────┬─────┘
          │                        │
    ┌─────┴─────┐            ┌─────┴─────┐
    │ Private   │            │ Private   │
    │ Subnet 1  │            │ Subnet 2  │
    │ - ECS     │            │ - ECS     │
    │ - EKS     │            │ - EKS     │
    │ - Lambda  │            │ - Lambda  │
    └─────┬─────┘            └─────┬─────┘
          │                        │
    ┌─────┴─────┐            ┌─────┴─────┐
    │ Database  │            │ Database  │
    │ Subnet 1  │            │ Subnet 2  │
    │ - RDS     │            │ - RDS     │
    │ - DynamoDB│            │ - DynamoDB│
    │ - ElastiC │            │ - ElastiC │
    └───────────┘            └───────────┘
```

### Pattern 2: Hub-and-Spoke Architecture

#### Centralized Services Hub
```
                    Shared Services VPC (Hub)
                    ┌─────────────────────┐
                    │ - DNS (Route53)     │
                    │ - Monitoring        │
                    │ - Security Tools    │
                    │ - AD/LDAP          │
                    │ - CI/CD            │
                    └──────────┬──────────┘
                               │
                    Transit Gateway
            ┌──────────────────┼──────────────────┐
            │                  │                  │
    ┌───────┴────────┐ ┌───────┴────────┐ ┌───────┴────────┐
    │ Production VPC │ │ Staging VPC    │ │Development VPC │
    │ (Spoke)        │ │ (Spoke)        │ │ (Spoke)        │
    │ - Web Apps     │ │ - Web Apps     │ │ - Web Apps     │
    │ - Databases    │ │ - Databases    │ │ - Databases    │
    │ - Microsvcs    │ │ - Microsvcs    │ │ - Microsvcs    │
    └────────────────┘ └────────────────┘ └────────────────┘
```

#### Route Table Design for Hub-and-Spoke
```
Hub Route Table:
┌─────────────────┬──────────────────┬─────────────────┐
│ Destination     │ Target           │ Purpose         │
├─────────────────┼──────────────────┼─────────────────┤
│ 10.0.0.0/16    │ local            │ Hub VPC local   │
│ 10.1.0.0/16    │ tgw-attachment   │ Production VPC  │
│ 10.2.0.0/16    │ tgw-attachment   │ Staging VPC     │
│ 10.3.0.0/16    │ tgw-attachment   │ Development VPC │
│ 192.168.0.0/16 │ vpn-attachment   │ On-premises     │
│ 0.0.0.0/0      │ igw              │ Internet access │
└─────────────────┴──────────────────┴─────────────────┘

Spoke Route Table (Production):
┌─────────────────┬──────────────────┬─────────────────┐
│ Destination     │ Target           │ Purpose         │
├─────────────────┼──────────────────┼─────────────────┤
│ 10.1.0.0/16    │ local            │ Production local│
│ 10.0.0.0/16    │ tgw-attachment   │ Hub services    │
│ 192.168.0.0/16 │ tgw-attachment   │ On-premises     │
│ 0.0.0.0/0      │ tgw-attachment   │ Internet via Hub│
└─────────────────┴──────────────────┴─────────────────┘
```

### Pattern 3: Multi-Region Architecture

#### Active-Active Multi-Region Design
```
Region A (us-west-2)              Region B (us-east-1)
┌─────────────────────┐          ┌─────────────────────┐
│     Primary VPC     │          │    Secondary VPC    │
│                     │          │                     │
│ ┌─────────────────┐ │          │ ┌─────────────────┐ │
│ │  Web Tier       │ │          │ │  Web Tier       │ │
│ │  - ALB (50%)    │ │          │ │  - ALB (50%)    │ │
│ └─────────────────┘ │          │ └─────────────────┘ │
│ ┌─────────────────┐ │          │ ┌─────────────────┐ │
│ │  App Tier       │ │          │ │  App Tier       │ │
│ │  - ECS/EKS      │ │          │ │  - ECS/EKS      │ │
│ └─────────────────┘ │          │ └─────────────────┘ │
│ ┌─────────────────┐ │◄────────►│ ┌─────────────────┐ │
│ │  Data Tier      │ │          │ │  Data Tier      │ │
│ │  - RDS Primary  │ │          │ │  - RDS Read Rep │ │
│ │  - DynamoDB     │ │          │ │  - DynamoDB     │ │
│ └─────────────────┘ │          │ └─────────────────┘ │
└─────────────────────┘          └─────────────────────┘
         │                                 │
         └─────── VPC Peering ────────────┘
              or TGW Peering
```

#### Disaster Recovery Multi-Region Design
```
Primary Region (us-west-2)        DR Region (us-east-1)
┌─────────────────────┐          ┌─────────────────────┐
│   Production VPC    │          │    DR VPC (Cold)    │
│                     │          │                     │
│ ┌─────────────────┐ │          │ ┌─────────────────┐ │
│ │  Active Workld  │ │          │ │  Standby Workld │ │
│ │  - Full Traffic │ │          │ │  - Minimal      │ │
│ │  - All Services │ │          │ │  - Essential    │ │
│ └─────────────────┘ │          │ └─────────────────┘ │
│ ┌─────────────────┐ │          │ ┌─────────────────┐ │
│ │  Database       │ │ Async    │ │  Database       │ │
│ │  - RDS Primary  │ │─ Repl ──►│ │  - RDS Standby  │ │
│ │  - Backup to S3 │ │          │ │  - S3 Replica   │ │
│ └─────────────────┘ │          │ └─────────────────┘ │
└─────────────────────┘          └─────────────────────┘
```

### Pattern 4: Multi-Account VPC Architecture

#### Organizational Unit Structure
```
Master Account (Org Root)
├── Security OU
│   ├── Audit Account
│   │   └── Audit VPC (10.0.0.0/16)
│   └── Log Archive Account
│       └── Logging VPC (10.1.0.0/16)
├── Production OU
│   ├── Prod Account 1
│   │   └── Prod VPC 1 (10.10.0.0/16)
│   └── Prod Account 2
│       └── Prod VPC 2 (10.11.0.0/16)
├── Non-Production OU
│   ├── Development Account
│   │   └── Dev VPC (10.20.0.0/16)
│   └── Staging Account
│       └── Staging VPC (10.21.0.0/16)
└── Shared Services OU
    ├── Network Account
    │   ├── Transit Gateway
    │   ├── DNS VPC (10.30.0.0/16)
    │   └── Inspection VPC (10.31.0.0/16)
    └── DevOps Account
        └── CI/CD VPC (10.32.0.0/16)
```

#### Cross-Account Resource Sharing
```
Network Account (Hub)
┌─────────────────────────────────┐
│ Transit Gateway                 │
│ ├── Route Tables               │
│ ├── VPC Attachments            │
│ └── Cross-Account Sharing       │
│                                 │
│ Route 53 Resolver               │
│ ├── Inbound Endpoints          │
│ ├── Outbound Endpoints         │
│ └── Resolver Rules              │
│                                 │
│ VPC Endpoints (PrivateLink)     │
│ ├── S3 Gateway Endpoint        │
│ ├── DynamoDB Gateway Endpoint  │
│ └── Interface Endpoints        │
└─────────────────────────────────┘
           │ (Resource Sharing)
           ▼
┌─────────────────────────────────┐
│ Application Accounts (Spokes)   │
│                                 │
│ ┌─────────────────────────────┐ │
│ │ Application VPC 1           │ │
│ │ - Uses shared TGW          │ │
│ │ - Uses shared DNS          │ │
│ │ │ - Uses shared Endpoints   │ │
│ └─────────────────────────────┘ │
│                                 │
│ ┌─────────────────────────────┐ │
│ │ Application VPC 2           │ │
│ │ - Uses shared TGW          │ │
│ │ - Uses shared DNS          │ │
│ │ - Uses shared Endpoints     │ │
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
```

### Pattern 5: Segmented Network Architecture

#### Security Zone Design
```
DMZ Zone (Public Subnets)
┌─────────────────────────────────┐
│ - Web Application Firewalls    │
│ - Load Balancers               │
│ - Bastion Hosts                │
│ - NAT Gateways                 │
└─────────────┬───────────────────┘
              │ Restricted Access
┌─────────────▼───────────────────┐
│ Application Zone (Private)      │
│ - Application Servers          │
│ - Microservices                │
│ - Container Orchestration      │
│ - API Gateways                 │
└─────────────┬───────────────────┘
              │ Database Access Only
┌─────────────▼───────────────────┐
│ Data Zone (Isolated)           │
│ - Database Servers             │
│ - Data Warehouses              │
│ - Backup Systems               │
│ - Compliance Data              │
└─────────────┬───────────────────┘
              │ Management Access
┌─────────────▼───────────────────┐
│ Management Zone (Isolated)      │
│ - Domain Controllers           │
│ - Configuration Management     │
│ - Monitoring Systems           │
│ - Security Tools               │
└─────────────────────────────────┘
```

#### Microsegmentation with Security Groups
```
Application-Level Segmentation:
┌─────────────────┬─────────────────┬─────────────────┐
│ Frontend SG     │ Backend SG      │ Database SG     │
├─────────────────┼─────────────────┼─────────────────┤
│ Inbound:        │ Inbound:        │ Inbound:        │
│ - 80/443 from   │ - 8080 from     │ - 3306 from     │
│   ALB SG        │   Frontend SG   │   Backend SG    │
│ - 22 from       │ - 22 from       │ - 22 from       │
│   Bastion SG    │   Bastion SG    │   Bastion SG    │
│                 │                 │                 │
│ Outbound:       │ Outbound:       │ Outbound:       │
│ - 8080 to       │ - 3306 to       │ - All traffic   │
│   Backend SG    │   Database SG   │   denied except │
│ - 443 to        │ - 443 to        │   response      │
│   Internet      │   Internet      │   traffic       │
└─────────────────┴─────────────────┴─────────────────┘
```

### Pattern 6: High-Performance Computing (HPC) Architecture

#### HPC Cluster Design
```
HPC VPC Architecture:
┌─────────────────────────────────────────────────────┐
│                Management Subnet                    │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ Head Node   │ │ Login Node  │ │ Bastion     │   │
│ │ - Scheduler │ │ - User      │ │ - SSH       │   │
│ │ - NFS       │ │   Access    │ │   Gateway   │   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────┬───────────────────────────────────┘
                  │ 100G Network
┌─────────────────▼───────────────────────────────────┐
│                Compute Subnet                       │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ Compute     │ │ Compute     │ │ Compute     │   │
│ │ Node 1      │ │ Node 2      │ │ Node N      │   │
│ │ - GPU/CPU   │ │ - GPU/CPU   │ │ - GPU/CPU   │   │
│ │ - Cluster   │ │ - Cluster   │ │ - Cluster   │   │
│ │   Placement │ │   Placement │ │   Placement │   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────┬───────────────────────────────────┘
                  │ High-Speed Storage Access
┌─────────────────▼───────────────────────────────────┐
│                Storage Subnet                       │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ FSx Lustre  │ │ EFS         │ │ S3 Gateway  │   │
│ │ - Scratch   │ │ - Home      │ │ - Archive   │   │
│ │   Storage   │ │   Dirs      │ │   Storage   │   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Pattern 7: Container Orchestration Architecture

#### EKS Multi-Cluster Design
```
                    Route 53 (DNS)
                          │
                    Application LB
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼────────┐ ┌──────▼──────┐ ┌───────▼────────┐
│ Production     │ │ Staging     │ │ Development    │
│ EKS Cluster    │ │ EKS Cluster │ │ EKS Cluster    │
│                │ │             │ │                │
│ ┌────────────┐ │ │ ┌─────────┐ │ │ ┌────────────┐ │
│ │ Namespace  │ │ │ │ Namepac │ │ │ │ Namespace  │ │
│ │ Frontend   │ │ │ │ Test    │ │ │ │ Dev        │ │
│ └────────────┘ │ │ └─────────┘ │ │ └────────────┘ │
│ ┌────────────┐ │ │ ┌─────────┐ │ │ ┌────────────┐ │
│ │ Namespace  │ │ │ │ Namepac │ │ │ │ Namespace  │ │
│ │ Backend    │ │ │ │ UAT     │ │ │ │ Exp        │ │
│ └────────────┘ │ │ └─────────┘ │ │ └────────────┘ │
│ ┌────────────┐ │ │             │ │                │
│ │ Namespace  │ │ │             │ │                │
│ │ Data       │ │ │             │ │                │
│ └────────────┘ │ │             │ │                │
└────────────────┘ └─────────────┘ └────────────────┘
```

#### Service Mesh Integration
```
VPC with Service Mesh (Istio):
┌─────────────────────────────────────────────────────┐
│                 Control Plane                       │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ Pilot       │ │ Citadel     │ │ Galley      │   │
│ │ - Config    │ │ - Security  │ │ - Config    │   │
│ │ - Discovery │ │ - Certs     │ │ - Validation│   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────┬───────────────────────────────────┘
                  │ Control Traffic
┌─────────────────▼───────────────────────────────────┐
│                 Data Plane                          │
│                                                     │
│ ┌─────────────────────────────────────────────────┐ │
│ │            Application Pods                     │ │
│ │  ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐        │ │
│ │  │App  │   │App  │   │App  │   │App  │        │ │
│ │  │ +   │   │ +   │   │ +   │   │ +   │        │ │
│ │  │Proxy│   │Proxy│   │Proxy│   │Proxy│        │ │
│ │  └─────┘   └─────┘   └─────┘   └─────┘        │ │
│ └─────────────────────────────────────────────────┘ │
│                                                     │
│ Features:                                           │
│ - Traffic Management                                │
│ - Security Policies                                 │
│ - Observability                                     │
│ - Load Balancing                                    │
└─────────────────────────────────────────────────────┘
```

## Lab Exercise: Advanced VPC Architecture Implementation

### Lab Duration: 360 minutes

### Step 1: Multi-Tier VPC with Advanced Patterns
**Objective**: Implement enterprise multi-tier VPC with modern architectural patterns

**Deploy Multi-Tier Architecture with Terraform**:
```hcl
# multi-tier-vpc/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}

# Local values for subnet calculation
locals {
  azs_count = 3
  azs       = slice(data.aws_availability_zones.available.names, 0, local.azs_count)
  
  # CIDR calculations for different tiers
  public_subnet_cidrs    = [for i in range(local.azs_count) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs   = [for i in range(local.azs_count) : cidrsubnet(var.vpc_cidr, 8, i + 10)]
  database_subnet_cidrs  = [for i in range(local.azs_count) : cidrsubnet(var.vpc_cidr, 8, i + 20)]
  management_subnet_cidrs = [for i in range(local.azs_count) : cidrsubnet(var.vpc_cidr, 8, i + 30)]
  
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    Architecture = "MultiTier"
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${var.environment}-multitier-vpc"
    Type = "VPC"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-igw"
    Type = "InternetGateway"
  })
}

# Public Subnets (Web Tier)
resource "aws_subnet" "public" {
  count = local.azs_count

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${var.environment}-public-subnet-${count.index + 1}"
    Type = "Public"
    Tier = "Web"
    AZ   = local.azs[count.index]
  })
}

# Private Subnets (Application Tier)
resource "aws_subnet" "private" {
  count = local.azs_count

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-private-subnet-${count.index + 1}"
    Type = "Private"
    Tier = "Application"
    AZ   = local.azs[count.index]
  })
}

# Database Subnets (Data Tier)
resource "aws_subnet" "database" {
  count = local.azs_count

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.database_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-database-subnet-${count.index + 1}"
    Type = "Database"
    Tier = "Data"
    AZ   = local.azs[count.index]
  })
}

# Management Subnets (Management Tier)
resource "aws_subnet" "management" {
  count = local.azs_count

  vpc_id            = aws_vpc.main.id
  cidr_block        = local.management_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-management-subnet-${count.index + 1}"
    Type = "Management"
    Tier = "Management"
    AZ   = local.azs[count.index]
  })
}

# NAT Gateways
resource "aws_eip" "nat" {
  count = local.azs_count

  domain = "vpc"
  depends_on = [aws_internet_gateway.main]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-nat-eip-${count.index + 1}"
    Type = "NATGatewayEIP"
  })
}

resource "aws_nat_gateway" "main" {
  count = local.azs_count

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  depends_on    = [aws_internet_gateway.main]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-nat-gateway-${count.index + 1}"
    Type = "NATGateway"
    AZ   = local.azs[count.index]
  })
}

# Route Tables - Public
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-public-rt"
    Type = "PublicRouteTable"
    Tier = "Web"
  })
}

# Route Tables - Private (per AZ)
resource "aws_route_table" "private" {
  count = local.azs_count

  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-private-rt-${count.index + 1}"
    Type = "PrivateRouteTable"
    Tier = "Application"
    AZ   = local.azs[count.index]
  })
}

# Route Tables - Database
resource "aws_route_table" "database" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-database-rt"
    Type = "DatabaseRouteTable"
    Tier = "Data"
  })
}

# Route Tables - Management
resource "aws_route_table" "management" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[0].id
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-management-rt"
    Type = "ManagementRouteTable"
    Tier = "Management"
  })
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = local.azs_count

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = local.azs_count

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

resource "aws_route_table_association" "database" {
  count = local.azs_count

  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.database.id
}

resource "aws_route_table_association" "management" {
  count = local.azs_count

  subnet_id      = aws_subnet.management[count.index].id
  route_table_id = aws_route_table.management.id
}

# Security Groups - Web Tier
resource "aws_security_group" "web_tier" {
  name        = "${var.environment}-web-tier-sg"
  description = "Security group for web tier (ALB, CDN, WAF)"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from Internet"
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from Internet"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-web-tier-sg"
    Type = "SecurityGroup"
    Tier = "Web"
  })
}

# Security Groups - Application Tier
resource "aws_security_group" "app_tier" {
  name        = "${var.environment}-app-tier-sg"
  description = "Security group for application tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web_tier.id]
    description     = "Application port from web tier"
  }

  ingress {
    from_port       = 8443
    to_port         = 8443
    protocol        = "tcp"
    security_groups = [aws_security_group.web_tier.id]
    description     = "Secure application port from web tier"
  }

  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.management.id]
    description     = "SSH from management tier"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-app-tier-sg"
    Type = "SecurityGroup"
    Tier = "Application"
  })
}

# Security Groups - Database Tier
resource "aws_security_group" "database_tier" {
  name        = "${var.environment}-database-tier-sg"
  description = "Security group for database tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app_tier.id]
    description     = "MySQL from application tier"
  }

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app_tier.id]
    description     = "PostgreSQL from application tier"
  }

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.app_tier.id]
    description     = "Redis from application tier"
  }

  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.management.id]
    description     = "SSH from management tier"
  }

  # No outbound rules - database tier should be isolated

  tags = merge(local.common_tags, {
    Name = "${var.environment}-database-tier-sg"
    Type = "SecurityGroup"
    Tier = "Data"
  })
}

# Security Groups - Management Tier
resource "aws_security_group" "management" {
  name        = "${var.environment}-management-tier-sg"
  description = "Security group for management tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
    description = "SSH from admin network"
  }

  ingress {
    from_port   = 3389
    to_port     = 3389
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
    description = "RDP from admin network"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-management-tier-sg"
    Type = "SecurityGroup"
    Tier = "Management"
  })
}

# VPC Endpoints for AWS Services
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${data.aws_region.current.name}.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = concat(
    [aws_route_table.public.id],
    aws_route_table.private[*].id,
    [aws_route_table.database.id],
    [aws_route_table.management.id]
  )

  tags = merge(local.common_tags, {
    Name = "${var.environment}-s3-endpoint"
    Type = "VPCEndpoint"
    Service = "S3"
  })
}

resource "aws_vpc_endpoint" "ec2" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${data.aws_region.current.name}.ec2"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = merge(local.common_tags, {
    Name = "${var.environment}-ec2-endpoint"
    Type = "VPCEndpoint"
    Service = "EC2"
  })
}

resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${data.aws_region.current.name}.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.management[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true

  tags = merge(local.common_tags, {
    Name = "${var.environment}-ssm-endpoint"
    Type = "VPCEndpoint"
    Service = "SSM"
  })
}

# Security Group for VPC Endpoints
resource "aws_security_group" "vpc_endpoints" {
  name        = "${var.environment}-vpc-endpoints-sg"
  description = "Security group for VPC endpoints"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
    description = "HTTPS from VPC"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpc-endpoints-sg"
    Type = "SecurityGroup"
    Purpose = "VPCEndpoints"
  })
}

# VPC Flow Logs
resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/aws/vpc/flowlogs/${var.environment}"
  retention_in_days = 30

  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpc-flow-logs"
    Type = "CloudWatchLogGroup"
  })
}

resource "aws_iam_role" "vpc_flow_logs" {
  name = "${var.environment}-vpc-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "vpc_flow_logs" {
  name = "${var.environment}-vpc-flow-logs-policy"
  role = aws_iam_role.vpc_flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Effect = "Allow"
        Resource = "arn:aws:logs:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:*"
      }
    ]
  })
}

resource "aws_flow_log" "vpc" {
  iam_role_arn    = aws_iam_role.vpc_flow_logs.arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_logs.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpc-flow-log"
    Type = "VPCFlowLog"
  })
}

# Data source for current region
data "aws_region" "current" {}
```

**Create Variables File**:
```hcl
# multi-tier-vpc/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "prod"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "MultiTierVPC"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "admin_cidr" {
  description = "CIDR block for admin access"
  type        = string
  default     = "10.0.0.0/8"
}
```

**Deploy Multi-Tier Architecture**:
```bash
# Initialize and deploy
cd multi-tier-vpc
terraform init
terraform plan
terraform apply

# Verify deployment
aws ec2 describe-vpcs --filters "Name=tag:Environment,Values=prod"
aws ec2 describe-subnets --filters "Name=tag:Environment,Values=prod"
aws ec2 describe-security-groups --filters "Name=tag:Environment,Values=prod"
```

### Step 2: Hub-and-Spoke Architecture with Transit Gateway
**Objective**: Implement centralized network architecture using Transit Gateway

**Create Hub-and-Spoke Terraform Configuration**:
```hcl
# hub-spoke-architecture/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

# Hub VPC (Shared Services)
resource "aws_vpc" "hub" {
  cidr_block           = var.hub_vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "hub-vpc"
    Environment = "shared"
    Type        = "Hub"
    ManagedBy   = "Terraform"
  }
}

# Spoke VPCs
resource "aws_vpc" "spoke" {
  count = length(var.spoke_vpc_cidrs)

  cidr_block           = var.spoke_vpc_cidrs[count.index]
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "spoke-vpc-${count.index + 1}"
    Environment = var.spoke_environments[count.index]
    Type        = "Spoke"
    ManagedBy   = "Terraform"
  }
}

# Transit Gateway
resource "aws_ec2_transit_gateway" "main" {
  description                     = "Main Transit Gateway for Hub-and-Spoke"
  default_route_table_association = "enable"
  default_route_table_propagation = "enable"
  
  tags = {
    Name        = "main-tgw"
    Environment = "shared"
    Type        = "TransitGateway"
    ManagedBy   = "Terraform"
  }
}

# Internet Gateway for Hub VPC
resource "aws_internet_gateway" "hub" {
  vpc_id = aws_vpc.hub.id

  tags = {
    Name = "hub-igw"
    Type = "InternetGateway"
  }
}

# Hub VPC Subnets
resource "aws_subnet" "hub_public" {
  count = 2

  vpc_id                  = aws_vpc.hub.id
  cidr_block              = cidrsubnet(var.hub_vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "hub-public-subnet-${count.index + 1}"
    Type = "Public"
    Tier = "Shared"
  }
}

resource "aws_subnet" "hub_private" {
  count = 2

  vpc_id            = aws_vpc.hub.id
  cidr_block        = cidrsubnet(var.hub_vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "hub-private-subnet-${count.index + 1}"
    Type = "Private"
    Tier = "Shared"
  }
}

# Spoke VPC Subnets
resource "aws_subnet" "spoke_private" {
  count = length(var.spoke_vpc_cidrs) * 2

  vpc_id            = aws_vpc.spoke[floor(count.index / 2)].id
  cidr_block        = cidrsubnet(var.spoke_vpc_cidrs[floor(count.index / 2)], 8, (count.index % 2) + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index % 2]

  tags = {
    Name = "spoke-${floor(count.index / 2) + 1}-private-subnet-${(count.index % 2) + 1}"
    Type = "Private"
    Tier = "Application"
  }
}

# NAT Gateways for Hub VPC
resource "aws_eip" "hub_nat" {
  count = 2

  domain = "vpc"
  depends_on = [aws_internet_gateway.hub]

  tags = {
    Name = "hub-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "hub" {
  count = 2

  allocation_id = aws_eip.hub_nat[count.index].id
  subnet_id     = aws_subnet.hub_public[count.index].id
  depends_on    = [aws_internet_gateway.hub]

  tags = {
    Name = "hub-nat-gateway-${count.index + 1}"
  }
}

# Transit Gateway Attachments
resource "aws_ec2_transit_gateway_vpc_attachment" "hub" {
  subnet_ids         = aws_subnet.hub_private[*].id
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = aws_vpc.hub.id

  tags = {
    Name = "hub-tgw-attachment"
    Type = "HubAttachment"
  }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "spoke" {
  count = length(var.spoke_vpc_cidrs)

  subnet_ids         = [
    aws_subnet.spoke_private[count.index * 2].id,
    aws_subnet.spoke_private[count.index * 2 + 1].id
  ]
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = aws_vpc.spoke[count.index].id

  tags = {
    Name = "spoke-${count.index + 1}-tgw-attachment"
    Type = "SpokeAttachment"
  }
}

# Route Tables for Hub VPC
resource "aws_route_table" "hub_public" {
  vpc_id = aws_vpc.hub.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.hub.id
  }

  tags = {
    Name = "hub-public-rt"
    Type = "PublicRouteTable"
  }
}

resource "aws_route_table" "hub_private" {
  count = 2

  vpc_id = aws_vpc.hub.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.hub[count.index].id
  }

  # Routes to spoke VPCs via TGW
  dynamic "route" {
    for_each = var.spoke_vpc_cidrs
    content {
      cidr_block         = route.value
      transit_gateway_id = aws_ec2_transit_gateway.main.id
    }
  }

  depends_on = [aws_ec2_transit_gateway_vpc_attachment.hub]

  tags = {
    Name = "hub-private-rt-${count.index + 1}"
    Type = "PrivateRouteTable"
  }
}

# Route Tables for Spoke VPCs
resource "aws_route_table" "spoke_private" {
  count = length(var.spoke_vpc_cidrs)

  vpc_id = aws_vpc.spoke[count.index].id

  # Default route to TGW (for internet access via hub)
  route {
    cidr_block         = "0.0.0.0/0"
    transit_gateway_id = aws_ec2_transit_gateway.main.id
  }

  # Route to hub VPC
  route {
    cidr_block         = var.hub_vpc_cidr
    transit_gateway_id = aws_ec2_transit_gateway.main.id
  }

  depends_on = [aws_ec2_transit_gateway_vpc_attachment.spoke]

  tags = {
    Name = "spoke-${count.index + 1}-private-rt"
    Type = "SpokeRouteTable"
  }
}

# Route Table Associations
resource "aws_route_table_association" "hub_public" {
  count = 2

  subnet_id      = aws_subnet.hub_public[count.index].id
  route_table_id = aws_route_table.hub_public.id
}

resource "aws_route_table_association" "hub_private" {
  count = 2

  subnet_id      = aws_subnet.hub_private[count.index].id
  route_table_id = aws_route_table.hub_private[count.index].id
}

resource "aws_route_table_association" "spoke_private" {
  count = length(var.spoke_vpc_cidrs) * 2

  subnet_id      = aws_subnet.spoke_private[count.index].id
  route_table_id = aws_route_table.spoke_private[floor(count.index / 2)].id
}

# Transit Gateway Route Tables for advanced routing
resource "aws_ec2_transit_gateway_route_table" "hub" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id

  tags = {
    Name = "hub-tgw-route-table"
    Type = "HubRouteTable"
  }
}

resource "aws_ec2_transit_gateway_route_table" "production" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id

  tags = {
    Name = "production-tgw-route-table"
    Type = "ProductionRouteTable"
  }
}

resource "aws_ec2_transit_gateway_route_table" "nonproduction" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id

  tags = {
    Name = "nonproduction-tgw-route-table"
    Type = "NonProductionRouteTable"
  }
}

# Route Table Associations
resource "aws_ec2_transit_gateway_route_table_association" "hub" {
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.hub.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.hub.id
}

resource "aws_ec2_transit_gateway_route_table_association" "spoke" {
  count = length(var.spoke_vpc_cidrs)

  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.spoke[count.index].id
  transit_gateway_route_table_id = var.spoke_environments[count.index] == "production" ? aws_ec2_transit_gateway_route_table.production.id : aws_ec2_transit_gateway_route_table.nonproduction.id
}

# Route Propagations - Hub can reach all spokes
resource "aws_ec2_transit_gateway_route_table_propagation" "hub_to_spokes" {
  count = length(var.spoke_vpc_cidrs)

  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.spoke[count.index].id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.hub.id
}

# Route Propagations - Spokes can reach hub
resource "aws_ec2_transit_gateway_route_table_propagation" "spokes_to_hub" {
  count = length(var.spoke_vpc_cidrs)

  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.hub.id
  transit_gateway_route_table_id = var.spoke_environments[count.index] == "production" ? aws_ec2_transit_gateway_route_table.production.id : aws_ec2_transit_gateway_route_table.nonproduction.id
}

# Security Groups
resource "aws_security_group" "hub_services" {
  name        = "hub-services-sg"
  description = "Security group for hub shared services"
  vpc_id      = aws_vpc.hub.id

  ingress {
    from_port   = 53
    to_port     = 53
    protocol    = "udp"
    cidr_blocks = var.spoke_vpc_cidrs
    description = "DNS from spokes"
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.spoke_vpc_cidrs
    description = "HTTP from spokes"
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = var.spoke_vpc_cidrs
    description = "HTTPS from spokes"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "hub-services-sg"
    Type = "SecurityGroup"
  }
}

resource "aws_security_group" "spoke_apps" {
  count = length(var.spoke_vpc_cidrs)

  name        = "spoke-${count.index + 1}-apps-sg"
  description = "Security group for spoke applications"
  vpc_id      = aws_vpc.spoke[count.index].id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [var.hub_vpc_cidr]
    description = "HTTP from hub"
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.hub_vpc_cidr]
    description = "HTTPS from hub"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "spoke-${count.index + 1}-apps-sg"
    Type = "SecurityGroup"
  }
}
```

**Create Variables for Hub-and-Spoke**:
```hcl
# hub-spoke-architecture/variables.tf
variable "hub_vpc_cidr" {
  description = "CIDR block for hub VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "spoke_vpc_cidrs" {
  description = "CIDR blocks for spoke VPCs"
  type        = list(string)
  default     = [
    "10.1.0.0/16",  # Production
    "10.2.0.0/16",  # Staging
    "10.3.0.0/16"   # Development
  ]
}

variable "spoke_environments" {
  description = "Environment names for spoke VPCs"
  type        = list(string)
  default     = [
    "production",
    "staging",
    "development"
  ]
}
```

**Deploy Hub-and-Spoke Architecture**:
```bash
# Deploy hub-and-spoke
cd hub-spoke-architecture
terraform init
terraform plan
terraform apply

# Verify Transit Gateway
aws ec2 describe-transit-gateways
aws ec2 describe-transit-gateway-vpc-attachments
aws ec2 describe-transit-gateway-route-tables
```

### Step 3: Multi-Region VPC Architecture
**Objective**: Implement multi-region VPC architecture with inter-region connectivity

**Create Multi-Region Terraform Configuration**:
```hcl
# multi-region-vpc/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure AWS providers for multiple regions
provider "aws" {
  alias  = "primary"
  region = var.primary_region
}

provider "aws" {
  alias  = "secondary"
  region = var.secondary_region
}

# Data sources
data "aws_availability_zones" "primary" {
  provider = aws.primary
  state    = "available"
}

data "aws_availability_zones" "secondary" {
  provider = aws.secondary
  state    = "available"
}

# Primary Region VPC
resource "aws_vpc" "primary" {
  provider = aws.primary

  cidr_block           = var.primary_vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "primary-region-vpc"
    Region      = var.primary_region
    Type        = "Primary"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Secondary Region VPC
resource "aws_vpc" "secondary" {
  provider = aws.secondary

  cidr_block           = var.secondary_vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "secondary-region-vpc"
    Region      = var.secondary_region
    Type        = "Secondary"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Primary Region Subnets
resource "aws_subnet" "primary_public" {
  provider = aws.primary
  count    = 2

  vpc_id                  = aws_vpc.primary.id
  cidr_block              = cidrsubnet(var.primary_vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.primary.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name   = "primary-public-subnet-${count.index + 1}"
    Type   = "Public"
    Region = var.primary_region
  }
}

resource "aws_subnet" "primary_private" {
  provider = aws.primary
  count    = 2

  vpc_id            = aws_vpc.primary.id
  cidr_block        = cidrsubnet(var.primary_vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.primary.names[count.index]

  tags = {
    Name   = "primary-private-subnet-${count.index + 1}"
    Type   = "Private"
    Region = var.primary_region
  }
}

# Secondary Region Subnets
resource "aws_subnet" "secondary_public" {
  provider = aws.secondary
  count    = 2

  vpc_id                  = aws_vpc.secondary.id
  cidr_block              = cidrsubnet(var.secondary_vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.secondary.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name   = "secondary-public-subnet-${count.index + 1}"
    Type   = "Public"
    Region = var.secondary_region
  }
}

resource "aws_subnet" "secondary_private" {
  provider = aws.secondary
  count    = 2

  vpc_id            = aws_vpc.secondary.id
  cidr_block        = cidrsubnet(var.secondary_vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.secondary.names[count.index]

  tags = {
    Name   = "secondary-private-subnet-${count.index + 1}"
    Type   = "Private"
    Region = var.secondary_region
  }
}

# Internet Gateways
resource "aws_internet_gateway" "primary" {
  provider = aws.primary

  vpc_id = aws_vpc.primary.id

  tags = {
    Name   = "primary-igw"
    Region = var.primary_region
  }
}

resource "aws_internet_gateway" "secondary" {
  provider = aws.secondary

  vpc_id = aws_vpc.secondary.id

  tags = {
    Name   = "secondary-igw"
    Region = var.secondary_region
  }
}

# NAT Gateways for Primary Region
resource "aws_eip" "primary_nat" {
  provider = aws.primary
  count    = 2

  domain = "vpc"
  depends_on = [aws_internet_gateway.primary]

  tags = {
    Name   = "primary-nat-eip-${count.index + 1}"
    Region = var.primary_region
  }
}

resource "aws_nat_gateway" "primary" {
  provider = aws.primary
  count    = 2

  allocation_id = aws_eip.primary_nat[count.index].id
  subnet_id     = aws_subnet.primary_public[count.index].id
  depends_on    = [aws_internet_gateway.primary]

  tags = {
    Name   = "primary-nat-gateway-${count.index + 1}"
    Region = var.primary_region
  }
}

# NAT Gateways for Secondary Region
resource "aws_eip" "secondary_nat" {
  provider = aws.secondary
  count    = 2

  domain = "vpc"
  depends_on = [aws_internet_gateway.secondary]

  tags = {
    Name   = "secondary-nat-eip-${count.index + 1}"
    Region = var.secondary_region
  }
}

resource "aws_nat_gateway" "secondary" {
  provider = aws.secondary
  count    = 2

  allocation_id = aws_eip.secondary_nat[count.index].id
  subnet_id     = aws_subnet.secondary_public[count.index].id
  depends_on    = [aws_internet_gateway.secondary]

  tags = {
    Name   = "secondary-nat-gateway-${count.index + 1}"
    Region = var.secondary_region
  }
}

# VPC Peering Connection
resource "aws_vpc_peering_connection" "cross_region" {
  provider = aws.primary

  vpc_id        = aws_vpc.primary.id
  peer_vpc_id   = aws_vpc.secondary.id
  peer_region   = var.secondary_region
  auto_accept   = false

  tags = {
    Name = "primary-to-secondary-peering"
    Type = "CrossRegionPeering"
  }
}

# Accept VPC Peering Connection in Secondary Region
resource "aws_vpc_peering_connection_accepter" "cross_region" {
  provider = aws.secondary

  vpc_peering_connection_id = aws_vpc_peering_connection.cross_region.id
  auto_accept               = true

  tags = {
    Name = "secondary-accept-peering"
    Type = "CrossRegionPeeringAccepter"
  }
}

# Route Tables - Primary Region
resource "aws_route_table" "primary_public" {
  provider = aws.primary

  vpc_id = aws_vpc.primary.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.primary.id
  }

  route {
    cidr_block                = var.secondary_vpc_cidr
    vpc_peering_connection_id = aws_vpc_peering_connection.cross_region.id
  }

  tags = {
    Name   = "primary-public-rt"
    Region = var.primary_region
  }
}

resource "aws_route_table" "primary_private" {
  provider = aws.primary
  count    = 2

  vpc_id = aws_vpc.primary.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.primary[count.index].id
  }

  route {
    cidr_block                = var.secondary_vpc_cidr
    vpc_peering_connection_id = aws_vpc_peering_connection.cross_region.id
  }

  tags = {
    Name   = "primary-private-rt-${count.index + 1}"
    Region = var.primary_region
  }
}

# Route Tables - Secondary Region
resource "aws_route_table" "secondary_public" {
  provider = aws.secondary

  vpc_id = aws_vpc.secondary.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.secondary.id
  }

  route {
    cidr_block                = var.primary_vpc_cidr
    vpc_peering_connection_id = aws_vpc_peering_connection.cross_region.id
  }

  tags = {
    Name   = "secondary-public-rt"
    Region = var.secondary_region
  }
}

resource "aws_route_table" "secondary_private" {
  provider = aws.secondary
  count    = 2

  vpc_id = aws_vpc.secondary.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.secondary[count.index].id
  }

  route {
    cidr_block                = var.primary_vpc_cidr
    vpc_peering_connection_id = aws_vpc_peering_connection.cross_region.id
  }

  tags = {
    Name   = "secondary-private-rt-${count.index + 1}"
    Region = var.secondary_region
  }
}

# Route Table Associations - Primary Region
resource "aws_route_table_association" "primary_public" {
  provider = aws.primary
  count    = 2

  subnet_id      = aws_subnet.primary_public[count.index].id
  route_table_id = aws_route_table.primary_public.id
}

resource "aws_route_table_association" "primary_private" {
  provider = aws.primary
  count    = 2

  subnet_id      = aws_subnet.primary_private[count.index].id
  route_table_id = aws_route_table.primary_private[count.index].id
}

# Route Table Associations - Secondary Region
resource "aws_route_table_association" "secondary_public" {
  provider = aws.secondary
  count    = 2

  subnet_id      = aws_subnet.secondary_public[count.index].id
  route_table_id = aws_route_table.secondary_public.id
}

resource "aws_route_table_association" "secondary_private" {
  provider = aws.secondary
  count    = 2

  subnet_id      = aws_subnet.secondary_private[count.index].id
  route_table_id = aws_route_table.secondary_private[count.index].id
}

# Security Groups for Cross-Region Communication
resource "aws_security_group" "primary_cross_region" {
  provider = aws.primary

  name        = "primary-cross-region-sg"
  description = "Security group for cross-region communication"
  vpc_id      = aws_vpc.primary.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.secondary_vpc_cidr]
    description = "HTTPS from secondary region"
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [var.secondary_vpc_cidr]
    description = "HTTP from secondary region"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name   = "primary-cross-region-sg"
    Region = var.primary_region
  }
}

resource "aws_security_group" "secondary_cross_region" {
  provider = aws.secondary

  name        = "secondary-cross-region-sg"
  description = "Security group for cross-region communication"
  vpc_id      = aws_vpc.secondary.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [var.primary_vpc_cidr]
    description = "HTTPS from primary region"
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = [var.primary_vpc_cidr]
    description = "HTTP from primary region"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name   = "secondary-cross-region-sg"
    Region = var.secondary_region
  }
}
```

**Create Variables for Multi-Region**:
```hcl
# multi-region-vpc/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "prod"
}

variable "primary_region" {
  description = "Primary AWS region"
  type        = string
  default     = "us-west-2"
}

variable "secondary_region" {
  description = "Secondary AWS region"
  type        = string
  default     = "us-east-1"
}

variable "primary_vpc_cidr" {
  description = "CIDR block for primary VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "secondary_vpc_cidr" {
  description = "CIDR block for secondary VPC"
  type        = string
  default     = "10.1.0.0/16"
}
```

**Deploy Multi-Region Architecture**:
```bash
# Deploy multi-region architecture
cd multi-region-vpc
terraform init
terraform plan
terraform apply

# Test cross-region connectivity
aws ec2 describe-vpc-peering-connections --region us-west-2
aws ec2 describe-vpc-peering-connections --region us-east-1
```

### Step 4: Architecture Validation and Testing
**Objective**: Validate and test the implemented VPC architectures

**Create Architecture Validation Script**:
```bash
#!/bin/bash
# scripts/validate-vpc-architectures.sh

set -euo pipefail

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Logging function
log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"
}

warn() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] WARNING: $1${NC}"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERROR: $1${NC}"
}

# Function to validate multi-tier architecture
validate_multi_tier() {
    log "Validating Multi-Tier VPC Architecture..."
    
    # Check VPC exists
    VPC_ID=$(aws ec2 describe-vpcs \
        --filters "Name=tag:Environment,Values=prod" "Name=tag:Architecture,Values=MultiTier" \
        --query 'Vpcs[0].VpcId' --output text)
    
    if [[ "$VPC_ID" == "None" ]]; then
        error "Multi-tier VPC not found"
        return 1
    fi
    
    log "Found Multi-tier VPC: $VPC_ID"
    
    # Check subnets
    PUBLIC_SUBNETS=$(aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Tier,Values=Web" \
        --query 'Subnets | length(@)')
    
    PRIVATE_SUBNETS=$(aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Tier,Values=Application" \
        --query 'Subnets | length(@)')
    
    DATABASE_SUBNETS=$(aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Tier,Values=Data" \
        --query 'Subnets | length(@)')
    
    MANAGEMENT_SUBNETS=$(aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Tier,Values=Management" \
        --query 'Subnets | length(@)')
    
    log "Subnet counts - Public: $PUBLIC_SUBNETS, Private: $PRIVATE_SUBNETS, Database: $DATABASE_SUBNETS, Management: $MANAGEMENT_SUBNETS"
    
    if [[ $PUBLIC_SUBNETS -lt 3 ]] || [[ $PRIVATE_SUBNETS -lt 3 ]] || [[ $DATABASE_SUBNETS -lt 3 ]] || [[ $MANAGEMENT_SUBNETS -lt 3 ]]; then
        error "Insufficient subnets for multi-tier architecture"
        return 1
    fi
    
    # Check security groups
    SG_COUNT=$(aws ec2 describe-security-groups \
        --filters "Name=vpc-id,Values=$VPC_ID" \
        --query 'SecurityGroups | length(@)')
    
    log "Security Groups count: $SG_COUNT"
    
    if [[ $SG_COUNT -lt 5 ]]; then
        warn "Expected at least 5 security groups for proper tier isolation"
    fi
    
    log "✅ Multi-tier VPC validation completed"
}

# Function to validate hub-and-spoke architecture
validate_hub_spoke() {
    log "Validating Hub-and-Spoke Architecture..."
    
    # Check Transit Gateway
    TGW_ID=$(aws ec2 describe-transit-gateways \
        --filters "Name=tag:Name,Values=main-tgw" \
        --query 'TransitGateways[0].TransitGatewayId' --output text)
    
    if [[ "$TGW_ID" == "None" ]]; then
        error "Transit Gateway not found"
        return 1
    fi
    
    log "Found Transit Gateway: $TGW_ID"
    
    # Check VPC attachments
    ATTACHMENT_COUNT=$(aws ec2 describe-transit-gateway-vpc-attachments \
        --filters "Name=transit-gateway-id,Values=$TGW_ID" "Name=state,Values=available" \
        --query 'TransitGatewayVpcAttachments | length(@)')
    
    log "Transit Gateway VPC Attachments: $ATTACHMENT_COUNT"
    
    if [[ $ATTACHMENT_COUNT -lt 4 ]]; then
        error "Expected at least 4 VPC attachments (1 hub + 3 spokes)"
        return 1
    fi
    
    # Check route tables
    ROUTE_TABLE_COUNT=$(aws ec2 describe-transit-gateway-route-tables \
        --filters "Name=transit-gateway-id,Values=$TGW_ID" \
        --query 'TransitGatewayRouteTables | length(@)')
    
    log "Transit Gateway Route Tables: $ROUTE_TABLE_COUNT"
    
    log "✅ Hub-and-Spoke validation completed"
}

# Function to validate multi-region architecture
validate_multi_region() {
    log "Validating Multi-Region Architecture..."
    
    # Check primary region VPC
    PRIMARY_VPC=$(aws ec2 describe-vpcs \
        --region us-west-2 \
        --filters "Name=tag:Type,Values=Primary" \
        --query 'Vpcs[0].VpcId' --output text)
    
    # Check secondary region VPC
    SECONDARY_VPC=$(aws ec2 describe-vpcs \
        --region us-east-1 \
        --filters "Name=tag:Type,Values=Secondary" \
        --query 'Vpcs[0].VpcId' --output text)
    
    if [[ "$PRIMARY_VPC" == "None" ]] || [[ "$SECONDARY_VPC" == "None" ]]; then
        error "Multi-region VPCs not found"
        return 1
    fi
    
    log "Primary VPC (us-west-2): $PRIMARY_VPC"
    log "Secondary VPC (us-east-1): $SECONDARY_VPC"
    
    # Check VPC peering connection
    PEERING_CONNECTION=$(aws ec2 describe-vpc-peering-connections \
        --region us-west-2 \
        --filters "Name=requester-vpc-info.vpc-id,Values=$PRIMARY_VPC" \
        --query 'VpcPeeringConnections[0].VpcPeeringConnectionId' --output text)
    
    if [[ "$PEERING_CONNECTION" == "None" ]]; then
        error "VPC peering connection not found"
        return 1
    fi
    
    log "VPC Peering Connection: $PEERING_CONNECTION"
    
    # Check peering connection status
    PEERING_STATUS=$(aws ec2 describe-vpc-peering-connections \
        --region us-west-2 \
        --vpc-peering-connection-ids $PEERING_CONNECTION \
        --query 'VpcPeeringConnections[0].Status.Code' --output text)
    
    if [[ "$PEERING_STATUS" != "active" ]]; then
        error "VPC peering connection is not active: $PEERING_STATUS"
        return 1
    fi
    
    log "✅ Multi-region validation completed"
}

# Function to test connectivity
test_connectivity() {
    log "Testing VPC Connectivity..."
    
    # This would typically involve launching test instances
    # For this example, we'll test route table configurations
    
    # Test multi-tier routing
    if validate_multi_tier_routing; then
        log "✅ Multi-tier routing test passed"
    else
        error "Multi-tier routing test failed"
    fi
    
    # Test hub-spoke routing
    if validate_hub_spoke_routing; then
        log "✅ Hub-spoke routing test passed"
    else
        error "Hub-spoke routing test failed"
    fi
}

validate_multi_tier_routing() {
    # Check if private subnets have routes to NAT Gateway
    VPC_ID=$(aws ec2 describe-vpcs \
        --filters "Name=tag:Environment,Values=prod" "Name=tag:Architecture,Values=MultiTier" \
        --query 'Vpcs[0].VpcId' --output text)
    
    PRIVATE_SUBNET_ID=$(aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Tier,Values=Application" \
        --query 'Subnets[0].SubnetId' --output text)
    
    ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
        --filters "Name=association.subnet-id,Values=$PRIVATE_SUBNET_ID" \
        --query 'RouteTables[0].RouteTableId' --output text)
    
    NAT_ROUTE=$(aws ec2 describe-route-tables \
        --route-table-ids $ROUTE_TABLE_ID \
        --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`].NatGatewayId' --output text)
    
    [[ "$NAT_ROUTE" != "None" ]] && [[ -n "$NAT_ROUTE" ]]
}

validate_hub_spoke_routing() {
    # Check Transit Gateway route propagation
    TGW_ID=$(aws ec2 describe-transit-gateways \
        --filters "Name=tag:Name,Values=main-tgw" \
        --query 'TransitGateways[0].TransitGatewayId' --output text)
    
    ROUTE_TABLE_ID=$(aws ec2 describe-transit-gateway-route-tables \
        --filters "Name=transit-gateway-id,Values=$TGW_ID" \
        --query 'TransitGatewayRouteTables[0].TransitGatewayRouteTableId' --output text)
    
    ROUTE_COUNT=$(aws ec2 search-transit-gateway-routes \
        --transit-gateway-route-table-id $ROUTE_TABLE_ID \
        --filters "Name=state,Values=active" \
        --query 'Routes | length(@)')
    
    [[ $ROUTE_COUNT -gt 0 ]]
}

# Function to generate architecture report
generate_report() {
    local REPORT_FILE="vpc-architecture-validation-report.json"
    
    log "Generating architecture validation report..."
    
    cat > $REPORT_FILE << EOF
{
  "validation_timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "architectures": {
    "multi_tier": {
      "status": "$(validate_multi_tier && echo "PASS" || echo "FAIL")",
      "vpc_count": $(aws ec2 describe-vpcs --filters "Name=tag:Architecture,Values=MultiTier" --query 'Vpcs | length(@)'),
      "subnet_count": $(aws ec2 describe-subnets --filters "Name=tag:Environment,Values=prod" --query 'Subnets | length(@)')
    },
    "hub_spoke": {
      "status": "$(validate_hub_spoke && echo "PASS" || echo "FAIL")",
      "transit_gateway_count": $(aws ec2 describe-transit-gateways --query 'TransitGateways | length(@)'),
      "vpc_attachments": $(aws ec2 describe-transit-gateway-vpc-attachments --query 'TransitGatewayVpcAttachments | length(@)')
    },
    "multi_region": {
      "status": "$(validate_multi_region && echo "PASS" || echo "FAIL")",
      "peering_connections": $(aws ec2 describe-vpc-peering-connections --query 'VpcPeeringConnections | length(@)')
    }
  },
  "summary": {
    "total_vpcs": $(aws ec2 describe-vpcs --query 'Vpcs | length(@)'),
    "total_subnets": $(aws ec2 describe-subnets --query 'Subnets | length(@)'),
    "total_security_groups": $(aws ec2 describe-security-groups --query 'SecurityGroups | length(@)')
  }
}
EOF
    
    log "Report generated: $REPORT_FILE"
    cat $REPORT_FILE
}

# Main execution
main() {
    log "Starting VPC Architecture Validation..."
    
    local EXIT_CODE=0
    
    # Run validations
    validate_multi_tier || EXIT_CODE=1
    validate_hub_spoke || EXIT_CODE=1
    validate_multi_region || EXIT_CODE=1
    
    # Test connectivity
    test_connectivity || EXIT_CODE=1
    
    # Generate report
    generate_report
    
    if [[ $EXIT_CODE -eq 0 ]]; then
        log "🎉 All architecture validations passed!"
    else
        error "Some validations failed. Check the logs above."
    fi
    
    exit $EXIT_CODE
}

# Run main function
main "$@"
```

**Make the script executable and run validation**:
```bash
# Make script executable
chmod +x scripts/validate-vpc-architectures.sh

# Run validation
./scripts/validate-vpc-architectures.sh

# Check results
echo "Validation completed. Check the generated report."
```

## Best Practices Summary

### Architecture Design Best Practices
1. **Design for Scale**: Plan for future growth in CIDR allocation and subnet design
2. **Security by Design**: Implement defense-in-depth with multiple security layers
3. **High Availability**: Distribute resources across multiple AZs and regions
4. **Cost Optimization**: Use appropriate instance types and reserved capacity
5. **Operational Excellence**: Design for automation and maintainability

### Network Segmentation Best Practices
1. **Least Privilege**: Implement minimal required network access
2. **Microsegmentation**: Use security groups for application-level isolation
3. **Network Zones**: Separate environments and tiers with different security requirements
4. **Monitoring**: Implement comprehensive network monitoring and logging
5. **Compliance**: Ensure network design meets regulatory requirements

### Multi-Region Best Practices
1. **Data Sovereignty**: Consider data residency requirements
2. **Latency Optimization**: Choose regions based on user proximity
3. **Disaster Recovery**: Implement appropriate DR strategies
4. **Cost Considerations**: Balance availability with cost requirements
5. **Operational Complexity**: Manage complexity of multi-region operations

## Troubleshooting Guide

### Common Architecture Issues
1. **CIDR Overlap**: Ensure non-overlapping CIDR blocks across VPCs
2. **Route Table Conflicts**: Verify route table configurations and priorities
3. **Security Group Rules**: Check security group rules for proper communication
4. **Transit Gateway Limits**: Monitor Transit Gateway attachment and route limits
5. **Cross-Region Latency**: Optimize for network latency in multi-region setups

### Performance Optimization
1. **Instance Placement**: Use placement groups for high-performance computing
2. **Network Optimization**: Enable enhanced networking and SR-IOV
3. **Monitoring**: Use CloudWatch and VPC Flow Logs for performance analysis
4. **Load Balancing**: Implement appropriate load balancing strategies
5. **Caching**: Use caching solutions to reduce network traffic

## Additional Resources
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [VPC Design Patterns](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Scenarios.html)
- [Transit Gateway Best Practices](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html)
- [Multi-Region Architecture Guide](https://aws.amazon.com/blogs/architecture/)

## Exam Tips
- Understand when to use each architectural pattern
- Know the trade-offs between different connectivity options
- Be familiar with Transit Gateway routing and segmentation
- Understand multi-region connectivity patterns
- Know security best practices for each architectural pattern