# Topic 60: Multi-Region Failover Strategies - RTO/RPO and Implementation

## Table of Contents
1. [Failover Strategy Overview](#overview)
2. [Route 53 Health Checks](#health-checks)
3. [Failover Routing Policies](#routing-policies)
4. [Active-Active vs Active-Passive Patterns](#patterns)
5. [RTO/RPO Considerations](#rto-rpo)
6. [Automated Failover Implementation](#automation)
7. [Database Failover Strategies](#database)
8. [Application-Level Failover](#application)
9. [Monitoring and Alerting](#monitoring)
10. [Testing and Validation](#testing)
11. [Cost Optimization](#cost-optimization)
12. [Best Practices](#best-practices)

## Failover Strategy Overview {#overview}

### Multi-Region Failover Fundamentals

Multi-region failover strategies ensure business continuity by automatically redirecting traffic from a failed primary region to healthy secondary regions. These strategies are critical for achieving high availability and meeting disaster recovery requirements.

**Key Components of Failover Architecture**:

```
Multi-Region Failover Architecture:

                    ┌─────────────────────────────────┐
                    │        Route 53 (Global)        │
                    │     Health Check Engine         │
                    │    Failover Routing Policy      │
                    └─────────────────┬───────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
                    ▼                 ▼                 ▼
        ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
        │ Primary Region  │ │Secondary Region │ │ Tertiary Region │
        │   (us-east-1)   │ │   (us-west-2)   │ │   (eu-west-1)   │
        │                 │ │                 │ │                 │
        │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
        │ │   Active    │ │ │ │   Standby   │ │ │ │   Standby   │ │
        │ │ Application │ │ │ │ Application │ │ │ │ Application │ │
        │ │    Stack    │ │ │ │    Stack    │ │ │ │    Stack    │ │
        │ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │
        │                 │ │                 │ │                 │
        │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
        │ │  Database   │ │ │ │  Database   │ │ │ │  Database   │ │
        │ │   Primary   │◄┼─┼─┤│Read Replica │ │ │ │Read Replica │ │
        │ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │
        └─────────────────┘ └─────────────────┘ └─────────────────┘
               │                     │                     │
               ▼                     ▼                     ▼
        Health: Healthy      Health: Healthy      Health: Healthy
        Traffic: 100%        Traffic: 0%          Traffic: 0%
        
        Failure Scenario:
        Primary Failed ──→ Secondary Activated ──→ Traffic: 100%
```

### Failover Strategy Types

**1. DNS-Based Failover**
```yaml
DNS_Failover:
  Mechanism: Route 53 health checks and routing policies
  Advantages:
    - Simple to implement
    - No application changes required
    - Global load distribution
    - Cost-effective
  Disadvantages:
    - DNS caching can delay failover
    - Limited granular control
    - Potential for split-brain scenarios
  TTL_Considerations:
    - Low TTL (60s) for faster failover
    - Trade-off with increased DNS queries
    - Client-side caching behavior
```

**2. Application-Level Failover**
```yaml
Application_Failover:
  Mechanism: Application logic detects failures and redirects
  Advantages:
    - Immediate failover response
    - Fine-grained control
    - Custom failure detection logic
    - No DNS caching delays
  Disadvantages:
    - Application complexity increases
    - Requires code changes
    - Potential for inconsistent behavior
    - Higher development effort
```

**3. Infrastructure-Level Failover**
```yaml
Infrastructure_Failover:
  Mechanism: Load balancers and proxies handle failover
  Advantages:
    - Transparent to applications
    - Fast failover times
    - Health check integration
    - Session persistence options
  Disadvantages:
    - Infrastructure complexity
    - Single points of failure
    - Limited to supported protocols
    - Geographic constraints
```

## Route 53 Health Checks {#health-checks}

### Health Check Configuration

Route 53 health checks are fundamental to implementing robust failover strategies. They continuously monitor endpoint health and trigger failover when failures are detected.

```python
import boto3
import json
from typing import Dict, List, Optional

class Route53HealthCheckManager:
    def __init__(self):
        self.route53 = boto3.client('route53')
        
    def create_health_check(self, 
                          endpoint_config: Dict,
                          health_check_config: Dict) -> str:
        """Create a comprehensive health check"""
        
        health_check_params = {
            'Type': health_check_config.get('type', 'HTTPS'),
            'ResourcePath': health_check_config.get('path', '/health'),
            'FullyQualifiedDomainName': endpoint_config['fqdn'],
            'Port': health_check_config.get('port', 443),
            'RequestInterval': health_check_config.get('interval', 30),
            'FailureThreshold': health_check_config.get('failure_threshold', 3),
            'MeasureLatency': health_check_config.get('measure_latency', True),
            'Inverted': health_check_config.get('inverted', False),
            'Disabled': health_check_config.get('disabled', False),
            'EnableSNI': health_check_config.get('enable_sni', True)
        }
        
        # Add search string for content verification
        if 'search_string' in health_check_config:
            health_check_params['SearchString'] = health_check_config['search_string']
        
        # Add regions for latency measurement
        if 'regions' in health_check_config:
            health_check_params['Regions'] = health_check_config['regions']
        
        try:
            response = self.route53.create_health_check(
                CallerReference=str(hash(f"{endpoint_config['fqdn']}-{health_check_config.get('name', 'default')}")),
                HealthCheckConfig=health_check_params,
                Tags=[
                    {'Key': 'Name', 'Value': health_check_config.get('name', 'Unnamed Health Check')},
                    {'Key': 'Environment', 'Value': endpoint_config.get('environment', 'production')},
                    {'Key': 'Region', 'Value': endpoint_config.get('region', 'unknown')},
                    {'Key': 'Purpose', 'Value': 'Failover'}
                ]
            )
            
            health_check_id = response['HealthCheck']['Id']
            print(f"Created health check {health_check_id} for {endpoint_config['fqdn']}")
            
            return health_check_id
            
        except Exception as e:
            print(f"Failed to create health check for {endpoint_config['fqdn']}: {e}")
            return None
    
    def create_calculated_health_check(self, 
                                     child_health_checks: List[str],
                                     calculation_config: Dict) -> str:
        """Create calculated health check from multiple child checks"""
        
        health_check_params = {
            'Type': 'CALCULATED',
            'ChildHealthChecks': child_health_checks,
            'CloudWatchAlarmRegion': calculation_config.get('cloudwatch_region', 'us-east-1'),
            'InsufficientDataHealthStatus': calculation_config.get('insufficient_data_status', 'Failure'),
            'Inverted': calculation_config.get('inverted', False),
            'Disabled': calculation_config.get('disabled', False)
        }
        
        # Set health threshold
        if 'health_threshold' in calculation_config:
            health_check_params['HealthThreshold'] = calculation_config['health_threshold']
        
        try:
            response = self.route53.create_health_check(
                CallerReference=str(hash(f"calculated-{'-'.join(child_health_checks)}")),
                HealthCheckConfig=health_check_params,
                Tags=[
                    {'Key': 'Name', 'Value': calculation_config.get('name', 'Calculated Health Check')},
                    {'Key': 'Type', 'Value': 'Calculated'},
                    {'Key': 'Purpose', 'Value': 'Aggregate Health Monitoring'}
                ]
            )
            
            health_check_id = response['HealthCheck']['Id']
            print(f"Created calculated health check {health_check_id}")
            
            return health_check_id
            
        except Exception as e:
            print(f"Failed to create calculated health check: {e}")
            return None
    
    def create_cloudwatch_alarm_health_check(self, 
                                           alarm_config: Dict) -> str:
        """Create health check based on CloudWatch alarm"""
        
        health_check_params = {
            'Type': 'CLOUDWATCH_METRIC',
            'AlarmIdentifier': {
                'Region': alarm_config['region'],
                'Name': alarm_config['alarm_name']
            },
            'InsufficientDataHealthStatus': alarm_config.get('insufficient_data_status', 'Failure'),
            'Inverted': alarm_config.get('inverted', False),
            'Disabled': alarm_config.get('disabled', False)
        }
        
        try:
            response = self.route53.create_health_check(
                CallerReference=str(hash(f"cloudwatch-{alarm_config['alarm_name']}")),
                HealthCheckConfig=health_check_params,
                Tags=[
                    {'Key': 'Name', 'Value': alarm_config.get('name', 'CloudWatch Health Check')},
                    {'Key': 'Type', 'Value': 'CloudWatch'},
                    {'Key': 'AlarmName', 'Value': alarm_config['alarm_name']},
                    {'Key': 'Region', 'Value': alarm_config['region']}
                ]
            )
            
            health_check_id = response['HealthCheck']['Id']
            print(f"Created CloudWatch health check {health_check_id}")
            
            return health_check_id
            
        except Exception as e:
            print(f"Failed to create CloudWatch health check: {e}")
            return None
    
    def get_health_check_status(self, health_check_id: str) -> Dict:
        """Get current health check status and metrics"""
        
        try:
            # Get health check details
            response = self.route53.get_health_check(Id=health_check_id)
            health_check = response['HealthCheck']
            
            # Get health check status
            status_response = self.route53.get_health_check_status(Id=health_check_id)
            status_checkers = status_response['StatusCheckers']
            
            # Calculate overall status
            healthy_checkers = sum(1 for checker in status_checkers if checker['Status'] == 'Success')
            total_checkers = len(status_checkers)
            health_percentage = (healthy_checkers / total_checkers * 100) if total_checkers > 0 else 0
            
            return {
                'health_check_id': health_check_id,
                'type': health_check['HealthCheckConfig']['Type'],
                'endpoint': health_check['HealthCheckConfig'].get('FullyQualifiedDomainName', 'N/A'),
                'healthy_checkers': healthy_checkers,
                'total_checkers': total_checkers,
                'health_percentage': health_percentage,
                'overall_status': 'Healthy' if health_percentage >= 66 else 'Unhealthy',
                'status_details': status_checkers,
                'health_check_version': health_check['HealthCheckVersion']
            }
            
        except Exception as e:
            return {
                'health_check_id': health_check_id,
                'error': str(e),
                'overall_status': 'Error'
            }
    
    def setup_comprehensive_health_monitoring(self, 
                                            application_config: Dict) -> Dict:
        """Set up comprehensive health monitoring for multi-region application"""
        
        health_check_ids = {}
        
        # Create individual endpoint health checks
        for region, endpoint_info in application_config['endpoints'].items():
            # Application health check
            app_health_config = {
                'name': f"{application_config['name']}-app-{region}",
                'type': 'HTTPS',
                'path': '/health',
                'port': 443,
                'interval': 30,
                'failure_threshold': 3,
                'search_string': 'OK',
                'measure_latency': True,
                'regions': ['us-east-1', 'eu-west-1', 'ap-southeast-1']
            }
            
            app_health_id = self.create_health_check(
                endpoint_info, 
                app_health_config
            )
            
            if app_health_id:
                health_check_ids[f"{region}_app"] = app_health_id
            
            # Database health check
            if 'database_endpoint' in endpoint_info:
                db_health_config = {
                    'name': f"{application_config['name']}-db-{region}",
                    'type': 'TCP',
                    'port': 3306,
                    'interval': 30,
                    'failure_threshold': 2
                }
                
                db_endpoint_config = {
                    'fqdn': endpoint_info['database_endpoint'],
                    'environment': endpoint_info.get('environment', 'production'),
                    'region': region
                }
                
                db_health_id = self.create_health_check(
                    db_endpoint_config,
                    db_health_config
                )
                
                if db_health_id:
                    health_check_ids[f"{region}_db"] = db_health_id
        
        # Create calculated health checks for each region
        for region in application_config['endpoints'].keys():
            region_checks = [
                health_check_ids.get(f"{region}_app"),
                health_check_ids.get(f"{region}_db")
            ]
            region_checks = [hc for hc in region_checks if hc is not None]
            
            if len(region_checks) >= 2:
                calculated_config = {
                    'name': f"{application_config['name']}-{region}-overall",
                    'health_threshold': len(region_checks),  # All checks must pass
                    'insufficient_data_status': 'Failure'
                }
                
                calculated_health_id = self.create_calculated_health_check(
                    region_checks,
                    calculated_config
                )
                
                if calculated_health_id:
                    health_check_ids[f"{region}_overall"] = calculated_health_id
        
        return health_check_ids

# Usage example
health_manager = Route53HealthCheckManager()

# Application configuration
app_config = {
    'name': 'ecommerce-platform',
    'endpoints': {
        'us-east-1': {
            'fqdn': 'api-us-east-1.example.com',
            'database_endpoint': 'db-us-east-1.example.com',
            'environment': 'production',
            'region': 'us-east-1'
        },
        'eu-west-1': {
            'fqdn': 'api-eu-west-1.example.com',
            'database_endpoint': 'db-eu-west-1.example.com',
            'environment': 'production',
            'region': 'eu-west-1'
        },
        'ap-south-1': {
            'fqdn': 'api-ap-south-1.example.com',
            'database_endpoint': 'db-ap-south-1.example.com',
            'environment': 'production',
            'region': 'ap-south-1'
        }
    }
}

# Set up comprehensive health monitoring
health_check_ids = health_manager.setup_comprehensive_health_monitoring(app_config)
print("Created health checks:", json.dumps(health_check_ids, indent=2))

# Monitor health status
for name, health_check_id in health_check_ids.items():
    if health_check_id:
        status = health_manager.get_health_check_status(health_check_id)
        print(f"{name}: {status['overall_status']} ({status.get('health_percentage', 0):.1f}%)")
```

### Advanced Health Check Patterns

```bash
#!/bin/bash
# Advanced health check setup script

# Function to create deep health check
create_deep_health_check() {
    local name=$1
    local fqdn=$2
    local path=$3
    local search_string=$4
    
    echo "Creating deep health check for $fqdn"
    
    HEALTH_CHECK_ID=$(aws route53 create-health-check \
        --caller-reference "$(date +%s)-$name" \
        --health-check-config '{
            "Type": "HTTPS_STR_MATCH",
            "ResourcePath": "'$path'",
            "FullyQualifiedDomainName": "'$fqdn'",
            "Port": 443,
            "RequestInterval": 30,
            "FailureThreshold": 3,
            "SearchString": "'$search_string'",
            "MeasureLatency": true,
            "EnableSNI": true
        }' \
        --query 'HealthCheck.Id' \
        --output text)
    
    # Add tags
    aws route53 change-tags-for-resource \
        --resource-type healthcheck \
        --resource-id $HEALTH_CHECK_ID \
        --add-tags Key=Name,Value="$name" Key=Type,Value=DeepHealthCheck
    
    echo "Created health check: $HEALTH_CHECK_ID"
    echo $HEALTH_CHECK_ID
}

# Function to create multi-region calculated health check
create_calculated_health_check() {
    local name=$1
    local child_checks=($2)
    local health_threshold=$3
    
    echo "Creating calculated health check: $name"
    
    # Build child health checks array
    CHILD_CHECKS_JSON=$(printf '"%s",' "${child_checks[@]}")
    CHILD_CHECKS_JSON="[${CHILD_CHECKS_JSON%,}]"
    
    CALC_HEALTH_CHECK_ID=$(aws route53 create-health-check \
        --caller-reference "$(date +%s)-calc-$name" \
        --health-check-config '{
            "Type": "CALCULATED",
            "ChildHealthChecks": '$CHILD_CHECKS_JSON',
            "HealthThreshold": '$health_threshold',
            "InsufficientDataHealthStatus": "Failure"
        }' \
        --query 'HealthCheck.Id' \
        --output text)
    
    # Add tags
    aws route53 change-tags-for-resource \
        --resource-type healthcheck \
        --resource-id $CALC_HEALTH_CHECK_ID \
        --add-tags Key=Name,Value="$name" Key=Type,Value=CalculatedHealthCheck
    
    echo "Created calculated health check: $CALC_HEALTH_CHECK_ID"
    echo $CALC_HEALTH_CHECK_ID
}

# Create health checks for multi-region setup
echo "Setting up multi-region health checks..."

# Primary region health checks
PRIMARY_APP_HC=$(create_deep_health_check "primary-app" "api.example.com" "/health" "healthy")
PRIMARY_DB_HC=$(create_deep_health_check "primary-db" "api.example.com" "/health/db" "database_ok")

# Secondary region health checks  
SECONDARY_APP_HC=$(create_deep_health_check "secondary-app" "api-backup.example.com" "/health" "healthy")
SECONDARY_DB_HC=$(create_deep_health_check "secondary-db" "api-backup.example.com" "/health/db" "database_ok")

# Calculated health checks for each region
PRIMARY_CALC_HC=$(create_calculated_health_check "primary-overall" "$PRIMARY_APP_HC $PRIMARY_DB_HC" 2)
SECONDARY_CALC_HC=$(create_calculated_health_check "secondary-overall" "$SECONDARY_APP_HC $SECONDARY_DB_HC" 2)

echo "Health check setup completed:"
echo "Primary calculated: $PRIMARY_CALC_HC"
echo "Secondary calculated: $SECONDARY_CALC_HC"

# Store health check IDs for use in DNS records
cat > health_check_ids.env << EOF
PRIMARY_OVERALL_HC=$PRIMARY_CALC_HC
SECONDARY_OVERALL_HC=$SECONDARY_CALC_HC
PRIMARY_APP_HC=$PRIMARY_APP_HC
PRIMARY_DB_HC=$PRIMARY_DB_HC
SECONDARY_APP_HC=$SECONDARY_APP_HC
SECONDARY_DB_HC=$SECONDARY_DB_HC
EOF

echo "Health check IDs saved to health_check_ids.env"
```

## Failover Routing Policies {#routing-policies}

### DNS Failover Configuration

```python
import boto3
import json
from typing import Dict, List, Optional

class Route53FailoverManager:
    def __init__(self):
        self.route53 = boto3.client('route53')
        
    def create_failover_record_set(self,
                                 hosted_zone_id: str,
                                 record_config: Dict) -> bool:
        """Create failover record set with comprehensive configuration"""
        
        change_batch = {
            'Comment': f"Failover configuration for {record_config['name']}",
            'Changes': []
        }
        
        # Primary record
        primary_change = {
            'Action': 'UPSERT',
            'ResourceRecordSet': {
                'Name': record_config['name'],
                'Type': record_config['type'],
                'SetIdentifier': f"{record_config['name']}-primary",
                'Failover': 'PRIMARY',
                'TTL': record_config.get('ttl', 60),
                'ResourceRecords': [{'Value': record_config['primary_value']}]
            }
        }
        
        # Add health check if provided
        if 'primary_health_check_id' in record_config:
            primary_change['ResourceRecordSet']['HealthCheckId'] = record_config['primary_health_check_id']
        
        change_batch['Changes'].append(primary_change)
        
        # Secondary record
        secondary_change = {
            'Action': 'UPSERT',
            'ResourceRecordSet': {
                'Name': record_config['name'],
                'Type': record_config['type'],
                'SetIdentifier': f"{record_config['name']}-secondary",
                'Failover': 'SECONDARY',
                'TTL': record_config.get('ttl', 60),
                'ResourceRecords': [{'Value': record_config['secondary_value']}]
            }
        }
        
        # Add health check for secondary if provided
        if 'secondary_health_check_id' in record_config:
            secondary_change['ResourceRecordSet']['HealthCheckId'] = record_config['secondary_health_check_id']
        
        change_batch['Changes'].append(secondary_change)
        
        try:
            response = self.route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch=change_batch
            )
            
            change_id = response['ChangeInfo']['Id']
            print(f"Created failover records for {record_config['name']}, Change ID: {change_id}")
            
            # Wait for change to propagate
            waiter = self.route53.get_waiter('resource_record_sets_changed')
            waiter.wait(Id=change_id)
            
            return True
            
        except Exception as e:
            print(f"Failed to create failover records: {e}")
            return False
    
    def create_weighted_failover_setup(self,
                                     hosted_zone_id: str,
                                     weighted_config: Dict) -> bool:
        """Create weighted routing with failover for active-active setup"""
        
        change_batch = {
            'Comment': f"Weighted failover for {weighted_config['name']}",
            'Changes': []
        }
        
        for region_config in weighted_config['regions']:
            # Primary weighted record
            primary_change = {
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': weighted_config['name'],
                    'Type': weighted_config['type'],
                    'SetIdentifier': f"{region_config['region']}-primary",
                    'Weight': region_config['weight'],
                    'TTL': weighted_config.get('ttl', 60),
                    'ResourceRecords': [{'Value': region_config['primary_value']}]
                }
            }
            
            if 'health_check_id' in region_config:
                primary_change['ResourceRecordSet']['HealthCheckId'] = region_config['health_check_id']
            
            change_batch['Changes'].append(primary_change)
            
            # Backup record for this region (if provided)
            if 'backup_value' in region_config:
                backup_change = {
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': weighted_config['name'],
                        'Type': weighted_config['type'],
                        'SetIdentifier': f"{region_config['region']}-backup",
                        'Weight': 0,  # Initially 0 weight
                        'TTL': weighted_config.get('ttl', 60),
                        'ResourceRecords': [{'Value': region_config['backup_value']}]
                    }
                }
                
                if 'backup_health_check_id' in region_config:
                    backup_change['ResourceRecordSet']['HealthCheckId'] = region_config['backup_health_check_id']
                
                change_batch['Changes'].append(backup_change)
        
        try:
            response = self.route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch=change_batch
            )
            
            change_id = response['ChangeInfo']['Id']
            print(f"Created weighted failover setup for {weighted_config['name']}")
            
            return True
            
        except Exception as e:
            print(f"Failed to create weighted failover setup: {e}")
            return False
    
    def create_latency_based_failover(self,
                                    hosted_zone_id: str,
                                    latency_config: Dict) -> bool:
        """Create latency-based routing with failover capabilities"""
        
        change_batch = {
            'Comment': f"Latency-based failover for {latency_config['name']}",
            'Changes': []
        }
        
        for region_config in latency_config['regions']:
            # Primary latency record
            primary_change = {
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': latency_config['name'],
                    'Type': latency_config['type'],
                    'SetIdentifier': f"{region_config['region']}-latency",
                    'Region': region_config['region'],
                    'TTL': latency_config.get('ttl', 60),
                    'ResourceRecords': [{'Value': region_config['value']}]
                }
            }
            
            if 'health_check_id' in region_config:
                primary_change['ResourceRecordSet']['HealthCheckId'] = region_config['health_check_id']
            
            change_batch['Changes'].append(primary_change)
        
        try:
            response = self.route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch=change_batch
            )
            
            change_id = response['ChangeInfo']['Id']
            print(f"Created latency-based failover for {latency_config['name']}")
            
            return True
            
        except Exception as e:
            print(f"Failed to create latency-based failover: {e}")
            return False
    
    def setup_comprehensive_failover_strategy(self,
                                            hosted_zone_id: str,
                                            strategy_config: Dict) -> Dict:
        """Set up comprehensive multi-tier failover strategy"""
        
        results = {}
        
        # 1. Create primary failover (simple failover)
        if 'simple_failover' in strategy_config:
            simple_result = self.create_failover_record_set(
                hosted_zone_id,
                strategy_config['simple_failover']
            )
            results['simple_failover'] = simple_result
        
        # 2. Create weighted failover for load distribution
        if 'weighted_failover' in strategy_config:
            weighted_result = self.create_weighted_failover_setup(
                hosted_zone_id,
                strategy_config['weighted_failover']
            )
            results['weighted_failover'] = weighted_result
        
        # 3. Create latency-based failover for performance optimization
        if 'latency_failover' in strategy_config:
            latency_result = self.create_latency_based_failover(
                hosted_zone_id,
                strategy_config['latency_failover']
            )
            results['latency_failover'] = latency_result
        
        # 4. Create alias records for CloudFront/ALB integration
        if 'alias_records' in strategy_config:
            alias_results = self.create_alias_failover_records(
                hosted_zone_id,
                strategy_config['alias_records']
            )
            results['alias_records'] = alias_results
        
        return results
    
    def create_alias_failover_records(self,
                                    hosted_zone_id: str,
                                    alias_config: Dict) -> bool:
        """Create alias records with failover for ALB/CloudFront"""
        
        change_batch = {
            'Comment': f"Alias failover for {alias_config['name']}",
            'Changes': []
        }
        
        # Primary alias record
        primary_change = {
            'Action': 'UPSERT',
            'ResourceRecordSet': {
                'Name': alias_config['name'],
                'Type': 'A',
                'SetIdentifier': f"{alias_config['name']}-primary-alias",
                'Failover': 'PRIMARY',
                'AliasTarget': {
                    'DNSName': alias_config['primary_alias_target'],
                    'EvaluateTargetHealth': True,
                    'HostedZoneId': alias_config['primary_hosted_zone_id']
                }
            }
        }
        
        if 'primary_health_check_id' in alias_config:
            primary_change['ResourceRecordSet']['HealthCheckId'] = alias_config['primary_health_check_id']
        
        change_batch['Changes'].append(primary_change)
        
        # Secondary alias record
        secondary_change = {
            'Action': 'UPSERT',
            'ResourceRecordSet': {
                'Name': alias_config['name'],
                'Type': 'A',
                'SetIdentifier': f"{alias_config['name']}-secondary-alias",
                'Failover': 'SECONDARY',
                'AliasTarget': {
                    'DNSName': alias_config['secondary_alias_target'],
                    'EvaluateTargetHealth': True,
                    'HostedZoneId': alias_config['secondary_hosted_zone_id']
                }
            }
        }
        
        if 'secondary_health_check_id' in alias_config:
            secondary_change['ResourceRecordSet']['HealthCheckId'] = alias_config['secondary_health_check_id']
        
        change_batch['Changes'].append(secondary_change)
        
        try:
            response = self.route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch=change_batch
            )
            
            print(f"Created alias failover records for {alias_config['name']}")
            return True
            
        except Exception as e:
            print(f"Failed to create alias failover records: {e}")
            return False

# Usage example
failover_manager = Route53FailoverManager()

# Comprehensive failover strategy configuration
strategy_config = {
    'simple_failover': {
        'name': 'api.example.com',
        'type': 'A',
        'primary_value': '1.2.3.4',
        'secondary_value': '5.6.7.8',
        'primary_health_check_id': 'primary-health-check-id',
        'secondary_health_check_id': 'secondary-health-check-id',
        'ttl': 60
    },
    'weighted_failover': {
        'name': 'app.example.com',
        'type': 'A',
        'ttl': 60,
        'regions': [
            {
                'region': 'us-east-1',
                'weight': 70,
                'primary_value': '10.0.1.100',
                'backup_value': '10.0.1.200',
                'health_check_id': 'us-east-1-health-check'
            },
            {
                'region': 'eu-west-1',
                'weight': 30,
                'primary_value': '10.1.1.100',
                'backup_value': '10.1.1.200',
                'health_check_id': 'eu-west-1-health-check'
            }
        ]
    },
    'latency_failover': {
        'name': 'cdn.example.com',
        'type': 'A',
        'ttl': 300,
        'regions': [
            {
                'region': 'us-east-1',
                'value': '52.1.1.1',
                'health_check_id': 'us-east-1-cdn-health'
            },
            {
                'region': 'eu-west-1',
                'value': '52.2.2.2',
                'health_check_id': 'eu-west-1-cdn-health'
            },
            {
                'region': 'ap-south-1',
                'value': '52.3.3.3',
                'health_check_id': 'ap-south-1-cdn-health'
            }
        ]
    },
    'alias_records': {
        'name': 'www.example.com',
        'primary_alias_target': 'primary-alb-123456.us-east-1.elb.amazonaws.com',
        'primary_hosted_zone_id': 'Z35SXDOTRQ7X7K',  # us-east-1 ALB zone
        'secondary_alias_target': 'secondary-alb-789012.us-west-2.elb.amazonaws.com',
        'secondary_hosted_zone_id': 'Z1D633PJN98FT9',  # us-west-2 ALB zone
        'primary_health_check_id': 'primary-alb-health',
        'secondary_health_check_id': 'secondary-alb-health'
    }
}

# Set up the comprehensive failover strategy
hosted_zone_id = 'Z123456789ABCDEF'
results = failover_manager.setup_comprehensive_failover_strategy(
    hosted_zone_id,
    strategy_config
)

print("Failover strategy setup results:")
print(json.dumps(results, indent=2))
```

## Active-Active vs Active-Passive Patterns {#patterns}

### Active-Active Implementation

```yaml
# Active-Active Multi-Region Configuration
ActiveActivePattern:
  Description: "All regions actively serve traffic with automatic load distribution"
  
  Architecture:
    Traffic_Distribution:
      Method: "Weighted routing with health checks"
      Primary_Region:
        Region: us-east-1
        Weight: 60%
        Capacity: "Full production capacity"
        Services: "Complete application stack"
        
      Secondary_Region:
        Region: eu-west-1
        Weight: 40%
        Capacity: "Full production capacity"
        Services: "Complete application stack"
        
    Data_Synchronization:
      Database_Strategy: "Multi-master replication"
      Conflict_Resolution: "Last writer wins with timestamps"
      Consistency_Model: "Eventual consistency"
      Replication_Lag: "< 5 seconds"
      
    Session_Management:
      Strategy: "Stateless applications with external session store"
      Session_Store: "ElastiCache Global Datastore"
      Sticky_Sessions: false
      Session_Replication: "Real-time across regions"
      
  Benefits:
    - Maximum resource utilization
    - Optimal user experience (lowest latency)
    - Horizontal scaling across regions
    - No wasted standby resources
    - Load distribution reduces single region pressure
    
  Challenges:
    - Complex data synchronization
    - Potential consistency issues
    - Higher operational complexity
    - Conflict resolution requirements
    - Cross-region transaction challenges
    
  Use_Cases:
    - Global web applications
    - Content delivery platforms
    - Read-heavy workloads
    - Geographically distributed user base
    - High availability requirements
```

### Active-Passive Implementation

```yaml
# Active-Passive Multi-Region Configuration
ActivePassivePattern:
  Description: "Primary region serves all traffic, secondary regions in standby"
  
  Architecture:
    Traffic_Distribution:
      Method: "Failover routing with health checks"
      Primary_Region:
        Region: us-east-1
        Status: Active
        Traffic: 100%
        Capacity: "Full production capacity"
        
      Secondary_Region:
        Region: us-west-2
        Status: Warm standby
        Traffic: 0%
        Capacity: "Reduced capacity, auto-scaling ready"
        
      Tertiary_Region:
        Region: eu-west-1
        Status: Cold standby
        Traffic: 0%
        Capacity: "Minimal resources, rapid deployment capability"
        
    Data_Synchronization:
      Database_Strategy: "Master-slave replication"
      Primary_Database: "Read-write master in primary region"
      Secondary_Databases: "Read-only replicas in standby regions"
      Replication_Method: "Asynchronous with monitoring"
      
    Failover_Process:
      Detection_Time: "2-5 minutes"
      Promotion_Time: "5-10 minutes"
      Total_RTO: "7-15 minutes"
      RPO: "1-5 minutes"
      
  Benefits:
    - Simpler data consistency
    - Lower operational complexity
    - Cost-effective for standby resources
    - Predictable failover behavior
    - Easier testing and validation
    
  Challenges:
    - Underutilized standby resources
    - Higher RTO compared to active-active
    - Single point of failure (primary region)
    - Potential data loss during failover
    - Regular failover testing required
    
  Use_Cases:
    - Traditional enterprise applications
    - Write-heavy workloads
    - Strict consistency requirements
    - Budget-constrained deployments
    - Disaster recovery focused scenarios
```

### Implementation Comparison

```python
import boto3
import json
from typing import Dict, List
from datetime import datetime, timedelta

class MultiRegionPatternManager:
    def __init__(self):
        self.route53 = boto3.client('route53')
        self.cloudwatch = boto3.client('cloudwatch')
        
    def implement_active_active_pattern(self, 
                                      config: Dict) -> Dict:
        """Implement active-active multi-region pattern"""
        
        implementation_steps = []
        
        # Step 1: Set up weighted routing
        weighted_config = {
            'name': config['domain_name'],
            'type': 'A',
            'ttl': 60,
            'regions': []
        }
        
        for region_name, region_config in config['regions'].items():
            weighted_config['regions'].append({
                'region': region_name,
                'weight': region_config['weight'],
                'primary_value': region_config['endpoint'],
                'health_check_id': region_config.get('health_check_id')
            })
        
        implementation_steps.append({
            'step': 'weighted_routing',
            'action': 'create_weighted_records',
            'config': weighted_config
        })
        
        # Step 2: Set up cross-region data replication
        replication_config = {
            'database_type': config.get('database_type', 'aurora'),
            'replication_strategy': 'multi-master',
            'regions': list(config['regions'].keys())
        }
        
        implementation_steps.append({
            'step': 'data_replication',
            'action': 'setup_multi_master_replication',
            'config': replication_config
        })
        
        # Step 3: Configure session management
        session_config = {
            'session_store': 'elasticache_global',
            'regions': list(config['regions'].keys()),
            'replication_mode': 'real_time'
        }
        
        implementation_steps.append({
            'step': 'session_management',
            'action': 'setup_global_session_store',
            'config': session_config
        })
        
        # Step 4: Set up monitoring and alerting
        monitoring_config = {
            'metrics': ['latency', 'error_rate', 'throughput'],
            'regions': list(config['regions'].keys()),
            'cross_region_comparison': True
        }
        
        implementation_steps.append({
            'step': 'monitoring',
            'action': 'setup_cross_region_monitoring',
            'config': monitoring_config
        })
        
        return {
            'pattern': 'active-active',
            'implementation_steps': implementation_steps,
            'estimated_rto': '< 1 minute',
            'estimated_rpo': '< 30 seconds',
            'complexity': 'high',
            'cost_factor': 'high'
        }
    
    def implement_active_passive_pattern(self, 
                                       config: Dict) -> Dict:
        """Implement active-passive multi-region pattern"""
        
        implementation_steps = []
        
        # Step 1: Set up failover routing
        failover_config = {
            'name': config['domain_name'],
            'type': 'A',
            'primary_value': config['primary_region']['endpoint'],
            'secondary_value': config['secondary_region']['endpoint'],
            'primary_health_check_id': config['primary_region'].get('health_check_id'),
            'secondary_health_check_id': config['secondary_region'].get('health_check_id'),
            'ttl': 60
        }
        
        implementation_steps.append({
            'step': 'failover_routing',
            'action': 'create_failover_records',
            'config': failover_config
        })
        
        # Step 2: Set up master-slave replication
        replication_config = {
            'database_type': config.get('database_type', 'rds'),
            'replication_strategy': 'master-slave',
            'primary_region': config['primary_region']['name'],
            'secondary_regions': [config['secondary_region']['name']]
        }
        
        implementation_steps.append({
            'step': 'data_replication',
            'action': 'setup_master_slave_replication',
            'config': replication_config
        })
        
        # Step 3: Configure standby infrastructure
        standby_config = {
            'secondary_region': config['secondary_region']['name'],
            'scaling_mode': 'minimal_with_autoscaling',
            'warm_up_time': '5-10 minutes'
        }
        
        implementation_steps.append({
            'step': 'standby_infrastructure',
            'action': 'setup_warm_standby',
            'config': standby_config
        })
        
        # Step 4: Set up automated failover
        failover_automation_config = {
            'trigger_conditions': ['health_check_failure', 'high_error_rate'],
            'promotion_lambda': 'auto_promote_secondary',
            'notification_topics': ['ops-team', 'management']
        }
        
        implementation_steps.append({
            'step': 'automated_failover',
            'action': 'setup_failover_automation',
            'config': failover_automation_config
        })
        
        return {
            'pattern': 'active-passive',
            'implementation_steps': implementation_steps,
            'estimated_rto': '5-15 minutes',
            'estimated_rpo': '1-5 minutes',
            'complexity': 'medium',
            'cost_factor': 'medium'
        }
    
    def compare_patterns(self, 
                        use_case_requirements: Dict) -> Dict:
        """Compare patterns based on specific requirements"""
        
        comparison = {
            'active_active': {
                'score': 0,
                'pros': [],
                'cons': [],
                'suitability': 'unknown'
            },
            'active_passive': {
                'score': 0,
                'pros': [],
                'cons': [],
                'suitability': 'unknown'
            }
        }
        
        # Evaluate based on RTO requirements
        if use_case_requirements.get('rto_requirement', 600) < 60:  # < 1 minute RTO
            comparison['active_active']['score'] += 3
            comparison['active_active']['pros'].append('Immediate failover capability')
            comparison['active_passive']['cons'].append('Higher RTO than requirement')
        elif use_case_requirements.get('rto_requirement', 600) < 600:  # < 10 minutes RTO
            comparison['active_active']['score'] += 2
            comparison['active_passive']['score'] += 2
        else:
            comparison['active_passive']['score'] += 1
            
        # Evaluate based on RPO requirements
        if use_case_requirements.get('rpo_requirement', 300) < 60:  # < 1 minute RPO
            comparison['active_active']['score'] += 2
            comparison['active_active']['pros'].append('Near-zero data loss')
        elif use_case_requirements.get('rpo_requirement', 300) < 300:  # < 5 minutes RPO
            comparison['active_active']['score'] += 1
            comparison['active_passive']['score'] += 1
        else:
            comparison['active_passive']['score'] += 1
            
        # Evaluate based on budget constraints
        budget_factor = use_case_requirements.get('budget_factor', 'medium')
        if budget_factor == 'low':
            comparison['active_passive']['score'] += 2
            comparison['active_passive']['pros'].append('Lower infrastructure costs')
            comparison['active_active']['cons'].append('Higher resource utilization costs')
        elif budget_factor == 'high':
            comparison['active_active']['score'] += 1
            
        # Evaluate based on complexity tolerance
        complexity_tolerance = use_case_requirements.get('complexity_tolerance', 'medium')
        if complexity_tolerance == 'low':
            comparison['active_passive']['score'] += 2
            comparison['active_passive']['pros'].append('Simpler operational model')
            comparison['active_active']['cons'].append('Complex data consistency management')
        elif complexity_tolerance == 'high':
            comparison['active_active']['score'] += 1
            
        # Evaluate based on global user distribution
        global_users = use_case_requirements.get('global_user_distribution', False)
        if global_users:
            comparison['active_active']['score'] += 3
            comparison['active_active']['pros'].append('Optimal global performance')
            comparison['active_passive']['cons'].append('Single region serves all traffic')
        else:
            comparison['active_passive']['score'] += 1
            
        # Evaluate based on consistency requirements
        consistency_requirement = use_case_requirements.get('consistency_requirement', 'eventual')
        if consistency_requirement == 'strong':
            comparison['active_passive']['score'] += 2
            comparison['active_passive']['pros'].append('Simpler consistency model')
            comparison['active_active']['cons'].append('Eventual consistency challenges')
        elif consistency_requirement == 'eventual':
            comparison['active_active']['score'] += 1
            
        # Determine suitability
        for pattern in ['active_active', 'active_passive']:
            score = comparison[pattern]['score']
            if score >= 8:
                comparison[pattern]['suitability'] = 'excellent'
            elif score >= 6:
                comparison[pattern]['suitability'] = 'good'
            elif score >= 4:
                comparison[pattern]['suitability'] = 'fair'
            else:
                comparison[pattern]['suitability'] = 'poor'
                
        return comparison

# Usage example
pattern_manager = MultiRegionPatternManager()

# Define use case requirements
use_case = {
    'rto_requirement': 300,  # 5 minutes
    'rpo_requirement': 60,   # 1 minute
    'budget_factor': 'medium',
    'complexity_tolerance': 'medium',
    'global_user_distribution': True,
    'consistency_requirement': 'eventual'
}

# Compare patterns
comparison = pattern_manager.compare_patterns(use_case)
print("Pattern Comparison:")
print(json.dumps(comparison, indent=2))

# Implement recommended pattern
if comparison['active_active']['score'] > comparison['active_passive']['score']:
    print("\nRecommended Pattern: Active-Active")
    
    config = {
        'domain_name': 'api.example.com',
        'database_type': 'aurora',
        'regions': {
            'us-east-1': {
                'weight': 60,
                'endpoint': '1.2.3.4',
                'health_check_id': 'hc-12345'
            },
            'eu-west-1': {
                'weight': 40,
                'endpoint': '5.6.7.8',
                'health_check_id': 'hc-67890'
            }
        }
    }
    
    implementation = pattern_manager.implement_active_active_pattern(config)
    
else:
    print("\nRecommended Pattern: Active-Passive")
    
    config = {
        'domain_name': 'api.example.com',
        'database_type': 'rds',
        'primary_region': {
            'name': 'us-east-1',
            'endpoint': '1.2.3.4',
            'health_check_id': 'hc-12345'
        },
        'secondary_region': {
            'name': 'us-west-2',
            'endpoint': '5.6.7.8',
            'health_check_id': 'hc-67890'
        }
    }
    
    implementation = pattern_manager.implement_active_passive_pattern(config)

print("\nImplementation Plan:")
print(json.dumps(implementation, indent=2))
```

## RTO/RPO Considerations {#rto-rpo}

### RTO/RPO Analysis Framework

```python
import boto3
import json
from typing import Dict, List, Tuple
from datetime import datetime, timedelta
import math

class RTORPOAnalyzer:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        
    def calculate_rto_components(self, 
                               failover_config: Dict) -> Dict:
        """Calculate RTO components for different failover strategies"""
        
        rto_breakdown = {
            'detection_time': 0,
            'decision_time': 0,
            'failover_execution_time': 0,
            'recovery_verification_time': 0,
            'total_rto': 0
        }
        
        # Detection time based on health check configuration
        health_check_interval = failover_config.get('health_check_interval', 30)
        failure_threshold = failover_config.get('failure_threshold', 3)
        detection_time = health_check_interval * failure_threshold
        
        rto_breakdown['detection_time'] = detection_time
        
        # Decision time (automated vs manual)
        if failover_config.get('automated_failover', True):
            rto_breakdown['decision_time'] = 30  # 30 seconds for automated decision
        else:
            rto_breakdown['decision_time'] = 300  # 5 minutes for manual decision
            
        # Failover execution time based on strategy
        failover_strategy = failover_config.get('strategy', 'dns_failover')
        
        if failover_strategy == 'dns_failover':
            # DNS TTL + propagation time
            dns_ttl = failover_config.get('dns_ttl', 60)
            propagation_time = 60  # Additional propagation buffer
            rto_breakdown['failover_execution_time'] = dns_ttl + propagation_time
            
        elif failover_strategy == 'application_failover':
            # Application restart/redirect time
            rto_breakdown['failover_execution_time'] = 30
            
        elif failover_strategy == 'infrastructure_failover':
            # Infrastructure scaling/startup time
            startup_time = failover_config.get('infrastructure_startup_time', 300)
            rto_breakdown['failover_execution_time'] = startup_time
            
        # Recovery verification time
        rto_breakdown['recovery_verification_time'] = 60  # 1 minute for verification
        
        # Calculate total RTO
        rto_breakdown['total_rto'] = sum([
            rto_breakdown['detection_time'],
            rto_breakdown['decision_time'],
            rto_breakdown['failover_execution_time'],
            rto_breakdown['recovery_verification_time']
        ])
        
        return rto_breakdown
    
    def calculate_rpo_components(self, 
                               replication_config: Dict) -> Dict:
        """Calculate RPO components for different replication strategies"""
        
        rpo_breakdown = {
            'replication_lag': 0,
            'checkpoint_interval': 0,
            'failure_detection_window': 0,
            'total_rpo': 0
        }
        
        replication_type = replication_config.get('type', 'async')
        
        if replication_type == 'synchronous':
            # Synchronous replication - minimal data loss
            rpo_breakdown['replication_lag'] = 1  # 1 second worst case
            rpo_breakdown['total_rpo'] = 1
            
        elif replication_type == 'asynchronous':
            # Asynchronous replication
            replication_frequency = replication_config.get('frequency', 60)
            rpo_breakdown['replication_lag'] = replication_frequency
            
            # Add checkpoint interval if applicable
            if 'checkpoint_interval' in replication_config:
                rpo_breakdown['checkpoint_interval'] = replication_config['checkpoint_interval']
                
            rpo_breakdown['total_rpo'] = max(
                rpo_breakdown['replication_lag'],
                rpo_breakdown['checkpoint_interval']
            )
            
        elif replication_type == 'backup_based':
            # Backup-based recovery
            backup_frequency = replication_config.get('backup_frequency', 3600)  # 1 hour
            rpo_breakdown['total_rpo'] = backup_frequency
            
        return rpo_breakdown
    
    def analyze_business_impact(self, 
                              rto_rpo_values: Dict,
                              business_config: Dict) -> Dict:
        """Analyze business impact of RTO/RPO values"""
        
        impact_analysis = {
            'revenue_impact': {},
            'operational_impact': {},
            'compliance_impact': {},
            'customer_impact': {},
            'recommendations': []
        }
        
        # Revenue impact calculation
        hourly_revenue = business_config.get('hourly_revenue', 0)
        rto_hours = rto_rpo_values['total_rto'] / 3600  # Convert to hours
        rpo_hours = rto_rpo_values['total_rpo'] / 3600
        
        impact_analysis['revenue_impact'] = {
            'downtime_cost': hourly_revenue * rto_hours,
            'data_loss_cost': business_config.get('data_loss_cost_per_hour', 0) * rpo_hours,
            'total_financial_impact': (hourly_revenue * rto_hours) + 
                                    (business_config.get('data_loss_cost_per_hour', 0) * rpo_hours)
        }
        
        # Operational impact
        impact_analysis['operational_impact'] = {
            'affected_users': business_config.get('total_users', 0),
            'service_degradation_duration': rto_rpo_values['total_rto'],
            'manual_intervention_required': rto_rpo_values.get('manual_steps', 0)
        }
        
        # Compliance impact
        compliance_requirements = business_config.get('compliance_requirements', {})
        
        for requirement, threshold in compliance_requirements.items():
            if rto_rpo_values['total_rto'] > threshold.get('rto_limit', float('inf')):
                impact_analysis['compliance_impact'][requirement] = 'RTO_VIOLATION'
            elif rto_rpo_values['total_rpo'] > threshold.get('rpo_limit', float('inf')):
                impact_analysis['compliance_impact'][requirement] = 'RPO_VIOLATION'
            else:
                impact_analysis['compliance_impact'][requirement] = 'COMPLIANT'
        
        # Generate recommendations
        if rto_rpo_values['total_rto'] > 300:  # > 5 minutes
            impact_analysis['recommendations'].append({
                'type': 'RTO_IMPROVEMENT',
                'suggestion': 'Consider implementing active-active pattern for faster failover',
                'potential_improvement': '< 60 seconds RTO'
            })
            
        if rto_rpo_values['total_rpo'] > 60:  # > 1 minute
            impact_analysis['recommendations'].append({
                'type': 'RPO_IMPROVEMENT',
                'suggestion': 'Implement synchronous replication or increase replication frequency',
                'potential_improvement': '< 30 seconds RPO'
            })
        
        return impact_analysis
    
    def optimize_rto_rpo(self, 
                        current_config: Dict,
                        target_requirements: Dict) -> Dict:
        """Provide optimization recommendations to meet RTO/RPO targets"""
        
        optimizations = {
            'rto_optimizations': [],
            'rpo_optimizations': [],
            'cost_implications': {},
            'implementation_complexity': {}
        }
        
        current_rto = current_config['rto']['total_rto']
        current_rpo = current_config['rpo']['total_rpo']
        target_rto = target_requirements['rto_target']
        target_rpo = target_requirements['rpo_target']
        
        # RTO optimizations
        if current_rto > target_rto:
            rto_gap = current_rto - target_rto
            
            if rto_gap > 300:  # Large gap
                optimizations['rto_optimizations'].append({
                    'strategy': 'Implement Global Accelerator',
                    'improvement': '60-120 seconds',
                    'cost': 'Medium',
                    'complexity': 'Low'
                })
                
                optimizations['rto_optimizations'].append({
                    'strategy': 'Deploy active-active architecture',
                    'improvement': '180-300 seconds',
                    'cost': 'High',
                    'complexity': 'High'
                })
            
            if rto_gap > 60:  # Medium gap
                optimizations['rto_optimizations'].append({
                    'strategy': 'Reduce DNS TTL to 30 seconds',
                    'improvement': '30-60 seconds',
                    'cost': 'Low',
                    'complexity': 'Low'
                })
                
                optimizations['rto_optimizations'].append({
                    'strategy': 'Implement automated failover',
                    'improvement': '60-240 seconds',
                    'cost': 'Medium',
                    'complexity': 'Medium'
                })
        
        # RPO optimizations
        if current_rpo > target_rpo:
            rpo_gap = current_rpo - target_rpo
            
            if rpo_gap > 300:  # Large gap
                optimizations['rpo_optimizations'].append({
                    'strategy': 'Implement synchronous replication',
                    'improvement': 'Near-zero RPO',
                    'cost': 'High',
                    'complexity': 'High'
                })
            
            if rpo_gap > 60:  # Medium gap
                optimizations['rpo_optimizations'].append({
                    'strategy': 'Increase replication frequency to 30 seconds',
                    'improvement': '30-60 seconds',
                    'cost': 'Medium',
                    'complexity': 'Low'
                })
                
                optimizations['rpo_optimizations'].append({
                    'strategy': 'Implement Aurora Global Database',
                    'improvement': '< 1 second RPO',
                    'cost': 'High',
                    'complexity': 'Medium'
                })
        
        # Calculate cost implications
        total_optimizations = len(optimizations['rto_optimizations']) + len(optimizations['rpo_optimizations'])
        estimated_monthly_cost_increase = 0
        
        for opt in optimizations['rto_optimizations'] + optimizations['rpo_optimizations']:
            if opt['cost'] == 'Low':
                estimated_monthly_cost_increase += 100
            elif opt['cost'] == 'Medium':
                estimated_monthly_cost_increase += 500
            elif opt['cost'] == 'High':
                estimated_monthly_cost_increase += 2000
        
        optimizations['cost_implications'] = {
            'estimated_monthly_increase': estimated_monthly_cost_increase,
            'roi_calculation': 'Cost vs downtime reduction analysis needed'
        }
        
        return optimizations

# Usage example
analyzer = RTORPOAnalyzer()

# Current failover configuration
current_failover_config = {
    'health_check_interval': 30,
    'failure_threshold': 3,
    'automated_failover': True,
    'strategy': 'dns_failover',
    'dns_ttl': 60
}

# Current replication configuration
current_replication_config = {
    'type': 'asynchronous',
    'frequency': 120,  # 2 minutes
    'checkpoint_interval': 60
}

# Business configuration
business_config = {
    'hourly_revenue': 10000,
    'data_loss_cost_per_hour': 5000,
    'total_users': 100000,
    'compliance_requirements': {
        'SOX': {'rto_limit': 900, 'rpo_limit': 300},  # 15 min RTO, 5 min RPO
        'GDPR': {'rto_limit': 3600, 'rpo_limit': 600}  # 1 hour RTO, 10 min RPO
    }
}

# Calculate current RTO/RPO
current_rto = analyzer.calculate_rto_components(current_failover_config)
current_rpo = analyzer.calculate_rpo_components(current_replication_config)

print("Current RTO Analysis:")
print(json.dumps(current_rto, indent=2))

print("\nCurrent RPO Analysis:")
print(json.dumps(current_rpo, indent=2))

# Analyze business impact
impact = analyzer.analyze_business_impact(
    {'total_rto': current_rto['total_rto'], 'total_rpo': current_rpo['total_rpo']},
    business_config
)

print("\nBusiness Impact Analysis:")
print(json.dumps(impact, indent=2))

# Optimize for target requirements
target_requirements = {
    'rto_target': 300,  # 5 minutes
    'rpo_target': 60    # 1 minute
}

optimizations = analyzer.optimize_rto_rpo(
    {
        'rto': current_rto,
        'rpo': current_rpo
    },
    target_requirements
)

print("\nOptimization Recommendations:")
print(json.dumps(optimizations, indent=2))
```

This comprehensive guide provides a thorough foundation for implementing multi-region failover strategies with proper consideration of RTO/RPO requirements, health check configuration, routing policies, and business impact analysis. The implementations can be adapted to specific organizational needs while maintaining best practices for reliability and performance.