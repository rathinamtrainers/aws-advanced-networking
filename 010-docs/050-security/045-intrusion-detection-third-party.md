# Topic 45: Intrusion Detection with Third-Party Appliances

## Introduction

While AWS provides native security services, many organizations require specialized intrusion detection and prevention systems (IDS/IPS) from third-party vendors. This guide covers deploying and managing third-party security appliances in AWS environments.

## IDS/IPS Fundamentals

### Types of Intrusion Detection Systems

#### Network-Based IDS (NIDS)
- **Traffic Analysis**: Monitors network segments for suspicious activity
- **Signature Detection**: Pattern matching against known threats
- **Anomaly Detection**: Behavioral analysis for unknown threats
- **Real-time Monitoring**: Continuous network surveillance

#### Host-Based IDS (HIDS)
- **System Monitoring**: File integrity and system call monitoring
- **Log Analysis**: Security event correlation and analysis
- **Behavioral Monitoring**: Process and user activity tracking
- **Endpoint Protection**: Individual host security

#### Hybrid Systems
- **Network + Host**: Combined NIDS and HIDS capabilities
- **Cloud-Native**: AWS-optimized security appliances
- **Management Integration**: Centralized security orchestration

### Detection Methods

#### Signature-Based Detection
```
Known Threat Patterns → Signature Database → Pattern Matching → Alert Generation
```

#### Anomaly-Based Detection
```
Baseline Learning → Behavioral Model → Deviation Detection → Alert Scoring
```

#### Heuristic Analysis
```
Rule-Based Logic → Suspicious Behavior → Risk Scoring → Threat Classification
```

## AWS Integration Architecture

### Deployment Models

#### Inline Security Appliances
```
Internet → IGW → IDS/IPS Appliance → Protected Resources
```

**Characteristics**:
- All traffic passes through security appliance
- Active blocking capabilities
- Potential single point of failure
- Latency impact on traffic flow

#### Out-of-Band Monitoring
```
Production Traffic → Mirror/Copy → IDS Appliance → SIEM/Alerting
```

**Characteristics**:
- No impact on production traffic
- Detection only (no blocking)
- Requires traffic mirroring mechanisms
- Parallel processing architecture

#### Hybrid Deployment
```
Critical Traffic → Inline IPS
All Traffic → Out-of-Band IDS → Analytics Platform
```

### Traffic Mirroring Options

#### VPC Traffic Mirroring
```bash
# Create traffic mirror target
aws ec2 create-traffic-mirror-target \
    --description "IDS Appliance Target" \
    --network-interface-id eni-ids-appliance

# Create traffic mirror filter
aws ec2 create-traffic-mirror-filter \
    --description "All Traffic Filter" \
    --tag-specifications 'ResourceType=traffic-mirror-filter,Tags=[{Key=Name,Value=IDS-Filter}]'

# Add filter rules
aws ec2 create-traffic-mirror-filter-rule \
    --traffic-mirror-filter-id tmf-12345678 \
    --traffic-direction ingress \
    --rule-number 100 \
    --rule-action accept \
    --destination-cidr-block 0.0.0.0/0 \
    --source-cidr-block 0.0.0.0/0

# Create mirror session
aws ec2 create-traffic-mirror-session \
    --description "Web Server Monitoring" \
    --traffic-mirror-target-id tmt-12345678 \
    --traffic-mirror-filter-id tmf-12345678 \
    --network-interface-id eni-webserver \
    --session-number 1
```

#### Gateway Load Balancer Integration
```bash
# Deploy IDS behind GWLB
Internet → IGW → GWLB → IDS Appliances → Protected Subnets
```

## Popular Third-Party Solutions

### Palo Alto Networks VM-Series

#### Deployment Architecture
```yaml
VM-Series Configuration:
  Instance Type: c5.xlarge or larger
  Management Interface: Dedicated management subnet
  Data Interfaces: 
    - Trust interface (internal traffic)
    - Untrust interface (external traffic)
  Licensing: BYOL or PAYG models
```

#### Bootstrap Configuration
```xml
<?xml version="1.0"?>
<config>
  <devices>
    <entry name="localhost.localdomain">
      <deviceconfig>
        <system>
          <hostname>palo-alto-fw</hostname>
          <dns-setting>
            <servers>
              <primary>169.254.169.253</primary>
            </servers>
          </dns-setting>
        </system>
      </deviceconfig>
      <network>
        <interface>
          <ethernet>
            <entry name="ethernet1/1">
              <layer3>
                <dhcp-client>
                  <enable>yes</enable>
                </dhcp-client>
              </layer3>
            </entry>
          </ethernet>
        </interface>
      </network>
    </entry>
  </devices>
</config>
```

### Check Point CloudGuard

#### Auto Scaling Configuration
```json
{
  "AutoScalingGroup": {
    "MinSize": 2,
    "MaxSize": 10,
    "DesiredCapacity": 2,
    "HealthCheckType": "ELB",
    "HealthCheckGracePeriod": 300,
    "LaunchTemplate": {
      "ImageId": "ami-checkpoint-r81",
      "InstanceType": "c5.large",
      "SecurityGroups": ["sg-checkpoint"],
      "UserData": {
        "template_name": "gateway-scaling",
        "management_server": "checkpoint-mgmt.example.com",
        "sic_key": "secret-key"
      }
    }
  }
}
```

### Fortinet FortiGate

#### High Availability Setup
```bash
# Master FortiGate configuration
config system ha
    set group-name "aws-ha-cluster"
    set mode a-p
    set hbdev "port1" 50
    set session-pickup enable
    set session-pickup-connectionless enable
    set ha-mgmt-status enable
    config ha-mgmt-interfaces
        edit 1
            set interface "port1"
            set gateway 192.168.1.1
        next
    end
end
```

### Suricata Open Source

#### Custom Deployment
```bash
#!/bin/bash
# Suricata installation script

# Install dependencies
yum update -y
yum install -y epel-release
yum install -y suricata

# Configure network interface
cat > /etc/sysconfig/suricata << EOF
# Interface to monitor
SURICATA_INTERFACE=eth1
# Additional options
SURICATA_OPTIONS="--user suricata"
EOF

# Configure Suricata
cat > /etc/suricata/suricata.yaml << EOF
vars:
  address-groups:
    HOME_NET: "[10.0.0.0/8,192.168.0.0/16,172.16.0.0/12]"
    EXTERNAL_NET: "!$HOME_NET"

af-packet:
  - interface: eth1
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes

outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: eve.json
      types:
        - alert
        - http
        - dns
        - tls

rule-files:
  - suricata.rules
  - emerging-threats.rules
EOF

# Start services
systemctl enable suricata
systemctl start suricata
```

## Integration with AWS Services

### CloudWatch Integration

#### Custom Metrics Publishing
```python
import boto3
import json
from datetime import datetime

def publish_ids_metrics(alerts):
    cloudwatch = boto3.client('cloudwatch')
    
    for alert in alerts:
        cloudwatch.put_metric_data(
            Namespace='Security/IDS',
            MetricData=[
                {
                    'MetricName': 'ThreatDetections',
                    'Dimensions': [
                        {
                            'Name': 'Severity',
                            'Value': alert['severity']
                        },
                        {
                            'Name': 'SourceIP',
                            'Value': alert['src_ip']
                        }
                    ],
                    'Value': 1,
                    'Unit': 'Count',
                    'Timestamp': datetime.utcnow()
                }
            ]
        )
```

#### CloudWatch Alarms
```bash
# Create alarm for high threat activity
aws cloudwatch put-metric-alarm \
    --alarm-name "High-Threat-Activity" \
    --alarm-description "High number of security threats detected" \
    --metric-name ThreatDetections \
    --namespace Security/IDS \
    --statistic Sum \
    --period 300 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:us-west-2:123456789012:security-alerts
```

### SIEM Integration

#### Splunk Integration
```python
import splunklib.client as client

def send_to_splunk(alert_data):
    service = client.connect(
        host='splunk.example.com',
        port=8089,
        username='admin',
        password='password'
    )
    
    index = service.indexes['security']
    index.submit(
        json.dumps(alert_data),
        sourcetype='ids_alert',
        source='aws_ids'
    )
```

#### Elasticsearch Integration
```python
from elasticsearch import Elasticsearch

def send_to_elasticsearch(alert):
    es = Elasticsearch(['elasticsearch.example.com:9200'])
    
    doc = {
        'timestamp': alert['timestamp'],
        'source_ip': alert['src_ip'],
        'destination_ip': alert['dest_ip'],
        'alert_severity': alert['severity'],
        'signature': alert['signature'],
        'classification': alert['classification']
    }
    
    es.index(
        index='security-alerts',
        doc_type='ids_alert',
        body=doc
    )
```

## Automation and Orchestration

### Automated Response Systems

#### Lambda-Based Response
```python
import boto3
import json

def lambda_handler(event, context):
    """
    Automated response to IDS alerts
    """
    ec2 = boto3.client('ec2')
    
    # Parse alert from IDS
    alert = json.loads(event['Records'][0]['body'])
    
    if alert['severity'] == 'critical':
        # Isolate compromised instance
        instance_id = get_instance_by_ip(alert['src_ip'])
        
        if instance_id:
            # Create isolation security group
            isolation_sg = create_isolation_sg()
            
            # Replace instance security groups
            ec2.modify_instance_attribute(
                InstanceId=instance_id,
                Groups=[isolation_sg]
            )
            
            # Send notification
            send_notification(f"Instance {instance_id} isolated due to critical alert")
    
    return {'statusCode': 200}

def create_isolation_sg():
    ec2 = boto3.client('ec2')
    
    response = ec2.create_security_group(
        GroupName='isolation-sg',
        Description='Isolation security group for compromised instances',
        VpcId='vpc-12345678'
    )
    
    sg_id = response['GroupId']
    
    # Add minimal rules for investigation
    ec2.authorize_security_group_ingress(
        GroupId=sg_id,
        IpPermissions=[
            {
                'IpProtocol': 'tcp',
                'FromPort': 22,
                'ToPort': 22,
                'IpRanges': [{'CidrIp': '10.0.0.0/8'}]
            }
        ]
    )
    
    return sg_id
```

### Infrastructure as Code

#### Terraform Configuration
```hcl
# Third-party IDS deployment
resource "aws_instance" "ids_appliance" {
  count           = var.ids_instance_count
  ami             = var.ids_ami_id
  instance_type   = var.ids_instance_type
  subnet_id       = aws_subnet.ids_subnet[count.index].id
  security_groups = [aws_security_group.ids_sg.id]
  
  root_block_device {
    volume_type = "gp3"
    volume_size = 100
    encrypted   = true
  }
  
  user_data = templatefile("${path.module}/scripts/ids-bootstrap.sh", {
    management_server = var.ids_management_server
    license_key      = var.ids_license_key
  })
  
  tags = {
    Name = "IDS-Appliance-${count.index + 1}"
    Type = "Security"
  }
}

# Traffic mirroring setup
resource "aws_ec2_traffic_mirror_target" "ids_target" {
  description          = "IDS Traffic Mirror Target"
  network_interface_id = aws_instance.ids_appliance[0].primary_network_interface_id
  
  tags = {
    Name = "IDS-Mirror-Target"
  }
}

resource "aws_ec2_traffic_mirror_filter" "ids_filter" {
  description = "All Traffic Filter for IDS"
  
  tags = {
    Name = "IDS-Traffic-Filter"
  }
}

resource "aws_ec2_traffic_mirror_filter_rule" "ids_rule_ingress" {
  description              = "Ingress traffic rule"
  traffic_mirror_filter_id = aws_ec2_traffic_mirror_filter.ids_filter.id
  destination_cidr_block   = "0.0.0.0/0"
  source_cidr_block        = "0.0.0.0/0"
  rule_action              = "accept"
  rule_number              = 100
  traffic_direction        = "ingress"
}
```

## Performance and Scaling

### Capacity Planning

#### Throughput Requirements
```bash
# Calculate required bandwidth
Peak_Traffic_Gbps = 10
Inspection_Overhead = 1.2
Required_Capacity = Peak_Traffic_Gbps * Inspection_Overhead
Instance_Capacity = 2  # Gbps per instance
Required_Instances = ceil(Required_Capacity / Instance_Capacity)
```

#### Instance Sizing Guidelines
```yaml
Traffic Volume Guidelines:
  Low (< 1 Gbps): c5.large
  Medium (1-5 Gbps): c5.xlarge  
  High (5-10 Gbps): c5.2xlarge
  Very High (> 10 Gbps): c5.4xlarge or cluster
```

### Auto Scaling Configuration

#### CloudWatch-Based Scaling
```json
{
  "AutoScalingPolicy": {
    "PolicyName": "IDS-CPU-Scaling",
    "PolicyType": "TargetTrackingScaling",
    "TargetTrackingConfiguration": {
      "TargetValue": 70.0,
      "PredefinedMetricSpecification": {
        "PredefinedMetricType": "ASGAverageCPUUtilization"
      }
    }
  }
}
```

## Cost Optimization

### Licensing Models

#### BYOL (Bring Your Own License)
- **Lower Instance Costs**: Pay only for compute
- **Flexible Licensing**: Use existing enterprise licenses
- **Long-term Savings**: Better for sustained usage

#### PAYG (Pay As You Go)
- **No Upfront Costs**: License included in hourly rate
- **Flexible Scaling**: Scale up/down without license constraints
- **Good for Testing**: Lower initial investment

### Cost Management Strategies

#### Reserved Instances
```bash
# Purchase reserved instances for predictable workloads
aws ec2 purchase-reserved-instances-offering \
    --reserved-instances-offering-id offering-12345678 \
    --instance-count 2
```

#### Spot Instances for Development
```bash
# Use spot instances for non-critical IDS testing
aws ec2 request-spot-instances \
    --spot-price "0.05" \
    --instance-count 1 \
    --type "one-time" \
    --launch-specification file://ids-launch-spec.json
```

## Monitoring and Alerting

### Key Performance Metrics

#### System Metrics
- **CPU Utilization**: Processing load
- **Memory Usage**: Buffer and cache utilization
- **Network Throughput**: Traffic processing rate
- **Disk I/O**: Log writing performance

#### Security Metrics
- **Alert Volume**: Number of security alerts
- **False Positive Rate**: Alert accuracy
- **Detection Rate**: Threat identification efficiency
- **Response Time**: Alert to action time

### Alerting Configuration

#### Critical Alert Automation
```python
def process_critical_alert(alert):
    """
    Handle critical security alerts
    """
    # Immediate notification
    send_sms_alert(alert['summary'])
    
    # Create incident ticket
    create_incident_ticket(alert)
    
    # Auto-isolate if configured
    if alert['confidence'] > 0.9:
        isolate_source_ip(alert['src_ip'])
    
    # Update security dashboard
    update_security_dashboard(alert)
```

## Best Practices

### Deployment Best Practices
1. **High Availability**: Deploy across multiple AZs
2. **Performance Testing**: Validate throughput requirements
3. **Security Hardening**: Secure management interfaces
4. **Network Segmentation**: Isolate security appliances

### Operational Best Practices
1. **Regular Updates**: Keep signatures and software current
2. **Tuning**: Minimize false positives
3. **Monitoring**: Continuous performance monitoring
4. **Documentation**: Maintain operational procedures

### Integration Best Practices
1. **Centralized Management**: Use unified security console
2. **Automation**: Implement automated response workflows
3. **Correlation**: Integrate with SIEM systems
4. **Compliance**: Ensure regulatory requirements are met

## Conclusion

Third-party IDS/IPS solutions provide specialized security capabilities that complement AWS native services. Proper deployment, integration, and management ensure effective threat detection while maintaining performance and cost efficiency in AWS environments.