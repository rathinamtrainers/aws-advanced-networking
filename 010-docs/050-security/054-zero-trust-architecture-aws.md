# Topic 54: Zero Trust Architecture on AWS

## Introduction

Zero Trust is a security framework that operates on the principle "never trust, always verify." Unlike traditional perimeter-based security models, Zero Trust assumes no implicit trust and continuously validates every transaction. This topic covers implementing Zero Trust architecture on AWS, including identity verification, device trust, network micro-segmentation, and continuous monitoring strategies.

## Zero Trust Principles and AWS Implementation

### Core Zero Trust Principles

1. **Verify Explicitly**: Always authenticate and authorize based on all available data points
2. **Use Least Privilege Access**: Limit user access with Just-In-Time and Just-Enough-Access
3. **Assume Breach**: Minimize blast radius and segment access verification

### AWS Zero Trust Architecture Components

```
Zero Trust Architecture on AWS:

Identity Layer:
├── AWS IAM Identity Center (SSO)
├── AWS Cognito
├── External Identity Providers
└── Multi-Factor Authentication

Device Layer:
├── AWS Systems Manager
├── Device Trust Validation
├── Certificate-based Authentication
└── Endpoint Detection & Response

Network Layer:
├── VPC Security Groups
├── Network ACLs
├── AWS Network Firewall
├── VPC Endpoints
└── Micro-segmentation

Data Layer:
├── Encryption at Rest/Transit
├── AWS Key Management Service
├── Data Classification
└── Access Logging

Application Layer:
├── API Gateway Authentication
├── Application Load Balancer
├── AWS WAF
└── Container Security

Monitoring Layer:
├── AWS CloudTrail
├── Amazon GuardDuty
├── AWS Security Hub
├── AWS Config
└── Custom Monitoring
```

## Identity Verification and Authentication

### Multi-Factor Authentication Implementation

```python
import boto3
import json
from datetime import datetime, timedelta

class ZeroTrustIdentityManager:
    def __init__(self):
        self.iam = boto3.client('iam')
        self.cognito = boto3.client('cognito-idp')
        self.sts = boto3.client('sts')
    
    def enforce_mfa_policy(self):
        """Create and enforce MFA policy for all users"""
        
        mfa_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowViewAccountInfo",
                    "Effect": "Allow",
                    "Action": [
                        "iam:GetAccountPasswordPolicy",
                        "iam:GetAccountSummary",
                        "iam:ListVirtualMFADevices"
                    ],
                    "Resource": "*"
                },
                {
                    "Sid": "AllowManageOwnPasswords",
                    "Effect": "Allow",
                    "Action": [
                        "iam:ChangePassword",
                        "iam:GetUser"
                    ],
                    "Resource": "arn:aws:iam::*:user/${aws:username}"
                },
                {
                    "Sid": "AllowManageOwnMFA",
                    "Effect": "Allow",
                    "Action": [
                        "iam:CreateVirtualMFADevice",
                        "iam:DeleteVirtualMFADevice",
                        "iam:EnableMFADevice",
                        "iam:DeactivateMFADevice",
                        "iam:ListMFADevices",
                        "iam:ResyncMFADevice"
                    ],
                    "Resource": [
                        "arn:aws:iam::*:mfa/${aws:username}",
                        "arn:aws:iam::*:user/${aws:username}"
                    ]
                },
                {
                    "Sid": "DenyAllExceptUnlessSignedInWithMFA",
                    "Effect": "Deny",
                    "NotAction": [
                        "iam:CreateVirtualMFADevice",
                        "iam:EnableMFADevice",
                        "iam:GetUser",
                        "iam:ListMFADevices",
                        "iam:ListVirtualMFADevices",
                        "iam:ResyncMFADevice",
                        "sts:GetSessionToken"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "BoolIfExists": {
                            "aws:MultiFactorAuthPresent": "false"
                        }
                    }
                }
            ]
        }
        
        return json.dumps(mfa_policy, indent=2)
    
    def create_adaptive_authentication_policy(self):
        """Create adaptive authentication policy based on risk factors"""
        
        adaptive_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowTrustedNetworks",
                    "Effect": "Allow",
                    "Action": "*",
                    "Resource": "*",
                    "Condition": {
                        "IpAddress": {
                            "aws:SourceIp": [
                                "203.0.113.0/24",  # Corporate network
                                "198.51.100.0/24"  # VPN range
                            ]
                        },
                        "Bool": {
                            "aws:MultiFactorAuthPresent": "true"
                        }
                    }
                },
                {
                    "Sid": "RequireStepUpForUntrustedNetworks",
                    "Effect": "Deny",
                    "Action": [
                        "iam:*",
                        "ec2:TerminateInstances",
                        "rds:DeleteDBInstance",
                        "s3:DeleteBucket"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "IpAddressIfExists": {
                            "aws:SourceIp": "0.0.0.0/0"
                        },
                        "StringNotEquals": {
                            "aws:SourceIp": [
                                "203.0.113.0/24",
                                "198.51.100.0/24"
                            ]
                        },
                        "NumericLessThan": {
                            "aws:MultiFactorAuthAge": "1800"  # 30 minutes
                        }
                    }
                },
                {
                    "Sid": "RequireRecentMFAForCriticalActions",
                    "Effect": "Deny",
                    "Action": [
                        "iam:CreateUser",
                        "iam:DeleteUser",
                        "iam:CreateRole",
                        "iam:DeleteRole",
                        "iam:PutRolePolicy",
                        "iam:AttachRolePolicy"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "NumericGreaterThan": {
                            "aws:MultiFactorAuthAge": "300"  # 5 minutes
                        }
                    }
                }
            ]
        }
        
        return json.dumps(adaptive_policy, indent=2)
    
    def implement_cognito_zero_trust(self, user_pool_name):
        """Implement Zero Trust authentication with Cognito"""
        
        # Create user pool with advanced security features
        user_pool = self.cognito.create_user_pool(
            PoolName=user_pool_name,
            Policies={
                'PasswordPolicy': {
                    'MinimumLength': 12,
                    'RequireUppercase': True,
                    'RequireLowercase': True,
                    'RequireNumbers': True,
                    'RequireSymbols': True,
                    'TemporaryPasswordValidityDays': 1
                }
            },
            MfaConfiguration='REQUIRED',
            DeviceConfiguration={
                'ChallengeRequiredOnNewDevice': True,
                'DeviceOnlyRememberedOnUserPrompt': True
            },
            UserPoolAddOns={
                'AdvancedSecurityMode': 'ENFORCED'
            },
            AdminCreateUserConfig={
                'AllowAdminCreateUserOnly': True,
                'UnusedAccountValidityDays': 7,
                'TemporaryPasswordValidityDays': 1
            },
            AccountRecoverySetting={
                'RecoveryMechanisms': [
                    {
                        'Priority': 1,
                        'Name': 'admin_only'
                    }
                ]
            }
        )
        
        user_pool_id = user_pool['UserPool']['Id']
        
        # Configure risk-based authentication
        risk_configuration = {
            'CompromisedCredentialsRiskConfiguration': {
                'EventFilter': ['SIGN_IN', 'PASSWORD_CHANGE', 'SIGN_UP'],
                'Actions': {
                    'EventAction': 'BLOCK'
                }
            },
            'AccountTakeoverRiskConfiguration': {
                'NotifyConfiguration': {
                    'From': 'security@company.com',
                    'ReplyTo': 'security@company.com',
                    'SourceArn': 'arn:aws:ses:us-east-1:123456789012:identity/security@company.com',
                    'BlockEmail': {
                        'Subject': 'Account Takeover Risk Detected',
                        'HtmlBody': 'Suspicious activity detected on your account.',
                        'TextBody': 'Suspicious activity detected on your account.'
                    }
                },
                'Actions': {
                    'LowAction': {
                        'Notify': True,
                        'EventAction': 'BLOCK'
                    },
                    'MediumAction': {
                        'Notify': True,
                        'EventAction': 'BLOCK'
                    },
                    'HighAction': {
                        'Notify': True,
                        'EventAction': 'BLOCK'
                    }
                }
            },
            'RiskExceptionConfiguration': {
                'BlockedIPRangeList': ['192.0.2.0/24'],  # Known malicious IPs
                'SkippedIPRangeList': ['203.0.113.0/24']  # Trusted corporate IPs
            }
        }
        
        return {
            'user_pool_id': user_pool_id,
            'risk_configuration': risk_configuration
        }

# Identity verification with external providers
def setup_external_identity_federation():
    """Setup external identity federation for Zero Trust"""
    
    identity_providers = {
        'SAML': {
            'provider_type': 'SAML',
            'metadata_document': 'https://company.com/saml/metadata',
            'attribute_mapping': {
                'email': 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress',
                'given_name': 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname',
                'family_name': 'http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname',
                'groups': 'http://schemas.microsoft.com/ws/2008/06/identity/claims/groups'
            }
        },
        'OIDC': {
            'provider_type': 'OIDC',
            'client_id': 'your-oidc-client-id',
            'client_secret': 'your-oidc-client-secret',
            'issuer': 'https://login.company.com',
            'authorize_scopes': 'openid email profile groups'
        }
    }
    
    return identity_providers
```

### Just-In-Time (JIT) Access Implementation

```python
class JITAccessManager:
    def __init__(self):
        self.iam = boto3.client('iam')
        self.lambda_client = boto3.client('lambda')
        self.dynamodb = boto3.resource('dynamodb')
    
    def create_jit_access_system(self):
        """Create Just-In-Time access management system"""
        
        # DynamoDB table for access requests
        access_requests_table = {
            'TableName': 'JITAccessRequests',
            'KeySchema': [
                {
                    'AttributeName': 'RequestId',
                    'KeyType': 'HASH'
                }
            ],
            'AttributeDefinitions': [
                {
                    'AttributeName': 'RequestId',
                    'AttributeType': 'S'
                },
                {
                    'AttributeName': 'UserId',
                    'AttributeType': 'S'
                },
                {
                    'AttributeName': 'Status',
                    'AttributeType': 'S'
                }
            ],
            'GlobalSecondaryIndexes': [
                {
                    'IndexName': 'UserIdIndex',
                    'KeySchema': [
                        {
                            'AttributeName': 'UserId',
                            'KeyType': 'HASH'
                        }
                    ],
                    'Projection': {
                        'ProjectionType': 'ALL'
                    },
                    'BillingMode': 'PAY_PER_REQUEST'
                },
                {
                    'IndexName': 'StatusIndex',
                    'KeySchema': [
                        {
                            'AttributeName': 'Status',
                            'KeyType': 'HASH'
                        }
                    ],
                    'Projection': {
                        'ProjectionType': 'ALL'
                    },
                    'BillingMode': 'PAY_PER_REQUEST'
                }
            ],
            'BillingMode': 'PAY_PER_REQUEST',
            'StreamSpecification': {
                'StreamEnabled': True,
                'StreamViewType': 'NEW_AND_OLD_IMAGES'
            }
        }
        
        return access_requests_table
    
    def request_temporary_access(self, user_id, resource_arn, justification, duration_hours=2):
        """Request temporary access to resources"""
        
        import uuid
        
        request_id = str(uuid.uuid4())
        expiry_time = datetime.utcnow() + timedelta(hours=duration_hours)
        
        access_request = {
            'RequestId': request_id,
            'UserId': user_id,
            'ResourceArn': resource_arn,
            'Justification': justification,
            'RequestedAt': datetime.utcnow().isoformat(),
            'ExpiryTime': expiry_time.isoformat(),
            'Duration': duration_hours,
            'Status': 'PENDING',
            'ApprovalRequired': self._requires_approval(resource_arn)
        }
        
        # Store request in DynamoDB
        table = self.dynamodb.Table('JITAccessRequests')
        table.put_item(Item=access_request)
        
        # Trigger approval workflow if required
        if access_request['ApprovalRequired']:
            self._trigger_approval_workflow(request_id, access_request)
        else:
            self._auto_approve_request(request_id)
        
        return request_id
    
    def _requires_approval(self, resource_arn):
        """Determine if resource access requires approval"""
        
        high_risk_patterns = [
            'arn:aws:iam::*:role/AdminRole',
            'arn:aws:s3:::production-*',
            'arn:aws:rds:*:*:db:production-*',
            'arn:aws:ec2:*:*:instance/*'
        ]
        
        for pattern in high_risk_patterns:
            if self._matches_pattern(resource_arn, pattern):
                return True
        
        return False
    
    def _matches_pattern(self, arn, pattern):
        """Check if ARN matches pattern with wildcards"""
        import fnmatch
        return fnmatch.fnmatch(arn, pattern)
    
    def _trigger_approval_workflow(self, request_id, access_request):
        """Trigger approval workflow for access request"""
        
        # Send approval request via SNS
        sns = boto3.client('sns')
        
        approval_message = {
            'request_id': request_id,
            'user_id': access_request['UserId'],
            'resource': access_request['ResourceArn'],
            'justification': access_request['Justification'],
            'duration': access_request['Duration'],
            'approval_url': f'https://company.com/approve/{request_id}'
        }
        
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:access-approvals',
            Message=json.dumps(approval_message),
            Subject=f'Access Request Pending Approval: {request_id}'
        )
    
    def approve_access_request(self, request_id, approver_id):
        """Approve access request and grant temporary permissions"""
        
        table = self.dynamodb.Table('JITAccessRequests')
        
        # Get request details
        response = table.get_item(Key={'RequestId': request_id})
        if 'Item' not in response:
            raise Exception('Request not found')
        
        request_item = response['Item']
        
        if request_item['Status'] != 'PENDING':
            raise Exception('Request already processed')
        
        # Create temporary role
        temp_role_arn = self._create_temporary_role(request_item)
        
        # Update request status
        table.update_item(
            Key={'RequestId': request_id},
            UpdateExpression='SET #status = :status, ApprovedBy = :approver, ApprovedAt = :approved_at, TempRoleArn = :role_arn',
            ExpressionAttributeNames={
                '#status': 'Status'
            },
            ExpressionAttributeValues={
                ':status': 'APPROVED',
                ':approver': approver_id,
                ':approved_at': datetime.utcnow().isoformat(),
                ':role_arn': temp_role_arn
            }
        )
        
        # Schedule role deletion
        self._schedule_role_cleanup(temp_role_arn, request_item['ExpiryTime'])
        
        return temp_role_arn
    
    def _create_temporary_role(self, request_item):
        """Create temporary IAM role with specific permissions"""
        
        role_name = f"JIT-{request_item['UserId']}-{request_item['RequestId'][:8]}"
        
        trust_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": f"arn:aws:iam::{boto3.client('sts').get_caller_identity()['Account']}:user/{request_item['UserId']}"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "StringEquals": {
                            "sts:ExternalId": request_item['RequestId']
                        },
                        "DateLessThan": {
                            "aws:CurrentTime": request_item['ExpiryTime']
                        }
                    }
                }
            ]
        }
        
        # Create role
        role_response = self.iam.create_role(
            RoleName=role_name,
            AssumeRolePolicyDocument=json.dumps(trust_policy),
            Description=f"Temporary JIT access role for {request_item['UserId']}",
            MaxSessionDuration=3600,
            Tags=[
                {
                    'Key': 'Purpose',
                    'Value': 'JITAccess'
                },
                {
                    'Key': 'RequestId',
                    'Value': request_item['RequestId']
                },
                {
                    'Key': 'ExpiryTime',
                    'Value': request_item['ExpiryTime']
                }
            ]
        )
        
        # Attach minimal permissions based on resource
        permissions = self._generate_minimal_permissions(request_item['ResourceArn'])
        
        self.iam.put_role_policy(
            RoleName=role_name,
            PolicyName='JITPermissions',
            PolicyDocument=json.dumps(permissions)
        )
        
        return role_response['Role']['Arn']
    
    def _generate_minimal_permissions(self, resource_arn):
        """Generate minimal permissions for specific resource"""
        
        if 's3' in resource_arn:
            return {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Action": [
                            "s3:GetObject",
                            "s3:ListBucket"
                        ],
                        "Resource": [
                            resource_arn,
                            f"{resource_arn}/*"
                        ]
                    }
                ]
            }
        elif 'ec2' in resource_arn:
            return {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Action": [
                            "ec2:DescribeInstances",
                            "ec2:StartInstances",
                            "ec2:StopInstances"
                        ],
                        "Resource": resource_arn
                    }
                ]
            }
        else:
            return {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Effect": "Allow",
                        "Action": "iam:GetRole",
                        "Resource": resource_arn
                    }
                ]
            }
```

## Device Trust and Verification

### Device Trust Implementation

```python
class DeviceTrustManager:
    def __init__(self):
        self.ssm = boto3.client('ssm')
        self.iam = boto3.client('iam')
        self.dynamodb = boto3.resource('dynamodb')
    
    def register_trusted_device(self, device_id, user_id, device_info):
        """Register device as trusted after verification"""
        
        device_table = self.dynamodb.Table('TrustedDevices')
        
        device_record = {
            'DeviceId': device_id,
            'UserId': user_id,
            'DeviceInfo': device_info,
            'RegisteredAt': datetime.utcnow().isoformat(),
            'LastSeen': datetime.utcnow().isoformat(),
            'Status': 'ACTIVE',
            'TrustLevel': self._calculate_trust_level(device_info),
            'Certificates': []
        }
        
        # Generate device certificate
        certificate_arn = self._generate_device_certificate(device_id, user_id)
        device_record['Certificates'].append(certificate_arn)
        
        device_table.put_item(Item=device_record)
        
        return device_id
    
    def _calculate_trust_level(self, device_info):
        """Calculate device trust level based on various factors"""
        
        trust_score = 50  # Base score
        
        # Operating system trust
        if device_info.get('os') in ['Windows 10', 'Windows 11', 'macOS', 'iOS']:
            trust_score += 20
        elif device_info.get('os') in ['Android', 'Linux']:
            trust_score += 10
        
        # Device management
        if device_info.get('managed_device'):
            trust_score += 25
        
        # Encryption status
        if device_info.get('encrypted_storage'):
            trust_score += 15
        
        # Security software
        if device_info.get('antivirus_installed'):
            trust_score += 10
        
        # Determine trust level
        if trust_score >= 80:
            return 'HIGH'
        elif trust_score >= 60:
            return 'MEDIUM'
        else:
            return 'LOW'
    
    def _generate_device_certificate(self, device_id, user_id):
        """Generate device certificate for authentication"""
        
        acm = boto3.client('acm')
        
        # In production, this would integrate with your PKI
        # Here we simulate certificate generation
        certificate_request = {
            'DomainName': f'{device_id}.devices.company.com',
            'ValidationMethod': 'DNS',
            'SubjectAlternativeNames': [
                f'{user_id}.{device_id}.devices.company.com'
            ]
        }
        
        # Store certificate metadata
        return f'arn:aws:acm:us-east-1:123456789012:certificate/{device_id}'
    
    def verify_device_compliance(self, device_id):
        """Verify device compliance with security policies"""
        
        compliance_checks = {
            'os_version': self._check_os_version(device_id),
            'patch_level': self._check_patch_level(device_id),
            'antivirus_status': self._check_antivirus(device_id),
            'firewall_status': self._check_firewall(device_id),
            'encryption_status': self._check_encryption(device_id),
            'unauthorized_software': self._check_unauthorized_software(device_id)
        }
        
        # Calculate compliance score
        passed_checks = sum(1 for check in compliance_checks.values() if check)
        compliance_score = (passed_checks / len(compliance_checks)) * 100
        
        # Update device trust level based on compliance
        if compliance_score < 70:
            self._update_device_trust_level(device_id, 'LOW')
        elif compliance_score < 90:
            self._update_device_trust_level(device_id, 'MEDIUM')
        else:
            self._update_device_trust_level(device_id, 'HIGH')
        
        return {
            'device_id': device_id,
            'compliance_score': compliance_score,
            'checks': compliance_checks,
            'compliant': compliance_score >= 70
        }
    
    def _check_os_version(self, device_id):
        """Check if OS version is supported"""
        try:
            # Query Systems Manager for OS version
            response = self.ssm.get_inventory_summary(
                Filters=[
                    {
                        'Key': 'AWS:InstanceInformation.InstanceId',
                        'Values': [device_id]
                    }
                ]
            )
            # Implement OS version validation logic
            return True
        except:
            return False
    
    def _check_patch_level(self, device_id):
        """Check if device has latest security patches"""
        try:
            response = self.ssm.describe_instance_patch_states(
                InstanceIds=[device_id]
            )
            # Check patch compliance
            return response['InstancePatchStates'][0]['FailedCount'] == 0
        except:
            return False
    
    def _check_antivirus(self, device_id):
        """Check antivirus status"""
        # Integrate with endpoint protection solution
        return True  # Placeholder
    
    def _check_firewall(self, device_id):
        """Check firewall status"""
        # Check device firewall configuration
        return True  # Placeholder
    
    def _check_encryption(self, device_id):
        """Check disk encryption status"""
        # Verify full disk encryption
        return True  # Placeholder
    
    def _check_unauthorized_software(self, device_id):
        """Check for unauthorized software"""
        # Scan for prohibited applications
        return True  # Placeholder
    
    def _update_device_trust_level(self, device_id, trust_level):
        """Update device trust level in database"""
        
        device_table = self.dynamodb.Table('TrustedDevices')
        
        device_table.update_item(
            Key={'DeviceId': device_id},
            UpdateExpression='SET TrustLevel = :trust_level, LastComplianceCheck = :timestamp',
            ExpressionAttributeValues={
                ':trust_level': trust_level,
                ':timestamp': datetime.utcnow().isoformat()
            }
        )

# Certificate-based device authentication
def setup_device_certificates():
    """Setup certificate-based device authentication"""
    
    certificate_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowCertificateBasedAuth",
                "Effect": "Allow",
                "Action": [
                    "sts:AssumeRole"
                ],
                "Resource": "arn:aws:iam::*:role/DeviceRole-*",
                "Condition": {
                    "StringEquals": {
                        "aws:RequestedRegion": "us-east-1"
                    },
                    "Bool": {
                        "aws:ViaAWSService": "false"
                    },
                    "StringLike": {
                        "aws:userid": "AIDACKCEVSQ6C2EXAMPLE:*"
                    }
                }
            }
        ]
    }
    
    return certificate_policy
```

### Endpoint Detection and Response Integration

```python
class EndpointSecurityManager:
    def __init__(self):
        self.guardduty = boto3.client('guardduty')
        self.security_hub = boto3.client('securityhub')
        self.sns = boto3.client('sns')
    
    def setup_endpoint_monitoring(self):
        """Setup comprehensive endpoint monitoring"""
        
        # Enable GuardDuty
        detector_response = self.guardduty.create_detector(
            Enable=True,
            FindingPublishingFrequency='ONE_HOUR',
            DataSources={
                'S3Logs': {
                    'Enable': True
                },
                'KubernetesConfiguration': {
                    'AuditLogs': {
                        'Enable': True
                    }
                },
                'MalwareProtection': {
                    'ScanEc2InstanceWithFindings': {
                        'EbsVolumes': True
                    }
                }
            }
        )
        
        detector_id = detector_response['DetectorId']
        
        # Setup Security Hub
        self.security_hub.enable_security_hub(
            Tags={
                'Purpose': 'ZeroTrustMonitoring'
            },
            EnableDefaultStandards=True
        )
        
        # Configure findings export
        self._setup_findings_export(detector_id)
        
        return detector_id
    
    def _setup_findings_export(self, detector_id):
        """Setup findings export to security systems"""
        
        # Create SNS topic for security alerts
        topic_response = self.sns.create_topic(
            Name='SecurityFindings',
            Tags=[
                {
                    'Key': 'Purpose',
                    'Value': 'ZeroTrustAlerts'
                }
            ]
        )
        
        # Setup CloudWatch Events rule for GuardDuty findings
        events = boto3.client('events')
        
        events.put_rule(
            Name='GuardDutyFindingsRule',
            EventPattern=json.dumps({
                "source": ["aws.guardduty"],
                "detail-type": ["GuardDuty Finding"],
                "detail": {
                    "severity": [7, 8, 9]  # High severity only
                }
            }),
            State='ENABLED',
            Description='Route high-severity GuardDuty findings'
        )
        
        # Add SNS target
        events.put_targets(
            Rule='GuardDutyFindingsRule',
            Targets=[
                {
                    'Id': '1',
                    'Arn': topic_response['TopicArn'],
                    'InputTransformer': {
                        'InputPathsMap': {
                            'severity': '$.detail.severity',
                            'type': '$.detail.type',
                            'title': '$.detail.title',
                            'description': '$.detail.description'
                        },
                        'InputTemplate': '{"severity": "<severity>", "type": "<type>", "title": "<title>", "description": "<description>"}'
                    }
                }
            ]
        )
    
    def process_security_finding(self, finding):
        """Process security finding and take automated response"""
        
        severity = finding.get('severity', 0)
        finding_type = finding.get('type', '')
        
        response_actions = []
        
        # High severity findings
        if severity >= 7:
            if 'Trojan' in finding_type or 'Malware' in finding_type:
                response_actions.append('isolate_instance')
                response_actions.append('revoke_credentials')
            elif 'UnauthorizedAPICall' in finding_type:
                response_actions.append('disable_user')
                response_actions.append('force_password_reset')
            elif 'Cryptocurrency' in finding_type:
                response_actions.append('terminate_instance')
        
        # Execute response actions
        for action in response_actions:
            self._execute_response_action(action, finding)
        
        return response_actions
    
    def _execute_response_action(self, action, finding):
        """Execute automated response action"""
        
        if action == 'isolate_instance':
            self._isolate_instance(finding.get('instance_id'))
        elif action == 'revoke_credentials':
            self._revoke_user_credentials(finding.get('user_name'))
        elif action == 'disable_user':
            self._disable_user_account(finding.get('user_name'))
        elif action == 'terminate_instance':
            self._terminate_instance(finding.get('instance_id'))
    
    def _isolate_instance(self, instance_id):
        """Isolate compromised instance"""
        if not instance_id:
            return
        
        ec2 = boto3.client('ec2')
        
        # Create isolation security group
        isolation_sg = ec2.create_security_group(
            GroupName=f'isolation-{instance_id}',
            Description='Isolation security group for compromised instance',
            VpcId=self._get_instance_vpc(instance_id)
        )
        
        # Apply isolation security group
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=[isolation_sg['GroupId']]
        )
```

## Network Micro-segmentation

### Micro-segmentation with Security Groups

```python
class NetworkMicroSegmentation:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.elbv2 = boto3.client('elbv2')
    
    def create_microsegmentation_architecture(self, vpc_id):
        """Create comprehensive micro-segmentation architecture"""
        
        # Define application tiers
        tiers = {
            'web': {
                'description': 'Web tier - public facing',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 80, 'source': '0.0.0.0/0'},
                    {'protocol': 'tcp', 'port': 443, 'source': '0.0.0.0/0'}
                ],
                'egress_rules': [
                    {'protocol': 'tcp', 'port': 8080, 'destination': 'app-tier'}
                ]
            },
            'app': {
                'description': 'Application tier - business logic',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 8080, 'source': 'web-tier'}
                ],
                'egress_rules': [
                    {'protocol': 'tcp', 'port': 3306, 'destination': 'db-tier'},
                    {'protocol': 'tcp', 'port': 5432, 'destination': 'db-tier'}
                ]
            },
            'db': {
                'description': 'Database tier - data storage',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 3306, 'source': 'app-tier'},
                    {'protocol': 'tcp', 'port': 5432, 'source': 'app-tier'}
                ],
                'egress_rules': []
            },
            'mgmt': {
                'description': 'Management tier - admin access',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 22, 'source': 'bastion-sg'},
                    {'protocol': 'tcp', 'port': 3389, 'source': 'bastion-sg'}
                ],
                'egress_rules': [
                    {'protocol': 'tcp', 'port': 443, 'destination': '0.0.0.0/0'}
                ]
            }
        }
        
        security_groups = {}
        
        # Create security groups for each tier
        for tier_name, tier_config in tiers.items():
            sg_name = f'{tier_name}-tier-sg'
            
            sg_response = self.ec2.create_security_group(
                GroupName=sg_name,
                Description=tier_config['description'],
                VpcId=vpc_id,
                TagSpecifications=[
                    {
                        'ResourceType': 'security-group',
                        'Tags': [
                            {
                                'Key': 'Name',
                                'Value': sg_name
                            },
                            {
                                'Key': 'Tier',
                                'Value': tier_name
                            },
                            {
                                'Key': 'Purpose',
                                'Value': 'Microsegmentation'
                            }
                        ]
                    }
                ]
            )
            
            security_groups[f'{tier_name}-tier'] = sg_response['GroupId']
        
        # Configure security group rules
        self._configure_microsegmentation_rules(security_groups, tiers)
        
        return security_groups
    
    def _configure_microsegmentation_rules(self, security_groups, tiers):
        """Configure security group rules for micro-segmentation"""
        
        for tier_name, tier_config in tiers.items():
            sg_id = security_groups[f'{tier_name}-tier']
            
            # Configure ingress rules
            for rule in tier_config['ingress_rules']:
                source = rule['source']
                
                # Resolve source reference
                if source in security_groups:
                    source_reference = {'GroupId': security_groups[source]}
                else:
                    source_reference = {'CidrIp': source}
                
                try:
                    self.ec2.authorize_security_group_ingress(
                        GroupId=sg_id,
                        IpPermissions=[
                            {
                                'IpProtocol': rule['protocol'],
                                'FromPort': rule['port'],
                                'ToPort': rule['port'],
                                **source_reference
                            }
                        ]
                    )
                except Exception as e:
                    print(f"Error adding ingress rule: {e}")
            
            # Configure egress rules (remove default allow all first)
            try:
                self.ec2.revoke_security_group_egress(
                    GroupId=sg_id,
                    IpPermissions=[
                        {
                            'IpProtocol': '-1',
                            'CidrIp': '0.0.0.0/0'
                        }
                    ]
                )
            except:
                pass  # Rule might not exist
            
            for rule in tier_config['egress_rules']:
                destination = rule['destination']
                
                # Resolve destination reference
                if destination in security_groups:
                    dest_reference = {'DestinationSecurityGroupId': security_groups[destination]}
                else:
                    dest_reference = {'CidrIp': destination}
                
                try:
                    self.ec2.authorize_security_group_egress(
                        GroupId=sg_id,
                        IpPermissions=[
                            {
                                'IpProtocol': rule['protocol'],
                                'FromPort': rule['port'],
                                'ToPort': rule['port'],
                                **dest_reference
                            }
                        ]
                    )
                except Exception as e:
                    print(f"Error adding egress rule: {e}")
    
    def implement_application_level_segmentation(self, app_name):
        """Implement application-level micro-segmentation"""
        
        # Create application-specific security groups
        app_security_groups = {
            f'{app_name}-frontend': {
                'description': f'{app_name} frontend components',
                'rules': [
                    {'type': 'ingress', 'protocol': 'tcp', 'port': 80, 'source': 'alb'},
                    {'type': 'ingress', 'protocol': 'tcp', 'port': 443, 'source': 'alb'},
                    {'type': 'egress', 'protocol': 'tcp', 'port': 8080, 'destination': f'{app_name}-backend'}
                ]
            },
            f'{app_name}-backend': {
                'description': f'{app_name} backend services',
                'rules': [
                    {'type': 'ingress', 'protocol': 'tcp', 'port': 8080, 'source': f'{app_name}-frontend'},
                    {'type': 'egress', 'protocol': 'tcp', 'port': 5432, 'destination': f'{app_name}-database'}
                ]
            },
            f'{app_name}-database': {
                'description': f'{app_name} database tier',
                'rules': [
                    {'type': 'ingress', 'protocol': 'tcp', 'port': 5432, 'source': f'{app_name}-backend'}
                ]
            }
        }
        
        return app_security_groups
```

### Network ACLs for Subnet-Level Control

```yaml
# CloudFormation template for subnet-level micro-segmentation
NetworkACLMicrosegmentation:
  AWSTemplateFormatVersion: '2010-09-09'
  Description: Network ACL-based micro-segmentation
  
  Parameters:
    VPCId:
      Type: AWS::EC2::VPC::Id
    
  Resources:
    # Web tier NACL
    WebTierNACL:
      Type: AWS::EC2::NetworkAcl
      Properties:
        VpcId: !Ref VPCId
        Tags:
          - Key: Name
            Value: WebTier-NACL
          - Key: Tier
            Value: web
    
    # Web tier inbound rules
    WebTierInboundHTTP:
      Type: AWS::EC2::NetworkAclEntry
      Properties:
        NetworkAclId: !Ref WebTierNACL
        RuleNumber: 100
        Protocol: 6
        RuleAction: allow
        CidrBlock: 0.0.0.0/0
        PortRange:
          From: 80
          To: 80
    
    WebTierInboundHTTPS:
      Type: AWS::EC2::NetworkAclEntry
      Properties:
        NetworkAclId: !Ref WebTierNACL
        RuleNumber: 110
        Protocol: 6
        RuleAction: allow
        CidrBlock: 0.0.0.0/0
        PortRange:
          From: 443
          To: 443
    
    WebTierInboundReturn:
      Type: AWS::EC2::NetworkAclEntry
      Properties:
        NetworkAclId: !Ref WebTierNACL
        RuleNumber: 120
        Protocol: 6
        RuleAction: allow
        CidrBlock: 0.0.0.0/0
        PortRange:
          From: 1024
          To: 65535
    
    # Web tier outbound rules
    WebTierOutboundApp:
      Type: AWS::EC2::NetworkAclEntry
      Properties:
        NetworkAclId: !Ref WebTierNACL
        RuleNumber: 100
        Protocol: 6
        RuleAction: allow
        Egress: true
        CidrBlock: 10.0.2.0/24  # App tier subnet
        PortRange:
          From: 8080
          To: 8080
    
    WebTierOutboundReturn:
      Type: AWS::EC2::NetworkAclEntry
      Properties:
        NetworkAclId: !Ref WebTierNACL
        RuleNumber: 110
        Protocol: 6
        RuleAction: allow
        Egress: true
        CidrBlock: 0.0.0.0/0
        PortRange:
          From: 1024
          To: 65535
    
    # App tier NACL
    AppTierNACL:
      Type: AWS::EC2::NetworkAcl
      Properties:
        VpcId: !Ref VPCId
        Tags:
          - Key: Name
            Value: AppTier-NACL
          - Key: Tier
            Value: app
    
    # App tier rules - only allow from web tier
    AppTierInboundWeb:
      Type: AWS::EC2::NetworkAclEntry
      Properties:
        NetworkAclId: !Ref AppTierNACL
        RuleNumber: 100
        Protocol: 6
        RuleAction: allow
        CidrBlock: 10.0.1.0/24  # Web tier subnet
        PortRange:
          From: 8080
          To: 8080
    
    # Database tier NACL
    DatabaseTierNACL:
      Type: AWS::EC2::NetworkAcl
      Properties:
        VpcId: !Ref VPCId
        Tags:
          - Key: Name
            Value: DatabaseTier-NACL
          - Key: Tier
            Value: database
    
    # Database tier - only allow from app tier
    DatabaseTierInboundApp:
      Type: AWS::EC2::NetworkAclEntry
      Properties:
        NetworkAclId: !Ref DatabaseTierNACL
        RuleNumber: 100
        Protocol: 6
        RuleAction: allow
        CidrBlock: 10.0.2.0/24  # App tier subnet
        PortRange:
          From: 5432
          To: 5432
```

## Continuous Monitoring and Validation

### Comprehensive Monitoring Implementation

```python
class ZeroTrustMonitoring:
    def __init__(self):
        self.cloudtrail = boto3.client('cloudtrail')
        self.cloudwatch = boto3.client('cloudwatch')
        self.config = boto3.client('config')
        self.security_hub = boto3.client('securityhub')
    
    def setup_comprehensive_monitoring(self):
        """Setup comprehensive Zero Trust monitoring"""
        
        # Enable CloudTrail for all API calls
        trail_config = self._setup_cloudtrail()
        
        # Configure Config rules for compliance
        config_rules = self._setup_config_rules()
        
        # Setup custom metrics and alarms
        custom_metrics = self._setup_custom_metrics()
        
        # Configure Security Hub integrations
        security_hub_config = self._setup_security_hub()
        
        return {
            'cloudtrail': trail_config,
            'config_rules': config_rules,
            'custom_metrics': custom_metrics,
            'security_hub': security_hub_config
        }
    
    def _setup_cloudtrail(self):
        """Setup CloudTrail for comprehensive logging"""
        
        # Create S3 bucket for CloudTrail logs
        s3 = boto3.client('s3')
        bucket_name = 'zero-trust-cloudtrail-logs'
        
        try:
            s3.create_bucket(Bucket=bucket_name)
            
            # Configure bucket policy
            bucket_policy = {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "AWSCloudTrailAclCheck",
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "cloudtrail.amazonaws.com"
                        },
                        "Action": "s3:GetBucketAcl",
                        "Resource": f"arn:aws:s3:::{bucket_name}"
                    },
                    {
                        "Sid": "AWSCloudTrailWrite",
                        "Effect": "Allow",
                        "Principal": {
                            "Service": "cloudtrail.amazonaws.com"
                        },
                        "Action": "s3:PutObject",
                        "Resource": f"arn:aws:s3:::{bucket_name}/*",
                        "Condition": {
                            "StringEquals": {
                                "s3:x-amz-acl": "bucket-owner-full-control"
                            }
                        }
                    }
                ]
            }
            
            s3.put_bucket_policy(
                Bucket=bucket_name,
                Policy=json.dumps(bucket_policy)
            )
        except:
            pass  # Bucket might already exist
        
        # Create CloudTrail
        trail_response = self.cloudtrail.create_trail(
            Name='ZeroTrustTrail',
            S3BucketName=bucket_name,
            IncludeGlobalServiceEvents=True,
            IsMultiRegionTrail=True,
            EnableLogFileValidation=True,
            EventSelectors=[
                {
                    'ReadWriteType': 'All',
                    'IncludeManagementEvents': True,
                    'DataResources': [
                        {
                            'Type': 'AWS::S3::Object',
                            'Values': ['arn:aws:s3:::sensitive-data-bucket/*']
                        }
                    ]
                }
            ],
            InsightSelectors=[
                {
                    'InsightType': 'ApiCallRateInsight'
                }
            ]
        )
        
        # Start logging
        self.cloudtrail.start_logging(Name='ZeroTrustTrail')
        
        return trail_response
    
    def _setup_config_rules(self):
        """Setup AWS Config rules for compliance monitoring"""
        
        config_rules = [
            {
                'ConfigRuleName': 'root-access-key-check',
                'Source': {
                    'Owner': 'AWS',
                    'SourceIdentifier': 'ROOT_ACCESS_KEY_CHECK'
                }
            },
            {
                'ConfigRuleName': 'mfa-enabled-for-iam-console-access',
                'Source': {
                    'Owner': 'AWS',
                    'SourceIdentifier': 'MFA_ENABLED_FOR_IAM_CONSOLE_ACCESS'
                }
            },
            {
                'ConfigRuleName': 'iam-policy-no-statements-with-admin-access',
                'Source': {
                    'Owner': 'AWS',
                    'SourceIdentifier': 'IAM_POLICY_NO_STATEMENTS_WITH_ADMIN_ACCESS'
                }
            },
            {
                'ConfigRuleName': 'security-group-ssh-check',
                'Source': {
                    'Owner': 'AWS',
                    'SourceIdentifier': 'INCOMING_SSH_DISABLED'
                }
            },
            {
                'ConfigRuleName': 'vpc-default-security-group-closed',
                'Source': {
                    'Owner': 'AWS',
                    'SourceIdentifier': 'VPC_DEFAULT_SECURITY_GROUP_CLOSED'
                }
            }
        ]
        
        created_rules = []
        for rule in config_rules:
            try:
                response = self.config.put_config_rule(ConfigRule=rule)
                created_rules.append(rule['ConfigRuleName'])
            except Exception as e:
                print(f"Error creating config rule {rule['ConfigRuleName']}: {e}")
        
        return created_rules
    
    def _setup_custom_metrics(self):
        """Setup custom CloudWatch metrics for Zero Trust monitoring"""
        
        custom_alarms = [
            {
                'AlarmName': 'UnauthorizedAPICallsAlarm',
                'MetricName': 'UnauthorizedAPICalls',
                'Namespace': 'ZeroTrust/Security',
                'Statistic': 'Sum',
                'Period': 300,
                'EvaluationPeriods': 1,
                'Threshold': 5,
                'ComparisonOperator': 'GreaterThanThreshold'
            },
            {
                'AlarmName': 'HighPrivilegeRoleAssumptions',
                'MetricName': 'HighPrivilegeAssumeRole',
                'Namespace': 'ZeroTrust/Security',
                'Statistic': 'Sum',
                'Period': 300,
                'EvaluationPeriods': 1,
                'Threshold': 10,
                'ComparisonOperator': 'GreaterThanThreshold'
            },
            {
                'AlarmName': 'UnusualNetworkTrafficAlarm',
                'MetricName': 'UnusualNetworkTraffic',
                'Namespace': 'ZeroTrust/Network',
                'Statistic': 'Sum',
                'Period': 300,
                'EvaluationPeriods': 2,
                'Threshold': 1000,
                'ComparisonOperator': 'GreaterThanThreshold'
            }
        ]
        
        created_alarms = []
        for alarm_config in custom_alarms:
            try:
                self.cloudwatch.put_metric_alarm(
                    AlarmName=alarm_config['AlarmName'],
                    ComparisonOperator=alarm_config['ComparisonOperator'],
                    EvaluationPeriods=alarm_config['EvaluationPeriods'],
                    MetricName=alarm_config['MetricName'],
                    Namespace=alarm_config['Namespace'],
                    Period=alarm_config['Period'],
                    Statistic=alarm_config['Statistic'],
                    Threshold=alarm_config['Threshold'],
                    ActionsEnabled=True,
                    AlarmActions=[
                        'arn:aws:sns:us-east-1:123456789012:zero-trust-alerts'
                    ],
                    AlarmDescription=f'Zero Trust monitoring: {alarm_config["AlarmName"]}'
                )
                created_alarms.append(alarm_config['AlarmName'])
            except Exception as e:
                print(f"Error creating alarm {alarm_config['AlarmName']}: {e}")
        
        return created_alarms
    
    def analyze_trust_score(self, user_id, time_window_hours=24):
        """Calculate dynamic trust score for user"""
        
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=time_window_hours)
        
        trust_factors = {
            'authentication_failures': 0,
            'unusual_locations': 0,
            'device_compliance': 100,
            'privilege_escalation_attempts': 0,
            'successful_authentications': 0,
            'policy_violations': 0
        }
        
        # Analyze CloudTrail events for user
        events = self.cloudtrail.lookup_events(
            LookupAttributes=[
                {
                    'AttributeKey': 'Username',
                    'AttributeValue': user_id
                }
            ],
            StartTime=start_time,
            EndTime=end_time
        )
        
        for event in events['Events']:
            event_detail = json.loads(event['CloudTrailEvent'])
            
            # Count authentication failures
            if event['EventName'] == 'ConsoleLogin' and event_detail.get('responseElements', {}).get('ConsoleLogin') == 'Failure':
                trust_factors['authentication_failures'] += 1
            
            # Count successful authentications
            elif event['EventName'] == 'ConsoleLogin' and event_detail.get('responseElements', {}).get('ConsoleLogin') == 'Success':
                trust_factors['successful_authentications'] += 1
            
            # Detect privilege escalation attempts
            elif event['EventName'] in ['AttachUserPolicy', 'PutUserPolicy', 'AddUserToGroup']:
                trust_factors['privilege_escalation_attempts'] += 1
        
        # Calculate trust score
        base_score = 100
        
        # Deduct points for negative factors
        base_score -= min(trust_factors['authentication_failures'] * 10, 50)
        base_score -= min(trust_factors['unusual_locations'] * 15, 30)
        base_score -= min(trust_factors['privilege_escalation_attempts'] * 20, 40)
        base_score -= min(trust_factors['policy_violations'] * 5, 25)
        
        # Add points for positive factors
        if trust_factors['successful_authentications'] > 0:
            base_score += min(trust_factors['successful_authentications'] * 2, 10)
        
        # Factor in device compliance
        base_score = base_score * (trust_factors['device_compliance'] / 100)
        
        final_score = max(min(base_score, 100), 0)
        
        return {
            'user_id': user_id,
            'trust_score': final_score,
            'factors': trust_factors,
            'risk_level': self._categorize_risk_level(final_score),
            'recommendations': self._generate_trust_recommendations(trust_factors, final_score)
        }
    
    def _categorize_risk_level(self, trust_score):
        """Categorize risk level based on trust score"""
        if trust_score >= 80:
            return 'LOW'
        elif trust_score >= 60:
            return 'MEDIUM'
        elif trust_score >= 40:
            return 'HIGH'
        else:
            return 'CRITICAL'
    
    def _generate_trust_recommendations(self, factors, score):
        """Generate recommendations to improve trust score"""
        recommendations = []
        
        if factors['authentication_failures'] > 2:
            recommendations.append('Implement additional MFA methods')
        
        if factors['unusual_locations'] > 0:
            recommendations.append('Review location-based access policies')
        
        if factors['device_compliance'] < 90:
            recommendations.append('Update device security compliance')
        
        if factors['privilege_escalation_attempts'] > 0:
            recommendations.append('Review and restrict privilege escalation permissions')
        
        if score < 70:
            recommendations.append('Require step-up authentication for sensitive operations')
        
        return recommendations
```

This comprehensive Zero Trust architecture implementation provides the foundation for a robust security posture that continuously verifies trust, enforces least privilege access, and monitors for threats across all layers of the AWS infrastructure.