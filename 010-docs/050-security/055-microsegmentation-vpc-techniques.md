# Topic 55: Microsegmentation VPC Techniques

## Introduction

Microsegmentation is a security technique that creates secure zones within a network to isolate workloads and limit the blast radius of potential security breaches. In AWS VPC environments, microsegmentation leverages multiple layers including security groups, Network ACLs, subnets, and application-level controls to create granular security boundaries. This topic provides comprehensive coverage of microsegmentation strategies, implementation techniques, and compliance patterns.

## Fundamentals of VPC Microsegmentation

### Microsegmentation Architecture Layers

```
VPC Microsegmentation Stack:

Application Layer:
├── Container Security Groups
├── Service Mesh Security
├── API Gateway Policies
└── Lambda Function Isolation

Network Layer:
├── Security Groups (Instance Level)
├── Network ACLs (Subnet Level)
├── VPC Endpoints (Service Level)
└── AWS Network Firewall (VPC Level)

Infrastructure Layer:
├── Subnet Segmentation
├── Route Table Isolation
├── VPC Peering Controls
└── Transit Gateway Policies

Data Layer:
├── S3 Bucket Policies
├── Database Security Groups
├── Encryption Boundaries
└── Access Control Lists
```

### Security Groups as Primary Microsegmentation Tool

```python
import boto3
import json
from datetime import datetime

class MicrosegmentationManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.elbv2 = boto3.client('elbv2')
        self.rds = boto3.client('rds')
    
    def create_tiered_security_groups(self, vpc_id, application_name):
        """Create comprehensive tiered security group architecture"""
        
        # Define security group tiers with specific purposes
        security_group_tiers = {
            'alb': {
                'name': f'{application_name}-alb-sg',
                'description': f'ALB security group for {application_name}',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 80, 'source_type': 'cidr', 'source': '0.0.0.0/0'},
                    {'protocol': 'tcp', 'port': 443, 'source_type': 'cidr', 'source': '0.0.0.0/0'}
                ],
                'egress_rules': [
                    {'protocol': 'tcp', 'port': 8080, 'dest_type': 'sg', 'dest': 'web'}
                ]
            },
            'web': {
                'name': f'{application_name}-web-sg',
                'description': f'Web tier security group for {application_name}',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 8080, 'source_type': 'sg', 'source': 'alb'},
                    {'protocol': 'tcp', 'port': 22, 'source_type': 'sg', 'source': 'bastion'}
                ],
                'egress_rules': [
                    {'protocol': 'tcp', 'port': 8080, 'dest_type': 'sg', 'dest': 'app'},
                    {'protocol': 'tcp', 'port': 443, 'dest_type': 'cidr', 'dest': '0.0.0.0/0'}
                ]
            },
            'app': {
                'name': f'{application_name}-app-sg',
                'description': f'Application tier security group for {application_name}',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 8080, 'source_type': 'sg', 'source': 'web'},
                    {'protocol': 'tcp', 'port': 22, 'source_type': 'sg', 'source': 'bastion'}
                ],
                'egress_rules': [
                    {'protocol': 'tcp', 'port': 5432, 'dest_type': 'sg', 'dest': 'db'},
                    {'protocol': 'tcp', 'port': 443, 'dest_type': 'cidr', 'dest': '0.0.0.0/0'},
                    {'protocol': 'tcp', 'port': 6379, 'dest_type': 'sg', 'dest': 'cache'}
                ]
            },
            'db': {
                'name': f'{application_name}-db-sg',
                'description': f'Database tier security group for {application_name}',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 5432, 'source_type': 'sg', 'source': 'app'},
                    {'protocol': 'tcp', 'port': 22, 'source_type': 'sg', 'source': 'bastion'}
                ],
                'egress_rules': []  # Minimal egress for databases
            },
            'cache': {
                'name': f'{application_name}-cache-sg',
                'description': f'Cache tier security group for {application_name}',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 6379, 'source_type': 'sg', 'source': 'app'}
                ],
                'egress_rules': []
            },
            'bastion': {
                'name': f'{application_name}-bastion-sg',
                'description': f'Bastion host security group for {application_name}',
                'ingress_rules': [
                    {'protocol': 'tcp', 'port': 22, 'source_type': 'cidr', 'source': '203.0.113.0/24'}  # Office IP
                ],
                'egress_rules': [
                    {'protocol': 'tcp', 'port': 22, 'dest_type': 'sg', 'dest': 'web'},
                    {'protocol': 'tcp', 'port': 22, 'dest_type': 'sg', 'dest': 'app'},
                    {'protocol': 'tcp', 'port': 22, 'dest_type': 'sg', 'dest': 'db'}
                ]
            }
        }
        
        # Create security groups
        created_sgs = {}
        for tier, config in security_group_tiers.items():
            sg_response = self.ec2.create_security_group(
                GroupName=config['name'],
                Description=config['description'],
                VpcId=vpc_id,
                TagSpecifications=[
                    {
                        'ResourceType': 'security-group',
                        'Tags': [
                            {'Key': 'Name', 'Value': config['name']},
                            {'Key': 'Application', 'Value': application_name},
                            {'Key': 'Tier', 'Value': tier},
                            {'Key': 'Purpose', 'Value': 'Microsegmentation'},
                            {'Key': 'CreatedBy', 'Value': 'MicrosegmentationManager'}
                        ]
                    }
                ]
            )
            created_sgs[tier] = sg_response['GroupId']
        
        # Configure security group rules after all groups are created
        self._configure_security_group_rules(created_sgs, security_group_tiers)
        
        return created_sgs
    
    def _configure_security_group_rules(self, security_groups, tier_configs):
        """Configure security group rules with proper references"""
        
        for tier, sg_id in security_groups.items():
            tier_config = tier_configs[tier]
            
            # Configure ingress rules
            for rule in tier_config['ingress_rules']:
                if rule['source_type'] == 'sg':
                    # Reference another security group
                    source_sg_id = security_groups[rule['source']]
                    ip_permission = {
                        'IpProtocol': rule['protocol'],
                        'FromPort': rule['port'],
                        'ToPort': rule['port'],
                        'UserIdGroupPairs': [{'GroupId': source_sg_id}]
                    }
                else:
                    # CIDR block source
                    ip_permission = {
                        'IpProtocol': rule['protocol'],
                        'FromPort': rule['port'],
                        'ToPort': rule['port'],
                        'IpRanges': [{'CidrIp': rule['source']}]
                    }
                
                try:
                    self.ec2.authorize_security_group_ingress(
                        GroupId=sg_id,
                        IpPermissions=[ip_permission]
                    )
                except Exception as e:
                    print(f"Error adding ingress rule to {tier}: {e}")
            
            # Remove default egress rule (allow all outbound)
            try:
                self.ec2.revoke_security_group_egress(
                    GroupId=sg_id,
                    IpPermissions=[
                        {
                            'IpProtocol': '-1',
                            'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                        }
                    ]
                )
            except:
                pass  # Rule might not exist
            
            # Configure specific egress rules
            for rule in tier_config['egress_rules']:
                if rule['dest_type'] == 'sg':
                    # Reference another security group
                    dest_sg_id = security_groups[rule['dest']]
                    ip_permission = {
                        'IpProtocol': rule['protocol'],
                        'FromPort': rule['port'],
                        'ToPort': rule['port'],
                        'UserIdGroupPairs': [{'GroupId': dest_sg_id}]
                    }
                else:
                    # CIDR block destination
                    ip_permission = {
                        'IpProtocol': rule['protocol'],
                        'FromPort': rule['port'],
                        'ToPort': rule['port'],
                        'IpRanges': [{'CidrIp': rule['dest']}]
                    }
                
                try:
                    self.ec2.authorize_security_group_egress(
                        GroupId=sg_id,
                        IpPermissions=[ip_permission]
                    )
                except Exception as e:
                    print(f"Error adding egress rule to {tier}: {e}")
    
    def create_service_specific_security_groups(self, vpc_id, service_map):
        """Create service-specific security groups for microservices"""
        
        service_security_groups = {}
        
        for service_name, service_config in service_map.items():
            sg_name = f'{service_name}-service-sg'
            
            # Create security group
            sg_response = self.ec2.create_security_group(
                GroupName=sg_name,
                Description=f'Security group for {service_name} microservice',
                VpcId=vpc_id,
                TagSpecifications=[
                    {
                        'ResourceType': 'security-group',
                        'Tags': [
                            {'Key': 'Name', 'Value': sg_name},
                            {'Key': 'Service', 'Value': service_name},
                            {'Key': 'Type', 'Value': 'Microservice'},
                            {'Key': 'Purpose', 'Value': 'ServiceMicrosegmentation'}
                        ]
                    }
                ]
            )
            
            service_security_groups[service_name] = sg_response['GroupId']
        
        # Configure service-to-service communication rules
        self._configure_service_communication_rules(service_security_groups, service_map)
        
        return service_security_groups
    
    def _configure_service_communication_rules(self, security_groups, service_map):
        """Configure service-to-service communication rules"""
        
        for service_name, service_config in service_map.items():
            sg_id = security_groups[service_name]
            
            # Remove default egress
            try:
                self.ec2.revoke_security_group_egress(
                    GroupId=sg_id,
                    IpPermissions=[
                        {
                            'IpProtocol': '-1',
                            'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                        }
                    ]
                )
            except:
                pass
            
            # Configure ingress rules for this service
            for port in service_config.get('ports', []):
                # Allow access from dependent services
                for dependent_service in service_config.get('allowed_clients', []):
                    if dependent_service in security_groups:
                        try:
                            self.ec2.authorize_security_group_ingress(
                                GroupId=sg_id,
                                IpPermissions=[
                                    {
                                        'IpProtocol': 'tcp',
                                        'FromPort': port,
                                        'ToPort': port,
                                        'UserIdGroupPairs': [
                                            {'GroupId': security_groups[dependent_service]}
                                        ]
                                    }
                                ]
                            )
                        except Exception as e:
                            print(f"Error configuring ingress for {service_name}: {e}")
            
            # Configure egress rules for this service
            for dependency in service_config.get('dependencies', []):
                if dependency['service'] in security_groups:
                    try:
                        self.ec2.authorize_security_group_egress(
                            GroupId=sg_id,
                            IpPermissions=[
                                {
                                    'IpProtocol': 'tcp',
                                    'FromPort': dependency['port'],
                                    'ToPort': dependency['port'],
                                    'UserIdGroupPairs': [
                                        {'GroupId': security_groups[dependency['service']]}
                                    ]
                                }
                            ]
                        )
                    except Exception as e:
                        print(f"Error configuring egress for {service_name}: {e}")
            
            # Allow HTTPS egress for external APIs (if configured)
            if service_config.get('external_access', False):
                try:
                    self.ec2.authorize_security_group_egress(
                        GroupId=sg_id,
                        IpPermissions=[
                            {
                                'IpProtocol': 'tcp',
                                'FromPort': 443,
                                'ToPort': 443,
                                'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                            }
                        ]
                    )
                except Exception as e:
                    print(f"Error configuring external egress for {service_name}: {e}")

# Example service map for microservices
microservices_map = {
    'user-service': {
        'ports': [8080],
        'allowed_clients': ['api-gateway', 'auth-service'],
        'dependencies': [
            {'service': 'user-database', 'port': 5432},
            {'service': 'cache-service', 'port': 6379}
        ],
        'external_access': True
    },
    'auth-service': {
        'ports': [8081],
        'allowed_clients': ['api-gateway', 'user-service'],
        'dependencies': [
            {'service': 'auth-database', 'port': 5432}
        ],
        'external_access': True
    },
    'api-gateway': {
        'ports': [8080, 8443],
        'allowed_clients': ['load-balancer'],
        'dependencies': [
            {'service': 'user-service', 'port': 8080},
            {'service': 'auth-service', 'port': 8081},
            {'service': 'order-service', 'port': 8082}
        ],
        'external_access': True
    },
    'user-database': {
        'ports': [5432],
        'allowed_clients': ['user-service'],
        'dependencies': [],
        'external_access': False
    },
    'cache-service': {
        'ports': [6379],
        'allowed_clients': ['user-service', 'order-service'],
        'dependencies': [],
        'external_access': False
    }
}
```

## Network ACLs for Subnet-Level Segmentation

### Advanced NACL Configuration

```python
class NetworkACLManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
    
    def create_microsegmentation_nacls(self, vpc_id, subnet_mapping):
        """Create Network ACLs for subnet-level microsegmentation"""
        
        nacl_configurations = {
            'dmz': {
                'name': 'DMZ-NACL',
                'description': 'NACL for DMZ subnet',
                'inbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'action': 'allow', 'port_range': (80, 80), 'cidr': '0.0.0.0/0'},
                    {'rule_number': 110, 'protocol': 'tcp', 'action': 'allow', 'port_range': (443, 443), 'cidr': '0.0.0.0/0'},
                    {'rule_number': 120, 'protocol': 'tcp', 'action': 'allow', 'port_range': (1024, 65535), 'cidr': '0.0.0.0/0'},
                    {'rule_number': 130, 'protocol': 'tcp', 'action': 'allow', 'port_range': (22, 22), 'cidr': '203.0.113.0/24'}
                ],
                'outbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'action': 'allow', 'port_range': (8080, 8080), 'cidr': '10.0.2.0/24'},
                    {'rule_number': 110, 'protocol': 'tcp', 'action': 'allow', 'port_range': (1024, 65535), 'cidr': '0.0.0.0/0'},
                    {'rule_number': 120, 'protocol': 'tcp', 'action': 'allow', 'port_range': (443, 443), 'cidr': '0.0.0.0/0'}
                ]
            },
            'app': {
                'name': 'App-NACL',
                'description': 'NACL for application subnet',
                'inbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'action': 'allow', 'port_range': (8080, 8080), 'cidr': '10.0.1.0/24'},
                    {'rule_number': 110, 'protocol': 'tcp', 'action': 'allow', 'port_range': (1024, 65535), 'cidr': '0.0.0.0/0'},
                    {'rule_number': 120, 'protocol': 'tcp', 'action': 'allow', 'port_range': (22, 22), 'cidr': '10.0.1.0/24'}
                ],
                'outbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'action': 'allow', 'port_range': (5432, 5432), 'cidr': '10.0.3.0/24'},
                    {'rule_number': 110, 'protocol': 'tcp', 'action': 'allow', 'port_range': (6379, 6379), 'cidr': '10.0.4.0/24'},
                    {'rule_number': 120, 'protocol': 'tcp', 'action': 'allow', 'port_range': (1024, 65535), 'cidr': '0.0.0.0/0'},
                    {'rule_number': 130, 'protocol': 'tcp', 'action': 'allow', 'port_range': (443, 443), 'cidr': '0.0.0.0/0'}
                ]
            },
            'db': {
                'name': 'Database-NACL',
                'description': 'NACL for database subnet',
                'inbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'action': 'allow', 'port_range': (5432, 5432), 'cidr': '10.0.2.0/24'},
                    {'rule_number': 110, 'protocol': 'tcp', 'action': 'allow', 'port_range': (22, 22), 'cidr': '10.0.1.0/24'},
                    {'rule_number': 120, 'protocol': 'tcp', 'action': 'allow', 'port_range': (1024, 65535), 'cidr': '10.0.2.0/24'}
                ],
                'outbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'action': 'allow', 'port_range': (1024, 65535), 'cidr': '10.0.2.0/24'},
                    {'rule_number': 110, 'protocol': 'tcp', 'action': 'allow', 'port_range': (443, 443), 'cidr': '0.0.0.0/0'}
                ]
            },
            'management': {
                'name': 'Management-NACL',
                'description': 'NACL for management subnet',
                'inbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'action': 'allow', 'port_range': (22, 22), 'cidr': '203.0.113.0/24'},
                    {'rule_number': 110, 'protocol': 'tcp', 'action': 'allow', 'port_range': (3389, 3389), 'cidr': '203.0.113.0/24'},
                    {'rule_number': 120, 'protocol': 'tcp', 'action': 'allow', 'port_range': (1024, 65535), 'cidr': '0.0.0.0/0'}
                ],
                'outbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'action': 'allow', 'port_range': (22, 22), 'cidr': '10.0.0.0/16'},
                    {'rule_number': 110, 'protocol': 'tcp', 'action': 'allow', 'port_range': (3389, 3389), 'cidr': '10.0.0.0/16'},
                    {'rule_number': 120, 'protocol': 'tcp', 'action': 'allow', 'port_range': (443, 443), 'cidr': '0.0.0.0/0'},
                    {'rule_number': 130, 'protocol': 'tcp', 'action': 'allow', 'port_range': (1024, 65535), 'cidr': '0.0.0.0/0'}
                ]
            }
        }
        
        created_nacls = {}
        
        for nacl_type, config in nacl_configurations.items():
            # Create Network ACL
            nacl_response = self.ec2.create_network_acl(
                VpcId=vpc_id,
                TagSpecifications=[
                    {
                        'ResourceType': 'network-acl',
                        'Tags': [
                            {'Key': 'Name', 'Value': config['name']},
                            {'Key': 'Type', 'Value': nacl_type},
                            {'Key': 'Purpose', 'Value': 'Microsegmentation'}
                        ]
                    }
                ]
            )
            
            nacl_id = nacl_response['NetworkAcl']['NetworkAclId']
            created_nacls[nacl_type] = nacl_id
            
            # Configure inbound rules
            for rule in config['inbound_rules']:
                self._create_nacl_entry(nacl_id, rule, egress=False)
            
            # Configure outbound rules
            for rule in config['outbound_rules']:
                self._create_nacl_entry(nacl_id, rule, egress=True)
            
            # Associate with subnet if mapping provided
            if nacl_type in subnet_mapping:
                for subnet_id in subnet_mapping[nacl_type]:
                    try:
                        self.ec2.replace_network_acl_association(
                            AssociationId=self._get_nacl_association_id(subnet_id),
                            NetworkAclId=nacl_id
                        )
                    except Exception as e:
                        print(f"Error associating NACL with subnet {subnet_id}: {e}")
        
        return created_nacls
    
    def _create_nacl_entry(self, nacl_id, rule, egress=False):
        """Create individual NACL entry"""
        
        try:
            if rule['port_range'][0] == rule['port_range'][1]:
                # Single port
                port_range = {
                    'From': rule['port_range'][0],
                    'To': rule['port_range'][1]
                }
            else:
                # Port range
                port_range = {
                    'From': rule['port_range'][0],
                    'To': rule['port_range'][1]
                }
            
            self.ec2.create_network_acl_entry(
                NetworkAclId=nacl_id,
                RuleNumber=rule['rule_number'],
                Protocol=self._get_protocol_number(rule['protocol']),
                RuleAction=rule['action'],
                CidrBlock=rule['cidr'],
                Egress=egress,
                PortRange=port_range
            )
        except Exception as e:
            print(f"Error creating NACL entry: {e}")
    
    def _get_protocol_number(self, protocol):
        """Convert protocol name to number"""
        protocol_map = {
            'tcp': '6',
            'udp': '17',
            'icmp': '1',
            'all': '-1'
        }
        return protocol_map.get(protocol.lower(), '6')
    
    def _get_nacl_association_id(self, subnet_id):
        """Get current NACL association ID for subnet"""
        try:
            response = self.ec2.describe_network_acls(
                Filters=[
                    {
                        'Name': 'association.subnet-id',
                        'Values': [subnet_id]
                    }
                ]
            )
            
            for nacl in response['NetworkAcls']:
                for association in nacl['Associations']:
                    if association['SubnetId'] == subnet_id:
                        return association['NetworkAclAssociationId']
        except Exception as e:
            print(f"Error getting NACL association: {e}")
            return None

# CloudFormation template for comprehensive NACL setup
nacl_microsegmentation_template = """
AWSTemplateFormatVersion: '2010-09-09'
Description: Network ACL-based microsegmentation

Parameters:
  VPCId:
    Type: AWS::EC2::VPC::Id
    Description: VPC ID for NACL creation
  
  DMZSubnetId:
    Type: AWS::EC2::Subnet::Id
    Description: DMZ subnet ID
  
  AppSubnetId:
    Type: AWS::EC2::Subnet::Id
    Description: Application subnet ID
  
  DatabaseSubnetId:
    Type: AWS::EC2::Subnet::Id
    Description: Database subnet ID

Resources:
  # DMZ Network ACL
  DMZNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !Ref VPCId
      Tags:
        - Key: Name
          Value: DMZ-NACL
        - Key: Purpose
          Value: Microsegmentation

  # DMZ Inbound Rules
  DMZInboundHTTP:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DMZNACL
      RuleNumber: 100
      Protocol: 6
      RuleAction: allow
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 80
        To: 80

  DMZInboundHTTPS:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DMZNACL
      RuleNumber: 110
      Protocol: 6
      RuleAction: allow
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 443
        To: 443

  DMZInboundReturn:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DMZNACL
      RuleNumber: 120
      Protocol: 6
      RuleAction: allow
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 1024
        To: 65535

  # DMZ Outbound Rules
  DMZOutboundApp:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DMZNACL
      RuleNumber: 100
      Protocol: 6
      RuleAction: allow
      Egress: true
      CidrBlock: 10.0.2.0/24
      PortRange:
        From: 8080
        To: 8080

  DMZOutboundReturn:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref DMZNACL
      RuleNumber: 110
      Protocol: 6
      RuleAction: allow
      Egress: true
      CidrBlock: 0.0.0.0/0
      PortRange:
        From: 1024
        To: 65535

  # Application Network ACL
  AppNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !Ref VPCId
      Tags:
        - Key: Name
          Value: App-NACL
        - Key: Purpose
          Value: Microsegmentation

  # Database Network ACL
  DatabaseNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !Ref VPCId
      Tags:
        - Key: Name
          Value: Database-NACL
        - Key: Purpose
          Value: Microsegmentation

  # NACL Associations
  DMZNACLAssociation:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !Ref DMZSubnetId
      NetworkAclId: !Ref DMZNACL

  AppNACLAssociation:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !Ref AppSubnetId
      NetworkAclId: !Ref AppNACL

  DatabaseNACLAssociation:
    Type: AWS::EC2::SubnetNetworkAclAssociation
    Properties:
      SubnetId: !Ref DatabaseSubnetId
      NetworkAclId: !Ref DatabaseNACL

Outputs:
  DMZNACLId:
    Description: DMZ Network ACL ID
    Value: !Ref DMZNACL
    Export:
      Name: !Sub '${AWS::StackName}-DMZNACL'

  AppNACLId:
    Description: Application Network ACL ID
    Value: !Ref AppNACL
    Export:
      Name: !Sub '${AWS::StackName}-AppNACL'

  DatabaseNACLId:
    Description: Database Network ACL ID
    Value: !Ref DatabaseNACL
    Export:
      Name: !Sub '${AWS::StackName}-DatabaseNACL'
"""
```

## Subnet-Based Segmentation Strategies

### Subnet Design for Microsegmentation

```python
class SubnetMicrosegmentationManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
    
    def design_microsegmented_subnets(self, vpc_cidr, availability_zones):
        """Design subnet architecture for comprehensive microsegmentation"""
        
        import ipaddress
        
        vpc_network = ipaddress.IPv4Network(vpc_cidr)
        
        # Define subnet tiers with specific purposes
        subnet_tiers = {
            'public-dmz': {
                'purpose': 'Load balancers and internet-facing resources',
                'size': '/26',  # 62 usable IPs
                'public': True,
                'zones': len(availability_zones)
            },
            'private-web': {
                'purpose': 'Web tier application servers',
                'size': '/25',  # 126 usable IPs
                'public': False,
                'zones': len(availability_zones)
            },
            'private-app': {
                'purpose': 'Application tier microservices',
                'size': '/24',  # 254 usable IPs
                'public': False,
                'zones': len(availability_zones)
            },
            'private-data': {
                'purpose': 'Database and data storage',
                'size': '/26',  # 62 usable IPs
                'public': False,
                'zones': len(availability_zones)
            },
            'private-cache': {
                'purpose': 'Caching layer (Redis, Memcached)',
                'size': '/27',  # 30 usable IPs
                'public': False,
                'zones': len(availability_zones)
            },
            'private-mgmt': {
                'purpose': 'Management and monitoring tools',
                'size': '/27',  # 30 usable IPs
                'public': False,
                'zones': 1  # Only need one management subnet
            },
            'private-shared': {
                'purpose': 'Shared services (DNS, AD, etc.)',
                'size': '/27',  # 30 usable IPs
                'public': False,
                'zones': 2  # Redundancy for shared services
            }
        }
        
        # Calculate subnet allocations
        subnet_allocations = self._calculate_subnet_allocations(
            vpc_network, subnet_tiers, availability_zones
        )
        
        return subnet_allocations
    
    def _calculate_subnet_allocations(self, vpc_network, subnet_tiers, availability_zones):
        """Calculate specific subnet CIDR blocks"""
        
        allocations = {}
        current_network = vpc_network.network_address
        
        for tier_name, tier_config in subnet_tiers.items():
            tier_allocations = []
            subnet_size = int(tier_config['size'].replace('/', ''))
            
            for az_index in range(tier_config['zones']):
                # Calculate subnet CIDR
                subnet_cidr = f"{current_network}/{subnet_size}"
                
                tier_allocations.append({
                    'cidr': subnet_cidr,
                    'availability_zone': availability_zones[az_index % len(availability_zones)],
                    'purpose': tier_config['purpose'],
                    'public': tier_config['public']
                })
                
                # Move to next subnet
                subnet_network = ipaddress.IPv4Network(subnet_cidr)
                current_network = subnet_network.broadcast_address + 1
            
            allocations[tier_name] = tier_allocations
        
        return allocations
    
    def create_microsegmented_subnets(self, vpc_id, subnet_allocations):
        """Create subnets based on microsegmentation design"""
        
        created_subnets = {}
        
        for tier_name, tier_subnets in subnet_allocations.items():
            tier_subnet_ids = []
            
            for subnet_config in tier_subnets:
                # Create subnet
                subnet_response = self.ec2.create_subnet(
                    VpcId=vpc_id,
                    CidrBlock=subnet_config['cidr'],
                    AvailabilityZone=subnet_config['availability_zone'],
                    TagSpecifications=[
                        {
                            'ResourceType': 'subnet',
                            'Tags': [
                                {'Key': 'Name', 'Value': f"{tier_name}-{subnet_config['availability_zone']}"},
                                {'Key': 'Tier', 'Value': tier_name},
                                {'Key': 'Purpose', 'Value': subnet_config['purpose']},
                                {'Key': 'Public', 'Value': str(subnet_config['public'])},
                                {'Key': 'Microsegmentation', 'Value': 'true'}
                            ]
                        }
                    ]
                )
                
                subnet_id = subnet_response['Subnet']['SubnetId']
                tier_subnet_ids.append(subnet_id)
                
                # Configure public IP assignment for public subnets
                if subnet_config['public']:
                    self.ec2.modify_subnet_attribute(
                        SubnetId=subnet_id,
                        MapPublicIpOnLaunch={'Value': True}
                    )
            
            created_subnets[tier_name] = tier_subnet_ids
        
        return created_subnets
    
    def create_microsegmentation_route_tables(self, vpc_id, subnet_mapping, igw_id=None, nat_gw_ids=None):
        """Create route tables for subnet-level microsegmentation"""
        
        route_tables = {}
        
        for tier_name, subnet_ids in subnet_mapping.items():
            # Create route table for this tier
            rt_response = self.ec2.create_route_table(
                VpcId=vpc_id,
                TagSpecifications=[
                    {
                        'ResourceType': 'route-table',
                        'Tags': [
                            {'Key': 'Name', 'Value': f"{tier_name}-rt"},
                            {'Key': 'Tier', 'Value': tier_name},
                            {'Key': 'Purpose', 'Value': 'Microsegmentation'}
                        ]
                    }
                ]
            )
            
            route_table_id = rt_response['RouteTable']['RouteTableId']
            route_tables[tier_name] = route_table_id
            
            # Configure routes based on tier type
            if 'public' in tier_name and igw_id:
                # Public subnets get internet gateway route
                self.ec2.create_route(
                    RouteTableId=route_table_id,
                    DestinationCidrBlock='0.0.0.0/0',
                    GatewayId=igw_id
                )
            elif 'private' in tier_name and nat_gw_ids:
                # Private subnets get NAT gateway route
                # Use different NAT gateways for different AZs if available
                nat_gw_id = nat_gw_ids[0]  # Simplified - use first NAT gateway
                self.ec2.create_route(
                    RouteTableId=route_table_id,
                    DestinationCidrBlock='0.0.0.0/0',
                    NatGatewayId=nat_gw_id
                )
            
            # Associate subnets with route table
            for subnet_id in subnet_ids:
                self.ec2.associate_route_table(
                    RouteTableId=route_table_id,
                    SubnetId=subnet_id
                )
        
        return route_tables
```

## Application-Level Segmentation

### Container and ECS Microsegmentation

```python
class ContainerMicrosegmentation:
    def __init__(self):
        self.ecs = boto3.client('ecs')
        self.ec2 = boto3.client('ec2')
        self.elbv2 = boto3.client('elbv2')
    
    def create_ecs_service_segmentation(self, cluster_name, vpc_id, private_subnet_ids):
        """Create ECS service-level microsegmentation"""
        
        # Define service segmentation architecture
        service_definitions = {
            'frontend-service': {
                'image': 'nginx:latest',
                'cpu': 256,
                'memory': 512,
                'port': 80,
                'security_groups': ['frontend-sg'],
                'target_group_port': 80,
                'health_check_path': '/health'
            },
            'api-service': {
                'image': 'api:latest',
                'cpu': 512,
                'memory': 1024,
                'port': 8080,
                'security_groups': ['api-sg'],
                'target_group_port': 8080,
                'health_check_path': '/api/health'
            },
            'auth-service': {
                'image': 'auth:latest',
                'cpu': 256,
                'memory': 512,
                'port': 8081,
                'security_groups': ['auth-sg'],
                'target_group_port': 8081,
                'health_check_path': '/auth/health'
            },
            'data-service': {
                'image': 'data:latest',
                'cpu': 512,
                'memory': 1024,
                'port': 8082,
                'security_groups': ['data-sg'],
                'target_group_port': 8082,
                'health_check_path': '/data/health'
            }
        }
        
        # Create security groups for each service
        service_security_groups = self._create_service_security_groups(vpc_id, service_definitions)
        
        # Create target groups for each service
        target_groups = self._create_service_target_groups(vpc_id, service_definitions)
        
        # Create ECS task definitions and services
        ecs_services = self._create_ecs_services(
            cluster_name, service_definitions, service_security_groups, 
            target_groups, private_subnet_ids
        )
        
        return {
            'security_groups': service_security_groups,
            'target_groups': target_groups,
            'ecs_services': ecs_services
        }
    
    def _create_service_security_groups(self, vpc_id, service_definitions):
        """Create security groups for each microservice"""
        
        security_groups = {}
        
        # Define service communication matrix
        service_communication = {
            'frontend-service': ['api-service'],
            'api-service': ['auth-service', 'data-service'],
            'auth-service': [],  # No outbound service calls
            'data-service': []   # No outbound service calls
        }
        
        # Create security groups
        for service_name, service_config in service_definitions.items():
            sg_name = f"{service_name}-sg"
            
            sg_response = self.ec2.create_security_group(
                GroupName=sg_name,
                Description=f"Security group for {service_name}",
                VpcId=vpc_id,
                TagSpecifications=[
                    {
                        'ResourceType': 'security-group',
                        'Tags': [
                            {'Key': 'Name', 'Value': sg_name},
                            {'Key': 'Service', 'Value': service_name},
                            {'Key': 'Purpose', 'Value': 'ContainerMicrosegmentation'}
                        ]
                    }
                ]
            )
            
            security_groups[service_name] = sg_response['GroupId']
        
        # Configure service-to-service communication rules
        for service_name, allowed_destinations in service_communication.items():
            source_sg_id = security_groups[service_name]
            
            # Remove default egress rule
            try:
                self.ec2.revoke_security_group_egress(
                    GroupId=source_sg_id,
                    IpPermissions=[
                        {
                            'IpProtocol': '-1',
                            'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                        }
                    ]
                )
            except:
                pass
            
            # Add specific egress rules for allowed services
            for dest_service in allowed_destinations:
                dest_sg_id = security_groups[dest_service]
                dest_port = service_definitions[dest_service]['port']
                
                # Add egress rule to source
                self.ec2.authorize_security_group_egress(
                    GroupId=source_sg_id,
                    IpPermissions=[
                        {
                            'IpProtocol': 'tcp',
                            'FromPort': dest_port,
                            'ToPort': dest_port,
                            'UserIdGroupPairs': [{'GroupId': dest_sg_id}]
                        }
                    ]
                )
                
                # Add ingress rule to destination
                self.ec2.authorize_security_group_ingress(
                    GroupId=dest_sg_id,
                    IpPermissions=[
                        {
                            'IpProtocol': 'tcp',
                            'FromPort': dest_port,
                            'ToPort': dest_port,
                            'UserIdGroupPairs': [{'GroupId': source_sg_id}]
                        }
                    ]
                )
            
            # Allow HTTPS egress for external API calls (if needed)
            if service_name in ['api-service', 'auth-service']:
                self.ec2.authorize_security_group_egress(
                    GroupId=source_sg_id,
                    IpPermissions=[
                        {
                            'IpProtocol': 'tcp',
                            'FromPort': 443,
                            'ToPort': 443,
                            'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                        }
                    ]
                )
        
        return security_groups
    
    def _create_service_target_groups(self, vpc_id, service_definitions):
        """Create Application Load Balancer target groups for services"""
        
        target_groups = {}
        
        for service_name, service_config in service_definitions.items():
            tg_response = self.elbv2.create_target_group(
                Name=f"{service_name}-tg",
                Protocol='HTTP',
                Port=service_config['target_group_port'],
                VpcId=vpc_id,
                HealthCheckProtocol='HTTP',
                HealthCheckPath=service_config['health_check_path'],
                HealthCheckIntervalSeconds=30,
                HealthCheckTimeoutSeconds=5,
                HealthyThresholdCount=2,
                UnhealthyThresholdCount=3,
                TargetType='ip',
                Tags=[
                    {'Key': 'Name', 'Value': f"{service_name}-tg"},
                    {'Key': 'Service', 'Value': service_name},
                    {'Key': 'Purpose', 'Value': 'ContainerMicrosegmentation'}
                ]
            )
            
            target_groups[service_name] = tg_response['TargetGroups'][0]['TargetGroupArn']
        
        return target_groups
    
    def _create_ecs_services(self, cluster_name, service_definitions, security_groups, 
                           target_groups, subnet_ids):
        """Create ECS services with microsegmentation"""
        
        ecs_services = {}
        
        for service_name, service_config in service_definitions.items():
            # Create task definition
            task_definition = self._create_task_definition(service_name, service_config)
            
            # Create ECS service
            service_response = self.ecs.create_service(
                cluster=cluster_name,
                serviceName=service_name,
                taskDefinition=task_definition,
                desiredCount=2,
                launchType='FARGATE',
                networkConfiguration={
                    'awsvpcConfiguration': {
                        'subnets': subnet_ids,
                        'securityGroups': [security_groups[service_name]],
                        'assignPublicIp': 'DISABLED'
                    }
                },
                loadBalancers=[
                    {
                        'targetGroupArn': target_groups[service_name],
                        'containerName': service_name,
                        'containerPort': service_config['port']
                    }
                ],
                tags=[
                    {'key': 'Name', 'value': service_name},
                    {'key': 'Purpose', 'value': 'ContainerMicrosegmentation'}
                ]
            )
            
            ecs_services[service_name] = service_response['service']['serviceArn']
        
        return ecs_services
    
    def _create_task_definition(self, service_name, service_config):
        """Create ECS task definition for service"""
        
        task_def_response = self.ecs.register_task_definition(
            family=service_name,
            networkMode='awsvpc',
            requiresCompatibilities=['FARGATE'],
            cpu=str(service_config['cpu']),
            memory=str(service_config['memory']),
            executionRoleArn='arn:aws:iam::123456789012:role/ecsTaskExecutionRole',
            containerDefinitions=[
                {
                    'name': service_name,
                    'image': service_config['image'],
                    'portMappings': [
                        {
                            'containerPort': service_config['port'],
                            'protocol': 'tcp'
                        }
                    ],
                    'logConfiguration': {
                        'logDriver': 'awslogs',
                        'options': {
                            'awslogs-group': f'/ecs/{service_name}',
                            'awslogs-region': 'us-east-1',
                            'awslogs-stream-prefix': 'ecs'
                        }
                    },
                    'essential': True
                }
            ],
            tags=[
                {'key': 'Service', 'value': service_name},
                {'key': 'Purpose', 'value': 'ContainerMicrosegmentation'}
            ]
        )
        
        return task_def_response['taskDefinition']['taskDefinitionArn']
```

## Compliance and Audit Patterns

### Compliance Framework Implementation

```python
class MicrosegmentationCompliance:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.config = boto3.client('config')
        self.cloudtrail = boto3.client('cloudtrail')
    
    def implement_pci_dss_microsegmentation(self, vpc_id):
        """Implement PCI DSS compliant microsegmentation"""
        
        # PCI DSS requires network segmentation between CDE and non-CDE
        pci_architecture = {
            'cardholder-data-environment': {
                'description': 'CDE - Systems that store, process, or transmit cardholder data',
                'compliance_level': 'PCI_DSS_Level_1',
                'network_restrictions': {
                    'ingress': 'minimal_required_only',
                    'egress': 'explicit_allow_only',
                    'monitoring': 'comprehensive'
                }
            },
            'connected-environment': {
                'description': 'Systems connected to CDE but not storing cardholder data',
                'compliance_level': 'PCI_DSS_Connected',
                'network_restrictions': {
                    'ingress': 'controlled_access',
                    'egress': 'monitored',
                    'monitoring': 'enhanced'
                }
            },
            'corporate-environment': {
                'description': 'Standard corporate systems with no cardholder data access',
                'compliance_level': 'Standard',
                'network_restrictions': {
                    'ingress': 'standard_controls',
                    'egress': 'standard_controls',
                    'monitoring': 'standard'
                }
            }
        }
        
        # Create compliance-specific security groups
        compliance_security_groups = self._create_compliance_security_groups(
            vpc_id, pci_architecture
        )
        
        # Implement compliance monitoring
        compliance_monitoring = self._setup_compliance_monitoring(pci_architecture)
        
        return {
            'architecture': pci_architecture,
            'security_groups': compliance_security_groups,
            'monitoring': compliance_monitoring
        }
    
    def _create_compliance_security_groups(self, vpc_id, compliance_architecture):
        """Create security groups for compliance requirements"""
        
        compliance_sgs = {}
        
        for env_name, env_config in compliance_architecture.items():
            # Create security group for compliance environment
            sg_response = self.ec2.create_security_group(
                GroupName=f"{env_name}-compliance-sg",
                Description=f"Compliance security group for {env_config['description']}",
                VpcId=vpc_id,
                TagSpecifications=[
                    {
                        'ResourceType': 'security-group',
                        'Tags': [
                            {'Key': 'Name', 'Value': f"{env_name}-compliance-sg"},
                            {'Key': 'Environment', 'Value': env_name},
                            {'Key': 'ComplianceLevel', 'Value': env_config['compliance_level']},
                            {'Key': 'Purpose', 'Value': 'ComplianceMicrosegmentation'}
                        ]
                    }
                ]
            )
            
            sg_id = sg_response['GroupId']
            compliance_sgs[env_name] = sg_id
            
            # Configure rules based on compliance requirements
            self._configure_compliance_rules(sg_id, env_config)
        
        return compliance_sgs
    
    def _configure_compliance_rules(self, sg_id, env_config):
        """Configure security group rules based on compliance requirements"""
        
        # Remove default egress rule for strict control
        try:
            self.ec2.revoke_security_group_egress(
                GroupId=sg_id,
                IpPermissions=[
                    {
                        'IpProtocol': '-1',
                        'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
                    }
                ]
            )
        except:
            pass
        
        restrictions = env_config['network_restrictions']
        
        if restrictions['egress'] == 'explicit_allow_only':
            # Only allow specific outbound connections
            allowed_egress = [
                {'protocol': 'tcp', 'port': 443, 'dest': '0.0.0.0/0', 'description': 'HTTPS for updates'},
                {'protocol': 'tcp', 'port': 53, 'dest': '0.0.0.0/0', 'description': 'DNS'},
                {'protocol': 'udp', 'port': 53, 'dest': '0.0.0.0/0', 'description': 'DNS'},
                {'protocol': 'tcp', 'port': 123, 'dest': '0.0.0.0/0', 'description': 'NTP'}
            ]
            
            for rule in allowed_egress:
                try:
                    self.ec2.authorize_security_group_egress(
                        GroupId=sg_id,
                        IpPermissions=[
                            {
                                'IpProtocol': rule['protocol'],
                                'FromPort': rule['port'],
                                'ToPort': rule['port'],
                                'IpRanges': [{'CidrIp': rule['dest'], 'Description': rule['description']}]
                            }
                        ]
                    )
                except Exception as e:
                    print(f"Error adding egress rule: {e}")
    
    def _setup_compliance_monitoring(self, compliance_architecture):
        """Setup compliance-specific monitoring"""
        
        # Create Config rules for compliance monitoring
        compliance_rules = []
        
        for env_name, env_config in compliance_architecture.items():
            if env_config['compliance_level'] == 'PCI_DSS_Level_1':
                # Strict monitoring for PCI DSS environments
                compliance_rules.extend([
                    {
                        'ConfigRuleName': f'{env_name}-security-group-ssh-check',
                        'Source': {
                            'Owner': 'AWS',
                            'SourceIdentifier': 'INCOMING_SSH_DISABLED'
                        },
                        'Scope': {
                            'ComplianceResourceTypes': ['AWS::EC2::SecurityGroup']
                        }
                    },
                    {
                        'ConfigRuleName': f'{env_name}-security-group-unrestricted-common-ports',
                        'Source': {
                            'Owner': 'AWS',
                            'SourceIdentifier': 'SECURITY_GROUP_UNRESTRICTED_COMMON_PORTS'
                        }
                    }
                ])
        
        # Create the Config rules
        created_rules = []
        for rule in compliance_rules:
            try:
                self.config.put_config_rule(ConfigRule=rule)
                created_rules.append(rule['ConfigRuleName'])
            except Exception as e:
                print(f"Error creating Config rule {rule['ConfigRuleName']}: {e}")
        
        return created_rules
    
    def audit_microsegmentation_compliance(self, vpc_id):
        """Perform comprehensive audit of microsegmentation compliance"""
        
        audit_results = {
            'security_groups': self._audit_security_groups(vpc_id),
            'network_acls': self._audit_network_acls(vpc_id),
            'subnet_isolation': self._audit_subnet_isolation(vpc_id),
            'flow_logs': self._audit_flow_logs(vpc_id),
            'compliance_violations': []
        }
        
        # Analyze results for compliance violations
        audit_results['compliance_violations'] = self._analyze_compliance_violations(audit_results)
        
        return audit_results
    
    def _audit_security_groups(self, vpc_id):
        """Audit security group configurations"""
        
        response = self.ec2.describe_security_groups(
            Filters=[
                {
                    'Name': 'vpc-id',
                    'Values': [vpc_id]
                }
            ]
        )
        
        audit_findings = []
        
        for sg in response['SecurityGroups']:
            findings = {
                'group_id': sg['GroupId'],
                'group_name': sg['GroupName'],
                'issues': []
            }
            
            # Check for overly permissive rules
            for rule in sg['IpPermissions']:
                for ip_range in rule.get('IpRanges', []):
                    if ip_range.get('CidrIp') == '0.0.0.0/0':
                        if rule.get('FromPort', 0) == 22 or rule.get('ToPort', 0) == 22:
                            findings['issues'].append('SSH access from anywhere')
                        elif rule.get('FromPort', 0) == 3389 or rule.get('ToPort', 0) == 3389:
                            findings['issues'].append('RDP access from anywhere')
                        elif rule.get('IpProtocol') == '-1':
                            findings['issues'].append('All traffic allowed from anywhere')
            
            # Check egress rules
            for rule in sg['IpPermissionsEgress']:
                for ip_range in rule.get('IpRanges', []):
                    if ip_range.get('CidrIp') == '0.0.0.0/0' and rule.get('IpProtocol') == '-1':
                        findings['issues'].append('All outbound traffic allowed')
            
            if findings['issues']:
                audit_findings.append(findings)
        
        return audit_findings
    
    def _audit_network_acls(self, vpc_id):
        """Audit Network ACL configurations"""
        
        response = self.ec2.describe_network_acls(
            Filters=[
                {
                    'Name': 'vpc-id',
                    'Values': [vpc_id]
                }
            ]
        )
        
        nacl_findings = []
        
        for nacl in response['NetworkAcls']:
            # Check if NACL is properly configured (not using default allow-all)
            if nacl['IsDefault']:
                nacl_findings.append({
                    'nacl_id': nacl['NetworkAclId'],
                    'issue': 'Using default NACL with allow-all rules',
                    'severity': 'medium'
                })
        
        return nacl_findings
    
    def _audit_subnet_isolation(self, vpc_id):
        """Audit subnet isolation configuration"""
        
        subnets_response = self.ec2.describe_subnets(
            Filters=[
                {
                    'Name': 'vpc-id',
                    'Values': [vpc_id]
                }
            ]
        )
        
        route_tables_response = self.ec2.describe_route_tables(
            Filters=[
                {
                    'Name': 'vpc-id',
                    'Values': [vpc_id]
                }
            ]
        )
        
        isolation_findings = []
        
        # Check for proper subnet isolation
        public_subnets = []
        private_subnets = []
        
        for subnet in subnets_response['Subnets']:
            subnet_id = subnet['SubnetId']
            
            # Determine if subnet is public or private based on route table
            is_public = self._is_subnet_public(subnet_id, route_tables_response['RouteTables'])
            
            if is_public:
                public_subnets.append(subnet_id)
            else:
                private_subnets.append(subnet_id)
        
        # Validate isolation requirements
        if len(public_subnets) == 0:
            isolation_findings.append({
                'issue': 'No public subnets found - may impact internet access',
                'severity': 'low'
            })
        
        return {
            'public_subnets': public_subnets,
            'private_subnets': private_subnets,
            'findings': isolation_findings
        }
    
    def _is_subnet_public(self, subnet_id, route_tables):
        """Determine if subnet is public based on route table"""
        
        for rt in route_tables:
            for association in rt['Associations']:
                if association.get('SubnetId') == subnet_id:
                    # Check if route table has internet gateway route
                    for route in rt['Routes']:
                        if (route.get('DestinationCidrBlock') == '0.0.0.0/0' and 
                            route.get('GatewayId', '').startswith('igw-')):
                            return True
        return False
    
    def _audit_flow_logs(self, vpc_id):
        """Audit VPC Flow Logs configuration"""
        
        flow_logs_response = self.ec2.describe_flow_logs(
            Filters=[
                {
                    'Name': 'resource-id',
                    'Values': [vpc_id]
                }
            ]
        )
        
        flow_log_findings = []
        
        if not flow_logs_response['FlowLogs']:
            flow_log_findings.append({
                'issue': 'VPC Flow Logs not enabled',
                'severity': 'high',
                'recommendation': 'Enable VPC Flow Logs for security monitoring'
            })
        else:
            for flow_log in flow_logs_response['FlowLogs']:
                if flow_log['FlowLogStatus'] != 'ACTIVE':
                    flow_log_findings.append({
                        'issue': f'Flow log {flow_log["FlowLogId"]} is not active',
                        'severity': 'medium'
                    })
        
        return flow_log_findings
    
    def _analyze_compliance_violations(self, audit_results):
        """Analyze audit results for compliance violations"""
        
        violations = []
        
        # Check security group violations
        for sg_finding in audit_results['security_groups']:
            for issue in sg_finding['issues']:
                if 'SSH access from anywhere' in issue or 'RDP access from anywhere' in issue:
                    violations.append({
                        'type': 'critical_violation',
                        'resource': sg_finding['group_id'],
                        'issue': issue,
                        'compliance_impact': 'Violates network access control requirements'
                    })
        
        # Check flow log violations
        for flow_log_finding in audit_results['flow_logs']:
            if flow_log_finding['severity'] == 'high':
                violations.append({
                    'type': 'compliance_violation',
                    'resource': 'VPC',
                    'issue': flow_log_finding['issue'],
                    'compliance_impact': 'Required for audit logging and monitoring'
                })
        
        return violations
```

This comprehensive microsegmentation implementation provides multiple layers of network security controls, enabling organizations to create secure, compliant, and auditable network architectures that minimize blast radius and enhance overall security posture.