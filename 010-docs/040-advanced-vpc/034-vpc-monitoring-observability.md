# Topic 34: VPC Monitoring and Observability

## Prerequisites
- Completed Topic 33 (Advanced VPC Design Patterns and Architectures)
- Understanding of monitoring principles and observability concepts
- Knowledge of AWS monitoring services (CloudWatch, X-Ray, CloudTrail)
- Familiarity with network performance metrics and troubleshooting

## Learning Objectives
By the end of this topic, you will be able to:
- Implement comprehensive VPC monitoring and observability solutions
- Configure advanced VPC Flow Logs analysis and visualization
- Set up network performance monitoring and alerting systems
- Design distributed tracing for network-aware applications
- Create automated incident response and remediation workflows

## Theory

### VPC Observability Framework

#### Observability Pillars for VPC
```
VPC Observability Stack:
┌─────────────────────────────────────────────────────┐
│ Business Impact Metrics (SLA, Revenue Impact)      │
├─────────────────────────────────────────────────────┤
│ Application Performance (Latency, Throughput)      │
├─────────────────────────────────────────────────────┤
│ Network Performance (RTT, Packet Loss, Jitter)     │
├─────────────────────────────────────────────────────┤
│ Infrastructure Metrics (Utilization, Capacity)     │
├─────────────────────────────────────────────────────┤
│ Security Events (Threats, Violations, Anomalies)   │
└─────────────────────────────────────────────────────┘
```

#### Three Pillars of Observability
- **Metrics**: Quantitative measurements over time (latency, throughput, error rates)
- **Logs**: Discrete events with context (VPC Flow Logs, application logs, audit logs)
- **Traces**: Request journey across distributed systems (API calls, service interactions)

### VPC Flow Logs Advanced Analytics

#### Flow Log Record Format (Version 5)
```
Enhanced Flow Log Fields:
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Basic Fields    │ Traffic Fields  │ Network Fields  │ AWS Fields      │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ version         │ srcaddr         │ interface-id    │ vpc-id          │
│ account-id      │ dstaddr         │ subnet-id       │ region          │
│ start           │ srcport         │ az-id           │ pkt-src-aws-svc │
│ end             │ dstport         │ sublocation-id  │ pkt-dst-aws-svc │
│ packets         │ protocol        │ sublocation-type│ flow-direction  │
│ bytes           │ tcp-flags       │                 │ traffic-path    │
│ windowstart     │ type            │                 │                 │
│ windowend       │ pkt-srcaddr     │                 │                 │
│ action          │ pkt-dstaddr     │                 │                 │
│ flowlogstatus   │                 │                 │                 │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

#### Flow Log Analysis Patterns
```sql
-- Top talkers by bytes transferred
SELECT 
    srcaddr, 
    dstaddr, 
    SUM(bytes) as total_bytes,
    COUNT(*) as flow_count
FROM vpc_flow_logs 
WHERE start >= UNIX_TIMESTAMP(NOW() - INTERVAL 1 HOUR)
GROUP BY srcaddr, dstaddr 
ORDER BY total_bytes DESC 
LIMIT 10;

-- Rejected traffic analysis
SELECT 
    srcaddr,
    dstaddr,
    dstport,
    protocol,
    COUNT(*) as rejected_flows
FROM vpc_flow_logs 
WHERE action = 'REJECT' 
    AND start >= UNIX_TIMESTAMP(NOW() - INTERVAL 1 HOUR)
GROUP BY srcaddr, dstaddr, dstport, protocol
ORDER BY rejected_flows DESC;

-- Network anomaly detection
SELECT 
    srcaddr,
    AVG(bytes) as avg_bytes,
    STDDEV(bytes) as stddev_bytes,
    MAX(bytes) as max_bytes
FROM vpc_flow_logs 
WHERE start >= UNIX_TIMESTAMP(NOW() - INTERVAL 24 HOUR)
GROUP BY srcaddr
HAVING max_bytes > (avg_bytes + 3 * stddev_bytes);
```

### Network Performance Monitoring

#### Key Performance Indicators (KPIs)
```
Network Performance Metrics:
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Metric Category │ Primary KPI     │ Secondary KPI   │ Threshold       │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Latency         │ RTT (ms)        │ Jitter (ms)     │ <10ms intra-AZ  │
│ Throughput      │ Bandwidth (Gbps)│ PPS (packets/s) │ >80% utilization│
│ Reliability     │ Packet Loss (%) │ Error Rate (%)  │ <0.01% loss     │
│ Availability    │ Uptime (%)      │ MTTR (minutes)  │ >99.9% uptime   │
│ Security        │ Threat Count    │ Blocked Flows   │ 0 critical      │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

#### Enhanced Monitoring Architecture
```
CloudWatch Enhanced Monitoring:
┌─────────────────────────────────────────────────────┐
│                VPC Resources                        │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │   EC2       │ │    ELB      │ │   RDS       │   │
│ │ Instances   │ │ LoadBalancer│ │  Database   │   │
│ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘   │
│        │ CloudWatch    │ CloudWatch    │ CloudWatch│
│        │ Agent         │ Logs          │ Insights  │
└────────┼───────────────┼───────────────┼───────────┘
         │               │               │
         ▼               ▼               ▼
┌─────────────────────────────────────────────────────┐
│              CloudWatch Metrics                     │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │  Network    │ │ Application │ │ Infrastructure│ │
│ │  Metrics    │ │   Metrics   │ │   Metrics     │ │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────┼───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│              Analysis & Alerting                    │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ CloudWatch  │ │  X-Ray      │ │ Third-party │   │
│ │ Insights    │ │ Tracing     │ │ Tools       │   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Distributed Tracing for Network-Aware Applications

#### X-Ray Integration Architecture
```
Application Tracing with Network Context:
┌─────────────────────────────────────────────────────┐
│                 User Request                        │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────┐
│              Application Load Balancer              │
│ ┌─────────────────────────────────────────────────┐ │
│ │ X-Ray Trace: Request-ID, Timing, Status        │ │
│ └─────────────────────────────────────────────────┘ │
└─────────────────┬───────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────┐
│              Application Services                   │
│                                                     │
│ ┌─────────────┐    ┌─────────────┐    ┌──────────┐ │
│ │ Frontend    │    │  Backend    │    │ Database │ │
│ │ Service     │    │  Service    │    │ Service  │ │
│ │             │    │             │    │          │ │
│ │ Trace:      │────│ Trace:      │────│ Trace:   │ │
│ │ - Latency   │    │ - Latency   │    │ - Query  │ │
│ │ - Errors    │    │ - Calls     │    │ - Time   │ │
│ │ - Network   │    │ - Network   │    │ - Locks  │ │
│ └─────────────┘    └─────────────┘    └──────────┘ │
└─────────────────────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────┐
│              X-Ray Service Map                      │
│                                                     │
│  Service A ──(95ms)──► Service B ──(45ms)──► DB    │
│     │                     │                   │     │
│  Errors: 2%            Errors: 0.1%      Timeout:  │
│  Latency: p99=150ms    Latency: p99=80ms   0.01%   │
└─────────────────────────────────────────────────────┘
```

### Security Monitoring and Threat Detection

#### VPC Security Monitoring Framework
```
Security Event Detection Pipeline:
┌─────────────────────────────────────────────────────┐
│                 Data Sources                        │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ VPC Flow    │ │ CloudTrail  │ │   DNS       │   │
│ │ Logs        │ │ API Logs    │ │   Logs      │   │
│ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘   │
└────────┼───────────────┼───────────────┼───────────┘
         │               │               │
         ▼               ▼               ▼
┌─────────────────────────────────────────────────────┐
│              Real-time Processing                   │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │  Kinesis    │ │  Lambda     │ │ EventBridge │   │
│ │ Analytics   │ │ Functions   │ │    Rules    │   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────┼───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│              Threat Detection                       │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ GuardDuty   │ │   Custom    │ │  Security   │   │
│ │ Findings    │ │   Rules     │ │    Hub      │   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────┼───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│              Response Actions                       │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │   SNS       │ │  Lambda     │ │   SOAR      │   │
│ │ Alerts      │ │ Remediate   │ │ Platform    │   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────┘
```

#### Anomaly Detection Patterns
```python
# Network anomaly detection algorithms
class NetworkAnomalyDetector:
    def __init__(self):
        self.baseline_metrics = {}
        self.thresholds = {
            'bandwidth_spike': 3.0,  # 3 standard deviations
            'connection_rate': 1000,  # connections per minute
            'geo_anomaly': 0.95,     # 95% confidence
            'protocol_deviation': 0.1  # 10% deviation from normal
        }
    
    def detect_bandwidth_anomaly(self, current_bandwidth, historical_data):
        """Detect unusual bandwidth patterns"""
        mean = np.mean(historical_data)
        std = np.std(historical_data)
        z_score = (current_bandwidth - mean) / std
        
        return {
            'is_anomaly': abs(z_score) > self.thresholds['bandwidth_spike'],
            'severity': 'high' if abs(z_score) > 4 else 'medium',
            'z_score': z_score,
            'baseline': mean
        }
    
    def detect_geo_anomaly(self, source_ip, historical_locations):
        """Detect access from unusual geographic locations"""
        # Use IP geolocation service
        current_location = self.get_ip_location(source_ip)
        
        # Calculate distance from normal access patterns
        distances = [
            self.calculate_distance(current_location, loc) 
            for loc in historical_locations
        ]
        
        min_distance = min(distances) if distances else float('inf')
        
        return {
            'is_anomaly': min_distance > 1000,  # 1000km threshold
            'distance': min_distance,
            'location': current_location
        }
    
    def detect_protocol_anomaly(self, current_protocols, baseline_protocols):
        """Detect unusual protocol usage patterns"""
        anomalies = []
        
        for protocol, current_count in current_protocols.items():
            baseline_count = baseline_protocols.get(protocol, 0)
            baseline_total = sum(baseline_protocols.values())
            current_total = sum(current_protocols.values())
            
            baseline_ratio = baseline_count / baseline_total if baseline_total > 0 else 0
            current_ratio = current_count / current_total if current_total > 0 else 0
            
            deviation = abs(current_ratio - baseline_ratio)
            
            if deviation > self.thresholds['protocol_deviation']:
                anomalies.append({
                    'protocol': protocol,
                    'deviation': deviation,
                    'current_ratio': current_ratio,
                    'baseline_ratio': baseline_ratio
                })
        
        return anomalies
```

### Automated Incident Response

#### Incident Response Workflow
```
Automated Response Pipeline:
┌─────────────────────────────────────────────────────┐
│                Alert Generation                     │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ CloudWatch  │ │  Custom     │ │  GuardDuty  │   │
│ │ Alarms      │ │  Metrics    │ │  Findings   │   │
│ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘   │
└────────┼───────────────┼───────────────┼───────────┘
         │               │               │
         ▼               ▼               ▼
┌─────────────────────────────────────────────────────┐
│              Event Processing                       │
│ ┌─────────────────────────────────────────────────┐ │
│ │ EventBridge Rule Engine                         │ │
│ │ - Priority Classification                       │ │
│ │ - Context Enrichment                           │ │
│ │ - Escalation Logic                             │ │
│ └─────────────────────────────────────────────────┘ │
└─────────────────┼───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│              Response Actions                       │
│                                                     │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│ │ Immediate   │ │ Automated   │ │   Manual    │   │
│ │ Response    │ │ Remediation │ │ Escalation  │   │
│ │             │ │             │ │             │   │
│ │ - Block IP  │ │ - Restart   │ │ - PagerDuty │   │
│ │ - Isolate   │ │   Service   │ │ - Slack     │   │
│ │ - Scale     │ │ - Update SG │ │ - Email     │   │
│ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────┘
```

## Lab Exercise: Comprehensive VPC Monitoring and Observability

### Lab Duration: 360 minutes

### Step 1: Advanced VPC Flow Logs Setup
**Objective**: Configure comprehensive VPC Flow Logs with enhanced fields and analysis

**Create Enhanced Flow Logs Configuration**:
```yaml
# flow-logs-config.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Enhanced VPC Flow Logs with comprehensive monitoring'

Parameters:
  VpcId:
    Type: String
    Description: VPC ID for flow logs
  
  FlowLogFormat:
    Type: String
    Default: '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${windowstart} ${windowend} ${action} ${flowlogstatus} ${vpc-id} ${subnet-id} ${instance-id} ${tcp-flags} ${type} ${pkt-srcaddr} ${pkt-dstaddr} ${region} ${az-id} ${sublocation-type} ${sublocation-id} ${pkt-src-aws-service} ${pkt-dst-aws-service} ${flow-direction} ${traffic-path}'
    Description: Custom flow log format with enhanced fields

Resources:
  # S3 Bucket for Flow Logs
  FlowLogsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::StackName}-vpc-flow-logs-${AWS::AccountId}'
      VersioningConfiguration:
        Status: Enabled
      LifecycleConfiguration:
        Rules:
          - Id: FlowLogsRetention
            Status: Enabled
            ExpirationInDays: 90
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
              - TransitionInDays: 60
                StorageClass: GLACIER
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      NotificationConfiguration:
        CloudWatchConfigurations:
          - Event: s3:ObjectCreated:*
            CloudWatchConfiguration:
              LogGroupName: !Ref FlowLogsAnalysisLogGroup

  # CloudWatch Log Group for Analysis
  FlowLogsAnalysisLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/vpc/flowlogs-analysis/${AWS::StackName}'
      RetentionInDays: 30

  # Flow Logs to S3
  VPCFlowLogsS3:
    Type: AWS::EC2::FlowLog
    Properties:
      ResourceType: VPC
      ResourceId: !Ref VpcId
      TrafficType: ALL
      LogDestinationType: s3
      LogDestination: !Sub '${FlowLogsBucket}/vpc-flow-logs/'
      LogFormat: !Ref FlowLogFormat
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-vpc-flow-logs-s3'
        - Key: Type
          Value: VPCFlowLogs

  # Flow Logs to CloudWatch
  FlowLogsRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: vpc-flow-logs.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: FlowLogsDeliveryRolePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                  - logs:DescribeLogGroups
                  - logs:DescribeLogStreams
                Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:*'

  FlowLogsCloudWatch:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/vpc/flowlogs/${AWS::StackName}'
      RetentionInDays: 7

  VPCFlowLogsCloudWatch:
    Type: AWS::EC2::FlowLog
    Properties:
      ResourceType: VPC
      ResourceId: !Ref VpcId
      TrafficType: ALL
      LogDestinationType: cloud-watch-logs
      LogDestination: !GetAtt FlowLogsCloudWatch.Arn
      DeliverLogsPermissionArn: !GetAtt FlowLogsRole.Arn
      LogFormat: !Ref FlowLogFormat
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-vpc-flow-logs-cloudwatch'

  # Kinesis Data Firehose for Real-time Analysis
  FlowLogsFirehoseRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: firehose.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: FirehoseDeliveryPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:AbortMultipartUpload
                  - s3:GetBucketLocation
                  - s3:GetObject
                  - s3:ListBucket
                  - s3:ListBucketMultipartUploads
                  - s3:PutObject
                Resource:
                  - !Sub '${FlowLogsBucket}/*'
                  - !GetAtt FlowLogsBucket.Arn

  FlowLogsFirehose:
    Type: AWS::KinesisFirehose::DeliveryStream
    Properties:
      DeliveryStreamName: !Sub '${AWS::StackName}-vpc-flow-logs-firehose'
      DeliveryStreamType: DirectPut
      S3DestinationConfiguration:
        BucketARN: !GetAtt FlowLogsBucket.Arn
        Prefix: 'firehose-data/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/'
        ErrorOutputPrefix: 'errors/'
        BufferingHints:
          SizeInMBs: 128
          IntervalInSeconds: 60
        CompressionFormat: GZIP
        RoleARN: !GetAtt FlowLogsFirehoseRole.Arn
        ProcessingConfiguration:
          Enabled: true
          Processors:
            - Type: Lambda
              Parameters:
                - ParameterName: LambdaArn
                  ParameterValue: !GetAtt FlowLogsProcessor.Arn

  # Lambda function for Flow Logs processing
  FlowLogsProcessorRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: FlowLogsProcessorPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:*'

  FlowLogsProcessor:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub '${AWS::StackName}-flow-logs-processor'
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt FlowLogsProcessorRole.Arn
      Timeout: 60
      Code:
        ZipFile: |
          import json
          import base64
          import gzip
          from datetime import datetime
          
          def lambda_handler(event, context):
              output = []
              
              for record in event['records']:
                  # Decode the data
                  compressed_payload = base64.b64decode(record['data'])
                  uncompressed_payload = gzip.decompress(compressed_payload)
                  data = uncompressed_payload.decode('utf-8')
                  
                  # Process each flow log record
                  for line in data.strip().split('\n'):
                      if line.strip():
                          processed_record = process_flow_log_record(line)
                          
                          # Encode the processed data
                          output_record = {
                              'recordId': record['recordId'],
                              'result': 'Ok',
                              'data': base64.b64encode(
                                  (json.dumps(processed_record) + '\n').encode('utf-8')
                              ).decode('utf-8')
                          }
                          output.append(output_record)
              
              return {'records': output}
          
          def process_flow_log_record(log_line):
              fields = log_line.split(' ')
              
              # Parse the flow log record based on the format
              if len(fields) >= 14:
                  record = {
                      'timestamp': datetime.fromtimestamp(int(fields[9])).isoformat(),
                      'account_id': fields[1],
                      'vpc_id': fields[14] if len(fields) > 14 else '',
                      'subnet_id': fields[15] if len(fields) > 15 else '',
                      'instance_id': fields[16] if len(fields) > 16 else '',
                      'srcaddr': fields[3],
                      'dstaddr': fields[4],
                      'srcport': fields[5],
                      'dstport': fields[6],
                      'protocol': fields[7],
                      'packets': int(fields[8]) if fields[8].isdigit() else 0,
                      'bytes': int(fields[9]) if fields[9].isdigit() else 0,
                      'action': fields[12],
                      'flow_log_status': fields[13],
                      'traffic_type': classify_traffic(fields[5], fields[6], fields[7]),
                      'geo_location': get_geo_location(fields[3]),
                      'threat_indicator': detect_threats(fields)
                  }
                  
                  return record
              
              return {'error': 'Invalid log format', 'raw': log_line}
          
          def classify_traffic(src_port, dst_port, protocol):
              """Classify traffic type based on ports and protocol"""
              common_ports = {
                  '80': 'HTTP',
                  '443': 'HTTPS',
                  '22': 'SSH',
                  '21': 'FTP',
                  '25': 'SMTP',
                  '53': 'DNS',
                  '3306': 'MySQL',
                  '5432': 'PostgreSQL',
                  '6379': 'Redis',
                  '27017': 'MongoDB'
              }
              
              if dst_port in common_ports:
                  return common_ports[dst_port]
              elif src_port in common_ports:
                  return common_ports[src_port]
              else:
                  return f'Protocol-{protocol}'
          
          def get_geo_location(ip_address):
              """Get geographic location of IP address"""
              # In production, use a real geolocation service
              if ip_address.startswith('10.') or ip_address.startswith('192.168.') or ip_address.startswith('172.'):
                  return 'Private'
              else:
                  return 'External'
          
          def detect_threats(fields):
              """Basic threat detection"""
              threats = []
              
              src_addr = fields[3]
              dst_port = fields[6]
              packets = int(fields[8]) if fields[8].isdigit() else 0
              
              # Detect potential port scanning
              if packets == 1 and dst_port in ['22', '23', '80', '443', '3389']:
                  threats.append('potential_port_scan')
              
              # Detect high packet count (potential DDoS)
              if packets > 1000:
                  threats.append('high_packet_volume')
              
              # Detect suspicious protocols
              if fields[7] in ['1', '2']:  # ICMP or IGMP
                  threats.append('unusual_protocol')
              
              return threats

Outputs:
  FlowLogsBucketName:
    Description: S3 bucket for VPC Flow Logs
    Value: !Ref FlowLogsBucket
    Export:
      Name: !Sub '${AWS::StackName}-FlowLogsBucket'

  FlowLogsCloudWatchGroup:
    Description: CloudWatch Log Group for VPC Flow Logs
    Value: !Ref FlowLogsCloudWatch
    Export:
      Name: !Sub '${AWS::StackName}-FlowLogsCloudWatch'

  FirehoseDeliveryStream:
    Description: Kinesis Firehose delivery stream for real-time processing
    Value: !Ref FlowLogsFirehose
    Export:
      Name: !Sub '${AWS::StackName}-FirehoseStream'
```

**Deploy Enhanced Flow Logs**:
```bash
# Deploy the flow logs stack
aws cloudformation create-stack \
  --stack-name vpc-flow-logs-monitoring \
  --template-body file://flow-logs-config.yaml \
  --parameters ParameterKey=VpcId,ParameterValue=vpc-1234567890abcdef0 \
  --capabilities CAPABILITY_IAM

# Verify flow logs are working
aws logs describe-log-groups --log-group-name-prefix "/aws/vpc/flowlogs"
aws s3 ls s3://vpc-flow-logs-monitoring-bucket/vpc-flow-logs/ --recursive
```

### Step 2: Network Performance Monitoring Setup
**Objective**: Implement comprehensive network performance monitoring with custom metrics

**Create Network Performance Monitoring Script**:
```python
#!/usr/bin/env python3
# scripts/network_performance_monitor.py

import boto3
import json
import time
import threading
import subprocess
import statistics
from datetime import datetime, timedelta
from typing import Dict, List, Tuple
import psutil
import requests

class NetworkPerformanceMonitor:
    def __init__(self, region='us-west-2'):
        self.cloudwatch = boto3.client('cloudwatch', region_name=region)
        self.ec2 = boto3.client('ec2', region_name=region)
        self.region = region
        self.instance_id = self.get_instance_id()
        self.vpc_id = self.get_vpc_id()
        
    def get_instance_id(self):
        """Get current EC2 instance ID"""
        try:
            response = requests.get(
                'http://169.254.169.254/latest/meta-data/instance-id',
                timeout=2
            )
            return response.text
        except:
            return 'unknown'
    
    def get_vpc_id(self):
        """Get VPC ID for current instance"""
        try:
            if self.instance_id != 'unknown':
                response = self.ec2.describe_instances(
                    InstanceIds=[self.instance_id]
                )
                return response['Reservations'][0]['Instances'][0]['VpcId']
        except:
            pass
        return 'unknown'
    
    def measure_latency(self, target_host: str, count: int = 5) -> Dict:
        """Measure network latency to target host"""
        try:
            cmd = f"ping -c {count} {target_host}"
            result = subprocess.run(
                cmd.split(), 
                capture_output=True, 
                text=True, 
                timeout=30
            )
            
            if result.returncode == 0:
                lines = result.stdout.split('\n')
                
                # Parse ping output for latency metrics
                rtt_times = []
                packet_loss = 0
                
                for line in lines:
                    if 'time=' in line:
                        rtt = float(line.split('time=')[1].split()[0])
                        rtt_times.append(rtt)
                    elif 'packet loss' in line:
                        packet_loss = float(line.split('%')[0].split()[-1])
                
                if rtt_times:
                    return {
                        'target': target_host,
                        'packet_loss_percent': packet_loss,
                        'rtt_min': min(rtt_times),
                        'rtt_max': max(rtt_times),
                        'rtt_avg': statistics.mean(rtt_times),
                        'rtt_stddev': statistics.stdev(rtt_times) if len(rtt_times) > 1 else 0,
                        'jitter': statistics.stdev(rtt_times) if len(rtt_times) > 1 else 0,
                        'timestamp': datetime.utcnow().isoformat()
                    }
        except Exception as e:
            print(f"Error measuring latency to {target_host}: {e}")
        
        return {
            'target': target_host,
            'error': 'measurement_failed',
            'timestamp': datetime.utcnow().isoformat()
        }
    
    def measure_bandwidth(self, duration: int = 10) -> Dict:
        """Measure network bandwidth utilization"""
        try:
            # Get initial network stats
            initial_stats = psutil.net_io_counters()
            initial_time = time.time()
            
            # Wait for measurement period
            time.sleep(duration)
            
            # Get final network stats
            final_stats = psutil.net_io_counters()
            final_time = time.time()
            
            # Calculate bandwidth metrics
            time_delta = final_time - initial_time
            bytes_sent = final_stats.bytes_sent - initial_stats.bytes_sent
            bytes_recv = final_stats.bytes_recv - initial_stats.bytes_recv
            packets_sent = final_stats.packets_sent - initial_stats.packets_sent
            packets_recv = final_stats.packets_recv - initial_stats.packets_recv
            
            return {
                'duration_seconds': time_delta,
                'bytes_sent_per_second': bytes_sent / time_delta,
                'bytes_recv_per_second': bytes_recv / time_delta,
                'packets_sent_per_second': packets_sent / time_delta,
                'packets_recv_per_second': packets_recv / time_delta,
                'total_bytes_sent': bytes_sent,
                'total_bytes_recv': bytes_recv,
                'total_packets_sent': packets_sent,
                'total_packets_recv': packets_recv,
                'errors_in': final_stats.errin - initial_stats.errin,
                'errors_out': final_stats.errout - initial_stats.errout,
                'drops_in': final_stats.dropin - initial_stats.dropin,
                'drops_out': final_stats.dropout - initial_stats.dropout,
                'timestamp': datetime.utcnow().isoformat()
            }
        except Exception as e:
            print(f"Error measuring bandwidth: {e}")
            return {'error': 'measurement_failed', 'timestamp': datetime.utcnow().isoformat()}
    
    def measure_tcp_connections(self) -> Dict:
        """Measure TCP connection statistics"""
        try:
            connections = psutil.net_connections(kind='tcp')
            
            stats = {
                'total_connections': len(connections),
                'established': 0,
                'listen': 0,
                'time_wait': 0,
                'close_wait': 0,
                'syn_sent': 0,
                'syn_recv': 0,
                'fin_wait1': 0,
                'fin_wait2': 0,
                'closing': 0,
                'last_ack': 0,
                'unknown': 0,
                'timestamp': datetime.utcnow().isoformat()
            }
            
            for conn in connections:
                status = conn.status.lower() if hasattr(conn, 'status') else 'unknown'
                if status in stats:
                    stats[status] += 1
                else:
                    stats['unknown'] += 1
            
            return stats
            
        except Exception as e:
            print(f"Error measuring TCP connections: {e}")
            return {'error': 'measurement_failed', 'timestamp': datetime.utcnow().isoformat()}
    
    def publish_metrics(self, metrics: Dict, namespace: str = 'VPC/NetworkPerformance'):
        """Publish metrics to CloudWatch"""
        try:
            metric_data = []
            timestamp = datetime.utcnow()
            
            # Prepare metric data for CloudWatch
            for metric_name, value in metrics.items():
                if isinstance(value, (int, float)) and metric_name != 'timestamp':
                    metric_data.append({
                        'MetricName': metric_name,
                        'Value': value,
                        'Unit': self.get_metric_unit(metric_name),
                        'Timestamp': timestamp,
                        'Dimensions': [
                            {
                                'Name': 'InstanceId',
                                'Value': self.instance_id
                            },
                            {
                                'Name': 'VpcId',
                                'Value': self.vpc_id
                            }
                        ]
                    })
            
            # Publish metrics in batches (CloudWatch limit is 20 metrics per call)
            for i in range(0, len(metric_data), 20):
                batch = metric_data[i:i+20]
                self.cloudwatch.put_metric_data(
                    Namespace=namespace,
                    MetricData=batch
                )
                
            print(f"Published {len(metric_data)} metrics to CloudWatch")
            
        except Exception as e:
            print(f"Error publishing metrics: {e}")
    
    def get_metric_unit(self, metric_name: str) -> str:
        """Get appropriate unit for metric"""
        unit_mapping = {
            'rtt_min': 'Milliseconds',
            'rtt_max': 'Milliseconds',
            'rtt_avg': 'Milliseconds',
            'rtt_stddev': 'Milliseconds',
            'jitter': 'Milliseconds',
            'packet_loss_percent': 'Percent',
            'bytes_sent_per_second': 'Bytes/Second',
            'bytes_recv_per_second': 'Bytes/Second',
            'packets_sent_per_second': 'Count/Second',
            'packets_recv_per_second': 'Count/Second',
            'total_bytes_sent': 'Bytes',
            'total_bytes_recv': 'Bytes',
            'total_packets_sent': 'Count',
            'total_packets_recv': 'Count',
            'total_connections': 'Count',
            'established': 'Count',
            'listen': 'Count',
            'time_wait': 'Count',
            'errors_in': 'Count',
            'errors_out': 'Count',
            'drops_in': 'Count',
            'drops_out': 'Count'
        }
        
        return unit_mapping.get(metric_name, 'None')
    
    def run_continuous_monitoring(self, interval: int = 60):
        """Run continuous monitoring with specified interval"""
        print(f"Starting continuous network monitoring (interval: {interval}s)")
        
        # Define monitoring targets
        targets = [
            '8.8.8.8',  # Google DNS
            '1.1.1.1',  # Cloudflare DNS
            'aws.amazon.com'  # AWS
        ]
        
        while True:
            try:
                # Measure latency to multiple targets
                for target in targets:
                    latency_metrics = self.measure_latency(target)
                    if 'error' not in latency_metrics:
                        # Add target as dimension
                        latency_metrics['target_host'] = target
                        self.publish_metrics(latency_metrics, f'VPC/NetworkLatency/{target.replace(".", "_")}')
                
                # Measure bandwidth
                bandwidth_metrics = self.measure_bandwidth(10)
                if 'error' not in bandwidth_metrics:
                    self.publish_metrics(bandwidth_metrics, 'VPC/NetworkBandwidth')
                
                # Measure TCP connections
                tcp_metrics = self.measure_tcp_connections()
                if 'error' not in tcp_metrics:
                    self.publish_metrics(tcp_metrics, 'VPC/TCPConnections')
                
                print(f"Monitoring cycle completed at {datetime.utcnow().isoformat()}")
                
                # Wait for next interval
                time.sleep(interval)
                
            except KeyboardInterrupt:
                print("Monitoring stopped by user")
                break
            except Exception as e:
                print(f"Error in monitoring cycle: {e}")
                time.sleep(interval)

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='VPC Network Performance Monitor')
    parser.add_argument('--region', default='us-west-2', help='AWS region')
    parser.add_argument('--interval', type=int, default=60, help='Monitoring interval in seconds')
    parser.add_argument('--single-run', action='store_true', help='Run single monitoring cycle')
    
    args = parser.parse_args()
    
    monitor = NetworkPerformanceMonitor(region=args.region)
    
    if args.single_run:
        # Single monitoring cycle
        latency_metrics = monitor.measure_latency('8.8.8.8')
        bandwidth_metrics = monitor.measure_bandwidth(5)
        tcp_metrics = monitor.measure_tcp_connections()
        
        print("Latency Metrics:", json.dumps(latency_metrics, indent=2))
        print("Bandwidth Metrics:", json.dumps(bandwidth_metrics, indent=2))
        print("TCP Metrics:", json.dumps(tcp_metrics, indent=2))
        
        # Publish metrics
        monitor.publish_metrics(latency_metrics, 'VPC/NetworkLatency/Test')
        monitor.publish_metrics(bandwidth_metrics, 'VPC/NetworkBandwidth')
        monitor.publish_metrics(tcp_metrics, 'VPC/TCPConnections')
    else:
        # Continuous monitoring
        monitor.run_continuous_monitoring(args.interval)

if __name__ == '__main__':
    main()
```

**Install and Run Network Performance Monitor**:
```bash
# Install dependencies
pip3 install boto3 psutil requests

# Run single monitoring cycle
python3 scripts/network_performance_monitor.py --single-run

# Run continuous monitoring
python3 scripts/network_performance_monitor.py --interval 30

# Run as background service
nohup python3 scripts/network_performance_monitor.py --interval 60 > /var/log/network-monitor.log 2>&1 &
```

### Step 3: CloudWatch Dashboard and Alarms
**Objective**: Create comprehensive CloudWatch dashboards and alerting

**Create CloudWatch Dashboard Configuration**:
```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "VPC/NetworkLatency/8_8_8_8", "rtt_avg", { "stat": "Average" } ],
          [ ".", "rtt_max", { "stat": "Maximum" } ],
          [ ".", "rtt_min", { "stat": "Minimum" } ],
          [ ".", "jitter", { "stat": "Average" } ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-west-2",
        "title": "Network Latency Metrics",
        "period": 300,
        "yAxis": {
          "left": {
            "min": 0
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "VPC/NetworkBandwidth", "bytes_sent_per_second", { "stat": "Average" } ],
          [ ".", "bytes_recv_per_second", { "stat": "Average" } ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-west-2",
        "title": "Network Bandwidth Utilization",
        "period": 300,
        "yAxis": {
          "left": {
            "min": 0
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 6,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ "VPC/TCPConnections", "total_connections", { "stat": "Sum" } ],
          [ ".", "established", { "stat": "Sum" } ],
          [ ".", "time_wait", { "stat": "Sum" } ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-west-2",
        "title": "TCP Connection Statistics",
        "period": 300
      }
    },
    {
      "type": "metric",
      "x": 8,
      "y": 6,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ "VPC/NetworkBandwidth", "errors_in", { "stat": "Sum" } ],
          [ ".", "errors_out", { "stat": "Sum" } ],
          [ ".", "drops_in", { "stat": "Sum" } ],
          [ ".", "drops_out", { "stat": "Sum" } ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-west-2",
        "title": "Network Errors and Drops",
        "period": 300
      }
    },
    {
      "type": "metric",
      "x": 16,
      "y": 6,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ "VPC/NetworkLatency/8_8_8_8", "packet_loss_percent", { "stat": "Average" } ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-west-2",
        "title": "Packet Loss Percentage",
        "period": 300,
        "yAxis": {
          "left": {
            "min": 0,
            "max": 100
          }
        }
      }
    },
    {
      "type": "log",
      "x": 0,
      "y": 12,
      "width": 24,
      "height": 6,
      "properties": {
        "query": "SOURCE '/aws/vpc/flowlogs/vpc-flow-logs-monitoring'\n| fields @timestamp, srcaddr, dstaddr, srcport, dstport, protocol, action\n| filter action = \"REJECT\"\n| stats count() by srcaddr\n| sort count desc\n| limit 20",
        "region": "us-west-2",
        "title": "Top Rejected Source IPs (Last Hour)",
        "view": "table"
      }
    },
    {
      "type": "log",
      "x": 0,
      "y": 18,
      "width": 12,
      "height": 6,
      "properties": {
        "query": "SOURCE '/aws/vpc/flowlogs/vpc-flow-logs-monitoring'\n| fields @timestamp, srcaddr, dstaddr, bytes\n| filter @timestamp > @timestamp - 1h\n| stats sum(bytes) as total_bytes by bin(5m)\n| sort @timestamp desc",
        "region": "us-west-2",
        "title": "Traffic Volume Over Time",
        "view": "bar"
      }
    },
    {
      "type": "log",
      "x": 12,
      "y": 18,
      "width": 12,
      "height": 6,
      "properties": {
        "query": "SOURCE '/aws/vpc/flowlogs/vpc-flow-logs-monitoring'\n| fields @timestamp, dstport, protocol\n| filter @timestamp > @timestamp - 1h\n| stats count() as connection_count by dstport\n| sort connection_count desc\n| limit 10",
        "region": "us-west-2",
        "title": "Top Destination Ports",
        "view": "pie"
      }
    }
  ]
}
```

**Create CloudWatch Dashboard**:
```bash
# Create the dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "VPC-Network-Monitoring" \
  --dashboard-body file://dashboard-config.json

# Create CloudWatch Alarms
aws cloudwatch put-metric-alarm \
  --alarm-name "High-Network-Latency" \
  --alarm-description "Alert when network latency is high" \
  --metric-name "rtt_avg" \
  --namespace "VPC/NetworkLatency/8_8_8_8" \
  --statistic "Average" \
  --period 300 \
  --threshold 100 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-west-2:123456789012:network-alerts"

aws cloudwatch put-metric-alarm \
  --alarm-name "High-Packet-Loss" \
  --alarm-description "Alert when packet loss is detected" \
  --metric-name "packet_loss_percent" \
  --namespace "VPC/NetworkLatency/8_8_8_8" \
  --statistic "Average" \
  --period 300 \
  --threshold 1 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 1 \
  --alarm-actions "arn:aws:sns:us-west-2:123456789012:network-alerts"

aws cloudwatch put-metric-alarm \
  --alarm-name "High-Network-Errors" \
  --alarm-description "Alert when network errors are detected" \
  --metric-name "errors_in" \
  --namespace "VPC/NetworkBandwidth" \
  --statistic "Sum" \
  --period 300 \
  --threshold 10 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 2 \
  --alarm-actions "arn:aws:sns:us-west-2:123456789012:network-alerts"
```

### Step 4: X-Ray Distributed Tracing Setup
**Objective**: Implement distributed tracing for network-aware applications

**Create X-Ray Enabled Application**:
```python
# xray-network-app.py
import json
import time
import requests
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all
from flask import Flask, request, jsonify
import boto3

# Patch AWS SDK and requests
patch_all()

app = Flask(__name__)

# Configure X-Ray
xray_recorder.configure(
    context_missing='LOG_ERROR',
    plugins=('EC2Plugin', 'ECSPlugin'),
    daemon_address='127.0.0.1:2000',
    dynamic_naming='*'
)

@xray_recorder.capture('network_health_check')
def check_network_health():
    """Perform network health checks with X-Ray tracing"""
    results = {}
    
    # Test external connectivity
    external_endpoints = [
        'https://httpbin.org/status/200',
        'https://api.github.com/status',
        'https://aws.amazon.com'
    ]
    
    for endpoint in external_endpoints:
        subsegment = xray_recorder.begin_subsegment(f'external_call_{endpoint}')
        try:
            start_time = time.time()
            response = requests.get(endpoint, timeout=5)
            end_time = time.time()
            
            results[endpoint] = {
                'status_code': response.status_code,
                'latency_ms': (end_time - start_time) * 1000,
                'success': response.status_code == 200
            }
            
            # Add metadata to X-Ray trace
            subsegment.put_metadata('response_time', end_time - start_time)
            subsegment.put_metadata('status_code', response.status_code)
            subsegment.put_annotation('endpoint', endpoint)
            subsegment.put_annotation('success', response.status_code == 200)
            
        except Exception as e:
            results[endpoint] = {
                'error': str(e),
                'success': False
            }
            subsegment.add_exception(e)
        finally:
            xray_recorder.end_subsegment()
    
    return results

@xray_recorder.capture('aws_service_check')
def check_aws_services():
    """Check AWS service connectivity with X-Ray tracing"""
    results = {}
    
    # Test AWS service connectivity
    services = {
        's3': boto3.client('s3'),
        'ec2': boto3.client('ec2'),
        'cloudwatch': boto3.client('cloudwatch')
    }
    
    for service_name, client in services.items():
        subsegment = xray_recorder.begin_subsegment(f'aws_service_{service_name}')
        try:
            start_time = time.time()
            
            if service_name == 's3':
                response = client.list_buckets()
            elif service_name == 'ec2':
                response = client.describe_regions()
            elif service_name == 'cloudwatch':
                response = client.list_metrics(MaxRecords=1)
            
            end_time = time.time()
            
            results[service_name] = {
                'latency_ms': (end_time - start_time) * 1000,
                'success': True
            }
            
            # Add metadata to X-Ray trace
            subsegment.put_metadata('response_time', end_time - start_time)
            subsegment.put_annotation('service', service_name)
            subsegment.put_annotation('success', True)
            
        except Exception as e:
            results[service_name] = {
                'error': str(e),
                'success': False
            }
            subsegment.add_exception(e)
        finally:
            xray_recorder.end_subsegment()
    
    return results

@app.route('/health')
@xray_recorder.capture('health_endpoint')
def health():
    """Health check endpoint with distributed tracing"""
    
    # Add user agent and IP to trace
    trace = xray_recorder.get_trace_entity()
    trace.put_annotation('user_agent', request.headers.get('User-Agent', 'Unknown'))
    trace.put_annotation('client_ip', request.remote_addr)
    
    # Perform network health checks
    network_results = check_network_health()
    aws_results = check_aws_services()
    
    # Calculate overall health
    all_checks = list(network_results.values()) + list(aws_results.values())
    success_count = sum(1 for check in all_checks if check.get('success', False))
    total_checks = len(all_checks)
    health_score = (success_count / total_checks) * 100 if total_checks > 0 else 0
    
    # Add health score to trace
    trace.put_metadata('health_score', health_score)
    trace.put_annotation('healthy', health_score >= 80)
    
    response_data = {
        'timestamp': time.time(),
        'health_score': health_score,
        'network_checks': network_results,
        'aws_service_checks': aws_results,
        'overall_status': 'healthy' if health_score >= 80 else 'unhealthy'
    }
    
    return jsonify(response_data)

@app.route('/trace-test')
@xray_recorder.capture('trace_test')
def trace_test():
    """Test endpoint for generating complex traces"""
    
    # Simulate database call
    with xray_recorder.in_subsegment('database_query'):
        time.sleep(0.1)  # Simulate DB latency
        xray_recorder.current_subsegment().put_metadata('query', 'SELECT * FROM users')
        xray_recorder.current_subsegment().put_annotation('table', 'users')
    
    # Simulate cache lookup
    with xray_recorder.in_subsegment('cache_lookup'):
        time.sleep(0.02)  # Simulate cache latency
        xray_recorder.current_subsegment().put_metadata('cache_key', 'user:123')
        xray_recorder.current_subsegment().put_annotation('cache_hit', True)
    
    # Simulate external API call
    with xray_recorder.in_subsegment('external_api_call'):
        try:
            response = requests.get('https://httpbin.org/delay/1', timeout=2)
            xray_recorder.current_subsegment().put_metadata('api_response', response.status_code)
            xray_recorder.current_subsegment().put_annotation('api_success', True)
        except Exception as e:
            xray_recorder.current_subsegment().add_exception(e)
            xray_recorder.current_subsegment().put_annotation('api_success', False)
    
    return jsonify({
        'message': 'Trace test completed',
        'timestamp': time.time()
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080, debug=False)
```

**Create X-Ray Daemon Configuration**:
```bash
# Install X-Ray daemon
curl https://s3.us-east-2.amazonaws.com/aws-xray-assets.us-east-2/xray-daemon/aws-xray-daemon-3.x.rpm -o xray-daemon.rpm
sudo yum install -y xray-daemon.rpm

# Create X-Ray daemon configuration
sudo tee /etc/amazon/xray/cfg.yaml > /dev/null << EOF
# AWS X-Ray daemon configuration
TotalBufferSizeKB: 0
Concurrency: 8
Region: "us-west-2"
Socket:
  UDPAddress: "0.0.0.0:2000"
  TCPAddress: "0.0.0.0:2000"
LocalMode: false
ResourceARN: ""
RoleARN: ""
NoVerifySSL: false
ProxyAddress: ""
DaemonAddress: "127.0.0.1:2000"
LogLevel: "info"
LogPath: "/var/log/xray/xray-daemon.log"
EOF

# Start X-Ray daemon
sudo systemctl start xray
sudo systemctl enable xray

# Install Python dependencies for the app
pip3 install flask aws-xray-sdk requests boto3

# Run the application
python3 xray-network-app.py
```

### Step 5: Automated Incident Response
**Objective**: Create automated incident response for network issues

**Create Incident Response Lambda Function**:
```python
# incident-response-lambda.py
import json
import boto3
import urllib3
from datetime import datetime, timedelta

def lambda_handler(event, context):
    """
    Automated incident response for VPC network issues
    """
    
    # Initialize AWS clients
    ec2 = boto3.client('ec2')
    cloudwatch = boto3.client('cloudwatch')
    sns = boto3.client('sns')
    
    # Parse the CloudWatch alarm
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm_name = message['AlarmName']
    alarm_description = message['AlarmDescription']
    new_state = message['NewStateValue']
    reason = message['NewStateReason']
    
    print(f"Processing alarm: {alarm_name}, State: {new_state}")
    
    incident_response = {
        'alarm_name': alarm_name,
        'timestamp': datetime.utcnow().isoformat(),
        'actions_taken': [],
        'recommendations': []
    }
    
    try:
        if 'High-Network-Latency' in alarm_name and new_state == 'ALARM':
            response = handle_high_latency_alarm(ec2, cloudwatch, incident_response)
        elif 'High-Packet-Loss' in alarm_name and new_state == 'ALARM':
            response = handle_packet_loss_alarm(ec2, cloudwatch, incident_response)
        elif 'High-Network-Errors' in alarm_name and new_state == 'ALARM':
            response = handle_network_errors_alarm(ec2, cloudwatch, incident_response)
        elif 'Security-Threat-Detected' in alarm_name and new_state == 'ALARM':
            response = handle_security_threat_alarm(ec2, cloudwatch, incident_response)
        else:
            incident_response['actions_taken'].append('No automated response configured for this alarm')
        
        # Send incident report
        send_incident_report(sns, incident_response)
        
        return {
            'statusCode': 200,
            'body': json.dumps(incident_response)
        }
        
    except Exception as e:
        print(f"Error in incident response: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def handle_high_latency_alarm(ec2, cloudwatch, incident_response):
    """Handle high network latency incidents"""
    
    # Get recent network metrics
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(minutes=30)
    
    # Check if issue is widespread or isolated
    latency_metrics = cloudwatch.get_metric_statistics(
        Namespace='VPC/NetworkLatency/8_8_8_8',
        MetricName='rtt_avg',
        Dimensions=[],
        StartTime=start_time,
        EndTime=end_time,
        Period=300,
        Statistics=['Average', 'Maximum']
    )
    
    if latency_metrics['Datapoints']:
        recent_latency = latency_metrics['Datapoints'][-1]['Average']
        max_latency = max([dp['Maximum'] for dp in latency_metrics['Datapoints']])
        
        incident_response['actions_taken'].append(f'Analyzed latency: current={recent_latency:.2f}ms, max={max_latency:.2f}ms')
        
        # Check for instance performance issues
        if max_latency > 500:  # Very high latency
            incident_response['recommendations'].extend([
                'Check EC2 instance CPU and memory utilization',
                'Review security group rules for bottlenecks',
                'Consider instance type upgrade or enhanced networking',
                'Check for network ACL misconfigurations'
            ])
        elif max_latency > 200:  # Moderate latency
            incident_response['recommendations'].extend([
                'Monitor for sustained high latency',
                'Check application performance metrics',
                'Review VPC Flow Logs for traffic patterns'
            ])
    
    return incident_response

def handle_packet_loss_alarm(ec2, cloudwatch, incident_response):
    """Handle packet loss incidents"""
    
    # Get VPC Flow Logs to analyze rejected traffic
    logs_client = boto3.client('logs')
    
    query = """
    fields @timestamp, srcaddr, dstaddr, action
    | filter action = "REJECT"
    | filter @timestamp > @timestamp - 15m
    | stats count() as reject_count by srcaddr
    | sort reject_count desc
    | limit 10
    """
    
    try:
        start_query_response = logs_client.start_query(
            logGroupName='/aws/vpc/flowlogs/vpc-flow-logs-monitoring',
            startTime=int((datetime.utcnow() - timedelta(minutes=15)).timestamp()),
            endTime=int(datetime.utcnow().timestamp()),
            queryString=query
        )
        
        # Wait for query completion
        import time
        time.sleep(5)
        
        results = logs_client.get_query_results(queryId=start_query_response['queryId'])
        
        if results['results']:
            top_rejected_ips = [
                {'ip': row[0]['value'], 'rejects': row[1]['value']} 
                for row in results['results'][:5]
            ]
            
            incident_response['actions_taken'].append(f'Found {len(top_rejected_ips)} IPs with high reject rates')
            
            # Auto-block suspicious IPs
            for ip_info in top_rejected_ips:
                if int(ip_info['rejects']) > 100:  # Threshold for auto-blocking
                    block_suspicious_ip(ec2, ip_info['ip'])
                    incident_response['actions_taken'].append(f"Blocked suspicious IP: {ip_info['ip']}")
    
    except Exception as e:
        incident_response['actions_taken'].append(f'Error analyzing flow logs: {str(e)}')
    
    incident_response['recommendations'].extend([
        'Review security group and NACL rules',
        'Check for DDoS attacks or port scanning',
        'Monitor application health and performance',
        'Consider implementing AWS WAF if not already in place'
    ])
    
    return incident_response

def handle_network_errors_alarm(ec2, cloudwatch, incident_response):
    """Handle network errors incidents"""
    
    # Check for instance-level network issues
    instances_response = ec2.describe_instances(
        Filters=[
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )
    
    problem_instances = []
    
    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            
            # Check CloudWatch metrics for this instance
            error_metrics = cloudwatch.get_metric_statistics(
                Namespace='VPC/NetworkBandwidth',
                MetricName='errors_in',
                Dimensions=[
                    {'Name': 'InstanceId', 'Value': instance_id}
                ],
                StartTime=datetime.utcnow() - timedelta(minutes=15),
                EndTime=datetime.utcnow(),
                Period=300,
                Statistics=['Sum']
            )
            
            if error_metrics['Datapoints']:
                total_errors = sum([dp['Sum'] for dp in error_metrics['Datapoints']])
                if total_errors > 50:  # Threshold for problematic instance
                    problem_instances.append({
                        'instance_id': instance_id,
                        'errors': total_errors
                    })
    
    if problem_instances:
        incident_response['actions_taken'].append(f'Identified {len(problem_instances)} instances with high error rates')
        
        for instance in problem_instances:
            incident_response['recommendations'].append(
                f"Check instance {instance['instance_id']} - {instance['errors']} errors detected"
            )
    
    incident_response['recommendations'].extend([
        'Check for hardware issues on affected instances',
        'Review network interface configurations',
        'Monitor for resource exhaustion (CPU, memory, bandwidth)',
        'Consider instance replacement if issues persist'
    ])
    
    return incident_response

def handle_security_threat_alarm(ec2, cloudwatch, incident_response):
    """Handle security threat incidents"""
    
    # This would integrate with GuardDuty findings
    guardduty = boto3.client('guardduty')
    
    try:
        # Get active detectors
        detectors = guardduty.list_detectors()
        
        if detectors['DetectorIds']:
            detector_id = detectors['DetectorIds'][0]
            
            # Get recent findings
            findings = guardduty.list_findings(
                DetectorId=detector_id,
                FindingCriteria={
                    'Criterion': {
                        'updatedAt': {
                            'GreaterThan': int((datetime.utcnow() - timedelta(hours=1)).timestamp() * 1000)
                        },
                        'severity': {
                            'GreaterThanOrEqual': 7.0  # High severity
                        }
                    }
                }
            )
            
            if findings['FindingIds']:
                finding_details = guardduty.get_findings(
                    DetectorId=detector_id,
                    FindingIds=findings['FindingIds'][:5]  # Limit to 5 most recent
                )
                
                for finding in finding_details['Findings']:
                    threat_type = finding['Type']
                    severity = finding['Severity']
                    
                    incident_response['actions_taken'].append(
                        f'GuardDuty finding: {threat_type} (Severity: {severity})'
                    )
                    
                    # Take automated action based on threat type
                    if 'Trojan' in threat_type or 'Malware' in threat_type:
                        # Isolate affected instance
                        if 'InstanceId' in finding.get('Service', {}).get('RemoteIpDetails', {}):
                            instance_id = finding['Service']['RemoteIpDetails']['InstanceId']
                            isolate_instance(ec2, instance_id)
                            incident_response['actions_taken'].append(f'Isolated instance: {instance_id}')
    
    except Exception as e:
        incident_response['actions_taken'].append(f'Error checking GuardDuty: {str(e)}')
    
    return incident_response

def block_suspicious_ip(ec2, ip_address):
    """Block suspicious IP address using NACLs"""
    try:
        # Find default NACL and add deny rule
        vpcs = ec2.describe_vpcs()
        
        for vpc in vpcs['Vpcs']:
            nacls = ec2.describe_network_acls(
                Filters=[
                    {'Name': 'vpc-id', 'Values': [vpc['VpcId']]},
                    {'Name': 'default', 'Values': ['true']}
                ]
            )
            
            if nacls['NetworkAcls']:
                nacl_id = nacls['NetworkAcls'][0]['NetworkAclId']
                
                # Add deny rule
                ec2.create_network_acl_entry(
                    NetworkAclId=nacl_id,
                    RuleNumber=10,  # High priority
                    Protocol='-1',
                    RuleAction='deny',
                    CidrBlock=f'{ip_address}/32'
                )
                
                print(f'Blocked IP {ip_address} in NACL {nacl_id}')
                break
    
    except Exception as e:
        print(f'Error blocking IP {ip_address}: {str(e)}')

def isolate_instance(ec2, instance_id):
    """Isolate instance by creating restrictive security group"""
    try:
        # Create isolation security group
        sg_response = ec2.create_security_group(
            GroupName=f'isolation-sg-{instance_id}',
            Description='Isolation security group for incident response',
            VpcId=get_instance_vpc(ec2, instance_id)
        )
        
        isolation_sg_id = sg_response['GroupId']
        
        # Apply isolation security group to instance
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=[isolation_sg_id]
        )
        
        print(f'Isolated instance {instance_id} with security group {isolation_sg_id}')
        
    except Exception as e:
        print(f'Error isolating instance {instance_id}: {str(e)}')

def get_instance_vpc(ec2, instance_id):
    """Get VPC ID for an instance"""
    response = ec2.describe_instances(InstanceIds=[instance_id])
    return response['Reservations'][0]['Instances'][0]['VpcId']

def send_incident_report(sns, incident_response):
    """Send incident report via SNS"""
    try:
        message = f"""
VPC Network Incident Response Report

Alarm: {incident_response['alarm_name']}
Timestamp: {incident_response['timestamp']}

Actions Taken:
{chr(10).join(['- ' + action for action in incident_response['actions_taken']])}

Recommendations:
{chr(10).join(['- ' + rec for rec in incident_response['recommendations']])}
        """
        
        sns.publish(
            TopicArn='arn:aws:sns:us-west-2:123456789012:network-incidents',
            Subject=f'VPC Network Incident: {incident_response["alarm_name"]}',
            Message=message
        )
        
    except Exception as e:
        print(f'Error sending incident report: {str(e)}')

```

**Deploy Incident Response Lambda**:
```bash
# Create deployment package
zip -r incident-response.zip incident-response-lambda.py

# Create Lambda function
aws lambda create-function \
  --function-name vpc-incident-response \
  --runtime python3.9 \
  --role arn:aws:iam::123456789012:role/lambda-vpc-incident-response-role \
  --handler incident-response-lambda.lambda_handler \
  --zip-file fileb://incident-response.zip \
  --timeout 300 \
  --description "Automated VPC network incident response"

# Create SNS topic subscription for CloudWatch alarms
aws sns subscribe \
  --topic-arn arn:aws:sns:us-west-2:123456789012:network-alerts \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-west-2:123456789012:function:vpc-incident-response

# Grant SNS permission to invoke Lambda
aws lambda add-permission \
  --function-name vpc-incident-response \
  --statement-id sns-trigger \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn arn:aws:sns:us-west-2:123456789012:network-alerts
```

## Best Practices Summary

### Monitoring Best Practices
1. **Comprehensive Coverage**: Monitor all network layers and components
2. **Real-time Alerting**: Set up immediate alerts for critical issues
3. **Baseline Establishment**: Establish performance baselines for anomaly detection
4. **Cost Optimization**: Balance monitoring coverage with cost considerations
5. **Retention Policies**: Implement appropriate data retention strategies

### Observability Best Practices
1. **Three Pillars**: Implement metrics, logs, and traces comprehensively
2. **Context Correlation**: Correlate data across different observability pillars
3. **Service Level Objectives**: Define and monitor SLOs for network services
4. **Automated Analysis**: Use automated tools for pattern recognition
5. **Documentation**: Maintain runbooks and troubleshooting guides

### Incident Response Best Practices
1. **Automation**: Automate response to common network issues
2. **Escalation Procedures**: Define clear escalation paths
3. **Communication**: Ensure stakeholders are notified promptly
4. **Post-Incident Analysis**: Conduct thorough post-mortems
5. **Continuous Improvement**: Update procedures based on lessons learned

## Troubleshooting Guide

### Common Monitoring Issues
1. **Missing Metrics**: Verify CloudWatch agent configuration and permissions
2. **High Costs**: Optimize metric collection frequency and retention
3. **False Alarms**: Tune alarm thresholds based on historical data
4. **Data Gaps**: Check for network connectivity and authentication issues
5. **Performance Impact**: Monitor overhead of monitoring tools

### Flow Logs Issues
1. **No Data**: Verify IAM permissions and destination configuration
2. **Incomplete Data**: Check for networking issues or resource limits
3. **High Volume**: Implement filtering and sampling strategies
4. **Cost Concerns**: Use S3 storage classes and lifecycle policies
5. **Analysis Delays**: Consider real-time processing with Kinesis

## Additional Resources
- [VPC Flow Logs User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [CloudWatch Network Monitoring](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/)
- [AWS X-Ray Developer Guide](https://docs.aws.amazon.com/xray/latest/devguide/)
- [GuardDuty User Guide](https://docs.aws.amazon.com/guardduty/latest/ug/)
- [Network Performance Monitoring Best Practices](https://aws.amazon.com/blogs/networking-and-content-delivery/)

## Exam Tips
- Understand different types of VPC monitoring and their use cases
- Know how to configure and analyze VPC Flow Logs
- Be familiar with CloudWatch metrics for network monitoring
- Understand distributed tracing concepts and X-Ray integration
- Know automated incident response patterns and best practices