# Topic 62: Geolocation and Geoproximity Routing - Traffic Shaping and Compliance

## Table of Contents
1. [Geolocation and Geoproximity Overview](#overview)
2. [Geolocation Routing Configuration](#geolocation)
3. [Geoproximity Routing Setup](#geoproximity)
4. [Bias Configuration and Traffic Shaping](#bias)
5. [Compliance and Regulatory Requirements](#compliance)
6. [Advanced Traffic Management](#traffic-management)
7. [Monitoring and Analytics](#monitoring)
8. [Testing and Validation](#testing)
9. [Cost Optimization](#cost-optimization)
10. [Troubleshooting](#troubleshooting)
11. [Best Practices](#best-practices)
12. [Real-World Scenarios](#scenarios)

## Geolocation and Geoproximity Overview {#overview}

### Understanding the Differences

Route 53 provides two geographic routing policies that serve different purposes in global traffic management:

**Geolocation Routing**:
- Routes traffic based on the user's geographic location
- Allows routing by continent, country, or state/province
- Ideal for compliance, content localization, and load distribution
- Binary decision: user is either in a location or not

**Geoproximity Routing**:
- Routes traffic based on geographic distance with adjustable bias
- Uses bias values to expand or shrink geographic regions
- More sophisticated traffic shaping capabilities
- Allows fine-tuning of traffic distribution

### Architecture Comparison

```
Geolocation vs Geoproximity Routing:

Geolocation Routing:
┌─────────────────────────────────────────────────────────────────┐
│                    Geographic Boundaries                        │
│                                                                 │
│  Europe          │    North America    │    Asia Pacific       │
│  ┌─────────────┐ │  ┌─────────────────┐ │  ┌─────────────────┐ │
│  │   Users     │ │  │      Users      │ │  │      Users      │ │
│  │     ↓       │ │  │        ↓        │ │  │        ↓        │ │
│  │ eu-west-1   │ │  │    us-east-1    │ │  │  ap-south-1     │ │
│  │ Endpoint    │ │  │    Endpoint     │ │  │   Endpoint      │ │
│  └─────────────┘ │  └─────────────────┘ │  └─────────────────┘ │
│                  │                      │                      │
│  Fixed routing   │   Fixed routing     │   Fixed routing     │
│  by location     │   by location       │   by location       │
└─────────────────────────────────────────────────────────────────┘

Geoproximity Routing:
┌─────────────────────────────────────────────────────────────────┐
│                    Proximity with Bias                          │
│                                                                 │
│     Bias: +50    │      Bias: 0       │     Bias: -30         │
│  ┌─────────────┐ │  ┌─────────────────┐ │  ┌─────────────────┐ │
│  │ Expanded    │ │  │     Normal      │ │  │   Contracted    │ │
│  │  Region     │ │  │     Region      │ │  │    Region       │ │
│  │             │ │  │                 │ │  │                 │ │
│  │ eu-west-1   │ │  │    us-east-1    │ │  │  ap-south-1     │ │
│  │ Endpoint    │ │  │    Endpoint     │ │  │   Endpoint      │ │
│  └─────────────┘ │  └─────────────────┘ │  └─────────────────┘ │
│       ↑         │         ↑           │         ↑             │
│  Attracts more  │    Standard         │   Attracts less      │
│     traffic     │    proximity        │      traffic         │
└─────────────────────────────────────────────────────────────────┘
```

### Use Case Matrix

```yaml
Geographic_Routing_Use_Cases:
  
  Geolocation_Optimal_For:
    Compliance:
      - GDPR compliance (EU traffic to EU regions)
      - Data sovereignty requirements
      - Regional regulatory compliance
      - Content restrictions by geography
      
    Content_Localization:
      - Language-specific content
      - Currency localization
      - Regional promotions
      - Local pricing strategies
      
    Business_Requirements:
      - Licensing restrictions
      - Regional partnerships
      - Tax jurisdiction compliance
      - Market segmentation
      
  Geoproximity_Optimal_For:
    Performance_Optimization:
      - Latency-based routing with fine control
      - Traffic load balancing across regions
      - Capacity management
      - Resource utilization optimization
      
    Traffic_Engineering:
      - Gradual traffic migration
      - A/B testing across regions
      - Canary deployments
      - Maintenance routing
      
    Cost_Optimization:
      - Directing traffic to lower-cost regions
      - Resource efficiency improvements
      - Data transfer cost reduction
      - Infrastructure utilization balance
```

## Geolocation Routing Configuration {#geolocation}

### Basic Geolocation Setup

```python
import boto3
import json
from typing import Dict, List, Optional

class GeolocationRoutingManager:
    def __init__(self):
        self.route53 = boto3.client('route53')
        
    def create_geolocation_record(self,
                                hosted_zone_id: str,
                                record_config: Dict) -> bool:
        """Create geolocation-based DNS record"""
        
        change_batch = {
            'Comment': f"Geolocation routing for {record_config['name']}",
            'Changes': [{
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': record_config['name'],
                    'Type': record_config['type'],
                    'SetIdentifier': record_config['set_identifier'],
                    'TTL': record_config.get('ttl', 300),
                    'ResourceRecords': [{'Value': record_config['value']}]
                }
            }]
        }
        
        # Add geolocation configuration
        geolocation = {}
        if 'continent_code' in record_config:
            geolocation['ContinentCode'] = record_config['continent_code']
        if 'country_code' in record_config:
            geolocation['CountryCode'] = record_config['country_code']
        if 'subdivision_code' in record_config:
            geolocation['SubdivisionCode'] = record_config['subdivision_code']
            
        change_batch['Changes'][0]['ResourceRecordSet']['GeoLocation'] = geolocation
        
        # Add health check if provided
        if 'health_check_id' in record_config:
            change_batch['Changes'][0]['ResourceRecordSet']['HealthCheckId'] = record_config['health_check_id']
        
        try:
            response = self.route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch=change_batch
            )
            
            change_id = response['ChangeInfo']['Id']
            print(f"Created geolocation record for {record_config['name']}, Change ID: {change_id}")
            
            # Wait for change to propagate
            waiter = self.route53.get_waiter('resource_record_sets_changed')
            waiter.wait(Id=change_id)
            
            return True
            
        except Exception as e:
            print(f"Failed to create geolocation record: {e}")
            return False
    
    def setup_comprehensive_geolocation_routing(self,
                                              hosted_zone_id: str,
                                              domain_config: Dict) -> Dict:
        """Set up comprehensive geolocation routing for global application"""
        
        results = {'created_records': [], 'failed_records': []}
        
        # Create records for each geographic location
        for location_config in domain_config['locations']:
            success = self.create_geolocation_record(
                hosted_zone_id,
                location_config
            )
            
            if success:
                results['created_records'].append(location_config['set_identifier'])
            else:
                results['failed_records'].append(location_config['set_identifier'])
        
        # Create default record (required for geolocation routing)
        if 'default_record' in domain_config:
            default_config = domain_config['default_record']
            default_config['set_identifier'] = 'default-geolocation'
            
            # Default record has special geolocation setting
            change_batch = {
                'Comment': f"Default geolocation record for {default_config['name']}",
                'Changes': [{
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': default_config['name'],
                        'Type': default_config['type'],
                        'SetIdentifier': 'default-geolocation',
                        'TTL': default_config.get('ttl', 300),
                        'ResourceRecords': [{'Value': default_config['value']}],
                        'GeoLocation': {'ContinentCode': '*'}  # Default location
                    }
                }]
            }
            
            if 'health_check_id' in default_config:
                change_batch['Changes'][0]['ResourceRecordSet']['HealthCheckId'] = default_config['health_check_id']
            
            try:
                self.route53.change_resource_record_sets(
                    HostedZoneId=hosted_zone_id,
                    ChangeBatch=change_batch
                )
                results['created_records'].append('default-geolocation')
                
            except Exception as e:
                print(f"Failed to create default geolocation record: {e}")
                results['failed_records'].append('default-geolocation')
        
        return results
    
    def create_compliance_geolocation_setup(self,
                                          hosted_zone_id: str,
                                          compliance_config: Dict) -> Dict:
        """Create geolocation setup for regulatory compliance"""
        
        compliance_results = {
            'eu_gdpr_compliance': False,
            'us_data_sovereignty': False,
            'apac_local_requirements': False,
            'created_records': []
        }
        
        # EU GDPR Compliance - EU traffic stays in EU
        if 'eu_endpoint' in compliance_config:
            eu_record = {
                'name': compliance_config['domain_name'],
                'type': 'A',
                'set_identifier': 'eu-gdpr-compliance',
                'continent_code': 'EU',
                'value': compliance_config['eu_endpoint']['ip'],
                'health_check_id': compliance_config['eu_endpoint'].get('health_check_id'),
                'ttl': 60
            }
            
            if self.create_geolocation_record(hosted_zone_id, eu_record):
                compliance_results['eu_gdpr_compliance'] = True
                compliance_results['created_records'].append('eu-gdpr-compliance')
        
        # US Data Sovereignty
        if 'us_endpoint' in compliance_config:
            us_record = {
                'name': compliance_config['domain_name'],
                'type': 'A',
                'set_identifier': 'us-data-sovereignty',
                'country_code': 'US',
                'value': compliance_config['us_endpoint']['ip'],
                'health_check_id': compliance_config['us_endpoint'].get('health_check_id'),
                'ttl': 60
            }
            
            if self.create_geolocation_record(hosted_zone_id, us_record):
                compliance_results['us_data_sovereignty'] = True
                compliance_results['created_records'].append('us-data-sovereignty')
        
        # APAC Local Requirements
        if 'apac_countries' in compliance_config:
            for country, endpoint_config in compliance_config['apac_countries'].items():
                apac_record = {
                    'name': compliance_config['domain_name'],
                    'type': 'A',
                    'set_identifier': f'apac-{country.lower()}',
                    'country_code': country,
                    'value': endpoint_config['ip'],
                    'health_check_id': endpoint_config.get('health_check_id'),
                    'ttl': 60
                }
                
                if self.create_geolocation_record(hosted_zone_id, apac_record):
                    compliance_results['apac_local_requirements'] = True
                    compliance_results['created_records'].append(f'apac-{country.lower()}')
        
        return compliance_results
    
    def analyze_geolocation_coverage(self,
                                   hosted_zone_id: str,
                                   domain_name: str) -> Dict:
        """Analyze geolocation routing coverage"""
        
        try:
            # Get all records for the domain
            response = self.route53.list_resource_record_sets(
                HostedZoneId=hosted_zone_id
            )
            
            geolocation_records = []
            for record in response['ResourceRecordSets']:
                if (record['Name'].rstrip('.') == domain_name and 
                    'GeoLocation' in record):
                    geolocation_records.append(record)
            
            coverage_analysis = {
                'total_geolocation_records': len(geolocation_records),
                'continent_coverage': set(),
                'country_coverage': set(),
                'subdivision_coverage': set(),
                'has_default_record': False,
                'coverage_gaps': [],
                'recommendations': []
            }
            
            # Analyze coverage
            for record in geolocation_records:
                geo_location = record['GeoLocation']
                
                if 'ContinentCode' in geo_location:
                    if geo_location['ContinentCode'] == '*':
                        coverage_analysis['has_default_record'] = True
                    else:
                        coverage_analysis['continent_coverage'].add(geo_location['ContinentCode'])
                
                if 'CountryCode' in geo_location:
                    coverage_analysis['country_coverage'].add(geo_location['CountryCode'])
                
                if 'SubdivisionCode' in geo_location:
                    coverage_analysis['subdivision_coverage'].add(geo_location['SubdivisionCode'])
            
            # Check for gaps and provide recommendations
            all_continents = {'AF', 'AN', 'AS', 'EU', 'NA', 'OC', 'SA'}
            missing_continents = all_continents - coverage_analysis['continent_coverage']
            
            if missing_continents:
                coverage_analysis['coverage_gaps'].extend(list(missing_continents))
                coverage_analysis['recommendations'].append(
                    f"Consider adding coverage for continents: {', '.join(missing_continents)}"
                )
            
            if not coverage_analysis['has_default_record']:
                coverage_analysis['recommendations'].append(
                    "Add a default geolocation record to handle uncovered locations"
                )
            
            # Convert sets to lists for JSON serialization
            coverage_analysis['continent_coverage'] = list(coverage_analysis['continent_coverage'])
            coverage_analysis['country_coverage'] = list(coverage_analysis['country_coverage'])
            coverage_analysis['subdivision_coverage'] = list(coverage_analysis['subdivision_coverage'])
            
            return coverage_analysis
            
        except Exception as e:
            return {'error': str(e)}

# Usage example
geo_manager = GeolocationRoutingManager()

# Comprehensive geolocation configuration
domain_config = {
    'domain_name': 'api.example.com',
    'locations': [
        {
            'name': 'api.example.com',
            'type': 'A',
            'set_identifier': 'north-america',
            'continent_code': 'NA',
            'value': '54.239.28.85',
            'health_check_id': 'hc-na-12345',
            'ttl': 60
        },
        {
            'name': 'api.example.com',
            'type': 'A',
            'set_identifier': 'europe',
            'continent_code': 'EU',
            'value': '52.48.120.2',
            'health_check_id': 'hc-eu-67890',
            'ttl': 60
        },
        {
            'name': 'api.example.com',
            'type': 'A',
            'set_identifier': 'asia-pacific',
            'continent_code': 'AS',
            'value': '13.232.67.1',
            'health_check_id': 'hc-ap-abcde',
            'ttl': 60
        },
        {
            'name': 'api.example.com',
            'type': 'A',
            'set_identifier': 'germany-specific',
            'country_code': 'DE',
            'value': '3.120.181.40',
            'health_check_id': 'hc-de-fghij',
            'ttl': 60
        }
    ],
    'default_record': {
        'name': 'api.example.com',
        'type': 'A',
        'value': '54.239.28.85',  # Default to US endpoint
        'health_check_id': 'hc-default-klmno',
        'ttl': 60
    }
}

# Set up geolocation routing
hosted_zone_id = 'Z123456789ABCDEF'
results = geo_manager.setup_comprehensive_geolocation_routing(hosted_zone_id, domain_config)
print("Geolocation setup results:")
print(json.dumps(results, indent=2))

# Compliance configuration
compliance_config = {
    'domain_name': 'secure.example.com',
    'eu_endpoint': {
        'ip': '52.48.120.2',
        'health_check_id': 'hc-eu-compliance'
    },
    'us_endpoint': {
        'ip': '54.239.28.85',
        'health_check_id': 'hc-us-compliance'
    },
    'apac_countries': {
        'JP': {
            'ip': '13.113.196.1',
            'health_check_id': 'hc-jp-compliance'
        },
        'SG': {
            'ip': '13.229.187.8',
            'health_check_id': 'hc-sg-compliance'
        }
    }
}

# Set up compliance routing
compliance_results = geo_manager.create_compliance_geolocation_setup(
    hosted_zone_id, 
    compliance_config
)
print("\nCompliance setup results:")
print(json.dumps(compliance_results, indent=2))

# Analyze coverage
coverage_analysis = geo_manager.analyze_geolocation_coverage(
    hosted_zone_id, 
    'api.example.com'
)
print("\nCoverage analysis:")
print(json.dumps(coverage_analysis, indent=2))
```

### Advanced Geolocation Patterns

```bash
#!/bin/bash
# Advanced geolocation routing setup script

HOSTED_ZONE_ID="Z123456789ABCDEF"
DOMAIN_NAME="global.example.com"

echo "Setting up advanced geolocation routing patterns..."

# Function to create geolocation record
create_geo_record() {
    local identifier=$1
    local geo_type=$2
    local geo_value=$3
    local ip_address=$4
    local health_check_id=$5
    
    echo "Creating geolocation record: $identifier"
    
    # Build geolocation configuration
    case $geo_type in
        "continent")
            geo_config='"ContinentCode": "'$geo_value'"'
            ;;
        "country")
            geo_config='"CountryCode": "'$geo_value'"'
            ;;
        "subdivision")
            geo_config='"CountryCode": "US", "SubdivisionCode": "'$geo_value'"'
            ;;
        "default")
            geo_config='"ContinentCode": "*"'
            ;;
    esac
    
    # Create the record
    aws route53 change-resource-record-sets \
        --hosted-zone-id $HOSTED_ZONE_ID \
        --change-batch '{
            "Comment": "Geolocation record for '$identifier'",
            "Changes": [
                {
                    "Action": "UPSERT",
                    "ResourceRecordSet": {
                        "Name": "'$DOMAIN_NAME'",
                        "Type": "A",
                        "SetIdentifier": "'$identifier'",
                        "GeoLocation": {
                            '$geo_config'
                        },
                        "TTL": 60,
                        "ResourceRecords": [
                            {
                                "Value": "'$ip_address'"
                            }
                        ],
                        "HealthCheckId": "'$health_check_id'"
                    }
                }
            ]
        }'
}

# Continental routing
create_geo_record "north-america-continent" "continent" "NA" "54.239.28.85" "hc-na-001"
create_geo_record "europe-continent" "continent" "EU" "52.48.120.2" "hc-eu-001"
create_geo_record "asia-continent" "continent" "AS" "13.232.67.1" "hc-as-001"

# Country-specific routing for compliance
create_geo_record "germany-country" "country" "DE" "3.120.181.40" "hc-de-001"
create_geo_record "united-kingdom-country" "country" "GB" "18.130.91.1" "hc-gb-001"
create_geo_record "japan-country" "country" "JP" "13.113.196.1" "hc-jp-001"
create_geo_record "singapore-country" "country" "SG" "13.229.187.8" "hc-sg-001"

# US state-specific routing
create_geo_record "california-state" "subdivision" "CA" "54.215.226.1" "hc-ca-001"
create_geo_record "new-york-state" "subdivision" "NY" "54.158.220.1" "hc-ny-001"
create_geo_record "texas-state" "subdivision" "TX" "3.16.146.1" "hc-tx-001"

# Default record (required)
create_geo_record "default-global" "default" "*" "54.239.28.85" "hc-default-001"

echo "Advanced geolocation routing setup completed"

# Verify configuration
echo "Verifying geolocation configuration..."
aws route53 list-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --query "ResourceRecordSets[?Name=='$DOMAIN_NAME.']" \
    --output table
```

## Geoproximity Routing Setup {#geoproximity}

### Geoproximity Configuration

```python
import boto3
import json
from typing import Dict, List, Optional
import math

class GeoproximityRoutingManager:
    def __init__(self):
        self.route53 = boto3.client('route53')
        
    def create_geoproximity_record(self,
                                 hosted_zone_id: str,
                                 record_config: Dict) -> bool:
        """Create geoproximity-based DNS record"""
        
        change_batch = {
            'Comment': f"Geoproximity routing for {record_config['name']}",
            'Changes': [{
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': record_config['name'],
                    'Type': record_config['type'],
                    'SetIdentifier': record_config['set_identifier'],
                    'TTL': record_config.get('ttl', 300),
                    'ResourceRecords': [{'Value': record_config['value']}]
                }
            }]
        }
        
        # Add geoproximity configuration
        geoproximity = {}
        
        if 'aws_region' in record_config:
            geoproximity['AWSRegion'] = record_config['aws_region']
        elif 'coordinates' in record_config:
            geoproximity['LocalZoneGroup'] = record_config['coordinates']
        
        # Add bias if specified
        if 'bias' in record_config:
            geoproximity['Bias'] = record_config['bias']
        
        change_batch['Changes'][0]['ResourceRecordSet']['GeoProximityLocation'] = geoproximity
        
        # Add health check if provided
        if 'health_check_id' in record_config:
            change_batch['Changes'][0]['ResourceRecordSet']['HealthCheckId'] = record_config['health_check_id']
        
        try:
            response = self.route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch=change_batch
            )
            
            change_id = response['ChangeInfo']['Id']
            print(f"Created geoproximity record for {record_config['name']}, Change ID: {change_id}")
            
            return True
            
        except Exception as e:
            print(f"Failed to create geoproximity record: {e}")
            return False
    
    def setup_traffic_shaping_with_bias(self,
                                      hosted_zone_id: str,
                                      shaping_config: Dict) -> Dict:
        """Set up traffic shaping using geoproximity bias"""
        
        results = {
            'created_records': [],
            'bias_configuration': {},
            'expected_traffic_distribution': {}
        }
        
        total_capacity = sum(
            endpoint['capacity'] for endpoint in shaping_config['endpoints']
        )
        
        for endpoint in shaping_config['endpoints']:
            # Calculate bias based on desired traffic percentage
            desired_percentage = endpoint.get('traffic_percentage', 
                                            (endpoint['capacity'] / total_capacity) * 100)
            
            # Bias calculation: +bias expands region, -bias contracts it
            # Formula: bias = (desired_percentage - natural_percentage) * bias_factor
            natural_percentage = (endpoint['capacity'] / total_capacity) * 100
            bias_factor = shaping_config.get('bias_sensitivity', 2)
            calculated_bias = int((desired_percentage - natural_percentage) * bias_factor)
            
            # Clamp bias to Route 53 limits (-99 to +99)
            bias = max(-99, min(99, calculated_bias))
            
            record_config = {
                'name': shaping_config['domain_name'],
                'type': 'A',
                'set_identifier': f"{endpoint['region']}-geoproximity",
                'aws_region': endpoint['region'],
                'bias': bias,
                'value': endpoint['ip_address'],
                'health_check_id': endpoint.get('health_check_id'),
                'ttl': shaping_config.get('ttl', 60)
            }
            
            if self.create_geoproximity_record(hosted_zone_id, record_config):
                results['created_records'].append(record_config['set_identifier'])
                results['bias_configuration'][endpoint['region']] = {
                    'bias': bias,
                    'target_percentage': desired_percentage,
                    'natural_percentage': natural_percentage
                }
        
        return results
    
    def simulate_bias_impact(self,
                           endpoint_configs: List[Dict]) -> Dict:
        """Simulate the impact of different bias values on traffic distribution"""
        
        simulation_results = {
            'scenarios': {},
            'recommendations': []
        }
        
        # Test different bias scenarios
        bias_scenarios = [
            {'name': 'no_bias', 'description': 'Natural geoproximity', 'bias_adjustments': {}},
            {'name': 'expand_primary', 'description': 'Expand primary region', 'bias_adjustments': {'primary': +30}},
            {'name': 'contract_secondary', 'description': 'Contract secondary region', 'bias_adjustments': {'secondary': -20}},
            {'name': 'balanced_adjustment', 'description': 'Balanced traffic shaping', 'bias_adjustments': {'primary': +10, 'secondary': -10}}
        ]
        
        for scenario in bias_scenarios:
            scenario_result = {
                'description': scenario['description'],
                'endpoint_configurations': [],
                'estimated_coverage': {}
            }
            
            for endpoint in endpoint_configs:
                endpoint_bias = scenario['bias_adjustments'].get(
                    endpoint.get('role', 'default'), 0
                )
                
                # Simulate coverage area based on bias
                base_coverage = endpoint.get('base_coverage_radius', 1000)  # km
                bias_multiplier = 1 + (endpoint_bias / 100)  # 1% per bias point
                estimated_coverage = base_coverage * bias_multiplier
                
                scenario_result['endpoint_configurations'].append({
                    'region': endpoint['region'],
                    'original_bias': endpoint.get('bias', 0),
                    'scenario_bias': endpoint_bias,
                    'estimated_coverage_km': estimated_coverage,
                    'coverage_change_percent': (bias_multiplier - 1) * 100
                })
                
                scenario_result['estimated_coverage'][endpoint['region']] = estimated_coverage
            
            simulation_results['scenarios'][scenario['name']] = scenario_result
        
        # Generate recommendations
        for scenario_name, scenario_data in simulation_results['scenarios'].items():
            if scenario_name != 'no_bias':
                total_coverage_change = sum(
                    abs(config['coverage_change_percent']) 
                    for config in scenario_data['endpoint_configurations']
                )
                
                if total_coverage_change > 50:  # Significant change
                    simulation_results['recommendations'].append({
                        'scenario': scenario_name,
                        'impact': 'high',
                        'description': f"Significant traffic redistribution ({total_coverage_change:.1f}% total change)",
                        'considerations': [
                            'Monitor latency impacts',
                            'Validate capacity planning',
                            'Test gradual bias adjustments'
                        ]
                    })
        
        return simulation_results
    
    def optimize_geoproximity_bias(self,
                                 current_metrics: Dict,
                                 optimization_goals: Dict) -> Dict:
        """Optimize geoproximity bias based on current metrics and goals"""
        
        optimization_results = {
            'current_performance': {},
            'recommended_adjustments': {},
            'expected_improvements': {},
            'implementation_plan': []
        }
        
        # Analyze current performance
        for region, metrics in current_metrics.items():
            optimization_results['current_performance'][region] = {
                'avg_latency_ms': metrics.get('avg_latency', 0),
                'traffic_percentage': metrics.get('traffic_percentage', 0),
                'error_rate_percent': metrics.get('error_rate', 0),
                'capacity_utilization': metrics.get('capacity_utilization', 0)
            }
        
        # Generate optimization recommendations
        for region, goals in optimization_goals.items():
            current = optimization_results['current_performance'].get(region, {})
            
            recommended_bias = 0
            adjustments = []
            
            # Latency optimization
            if goals.get('target_latency') and current.get('avg_latency_ms', 0) > goals['target_latency']:
                # Increase bias to attract more local traffic
                latency_adjustment = min(20, (current['avg_latency_ms'] - goals['target_latency']) // 10)
                recommended_bias += latency_adjustment
                adjustments.append(f"Latency optimization: +{latency_adjustment} bias")
            
            # Traffic distribution optimization
            if goals.get('target_traffic_percentage'):
                current_traffic = current.get('traffic_percentage', 0)
                traffic_gap = goals['target_traffic_percentage'] - current_traffic
                
                if abs(traffic_gap) > 5:  # >5% difference
                    traffic_adjustment = int(traffic_gap * 0.8)  # Conservative adjustment
                    recommended_bias += traffic_adjustment
                    adjustments.append(f"Traffic distribution: {traffic_adjustment:+d} bias")
            
            # Capacity optimization
            if goals.get('target_utilization'):
                current_utilization = current.get('capacity_utilization', 0)
                if current_utilization > goals['target_utilization']:
                    # Reduce traffic to over-utilized region
                    utilization_adjustment = -min(15, (current_utilization - goals['target_utilization']) // 5)
                    recommended_bias += utilization_adjustment
                    adjustments.append(f"Capacity optimization: {utilization_adjustment} bias")
            
            # Clamp total bias to valid range
            recommended_bias = max(-99, min(99, recommended_bias))
            
            if recommended_bias != 0:
                optimization_results['recommended_adjustments'][region] = {
                    'current_bias': current_metrics[region].get('current_bias', 0),
                    'recommended_bias': recommended_bias,
                    'bias_change': recommended_bias - current_metrics[region].get('current_bias', 0),
                    'reasons': adjustments
                }
                
                # Estimate improvements
                optimization_results['expected_improvements'][region] = {
                    'latency_improvement_percent': max(0, (current.get('avg_latency_ms', 0) - goals.get('target_latency', current.get('avg_latency_ms', 0))) / current.get('avg_latency_ms', 1) * 100),
                    'traffic_redistribution_percent': abs(goals.get('target_traffic_percentage', current.get('traffic_percentage', 0)) - current.get('traffic_percentage', 0))
                }
        
        # Create implementation plan
        if optimization_results['recommended_adjustments']:
            optimization_results['implementation_plan'] = [
                {
                    'phase': 1,
                    'description': 'Implement 50% of recommended bias changes',
                    'duration': '1 week',
                    'monitoring': 'Intensive monitoring of latency and error rates'
                },
                {
                    'phase': 2,
                    'description': 'Evaluate results and implement remaining changes',
                    'duration': '1 week',
                    'monitoring': 'Continue monitoring and fine-tuning'
                },
                {
                    'phase': 3,
                    'description': 'Stabilization and ongoing optimization',
                    'duration': 'Ongoing',
                    'monitoring': 'Regular review and adjustment cycle'
                }
            ]
        
        return optimization_results

# Usage example
geoprox_manager = GeoproximityRoutingManager()

# Traffic shaping configuration
shaping_config = {
    'domain_name': 'app.example.com',
    'bias_sensitivity': 2,
    'ttl': 60,
    'endpoints': [
        {
            'region': 'us-east-1',
            'ip_address': '54.239.28.85',
            'capacity': 1000,
            'traffic_percentage': 60,  # Want 60% of traffic
            'health_check_id': 'hc-use1-001',
            'role': 'primary'
        },
        {
            'region': 'eu-west-1',
            'ip_address': '52.48.120.2',
            'capacity': 600,
            'traffic_percentage': 25,  # Want 25% of traffic
            'health_check_id': 'hc-euw1-001',
            'role': 'secondary'
        },
        {
            'region': 'ap-south-1',
            'ip_address': '13.232.67.1',
            'capacity': 400,
            'traffic_percentage': 15,  # Want 15% of traffic
            'health_check_id': 'hc-aps1-001'
        }
    ]
}

# Set up traffic shaping
hosted_zone_id = 'Z123456789ABCDEF'
shaping_results = geoprox_manager.setup_traffic_shaping_with_bias(
    hosted_zone_id, 
    shaping_config
)
print("Traffic shaping results:")
print(json.dumps(shaping_results, indent=2))

# Simulate bias impact
endpoint_configs = [
    {
        'region': 'us-east-1',
        'role': 'primary',
        'base_coverage_radius': 2000,
        'bias': 0
    },
    {
        'region': 'eu-west-1',
        'role': 'secondary',
        'base_coverage_radius': 1500,
        'bias': 0
    }
]

bias_simulation = geoprox_manager.simulate_bias_impact(endpoint_configs)
print("\nBias simulation results:")
print(json.dumps(bias_simulation, indent=2))

# Optimization example
current_metrics = {
    'us-east-1': {
        'avg_latency': 120,
        'traffic_percentage': 70,
        'error_rate': 2.1,
        'capacity_utilization': 85,
        'current_bias': 10
    },
    'eu-west-1': {
        'avg_latency': 80,
        'traffic_percentage': 20,
        'error_rate': 1.8,
        'capacity_utilization': 45,
        'current_bias': -5
    }
}

optimization_goals = {
    'us-east-1': {
        'target_latency': 100,
        'target_traffic_percentage': 60,
        'target_utilization': 75
    },
    'eu-west-1': {
        'target_latency': 90,
        'target_traffic_percentage': 30,
        'target_utilization': 65
    }
}

optimization_results = geoprox_manager.optimize_geoproximity_bias(
    current_metrics,
    optimization_goals
)
print("\nOptimization results:")
print(json.dumps(optimization_results, indent=2))
```

This comprehensive guide provides detailed implementations for both geolocation and geoproximity routing with practical examples, traffic shaping capabilities, compliance configurations, and optimization strategies. The code examples can be adapted to specific organizational requirements while maintaining best practices for global traffic management.