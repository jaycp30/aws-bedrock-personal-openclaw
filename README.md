AWSTemplateFormatVersion: '2010-09-09'
Description: 'OpenClaw - AWS Native Deployment (Bedrock + SSM + VPC Endpoints)'

Metadata:
  cfn-lint:
    config:
      ignore_checks:
        - E6101
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: "Basic Configuration"
        Parameters:
          - OpenClawModel
          - OpenClawVersion
          - InstanceType
          - KeyPairName
      - Label:
          default: "Network Configuration"
        Parameters:
          - CreateVPCEndpoints
          - AllowedSSHCIDR

Parameters:
  OpenClawModel:
    Type: String
    Default: "global.amazon.nova-2-lite-v1:0"
    Description: "Bedrock model ID - Nova 2 Lite offers best price-performance for everyday tasks"
    AllowedValues:
      - "global.amazon.nova-2-lite-v1:0"
      - "global.anthropic.claude-sonnet-4-5-20250929-v1:0"
      - "us.amazon.nova-pro-v1:0"
      - "global.anthropic.claude-opus-4-6-v1"
      - "global.anthropic.claude-opus-4-5-20251101-v1:0"
      - "global.anthropic.claude-haiku-4-5-20251001-v1:0"
      - "global.anthropic.claude-sonnet-4-20250514-v1:0"
      - "us.deepseek.r1-v1:0"
      - "us.meta.llama3-3-70b-instruct-v1:0"
      - "moonshotai.kimi-k2.5"

  InstanceType:
    Type: String
    Default: "c7g.large"
    Description: "Graviton (ARM) recommended for 20-40% better price-performance."
    AllowedValues:
      - "t4g.small"
      - "t4g.medium"
      - "t4g.large"
      - "t4g.xlarge"
      - "c6g.large"
      - "c6g.xlarge"
      - "c7g.large"
      - "c7g.xlarge"
      - "t3.small"
      - "t3.medium"
      - "t3.large"
      - "c5.xlarge"

  KeyPairName:
    Type: String
    Default: "none"
    Description: "EC2 key pair for emergency SSH access (optional)"

  AllowedSSHCIDR:
    Type: String
    Default: ""
    Description: "CIDR for SSH access (optional)"

  CreateVPCEndpoints:
    Type: String
    Default: "true"
    Description: "Create VPC endpoints for private network access to Bedrock and SSM"
    AllowedValues:
      - "true"
      - "false"

  EnableSandbox:
    Type: String
    Default: "true"
    Description: "Install Docker for sandboxed execution"

  OpenClawVersion:
    Type: String
    Default: "2026.3.24"
    AllowedValues:
      - "2026.3.24"
      - "2026.4.5"
      - "latest"
    Description: "OpenClaw version."

  EnableDataProtection:
    Type: String
    Default: "false"
    Description: "Retain data volume when stack is deleted"
    AllowedValues:
      - "true"
      - "false"

Conditions:
  CreateEndpoints: !Equals [!Ref CreateVPCEndpoints, "true"]
  HasKeyPair: !Not [!Equals [!Ref KeyPairName, "none"]]
  AllowSSH: !And
    - !Not [!Equals [!Ref AllowedSSHCIDR, ""]]
    - !Not [!Equals [!Ref KeyPairName, "none"]]
  IsUsEast1: !Equals [ !Ref "AWS::Region", "us-east-1" ]
  IsUsEast2: !Equals [ !Ref "AWS::Region", "us-east-2" ]
  IsUsWest2: !Equals [ !Ref "AWS::Region", "us-west-2" ]
  IsApSoutheast3: !Equals [ !Ref "AWS::Region", "ap-southeast-3" ]
  IsApSouth1: !Equals [ !Ref "AWS::Region", "ap-south-1" ]
  IsApNortheast1: !Equals [ !Ref "AWS::Region", "ap-northeast-1" ]
  IsEuCentral1: !Equals [ !Ref "AWS::Region", "eu-central-1" ]
  IsEuWest1: !Equals [ !Ref "AWS::Region", "eu-west-1" ]
  IsEuWest2: !Equals [ !Ref "AWS::Region", "eu-west-2" ]
  IsEuSouth1: !Equals [ !Ref "AWS::Region", "eu-south-1" ]
  IsEuNorth1: !Equals [ !Ref "AWS::Region", "eu-north-1" ]
  IsSaEast1: !Equals [ !Ref "AWS::Region", "sa-east-1" ]
  IsMantleSupportedRegion: !Or
    - !Or
      - Condition: IsUsEast1
      - Condition: IsUsEast2
      - Condition: IsUsWest2
      - Condition: IsApSoutheast3
      - Condition: IsApSouth1
      - Condition: IsApNortheast1
    - !Or
      - Condition: IsEuCentral1
      - Condition: IsEuWest1
      - Condition: IsEuWest2
      - Condition: IsEuSouth1
      - Condition: IsEuNorth1
      - Condition: IsSaEast1
  CreateMantleEndpoint: !And [ Condition: CreateEndpoints, Condition: IsMantleSupportedRegion ]
  EnableDocker: !Equals [!Ref EnableSandbox, "true"]
  ProtectData: !Equals [!Ref EnableDataProtection, "true"]
  DeleteData: !Not [!Equals [!Ref EnableDataProtection, "true"]]

Mappings:
  ArchitectureMap:
    t3.small: { Arch: "amd64" }
    t3.medium: { Arch: "amd64" }
    t3.large: { Arch: "amd64" }
    t3.xlarge: { Arch: "amd64" }
    c5.xlarge: { Arch: "amd64" }
    t4g.small: { Arch: "arm64" }
    t4g.medium: { Arch: "arm64" }
    t4g.large: { Arch: "arm64" }
    t4g.xlarge: { Arch: "arm64" }
    c6g.large: { Arch: "arm64" }
    c6g.xlarge: { Arch: "arm64" }
    c7g.large: { Arch: "arm64" }
    c7g.xlarge: { Arch: "arm64" }

Resources:
  OpenClawWaitHandle:
    Type: AWS::CloudFormation::WaitConditionHandle

  OpenClawWaitCondition:
    Type: AWS::CloudFormation::WaitCondition
    DependsOn: OpenClawInstance
    Properties:
      Handle: !Ref OpenClawWaitHandle
      Timeout: '900'
      Count: 1

  OpenClawVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-vpc"

  OpenClawInternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref OpenClawVPC
      InternetGatewayId: !Ref OpenClawInternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref OpenClawVPC
      CidrBlock: "10.0.1.0/24"
      MapPublicIpOnLaunch: true
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-public-subnet"

  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref OpenClawVPC
      CidrBlock: "10.0.2.0/24"
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-private-subnet-az1"

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Condition: CreateEndpoints
    Properties:
      VpcId: !Ref OpenClawVPC
      CidrBlock: "10.0.3.0/24"
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-private-subnet-az2"

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref OpenClawVPC

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: "0.0.0.0/0"
      GatewayId: !Ref OpenClawInternetGateway

  SubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

  VPCEndpointSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Condition: CreateEndpoints
    Properties:
      GroupDescription: "Security group for VPC endpoints"
      VpcId: !Ref OpenClawVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          SourceSecurityGroupId: !Ref OpenClawSecurityGroup

  BedrockRuntimeVPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateEndpoints
    Properties:
      VpcId: !Ref OpenClawVPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.bedrock-runtime'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet
        - !Ref PrivateSubnet2
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  BedrockMantleVPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateMantleEndpoint
    Properties:
      VpcId: !Ref OpenClawVPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.bedrock-mantle'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet
        - !Ref PrivateSubnet2
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  SSMVPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateEndpoints
    Properties:
      VpcId: !Ref OpenClawVPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ssm'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet
        - !Ref PrivateSubnet2
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  SSMMessagesVPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateEndpoints
    Properties:
      VpcId: !Ref OpenClawVPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ssmmessages'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet
        - !Ref PrivateSubnet2
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  EC2MessagesVPCEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Condition: CreateEndpoints
    Properties:
      VpcId: !Ref OpenClawVPC
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ec2messages'
      VpcEndpointType: Interface
      PrivateDnsEnabled: true
      SubnetIds:
        - !Ref PrivateSubnet
        - !Ref PrivateSubnet2
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup

  OpenClawInstanceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: 'sts:AssumeRole'
      ManagedPolicyArns:
        - 'arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore'
        - 'arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy'
      Policies:
        - PolicyName: BedrockAccessPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'bedrock:InvokeModel'
                  - 'bedrock:InvokeModelWithResponseStream'
                  - 'bedrock:ListFoundationModels'
                  - 'bedrock:GetFoundationModel'
                  - 'bedrock:ListInferenceProfiles'
                Resource: '*'
        - PolicyName: BedrockMantleAccessPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'bedrock-mantle:*'
                Resource: '*'
        - PolicyName: SSMParameterPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'ssm:PutParameter'
                  - 'ssm:GetParameter'
                Resource: !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter/openclaw/${AWS::StackName}/*'
        - PolicyName: EC2DescribeTagsPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'ec2:DescribeTags'
                Resource: '*'
        - PolicyName: CloudFormationDescribePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - 'cloudformation:DescribeStackResource'
                  - 'cloudformation:SignalResource'
                Resource: !Sub 'arn:aws:cloudformation:${AWS::Region}:${AWS::AccountId}:stack/${AWS::StackName}/*'

  OpenClawInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref OpenClawInstanceRole

  OpenClawSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "OpenClaw instance security group"
      VpcId: !Ref OpenClawVPC
      SecurityGroupIngress:
        - !If
          - AllowSSH
          - IpProtocol: tcp
            FromPort: 22
            ToPort: 22
            CidrIp: !Ref AllowedSSHCIDR
            Description: "SSH access (fallback)"
          - !Ref AWS::NoValue
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: "0.0.0.0/0"

  OpenClawDataVolumeRetained:
    Type: AWS::EC2::Volume
    Condition: ProtectData
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      AvailabilityZone: !Select [0, !GetAZs '']
      Size: 30
      VolumeType: gp3
      Encrypted: true

  OpenClawDataVolumeNotRetained:
    Type: AWS::EC2::Volume
    Condition: DeleteData
    DeletionPolicy: Delete
    UpdateReplacePolicy: Delete
    Properties:
      AvailabilityZone: !Select [0, !GetAZs '']
      Size: 30
      VolumeType: gp3
      Encrypted: true

  OpenClawInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Sub
        - '{{resolve:ssm:/aws/service/canonical/ubuntu/server/24.04/stable/current/${Arch}/hvm/ebs-gp3/ami-id}}'
        - Arch: !FindInMap [ArchitectureMap, !Ref InstanceType, Arch]
      InstanceType: !Ref InstanceType
      KeyName: !If [HasKeyPair, !Ref KeyPairName, !Ref "AWS::NoValue"]
      IamInstanceProfile: !Ref OpenClawInstanceProfile
      Volumes:
        - Device: /dev/sdf
          VolumeId: !If [ProtectData, !Ref OpenClawDataVolumeRetained, !Ref OpenClawDataVolumeNotRetained]
      NetworkInterfaces:
        - AssociatePublicIpAddress: true
          DeviceIndex: 0
          GroupSet:
            - !Ref OpenClawSecurityGroup
          SubnetId: !Ref PublicSubnet
      BlockDeviceMappings:
        - DeviceName: /dev/sda1
          Ebs:
            VolumeSize: 30
            VolumeType: gp3
            DeleteOnTermination: true
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-instance"

Outputs:
  Step1InstallSSMPlugin:
    Description: "STEP 1: Install SSM Session Manager Plugin on your local computer"
    Value: "https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html"

  Step2PortForwarding:
    Description: "STEP 2: Run this command on LOCAL computer (keep terminal open)"
    Value: !Sub |
      aws ssm start-session --target ${OpenClawInstance} --region ${AWS::Region} --document-name AWS-StartPortForwardingSession --parameters '{"portNumber":["18789"],"localPortNumber":["18789"]}'

  Step3AccessURL:
    Description: "STEP 3: Open this URL in browser"
    Value: !Sub
      - "http://localhost:18789/?token=${Token}"
      - Token: !Select [3, !Split ['"', !GetAtt OpenClawWaitCondition.Data]]

  Step4StartChatting:
    Description: "STEP 4: Start using OpenClaw!"
    Value: "Connect WhatsApp, Telegram, Discord. See README: https://github.com/aws-samples/sample-OpenClaw-on-AWS-with-Bedrock"

  InstanceId:
    Description: "EC2 Instance ID"
    Value: !Ref OpenClawInstance

  BedrockModel:
    Description: "Bedrock model in use"
    Value: !Ref OpenClawModel

  MonthlyCost:
    Description: "Estimated monthly cost (USD)"
    Value: !Sub
      - |
        EC2 (${InstanceType}): ~$20-40
        EBS (30GB): ~$2.40
        VPC Endpoints: ${EndpointCost}
        Bedrock: Pay-per-use
        Total: ~${TotalCost}/month
      - EndpointCost: !If [CreateEndpoints, "~$29 ($0.01/hour x 5 endpoints)", "$0"]
        TotalCost: !If [CreateEndpoints, "$45-65", "$23-43"]
