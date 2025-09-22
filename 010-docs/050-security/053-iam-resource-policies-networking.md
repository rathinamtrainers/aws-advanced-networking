# Topic 53: IAM and Resource Policies for Networking

## Introduction

Identity and Access Management (IAM) for networking services requires a deep understanding of AWS's complex permission model. This topic covers IAM policies for networking services, VPC endpoint policies, resource-based policies, cross-account access patterns, and service-linked roles. Proper IAM configuration ensures secure access control while enabling necessary functionality across networking resources.

## IAM for Networking Services Overview

### Networking Service Permissions Model

AWS networking services use multiple permission layers:

```
Request Flow:
User/Role → IAM Policy → Resource Policy → Service Policy → Resource Access
    ↓           ↓           ↓               ↓               ↓
 Identity    Action      Resource        Condition     Final Decision
```

### Core Networking IAM Actions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VPCManagement",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:DeleteVpc",
        "ec2:DescribeVpcs",
        "ec2:ModifyVpcAttribute",
        "ec2:CreateSubnet",
        "ec2:DeleteSubnet",
        "ec2:DescribeSubnets",
        "ec2:ModifySubnetAttribute",
        "ec2:CreateRouteTable",
        "ec2:DeleteRouteTable",
        "ec2:DescribeRouteTables",
        "ec2:AssociateRouteTable",
        "ec2:DisassociateRouteTable",
        "ec2:CreateRoute",
        "ec2:DeleteRoute",
        "ec2:ReplaceRoute"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SecurityGroupManagement",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateSecurityGroup",
        "ec2:DeleteSecurityGroup",
        "ec2:DescribeSecurityGroups",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupEgress"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:Region": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
```

### Advanced IAM Policy for Network Administrators

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "NetworkAdministratorFullAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "elasticloadbalancing:*",
        "route53:*",
        "cloudfront:*",
        "directconnect:*",
        "globalaccelerator:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RestrictCriticalActions",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "ec2:DeleteVolume",
        "rds:DeleteDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    },
    {
      "Sid": "AllowVPCEndpointManagement",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpcEndpoint",
        "ec2:DeleteVpcEndpoint",
        "ec2:DescribeVpcEndpoints",
        "ec2:ModifyVpcEndpoint",
        "ec2:CreateVpcEndpointConnectionNotification",
        "ec2:DeleteVpcEndpointConnectionNotification"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMPassRoleForNetworking",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::*:role/NetworkingServiceRole",
        "arn:aws:iam::*:role/VPCFlowLogsRole"
      ]
    }
  ]
}
```

## VPC Endpoint Policies

### Interface VPC Endpoint Policies

```python
import boto3
import json

class VPCEndpointPolicyManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
    
    def create_s3_vpc_endpoint_policy(self):
        """Create restrictive S3 VPC endpoint policy"""
        
        policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowSpecificBucketAccess",
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": [
                        "s3:GetObject",
                        "s3:PutObject",
                        "s3:DeleteObject",
                        "s3:ListBucket"
                    ],
                    "Resource": [
                        "arn:aws:s3:::company-data-bucket/*",
                        "arn:aws:s3:::company-data-bucket",
                        "arn:aws:s3:::company-logs-bucket/*",
                        "arn:aws:s3:::company-logs-bucket"
                    ],
                    "Condition": {
                        "StringEquals": {
                            "aws:PrincipalAccount": "123456789012"
                        },
                        "IpAddress": {
                            "aws:SourceIp": [
                                "10.0.0.0/16",
                                "172.16.0.0/12"
                            ]
                        }
                    }
                },
                {
                    "Sid": "DenyUnencryptedUploads",
                    "Effect": "Deny",
                    "Principal": "*",
                    "Action": "s3:PutObject",
                    "Resource": "arn:aws:s3:::company-data-bucket/*",
                    "Condition": {
                        "StringNotEquals": {
                            "s3:x-amz-server-side-encryption": "AES256"
                        }
                    }
                }
            ]
        }
        
        return json.dumps(policy, indent=2)
    
    def create_interface_endpoint_policy(self, service_name, allowed_actions):
        """Create policy for interface VPC endpoints"""
        
        policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": f"Allow{service_name}Access",
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": allowed_actions,
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "aws:PrincipalAccount": "123456789012"
                        },
                        "StringLike": {
                            "aws:PrincipalArn": [
                                "arn:aws:iam::123456789012:role/ProductionRole*",
                                "arn:aws:iam::123456789012:user/ServiceAccount*"
                            ]
                        }
                    }
                },
                {
                    "Sid": "DenyExternalAccess",
                    "Effect": "Deny",
                    "Principal": "*",
                    "Action": "*",
                    "Resource": "*",
                    "Condition": {
                        "StringNotEquals": {
                            "aws:PrincipalAccount": "123456789012"
                        }
                    }
                }
            ]
        }
        
        return json.dumps(policy, indent=2)
    
    def apply_vpc_endpoint_policy(self, vpc_endpoint_id, policy_document):
        """Apply policy to VPC endpoint"""
        
        response = self.ec2.modify_vpc_endpoint(
            VpcEndpointId=vpc_endpoint_id,
            PolicyDocument=policy_document
        )
        
        return response
    
    def create_lambda_vpc_endpoint_policy(self):
        """Create policy for Lambda VPC endpoint with fine-grained control"""
        
        policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowLambdaInvocation",
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": [
                        "lambda:InvokeFunction",
                        "lambda:GetFunction"
                    ],
                    "Resource": [
                        "arn:aws:lambda:*:123456789012:function:production-*",
                        "arn:aws:lambda:*:123456789012:function:staging-*"
                    ],
                    "Condition": {
                        "StringEquals": {
                            "aws:PrincipalAccount": "123456789012"
                        },
                        "DateGreaterThan": {
                            "aws:CurrentTime": "2024-01-01T00:00:00Z"
                        },
                        "IpAddress": {
                            "aws:SourceIp": [
                                "10.0.0.0/8",
                                "172.16.0.0/12",
                                "192.168.0.0/16"
                            ]
                        }
                    }
                },
                {
                    "Sid": "AllowLambdaListFunctions",
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": [
                        "lambda:ListFunctions",
                        "lambda:ListVersionsByFunction"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "aws:PrincipalAccount": "123456789012"
                        }
                    }
                },
                {
                    "Sid": "DenyDevelopmentAccess",
                    "Effect": "Deny",
                    "Principal": "*",
                    "Action": "*",
                    "Resource": "arn:aws:lambda:*:123456789012:function:dev-*",
                    "Condition": {
                        "StringNotEquals": {
                            "aws:PrincipalTag/Environment": "development"
                        }
                    }
                }
            ]
        }
        
        return json.dumps(policy, indent=2)

# CloudFormation template for VPC endpoints with policies
vpc_endpoint_template = """
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC Endpoints with restrictive policies

Parameters:
  VPCId:
    Type: AWS::EC2::VPC::Id
  PrivateSubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
  AccountId:
    Type: String
    Default: !Ref AWS::AccountId

Resources:
  S3VPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPCId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.s3'
      VpcEndpointType: Gateway
      RouteTableIds:
        - !Ref PrivateRouteTable
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: AllowSSLRequestsOnly
            Effect: Deny
            Principal: '*'
            Action: 's3:*'
            Resource:
              - 'arn:aws:s3:::*/*'
              - 'arn:aws:s3:::*'
            Condition:
              Bool:
                'aws:SecureTransport': 'false'
          - Sid: RestrictToAccountResources
            Effect: Allow
            Principal: '*'
            Action: 's3:*'
            Resource:
              - !Sub 'arn:aws:s3:::${AccountId}-*/*'
              - !Sub 'arn:aws:s3:::${AccountId}-*'

  LambdaVPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VPCId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.lambda'
      VpcEndpointType: Interface
      SubnetIds: !Ref PrivateSubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
      PrivateDnsEnabled: true
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Sid: AllowLambdaAccess
            Effect: Allow
            Principal: '*'
            Action:
              - lambda:InvokeFunction
              - lambda:GetFunction
              - lambda:ListFunctions
            Resource: '*'
            Condition:
              StringEquals:
                'aws:PrincipalAccount': !Ref AccountId

  VPCEndpointSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for VPC endpoints
      VpcId: !Ref VPCId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 10.0.0.0/16
          Description: HTTPS from VPC
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
          Description: HTTPS to AWS services
"""
```

### Gateway VPC Endpoint Policies

```python
def create_dynamodb_vpc_endpoint_policy():
    """Create policy for DynamoDB VPC endpoint"""
    
    policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowDynamoDBAccess",
                "Effect": "Allow",
                "Principal": "*",
                "Action": [
                    "dynamodb:GetItem",
                    "dynamodb:PutItem",
                    "dynamodb:UpdateItem",
                    "dynamodb:DeleteItem",
                    "dynamodb:Query",
                    "dynamodb:Scan",
                    "dynamodb:BatchGetItem",
                    "dynamodb:BatchWriteItem"
                ],
                "Resource": [
                    "arn:aws:dynamodb:us-east-1:123456789012:table/ProductionTable*",
                    "arn:aws:dynamodb:us-east-1:123456789012:table/UserData*"
                ],
                "Condition": {
                    "StringEquals": {
                        "aws:PrincipalAccount": "123456789012"
                    },
                    "ForAllValues:StringLike": {
                        "dynamodb:Attributes": [
                            "UserId",
                            "ProductId",
                            "Timestamp",
                            "Data"
                        ]
                    }
                }
            },
            {
                "Sid": "DenyTableCreation",
                "Effect": "Deny",
                "Principal": "*",
                "Action": [
                    "dynamodb:CreateTable",
                    "dynamodb:DeleteTable",
                    "dynamodb:UpdateTable"
                ],
                "Resource": "*"
            },
            {
                "Sid": "RequireEncryption",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "dynamodb:*",
                "Resource": "*",
                "Condition": {
                    "Null": {
                        "dynamodb:EnclosingOperation": "true"
                    }
                }
            }
        ]
    }
    
    return json.dumps(policy, indent=2)
```

## Resource-Based Policies

### Load Balancer Resource Policies

```python
class LoadBalancerPolicyManager:
    def __init__(self):
        self.elbv2 = boto3.client('elbv2')
        self.wafv2 = boto3.client('wafv2')
    
    def create_alb_access_policy(self):
        """Create resource-based access policy for ALB"""
        
        # ALB doesn't directly support resource policies, but uses security groups
        # and WAF for access control. Here's how to configure comprehensive access control:
        
        security_group_rules = {
            "ingress_rules": [
                {
                    "IpProtocol": "tcp",
                    "FromPort": 443,
                    "ToPort": 443,
                    "CidrIp": "0.0.0.0/0",
                    "Description": "HTTPS from internet"
                },
                {
                    "IpProtocol": "tcp", 
                    "FromPort": 80,
                    "ToPort": 80,
                    "CidrIp": "0.0.0.0/0",
                    "Description": "HTTP redirect to HTTPS"
                }
            ],
            "egress_rules": [
                {
                    "IpProtocol": "tcp",
                    "FromPort": 80,
                    "ToPort": 80,
                    "CidrIp": "10.0.0.0/16",
                    "Description": "HTTP to backend instances"
                },
                {
                    "IpProtocol": "tcp",
                    "FromPort": 443,
                    "ToPort": 443,
                    "CidrIp": "10.0.0.0/16",
                    "Description": "HTTPS to backend instances"
                }
            ]
        }
        
        return security_group_rules
    
    def create_waf_access_policy(self):
        """Create WAF rules for application-layer access control"""
        
        waf_rules = {
            "WebACL": {
                "Name": "ALB-AccessControl",
                "Description": "Access control for ALB",
                "Rules": [
                    {
                        "Name": "IPWhitelistRule",
                        "Priority": 1,
                        "Action": {"Allow": {}},
                        "Statement": {
                            "IPSetReferenceStatement": {
                                "ARN": "arn:aws:wafv2:us-east-1:123456789012:global/ipset/trusted-ips"
                            }
                        }
                    },
                    {
                        "Name": "GeoBlockRule",
                        "Priority": 2,
                        "Action": {"Block": {}},
                        "Statement": {
                            "GeoMatchStatement": {
                                "CountryCodes": ["CN", "RU", "KP"]
                            }
                        }
                    },
                    {
                        "Name": "RateLimitRule",
                        "Priority": 3,
                        "Action": {"Block": {}},
                        "Statement": {
                            "RateBasedStatement": {
                                "Limit": 2000,
                                "AggregateKeyType": "IP"
                            }
                        }
                    }
                ]
            }
        }
        
        return waf_rules

# CloudFormation template for ALB with comprehensive access control
alb_access_control_template = """
AWSTemplateFormatVersion: '2010-09-09'
Description: ALB with comprehensive access control

Resources:
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for ALB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          SourceSecurityGroupId: !Ref CloudFrontSecurityGroup
          Description: HTTPS from CloudFront
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref CloudFrontSecurityGroup
          Description: HTTP from CloudFront
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          DestinationSecurityGroupId: !Ref BackendSecurityGroup
          Description: HTTP to backend
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          DestinationSecurityGroupId: !Ref BackendSecurityGroup
          Description: HTTPS to backend

  WebACL:
    Type: AWS::WAFv2::WebACL
    Properties:
      Scope: REGIONAL
      DefaultAction:
        Allow: {}
      Rules:
        - Name: IPWhitelistRule
          Priority: 1
          Action:
            Allow: {}
          Statement:
            IPSetReferenceStatement:
              Arn: !GetAtt TrustedIPSet.Arn
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: IPWhitelistRule
        
        - Name: AWSManagedRulesCommonRuleSet
          Priority: 2
          OverrideAction:
            None: {}
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesCommonRuleSet
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: CommonRuleSetMetric

  TrustedIPSet:
    Type: AWS::WAFv2::IPSet
    Properties:
      Scope: REGIONAL
      IPAddressVersion: IPV4
      Addresses:
        - 203.0.113.0/24
        - 198.51.100.0/24
      Description: Trusted IP addresses

  WebACLAssociation:
    Type: AWS::WAFv2::WebACLAssociation
    Properties:
      ResourceArn: !Ref ApplicationLoadBalancer
      WebACLArn: !GetAtt WebACL.Arn
"""
```

### API Gateway Resource Policies

```python
def create_api_gateway_resource_policy():
    """Create comprehensive API Gateway resource policy"""
    
    policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowVPCAccess",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "execute-api:Invoke",
                "Resource": "arn:aws:execute-api:us-east-1:123456789012:*/*/*",
                "Condition": {
                    "StringEquals": {
                        "aws:SourceVpce": "vpce-1234567890abcdef0"
                    }
                }
            },
            {
                "Sid": "AllowSpecificIPs",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "execute-api:Invoke",
                "Resource": "arn:aws:execute-api:us-east-1:123456789012:*/prod/GET/public/*",
                "Condition": {
                    "IpAddress": {
                        "aws:SourceIp": [
                            "203.0.113.0/24",
                            "198.51.100.0/24"
                        ]
                    }
                }
            },
            {
                "Sid": "DenyUnauthorizedAccess",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "execute-api:Invoke",
                "Resource": "arn:aws:execute-api:us-east-1:123456789012:*/prod/*/admin/*",
                "Condition": {
                    "Bool": {
                        "aws:SecureTransport": "false"
                    }
                }
            },
            {
                "Sid": "RequireMFAForSensitiveEndpoints",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "execute-api:Invoke",
                "Resource": [
                    "arn:aws:execute-api:us-east-1:123456789012:*/prod/DELETE/*",
                    "arn:aws:execute-api:us-east-1:123456789012:*/prod/POST/users/*/delete"
                ],
                "Condition": {
                    "Bool": {
                        "aws:MultiFactorAuthPresent": "false"
                    }
                }
            },
            {
                "Sid": "TimeBasedAccess",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "execute-api:Invoke",
                "Resource": "arn:aws:execute-api:us-east-1:123456789012:*/prod/*/maintenance/*",
                "Condition": {
                    "DateGreaterThan": {
                        "aws:CurrentTime": "2024-12-31T23:59:59Z"
                    }
                }
            }
        ]
    }
    
    return json.dumps(policy, indent=2)

def create_private_api_policy():
    """Create policy for private API Gateway"""
    
    policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowVPCEndpointAccess",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "execute-api:Invoke",
                "Resource": "*",
                "Condition": {
                    "StringEquals": {
                        "aws:SourceVpce": [
                            "vpce-1234567890abcdef0",
                            "vpce-0987654321fedcba0"
                        ]
                    }
                }
            },
            {
                "Sid": "DenyDirectInternetAccess",
                "Effect": "Deny",
                "Principal": "*",
                "Action": "execute-api:Invoke",
                "Resource": "*",
                "Condition": {
                    "StringNotEquals": {
                        "aws:SourceVpce": [
                            "vpce-1234567890abcdef0",
                            "vpce-0987654321fedcba0"
                        ]
                    }
                }
            }
        ]
    }
    
    return json.dumps(policy, indent=2)
```

## Cross-Account Access Patterns

### Cross-Account VPC Peering

```python
class CrossAccountNetworkingManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.iam = boto3.client('iam')
    
    def create_cross_account_peering_role(self, trusted_account_id):
        """Create IAM role for cross-account VPC peering"""
        
        trust_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": f"arn:aws:iam::{trusted_account_id}:root"
                    },
                    "Action": "sts:AssumeRole",
                    "Condition": {
                        "StringEquals": {
                            "sts:ExternalId": "unique-external-id-12345"
                        },
                        "Bool": {
                            "aws:MultiFactorAuthPresent": "true"
                        }
                    }
                }
            ]
        }
        
        permission_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowVPCPeeringManagement",
                    "Effect": "Allow",
                    "Action": [
                        "ec2:CreateVpcPeeringConnection",
                        "ec2:AcceptVpcPeeringConnection",
                        "ec2:DescribeVpcPeeringConnections",
                        "ec2:DeleteVpcPeeringConnection",
                        "ec2:CreateRoute",
                        "ec2:DeleteRoute",
                        "ec2:DescribeRouteTables"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "ec2:Region": ["us-east-1", "us-west-2"]
                        }
                    }
                },
                {
                    "Sid": "AllowVPCInformation",
                    "Effect": "Allow",
                    "Action": [
                        "ec2:DescribeVpcs",
                        "ec2:DescribeSubnets",
                        "ec2:DescribeAvailabilityZones"
                    ],
                    "Resource": "*"
                }
            ]
        }
        
        # Create role
        role_response = self.iam.create_role(
            RoleName='CrossAccountVPCPeeringRole',
            AssumeRolePolicyDocument=json.dumps(trust_policy),
            Description='Role for cross-account VPC peering operations',
            MaxSessionDuration=3600,
            Tags=[
                {
                    'Key': 'Purpose',
                    'Value': 'CrossAccountNetworking'
                }
            ]
        )
        
        # Attach policy
        self.iam.put_role_policy(
            RoleName='CrossAccountVPCPeeringRole',
            PolicyName='VPCPeeringPermissions',
            PolicyDocument=json.dumps(permission_policy)
        )
        
        return role_response['Role']['Arn']
    
    def setup_cross_account_resource_sharing(self, resource_arn, trusted_accounts):
        """Setup AWS Resource Access Manager for cross-account sharing"""
        
        ram = boto3.client('ram')
        
        # Create resource share
        response = ram.create_resource_share(
            name='NetworkingResourceShare',
            resourceArns=[resource_arn],
            principals=trusted_accounts,
            allowExternalPrincipals=True,
            tags=[
                {
                    'key': 'Purpose',
                    'value': 'CrossAccountNetworking'
                }
            ]
        )
        
        return response['resourceShare']
    
    def create_cross_account_vpc_endpoint_policy(self, allowed_accounts):
        """Create VPC endpoint policy allowing cross-account access"""
        
        policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "AllowCrossAccountAccess",
                    "Effect": "Allow",
                    "Principal": {
                        "AWS": [f"arn:aws:iam::{account}:root" for account in allowed_accounts]
                    },
                    "Action": [
                        "s3:GetObject",
                        "s3:PutObject"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "aws:PrincipalAccount": allowed_accounts
                        },
                        "Bool": {
                            "aws:SecureTransport": "true"
                        }
                    }
                }
            ]
        }
        
        return json.dumps(policy, indent=2)
```

### Transit Gateway Cross-Account Sharing

```yaml
# CloudFormation template for cross-account Transit Gateway sharing
CrossAccountTransitGatewayShare:
  AWSTemplateFormatVersion: '2010-09-09'
  Description: Cross-account Transit Gateway sharing setup
  
  Parameters:
    TrustedAccountIds:
      Type: CommaDelimitedList
      Description: List of trusted account IDs
    
  Resources:
    TransitGateway:
      Type: AWS::EC2::TransitGateway
      Properties:
        AmazonSideAsn: 64512
        Description: Shared Transit Gateway for cross-account connectivity
        DefaultRouteTableAssociation: enable
        DefaultRouteTablePropagation: enable
        DnsSupport: enable
        VpnEcmpSupport: enable
        Tags:
          - Key: Name
            Value: SharedTransitGateway
    
    TransitGatewayResourceShare:
      Type: AWS::RAM::ResourceShare
      Properties:
        Name: TransitGatewayShare
        ResourceArns:
          - !Sub '${TransitGateway}'
        Principals: !Ref TrustedAccountIds
        AllowExternalPrincipals: true
        Tags:
          - Key: Purpose
            Value: CrossAccountNetworking
    
    # IAM role for cross-account TGW management
    CrossAccountTGWRole:
      Type: AWS::IAM::Role
      Properties:
        RoleName: CrossAccountTransitGatewayRole
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                AWS: !Split
                  - ','
                  - !Sub
                    - 'arn:aws:iam::${inner}:root'
                    - inner: !Join
                      - ':root,arn:aws:iam::'
                      - !Ref TrustedAccountIds
              Action: sts:AssumeRole
              Condition:
                StringEquals:
                  'sts:ExternalId': 'tgw-cross-account-access'
        Policies:
          - PolicyName: TransitGatewayAccess
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
                - Effect: Allow
                  Action:
                    - ec2:CreateTransitGatewayVpcAttachment
                    - ec2:DeleteTransitGatewayVpcAttachment
                    - ec2:DescribeTransitGateways
                    - ec2:DescribeTransitGatewayVpcAttachments
                    - ec2:DescribeTransitGatewayRouteTables
                    - ec2:CreateRoute
                    - ec2:DeleteRoute
                  Resource: '*'
                  Condition:
                    StringEquals:
                      'ec2:Region': !Ref AWS::Region
```

## Service-Linked Roles

### Understanding Service-Linked Roles for Networking

```python
class ServiceLinkedRoleManager:
    def __init__(self):
        self.iam = boto3.client('iam')
    
    def list_networking_service_linked_roles(self):
        """List all service-linked roles related to networking"""
        
        networking_services = [
            'elasticloadbalancing.amazonaws.com',
            'spot.amazonaws.com',
            'autoscaling.amazonaws.com',
            'globalaccelerator.amazonaws.com',
            'elasticfilesystem.amazonaws.com'
        ]
        
        service_linked_roles = {}
        
        try:
            roles = self.iam.list_roles(PathPrefix='/aws-service-role/')
            
            for role in roles['Roles']:
                for service in networking_services:
                    if service in role['AssumeRolePolicyDocument']:
                        service_linked_roles[service] = {
                            'role_name': role['RoleName'],
                            'role_arn': role['Arn'],
                            'creation_date': role['CreateDate'],
                            'description': role.get('Description', ''),
                            'max_session_duration': role.get('MaxSessionDuration', 0)
                        }
            
            return service_linked_roles
            
        except Exception as e:
            print(f"Error listing service-linked roles: {str(e)}")
            return {}
    
    def create_elb_service_linked_role(self):
        """Create service-linked role for Elastic Load Balancing"""
        
        try:
            response = self.iam.create_service_linked_role(
                AWSServiceName='elasticloadbalancing.amazonaws.com',
                Description='Service-linked role for Elastic Load Balancing'
            )
            return response['Role']
        except self.iam.exceptions.InvalidInputException as e:
            if 'already exists' in str(e):
                print("Service-linked role already exists")
                return None
            raise
    
    def verify_service_linked_role_permissions(self, role_name):
        """Verify permissions of a service-linked role"""
        
        try:
            # Get role policies
            attached_policies = self.iam.list_attached_role_policies(RoleName=role_name)
            inline_policies = self.iam.list_role_policies(RoleName=role_name)
            
            permissions_summary = {
                'role_name': role_name,
                'attached_policies': [],
                'inline_policies': []
            }
            
            # Get attached policy details
            for policy in attached_policies['AttachedPolicies']:
                policy_details = self.iam.get_policy(PolicyArn=policy['PolicyArn'])
                policy_version = self.iam.get_policy_version(
                    PolicyArn=policy['PolicyArn'],
                    VersionId=policy_details['Policy']['DefaultVersionId']
                )
                
                permissions_summary['attached_policies'].append({
                    'policy_name': policy['PolicyName'],
                    'policy_arn': policy['PolicyArn'],
                    'document': policy_version['PolicyVersion']['Document']
                })
            
            # Get inline policy details
            for policy_name in inline_policies['PolicyNames']:
                policy_document = self.iam.get_role_policy(
                    RoleName=role_name,
                    PolicyName=policy_name
                )
                
                permissions_summary['inline_policies'].append({
                    'policy_name': policy_name,
                    'document': policy_document['PolicyDocument']
                })
            
            return permissions_summary
            
        except Exception as e:
            print(f"Error verifying service-linked role permissions: {str(e)}")
            return None

# Service-linked role monitoring
def monitor_service_linked_roles():
    """Monitor service-linked role usage and compliance"""
    
    manager = ServiceLinkedRoleManager()
    cloudtrail = boto3.client('cloudtrail')
    
    # Get recent CloudTrail events for service-linked role usage
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=7)
    
    events = cloudtrail.lookup_events(
        LookupAttributes=[
            {
                'AttributeKey': 'EventName',
                'AttributeValue': 'AssumeRole'
            }
        ],
        StartTime=start_time,
        EndTime=end_time
    )
    
    service_linked_usage = {}
    
    for event in events['Events']:
        event_detail = json.loads(event['CloudTrailEvent'])
        
        if 'servicelinkedrole' in event_detail.get('requestParameters', {}).get('roleArn', '').lower():
            role_arn = event_detail['requestParameters']['roleArn']
            service = role_arn.split('/')[1].split('.')[0]
            
            if service not in service_linked_usage:
                service_linked_usage[service] = {
                    'assume_count': 0,
                    'last_used': event['EventTime'],
                    'users': set()
                }
            
            service_linked_usage[service]['assume_count'] += 1
            service_linked_usage[service]['users'].add(
                event_detail.get('userIdentity', {}).get('userName', 'Unknown')
            )
    
    # Convert sets to lists for JSON serialization
    for service in service_linked_usage:
        service_linked_usage[service]['users'] = list(service_linked_usage[service]['users'])
    
    return service_linked_usage
```

### Custom Service Roles for Networking

```python
def create_custom_networking_service_role():
    """Create custom service role for networking operations"""
    
    iam = boto3.client('iam')
    
    trust_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": [
                        "lambda.amazonaws.com",
                        "ec2.amazonaws.com"
                    ]
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }
    
    permission_policy = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "NetworkingOperations",
                "Effect": "Allow",
                "Action": [
                    "ec2:DescribeVpcs",
                    "ec2:DescribeSubnets",
                    "ec2:DescribeSecurityGroups",
                    "ec2:DescribeNetworkInterfaces",
                    "ec2:CreateNetworkInterface",
                    "ec2:DeleteNetworkInterface",
                    "ec2:AttachNetworkInterface",
                    "ec2:DetachNetworkInterface",
                    "ec2:ModifyNetworkInterfaceAttribute"
                ],
                "Resource": "*"
            },
            {
                "Sid": "VPCFlowLogs",
                "Effect": "Allow",
                "Action": [
                    "logs:CreateLogGroup",
                    "logs:CreateLogStream",
                    "logs:PutLogEvents",
                    "logs:DescribeLogGroups",
                    "logs:DescribeLogStreams"
                ],
                "Resource": "arn:aws:logs:*:*:*"
            },
            {
                "Sid": "CloudWatchMetrics",
                "Effect": "Allow",
                "Action": [
                    "cloudwatch:PutMetricData",
                    "cloudwatch:GetMetricStatistics",
                    "cloudwatch:ListMetrics"
                ],
                "Resource": "*"
            }
        ]
    }
    
    # Create role
    role_response = iam.create_role(
        RoleName='CustomNetworkingServiceRole',
        AssumeRolePolicyDocument=json.dumps(trust_policy),
        Description='Custom service role for networking operations',
        Tags=[
            {
                'Key': 'Purpose',
                'Value': 'NetworkingAutomation'
            }
        ]
    )
    
    # Attach policy
    iam.put_role_policy(
        RoleName='CustomNetworkingServiceRole',
        PolicyName='NetworkingServicePermissions',
        PolicyDocument=json.dumps(permission_policy)
    )
    
    return role_response['Role']['Arn']
```

## IAM Best Practices for Networking

### Principle of Least Privilege Implementation

```python
class NetworkingIAMBestPractices:
    def __init__(self):
        self.iam = boto3.client('iam')
    
    def create_granular_network_policies(self):
        """Create granular IAM policies for different networking roles"""
        
        policies = {
            'network_viewer': {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "NetworkingReadOnly",
                        "Effect": "Allow",
                        "Action": [
                            "ec2:Describe*",
                            "elasticloadbalancing:Describe*",
                            "route53:List*",
                            "route53:Get*",
                            "directconnect:Describe*",
                            "cloudfront:List*",
                            "cloudfront:Get*"
                        ],
                        "Resource": "*"
                    }
                ]
            },
            'vpc_administrator': {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "VPCFullAccess",
                        "Effect": "Allow",
                        "Action": [
                            "ec2:*Vpc*",
                            "ec2:*Subnet*",
                            "ec2:*RouteTable*",
                            "ec2:*InternetGateway*",
                            "ec2:*NatGateway*",
                            "ec2:*VpcEndpoint*"
                        ],
                        "Resource": "*",
                        "Condition": {
                            "StringEquals": {
                                "aws:RequestedRegion": ["us-east-1", "us-west-2"]
                            }
                        }
                    },
                    {
                        "Sid": "DenyProductionVPCDeletion",
                        "Effect": "Deny",
                        "Action": [
                            "ec2:DeleteVpc",
                            "ec2:DeleteSubnet"
                        ],
                        "Resource": "*",
                        "Condition": {
                            "StringEquals": {
                                "ec2:ResourceTag/Environment": "production"
                            }
                        }
                    }
                ]
            },
            'security_group_manager': {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "SecurityGroupManagement",
                        "Effect": "Allow",
                        "Action": [
                            "ec2:CreateSecurityGroup",
                            "ec2:DeleteSecurityGroup",
                            "ec2:AuthorizeSecurityGroupIngress",
                            "ec2:AuthorizeSecurityGroupEgress",
                            "ec2:RevokeSecurityGroupIngress",
                            "ec2:RevokeSecurityGroupEgress",
                            "ec2:DescribeSecurityGroups"
                        ],
                        "Resource": "*",
                        "Condition": {
                            "StringEquals": {
                                "ec2:ResourceTag/ManagedBy": "${aws:username}"
                            }
                        }
                    },
                    {
                        "Sid": "RequireTagging",
                        "Effect": "Deny",
                        "Action": "ec2:CreateSecurityGroup",
                        "Resource": "*",
                        "Condition": {
                            "Null": {
                                "aws:RequestTag/Owner": "true"
                            }
                        }
                    }
                ]
            }
        }
        
        return policies
    
    def implement_time_based_access(self):
        """Implement time-based access controls for networking resources"""
        
        time_based_policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "BusinessHoursAccess",
                    "Effect": "Allow",
                    "Action": [
                        "ec2:*",
                        "elasticloadbalancing:*"
                    ],
                    "Resource": "*",
                    "Condition": {
                        "DateGreaterThan": {
                            "aws:CurrentTime": "08:00Z"
                        },
                        "DateLessThan": {
                            "aws:CurrentTime": "18:00Z"
                        },
                        "ForAllValues:StringEquals": {
                            "aws:RequestedRegion": ["us-east-1", "us-west-2"]
                        }
                    }
                },
                {
                    "Sid": "EmergencyAccess",
                    "Effect": "Allow",
                    "Action": [
                        "ec2:DescribeInstances",
                        "ec2:DescribeSecurityGroups",
                        "elasticloadbalancing:DescribeLoadBalancers"
                    ],
                    "Resource": "*"
                }
            ]
        }
        
        return time_based_policy
    
    def create_condition_based_policies(self):
        """Create policies with various condition keys for enhanced security"""
        
        condition_policies = {
            'mfa_required_policy': {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "RequireMFAForCriticalActions",
                        "Effect": "Deny",
                        "Action": [
                            "ec2:TerminateInstances",
                            "ec2:DeleteVpc",
                            "ec2:DeleteSubnet",
                            "elasticloadbalancing:DeleteLoadBalancer"
                        ],
                        "Resource": "*",
                        "Condition": {
                            "Bool": {
                                "aws:MultiFactorAuthPresent": "false"
                            }
                        }
                    }
                ]
            },
            'source_ip_restriction': {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "RestrictToOfficeIPs",
                        "Effect": "Deny",
                        "Action": "*",
                        "Resource": "*",
                        "Condition": {
                            "IpAddressIfExists": {
                                "aws:SourceIp": "203.0.113.0/24"
                            },
                            "Bool": {
                                "aws:ViaAWSService": "false"
                            }
                        }
                    }
                ]
            },
            'resource_tag_enforcement': {
                "Version": "2012-10-17",
                "Statement": [
                    {
                        "Sid": "EnforceResourceTagging",
                        "Effect": "Deny",
                        "Action": [
                            "ec2:CreateVpc",
                            "ec2:CreateSubnet",
                            "ec2:CreateSecurityGroup"
                        ],
                        "Resource": "*",
                        "Condition": {
                            "Null": {
                                "aws:RequestTag/Environment": "true"
                            }
                        }
                    }
                ]
            }
        }
        
        return condition_policies
```

This comprehensive coverage of IAM and resource policies for networking provides the foundation for secure, controlled access to AWS networking resources while maintaining operational flexibility and compliance requirements.