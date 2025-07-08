# Topic 22: Route 53 Resolver for Hybrid DNS

## Prerequisites
- Completed Topics 11-21 (Hybrid Networking Foundation)
- Understanding of DNS concepts and resolution processes
- Knowledge of Active Directory and enterprise DNS architectures
- Familiarity with VPC networking and hybrid connectivity

## Learning Objectives
By the end of this topic, you will be able to:
- Design hybrid DNS architectures using Route 53 Resolver
- Configure DNS forwarding rules for seamless name resolution
- Implement conditional forwarding for multi-domain environments
- Monitor and troubleshoot hybrid DNS resolution issues

## Theory

### Route 53 Resolver Overview

#### Definition
Route 53 Resolver is a regional DNS service that provides recursive DNS resolution for VPCs and enables hybrid DNS architectures between AWS and on-premises environments.

#### Key Components
- **Resolver Endpoints**: Inbound and outbound endpoints for DNS traffic
- **Forwarding Rules**: Conditional forwarding based on domain names
- **Query Logging**: DNS query logging for monitoring and troubleshooting
- **DNSSEC Validation**: Security validation for DNS responses

#### DNS Resolution Flow
```
On-premises Client → On-premises DNS → Outbound Endpoint → Route 53 Resolver
                                                                   ↓
VPC Client → VPC DNS (169.254.169.253) → Route 53 Resolver → Authoritative DNS
```

### Hybrid DNS Architecture Patterns

#### Pattern 1: Centralized DNS (AWS-centric)
```
On-premises → Outbound Endpoint → Route 53 Resolver → AWS DNS Services
                                        ↓
                                 Route 53 Hosted Zones
                                 Route 53 Private Zones
```

#### Pattern 2: Distributed DNS (Bidirectional)
```
On-premises DNS ←→ Conditional Forwarding ←→ Route 53 Resolver
      ↓                                              ↓
Local Domains                                  AWS Domains
External Domains                               Route 53 Zones
```

#### Pattern 3: DNS Proxy Architecture
```
Clients → DNS Proxy/Forwarder → Route 53 Resolver Endpoints
              ↓                         ↓
      Cache + Filtering           Authoritative Resolution
```

### Route 53 Resolver Components

#### Inbound Endpoints
- **Purpose**: Receive DNS queries from on-premises networks
- **Configuration**: 2-8 IP addresses across multiple AZs
- **Use Case**: On-premises queries for AWS resources
- **Security**: VPC security groups control access

#### Outbound Endpoints
- **Purpose**: Forward DNS queries to on-premises DNS servers
- **Configuration**: 2-8 IP addresses for high availability
- **Use Case**: AWS resources querying on-premises domains
- **Rules**: Forwarding rules define which domains to forward

#### Forwarding Rules
- **Conditional Forwarding**: Forward specific domains to target resolvers
- **System Rules**: AWS-managed rules for AWS services
- **Auto-defined Rules**: Automatically created for associated VPCs
- **Priority**: Rule precedence for overlapping domains

## Lab Exercise: Comprehensive Hybrid DNS Implementation

### Lab Duration: 360 minutes

### Step 1: Plan Hybrid DNS Architecture
**Objective**: Design DNS strategy for hybrid environment

**DNS Architecture Planning**:
```bash
# Create hybrid DNS planning assessment
cat << 'EOF' > hybrid-dns-planning.sh
#!/bin/bash

echo "=== Hybrid DNS Architecture Planning ==="
echo ""

echo "1. DNS REQUIREMENTS ANALYSIS"
echo "   On-premises DNS servers: _______"
echo "   On-premises domain names: _______"
echo "   AWS domain requirements: _______"
echo "   External domain resolution: Internet/On-premises"
echo "   DNS query volume: _______ queries/second"
echo ""

echo "2. NETWORK TOPOLOGY"
echo "   VPC CIDR blocks: _______"
echo "   On-premises networks: _______"
echo "   Connectivity method: Direct Connect/VPN/Both"
echo "   DNS traffic encryption: Yes/No"
echo "   Network segmentation: Yes/No"
echo ""

echo "3. DOMAIN STRUCTURE"
echo "   Internal domain names: _______"
echo "   AWS-specific domains: _______"
echo "   Shared domain names: _______"
echo "   Domain delegation strategy: _______"
echo "   Split-horizon DNS: Required/Optional"
echo ""

echo "4. RESOLUTION STRATEGY"
echo "   Primary DNS resolution: AWS/On-premises/Hybrid"
echo "   Conditional forwarding domains: _______"
echo "   Recursive resolution: AWS/On-premises/Both"
echo "   Caching strategy: Client/Server/Distributed"
echo "   Failover DNS servers: Yes/No"
echo ""

echo "5. SECURITY AND COMPLIANCE"
echo "   DNS query logging: Required/Optional"
echo "   DNSSEC validation: Required/Optional"
echo "   Access control: IP-based/VPC-based/Both"
echo "   Audit requirements: _______"
echo "   Data residency: Specific regions required"
echo ""

echo "6. PERFORMANCE REQUIREMENTS"
echo "   Acceptable query latency: _______ ms"
echo "   Geographic distribution: Single/Multi-region"
echo "   High availability: 99.9%/99.95%/99.99%"
echo "   Disaster recovery: RTO _______ RPO _______"
echo "   Monitoring requirements: Basic/Advanced/Custom"

EOF

chmod +x hybrid-dns-planning.sh
./hybrid-dns-planning.sh
```

### Step 2: Set Up Route 53 Resolver Infrastructure
**Objective**: Create resolver endpoints and basic configuration

**Resolver Infrastructure Setup**:
```bash
# Set up Route 53 Resolver infrastructure
cat << 'EOF' > setup-route53-resolver.sh
#!/bin/bash

# Use existing VPC from previous labs or create new one
if [ -f "nlb-vpc-config.env" ]; then
    source nlb-vpc-config.env
    echo "Using existing VPC infrastructure from previous labs"
else
    echo "Creating new VPC for Route 53 Resolver lab..."
    
    # Create VPC for DNS lab
    VPC_ID=$(aws ec2 create-vpc \
        --cidr-block 10.100.0.0/16 \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=DNS-Resolver-VPC}]" \
        --query 'Vpc.VpcId' \
        --output text)
    
    # Enable DNS support
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
    
    echo "✅ VPC created: $VPC_ID"
    
    # Create subnets for resolver endpoints
    AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[0:2].ZoneName' --output text))
    PRIVATE_SUBNET_IDS=()
    
    for i in "${!AZS[@]}"; do
        SUBNET_ID=$(aws ec2 create-subnet \
            --vpc-id $VPC_ID \
            --cidr-block 10.100.$((i+1)).0/24 \
            --availability-zone ${AZS[$i]} \
            --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=DNS-Resolver-Subnet-${AZS[$i]}}]" \
            --query 'Subnet.SubnetId' \
            --output text)
        
        PRIVATE_SUBNET_IDS+=($SUBNET_ID)
        echo "✅ Subnet created: $SUBNET_ID"
    done
fi

echo "=== Setting Up Route 53 Resolver Infrastructure ==="
echo ""

echo "1. CREATING SECURITY GROUPS FOR RESOLVER ENDPOINTS"

# Security group for inbound resolver endpoint
INBOUND_SG_ID=$(aws ec2 create-security-group \
    --group-name Route53-Resolver-Inbound-SG \
    --description "Security group for Route 53 Resolver inbound endpoint" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Route53-Resolver-Inbound-SG}]" \
    --query 'GroupId' \
    --output text)

# Allow DNS traffic from on-premises networks
ONPREM_CIDR="10.1.0.0/16"  # Replace with actual on-premises CIDR

aws ec2 authorize-security-group-ingress \
    --group-id $INBOUND_SG_ID \
    --protocol tcp \
    --port 53 \
    --cidr $ONPREM_CIDR

aws ec2 authorize-security-group-ingress \
    --group-id $INBOUND_SG_ID \
    --protocol udp \
    --port 53 \
    --cidr $ONPREM_CIDR

echo "✅ Inbound resolver security group created: $INBOUND_SG_ID"

# Security group for outbound resolver endpoint
OUTBOUND_SG_ID=$(aws ec2 create-security-group \
    --group-name Route53-Resolver-Outbound-SG \
    --description "Security group for Route 53 Resolver outbound endpoint" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Route53-Resolver-Outbound-SG}]" \
    --query 'GroupId' \
    --output text)

# Allow DNS traffic to on-premises DNS servers
aws ec2 authorize-security-group-egress \
    --group-id $OUTBOUND_SG_ID \
    --protocol tcp \
    --port 53 \
    --cidr $ONPREM_CIDR

aws ec2 authorize-security-group-egress \
    --group-id $OUTBOUND_SG_ID \
    --protocol udp \
    --port 53 \
    --cidr $ONPREM_CIDR

echo "✅ Outbound resolver security group created: $OUTBOUND_SG_ID"

echo ""
echo "2. CREATING INBOUND RESOLVER ENDPOINT"

# Create inbound resolver endpoint
INBOUND_ENDPOINT_ID=$(aws route53resolver create-resolver-endpoint \
    --creator-request-id "inbound-$(date +%s)" \
    --name "Hybrid-DNS-Inbound-Endpoint" \
    --security-group-ids $INBOUND_SG_ID \
    --direction INBOUND \
    --ip-addresses SubnetId=${PRIVATE_SUBNET_IDS[0]} SubnetId=${PRIVATE_SUBNET_IDS[1]} \
    --tags Key=Name,Value=Hybrid-DNS-Inbound-Endpoint \
    --query 'ResolverEndpoint.Id' \
    --output text)

echo "✅ Inbound resolver endpoint created: $INBOUND_ENDPOINT_ID"

echo ""
echo "3. CREATING OUTBOUND RESOLVER ENDPOINT"

# Create outbound resolver endpoint
OUTBOUND_ENDPOINT_ID=$(aws route53resolver create-resolver-endpoint \
    --creator-request-id "outbound-$(date +%s)" \
    --name "Hybrid-DNS-Outbound-Endpoint" \
    --security-group-ids $OUTBOUND_SG_ID \
    --direction OUTBOUND \
    --ip-addresses SubnetId=${PRIVATE_SUBNET_IDS[0]} SubnetId=${PRIVATE_SUBNET_IDS[1]} \
    --tags Key=Name,Value=Hybrid-DNS-Outbound-Endpoint \
    --query 'ResolverEndpoint.Id' \
    --output text)

echo "✅ Outbound resolver endpoint created: $OUTBOUND_ENDPOINT_ID"

echo ""
echo "4. WAITING FOR ENDPOINTS TO BECOME AVAILABLE"

# Wait for endpoints to be available
echo "Waiting for inbound endpoint to become available..."
while true; do
    STATUS=$(aws route53resolver get-resolver-endpoint \
        --resolver-endpoint-id $INBOUND_ENDPOINT_ID \
        --query 'ResolverEndpoint.Status' \
        --output text)
    
    if [ "$STATUS" = "OPERATIONAL" ]; then
        echo "✅ Inbound endpoint is operational"
        break
    elif [ "$STATUS" = "CREATING" ]; then
        echo "Inbound endpoint is still creating... waiting"
        sleep 30
    else
        echo "❌ Inbound endpoint creation failed with status: $STATUS"
        exit 1
    fi
done

echo "Waiting for outbound endpoint to become available..."
while true; do
    STATUS=$(aws route53resolver get-resolver-endpoint \
        --resolver-endpoint-id $OUTBOUND_ENDPOINT_ID \
        --query 'ResolverEndpoint.Status' \
        --output text)
    
    if [ "$STATUS" = "OPERATIONAL" ]; then
        echo "✅ Outbound endpoint is operational"
        break
    elif [ "$STATUS" = "CREATING" ]; then
        echo "Outbound endpoint is still creating... waiting"
        sleep 30
    else
        echo "❌ Outbound endpoint creation failed with status: $STATUS"
        exit 1
    fi
done

echo ""
echo "5. GETTING ENDPOINT IP ADDRESSES"

# Get inbound endpoint IPs
INBOUND_IPS=$(aws route53resolver get-resolver-endpoint \
    --resolver-endpoint-id $INBOUND_ENDPOINT_ID \
    --query 'ResolverEndpoint.IpAddresses[*].Ip' \
    --output text)

# Get outbound endpoint IPs
OUTBOUND_IPS=$(aws route53resolver get-resolver-endpoint \
    --resolver-endpoint-id $OUTBOUND_ENDPOINT_ID \
    --query 'ResolverEndpoint.IpAddresses[*].Ip' \
    --output text)

echo "Inbound endpoint IPs: $INBOUND_IPS"
echo "Outbound endpoint IPs: $OUTBOUND_IPS"

# Save configuration
cat << CONFIG > route53-resolver-config.env
export VPC_ID="$VPC_ID"
export PRIVATE_SUBNET_IDS=(${PRIVATE_SUBNET_IDS[@]})
export INBOUND_SG_ID="$INBOUND_SG_ID"
export OUTBOUND_SG_ID="$OUTBOUND_SG_ID"
export INBOUND_ENDPOINT_ID="$INBOUND_ENDPOINT_ID"
export OUTBOUND_ENDPOINT_ID="$OUTBOUND_ENDPOINT_ID"
export INBOUND_IPS="$INBOUND_IPS"
export OUTBOUND_IPS="$OUTBOUND_IPS"
export ONPREM_CIDR="$ONPREM_CIDR"
CONFIG

echo ""
echo "ROUTE 53 RESOLVER INFRASTRUCTURE SETUP COMPLETE"
echo ""
echo "Configuration Summary:"
echo "- VPC: $VPC_ID"
echo "- Inbound Endpoint: $INBOUND_ENDPOINT_ID"
echo "- Outbound Endpoint: $OUTBOUND_ENDPOINT_ID"
echo "- Inbound IPs: $INBOUND_IPS"
echo "- Outbound IPs: $OUTBOUND_IPS"
echo ""
echo "Configuration saved to route53-resolver-config.env"

EOF

chmod +x setup-route53-resolver.sh
./setup-route53-resolver.sh
```

### Step 3: Configure DNS Forwarding Rules
**Objective**: Set up conditional DNS forwarding for hybrid environments

**DNS Forwarding Configuration**:
```bash
# Configure DNS forwarding rules
cat << 'EOF' > configure-dns-forwarding.sh
#!/bin/bash

source route53-resolver-config.env

echo "=== Configuring DNS Forwarding Rules ==="
echo ""

echo "1. CREATING FORWARDING RULES FOR ON-PREMISES DOMAINS"

# Define on-premises DNS servers and domains
ONPREM_DNS_SERVERS=("10.1.1.10" "10.1.1.11")  # Replace with actual DNS server IPs
ONPREM_DOMAINS=("corp.local" "internal.company.com" "ad.company.com")

# Create forwarding rules for each on-premises domain
FORWARDING_RULE_IDS=()

for domain in "${ONPREM_DOMAINS[@]}"; do
    echo "Creating forwarding rule for domain: $domain"
    
    # Create target IPs array for the rule
    TARGET_IPS_JSON=""
    for dns_server in "${ONPREM_DNS_SERVERS[@]}"; do
        if [ -z "$TARGET_IPS_JSON" ]; then
            TARGET_IPS_JSON="Ip=$dns_server,Port=53"
        else
            TARGET_IPS_JSON="$TARGET_IPS_JSON Ip=$dns_server,Port=53"
        fi
    done
    
    # Create forwarding rule
    RULE_ID=$(aws route53resolver create-resolver-rule \
        --creator-request-id "rule-$(date +%s)-$domain" \
        --name "Forward-$domain" \
        --rule-type FORWARD \
        --domain-name $domain \
        --resolver-endpoint-id $OUTBOUND_ENDPOINT_ID \
        --target-ips $TARGET_IPS_JSON \
        --tags Key=Name,Value=Forward-$domain Key=Type,Value=OnPremises \
        --query 'ResolverRule.Id' \
        --output text)
    
    FORWARDING_RULE_IDS+=($RULE_ID)
    echo "✅ Forwarding rule created for $domain: $RULE_ID"
done

echo ""
echo "2. ASSOCIATING FORWARDING RULES WITH VPC"

# Associate each forwarding rule with the VPC
for rule_id in "${FORWARDING_RULE_IDS[@]}"; do
    echo "Associating rule $rule_id with VPC $VPC_ID..."
    
    aws route53resolver associate-resolver-rule \
        --resolver-rule-id $rule_id \
        --vpc-id $VPC_ID \
        --name "VPC-Association-$(date +%s)"
    
    echo "✅ Rule associated with VPC"
done

echo ""
echo "3. CREATING CONDITIONAL FORWARDING FOR AWS SERVICES"

# Create forwarding rule for AWS service domains (if needed)
AWS_SERVICE_RULE_ID=$(aws route53resolver create-resolver-rule \
    --creator-request-id "aws-services-$(date +%s)" \
    --name "AWS-Services-Forward" \
    --rule-type FORWARD \
    --domain-name "aws.company.com" \
    --resolver-endpoint-id $OUTBOUND_ENDPOINT_ID \
    --target-ips Ip=169.254.169.253,Port=53 \
    --tags Key=Name,Value=AWS-Services-Forward Key=Type,Value=AWS \
    --query 'ResolverRule.Id' \
    --output text)

echo "✅ AWS services forwarding rule created: $AWS_SERVICE_RULE_ID"

# Associate AWS services rule with VPC
aws route53resolver associate-resolver-rule \
    --resolver-rule-id $AWS_SERVICE_RULE_ID \
    --vpc-id $VPC_ID \
    --name "AWS-Services-VPC-Association"

echo "✅ AWS services rule associated with VPC"

echo ""
echo "4. CREATING REVERSE DNS FORWARDING RULES"

# Create reverse DNS forwarding for on-premises IP ranges
ONPREM_REVERSE_DOMAINS=("1.10.in-addr.arpa" "2.10.in-addr.arpa")  # Adjust for your IP ranges

for reverse_domain in "${ONPREM_REVERSE_DOMAINS[@]}"; do
    echo "Creating reverse DNS rule for: $reverse_domain"
    
    REVERSE_RULE_ID=$(aws route53resolver create-resolver-rule \
        --creator-request-id "reverse-$(date +%s)-$reverse_domain" \
        --name "Reverse-$reverse_domain" \
        --rule-type FORWARD \
        --domain-name $reverse_domain \
        --resolver-endpoint-id $OUTBOUND_ENDPOINT_ID \
        --target-ips Ip=${ONPREM_DNS_SERVERS[0]},Port=53 Ip=${ONPREM_DNS_SERVERS[1]},Port=53 \
        --tags Key=Name,Value=Reverse-$reverse_domain Key=Type,Value=ReverseDNS \
        --query 'ResolverRule.Id' \
        --output text)
    
    # Associate reverse DNS rule with VPC
    aws route53resolver associate-resolver-rule \
        --resolver-rule-id $REVERSE_RULE_ID \
        --vpc-id $VPC_ID \
        --name "Reverse-VPC-Association-$(date +%s)"
    
    echo "✅ Reverse DNS rule created and associated: $REVERSE_RULE_ID"
done

echo ""
echo "5. VERIFYING FORWARDING RULES"

echo "Listing all resolver rules..."
aws route53resolver list-resolver-rules \
    --query 'ResolverRules[*].[Id,Name,DomainName,Status]' \
    --output table

echo ""
echo "Listing VPC associations..."
aws route53resolver list-resolver-rule-associations \
    --query 'ResolverRuleAssociations[*].[ResolverRuleId,VPCId,Status]' \
    --output table

# Update configuration file
cat << CONFIG >> route53-resolver-config.env

# Forwarding Rules Configuration
export FORWARDING_RULE_IDS=(${FORWARDING_RULE_IDS[@]})
export AWS_SERVICE_RULE_ID="$AWS_SERVICE_RULE_ID"
export ONPREM_DNS_SERVERS=(${ONPREM_DNS_SERVERS[@]})
export ONPREM_DOMAINS=(${ONPREM_DOMAINS[@]})
CONFIG

echo ""
echo "DNS FORWARDING CONFIGURATION COMPLETE"
echo ""
echo "Forwarding Rules Created:"
for i in "${!ONPREM_DOMAINS[@]}"; do
    echo "- ${ONPREM_DOMAINS[$i]} → ${ONPREM_DNS_SERVERS[@]}"
done
echo "- aws.company.com → AWS Resolver"
echo "- Reverse DNS for on-premises IP ranges"
echo ""
echo "All rules are associated with VPC: $VPC_ID"

EOF

chmod +x configure-dns-forwarding.sh
./configure-dns-forwarding.sh
```

### Step 4: Set Up Route 53 Private Hosted Zones
**Objective**: Create private DNS zones for AWS resources

**Private Hosted Zones Setup**:
```bash
# Set up Route 53 private hosted zones
cat << 'EOF' > setup-private-hosted-zones.sh
#!/bin/bash

source route53-resolver-config.env

echo "=== Setting Up Route 53 Private Hosted Zones ==="
echo ""

echo "1. CREATING PRIVATE HOSTED ZONES FOR AWS RESOURCES"

# Create private hosted zone for AWS internal domain
AWS_INTERNAL_DOMAIN="aws.company.com"

AWS_ZONE_ID=$(aws route53 create-hosted-zone \
    --name $AWS_INTERNAL_DOMAIN \
    --vpc VPCRegion=us-east-1,VPCId=$VPC_ID \
    --caller-reference "aws-internal-$(date +%s)" \
    --hosted-zone-config Comment="Private zone for AWS resources",PrivateZone=true \
    --query 'HostedZone.Id' \
    --output text | cut -d'/' -f3)

echo "✅ AWS internal private hosted zone created: $AWS_ZONE_ID"

# Create private hosted zone for development environment
DEV_DOMAIN="dev.aws.company.com"

DEV_ZONE_ID=$(aws route53 create-hosted-zone \
    --name $DEV_DOMAIN \
    --vpc VPCRegion=us-east-1,VPCId=$VPC_ID \
    --caller-reference "dev-$(date +%s)" \
    --hosted-zone-config Comment="Private zone for development resources",PrivateZone=true \
    --query 'HostedZone.Id' \
    --output text | cut -d'/' -f3)

echo "✅ Development private hosted zone created: $DEV_ZONE_ID"

# Create private hosted zone for production environment
PROD_DOMAIN="prod.aws.company.com"

PROD_ZONE_ID=$(aws route53 create-hosted-zone \
    --name $PROD_DOMAIN \
    --vpc VPCRegion=us-east-1,VPCId=$VPC_ID \
    --caller-reference "prod-$(date +%s)" \
    --hosted-zone-config Comment="Private zone for production resources",PrivateZone=true \
    --query 'HostedZone.Id' \
    --output text | cut -d'/' -f3)

echo "✅ Production private hosted zone created: $PROD_ZONE_ID"

echo ""
echo "2. CREATING DNS RECORDS FOR AWS RESOURCES"

# Function to create DNS records
create_dns_record() {
    local zone_id=$1
    local record_name=$2
    local record_type=$3
    local record_value=$4
    local ttl=$5
    
    cat << RECORD_JSON > temp_record.json
{
    "Comment": "Creating $record_name record",
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "$record_name",
                "Type": "$record_type",
                "TTL": $ttl,
                "ResourceRecords": [
                    {
                        "Value": "$record_value"
                    }
                ]
            }
        }
    ]
}
RECORD_JSON

    aws route53 change-resource-record-sets \
        --hosted-zone-id $zone_id \
        --change-batch file://temp_record.json > /dev/null
    
    rm temp_record.json
    echo "✅ Created $record_type record: $record_name → $record_value"
}

# Create records in AWS internal zone
echo "Creating records in AWS internal zone ($AWS_INTERNAL_DOMAIN)..."

create_dns_record $AWS_ZONE_ID "api.$AWS_INTERNAL_DOMAIN" "A" "10.100.1.100" 300
create_dns_record $AWS_ZONE_ID "web.$AWS_INTERNAL_DOMAIN" "A" "10.100.1.101" 300
create_dns_record $AWS_ZONE_ID "db.$AWS_INTERNAL_DOMAIN" "A" "10.100.1.102" 300
create_dns_record $AWS_ZONE_ID "cache.$AWS_INTERNAL_DOMAIN" "A" "10.100.1.103" 300

# Create CNAME records
create_dns_record $AWS_ZONE_ID "www.$AWS_INTERNAL_DOMAIN" "CNAME" "web.$AWS_INTERNAL_DOMAIN" 300
create_dns_record $AWS_ZONE_ID "app.$AWS_INTERNAL_DOMAIN" "CNAME" "api.$AWS_INTERNAL_DOMAIN" 300

# Create records in development zone
echo ""
echo "Creating records in development zone ($DEV_DOMAIN)..."

create_dns_record $DEV_ZONE_ID "api.$DEV_DOMAIN" "A" "10.100.2.100" 300
create_dns_record $DEV_ZONE_ID "web.$DEV_DOMAIN" "A" "10.100.2.101" 300
create_dns_record $DEV_ZONE_ID "test.$DEV_DOMAIN" "A" "10.100.2.102" 300

# Create records in production zone
echo ""
echo "Creating records in production zone ($PROD_DOMAIN)..."

create_dns_record $PROD_ZONE_ID "api.$PROD_DOMAIN" "A" "10.100.3.100" 300
create_dns_record $PROD_ZONE_ID "web.$PROD_DOMAIN" "A" "10.100.3.101" 300
create_dns_record $PROD_ZONE_ID "db.$PROD_DOMAIN" "A" "10.100.3.102" 300

echo ""
echo "3. CREATING SERVICE DISCOVERY RECORDS"

# Create SRV records for service discovery
echo "Creating SRV records for service discovery..."

# SRV record format: priority weight port target
cat << SRV_JSON > srv_record.json
{
    "Comment": "Creating SRV record for LDAP service",
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "_ldap._tcp.$AWS_INTERNAL_DOMAIN",
                "Type": "SRV",
                "TTL": 300,
                "ResourceRecords": [
                    {
                        "Value": "10 10 389 ldap.$AWS_INTERNAL_DOMAIN"
                    }
                ]
            }
        }
    ]
}
SRV_JSON

aws route53 change-resource-record-sets \
    --hosted-zone-id $AWS_ZONE_ID \
    --change-batch file://srv_record.json > /dev/null

rm srv_record.json
echo "✅ Created SRV record for LDAP service"

echo ""
echo "4. SETTING UP HEALTH CHECKS FOR CRITICAL SERVICES"

# Create health check for critical API service
HEALTH_CHECK_ID=$(aws route53 create-health-check \
    --caller-reference "api-health-$(date +%s)" \
    --health-check-config Type=HTTPS,ResourcePath=/health,FullyQualifiedDomainName=api.$PROD_DOMAIN,Port=443,RequestInterval=30,FailureThreshold=3 \
    --query 'HealthCheck.Id' \
    --output text)

echo "✅ Health check created for API service: $HEALTH_CHECK_ID"

# Create failover record with health check
cat << FAILOVER_JSON > failover_record.json
{
    "Comment": "Creating failover record for API service",
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "api-ha.$PROD_DOMAIN",
                "Type": "A",
                "SetIdentifier": "Primary",
                "Failover": "PRIMARY",
                "TTL": 60,
                "ResourceRecords": [
                    {
                        "Value": "10.100.3.100"
                    }
                ],
                "HealthCheckId": "$HEALTH_CHECK_ID"
            }
        },
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "api-ha.$PROD_DOMAIN",
                "Type": "A",
                "SetIdentifier": "Secondary",
                "Failover": "SECONDARY",
                "TTL": 60,
                "ResourceRecords": [
                    {
                        "Value": "10.100.3.200"
                    }
                ]
            }
        }
    ]
}
FAILOVER_JSON

aws route53 change-resource-record-sets \
    --hosted-zone-id $PROD_ZONE_ID \
    --change-batch file://failover_record.json > /dev/null

rm failover_record.json
echo "✅ Failover DNS records created for high availability"

# Update configuration
cat << CONFIG >> route53-resolver-config.env

# Private Hosted Zones Configuration
export AWS_ZONE_ID="$AWS_ZONE_ID"
export DEV_ZONE_ID="$DEV_ZONE_ID"
export PROD_ZONE_ID="$PROD_ZONE_ID"
export AWS_INTERNAL_DOMAIN="$AWS_INTERNAL_DOMAIN"
export DEV_DOMAIN="$DEV_DOMAIN"
export PROD_DOMAIN="$PROD_DOMAIN"
export HEALTH_CHECK_ID="$HEALTH_CHECK_ID"
CONFIG

echo ""
echo "PRIVATE HOSTED ZONES SETUP COMPLETE"
echo ""
echo "Created Hosted Zones:"
echo "- $AWS_INTERNAL_DOMAIN (Zone ID: $AWS_ZONE_ID)"
echo "- $DEV_DOMAIN (Zone ID: $DEV_ZONE_ID)"
echo "- $PROD_DOMAIN (Zone ID: $PROD_ZONE_ID)"
echo ""
echo "DNS Records Created:"
echo "- A records for api, web, db, cache services"
echo "- CNAME records for aliases"
echo "- SRV records for service discovery"
echo "- Failover records for high availability"
echo ""
echo "Health Check: $HEALTH_CHECK_ID"

EOF

chmod +x setup-private-hosted-zones.sh
./setup-private-hosted-zones.sh
```

### Step 5: Configure DNS Query Logging
**Objective**: Enable monitoring and troubleshooting capabilities

**DNS Query Logging Setup**:
```bash
# Set up DNS query logging
cat << 'EOF' > setup-dns-query-logging.sh
#!/bin/bash

source route53-resolver-config.env

echo "=== Setting Up DNS Query Logging ==="
echo ""

echo "1. CREATING CLOUDWATCH LOG GROUP FOR DNS QUERIES"

# Create CloudWatch log group for DNS query logs
LOG_GROUP_NAME="/aws/route53resolver/querylogs"

aws logs create-log-group \
    --log-group-name $LOG_GROUP_NAME \
    --tags Key=Service,Value=Route53Resolver Key=Purpose,Value=QueryLogging

echo "✅ CloudWatch log group created: $LOG_GROUP_NAME"

# Set log retention policy
aws logs put-retention-policy \
    --log-group-name $LOG_GROUP_NAME \
    --retention-in-days 30

echo "✅ Log retention set to 30 days"

echo ""
echo "2. CREATING IAM ROLE FOR QUERY LOGGING"

# Create IAM role for Route 53 Resolver query logging
TRUST_POLICY=$(cat << 'TRUST_POLICY'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "route53resolver.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
TRUST_POLICY
)

# Create the role
ROLE_ARN=$(aws iam create-role \
    --role-name Route53ResolverQueryLoggingRole \
    --assume-role-policy-document "$TRUST_POLICY" \
    --description "Role for Route 53 Resolver query logging" \
    --tags Key=Service,Value=Route53Resolver \
    --query 'Role.Arn' \
    --output text 2>/dev/null || \
    aws iam get-role --role-name Route53ResolverQueryLoggingRole --query 'Role.Arn' --output text)

echo "✅ IAM role created/exists: $ROLE_ARN"

# Create and attach policy for CloudWatch logs
LOGGING_POLICY=$(cat << 'LOGGING_POLICY'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:log-group:/aws/route53resolver/querylogs:*"
        }
    ]
}
LOGGING_POLICY
)

aws iam put-role-policy \
    --role-name Route53ResolverQueryLoggingRole \
    --policy-name Route53ResolverQueryLoggingPolicy \
    --policy-document "$LOGGING_POLICY"

echo "✅ IAM policy attached to role"

echo ""
echo "3. ENABLING QUERY LOGGING FOR VPC"

# Wait a moment for IAM role to propagate
sleep 10

# Enable query logging configuration
QUERY_LOG_CONFIG_ID=$(aws route53resolver create-resolver-query-log-config \
    --name "VPC-DNS-Query-Logging" \
    --destination-arn "arn:aws:logs:us-east-1:$(aws sts get-caller-identity --query 'Account' --output text):log-group:$LOG_GROUP_NAME" \
    --creator-request-id "query-logging-$(date +%s)" \
    --tags Key=Name,Value=VPC-DNS-Query-Logging \
    --query 'ResolverQueryLogConfig.Id' \
    --output text)

echo "✅ Query logging configuration created: $QUERY_LOG_CONFIG_ID"

# Associate query logging with VPC
aws route53resolver associate-resolver-query-log-config \
    --resolver-query-log-config-id $QUERY_LOG_CONFIG_ID \
    --resource-id $VPC_ID \
    --name "VPC-Query-Log-Association"

echo "✅ Query logging associated with VPC: $VPC_ID"

echo ""
echo "4. CREATING CLOUDWATCH DASHBOARD FOR DNS MONITORING"

# Create CloudWatch dashboard for DNS monitoring
DASHBOARD_BODY='{
    "widgets": [
        {
            "type": "log",
            "x": 0,
            "y": 0,
            "width": 24,
            "height": 6,
            "properties": {
                "query": "SOURCE '\''$LOG_GROUP_NAME'\''\n| fields @timestamp, query_name, query_type, response_code, srcaddr\n| filter @message like /NOERROR/ or @message like /NXDOMAIN/\n| stats count() by query_name\n| sort count desc\n| limit 20",
                "region": "us-east-1",
                "title": "Top DNS Queries",
                "view": "table"
            }
        },
        {
            "type": "log",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "query": "SOURCE '\''$LOG_GROUP_NAME'\''\n| fields @timestamp, response_code\n| filter response_code = '\''NXDOMAIN'\''\n| stats count() by bin(5m)",
                "region": "us-east-1",
                "title": "DNS Resolution Failures (NXDOMAIN)",
                "view": "line"
            }
        },
        {
            "type": "log",
            "x": 12,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "query": "SOURCE '\''$LOG_GROUP_NAME'\''\n| fields @timestamp, srcaddr\n| stats count() by srcaddr\n| sort count desc\n| limit 10",
                "region": "us-east-1",
                "title": "Top DNS Clients",
                "view": "table"
            }
        }
    ]
}'

aws cloudwatch put-dashboard \
    --dashboard-name "Route53-Resolver-DNS-Monitoring" \
    --dashboard-body "$DASHBOARD_BODY"

echo "✅ CloudWatch dashboard created: Route53-Resolver-DNS-Monitoring"

echo ""
echo "5. CREATING CLOUDWATCH ALARMS FOR DNS ISSUES"

# Create alarm for high DNS failure rate
aws cloudwatch put-metric-alarm \
    --alarm-name "DNS-High-Failure-Rate" \
    --alarm-description "Alert when DNS failure rate is high" \
    --metric-name "QueryCount" \
    --namespace "AWS/Route53Resolver" \
    --statistic "Sum" \
    --period 300 \
    --threshold 100 \
    --comparison-operator "GreaterThanThreshold" \
    --evaluation-periods 2 \
    --dimensions Name=VPC,Value=$VPC_ID

echo "✅ DNS failure rate alarm created"

echo ""
echo "6. CREATING DNS QUERY ANALYSIS SCRIPT"

cat << 'ANALYSIS_SCRIPT' > analyze-dns-queries.sh
#!/bin/bash

# DNS Query Analysis Script

LOG_GROUP_NAME="$LOG_GROUP_NAME"
HOURS_AGO=${1:-1}  # Default to last 1 hour

echo "=== DNS Query Analysis (Last $HOURS_AGO hours) ==="
echo ""

# Calculate start time
START_TIME=$(date -u -d "$HOURS_AGO hours ago" +%s)000
END_TIME=$(date -u +%s)000

echo "1. TOP QUERIED DOMAINS"
aws logs start-query \
    --log-group-name $LOG_GROUP_NAME \
    --start-time $START_TIME \
    --end-time $END_TIME \
    --query-string 'fields @timestamp, query_name, response_code | filter response_code = "NOERROR" | stats count() by query_name | sort count desc | limit 20' \
    --query 'queryId' \
    --output text > query_id.tmp

sleep 5

aws logs get-query-results \
    --query-id $(cat query_id.tmp) \
    --query 'results[*][1].value' \
    --output table

echo ""
echo "2. DNS RESOLUTION FAILURES"
aws logs start-query \
    --log-group-name $LOG_GROUP_NAME \
    --start-time $START_TIME \
    --end-time $END_TIME \
    --query-string 'fields @timestamp, query_name, response_code | filter response_code = "NXDOMAIN" | stats count() by query_name | sort count desc | limit 10' \
    --query 'queryId' \
    --output text > query_id2.tmp

sleep 5

aws logs get-query-results \
    --query-id $(cat query_id2.tmp) \
    --query 'results[*][1].value' \
    --output table

echo ""
echo "3. TOP CLIENT IPs"
aws logs start-query \
    --log-group-name $LOG_GROUP_NAME \
    --start-time $START_TIME \
    --end-time $END_TIME \
    --query-string 'fields @timestamp, srcaddr | stats count() by srcaddr | sort count desc | limit 10' \
    --query 'queryId' \
    --output text > query_id3.tmp

sleep 5

aws logs get-query-results \
    --query-id $(cat query_id3.tmp) \
    --query 'results[*][1].value' \
    --output table

# Cleanup
rm -f query_id*.tmp

echo ""
echo "Analysis completed. Check CloudWatch dashboard for real-time monitoring."

ANALYSIS_SCRIPT

chmod +x analyze-dns-queries.sh

# Update configuration
cat << CONFIG >> route53-resolver-config.env

# DNS Query Logging Configuration
export LOG_GROUP_NAME="$LOG_GROUP_NAME"
export ROLE_ARN="$ROLE_ARN"
export QUERY_LOG_CONFIG_ID="$QUERY_LOG_CONFIG_ID"
CONFIG

echo ""
echo "DNS QUERY LOGGING SETUP COMPLETE"
echo ""
echo "Logging Configuration:"
echo "- CloudWatch Log Group: $LOG_GROUP_NAME"
echo "- IAM Role: $ROLE_ARN"
echo "- Query Log Config: $QUERY_LOG_CONFIG_ID"
echo "- Dashboard: Route53-Resolver-DNS-Monitoring"
echo ""
echo "Analysis Tools:"
echo "- analyze-dns-queries.sh [hours] - Analyze DNS query patterns"
echo "- CloudWatch Insights queries for detailed analysis"
echo ""
echo "View logs: AWS Console → CloudWatch → Log Groups → $LOG_GROUP_NAME"

EOF

chmod +x setup-dns-query-logging.sh
./setup-dns-query-logging.sh
```

### Step 6: Test and Validate Hybrid DNS
**Objective**: Verify DNS resolution and troubleshoot issues

**DNS Testing and Validation**:
```bash
# Test and validate hybrid DNS setup
cat << 'EOF' > test-hybrid-dns.sh
#!/bin/bash

source route53-resolver-config.env

echo "=== Testing and Validating Hybrid DNS Setup ==="
echo ""

echo "1. TESTING DNS RESOLUTION FROM VPC"

# Test resolving AWS internal domains
echo "Testing AWS internal domain resolution..."
for domain in "api.$AWS_INTERNAL_DOMAIN" "web.$AWS_INTERNAL_DOMAIN" "db.$AWS_INTERNAL_DOMAIN"; do
    echo "Testing resolution: $domain"
    
    # Use nslookup if available
    if command -v nslookup >/dev/null 2>&1; then
        nslookup $domain 169.254.169.253
    fi
    
    # Use dig if available
    if command -v dig >/dev/null 2>&1; then
        dig @169.254.169.253 $domain +short
    fi
    
    echo ""
done

echo ""
echo "2. TESTING ON-PREMISES DOMAIN FORWARDING"

# Test forwarding to on-premises domains (simulated)
echo "Testing on-premises domain forwarding..."
for domain in "${ONPREM_DOMAINS[@]}"; do
    echo "Testing forwarding: $domain"
    echo "Note: This requires actual on-premises DNS servers to be configured"
    
    # Show the forwarding rule configuration
    aws route53resolver list-resolver-rules \
        --filters Name=DomainName,Values=$domain \
        --query 'ResolverRules[*].[Name,DomainName,Status,TargetIps[*].Ip]' \
        --output table
    
    echo ""
done

echo ""
echo "3. TESTING REVERSE DNS RESOLUTION"

# Test reverse DNS resolution
echo "Testing reverse DNS resolution..."
TEST_IPS=("10.100.1.100" "10.100.2.100" "10.100.3.100")

for ip in "${TEST_IPS[@]}"; do
    echo "Testing reverse resolution for: $ip"
    
    if command -v dig >/dev/null 2>&1; then
        # Convert IP to reverse DNS format
        IFS='.' read -ra ADDR <<< "$ip"
        REVERSE_IP="${ADDR[3]}.${ADDR[2]}.${ADDR[1]}.${ADDR[0]}.in-addr.arpa"
        
        dig @169.254.169.253 $REVERSE_IP PTR +short
    fi
    
    echo ""
done

echo ""
echo "4. TESTING DNS PERFORMANCE"

# Performance testing for DNS resolution
echo "Testing DNS resolution performance..."

# Function to measure DNS query time
measure_dns_time() {
    local domain=$1
    local dns_server=$2
    
    if command -v dig >/dev/null 2>&1; then
        echo "Query time for $domain using $dns_server:"
        dig @$dns_server $domain | grep "Query time"
    fi
}

# Test performance for different domains
for domain in "api.$AWS_INTERNAL_DOMAIN" "www.example.com" "aws.amazon.com"; do
    measure_dns_time $domain "169.254.169.253"
done

echo ""
echo "5. TESTING DNS FAILOVER"

# Test DNS health checks and failover
echo "Testing DNS failover configuration..."

# Check health check status
if [ ! -z "$HEALTH_CHECK_ID" ]; then
    echo "Health check status for API service:"
    aws route53 get-health-check \
        --health-check-id $HEALTH_CHECK_ID \
        --query 'HealthCheck.[Id,HealthCheckConfig.FullyQualifiedDomainName,Status]' \
        --output table
    
    # Get health check status
    aws route53 get-health-check-status \
        --health-check-id $HEALTH_CHECK_ID \
        --query 'CheckStatusList[*].[IPAddress,Status,Region]' \
        --output table
fi

echo ""
echo "6. CREATING DNS TESTING SCRIPT FOR ONGOING VALIDATION"

cat << 'DNS_TEST_SCRIPT' > ongoing-dns-test.sh
#!/bin/bash

# Ongoing DNS testing script

source route53-resolver-config.env

echo "=== Ongoing DNS Validation Test ==="
echo "Test started at: $(date)"
echo ""

# Function to test DNS resolution
test_dns_resolution() {
    local domain=$1
    local expected_ip=$2
    local dns_server=${3:-169.254.169.253}
    
    echo "Testing: $domain"
    
    if command -v dig >/dev/null 2>&1; then
        RESULT=$(dig @$dns_server $domain +short | head -1)
        
        if [ "$RESULT" = "$expected_ip" ]; then
            echo "✅ PASS: $domain resolves to $RESULT"
        elif [ -z "$RESULT" ]; then
            echo "❌ FAIL: $domain does not resolve"
        else
            echo "⚠️  WARNING: $domain resolves to $RESULT (expected $expected_ip)"
        fi
    else
        echo "dig command not available for testing"
    fi
    
    echo ""
}

# Test critical DNS records
echo "1. TESTING CRITICAL DNS RECORDS"
test_dns_resolution "api.$AWS_INTERNAL_DOMAIN" "10.100.1.100"
test_dns_resolution "web.$AWS_INTERNAL_DOMAIN" "10.100.1.101"
test_dns_resolution "db.$AWS_INTERNAL_DOMAIN" "10.100.1.102"

echo ""
echo "2. TESTING EXTERNAL DNS RESOLUTION"
test_dns_resolution "www.amazon.com" "" # Don't check specific IP for external domains
test_dns_resolution "aws.amazon.com" ""

echo ""
echo "3. TESTING DNS QUERY RESPONSE TIME"

if command -v dig >/dev/null 2>&1; then
    echo "DNS Response Times:"
    for domain in "api.$AWS_INTERNAL_DOMAIN" "www.amazon.com"; do
        echo "Testing response time for: $domain"
        dig @169.254.169.253 $domain | grep "Query time" || echo "No timing information available"
    done
fi

echo ""
echo "4. CHECKING RESOLVER ENDPOINT STATUS"

# Check resolver endpoint health
for endpoint_id in $INBOUND_ENDPOINT_ID $OUTBOUND_ENDPOINT_ID; do
    STATUS=$(aws route53resolver get-resolver-endpoint \
        --resolver-endpoint-id $endpoint_id \
        --query 'ResolverEndpoint.Status' \
        --output text)
    
    DIRECTION=$(aws route53resolver get-resolver-endpoint \
        --resolver-endpoint-id $endpoint_id \
        --query 'ResolverEndpoint.Direction' \
        --output text)
    
    if [ "$STATUS" = "OPERATIONAL" ]; then
        echo "✅ $DIRECTION endpoint ($endpoint_id) is operational"
    else
        echo "❌ $DIRECTION endpoint ($endpoint_id) status: $STATUS"
    fi
done

echo ""
echo "DNS validation test completed at: $(date)"
echo ""
echo "Summary:"
echo "- Run this script regularly to monitor DNS health"
echo "- Check CloudWatch logs for detailed query analysis"
echo "- Monitor resolver endpoint status in AWS console"

DNS_TEST_SCRIPT

chmod +x ongoing-dns-test.sh

echo ""
echo "7. CREATING DNS TROUBLESHOOTING GUIDE"

cat << 'TROUBLESHOOTING_GUIDE' > dns-troubleshooting-guide.md
# DNS Troubleshooting Guide for Hybrid Route 53 Resolver

## Common Issues and Solutions

### 1. DNS Resolution Not Working

**Symptoms:**
- Domains not resolving from VPC instances
- NXDOMAIN responses for valid domains

**Troubleshooting Steps:**
1. Check resolver endpoint status:
   ```bash
   aws route53resolver get-resolver-endpoint --resolver-endpoint-id <endpoint-id>
   ```

2. Verify security group rules:
   - Inbound endpoint: Allow TCP/UDP 53 from on-premises
   - Outbound endpoint: Allow TCP/UDP 53 to on-premises DNS

3. Check forwarding rule associations:
   ```bash
   aws route53resolver list-resolver-rule-associations
   ```

4. Test DNS resolution manually:
   ```bash
   dig @169.254.169.253 domain.name
   ```

### 2. Slow DNS Resolution

**Symptoms:**
- High DNS query response times
- Application timeouts

**Troubleshooting Steps:**
1. Check resolver endpoint region/AZ placement
2. Verify network connectivity to on-premises DNS
3. Monitor DNS query logs for patterns
4. Check on-premises DNS server performance

### 3. Forwarding Rules Not Working

**Symptoms:**
- Queries not being forwarded to on-premises
- Getting AWS responses for on-premises domains

**Troubleshooting Steps:**
1. Verify rule domain name matches exactly
2. Check rule association with VPC
3. Verify target DNS server accessibility
4. Test outbound endpoint connectivity

### 4. High DNS Query Volume

**Symptoms:**
- Unexpected high DNS query volume
- DNS resolver costs increasing

**Investigation Steps:**
1. Analyze query logs for patterns
2. Identify top querying clients
3. Look for DNS amplification or loops
4. Implement DNS caching strategies

## Monitoring and Alerting

### Key Metrics to Monitor:
- DNS query volume and patterns
- DNS resolution failures (NXDOMAIN)
- Resolver endpoint health
- Query response times

### CloudWatch Insights Queries:

#### Top Queried Domains:
```
fields @timestamp, query_name, response_code
| filter response_code = "NOERROR"
| stats count() by query_name
| sort count desc
| limit 20
```

#### DNS Failures:
```
fields @timestamp, query_name, response_code
| filter response_code = "NXDOMAIN"
| stats count() by query_name
| sort count desc
| limit 10
```

#### Query Volume by Client:
```
fields @timestamp, srcaddr
| stats count() by srcaddr
| sort count desc
| limit 10
```

## Best Practices

1. **Security Groups**: Properly configure SG rules for resolver endpoints
2. **High Availability**: Deploy endpoints across multiple AZs
3. **Monitoring**: Enable query logging and set up alerting
4. **Performance**: Place resolver endpoints close to workloads
5. **Cost Optimization**: Monitor query patterns and implement caching

TROUBLESHOOTING_GUIDE

echo "✅ DNS troubleshooting guide created: dns-troubleshooting-guide.md"

echo ""
echo "HYBRID DNS TESTING AND VALIDATION COMPLETE"
echo ""
echo "Testing Tools Created:"
echo "- ongoing-dns-test.sh - Regular DNS health validation"
echo "- dns-troubleshooting-guide.md - Comprehensive troubleshooting guide"
echo "- analyze-dns-queries.sh - Query pattern analysis"
echo ""
echo "Key Testing Points:"
echo "✅ AWS internal domain resolution"
echo "✅ On-premises domain forwarding rules"
echo "✅ Reverse DNS configuration"
echo "✅ DNS performance measurement"
echo "✅ Health check and failover testing"
echo ""
echo "Monitor DNS health regularly and check CloudWatch logs for issues"

EOF

chmod +x test-hybrid-dns.sh
./test-hybrid-dns.sh
```

## Route 53 Resolver Best Practices

### Design Principles
- **High Availability**: Deploy resolver endpoints across multiple AZs
- **Security**: Use security groups to control DNS traffic access
- **Performance**: Place endpoints close to workloads and DNS servers
- **Monitoring**: Enable comprehensive query logging and alerting

### Network Architecture
- **Endpoint Placement**: Deploy in private subnets for security
- **IP Address Planning**: Reserve IP addresses for resolver endpoints
- **Connectivity**: Ensure reliable connectivity to on-premises DNS
- **Redundancy**: Use multiple on-premises DNS servers for forwarding

### DNS Design Patterns
- **Conditional Forwarding**: Forward specific domains to appropriate resolvers
- **Split-Horizon DNS**: Different responses for internal vs external clients
- **Domain Delegation**: Proper delegation between AWS and on-premises
- **Reverse DNS**: Configure reverse lookups for both environments

### Security Considerations
- **Access Control**: Use security groups and NACLs to control DNS traffic
- **DNSSEC**: Enable DNSSEC validation where supported
- **Query Logging**: Monitor DNS queries for security analysis
- **Network Segmentation**: Isolate DNS traffic using proper subnets

### Performance Optimization
- **Caching**: Implement appropriate DNS caching strategies
- **Query Patterns**: Analyze and optimize frequent DNS queries
- **Endpoint Sizing**: Right-size resolver endpoints for query volume
- **Network Path**: Optimize network paths to DNS servers

## Key Takeaways
- Route 53 Resolver enables seamless hybrid DNS architectures
- Inbound and outbound endpoints provide bidirectional DNS forwarding
- Forwarding rules enable conditional DNS resolution based on domain names
- Query logging provides valuable insights for monitoring and troubleshooting
- Private hosted zones integrate with on-premises DNS infrastructure
- Proper security group configuration is critical for DNS traffic flow

## Additional Resources
- [Route 53 Resolver Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html)
- [DNS Resolution in Hybrid Architectures](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-overview-DSN-queries.html)
- [Route 53 Resolver Best Practices](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-best-practices.html)

## Exam Tips
- Resolver endpoints require 2-8 IP addresses across multiple AZs
- Inbound endpoints receive queries from on-premises networks
- Outbound endpoints forward queries to on-premises DNS servers
- Forwarding rules support domain-based conditional forwarding
- Query logging requires CloudWatch Logs destination
- Security groups control DNS traffic (TCP/UDP port 53)
- VPC associations determine which VPCs use resolver rules
- System rules are AWS-managed and cannot be modified
- Resolver endpoints are regional resources with cross-AZ redundancy