---
title: CloudFormation Multi-Stack Architectures
type: pattern
tags: [cloudformation, iac, nested-stacks, cross-stack-references, exports, importvalue, getstackoutput, stacksets, multi-account, multi-region, organizations]
certs: [aws-saa]
updated: 2026-11-18
---

# CloudFormation Multi-Stack Architectures

A single [CloudFormation](../services/cloudformation.md) stack is **isolated by design**. Everything in it is created, updated, and deleted together, and nothing in one stack can `Ref` a resource in another. That works until a solution outgrows one stack. This page covers the three ways CloudFormation lets you split infrastructure across stacks, starting with the decision and then going into each mechanism.

## Choosing a Multi-Stack Design

The question that decides the design is **lifecycle**: do the pieces get created, changed, and deleted together, or do they live independently?

| Need | Use | What is shared |
| --- | --- | --- |
| One solution, too big or too complex for one template, all parts created and deleted together | **[Nested stacks](#nested-stacks)** | The **template code** is reused; each deployment creates its own resources |
| A long-lived shared resource (a central VPC, a shared KMS key) used by many independently managed stacks in the **same account and Region** | **[Cross-stack references with Export and `Fn::ImportValue`](#export-and-fnimportvalue)** | The **actual resources**; the consumer is locked to the producer |
| The same, but the consumer is in **another account or Region**, or you do not want consumers to block the producer | **[`Fn::GetStackOutput`](#fngetstackoutput)** | The actual resources, through a weak reference |
| The **same template deployed as many separate stacks** across many accounts and Regions | **[StackSets](#stacksets)** | Nothing at runtime; StackSets manages many independent stacks from one place |

> [!tip] Exam Tip
> "Reuse the VPC **template**" and "reuse the **VPC**" are different questions. Nested stacks reuse code, so every root stack that includes the VPC template gets its own new VPC. Cross-stack references reuse a VPC that already exists. "Deploy the same baseline to every account in an OU" is StackSets.

## Why One Stack Stops Being Enough

The lecture describes the single, monolithic stack as perfectly fine until one of three limits bites:

- **Resource quota.** A template can declare at most **500 resources**, and the quota is not adjustable. AWS's suggested workaround is to split the template into nested stacks ([CloudFormation quotas](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html), verified 2026-11-18).
- **No reuse of a running resource.** A second stack cannot reference the VPC that the first stack created, because stacks are isolated.
- **Human manageability.** A thousand-line template that mixes networking, databases, and application code is hard to review, and different teams usually own those parts.

## Nested Stacks

A nested stack is a stack created **by another stack**. Inside the parent template it is just another logical resource, of type `AWS::CloudFormation::Stack`, whose `TemplateURL` points at an ordinary template in S3.

![CloudFormation nested stacks architecture. A root stack at the top receives Parameters and returns Outputs. Its template contains a VPCStack logical resource of type AWS::CloudFormation::Stack with a TemplateURL in S3 (https://s3.amazonaws.com/aemcf/vpctemplate.yaml) and Parameters VpcName, VpcCidr, and EnableDnsHostnames passed from the root's own parameters. Below the root sit three nested stacks, APPStack, DBStack, and VPCStack, each with a green tick for CREATE_COMPLETE. Red arrows return each nested stack's outputs to the root, and blue arrows pass values back down, labelled VPCStack.Outputs.XXXXX into DBStack and DBStack.Outputs.XXXXX into APPStack. Orange DependsOn arrows show APPStack depends on DBStack and DBStack depends on VPCStack. APPStack itself creates a further nested INTStack. A note says whole templates can be reused in other stacks, and the footer says use nested stacks when the stacks form part of one solution and are lifecycle linked](../assets/cfn-nested-stacks.png)

Read the diagram from the top:

1. You create **one root stack** the normal way (console, CLI, or a pipeline). It is an ordinary stack with parameters and outputs. There is only ever one root in a hierarchy.
2. The root template contains `AWS::CloudFormation::Stack` resources. Each creates a child stack from a template stored in S3 and passes it parameters. A child can contain its own nested stacks, so `APPStack` is the **parent** of `INTStack`, while the root is the parent of `APPStack`.
3. When a child reaches `CREATE_COMPLETE`, its **outputs are returned to its parent**, and only to its parent. `INTStack`'s outputs go to `APPStack`, not to the root.
4. The parent wires children together by passing one child's outputs into another child's parameters. The parent can read a child's outputs but **cannot reference resources inside the child directly**.
5. The root reaches `CREATE_COMPLETE` once every nested stack resource has.

### Completing the lecture example

The slide shows only the `VPCStack` resource, and labels the cross-stack values as `VPCStack.Outputs.XXXXX`. The real syntax for reading a child's output is `!GetAtt VPCStack.Outputs.<OutputName>`. The templates below fill in the rest: the root's own parameters, the other two children, and the child template whose outputs make the wiring work. They were checked against the AWS documentation, not deployed.

Root template, `root.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Root stack for the Aus-E-Mart web application

Parameters:
  TemplateBaseUrl:
    Type: String
    Description: HTTPS URL of the S3 location that holds the child templates
    Default: https://aemcf.s3.us-east-1.amazonaws.com
  VpcName:
    Type: String
    Default: aem-vpc1
  VpcCidrBlock:
    Type: String
    Default: 10.16.0.0/16
  EnableDnsHostnames:
    Type: String
    AllowedValues: ["true", "false"]
    Default: "true"

Resources:
  VPCStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub "${TemplateBaseUrl}/vpctemplate.yaml"
      Parameters:
        VpcName: !Ref VpcName
        VpcCidr: !Ref VpcCidrBlock
        EnableDnsHostnames: !Ref EnableDnsHostnames

  DBStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub "${TemplateBaseUrl}/dbtemplate.yaml"
      Parameters:
        VpcId: !GetAtt VPCStack.Outputs.VpcId
        SubnetIds: !GetAtt VPCStack.Outputs.PrivateSubnetIds

  APPStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Sub "${TemplateBaseUrl}/apptemplate.yaml"
      Parameters:
        VpcId: !GetAtt VPCStack.Outputs.VpcId
        SubnetIds: !GetAtt VPCStack.Outputs.PublicSubnetIds
        DatabaseEndpoint: !GetAtt DBStack.Outputs.DatabaseEndpoint

Outputs:
  AppUrl:
    Description: Login URL returned by the application stack
    Value: !GetAtt APPStack.Outputs.AppUrl
```

The lecture draws explicit `DependsOn` arrows between the stacks. In this version they are not needed: `!GetAtt VPCStack.Outputs.VpcId` already creates an implicit dependency, so CloudFormation creates `VPCStack`, then `DBStack`, then `APPStack`. Add `DependsOn` only when one child must wait for another without consuming any of its outputs (see [DependsOn](../services/cloudformation.md#dependson-and-resource-ordering)).

The reusable child template, `vpctemplate.yaml`. Its parameter names must match the keys the parent passes in, and its outputs are the only thing the parent can see:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Reusable two-AZ VPC with public and private subnets

Parameters:
  VpcName:
    Type: String
  VpcCidr:
    Type: String
    Default: 10.16.0.0/16
  EnableDnsHostnames:
    Type: String
    AllowedValues: ["true", "false"]
    Default: "true"

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: !Ref EnableDnsHostnames
      Tags:
        - Key: Name
          Value: !Ref VpcName

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: !Select [0, !Cidr [!GetAtt VPC.CidrBlock, 16, 12]]
      MapPublicIpOnLaunch: true

  PublicSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs ""]
      CidrBlock: !Select [1, !Cidr [!GetAtt VPC.CidrBlock, 16, 12]]
      MapPublicIpOnLaunch: true

  PrivateSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: !Select [2, !Cidr [!GetAtt VPC.CidrBlock, 16, 12]]

  PrivateSubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs ""]
      CidrBlock: !Select [3, !Cidr [!GetAtt VPC.CidrBlock, 16, 12]]

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnetARouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetA
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetBRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetB
      RouteTableId: !Ref PublicRouteTable

Outputs:
  VpcId:
    Value: !Ref VPC
  PublicSubnetIds:
    Description: Comma-delimited public subnet IDs
    Value: !Join [",", [!Ref PublicSubnetA, !Ref PublicSubnetB]]
  PrivateSubnetIds:
    Description: Comma-delimited private subnet IDs
    Value: !Join [",", [!Ref PrivateSubnetA, !Ref PrivateSubnetB]]
```

The database child only needs to declare a matching interface. Outputs are always strings, so the subnet list travels as a comma-delimited string; a `List<AWS::EC2::Subnet::Id>` parameter accepts that string and turns it back into a list. The private subnets have no route to the internet, which is fine for an RDS instance. `dbtemplate.yaml`:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Database tier for the Aus-E-Mart web application

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>

Resources:
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Private subnets for the web store database
      SubnetIds: !Ref SubnetIds

  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Database access from inside the VPC
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          CidrIp: 10.16.0.0/16

  Database:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    UpdateReplacePolicy: Snapshot
    Properties:
      Engine: mysql
      DBInstanceClass: db.t4g.micro
      AllocatedStorage: "20"
      MasterUsername: aemadmin
      ManageMasterUserPassword: true
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !GetAtt DBSecurityGroup.GroupId

Outputs:
  DatabaseEndpoint:
    Value: !GetAtt Database.Endpoint.Address
```

`apptemplate.yaml` follows the same shape: parameters `VpcId`, `SubnetIds`, and `DatabaseEndpoint`, its own resources, and an `AppUrl` output. `ManageMasterUserPassword` makes RDS generate the password and store it in [Secrets Manager](../services/secrets-manager.md), so no password appears in any template or parameter. The `DeletionPolicy` and `UpdateReplacePolicy` lines are explained in [Deletion and Replacement Policies](../services/cloudformation.md#deletion-and-replacement-policies).

### Getting the child templates into S3

`TemplateURL` must point at a template in S3. You do not have to upload children by hand: `aws cloudformation package` uploads local child templates referenced by relative path and rewrites `TemplateURL` to the S3 location.

```bash
# root.yaml can use TemplateURL: ./vpctemplate.yaml when packaged this way
aws cloudformation package \
  --template-file root.yaml \
  --s3-bucket aemcf \
  --output-template-file root.packaged.yaml

aws cloudformation deploy \
  --template-file root.packaged.yaml \
  --stack-name aem-webapp \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

`CAPABILITY_AUTO_EXPAND` acknowledges that the template contains nested stacks; `CAPABILITY_NAMED_IAM` is only needed when a child creates named IAM resources.

> [!warning] Gotcha
> Change a nested stack **through its root**. Updating the root re-evaluates every child, and CloudFormation keeps the hierarchy consistent. Updating a child stack directly makes it drift from what the root template describes, and the next root update can overwrite it.

### When nested stacks fit

- **Resource limit.** Each nested template has its own 500-resource quota. A separate quota caps how many resources one operation on a nested hierarchy can create, update, or delete at once (2,500 at the time of writing), so the total in a hierarchy can exceed that ([CloudFormation quotas](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)).
- **Modular templates.** The network team owns `vpctemplate.yaml`, and every application root stack includes it. Each inclusion builds a new VPC.
- **Complex solutions.** The root reads like an outline: network, database, application.
- **One lifecycle.** The lecture's single decision rule: use nested stacks **only when everything is lifecycle linked**. If you expect to keep the VPC after deleting the application, nested stacks are the wrong tool.

## Cross-Stack References

Cross-stack references let **independently managed stacks** use real resources that another stack owns. The lecture's example is a central Aus-E-Mart VPC stack managed by the network team, with a Web Store stack and an HR application stack, each owned by a different team, that both need its public subnet ID.

Without cross-stack references, the consumer teams would have to look up the subnet ID and type it into their templates as a static parameter. Both methods below start from the same place: the producer stack publishes the value as an [output](../services/cloudformation.md#outputs).

### Export and Fn::ImportValue

![Cross-stack references with Export and ImportValue in us-east-1. On the left, a Shared VPC Stack contains a VPC, an internet gateway, a route table, and three subnets; an orange arrow links the public subnet to an Outputs block declaring SharedPubSubID with Description Shared VPC Public Subnet, Value !Ref PublicSubnet, and Export Name SharedPubSubID. A pink line carries the export to a table at the top right labelled Exports per Region plus Account, with columns Export Name, Export Value, and Stack Name holding SharedPubSubID, subnet-0abc12345, and SharedVPCStack. Two consumer stacks on the right read it with !ImportValue SharedPubSubID, noted as used instead of Ref to reference Region-exported values. The footer reads: an active stack Import prevents deletion of an Export](../assets/cfn-cross-stack-export-import.png)

1. The producer adds an `Export` with a `Name` to an output.
2. CloudFormation writes the name and value to the **exports list**, which exists **per account per Region**. Export names must be unique within that account and Region.
3. Any other stack in the **same account and Region** reads the value with `Fn::ImportValue`, in the places where it would otherwise use `Ref`.
4. While any stack imports an export, CloudFormation **will not let you delete the producer stack or change or remove that export**. The link protects consumers from having the subnet pulled out from under them, and it is also what makes this pattern rigid.

The producer, `SharedVPCStack`, abbreviated to the resources the output needs. A real shared VPC would look like `vpctemplate.yaml` above:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Shared services VPC owned by the network team

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.16.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: 10.16.48.0/20
      MapPublicIpOnLaunch: true

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

Outputs:
  SharedPubSubID:
    Description: Shared VPC Public Subnet
    Value: !Ref PublicSubnet
    Export:
      Name: SharedPubSubID
```

A consumer in the same account and Region, the Web Store stack. The lecture's consumer uses `t4g.nano`, which is a Graviton (arm64) instance type, so the AMI parameter must point at an arm64 image or the launch fails:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Aus-E-Mart Web Store in the shared VPC

Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64

Resources:
  WebStoreInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t4g.nano
      ImageId: !Ref LatestAmiId
      SubnetId: !ImportValue SharedPubSubID
```

A fixed export name such as `SharedPubSubID` collides if the same producer template is deployed twice in one Region. The common convention is to prefix exports with the stack name and pass that name to consumers as a parameter. The short form `!ImportValue` cannot contain the short form `!Sub`, so the consumer switches to the full `Fn::ImportValue` form ([Fn::ImportValue](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-importvalue.html)):

```yaml
# Producer
Outputs:
  PublicSubnetId:
    Value: !Ref PublicSubnet
    Export:
      Name: !Sub "${AWS::StackName}-PublicSubnetId"

# Consumer
Parameters:
  NetworkStackName:
    Type: String
    Default: SharedVPCStack
Resources:
  WebStoreInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t4g.nano
      ImageId: !Ref LatestAmiId
      SubnetId:
        Fn::ImportValue: !Sub "${NetworkStackName}-PublicSubnetId"
```

### Fn::GetStackOutput

`Fn::GetStackOutput` is the newer method, launched in May 2026 ([What's New](https://aws.amazon.com/about-aws/whats-new/2026/05/aws-cloudformation-cdk-stack/)). The consumer names a stack and an output and reads it directly. The producer does **not** need an `Export`, and the stack can be in **another Region or another account** in the same partition ([Fn::GetStackOutput](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getstackoutput.html), verified 2026-11-18).

![Cross-stack references with Fn::GetStackOutput in us-east-1. The same Shared VPC Stack has an Outputs block declaring SharedPubSubID with Description and Value !Ref PublicSubnet but no Export, under the heading No Export required in Outputs. On the right, consumer stacks call GetStackOutput SharedPubSubID through an IAM Role that holds cloudformation:DescribeStacks. A code block shows a WebStoreInstance of type AWS::EC2::Instance with InstanceType t4g.nano, ImageId !Ref LatestAmiId, and SubnetId set by Fn::GetStackOutput with StackName SharedVPCStack, OutputName SharedPubSubID, Region us-east-1, and RoleArn arn:aws:iam::1234...:role/XStackOutRole. Notes say any stack output can be retrieved as long as permissions have been granted, and that for cross-account or cross-Region references the Region and role ARN must be specified](../assets/cfn-cross-stack-getstackoutput.png)

| Key | Required | Meaning |
| --- | --- | --- |
| `StackName` | Yes | The producer stack's name |
| `OutputName` | Yes | The producer output's logical ID |
| `Region` | No | Defaults to the consumer stack's Region |
| `RoleArn` | No | A role to assume for the read. Defaults to the consumer stack's execution role; required in practice for another account |

The role must allow `cloudformation:DescribeStacks` on the producer stack and must trust the identity that runs the consumer stack. If the consumer stack has a [stack role](../services/cloudformation.md#stack-roles), that is its execution role. The lecture's slide abbreviates both sides. Here are both, assuming the producer is in account `111111111111` in `us-east-1` and the consumer stack runs in account `222222222222` with a stack role named `CloudFormationExecutionRole`.

Deployed in the **producer** account, a role that only lets the consumer read that one stack:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Lets another account read SharedVPCStack outputs

Parameters:
  ConsumerExecutionRoleArn:
    Type: String
    Default: arn:aws:iam::222222222222:role/CloudFormationExecutionRole

Resources:
  XStackOutRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: XStackOutRole
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Ref ConsumerExecutionRoleArn
            Action: sts:AssumeRole
      Policies:
        - PolicyName: DescribeSharedVpcStack
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: cloudformation:DescribeStacks
                Resource: !Sub "arn:${AWS::Partition}:cloudformation:us-east-1:${AWS::AccountId}:stack/SharedVPCStack/*"
```

In the **consumer** account, the Web Store instance. The consumer's execution role also needs `sts:AssumeRole` on `XStackOutRole`. The subnet must be usable from the consumer account, which for a VPC in another account means it has been shared with [AWS RAM](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-sharing.html); reading the subnet ID is not the same as being allowed to launch into it.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Web Store reading a subnet ID from another account's stack

Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64

Resources:
  WebStoreInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t4g.nano
      ImageId: !Ref LatestAmiId
      SubnetId:
        Fn::GetStackOutput:
          StackName: SharedVPCStack
          OutputName: SharedPubSubID
          Region: us-east-1
          RoleArn: arn:aws:iam::111111111111:role/XStackOutRole
```

According to the [function reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getstackoutput.html):

- It is a **weak reference**. The value is read when the consumer is created or updated. Later changes in the producer are not pushed to the consumer, and **nothing stops you deleting the producer** or removing the output; the consumer's next update then fails. Protect the producer with deletion protection or a stack policy.
- It can be used as a resource property value, or inside `Fn::Join`, `Fn::If`, and `Fn::Select`. It **cannot** be used inside `Fn::Sub` variable maps, inside `Fn::Base64`, directly as an `Outputs` value, inside `Fn::Equals` in `Conditions`, or inside `Fn::ImportValue`.
- The short form `!GetStackOutput` exists but cannot contain other short-form functions such as `!Ref`; use the full form in that case.
- It does not work across partitions, such as from a commercial Region to AWS GovCloud (US).

| | Export + `Fn::ImportValue` | `Fn::GetStackOutput` |
| --- | --- | --- |
| Producer change needed | Add `Export` with a unique name | None; any output |
| Scope | Same account and Region | Any account or Region in the partition |
| Permissions | None beyond normal stack permissions | `DescribeStacks`, through a role for other accounts |
| Coupling | Strong: producer and export cannot be deleted or changed while imported | Weak: nothing prevents deletion; consumers fail on their next update |

## StackSets

StackSets solve a different problem: deploying **the same template as many separate stacks** across many accounts and Regions, managed from one place. The lecture's example is SpiderScan, an in-house security and compliance tool (an S3 bucket and a Lambda function) that Aus-E-Mart needs in every account and Region.

![CloudFormation StackSets architecture. On the left, an Admin Account holds the StackSet SpiderScan; a note says the template for the stack set is a normal CloudFormation template, and an illustration of the SpiderScan app with Lambda and S3 icons is captioned We need it everywhere. In the middle, permissions are granted via self-managed IAM roles or service-managed roles within an Organization, shown as IAM role helmets on dotted paths. Red arrows labelled Target Accounts point to two AWS accounts on the right, each with Region 1 and Region 2. In each Region a stack instance (a small stack icon pointed at by a red arrow in Region 1 of the top account) references an actual stack that creates the SpiderScan resources. The caption says StackSets gain access to target accounts and create stack instances and stacks](../assets/cfn-stacksets-architecture.png)

| Term | Meaning |
| --- | --- |
| **Administrator account** | Where you create and manage the stack set. With self-managed permissions it can be any account; with service-managed permissions it is the Organizations management account or a delegated administrator ([StackSets concepts](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html)) |
| **Stack set** | The management container in the administrator account: one template, its parameters, and the list of targets |
| **Target account** | An account the stack set deploys into |
| **Stack instance** | A reference to one stack in one account and one Region. It exists even if its stack failed to create, so the failure is still recorded |
| **Stack** | An ordinary CloudFormation stack in one account and one Region, created by StackSets |

### Permissions

StackSets has to act inside every target account:

- **Self-managed permissions.** You create the IAM roles yourself: an administration role in the administrator account (by convention `AWSCloudFormationStackSetAdministrationRole`) and an execution role in every target account (`AWSCloudFormationStackSetExecutionRole`) that trusts it. Targets are listed by account ID.
- **Service-managed permissions.** Used with [AWS Organizations](../services/organizations.md). After trusted access is enabled, StackSets creates the roles for you and targets are OUs or the organization root. Accounts that join a targeted OU later can receive the stack automatically.

### Completing the SpiderScan example

The lecture shows the architecture but no commands. A service-managed deployment to one OU in two Regions looks like this, run from the management account or with `--call-as DELEGATED_ADMIN` from a delegated administrator:

```bash
aws cloudformation create-stack-set \
  --stack-set-name SpiderScan \
  --template-body file://spiderscan.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
  --capabilities CAPABILITY_IAM

aws cloudformation create-stack-instances \
  --stack-set-name SpiderScan \
  --deployment-targets OrganizationalUnitIds=ou-ab12-34cd56ef \
  --regions us-east-1 ap-southeast-2 \
  --operation-preferences MaxConcurrentCount=2,FailureToleranceCount=1,RegionConcurrencyType=SEQUENTIAL
```

The template itself, `spiderscan.yaml`, is a normal template with one extra concern: the same template runs in many accounts and Regions, so nothing may be named in a way that collides. S3 bucket names are global, and IAM is a global service, so a hard-coded bucket name or role name fails in the second Region of the same account. Leaving names out lets CloudFormation generate unique ones:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: SpiderScan security scanner, deployed by StackSets

Resources:
  ScanResultsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256

  ScannerRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - !Sub "arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
        - !Sub "arn:${AWS::Partition}:iam::aws:policy/SecurityAudit"
      Policies:
        - PolicyName: WriteResults
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: s3:PutObject
                Resource: !Sub "${ScanResultsBucket.Arn}/*"

  ScannerFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.13
      Handler: index.handler
      Role: !GetAtt ScannerRole.Arn
      Timeout: 300
      Environment:
        Variables:
          RESULTS_BUCKET: !Ref ScanResultsBucket
      Code:
        ZipFile: |
          import os
          def handler(event, context):
              # Placeholder for the SpiderScan logic
              return {"bucket": os.environ["RESULTS_BUCKET"]}
```

Because the role is unnamed, `CAPABILITY_IAM` is enough; a named IAM resource would need `CAPABILITY_NAMED_IAM` and a name that includes `${AWS::Region}`.

### Operation settings

| Setting | What it controls |
| --- | --- |
| **Maximum concurrent accounts** | How many target accounts an operation works on at once, as a number or a percentage. It is a deployment pace, not a cap on how many accounts the stack set can hold |
| **Failure tolerance** | How many stack operations may fail **per Region** before CloudFormation stops the operation in that Region and does not continue to other Regions |
| **Concurrency mode** | The default **strict failure tolerance** lowers concurrency as failures occur, so concurrent accounts never exceed failure tolerance plus one; **soft failure tolerance** keeps running at the maximum concurrency regardless |
| **Retain stacks** | When removing stack instances, keep the actual stacks and their resources in the target accounts instead of deleting them. The default deletes them |

Under strict mode, `MaxConcurrentCount` cannot exceed `FailureToleranceCount` plus one, which is why the example above uses 2 and 1. A conservative first rollout is one account at a time with a failure tolerance of zero ([StackSets concepts](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html), [concurrency mode](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/concurrency-mode.html)).

Removing instances while keeping the stacks:

```bash
aws cloudformation delete-stack-instances \
  --stack-set-name SpiderScan \
  --deployment-targets OrganizationalUnitIds=ou-ab12-34cd56ef \
  --regions ap-southeast-2 \
  --retain-stacks
```

### Typical uses

- Enabling [AWS Config](../services/config.md) and deploying Config rules (MFA, unattached Elastic IPs, EBS encryption) across every account.
- Creating [IAM roles](../services/iam.md) for cross-account access in every account.
- Rolling out a security tool such as SpiderScan, or a logging baseline, organization-wide. [Control Tower](../services/control-tower.md) uses StackSets for this kind of baseline.

> [!tip] Exam Tip
> "Deploy the same resources to many accounts and Regions from a central account" points to **StackSets**. "Many accounts in an Organization, including future accounts" points to **service-managed** permissions with automatic deployment.

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03): "CloudFormation Nested Stacks", "CloudFormation Cross-Stack References", and "CloudFormation Stack Sets" ([course](https://learn.cantrill.io/))
- [AWS: CloudFormation quotas](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html) (verified 2026-11-18)
- [AWS: AWS::CloudFormation::Stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-stack.html)
- [AWS: Get exported outputs from a deployed stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html)
- [AWS: Fn::ImportValue](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-importvalue.html)
- [AWS: Fn::GetStackOutput](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getstackoutput.html) (verified 2026-11-18)
- [AWS What's New: reference stack outputs across accounts and Regions (May 2026)](https://aws.amazon.com/about-aws/whats-new/2026/05/aws-cloudformation-cdk-stack/)
- [AWS DevOps Blog: Fn::GetStackOutput](https://aws.amazon.com/blogs/devops/simplify-cross-account-and-cross-region-stack-output-references-with-aws-cloudformation-and-cdks-new-fngetstackoutput/)
- [AWS: StackSets concepts](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html)
- [AWS: StackSets with service-managed permissions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-associate-stackset-with-org.html)
- [AWS: StackSets with self-managed permissions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-getting-started-create-self-managed.html)
- [AWS: StackSets concurrency mode](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/concurrency-mode.html)
- [AWS: VPC sharing](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-sharing.html)
