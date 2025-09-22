# Topic 52: Encryption in Transit: TLS, VPN, MACsec

## Introduction

Encryption in transit protects data as it moves between systems, ensuring confidentiality and integrity during transmission. AWS provides multiple encryption technologies including TLS/SSL for web traffic, IPsec VPN for secure tunneling, and MACsec for Direct Connect links. This comprehensive guide covers implementation, configuration, and best practices for each encryption method.

## TLS/SSL Encryption Fundamentals

### TLS Protocol Overview

Transport Layer Security (TLS) operates at the session layer, providing:
- **Confidentiality**: Data encryption using symmetric algorithms
- **Integrity**: Message authentication codes (MAC)
- **Authentication**: Certificate-based identity verification
- **Non-repudiation**: Digital signatures

```
TLS Handshake Process:
Client → Server: ClientHello (cipher suites, random)
Server → Client: ServerHello (selected cipher, random, certificate)
Client → Server: Certificate verification, pre-master secret
Both: Generate master secret and session keys
Client/Server: Finished messages (encrypted)
```

### TLS Implementation in AWS Services

#### Application Load Balancer TLS Configuration

```yaml
# ALB with TLS termination
ApplicationLoadBalancer:
  Type: AWS::ElasticLoadBalancingV2::LoadBalancer
  Properties:
    Name: secure-alb
    Type: application
    Scheme: internet-facing
    SecurityGroups:
      - !Ref ALBSecurityGroup
    Subnets:
      - !Ref PublicSubnet1
      - !Ref PublicSubnet2

# HTTPS Listener with modern TLS
HTTPSListener:
  Type: AWS::ElasticLoadBalancingV2::Listener
  Properties:
    DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref TargetGroup
    LoadBalancerArn: !Ref ApplicationLoadBalancer
    Port: 443
    Protocol: HTTPS
    SslPolicy: ELBSecurityPolicy-TLS-1-2-2019-07
    Certificates:
      - CertificateArn: !Ref SSLCertificate

# Security policy configuration
SecureListenerPolicy:
  Type: AWS::ElasticLoadBalancingV2::Listener
  Properties:
    LoadBalancerArn: !Ref ApplicationLoadBalancer
    Port: 443
    Protocol: HTTPS
    SslPolicy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
    Certificates:
      - CertificateArn: !Ref SSLCertificate
    DefaultActions:
      - Type: redirect
        RedirectConfig:
          Protocol: HTTPS
          Port: 443
          StatusCode: HTTP_301
```

#### CloudFront TLS Configuration

```yaml
CloudFrontDistribution:
  Type: AWS::CloudFront::Distribution
  Properties:
    DistributionConfig:
      Aliases:
        - api.example.com
        - www.example.com
      ViewerCertificate:
        AcmCertificateArn: !Ref CloudFrontCertificate
        SslSupportMethod: sni-only
        MinimumProtocolVersion: TLSv1.2_2019
      DefaultCacheBehavior:
        ViewerProtocolPolicy: redirect-to-https
        AllowedMethods:
          - DELETE
          - GET
          - HEAD
          - OPTIONS
          - PATCH
          - POST
          - PUT
        CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
        OriginRequestPolicyId: acba4595-bd28-49b8-b9fe-13317c0390fa
      Origins:
        - Id: ALBOrigin
          DomainName: !GetAtt ApplicationLoadBalancer.DNSName
          CustomOriginConfig:
            HTTPPort: 80
            HTTPSPort: 443
            OriginProtocolPolicy: https-only
            OriginSSLProtocols:
              - TLSv1.2
```

### Certificate Management with ACM

```python
import boto3
import json
from datetime import datetime, timedelta

class CertificateManager:
    def __init__(self):
        self.acm = boto3.client('acm')
        self.route53 = boto3.client('route53')
    
    def request_certificate(self, domain_name, san_domains=None):
        """Request SSL certificate with DNS validation"""
        
        subject_alternative_names = san_domains or []
        
        response = self.acm.request_certificate(
            DomainName=domain_name,
            SubjectAlternativeNames=subject_alternative_names,
            ValidationMethod='DNS',
            Options={
                'CertificateTransparencyLoggingPreference': 'ENABLED'
            },
            Tags=[
                {
                    'Key': 'Environment',
                    'Value': 'production'
                },
                {
                    'Key': 'AutoRenewal',
                    'Value': 'enabled'
                }
            ]
        )
        
        return response['CertificateArn']
    
    def setup_dns_validation(self, certificate_arn, hosted_zone_id):
        """Setup DNS validation records"""
        
        # Get certificate details
        cert_details = self.acm.describe_certificate(
            CertificateArn=certificate_arn
        )
        
        # Create DNS validation records
        for validation in cert_details['Certificate']['DomainValidationOptions']:
            if 'ResourceRecord' in validation:
                record = validation['ResourceRecord']
                
                self.route53.change_resource_record_sets(
                    HostedZoneId=hosted_zone_id,
                    ChangeBatch={
                        'Changes': [{
                            'Action': 'CREATE',
                            'ResourceRecordSet': {
                                'Name': record['Name'],
                                'Type': record['Type'],
                                'TTL': 300,
                                'ResourceRecords': [{
                                    'Value': record['Value']
                                }]
                            }
                        }]
                    }
                )
    
    def setup_certificate_monitoring(self, certificate_arn):
        """Setup CloudWatch monitoring for certificate expiration"""
        
        cloudwatch = boto3.client('cloudwatch')
        
        # Create alarm for certificate expiration
        cloudwatch.put_metric_alarm(
            AlarmName=f'Certificate-Expiration-{certificate_arn.split("/")[-1]}',
            ComparisonOperator='LessThanThreshold',
            EvaluationPeriods=1,
            MetricName='DaysToExpiry',
            Namespace='AWS/CertificateManager',
            Period=86400,  # Daily check
            Statistic='Minimum',
            Threshold=30.0,  # Alert 30 days before expiration
            ActionsEnabled=True,
            AlarmActions=[
                'arn:aws:sns:us-east-1:123456789012:certificate-alerts'
            ],
            AlarmDescription='Certificate expiring soon',
            Dimensions=[
                {
                    'Name': 'CertificateArn',
                    'Value': certificate_arn
                }
            ]
        )
```

### Advanced TLS Configuration

```python
import ssl
import socket
import OpenSSL
from cryptography import x509
from cryptography.hazmat.backends import default_backend

class TLSValidator:
    def __init__(self):
        self.supported_protocols = ['TLSv1.2', 'TLSv1.3']
        self.secure_ciphers = [
            'ECDHE-RSA-AES256-GCM-SHA384',
            'ECDHE-RSA-AES128-GCM-SHA256',
            'ECDHE-RSA-CHACHA20-POLY1305',
            'ECDHE-ECDSA-AES256-GCM-SHA384',
            'ECDHE-ECDSA-AES128-GCM-SHA256',
            'ECDHE-ECDSA-CHACHA20-POLY1305'
        ]
    
    def validate_tls_configuration(self, hostname, port=443):
        """Validate TLS configuration of a service"""
        
        context = ssl.create_default_context()
        context.check_hostname = True
        context.verify_mode = ssl.CERT_REQUIRED
        
        try:
            with socket.create_connection((hostname, port), timeout=10) as sock:
                with context.wrap_socket(sock, server_hostname=hostname) as ssock:
                    cert = ssock.getpeercert()
                    cipher = ssock.cipher()
                    version = ssock.version()
                    
                    return {
                        'certificate': self._analyze_certificate(cert),
                        'cipher_suite': cipher,
                        'protocol_version': version,
                        'security_score': self._calculate_security_score(cipher, version)
                    }
        except ssl.SSLError as e:
            return {'error': f'SSL Error: {str(e)}'}
        except Exception as e:
            return {'error': f'Connection Error: {str(e)}'}
    
    def _analyze_certificate(self, cert):
        """Analyze certificate details"""
        return {
            'subject': dict(x[0] for x in cert['subject']),
            'issuer': dict(x[0] for x in cert['issuer']),
            'not_before': cert['notBefore'],
            'not_after': cert['notAfter'],
            'san': cert.get('subjectAltName', []),
            'serial_number': cert['serialNumber']
        }
    
    def _calculate_security_score(self, cipher, version):
        """Calculate security score based on cipher and protocol"""
        score = 0
        
        # Protocol version scoring
        if version == 'TLSv1.3':
            score += 40
        elif version == 'TLSv1.2':
            score += 30
        else:
            score += 10
        
        # Cipher suite scoring
        cipher_name = cipher[0] if cipher else ''
        if 'ECDHE' in cipher_name:
            score += 20  # Perfect Forward Secrecy
        if 'AES256' in cipher_name:
            score += 15
        elif 'AES128' in cipher_name:
            score += 10
        if 'GCM' in cipher_name or 'CHACHA20' in cipher_name:
            score += 15  # AEAD ciphers
        
        return min(score, 100)

# Usage example
validator = TLSValidator()
result = validator.validate_tls_configuration('api.example.com')
print(json.dumps(result, indent=2))
```

## VPN Encryption (IPsec)

### Site-to-Site VPN Configuration

```yaml
# Customer Gateway
CustomerGateway:
  Type: AWS::EC2::CustomerGateway
  Properties:
    Type: ipsec.1
    BgpAsn: 65000
    IpAddress: 203.0.113.12
    Tags:
      - Key: Name
        Value: MainOffice-CGW

# VPN Gateway
VPNGateway:
  Type: AWS::EC2::VPNGateway
  Properties:
    Type: ipsec.1
    AmazonSideAsn: 64512
    Tags:
      - Key: Name
        Value: Production-VGW

# VPN Connection with custom encryption
VPNConnection:
  Type: AWS::EC2::VPNConnection
  Properties:
    Type: ipsec.1
    StaticRoutesOnly: false
    CustomerGatewayId: !Ref CustomerGateway
    VpnGatewayId: !Ref VPNGateway
    VpnTunnelOptionsSpecifications:
      - PreSharedKey: !Ref PreSharedKey1
        TunnelInsideCidr: 169.254.10.0/30
        IKEVersions:
          - ikev2
        Phase1EncryptionAlgorithms:
          - AES256
        Phase1IntegrityAlgorithms:
          - SHA256
        Phase1DHGroupNumbers:
          - 14
        Phase1LifetimeSeconds: 28800
        Phase2EncryptionAlgorithms:
          - AES256
        Phase2IntegrityAlgorithms:
          - SHA256
        Phase2DHGroupNumbers:
          - 14
        Phase2LifetimeSeconds: 3600
      - PreSharedKey: !Ref PreSharedKey2
        TunnelInsideCidr: 169.254.11.0/30
        IKEVersions:
          - ikev2
        Phase1EncryptionAlgorithms:
          - AES256
        Phase1IntegrityAlgorithms:
          - SHA256
        Phase1DHGroupNumbers:
          - 14
        Phase1LifetimeSeconds: 28800
        Phase2EncryptionAlgorithms:
          - AES256
        Phase2IntegrityAlgorithms:
          - SHA256
        Phase2DHGroupNumbers:
          - 14
        Phase2LifetimeSeconds: 3600
    Tags:
      - Key: Name
        Value: Production-VPN
```

### Client VPN Endpoint Configuration

```yaml
# Client VPN Endpoint
ClientVPNEndpoint:
  Type: AWS::EC2::ClientVpnEndpoint
  Properties:
    Description: Secure remote access VPN
    ClientCidrBlock: 10.100.0.0/16
    ServerCertificateArn: !Ref ServerCertificate
    AuthenticationOptions:
      - Type: certificate-authentication
        MutualAuthentication:
          ClientRootCertificateChainArn: !Ref ClientCertificate
      - Type: federated-authentication
        FederatedAuthentication:
          SAMLProviderArn: !Ref SAMLProvider
    ConnectionLogOptions:
      Enabled: true
      CloudwatchLogGroup: !Ref VPNLogGroup
      CloudwatchLogStream: !Ref VPNLogStream
    DnsServers:
      - 10.0.0.2
      - 10.0.1.2
    TransportProtocol: udp
    VpnPort: 443
    SecurityGroupIds:
      - !Ref ClientVPNSecurityGroup
    VpcId: !Ref VPC
    TagSpecifications:
      - ResourceType: client-vpn-endpoint
        Tags:
          - Key: Name
            Value: Corporate-ClientVPN

# Network Association
ClientVPNNetworkAssociation:
  Type: AWS::EC2::ClientVpnTargetNetworkAssociation
  Properties:
    ClientVpnEndpointId: !Ref ClientVPNEndpoint
    SubnetId: !Ref PrivateSubnet1

# Authorization Rules
ClientVPNAuthorizationRule:
  Type: AWS::EC2::ClientVpnAuthorizationRule
  Properties:
    ClientVpnEndpointId: !Ref ClientVPNEndpoint
    TargetNetworkCidr: 10.0.0.0/16
    AuthorizeAllGroups: true
    Description: Allow access to VPC
```

### IPsec Tunnel Monitoring

```python
import boto3
import json
from datetime import datetime, timedelta

class VPNMonitor:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def get_vpn_status(self, vpn_connection_id):
        """Get detailed VPN connection status"""
        
        response = self.ec2.describe_vpn_connections(
            VpnConnectionIds=[vpn_connection_id]
        )
        
        vpn_conn = response['VpnConnections'][0]
        
        tunnel_status = []
        for tunnel in vpn_conn['VgwTelemetry']:
            tunnel_info = {
                'outside_ip': tunnel['OutsideIpAddress'],
                'status': tunnel['Status'],
                'status_message': tunnel['StatusMessage'],
                'accepted_route_count': tunnel['AcceptedRouteCount'],
                'last_status_change': tunnel['LastStatusChange']
            }
            tunnel_status.append(tunnel_info)
        
        return {
            'vpn_id': vpn_connection_id,
            'state': vpn_conn['State'],
            'type': vpn_conn['Type'],
            'tunnels': tunnel_status,
            'routes': vpn_conn.get('Routes', [])
        }
    
    def create_vpn_alarms(self, vpn_connection_id):
        """Create CloudWatch alarms for VPN monitoring"""
        
        alarms = [
            {
                'AlarmName': f'VPN-TunnelDown-{vpn_connection_id}',
                'MetricName': 'TunnelState',
                'Threshold': 0,
                'ComparisonOperator': 'LessThanOrEqualToThreshold',
                'Statistic': 'Maximum'
            },
            {
                'AlarmName': f'VPN-PacketDropCount-{vpn_connection_id}',
                'MetricName': 'PacketDropCount',
                'Threshold': 100,
                'ComparisonOperator': 'GreaterThanThreshold',
                'Statistic': 'Sum'
            }
        ]
        
        for alarm in alarms:
            self.cloudwatch.put_metric_alarm(
                AlarmName=alarm['AlarmName'],
                ComparisonOperator=alarm['ComparisonOperator'],
                EvaluationPeriods=2,
                MetricName=alarm['MetricName'],
                Namespace='AWS/VPN',
                Period=300,
                Statistic=alarm['Statistic'],
                Threshold=alarm['Threshold'],
                ActionsEnabled=True,
                AlarmActions=[
                    'arn:aws:sns:us-east-1:123456789012:vpn-alerts'
                ],
                AlarmDescription=f'VPN monitoring for {vpn_connection_id}',
                Dimensions=[
                    {
                        'Name': 'VpnId',
                        'Value': vpn_connection_id
                    }
                ]
            )
    
    def analyze_vpn_performance(self, vpn_connection_id, hours=24):
        """Analyze VPN performance metrics"""
        
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)
        
        metrics = [
            'TunnelState',
            'PacketDropCount',
            'TunnelIpAddress'
        ]
        
        results = {}
        for metric in metrics:
            response = self.cloudwatch.get_metric_statistics(
                Namespace='AWS/VPN',
                MetricName=metric,
                Dimensions=[
                    {
                        'Name': 'VpnId',
                        'Value': vpn_connection_id
                    }
                ],
                StartTime=start_time,
                EndTime=end_time,
                Period=3600,  # 1 hour periods
                Statistics=['Average', 'Maximum', 'Minimum']
            )
            results[metric] = response['Datapoints']
        
        return results
```

## Direct Connect MACsec

### MACsec Overview

Media Access Control Security (MACsec) provides:
- **Layer 2 encryption** between customer premises and AWS
- **Hop-by-hop security** at the data link layer  
- **Low latency** encryption with hardware acceleration
- **Standards compliance** with IEEE 802.1AE

### MACsec Configuration

```python
import boto3
import json

class DirectConnectMACsec:
    def __init__(self):
        self.dx = boto3.client('directconnect')
    
    def create_macsec_connection(self, location, bandwidth, connection_name):
        """Create Direct Connect connection with MACsec"""
        
        response = self.dx.create_connection(
            location=location,
            bandwidth=bandwidth,
            connectionName=connection_name,
            tags=[
                {
                    'key': 'MACsec',
                    'value': 'enabled'
                },
                {
                    'key': 'Environment',
                    'value': 'production'
                }
            ],
            requestMACSec=True
        )
        
        return response['connectionId']
    
    def associate_macsec_key(self, connection_id, secret_arn):
        """Associate MACsec security key with connection"""
        
        response = self.dx.associate_mac_sec_key(
            connectionId=connection_id,
            secretARN=secret_arn
        )
        
        return response
    
    def create_macsec_virtual_interface(self, connection_id, vlan, bgp_asn, 
                                     customer_address, amazon_address, address_family='ipv4'):
        """Create virtual interface with MACsec encryption"""
        
        response = self.dx.create_transit_virtual_interface(
            connectionId=connection_id,
            newTransitVirtualInterface={
                'vlan': vlan,
                'asn': bgp_asn,
                'customerAddress': customer_address,
                'amazonAddress': amazon_address,
                'addressFamily': address_family,
                'directConnectGatewayId': 'dx-gateway-id',
                'tags': [
                    {
                        'key': 'MACsec',
                        'value': 'enabled'
                    }
                ]
            }
        )
        
        return response['virtualInterface']['virtualInterfaceId']
    
    def monitor_macsec_status(self, connection_id):
        """Monitor MACsec connection status"""
        
        response = self.dx.describe_connections(
            connectionId=connection_id
        )
        
        connection = response['connections'][0]
        
        macsec_status = {
            'connection_id': connection_id,
            'macsec_capable': connection.get('macSecCapable', False),
            'encryption_mode': connection.get('encryptionMode', 'no_encrypt'),
            'mac_sec_keys': connection.get('macSecKeys', [])
        }
        
        return macsec_status

# MACsec key management with AWS Secrets Manager
def create_macsec_secret():
    """Create and manage MACsec keys in Secrets Manager"""
    
    secrets_client = boto3.client('secretsmanager')
    
    # Generate MACsec key
    secret_value = {
        'ConnectivityAssociationName': 'dx-macsec-key',
        'ConnectivityAssociationNameId': 'cak-name-id',
        'ConnectivityAssociationKey': 'your-256-bit-hex-key',
        'ConnectivityAssociationKeyId': 'ckn-key-id'
    }
    
    response = secrets_client.create_secret(
        Name='DirectConnect/MACsec/PrimaryKey',
        Description='MACsec key for Direct Connect encryption',
        SecretString=json.dumps(secret_value),
        Tags=[
            {
                'Key': 'Purpose',
                'Value': 'DirectConnect-MACsec'
            }
        ]
    )
    
    return response['ARN']
```

### MACsec Monitoring and Management

```yaml
# CloudWatch monitoring for Direct Connect
DirectConnectMonitoring:
  Type: AWS::CloudWatch::Dashboard
  Properties:
    DashboardName: DirectConnect-MACsec-Monitoring
    DashboardBody: !Sub |
      {
        "widgets": [
          {
            "type": "metric",
            "properties": {
              "metrics": [
                ["AWS/DX", "ConnectionState", "ConnectionId", "${DirectConnectConnection}"],
                [".", "ConnectionBpsEgress", ".", "."],
                [".", "ConnectionBpsIngress", ".", "."],
                [".", "ConnectionPpsEgress", ".", "."],
                [".", "ConnectionPpsIngress", ".", "."]
              ],
              "period": 300,
              "stat": "Average",
              "region": "${AWS::Region}",
              "title": "Direct Connect Metrics"
            }
          }
        ]
      }

# Alarm for connection state
ConnectionStateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: DirectConnect-ConnectionDown
    AlarmDescription: Direct Connect connection is down
    MetricName: ConnectionState
    Namespace: AWS/DX
    Statistic: Maximum
    Period: 60
    EvaluationPeriods: 2
    Threshold: 0
    ComparisonOperator: LessThanOrEqualToThreshold
    Dimensions:
      - Name: ConnectionId
        Value: !Ref DirectConnectConnection
    AlarmActions:
      - !Ref SNSTopicArn
```

## Certificate Management Best Practices

### Automated Certificate Lifecycle

```python
import boto3
import json
from datetime import datetime, timedelta

class CertificateLifecycleManager:
    def __init__(self):
        self.acm = boto3.client('acm')
        self.ssm = boto3.client('ssm')
        self.lambda_client = boto3.client('lambda')
    
    def setup_auto_renewal(self, certificate_arn, domains):
        """Setup automated certificate renewal"""
        
        # Create SSM parameter for tracking
        self.ssm.put_parameter(
            Name=f'/certificates/{certificate_arn.split("/")[-1]}',
            Value=json.dumps({
                'certificate_arn': certificate_arn,
                'domains': domains,
                'auto_renewal': True,
                'notification_days': [30, 7, 1]
            }),
            Type='String',
            Tags=[
                {
                    'Key': 'Purpose',
                    'Value': 'CertificateManagement'
                }
            ]
        )
        
        # Create CloudWatch event rule for daily checks
        events = boto3.client('events')
        events.put_rule(
            Name='CertificateExpiryCheck',
            ScheduleExpression='rate(1 day)',
            State='ENABLED',
            Description='Daily certificate expiry check'
        )
        
        # Add Lambda function as target
        events.put_targets(
            Rule='CertificateExpiryCheck',
            Targets=[
                {
                    'Id': '1',
                    'Arn': 'arn:aws:lambda:us-east-1:123456789012:function:certificate-manager',
                    'Input': json.dumps({
                        'action': 'check_expiry',
                        'certificate_arn': certificate_arn
                    })
                }
            ]
        )
    
    def check_certificate_expiry(self, certificate_arn):
        """Check certificate expiry and trigger renewal if needed"""
        
        try:
            response = self.acm.describe_certificate(
                CertificateArn=certificate_arn
            )
            
            cert = response['Certificate']
            not_after = cert['NotAfter']
            days_to_expiry = (not_after - datetime.now(not_after.tzinfo)).days
            
            if days_to_expiry <= 30:
                return self.initiate_renewal(certificate_arn, days_to_expiry)
            
            return {
                'status': 'healthy',
                'days_to_expiry': days_to_expiry
            }
            
        except Exception as e:
            return {
                'status': 'error',
                'error': str(e)
            }
    
    def initiate_renewal(self, certificate_arn, days_to_expiry):
        """Initiate certificate renewal process"""
        
        # Get certificate details
        response = self.acm.describe_certificate(
            CertificateArn=certificate_arn
        )
        
        cert = response['Certificate']
        domain_name = cert['DomainName']
        san_domains = [name for name in cert.get('SubjectAlternativeNames', []) 
                      if name != domain_name]
        
        # Request new certificate
        new_cert_response = self.acm.request_certificate(
            DomainName=domain_name,
            SubjectAlternativeNames=san_domains,
            ValidationMethod='DNS'
        )
        
        new_certificate_arn = new_cert_response['CertificateArn']
        
        # Schedule certificate replacement
        self.schedule_certificate_replacement(
            old_certificate_arn=certificate_arn,
            new_certificate_arn=new_certificate_arn,
            days_to_expiry=days_to_expiry
        )
        
        return {
            'status': 'renewal_initiated',
            'new_certificate_arn': new_certificate_arn,
            'days_to_expiry': days_to_expiry
        }
    
    def schedule_certificate_replacement(self, old_certificate_arn, 
                                       new_certificate_arn, days_to_expiry):
        """Schedule certificate replacement across services"""
        
        # Find services using the certificate
        services = self.find_certificate_usage(old_certificate_arn)
        
        replacement_plan = {
            'old_certificate': old_certificate_arn,
            'new_certificate': new_certificate_arn,
            'services_to_update': services,
            'execution_date': datetime.now() + timedelta(days=min(days_to_expiry-1, 7))
        }
        
        # Store replacement plan
        self.ssm.put_parameter(
            Name=f'/certificate-replacement/{new_certificate_arn.split("/")[-1]}',
            Value=json.dumps(replacement_plan, default=str),
            Type='String'
        )
        
        return replacement_plan
    
    def find_certificate_usage(self, certificate_arn):
        """Find all AWS services using a certificate"""
        
        services_using_cert = []
        
        # Check ELB
        elb = boto3.client('elbv2')
        try:
            listeners = elb.describe_listeners()
            for listener in listeners['Listeners']:
                for cert in listener.get('Certificates', []):
                    if cert['CertificateArn'] == certificate_arn:
                        services_using_cert.append({
                            'service': 'ELB',
                            'resource_arn': listener['ListenerArn'],
                            'load_balancer_arn': listener['LoadBalancerArn']
                        })
        except Exception as e:
            print(f"Error checking ELB: {e}")
        
        # Check CloudFront
        cloudfront = boto3.client('cloudfront')
        try:
            distributions = cloudfront.list_distributions()
            for dist in distributions['DistributionList']['Items']:
                viewer_cert = dist['ViewerCertificate']
                if viewer_cert.get('ACMCertificateArn') == certificate_arn:
                    services_using_cert.append({
                        'service': 'CloudFront',
                        'distribution_id': dist['Id'],
                        'domain_name': dist['DomainName']
                    })
        except Exception as e:
            print(f"Error checking CloudFront: {e}")
        
        # Check API Gateway
        apigateway = boto3.client('apigateway')
        try:
            domain_names = apigateway.get_domain_names()
            for domain in domain_names['items']:
                if domain.get('certificateArn') == certificate_arn:
                    services_using_cert.append({
                        'service': 'API Gateway',
                        'domain_name': domain['domainName'],
                        'certificate_arn': domain['certificateArn']
                    })
        except Exception as e:
            print(f"Error checking API Gateway: {e}")
        
        return services_using_cert
```

### Certificate Security Hardening

```python
class CertificateSecurityManager:
    def __init__(self):
        self.acm = boto3.client('acm')
        self.kms = boto3.client('kms')
    
    def validate_certificate_security(self, certificate_arn):
        """Validate certificate security configuration"""
        
        response = self.acm.describe_certificate(
            CertificateArn=certificate_arn
        )
        
        cert = response['Certificate']
        security_issues = []
        
        # Check key algorithm
        key_algorithm = cert.get('KeyAlgorithm', '')
        if key_algorithm not in ['RSA-2048', 'EC-256', 'EC-384']:
            security_issues.append(f'Weak key algorithm: {key_algorithm}')
        
        # Check signature algorithm
        signature_algorithm = cert.get('SignatureAlgorithm', '')
        if 'SHA1' in signature_algorithm:
            security_issues.append(f'Weak signature algorithm: {signature_algorithm}')
        
        # Check certificate transparency
        options = cert.get('Options', {})
        if not options.get('CertificateTransparencyLoggingPreference') == 'ENABLED':
            security_issues.append('Certificate transparency logging disabled')
        
        # Check domain validation
        domain_validation_options = cert.get('DomainValidationOptions', [])
        for domain_option in domain_validation_options:
            if domain_option.get('ValidationMethod') != 'DNS':
                security_issues.append(f'Insecure validation method for {domain_option.get("DomainName")}')
        
        return {
            'certificate_arn': certificate_arn,
            'security_score': self._calculate_security_score(security_issues),
            'issues': security_issues,
            'recommendations': self._generate_recommendations(security_issues)
        }
    
    def _calculate_security_score(self, issues):
        """Calculate security score based on issues found"""
        base_score = 100
        deductions = {
            'weak key algorithm': 30,
            'weak signature algorithm': 25,
            'certificate transparency': 10,
            'insecure validation': 15
        }
        
        for issue in issues:
            for key, deduction in deductions.items():
                if key in issue.lower():
                    base_score -= deduction
                    break
        
        return max(base_score, 0)
    
    def _generate_recommendations(self, issues):
        """Generate security recommendations"""
        recommendations = []
        
        for issue in issues:
            if 'weak key algorithm' in issue.lower():
                recommendations.append('Upgrade to RSA-2048 or EC-256/384 key algorithm')
            elif 'weak signature algorithm' in issue.lower():
                recommendations.append('Use SHA-256 or SHA-384 signature algorithm')
            elif 'certificate transparency' in issue.lower():
                recommendations.append('Enable certificate transparency logging')
            elif 'insecure validation' in issue.lower():
                recommendations.append('Use DNS validation method')
        
        return recommendations
```

## Encryption Best Practices

### Comprehensive Encryption Strategy

```yaml
# Complete encryption-in-transit stack
EncryptionStack:
  Description: Comprehensive encryption-in-transit implementation
  
  Parameters:
    DomainName:
      Type: String
      Description: Primary domain name
    
  Resources:
    # TLS Certificate
    TLSCertificate:
      Type: AWS::CertificateManager::Certificate
      Properties:
        DomainName: !Ref DomainName
        SubjectAlternativeNames:
          - !Sub 'api.${DomainName}'
          - !Sub 'www.${DomainName}'
        ValidationMethod: DNS
        DomainValidationOptions:
          - DomainName: !Ref DomainName
            HostedZoneId: !Ref HostedZone
        
    # Application Load Balancer with TLS
    SecureALB:
      Type: AWS::ElasticLoadBalancingV2::LoadBalancer
      Properties:
        Type: application
        Scheme: internet-facing
        SecurityGroups:
          - !Ref ALBSecurityGroup
        Subnets:
          - !Ref PublicSubnet1
          - !Ref PublicSubnet2
        LoadBalancerAttributes:
          - Key: access_logs.s3.enabled
            Value: true
          - Key: access_logs.s3.bucket
            Value: !Ref AccessLogsBucket
    
    # HTTPS Listener with strong TLS policy
    HTTPSListener:
      Type: AWS::ElasticLoadBalancingV2::Listener
      Properties:
        DefaultActions:
          - Type: forward
            TargetGroupArn: !Ref TargetGroup
        LoadBalancerArn: !Ref SecureALB
        Port: 443
        Protocol: HTTPS
        SslPolicy: ELBSecurityPolicy-TLS-1-2-Ext-2018-06
        Certificates:
          - CertificateArn: !Ref TLSCertificate
    
    # Redirect HTTP to HTTPS
    HTTPListener:
      Type: AWS::ElasticLoadBalancingV2::Listener
      Properties:
        DefaultActions:
          - Type: redirect
            RedirectConfig:
              Protocol: HTTPS
              Port: 443
              StatusCode: HTTP_301
        LoadBalancerArn: !Ref SecureALB
        Port: 80
        Protocol: HTTP
    
    # CloudFront with TLS
    CloudFrontDistribution:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          Origins:
            - Id: ALBOrigin
              DomainName: !GetAtt SecureALB.DNSName
              CustomOriginConfig:
                HTTPPort: 80
                HTTPSPort: 443
                OriginProtocolPolicy: https-only
                OriginSSLProtocols:
                  - TLSv1.2
          DefaultCacheBehavior:
            TargetOriginId: ALBOrigin
            ViewerProtocolPolicy: redirect-to-https
            AllowedMethods:
              - DELETE
              - GET
              - HEAD
              - OPTIONS
              - PATCH
              - POST
              - PUT
            CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
          ViewerCertificate:
            AcmCertificateArn: !Ref TLSCertificate
            SslSupportMethod: sni-only
            MinimumProtocolVersion: TLSv1.2_2019
          Enabled: true
```

### Performance Optimization

```python
class EncryptionPerformanceOptimizer:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.elbv2 = boto3.client('elbv2')
    
    def optimize_tls_performance(self, load_balancer_arn):
        """Optimize TLS performance for load balancer"""
        
        # Get current listeners
        listeners = self.elbv2.describe_listeners(
            LoadBalancerArn=load_balancer_arn
        )
        
        optimizations = []
        
        for listener in listeners['Listeners']:
            if listener['Protocol'] == 'HTTPS':
                # Check SSL policy
                current_policy = listener.get('SslPolicy', '')
                
                recommended_policies = [
                    'ELBSecurityPolicy-TLS-1-2-2019-07',  # Modern TLS 1.2
                    'ELBSecurityPolicy-TLS-1-3-2019-07'   # TLS 1.3 support
                ]
                
                if current_policy not in recommended_policies:
                    optimizations.append({
                        'type': 'ssl_policy_upgrade',
                        'listener_arn': listener['ListenerArn'],
                        'current_policy': current_policy,
                        'recommended_policy': recommended_policies[1],
                        'benefits': [
                            'Reduced handshake latency',
                            'Forward secrecy',
                            'Modern cipher suites'
                        ]
                    })
        
        return optimizations
    
    def implement_ssl_optimization(self, listener_arn, ssl_policy):
        """Implement SSL policy optimization"""
        
        response = self.elbv2.modify_listener(
            ListenerArn=listener_arn,
            SslPolicy=ssl_policy
        )
        
        return response
    
    def configure_connection_draining(self, target_group_arn):
        """Configure connection draining for graceful updates"""
        
        self.elbv2.modify_target_group_attributes(
            TargetGroupArn=target_group_arn,
            Attributes=[
                {
                    'Key': 'deregistration_delay.timeout_seconds',
                    'Value': '30'  # Reduced for faster updates
                },
                {
                    'Key': 'connection_termination.deregistration_delay_timeout_seconds',
                    'Value': '30'
                }
            ]
        )
```

This comprehensive guide provides detailed implementation strategies for encryption in transit across AWS services, ensuring data protection throughout its journey between systems while maintaining performance and operational efficiency.