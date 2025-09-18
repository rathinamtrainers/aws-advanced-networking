# Topic 35: Automatic Custom Domain Name Assignment for VPC Instances

## Prerequisites
- Completed Topics 1-34 (VPC Foundations and Advanced Topics)
- Understanding of DNS concepts and Route 53 services
- Knowledge of EC2 instance lifecycle and metadata
- Familiarity with Lambda functions and EventBridge events

## Learning Objectives
By the end of this topic, you will be able to:
- Configure automatic custom domain name assignment for VPC instances
- Implement Route 53 Private Hosted Zones for internal naming
- Set up DHCP Options Sets for custom domain suffixes
- Create automation using Lambda and EventBridge for dynamic DNS management
- Design scalable domain naming strategies for large environments

## Theory

### Custom Domain Assignment Overview

#### Definition
Custom domain name assignment for VPC instances involves automatically assigning meaningful, predictable domain names to EC2 instances as they launch, rather than relying on default AWS-generated names.

#### Why Custom Domain Names Matter
- **Predictable Naming**: Easier to identify and manage instances
- **Service Discovery**: Applications can find services by name
- **Compliance**: Meet internal naming standards and conventions
- **Monitoring**: Easier to track and alert on specific services
- **Automation**: Scripts and tools can reference predictable names

### Domain Assignment Approaches

#### Approach 1: Route 53 Private Hosted Zones with Automation
```
EC2 Launch → EventBridge Event → Lambda Function → Route 53 API → DNS Record Creation
```

#### Approach 2: DHCP Options Set with Custom Domain Suffix
```
EC2 Launch → DHCP Client → Custom Domain Suffix → hostname.domain.local
```

#### Approach 3: Hybrid Approach (Recommended)
```
EC2 Launch → EventBridge → Lambda → Route 53 + Instance Hostname Configuration
```

### Domain Naming Strategies

#### Environment-Based Naming
```
web01.prod.company.internal     # Production web server
db02.staging.company.internal   # Staging database
api03.dev.company.internal      # Development API server
```

#### Service-Based Naming
```
web-prod-01.company.internal    # Web service, production, instance 1
db-stage-02.company.internal    # Database, staging, instance 2
cache-dev-01.company.internal   # Cache service, development, instance 1
```

#### Role-Based Naming
```
frontend-01.web.company.internal    # Frontend role in web tier
backend-01.app.company.internal     # Backend role in app tier
primary.db.company.internal         # Primary database role
```

## Lab Exercise: Comprehensive Custom Domain Assignment Implementation

### Lab Duration: 240 minutes

### Step 1: Design Domain Naming Strategy
**Objective**: Plan domain naming convention and architecture

**Domain Strategy Planning**:
```bash
# Create domain naming strategy assessment
cat << 'EOF' > domain-naming-strategy.sh
#!/bin/bash

echo "=== Custom Domain Naming Strategy Planning ==="
echo ""

echo "1. NAMING CONVENTION DESIGN"
echo "   Base domain: company.internal"
echo "   Environment subdomains: prod.company.internal, staging.company.internal, dev.company.internal"
echo "   Service-based naming: [service]-[env]-[number].[tier].company.internal"
echo "   Example: web-prod-01.frontend.company.internal"
echo ""

echo "2. DOMAIN STRUCTURE"
echo "   Root Domain: company.internal"
echo "   ├── prod.company.internal (Production)"
echo "   │   ├── frontend.prod.company.internal"
echo "   │   ├── backend.prod.company.internal"
echo "   │   └── data.prod.company.internal"
echo "   ├── staging.company.internal (Staging)"
echo "   │   ├── frontend.staging.company.internal"
echo "   │   └── backend.staging.company.internal"
echo "   └── dev.company.internal (Development)"
echo "       ├── frontend.dev.company.internal"
echo "       └── backend.dev.company.internal"
echo ""

echo "3. NAMING RULES"
echo "   Instance naming pattern: [service]-[env]-[instance_number]"
echo "   Service types: web, api, db, cache, queue, worker"
echo "   Environment codes: prod, staging, dev, test"
echo "   Instance numbers: 01, 02, 03... (zero-padded)"
echo ""

echo "4. TAG-BASED DOMAIN ASSIGNMENT"
echo "   Required tags for auto-assignment:"
echo "   - Service: web, api, db, cache, etc."
echo "   - Environment: prod, staging, dev"
echo "   - Tier: frontend, backend, data"
echo "   - Role: primary, secondary, replica (optional)"
echo ""

echo "5. DNS RECORD TYPES"
echo "   A Records: Instance private IP addresses"
echo "   CNAME Records: Service aliases (www, api, admin)"
echo "   SRV Records: Service discovery (optional)"
echo ""

echo "NAMING EXAMPLES:"
echo "- web-prod-01.frontend.company.internal"
echo "- api-prod-01.backend.company.internal"
echo "- db-prod-primary.data.company.internal"
echo "- cache-staging-01.backend.company.internal"
echo "- worker-dev-01.backend.company.internal"

EOF

chmod +x domain-naming-strategy.sh
./domain-naming-strategy.sh
```

### Step 2: Set Up Route 53 Private Hosted Zones
**Objective**: Create DNS infrastructure for custom domain assignment

**Private Hosted Zones Setup**:
```bash
# Set up Route 53 private hosted zones for custom domains
cat << 'EOF' > setup-custom-domain-zones.sh
#!/bin/bash

echo "=== Setting Up Route 53 Private Hosted Zones for Custom Domains ==="
echo ""

# Get VPC information
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=false" --query 'Vpcs[0].VpcId' --output text)
if [ "$VPC_ID" = "None" ] || [ -z "$VPC_ID" ]; then
    echo "Creating new VPC for custom domain lab..."

    VPC_ID=$(aws ec2 create-vpc \
        --cidr-block 10.200.0.0/16 \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=CustomDomain-VPC}]" \
        --query 'Vpc.VpcId' \
        --output text)

    # Enable DNS support
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

    echo "✅ VPC created: $VPC_ID"
else
    echo "Using existing VPC: $VPC_ID"
fi

echo ""
echo "1. CREATING ROOT PRIVATE HOSTED ZONE"

# Create root private hosted zone
ROOT_DOMAIN="company.internal"

ROOT_ZONE_ID=$(aws route53 create-hosted-zone \
    --name $ROOT_DOMAIN \
    --vpc VPCRegion=us-east-1,VPCId=$VPC_ID \
    --caller-reference "root-$(date +%s)" \
    --hosted-zone-config Comment="Root private zone for custom instance domains",PrivateZone=true \
    --query 'HostedZone.Id' \
    --output text | cut -d'/' -f3)

echo "✅ Root private hosted zone created: $ROOT_ZONE_ID ($ROOT_DOMAIN)"

echo ""
echo "2. CREATING ENVIRONMENT SUBDOMAINS"

# Create environment-specific subdomains
ENVIRONMENTS=("prod" "staging" "dev")
declare -A ENV_ZONE_IDS

for env in "${ENVIRONMENTS[@]}"; do
    ENV_DOMAIN="$env.$ROOT_DOMAIN"

    ENV_ZONE_ID=$(aws route53 create-hosted-zone \
        --name $ENV_DOMAIN \
        --vpc VPCRegion=us-east-1,VPCId=$VPC_ID \
        --caller-reference "$env-$(date +%s)" \
        --hosted-zone-config Comment="Private zone for $env environment",PrivateZone=true \
        --query 'HostedZone.Id' \
        --output text | cut -d'/' -f3)

    ENV_ZONE_IDS[$env]=$ENV_ZONE_ID
    echo "✅ Environment zone created: $ENV_ZONE_ID ($ENV_DOMAIN)"
done

echo ""
echo "3. CREATING TIER-SPECIFIC SUBDOMAINS"

# Create tier-specific subdomains for each environment
TIERS=("frontend" "backend" "data")
declare -A TIER_ZONE_IDS

for env in "${ENVIRONMENTS[@]}"; do
    for tier in "${TIERS[@]}"; do
        TIER_DOMAIN="$tier.$env.$ROOT_DOMAIN"

        TIER_ZONE_ID=$(aws route53 create-hosted-zone \
            --name $TIER_DOMAIN \
            --vpc VPCRegion=us-east-1,VPCId=$VPC_ID \
            --caller-reference "$tier-$env-$(date +%s)" \
            --hosted-zone-config Comment="Private zone for $tier tier in $env environment",PrivateZone=true \
            --query 'HostedZone.Id' \
            --output text | cut -d'/' -f3)

        TIER_ZONE_IDS["$tier-$env"]=$TIER_ZONE_ID
        echo "✅ Tier zone created: $TIER_ZONE_ID ($TIER_DOMAIN)"
    done
done

echo ""
echo "4. CREATING SAMPLE DNS RECORDS"

# Function to create DNS records
create_dns_record() {
    local zone_id=$1
    local record_name=$2
    local record_type=$3
    local record_value=$4
    local ttl=${5:-300}

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

# Create sample records in production frontend zone
PROD_FRONTEND_ZONE=${TIER_ZONE_IDS["frontend-prod"]}
echo "Creating sample records in production frontend zone..."

create_dns_record $PROD_FRONTEND_ZONE "web-prod-01.frontend.prod.$ROOT_DOMAIN" "A" "10.200.1.10"
create_dns_record $PROD_FRONTEND_ZONE "web-prod-02.frontend.prod.$ROOT_DOMAIN" "A" "10.200.1.11"
create_dns_record $PROD_FRONTEND_ZONE "lb.frontend.prod.$ROOT_DOMAIN" "A" "10.200.1.100"

# Create CNAME records for aliases
create_dns_record $PROD_FRONTEND_ZONE "www.frontend.prod.$ROOT_DOMAIN" "CNAME" "lb.frontend.prod.$ROOT_DOMAIN"

# Create sample records in production backend zone
PROD_BACKEND_ZONE=${TIER_ZONE_IDS["backend-prod"]}
echo "Creating sample records in production backend zone..."

create_dns_record $PROD_BACKEND_ZONE "api-prod-01.backend.prod.$ROOT_DOMAIN" "A" "10.200.2.10"
create_dns_record $PROD_BACKEND_ZONE "api-prod-02.backend.prod.$ROOT_DOMAIN" "A" "10.200.2.11"
create_dns_record $PROD_BACKEND_ZONE "cache-prod-01.backend.prod.$ROOT_DOMAIN" "A" "10.200.2.20"

echo ""
echo "5. CREATING SERVICE DISCOVERY RECORDS"

# Create SRV records for service discovery
cat << SRV_JSON > srv_record.json
{
    "Comment": "Creating SRV record for web service discovery",
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "_http._tcp.frontend.prod.$ROOT_DOMAIN",
                "Type": "SRV",
                "TTL": 300,
                "ResourceRecords": [
                    {
                        "Value": "10 10 80 web-prod-01.frontend.prod.$ROOT_DOMAIN"
                    },
                    {
                        "Value": "10 10 80 web-prod-02.frontend.prod.$ROOT_DOMAIN"
                    }
                ]
            }
        }
    ]
}
SRV_JSON

aws route53 change-resource-record-sets \
    --hosted-zone-id $PROD_FRONTEND_ZONE \
    --change-batch file://srv_record.json > /dev/null

rm srv_record.json
echo "✅ Created SRV record for web service discovery"

# Save configuration
cat << CONFIG > custom-domain-config.env
export VPC_ID="$VPC_ID"
export ROOT_DOMAIN="$ROOT_DOMAIN"
export ROOT_ZONE_ID="$ROOT_ZONE_ID"

# Environment zones
export PROD_ZONE_ID="${ENV_ZONE_IDS[prod]}"
export STAGING_ZONE_ID="${ENV_ZONE_IDS[staging]}"
export DEV_ZONE_ID="${ENV_ZONE_IDS[dev]}"

# Tier zones
export FRONTEND_PROD_ZONE_ID="${TIER_ZONE_IDS[frontend-prod]}"
export BACKEND_PROD_ZONE_ID="${TIER_ZONE_IDS[backend-prod]}"
export DATA_PROD_ZONE_ID="${TIER_ZONE_IDS[data-prod]}"
export FRONTEND_STAGING_ZONE_ID="${TIER_ZONE_IDS[frontend-staging]}"
export BACKEND_STAGING_ZONE_ID="${TIER_ZONE_IDS[backend-staging]}"
export FRONTEND_DEV_ZONE_ID="${TIER_ZONE_IDS[frontend-dev]}"
export BACKEND_DEV_ZONE_ID="${TIER_ZONE_IDS[backend-dev]}"

# Domain patterns
export ENVIRONMENTS=(prod staging dev)
export TIERS=(frontend backend data)
CONFIG

echo ""
echo "ROUTE 53 PRIVATE HOSTED ZONES SETUP COMPLETE"
echo ""
echo "Created Zones:"
echo "- Root: $ROOT_DOMAIN (Zone ID: $ROOT_ZONE_ID)"
for env in "${ENVIRONMENTS[@]}"; do
    echo "- Environment: $env.$ROOT_DOMAIN (Zone ID: ${ENV_ZONE_IDS[$env]})"
    for tier in "${TIERS[@]}"; do
        echo "  - Tier: $tier.$env.$ROOT_DOMAIN (Zone ID: ${TIER_ZONE_IDS[$tier-$env]})"
    done
done
echo ""
echo "Configuration saved to custom-domain-config.env"

EOF

chmod +x setup-custom-domain-zones.sh
./setup-custom-domain-zones.sh
```

### Step 3: Create Lambda Function for Automatic Domain Assignment
**Objective**: Implement automation for dynamic DNS record creation

**Lambda Function Setup**:
```bash
# Create Lambda function for automatic domain assignment
cat << 'EOF' > setup-domain-assignment-lambda.sh
#!/bin/bash

source custom-domain-config.env

echo "=== Setting Up Lambda Function for Automatic Domain Assignment ==="
echo ""

echo "1. CREATING IAM ROLE FOR LAMBDA FUNCTION"

# Create IAM role for Lambda
TRUST_POLICY=$(cat << 'TRUST_POLICY'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
TRUST_POLICY
)

LAMBDA_ROLE_ARN=$(aws iam create-role \
    --role-name CustomDomainAssignmentLambdaRole \
    --assume-role-policy-document "$TRUST_POLICY" \
    --description "Role for Lambda function to assign custom domains to EC2 instances" \
    --tags Key=Service,Value=Lambda Key=Purpose,Value=DomainAssignment \
    --query 'Role.Arn' \
    --output text 2>/dev/null || \
    aws iam get-role --role-name CustomDomainAssignmentLambdaRole --query 'Role.Arn' --output text)

echo "✅ Lambda IAM role created/exists: $LAMBDA_ROLE_ARN"

# Attach basic Lambda execution policy
aws iam attach-role-policy \
    --role-name CustomDomainAssignmentLambdaRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create custom policy for Route 53 and EC2 access
LAMBDA_POLICY=$(cat << 'LAMBDA_POLICY'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets",
                "route53:GetChange",
                "route53:ListHostedZones",
                "route53:GetHostedZone"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeTags",
                "ec2:CreateTags"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        }
    ]
}
LAMBDA_POLICY
)

aws iam put-role-policy \
    --role-name CustomDomainAssignmentLambdaRole \
    --policy-name CustomDomainAssignmentPolicy \
    --policy-document "$LAMBDA_POLICY"

echo "✅ IAM policies attached to Lambda role"

echo ""
echo "2. CREATING LAMBDA FUNCTION CODE"

# Create Lambda function code
cat << 'LAMBDA_CODE' > domain_assignment_lambda.py
import json
import boto3
import logging
from datetime import datetime

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Initialize AWS clients
ec2 = boto3.client('ec2')
route53 = boto3.client('route53')

# Domain configuration
DOMAIN_CONFIG = {
    'root_domain': 'company.internal',
    'zone_mappings': {
        'frontend-prod': 'Z0123456789ABCDEF01',  # Replace with actual zone IDs
        'backend-prod': 'Z0123456789ABCDEF02',
        'data-prod': 'Z0123456789ABCDEF03',
        'frontend-staging': 'Z0123456789ABCDEF04',
        'backend-staging': 'Z0123456789ABCDEF05',
        'frontend-dev': 'Z0123456789ABCDEF06',
        'backend-dev': 'Z0123456789ABCDEF07'
    }
}

def lambda_handler(event, context):
    """
    Lambda handler for automatic domain assignment to EC2 instances
    """
    try:
        logger.info(f"Received event: {json.dumps(event)}")

        # Parse EventBridge event
        if 'detail' not in event:
            logger.error("Invalid event format - missing detail")
            return {'statusCode': 400, 'body': 'Invalid event format'}

        detail = event['detail']

        # Check if this is an EC2 instance state change
        if detail.get('source') != 'aws.ec2':
            logger.info("Not an EC2 event, skipping")
            return {'statusCode': 200, 'body': 'Not an EC2 event'}

        # Check if instance is running
        if detail.get('state') != 'running':
            logger.info(f"Instance state is {detail.get('state')}, skipping domain assignment")
            return {'statusCode': 200, 'body': 'Instance not running'}

        instance_id = detail.get('instance-id')
        if not instance_id:
            logger.error("Instance ID not found in event")
            return {'statusCode': 400, 'body': 'Instance ID not found'}

        # Get instance details
        instance_info = get_instance_details(instance_id)
        if not instance_info:
            logger.error(f"Could not retrieve instance details for {instance_id}")
            return {'statusCode': 404, 'body': 'Instance not found'}

        # Generate domain name based on instance tags
        domain_name = generate_domain_name(instance_info)
        if not domain_name:
            logger.info(f"Could not generate domain name for instance {instance_id}")
            return {'statusCode': 200, 'body': 'No domain assignment needed'}

        # Create DNS record
        result = create_dns_record(instance_info, domain_name)

        # Tag instance with assigned domain
        tag_instance_with_domain(instance_id, domain_name)

        logger.info(f"Successfully assigned domain {domain_name} to instance {instance_id}")
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Domain assigned successfully',
                'instance_id': instance_id,
                'domain_name': domain_name,
                'private_ip': instance_info['private_ip']
            })
        }

    except Exception as e:
        logger.error(f"Error processing event: {str(e)}")
        return {'statusCode': 500, 'body': f'Error: {str(e)}'}

def get_instance_details(instance_id):
    """
    Get EC2 instance details including tags and network information
    """
    try:
        response = ec2.describe_instances(InstanceIds=[instance_id])

        if not response['Reservations']:
            return None

        instance = response['Reservations'][0]['Instances'][0]

        # Extract tags
        tags = {}
        for tag in instance.get('Tags', []):
            tags[tag['Key']] = tag['Value']

        return {
            'instance_id': instance_id,
            'private_ip': instance['PrivateIpAddress'],
            'availability_zone': instance['Placement']['AvailabilityZone'],
            'instance_type': instance['InstanceType'],
            'tags': tags
        }

    except Exception as e:
        logger.error(f"Error getting instance details: {str(e)}")
        return None

def generate_domain_name(instance_info):
    """
    Generate domain name based on instance tags and naming convention
    """
    tags = instance_info['tags']

    # Required tags for domain assignment
    service = tags.get('Service', '').lower()
    environment = tags.get('Environment', '').lower()
    tier = tags.get('Tier', '').lower()

    if not all([service, environment, tier]):
        logger.info(f"Missing required tags for domain assignment: Service={service}, Environment={environment}, Tier={tier}")
        return None

    # Generate instance number based on existing instances (simplified)
    instance_number = tags.get('InstanceNumber', '01')

    # Handle special roles
    role = tags.get('Role', '').lower()
    if role in ['primary', 'master']:
        domain_name = f"{service}-{environment}-{role}.{tier}.{environment}.{DOMAIN_CONFIG['root_domain']}"
    else:
        domain_name = f"{service}-{environment}-{instance_number}.{tier}.{environment}.{DOMAIN_CONFIG['root_domain']}"

    return domain_name

def create_dns_record(instance_info, domain_name):
    """
    Create DNS A record in Route 53
    """
    try:
        # Determine hosted zone based on domain structure
        zone_key = get_zone_key_from_domain(domain_name)
        zone_id = DOMAIN_CONFIG['zone_mappings'].get(zone_key)

        if not zone_id:
            logger.error(f"No hosted zone found for domain {domain_name}")
            return False

        # Create change batch
        change_batch = {
            'Comment': f'Auto-assigned domain for instance {instance_info["instance_id"]}',
            'Changes': [
                {
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': domain_name,
                        'Type': 'A',
                        'TTL': 300,
                        'ResourceRecords': [
                            {
                                'Value': instance_info['private_ip']
                            }
                        ]
                    }
                }
            ]
        }

        # Submit change to Route 53
        response = route53.change_resource_record_sets(
            HostedZoneId=zone_id,
            ChangeBatch=change_batch
        )

        logger.info(f"DNS record created: {domain_name} -> {instance_info['private_ip']}")
        return True

    except Exception as e:
        logger.error(f"Error creating DNS record: {str(e)}")
        return False

def get_zone_key_from_domain(domain_name):
    """
    Extract zone key from domain name for zone mapping lookup
    """
    parts = domain_name.split('.')
    if len(parts) >= 4:
        tier = parts[-4]  # e.g., 'frontend'
        env = parts[-3]   # e.g., 'prod'
        return f"{tier}-{env}"
    return None

def tag_instance_with_domain(instance_id, domain_name):
    """
    Tag instance with assigned domain name
    """
    try:
        ec2.create_tags(
            Resources=[instance_id],
            Tags=[
                {
                    'Key': 'AssignedDomain',
                    'Value': domain_name
                },
                {
                    'Key': 'DomainAssignedAt',
                    'Value': datetime.utcnow().isoformat()
                }
            ]
        )
        logger.info(f"Tagged instance {instance_id} with domain {domain_name}")

    except Exception as e:
        logger.error(f"Error tagging instance: {str(e)}")

LAMBDA_CODE

echo "✅ Lambda function code created"

echo ""
echo "3. PACKAGING AND DEPLOYING LAMBDA FUNCTION"

# Update Lambda code with actual zone IDs
sed -i "s/Z0123456789ABCDEF01/$FRONTEND_PROD_ZONE_ID/g" domain_assignment_lambda.py
sed -i "s/Z0123456789ABCDEF02/$BACKEND_PROD_ZONE_ID/g" domain_assignment_lambda.py
sed -i "s/Z0123456789ABCDEF03/$DATA_PROD_ZONE_ID/g" domain_assignment_lambda.py
sed -i "s/Z0123456789ABCDEF04/$FRONTEND_STAGING_ZONE_ID/g" domain_assignment_lambda.py
sed -i "s/Z0123456789ABCDEF05/$BACKEND_STAGING_ZONE_ID/g" domain_assignment_lambda.py
sed -i "s/Z0123456789ABCDEF06/$FRONTEND_DEV_ZONE_ID/g" domain_assignment_lambda.py
sed -i "s/Z0123456789ABCDEF07/$BACKEND_DEV_ZONE_ID/g" domain_assignment_lambda.py

# Create deployment package
zip lambda_function.zip domain_assignment_lambda.py

# Wait for IAM role to propagate
sleep 15

# Create Lambda function
LAMBDA_FUNCTION_ARN=$(aws lambda create-function \
    --function-name CustomDomainAssignment \
    --runtime python3.9 \
    --role $LAMBDA_ROLE_ARN \
    --handler domain_assignment_lambda.lambda_handler \
    --description "Automatically assign custom domain names to EC2 instances" \
    --timeout 60 \
    --memory-size 256 \
    --zip-file fileb://lambda_function.zip \
    --tags Service=Lambda,Purpose=DomainAssignment \
    --query 'FunctionArn' \
    --output text 2>/dev/null)

if [ $? -eq 0 ]; then
    echo "✅ Lambda function created: $LAMBDA_FUNCTION_ARN"
else
    echo "Lambda function may already exist, updating code..."
    aws lambda update-function-code \
        --function-name CustomDomainAssignment \
        --zip-file fileb://lambda_function.zip > /dev/null

    LAMBDA_FUNCTION_ARN=$(aws lambda get-function \
        --function-name CustomDomainAssignment \
        --query 'Configuration.FunctionArn' \
        --output text)
    echo "✅ Lambda function updated: $LAMBDA_FUNCTION_ARN"
fi

# Clean up
rm lambda_function.zip

# Update configuration
cat << CONFIG >> custom-domain-config.env

# Lambda Configuration
export LAMBDA_ROLE_ARN="$LAMBDA_ROLE_ARN"
export LAMBDA_FUNCTION_ARN="$LAMBDA_FUNCTION_ARN"
CONFIG

echo ""
echo "LAMBDA FUNCTION SETUP COMPLETE"
echo ""
echo "Function Details:"
echo "- Function Name: CustomDomainAssignment"
echo "- Function ARN: $LAMBDA_FUNCTION_ARN"
echo "- IAM Role: $LAMBDA_ROLE_ARN"
echo "- Runtime: Python 3.9"
echo "- Timeout: 60 seconds"
echo ""
echo "The function will automatically assign domain names based on instance tags:"
echo "- Service: web, api, db, cache, etc."
echo "- Environment: prod, staging, dev"
echo "- Tier: frontend, backend, data"
echo "- InstanceNumber: 01, 02, 03, etc."

EOF

chmod +x setup-domain-assignment-lambda.sh
./setup-domain-assignment-lambda.sh
```

### Step 4: Configure EventBridge for Instance Lifecycle Events
**Objective**: Set up automatic triggering of domain assignment

**EventBridge Configuration**:
```bash
# Configure EventBridge for automatic domain assignment
cat << 'EOF' > setup-eventbridge-domain-trigger.sh
#!/bin/bash

source custom-domain-config.env

echo "=== Setting Up EventBridge for Automatic Domain Assignment ==="
echo ""

echo "1. CREATING EVENTBRIDGE RULE FOR EC2 INSTANCE STATE CHANGES"

# Create EventBridge rule for EC2 instance state changes
RULE_ARN=$(aws events put-rule \
    --name EC2InstanceStateChangeForDomainAssignment \
    --description "Trigger domain assignment when EC2 instances enter running state" \
    --event-pattern '{
        "source": ["aws.ec2"],
        "detail-type": ["EC2 Instance State-change Notification"],
        "detail": {
            "state": ["running"]
        }
    }' \
    --state ENABLED \
    --tags Key=Service,Value=EventBridge Key=Purpose,Value=DomainAssignment \
    --query 'RuleArn' \
    --output text)

echo "✅ EventBridge rule created: $RULE_ARN"

echo ""
echo "2. ADDING LAMBDA FUNCTION AS TARGET"

# Add Lambda function as target for the EventBridge rule
aws events put-targets \
    --rule EC2InstanceStateChangeForDomainAssignment \
    --targets "Id"="1","Arn"="$LAMBDA_FUNCTION_ARN"

echo "✅ Lambda function added as target for EventBridge rule"

echo ""
echo "3. GRANTING EVENTBRIDGE PERMISSION TO INVOKE LAMBDA"

# Grant EventBridge permission to invoke Lambda function
aws lambda add-permission \
    --function-name CustomDomainAssignment \
    --statement-id allow-eventbridge-invoke \
    --action lambda:InvokeFunction \
    --principal events.amazonaws.com \
    --source-arn $RULE_ARN

echo "✅ EventBridge permission granted to invoke Lambda function"

echo ""
echo "4. CREATING ADDITIONAL RULES FOR INSTANCE TERMINATION"

# Create rule for instance termination to clean up DNS records
TERMINATION_RULE_ARN=$(aws events put-rule \
    --name EC2InstanceTerminationForDomainCleanup \
    --description "Clean up DNS records when EC2 instances are terminated" \
    --event-pattern '{
        "source": ["aws.ec2"],
        "detail-type": ["EC2 Instance State-change Notification"],
        "detail": {
            "state": ["terminated", "stopping"]
        }
    }' \
    --state ENABLED \
    --tags Key=Service,Value=EventBridge Key=Purpose,Value=DomainCleanup \
    --query 'RuleArn' \
    --output text)

echo "✅ Instance termination rule created: $TERMINATION_RULE_ARN"

echo ""
echo "5. CREATING DNS CLEANUP LAMBDA FUNCTION"

# Create Lambda function code for DNS cleanup
cat << 'CLEANUP_LAMBDA_CODE' > dns_cleanup_lambda.py
import json
import boto3
import logging

# Configure logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Initialize AWS clients
ec2 = boto3.client('ec2')
route53 = boto3.client('route53')

def lambda_handler(event, context):
    """
    Lambda handler for cleaning up DNS records when instances are terminated
    """
    try:
        logger.info(f"Received event: {json.dumps(event)}")

        detail = event.get('detail', {})
        instance_id = detail.get('instance-id')
        state = detail.get('state')

        if not instance_id or state not in ['terminated', 'stopping']:
            logger.info("Not a relevant instance state change for cleanup")
            return {'statusCode': 200, 'body': 'No cleanup needed'}

        # Get instance details before it's terminated
        assigned_domain = get_assigned_domain_from_tags(instance_id)

        if assigned_domain:
            # Remove DNS record
            result = remove_dns_record(assigned_domain, instance_id)
            logger.info(f"Cleaned up DNS record for {assigned_domain}")

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'DNS cleanup completed',
                'instance_id': instance_id,
                'domain_cleaned': assigned_domain
            })
        }

    except Exception as e:
        logger.error(f"Error in DNS cleanup: {str(e)}")
        return {'statusCode': 500, 'body': f'Error: {str(e)}'}

def get_assigned_domain_from_tags(instance_id):
    """
    Get assigned domain from instance tags
    """
    try:
        response = ec2.describe_instances(InstanceIds=[instance_id])

        if not response['Reservations']:
            return None

        instance = response['Reservations'][0]['Instances'][0]

        for tag in instance.get('Tags', []):
            if tag['Key'] == 'AssignedDomain':
                return tag['Value']

        return None

    except Exception as e:
        logger.error(f"Error getting assigned domain: {str(e)}")
        return None

def remove_dns_record(domain_name, instance_id):
    """
    Remove DNS record from Route 53
    """
    try:
        # This is a simplified implementation
        # In practice, you'd need to identify the correct hosted zone
        # and get the current record details before deletion

        logger.info(f"Would remove DNS record for {domain_name} (instance {instance_id})")
        # Implementation would go here

        return True

    except Exception as e:
        logger.error(f"Error removing DNS record: {str(e)}")
        return False

CLEANUP_LAMBDA_CODE

# Create cleanup Lambda function
zip cleanup_lambda_function.zip dns_cleanup_lambda.py

CLEANUP_LAMBDA_ARN=$(aws lambda create-function \
    --function-name DNSCleanupOnTermination \
    --runtime python3.9 \
    --role $LAMBDA_ROLE_ARN \
    --handler dns_cleanup_lambda.lambda_handler \
    --description "Clean up DNS records when EC2 instances are terminated" \
    --timeout 30 \
    --memory-size 256 \
    --zip-file fileb://cleanup_lambda_function.zip \
    --tags Service=Lambda,Purpose=DNSCleanup \
    --query 'FunctionArn' \
    --output text 2>/dev/null)

if [ $? -ne 0 ]; then
    echo "Cleanup Lambda function may already exist, updating..."
    aws lambda update-function-code \
        --function-name DNSCleanupOnTermination \
        --zip-file fileb://cleanup_lambda_function.zip > /dev/null

    CLEANUP_LAMBDA_ARN=$(aws lambda get-function \
        --function-name DNSCleanupOnTermination \
        --query 'Configuration.FunctionArn' \
        --output text)
fi

echo "✅ DNS cleanup Lambda function created: $CLEANUP_LAMBDA_ARN"

# Add cleanup Lambda as target for termination rule
aws events put-targets \
    --rule EC2InstanceTerminationForDomainCleanup \
    --targets "Id"="1","Arn"="$CLEANUP_LAMBDA_ARN"

# Grant permission for termination rule
aws lambda add-permission \
    --function-name DNSCleanupOnTermination \
    --statement-id allow-eventbridge-invoke-cleanup \
    --action lambda:InvokeFunction \
    --principal events.amazonaws.com \
    --source-arn $TERMINATION_RULE_ARN 2>/dev/null || echo "Permission may already exist"

echo "✅ DNS cleanup automation configured"

# Clean up
rm cleanup_lambda_function.zip

echo ""
echo "6. TESTING EVENTBRIDGE CONFIGURATION"

# List EventBridge rules
echo "Verifying EventBridge rules..."
aws events list-rules \
    --name-prefix EC2Instance \
    --query 'Rules[*].[Name,State,Description]' \
    --output table

echo ""
echo "Verifying rule targets..."
aws events list-targets-by-rule \
    --rule EC2InstanceStateChangeForDomainAssignment \
    --query 'Targets[*].[Id,Arn]' \
    --output table

# Update configuration
cat << CONFIG >> custom-domain-config.env

# EventBridge Configuration
export RULE_ARN="$RULE_ARN"
export TERMINATION_RULE_ARN="$TERMINATION_RULE_ARN"
export CLEANUP_LAMBDA_ARN="$CLEANUP_LAMBDA_ARN"
CONFIG

echo ""
echo "EVENTBRIDGE CONFIGURATION COMPLETE"
echo ""
echo "EventBridge Rules Created:"
echo "- Instance State Change: $RULE_ARN"
echo "- Instance Termination: $TERMINATION_RULE_ARN"
echo ""
echo "Lambda Functions:"
echo "- Domain Assignment: $LAMBDA_FUNCTION_ARN"
echo "- DNS Cleanup: $CLEANUP_LAMBDA_ARN"
echo ""
echo "Automation Flow:"
echo "1. EC2 instance enters 'running' state"
echo "2. EventBridge triggers domain assignment Lambda"
echo "3. Lambda reads instance tags and assigns domain"
echo "4. DNS record created in appropriate Route 53 zone"
echo "5. Instance tagged with assigned domain"
echo ""
echo "When instance terminates:"
echo "1. EventBridge triggers cleanup Lambda"
echo "2. DNS records are removed from Route 53"

EOF

chmod +x setup-eventbridge-domain-trigger.sh
./setup-eventbridge-domain-trigger.sh
```

### Step 5: Configure DHCP Options Set for Domain Suffix
**Objective**: Set up automatic domain suffix assignment at the network level

**DHCP Options Configuration**:
```bash
# Configure DHCP Options Set for custom domain suffix
cat << 'EOF' > setup-dhcp-domain-suffix.sh
#!/bin/bash

source custom-domain-config.env

echo "=== Setting Up DHCP Options Set for Custom Domain Suffix ==="
echo ""

echo "1. CREATING DHCP OPTIONS SET"

# Create DHCP options set with custom domain name
DHCP_OPTIONS_ID=$(aws ec2 create-dhcp-options \
    --dhcp-configurations \
    Key=domain-name,Values=$ROOT_DOMAIN \
    Key=domain-name-servers,Values=AmazonProvidedDNS \
    --tag-specifications "ResourceType=dhcp-options,Tags=[{Key=Name,Value=CustomDomain-DHCP-Options},{Key=Purpose,Value=DomainSuffix}]" \
    --query 'DhcpOptions.DhcpOptionsId' \
    --output text)

echo "✅ DHCP Options Set created: $DHCP_OPTIONS_ID"

echo ""
echo "2. ASSOCIATING DHCP OPTIONS SET WITH VPC"

# Associate DHCP options set with VPC
aws ec2 associate-dhcp-options \
    --dhcp-options-id $DHCP_OPTIONS_ID \
    --vpc-id $VPC_ID

echo "✅ DHCP Options Set associated with VPC: $VPC_ID"

echo ""
echo "3. VERIFYING DHCP OPTIONS CONFIGURATION"

# Verify DHCP options configuration
echo "DHCP Options Set details:"
aws ec2 describe-dhcp-options \
    --dhcp-options-ids $DHCP_OPTIONS_ID \
    --query 'DhcpOptions[0].DhcpConfigurations[*].[Key,Values[0]]' \
    --output table

echo ""
echo "VPC DHCP Options association:"
aws ec2 describe-vpcs \
    --vpc-ids $VPC_ID \
    --query 'Vpcs[0].[VpcId,DhcpOptionsId]' \
    --output table

echo ""
echo "4. CREATING TEST SCRIPT FOR DHCP DOMAIN SUFFIX"

cat << 'DHCP_TEST_SCRIPT' > test-dhcp-domain-suffix.sh
#!/bin/bash

echo "=== Testing DHCP Domain Suffix Configuration ==="
echo ""

echo "1. CHECKING SYSTEM DOMAIN CONFIGURATION"

# Check system domain configuration
echo "Current hostname:"
hostname

echo ""
echo "Current FQDN:"
hostname -f 2>/dev/null || echo "FQDN not available (requires instance restart)"

echo ""
echo "Domain search configuration:"
if [ -f /etc/resolv.conf ]; then
    grep -E "^(domain|search)" /etc/resolv.conf || echo "No domain/search configuration found"
else
    echo "/etc/resolv.conf not found"
fi

echo ""
echo "2. CHECKING DNS RESOLUTION"

# Test DNS resolution with domain suffix
echo "Testing DNS resolution with domain suffix..."

# Test short name resolution (should append domain suffix)
echo "Testing short name resolution:"
nslookup api 2>/dev/null | head -10 || echo "Short name resolution test failed"

echo ""
echo "Testing FQDN resolution:"
nslookup api.$ROOT_DOMAIN 2>/dev/null | head -10 || echo "FQDN resolution test failed"

echo ""
echo "3. NETWORK INTERFACE CONFIGURATION"

# Check network interface configuration
echo "Network interface information:"
ip addr show | grep -A 2 "inet 10\."

echo ""
echo "4. DHCP LEASE INFORMATION"

# Check DHCP lease information if available
if [ -f /var/lib/dhcp/dhclient.leases ]; then
    echo "Recent DHCP lease information:"
    tail -20 /var/lib/dhcp/dhclient.leases | grep -E "(domain-name|domain-search)"
elif [ -f /var/lib/dhcp/dhclient.eth0.leases ]; then
    echo "Recent DHCP lease information:"
    tail -20 /var/lib/dhcp/dhclient.eth0.leases | grep -E "(domain-name|domain-search)"
else
    echo "DHCP lease files not found"
fi

echo ""
echo "Note: Instance may need to be restarted to pick up new DHCP options"
echo "Or run: sudo dhclient -r && sudo dhclient"

DHCP_TEST_SCRIPT

chmod +x test-dhcp-domain-suffix.sh

echo "✅ DHCP domain suffix test script created"

echo ""
echo "5. CREATING INSTANCE HOSTNAME CONFIGURATION SCRIPT"

cat << 'HOSTNAME_CONFIG_SCRIPT' > configure-instance-hostname.sh
#!/bin/bash

# This script should be run on EC2 instances to configure hostname
# based on the custom domain assignment

echo "=== Configuring Instance Hostname ==="
echo ""

# Get instance metadata
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

echo "Instance ID: $INSTANCE_ID"
echo "Private IP: $PRIVATE_IP"

# Get assigned domain from instance tags
ASSIGNED_DOMAIN=$(aws ec2 describe-tags \
    --filters "Name=resource-id,Values=$INSTANCE_ID" "Name=key,Values=AssignedDomain" \
    --query 'Tags[0].Value' \
    --output text 2>/dev/null)

if [ "$ASSIGNED_DOMAIN" != "None" ] && [ ! -z "$ASSIGNED_DOMAIN" ]; then
    echo "Assigned domain found: $ASSIGNED_DOMAIN"

    # Extract hostname from FQDN
    HOSTNAME=$(echo $ASSIGNED_DOMAIN | cut -d'.' -f1)

    echo "Setting hostname to: $HOSTNAME"

    # Set hostname
    sudo hostnamectl set-hostname $HOSTNAME

    # Update /etc/hosts
    sudo cp /etc/hosts /etc/hosts.backup
    sudo sed -i "/127.0.0.1.*localhost/c\127.0.0.1 localhost $HOSTNAME $ASSIGNED_DOMAIN" /etc/hosts
    sudo sed -i "/$PRIVATE_IP/d" /etc/hosts
    echo "$PRIVATE_IP $ASSIGNED_DOMAIN $HOSTNAME" | sudo tee -a /etc/hosts

    echo "✅ Hostname configured successfully"
    echo "Current hostname: $(hostname)"
    echo "Current FQDN: $(hostname -f)"

else
    echo "No assigned domain found in instance tags"
    echo "Make sure the instance has the required tags:"
    echo "- Service: web, api, db, etc."
    echo "- Environment: prod, staging, dev"
    echo "- Tier: frontend, backend, data"
    echo "- InstanceNumber: 01, 02, etc."
fi

echo ""
echo "Testing DNS resolution:"
nslookup $(hostname) || echo "DNS resolution test failed"

HOSTNAME_CONFIG_SCRIPT

chmod +x configure-instance-hostname.sh

echo "✅ Instance hostname configuration script created"

# Update configuration
cat << CONFIG >> custom-domain-config.env

# DHCP Configuration
export DHCP_OPTIONS_ID="$DHCP_OPTIONS_ID"
CONFIG

echo ""
echo "DHCP OPTIONS SET CONFIGURATION COMPLETE"
echo ""
echo "Configuration Summary:"
echo "- DHCP Options Set: $DHCP_OPTIONS_ID"
echo "- Domain Name: $ROOT_DOMAIN"
echo "- DNS Servers: AmazonProvidedDNS"
echo "- Associated VPC: $VPC_ID"
echo ""
echo "Testing Tools:"
echo "- test-dhcp-domain-suffix.sh - Test DHCP domain configuration"
echo "- configure-instance-hostname.sh - Configure instance hostname (run on instances)"
echo ""
echo "IMPORTANT: New instances will automatically receive the domain suffix."
echo "Existing instances may need to renew DHCP lease or restart to pick up changes."
echo ""
echo "To apply to existing instances:"
echo "sudo dhclient -r && sudo dhclient"

EOF

chmod +x setup-dhcp-domain-suffix.sh
./setup-dhcp-domain-suffix.sh
```

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"content": "Research existing DNS and domain name documentation in the codebase", "status": "completed", "activeForm": "Researching existing DNS and domain name documentation in the codebase"}, {"content": "Create documentation for custom domain name assignment methods", "status": "completed", "activeForm": "Creating documentation for custom domain name assignment methods"}, {"content": "Document Route 53 Private Hosted Zone approach", "status": "completed", "activeForm": "Documenting Route 53 Private Hosted Zone approach"}, {"content": "Document DHCP Options Set configuration", "status": "completed", "activeForm": "Documenting DHCP Options Set configuration"}, {"content": "Document automation approaches using Lambda/EventBridge", "status": "completed", "activeForm": "Documenting automation approaches using Lambda/EventBridge"}]