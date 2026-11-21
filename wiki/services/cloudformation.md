---
title: CloudFormation
type: service
tags: [cloudformation, iac, infrastructure-as-code, template, stack, logical-resource, physical-resource, parameters, pseudo-parameters, intrinsic-functions, mappings, conditions, outputs, dependson, creation-policy, wait-condition, cfn-init, configsets, cfn-signal, cfn-hup, deletion-policy, update-replace-policy, change-sets, drift, stack-roles, custom-resources, bootstrapping, ec2, user-data]
certs: [aws-saa]
updated: 2026-11-20
---

# CloudFormation

**AWS CloudFormation** is AWS's native **infrastructure-as-code** service. You describe the resources you want in a **template** (YAML or JSON). CloudFormation turns that description into real AWS resources and keeps them in step with the template as one managed unit, called a **stack**. You change infrastructure by changing the template, not by clicking through consoles.

This page works from the big picture down:

1. [The core model](#templates-stacks-and-resources): templates, stacks, logical and physical resources, and what happens on create, update, and delete.
2. [What goes in a template](#template-anatomy): its sections, then each section in turn, from [inputs](#template-inputs) and [intrinsic functions](#intrinsic-functions) through [mappings](#mappings), [conditions](#conditions), [outputs](#outputs), and [resource ordering](#dependson-and-resource-ordering).
3. [Bootstrapping EC2 from a stack](#bootstrapping-ec2-from-a-stack): how a stack learns whether an instance really finished configuring using `cfn-signal`, and how `cfn-init` and `cfn-hup` manage configuration inside it.
4. Operating stacks safely: [deletion and replacement policies](#deletion-and-replacement-policies), [change sets](#change-sets), [stack roles](#stack-roles), and [custom resources](#custom-resources).
5. [Beyond one stack](#beyond-one-stack): nested stacks, cross-stack references, and StackSets, covered on [CloudFormation Multi-Stack Architectures](../patterns/cloudformation-multi-stack.md).

Many lecture slides shorten templates with `... (stuff here)` or leave out required properties. The examples on this page are complete templates that fill those gaps. They were checked against the AWS documentation cited next to them, **not deployed**. Templates that launch instances assume a default VPC with public default subnets unless they say otherwise.

## Templates, Stacks, and Resources

CloudFormation separates **what** you want from **how** it gets built. The template says what; CloudFormation works out the API calls, their order, and the cleanup.

![CloudFormation physical and logical resources architecture. At the top left, a YAML template snippet declares Resources, then a logical resource named Instance with Type AWS::EC2::Instance and Properties ImageId !Ref LatestAmiId, InstanceType t3.micro, and KeyName AEM; arrows label Instance as the logical resource NAME and AWS::EC2::Instance as its TYPE, and a note says resource Properties are used by CloudFormation when creating the matching physical resources. An orange arrow leads down to the CloudFormation Template icon, and a long arrow labelled CreateStack uses a template, parameters and options to create a stack points right to the CloudFormation Stack icon. A note beside the stack says a stack creates, updates and deletes physical resources based on logical resources in the template. A pink arrow rises from the stack to an EC2 instance, labelled with physical ID i-1234567890abcdef0, built from an AMI and the AEM SSH key. A magnifier icon notes that once a logical resource moves to create_complete, meaning the physical resource is active, it can be queried for attributes of the physical resource within the template](../assets/cfn-logical-physical-architecture.png)

| Term                  | What it is                                                                                                                                                                 |
| --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Template**          | A YAML or JSON document that declares resources and, optionally, inputs, logic, and outputs. One template can create one stack or hundreds, in many accounts and Regions   |
| **Stack**             | The living deployment of a template. It owns the resources the template describes and keeps them in sync with it                                                           |
| **Logical resource**  | An entry under `Resources:` in the template, identified by a **logical ID** you choose (such as `Instance`), a **type** (such as `AWS::EC2::Instance`), and **properties** |
| **Physical resource** | The real AWS resource the stack created for a logical resource, identified by a **physical ID** (such as `i-1234567890abcdef0`)                                            |

A stack does three things with its logical resources:

1. **Create.** For every logical resource, the stack creates a physical resource using the properties in the template. When the physical resource is ready, the logical resource reaches `CREATE_COMPLETE`, and other resources can then read its physical ID and attributes.
2. **Update.** Apply a changed template to an existing stack and CloudFormation changes the physical resources to match.
3. **Delete.** Remove a logical resource from the template, or delete the whole stack, and CloudFormation normally deletes the physical resource too. [Deletion policies](#deletion-and-replacement-policies) change that.

If creation fails partway through, CloudFormation by default **rolls back**: it deletes whatever it had created, so a failed stack does not leave half-built infrastructure behind.

### What an update does to a resource

When you submit an update, CloudFormation updates resources based on differences between what you submit and the stack's current template. Resources that haven't changed run without disruption during the update process. For updated resources, CloudFormation uses one of the following update behaviors ([update behaviors of stack resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html)):

| Update behavior       | What happens                                                                                                                      | Example                                                                 |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **No interruption**   | Changed in place; the resource keeps running and keeps its physical ID                                                            | Changing an EC2 instance's tags or its `Monitoring` setting             |
| **Some interruption** | Changed in place but disrupted, such as a stop and start; the physical ID stays the same                                          | Changing an EBS-backed instance's `InstanceType`                        |
| **Replacement**       | CloudFormation creates a **new** physical resource with a new physical ID, repoints dependent resources, then deletes the old one | Changing an instance's `ImageId`, or an EBS volume's `AvailabilityZone` |

The update behavior CloudFormation uses depends on which property you update for a given resource type. Each property's behavior is listed under "Update requires" in the [AWS resource and property types reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-template-resource-type-ref.html). Replacement is where data loss happens: a replaced database or volume is a new, empty resource unless you protect it with an [`UpdateReplacePolicy`](#deletion-and-replacement-policies). [Change sets](#change-sets) show the behavior of every change before you run it.

### Portable templates

A template written once and reused unchanged in any account and Region is called **portable**. The two most common ways a template fails to be portable are covered in [Making a template portable](#making-a-template-portable) below.

## Template Anatomy

Only `Resources` is required. The other sections exist to make one template serve many deployments ([template sections](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html)).

| Section                    | Purpose                                                                                                                             |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `AWSTemplateFormatVersion` | Template format version. The only valid value is `"2010-09-09"`; if present, it should be first                                     |
| `Description`              | Free text describing the template. If present, it must directly follow `AWSTemplateFormatVersion`                                   |
| `Metadata`                 | Extra information about the template. `AWS::CloudFormation::Interface` groups and orders parameters in the console                  |
| `Parameters`               | Values supplied when the stack is created or updated. See [template parameters](#template-parameters)                               |
| `Rules`                    | Validates parameter values, or combinations of them, before any resources are created                                               |
| `Mappings`                 | Static lookup tables. See [Mappings](#mappings)                                                                                     |
| `Conditions`               | Named true or false expressions that control whether resources are created or how properties are set. See [Conditions](#conditions) |
| `Transform`                | Macros that process the template before deployment, such as `AWS::Serverless-2016-10-31` for AWS SAM                                |
| `Resources`                | **Required.** The logical resources                                                                                                 |
| `Outputs`                  | Values the stack reports after deployment. See [Outputs](#outputs)                                                                  |

The template below uses every common section once. Each section is explained later on this page; read it now for the overall shape. It creates one web server, sized from a mapping, and gives it an Elastic IP only in production.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Aus-E-Mart web server, a reference template that uses every common section

Metadata:
  AWS::CloudFormation::Interface:
    ParameterGroups:
      - Label:
          default: Environment
        Parameters:
          - EnvType
      - Label:
          default: Instance
        Parameters:
          - LatestAmiId

Parameters:
  EnvType:
    Type: String
    Default: dev
    AllowedValues: [dev, prod]
    Description: Deployment environment
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
    Description: SSM public parameter that resolves to the latest Amazon Linux 2023 AMI

Mappings:
  EnvSettings:
    dev:
      InstanceType: t3.micro
    prod:
      InstanceType: t3.small

Conditions:
  IsProd: !Equals [!Ref EnvType, prod]

Resources:
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP from anywhere
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: !FindInMap [EnvSettings, !Ref EnvType, InstanceType]
      SecurityGroupIds:
        - !GetAtt WebSecurityGroup.GroupId
      UserData:
        Fn::Base64: |
          #!/bin/bash
          dnf install -y httpd
          echo "<h1>Aus-E-Mart</h1>" > /var/www/html/index.html
          systemctl enable --now httpd
      Tags:
        - Key: Name
          Value: !Sub "${AWS::StackName}-web"

  WebServerEIP:
    Type: AWS::EC2::EIP
    Condition: IsProd
    Properties:
      Domain: vpc
      InstanceId: !Ref WebServer

Outputs:
  WebsiteURL:
    Description: URL of the web server
    Value: !If
      - IsProd
      - !Sub "http://${WebServerEIP}"
      - !Sub "http://${WebServer.PublicDnsName}"
```

Two details are easy to miss:

- `!GetAtt WebSecurityGroup.GroupId`, not `!Ref WebSecurityGroup`. For a security group created without `VpcId`, `Ref` returns the group **name**, while `SecurityGroupIds` needs the **ID** ([AWS::EC2::SecurityGroup return values](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-securitygroup.html#aws-resource-ec2-securitygroup-return-values)). Also, `!Ref WebSecurityGroup.GroupId` is invalid syntax: `!Ref` does not support the dotted `Resource.Attribute` form, which only `!GetAtt` and `${...}` inside `!Sub` accept. It takes only a name: a resource's logical ID, a parameter, or a pseudo parameter.
- The `WebsiteURL` in `Outputs:` uses the `IsProd` condition to pick the address. In production, the stack also creates `WebServerEIP`, so the URL uses the Elastic IP. In dev, there is no Elastic IP, so the URL uses `WebServer.PublicDnsName`. Associating an Elastic IP releases the instance's original public IPv4 address and changes its public DNS name to match the Elastic IP ([Elastic IP address basics](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)), so in production the Elastic IP is the stable address to publish.

## Template Inputs

### Template parameters

Parameters let the person or pipeline creating the stack supply values at deploy time, so the same template can create differently configured stacks ([Parameters section](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)).

![CloudFormation template parameters. A YAML Parameters block declares InstanceType of Type String with Default t3.micro, AllowedValues t3.micro, t3.medium and t3.large, and Description Pick a supported InstanceType, plus InstanceAmiId of Type String with Description AMI ID For Instances. An arrow labelled parameters loaded from template points to a user at a laptop, and below it a console Parameters form shows an empty InstanceAmiId text box and an InstanceType drop-down listing t3.micro, t3.medium and t3.large. An arrow labelled default, or explicit values chosen leads to the template, captioned parameter values referenced within template logical resources. A final arrow to the stack is labelled parameter references are used by CloudFormation to provision physical resources, influencing in this example InstanceType and AMI, with three EC2 instances of different sizes and an AMI ID icon](../assets/cfn-template-parameters.png)

The console reads the `Parameters` section and builds a form from it: a drop-down for `AllowedValues`, a text box otherwise. The values chosen are referenced in resources with `!Ref`.

| Property                  | What it does                                                                                                       |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `Type`                    | Required. `String`, `Number`, `List<Number>`, `CommaDelimitedList`, an AWS-specific type, or an SSM parameter type |
| `Default`                 | Value used when none is supplied                                                                                   |
| `AllowedValues`           | The only values accepted; the console shows a drop-down                                                            |
| `AllowedPattern`          | A regular expression a `String` must match                                                                         |
| `MinLength` / `MaxLength` | Length limits for a `String`                                                                                       |
| `MinValue` / `MaxValue`   | Range limits for a `Number`                                                                                        |
| `Description`             | Help text shown in the console                                                                                     |
| `ConstraintDescription`   | Message shown when a constraint fails                                                                              |
| `NoEcho`                  | Masks the value as asterisks in the console and in `describe` calls                                                |

**AWS-specific parameter types** such as `AWS::EC2::KeyPair::KeyName`, `AWS::EC2::VPC::Id`, and `List<AWS::EC2::Subnet::Id>` are validated against the account at deploy time, and the console offers a list of real values. **SSM parameter types** such as `AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>` take the **name** of a [Systems Manager Parameter Store](ssm.md#parameter-store) parameter and resolve it to its current value when the stack is created or updated.

> [!warning] Gotcha
> `NoEcho` only hides the value from display. It is not encryption, and the value can still leak through anything that echoes it, such as an output or user data. For secrets, reference [Secrets Manager](secrets-manager.md) with a dynamic reference such as `{{resolve:secretsmanager:MySecret:SecretString:password}}`, or let the service manage the secret, as `ManageMasterUserPassword` does for RDS ([dynamic references](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html)).

### Pseudo parameters

Pseudo parameters are values **AWS** supplies; you never declare them. They are used exactly like parameters, with `!Ref` or inside `!Sub` ([pseudo parameters reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/pseudo-parameter-reference.html)).

![CloudFormation pseudo parameters. A CloudFormation Template arrow points to a CloudFormation Stack. Above the stack, an AWS logo feeds values down into it: a dotted line from a us-east-1 flag is labelled AWS::Region, and a dotted line on the right is labelled AWS::StackId, AWS::StackName and AWS::AccountId. A note reads: AWS::Region will always reflect the region the stack is being created in; AWS::StackName and AWS::StackId will match the specific stack being created; and AWS::AccountId will be set by AWS to the actual account ID the stack is being created within](../assets/cfn-pseudo-parameters.png)

| Pseudo parameter        | Value                                                                                            |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| `AWS::Region`           | The Region the stack is being created in, such as `us-east-1`                                    |
| `AWS::AccountId`        | The account the stack is in                                                                      |
| `AWS::StackName`        | The stack's name                                                                                 |
| `AWS::StackId`          | The stack's ARN                                                                                  |
| `AWS::Partition`        | `aws`, `aws-cn`, or `aws-us-gov`; use it when building ARNs so templates work in every partition |
| `AWS::URLSuffix`        | The domain suffix, usually `amazonaws.com`                                                       |
| `AWS::NotificationARNs` | The SNS topics receiving stack events                                                            |
| `AWS::NoValue`          | Removes a property entirely when returned from `Fn::If`. See [Conditions](#conditions)           |

### Making a template portable

The lecture demo starts with a template that works exactly once. It has two problems:

1. It sets `BucketName: accatpics1333333337`. S3 bucket names are **globally unique**, so a second stack anywhere fails because the name is taken.
2. It sets `ImageId: ami-090fa75af13c156b4`. AMI IDs are **Region-specific**, so the template fails in any other Region, and the image eventually becomes outdated.

The fix happens in three stages:

1. **Remove `BucketName`.** CloudFormation then generates a unique name from the stack name, the logical ID, and a random suffix. Read it back with `!Ref Bucket` or an [output](#outputs).
2. **Make the AMI a parameter.** This works anywhere, but the person deploying has to look up the right AMI ID for the Region.
3. **Make the AMI parameter an SSM parameter type** that defaults to an AWS **public parameter**. AWS keeps it pointing at the latest image in every Region.

The final, portable template:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Portable bucket and instance

Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  Bucket:
    Type: AWS::S3::Bucket

  Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref LatestAmiId

Outputs:
  BucketName:
    Value: !Ref Bucket
```

> [!warning] Course correction
> The course's final stage uses the Amazon Linux 2 parameter `/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2` with a `t2.micro`. Amazon Linux 2 reached end of support on 2026-06-30 ([Amazon Linux 2 FAQs](https://aws.amazon.com/amazon-linux-2/faqs/)). Use the Amazon Linux 2023 path above, which uses `dnf` rather than `yum`. For Graviton instance types (`t4g`, `m6g`, and others), use the `arm64` variant of the path, `al2023-ami-kernel-default-arm64`, or the launch fails on an architecture mismatch.

## Intrinsic Functions

Intrinsic functions are built-in functions that compute values when the stack is created or updated: reading another resource's ID, choosing an AZ, building a string. Each has a full form (`Fn::GetAtt`) and a YAML short form (`!GetAtt`). Some short forms cannot directly contain other short forms; for example, `!ImportValue` cannot contain `!Sub`, and `!GetStackOutput` cannot contain `!Ref`. Write the outer function in its full form when that happens ([intrinsic function reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html)).

| Function                                               | Returns                                                                                                                                                 |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Ref`                                                  | A parameter's value, or a resource's main identifier                                                                                                    |
| `Fn::GetAtt`                                           | A named attribute of a resource                                                                                                                         |
| `Fn::GetAZs`                                           | The list of AZs in a Region                                                                                                                             |
| `Fn::Select`                                           | One item from a list, by index                                                                                                                          |
| `Fn::Join` / `Fn::Split`                               | A string built from a list, or a list split from a string                                                                                               |
| `Fn::Sub`                                              | A string with `${...}` variables substituted                                                                                                            |
| `Fn::Base64`                                           | Base64-encoded text, mainly for EC2 user data                                                                                                           |
| `Fn::Cidr`                                             | A list of subnet CIDR blocks carved from a larger block                                                                                                 |
| `Fn::FindInMap`                                        | A value from a [mapping](#mappings)                                                                                                                     |
| `Fn::If`, `Fn::Equals`, `Fn::And`, `Fn::Or`, `Fn::Not` | [Condition](#conditions) logic                                                                                                                          |
| `Fn::ImportValue`                                      | An exported output from another stack. See [cross-stack references](../patterns/cloudformation-multi-stack.md#export-and-fnimportvalue)                 |
| `Fn::GetStackOutput`                                   | Any output from another stack, also cross-account or cross-Region. See [Fn::GetStackOutput](../patterns/cloudformation-multi-stack.md#fngetstackoutput) |

### Ref and Fn::GetAtt

![Ref and Fn::GetAtt. At the bottom, a logical resource named Instance of Type AWS::EC2::Instance with ImageId !Ref LatestAmiId, InstanceType t3.micro and KeyName AEM flows through Template and Stack to a T3 physical resource. Above, !Ref Instance points to i-1234567890abcdef0, with the note: using !Ref on template or pseudo parameters returns their value, and when used with logical resources the physical ID is usually returned. Below it, !GetAtt LogicalResource.Attribute has two arrows: PublicIp returning 54.91.129.183 and PublicDnsName returning ec2-54-91-129-183.compute-1.amazonaws.com. The notes say GetAtt can retrieve any attribute associated with the resource, most logical resources return detailed configuration of the physical resource, and the attributes depend on the resource type](../assets/cfn-ref-getatt.png)

- `!Ref` on a **parameter** returns its value. On a **resource** it returns the resource's main identifier, usually the physical ID, such as an instance ID.
- `!GetAtt LogicalId.Attribute` returns one of the resource's other attributes, such as an instance's `PublicIp` or `PublicDnsName`.

"Usually" matters: what `Ref` returns is decided per resource type. Check the "Return values" section of the type's reference page before relying on it.

| Resource type                              | `!Ref` returns                                   | Common `!GetAtt` attributes                                  |
| ------------------------------------------ | ------------------------------------------------ | ------------------------------------------------------------ |
| `AWS::EC2::Instance`                       | Instance ID                                      | `PublicIp`, `PublicDnsName`, `PrivateIp`, `AvailabilityZone` |
| `AWS::EC2::SecurityGroup`                  | Group ID if `VpcId` is set, otherwise group name | `GroupId`, `VpcId`                                           |
| `AWS::EC2::EIP`                            | The Elastic IP address                           | `AllocationId`, `PublicIp`                                   |
| `AWS::S3::Bucket`                          | Bucket name                                      | `Arn`, `DomainName`, `RegionalDomainName`                    |
| `AWS::IAM::Role`                           | Role name                                        | `Arn`, `RoleId`                                              |
| `AWS::Lambda::Function`                    | Function name                                    | `Arn`                                                        |
| `AWS::CloudFormation::WaitConditionHandle` | The handle's presigned URL                       | None                                                         |

Using `!Ref` or `!GetAtt` also tells CloudFormation that one resource **depends on** another. See [DependsOn and resource ordering](#dependson-and-resource-ordering).

### Fn::GetAZs and Fn::Select

![Fn::GetAZs and Fn::Select. !GetAZs "us-east-1" or "" for the current region returns a list of AZs in the explicit region or the current region: us-east-1a, us-east-1b, us-east-1c, us-east-1d, us-east-1e and us-east-1f, matching six coloured AZ columns on the right inside a us-east-1 box. The note says the Fn::Select or !Select function returns an object from a list of objects, and lists start at index 0, meaning the first object. A code block shows an Instance with ImageId !Ref LatestAmiId, InstanceType t3.micro, KeyName AEM and AvailabilityZone !Select [ 0, !GetAZs '' ], and a dotted arrow places a T3 instance in us-east-1a. A caption says using !GetAZs and !Select makes a template portable: rather than explicitly specifying AZs, resources can be launched in AZ1, AZ2, AZ3, which are dynamically assigned from the available AZs at run time](../assets/cfn-getazs-select.png)

`!GetAZs ""` returns the AZs of the stack's own Region; `!Select [0, list]` picks the first. Together they avoid hard-coding AZ names such as `us-east-1a`, which do not exist in other Regions.

> [!warning] Gotcha
> `Fn::GetAZs` does not always return every AZ. If the account has a default VPC, it returns only the AZs that have a **default subnet** in it. If none do, for example because the default subnets were deleted, it returns all AZs in the Region ([Fn::GetAZs](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getavailabilityzones.html)). A template that assumes at least three entries can break in an account where someone deleted default subnets.

### Fn::Join and Fn::Split

`!Join [delimiter, [list]]` concatenates a list into one string. `!Split [delimiter, string]` does the reverse. They are often combined with other functions:

```yaml
# Build a URL: "http://" + the instance's public DNS name
Value: !Join ["", ["http://", !GetAtt Instance.PublicDnsName]]

# Split "a.b.c" on "." and take the second element, giving "b"
Value: !Select [1, !Split [".", "a.b.c"]]

# Turn a comma-delimited subnet list from another stack into a real list
SubnetIds: !Split [",", !ImportValue SharedSubnetIds]
```

### Fn::Base64 and Fn::Sub

EC2 expects **user data** to be base64-encoded. The console encodes it for you; a template does not, so user data must be wrapped in `Fn::Base64`. `Fn::Sub` replaces `${...}` variables in a string with real values when the stack runs:

| Syntax inside `!Sub`     | Replaced with                                                                                                                                                                         |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `${ParameterName}`       | The parameter's value                                                                                                                                                                 |
| `${LogicalId}`           | The same value as `!Ref LogicalId`                                                                                                                                                    |
| `${LogicalId.Attribute}` | The same value as `!GetAtt LogicalId.Attribute`                                                                                                                                       |
| `${AWS::StackName}`      | A pseudo parameter's value                                                                                                                                                            |
| `${!Literal}`            | The literal text `${Literal}`, left for the shell to expand ([Fn::Sub](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-sub.html)) |

A plain shell variable written as `$VAR`, without braces, passes through `Fn::Sub` unchanged.

The lecture's `Fn::Sub` slide puts `${Instance.InstanceId}` **inside that same instance's user data**, and the lecturer points out that this is deliberately invalid: the instance ID does not exist until the instance has been created, and the user data is needed to create it. CloudFormation rejects it as a circular dependency. The working approach is to let the instance discover its own ID at boot from the [Instance Metadata Service](ec2.md#instance-metadata-service-imds), and to use `${!...}` so `Fn::Sub` leaves that shell variable alone:

```yaml
Parameters:
  BackgroundColour:
    Type: String
    Default: lightblue
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: t3.micro
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -x
          dnf install -y httpd
          systemctl enable --now httpd
          # IMDSv2: get a session token, then read this instance's own ID
          TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
            -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
          INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
            http://169.254.169.254/latest/meta-data/instance-id)
          cat > /var/www/html/index.html <<EOF
          <html><body style="background-color:${BackgroundColour}">
          <h1>Instance ${!INSTANCE_ID} in stack ${AWS::StackName}</h1>
          </body></html>
          EOF
```

Here is what each variable becomes:

- `${BackgroundColour}` and `${AWS::StackName}` are replaced by CloudFormation before the script reaches the instance.
- `${!INSTANCE_ID}` reaches the instance as `${INSTANCE_ID}`, and bash fills it in at boot.
- `$TOKEN` and `$(...)` have no braces, so `Fn::Sub` ignores them.

Referencing **another** resource's attributes from user data, such as a database endpoint, works normally.

### Fn::Cidr

![Fn::Cidr. A code block declares a VPC of Type AWS::EC2::VPC with CidrBlock 10.16.0.0/16, and two subnets. Subnet1 has CidrBlock !Select [ "0", !Cidr [ !GetAtt VPC.CidrBlock, "16", "12" ] ] and VpcId !Ref VPC; Subnet2 is the same with index "1". Annotations point the GetAtt at the VPC's 10.16.0.0/16 block, label the "16" as how many subnets to generate from the input VPC range, and label the "12" as bits per CIDR, 32 minus 12 equals /20. The caption reads Fn::Cidr is used to generate a number of smaller CIDR ranges for subnets, from a larger VPC range](../assets/cfn-cidr.png)

`!Cidr [ipBlock, count, cidrBits]` splits a block into `count` subnets. The third argument is the number of **host bits** in each subnet, so the prefix length is 32 minus `cidrBits`: 12 bits gives a `/20` ([Fn::Cidr](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-cidr.html)). For `10.16.0.0/16`, `!Cidr [..., 16, 12]` returns `10.16.0.0/20`, `10.16.16.0/20`, `10.16.32.0/20`, and so on, up to 16 blocks. Pair each subnet with an AZ from `Fn::GetAZs`:

```yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.16.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true

  SubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs ""]
      CidrBlock: !Select [0, !Cidr [!GetAtt VPC.CidrBlock, 16, 12]]   # 10.16.0.0/20

  SubnetB:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs ""]
      CidrBlock: !Select [1, !Cidr [!GetAtt VPC.CidrBlock, 16, 12]]   # 10.16.16.0/20
```

Unlike the slide, this version adds the AZ. Without `AvailabilityZone`, EC2 picks an AZ for each subnet, and both can land in the same one, which defeats the purpose of having two. The [VPC page](vpc.md) explains how subnets and AZs relate.

## Mappings

A mapping is a static lookup table inside the template. `!FindInMap` always takes exactly **three** arguments: the map name, a top-level key, and a second-level key ([Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-findinmap.html)). The classic uses are per-Region values, such as AMI IDs before SSM public parameters existed, and per-environment settings.

![CloudFormation mappings example. A Mappings block declares AEMInstanceSizes with three top-level keys: Dev with Web t4g.nano and App t4g.small, Staging with Web t4g.small and App m6g.large, and Prod with Web m6g.large and App m6g.xlarge. The syntax !FindInMap [ MapName, TopLevelKey, SecondLevelKey ] is shown at the top right. Below, !FindInMap [ "AEMInstanceSizes", !Ref Environment, !Ref WorkloadTier ] is colour-coded: the map name links to AEMInstanceSizes as the logical resource (mapping) to reference, !Ref Environment links to the Dev key as the top level key, in this case referencing an Environment parameter, and !Ref WorkloadTier links to the Web key](../assets/cfn-mappings-findinmap.png)

The slide shows the lookup but not the parameters that feed it. A complete version follows. The parameters' `AllowedValues` must match the map keys **exactly, including case**; otherwise the lookup fails when the stack is created. Every instance type in the map is a Graviton type, so the AMI must be `arm64`.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Instance size chosen by environment and workload tier

Parameters:
  Environment:
    Type: String
    AllowedValues: [Dev, Staging, Prod]
    Default: Dev
  WorkloadTier:
    Type: String
    AllowedValues: [Web, App]
    Default: Web
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-arm64

Mappings:
  AEMInstanceSizes:
    Dev:
      Web: t4g.nano
      App: t4g.small
    Staging:
      Web: t4g.small
      App: m6g.large
    Prod:
      Web: m6g.large
      App: m6g.xlarge

Resources:
  Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: !FindInMap [AEMInstanceSizes, !Ref Environment, !Ref WorkloadTier]
```

Mappings are fixed when the template is written. If the values change often or are shared by many templates, keep them in Parameter Store and read them with an SSM parameter type instead.

## Conditions

Conditions let one template create different sets of resources. CloudFormation evaluates the `Conditions` section **first**, using the parameter values, so each condition is already true or false before any resource is processed ([Conditions section](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html)).

![CloudFormation conditions. Step 1: Bob, applying the Aus-E-Mart webstore template, chooses an environment type, Dev or Prod, via a template parameter; the Parameters block declares EnvType with Default dev, Type String and AllowedValues dev and prod. Step 2: the Conditions block declares IsProd: !Equals [ !Ref EnvType, 'prod' ], and a note says CloudFormation uses any Conditions block first, evaluating the TRUE or FALSE state of conditions via intrinsic functions, before the CFN stack processes resources. Step 3: the Resources block declares Webstore, an EC2 instance with ImageId !Ref LatestAMI and no condition; Webstore2, another instance with Condition IsProd; MyEIP, an Elastic IP with Condition IsProd and InstanceId !Ref Webstore; and MyEIP2 with Condition IsProd and InstanceId !Ref Webstore2. A note says that when processing Resources, if a Condition key is present the resource is only created if the condition is TRUE. The outcomes at the bottom right are one instance with a public IPv4 address, or two instances with Elastic IPs. A footer says the templates are not fully complete](../assets/cfn-conditions.png)

The slide's footer admits the templates are incomplete. Here is the full version: `dev` creates one instance with an auto-assigned public IPv4 address, and `prod` creates two instances, each with an Elastic IP.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Aus-E-Mart web store, one instance in dev and two with Elastic IPs in prod

Parameters:
  EnvType:
    Type: String
    Default: dev
    AllowedValues: [dev, prod]
  LatestAMI:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Conditions:
  IsProd: !Equals [!Ref EnvType, prod]

Resources:
  Webstore:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAMI
      InstanceType: t3.micro

  Webstore2:
    Type: AWS::EC2::Instance
    Condition: IsProd
    Properties:
      ImageId: !Ref LatestAMI
      InstanceType: t3.micro

  MyEIP:
    Type: AWS::EC2::EIP
    Condition: IsProd
    Properties:
      Domain: vpc
      InstanceId: !Ref Webstore

  MyEIP2:
    Type: AWS::EC2::EIP
    Condition: IsProd
    Properties:
      Domain: vpc
      InstanceId: !Ref Webstore2

Outputs:
  WebstoreAddress:
    Value: !If [IsProd, !Ref MyEIP, !GetAtt Webstore.PublicIp]
  Webstore2Address:
    Condition: IsProd
    Value: !Ref MyEIP2
```

Rules this template follows:

- A conditional resource can reference an unconditional one (`MyEIP` references `Webstore`), but **not the reverse**. If `Webstore` referenced `Webstore2`, a `dev` stack would fail because `Webstore2` does not exist.
- An output that references a conditional resource needs the same `Condition`, as `Webstore2Address` has.
- `Domain: vpc` creates the EIP for use in a VPC.

> [!note] Current AWS behavior
> Both outcomes on the slide are billed. AWS charges for every public IPv4 address, whether auto-assigned or Elastic, attached or idle ([VPC pricing](https://aws.amazon.com/vpc/pricing/)). The difference is that an EIP keeps the same address through stop, start, and instance replacement.

Conditions can be **combined** with `!And`, `!Or`, and `!Not`, and can refer to other conditions with `!Condition`. A condition can also change a **property** instead of a whole resource: `!If` with `AWS::NoValue` removes the property when the condition is false.

```yaml
Parameters:
  KeyName:
    Type: String
    Default: ""
    Description: Optional EC2 key pair name; leave empty for none

Conditions:
  IsProd: !Equals [!Ref EnvType, prod]
  IsUsEast1: !Equals [!Ref "AWS::Region", us-east-1]
  IsProdInUsEast1: !And [!Condition IsProd, !Condition IsUsEast1]
  HasKeyName: !Not [!Equals [!Ref KeyName, ""]]

Resources:
  Webstore:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAMI
      InstanceType: !If [IsProdInUsEast1, t3.small, t3.micro]
      KeyName: !If [HasKeyName, !Ref KeyName, !Ref "AWS::NoValue"]
```

This fragment builds on the template above, which declares `EnvType` and `LatestAMI`.

## Outputs

`Outputs` is optional. It reports values that only exist after deployment, such as a generated bucket name or a public DNS name. Outputs appear in the console and in `aws cloudformation describe-stacks`. They also connect stacks to each other ([Outputs section](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html)):

- A **nested stack's** outputs are returned to its parent, which reads them with `!GetAtt ChildStack.Outputs.Name`.
- An output with `Export` can be imported by other stacks in the same account and Region.
- Any output can be read from other stacks, accounts, or Regions with `Fn::GetStackOutput`.

All three are covered on [CloudFormation Multi-Stack Architectures](../patterns/cloudformation-multi-stack.md).

The lecture's example builds a URL for the Aus-E-Mart web store with `!Join`. Its slide uses `https://`, but a plain web server on an instance serves HTTP only, so the URL below uses `http://`. HTTPS needs a certificate on a load balancer or on the instance. See [TLS/HTTPS](../concepts/tls-https.md).

```yaml
Outputs:
  WebStoreURL:
    Description: Aus-E-Mart web store URL
    Value: !Join ["", ["http://", !GetAtt Instance.PublicDnsName]]
    Export:
      Name: !Sub "${AWS::StackName}-WebStoreURL"
```

`Description` is optional but worth writing: in a large or nested deployment, it is how other teams learn what an output means. `Export` is only needed for `Fn::ImportValue`, and prefixing export names with the stack name keeps them unique per Region.

## DependsOn and Resource Ordering

CloudFormation creates resources **in parallel** wherever it can, which is part of what makes it fast. It works out the order from **dependencies**:

- **Implicit dependency.** When resource A uses `!Ref B`, `!GetAtt B.Attr`, or `${B}` in `!Sub`, CloudFormation creates B first and deletes it last.
- **Explicit dependency.** `DependsOn: B` (or a list, `DependsOn: [B, C]`) forces the same ordering when nothing references B ([DependsOn attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-dependson.html)).

Explicit dependencies matter when one resource needs another to **exist** without needing any of its values.

![CloudFormation DependsOn example. A VPC with CidrBlock 10.16.0.0/16, DNS support and hostnames enabled and Name tag aem-vpc1, and an InternetGateway with Name tag aem-vpc1-igw, are joined by a two-way arrow captioned these can be created in parallel, they don't need each other in order to exist. In the middle, InternetGatewayAttachment of Type AWS::EC2::VPCGatewayAttachment has VpcId !Ref VPC and InternetGatewayId !Ref InternetGateway; arrows to the VPC and gateway explain that a gateway attachment requires a gateway and something (the VPC) to attach it to, and that !Ref creates an implicit dependency. On the right, a thumbs-down AEMWebServerEIP of Type AWS::EC2::EIP with no dependency is captioned: an Elastic IP requires an IGW attached to a VPC in order to work, but there is no dependency in the template, implicit or explicit. A thumbs-up AEMWebServerEIP adds DependsOn InternetGatewayAttachment and InstanceId !Ref AEMWebServerEC2, captioned: DependsOn creates an explicit dependency; AEMWebServerEIP will be created after InternetGatewayAttachment is complete and deleted before it's deleted](../assets/cfn-dependson-eip.png)

Read the slide in three parts:

- The VPC and the internet gateway reference nothing, so they are created in parallel.
- The attachment references both, so it waits for both.
- An Elastic IP needs an internet gateway **attached** to the VPC, but nothing in the EIP's properties references the attachment. Without `DependsOn`, CloudFormation may create the EIP first, and the operation fails. With `DependsOn`, the EIP is created after the attachment and, on stack deletion, **deleted before** it.

The slide shows only these four resources. The template below adds what a reachable web server also needs: a subnet, a route table with an internet route, and the instance. The default route has the same hidden dependency as the EIP, which the slide does not show: it references the gateway but not the attachment.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Aus-E-Mart VPC with a public web server and an Elastic IP

Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.16.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: aem-vpc1

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: aem-vpc1-igw

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

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC

  DefaultRoute:
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

  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  AEMWebServerEC2:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: t3.micro
      SubnetId: !Ref PublicSubnet
      SecurityGroupIds:
        - !Ref WebSecurityGroup

  AEMWebServerEIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      InstanceId: !Ref AEMWebServerEC2
```

Here `!Ref WebSecurityGroup` returns the group ID, because the group sets `VpcId`.

> [!tip] Exam Tip
> `DependsOn` is the answer when a resource must be created after another one that it does not reference, the classic case being an EIP or a public route after the internet gateway **attachment**. Where a `Ref` or `GetAtt` already exists, `DependsOn` is redundant.

## Bootstrapping EC2 from a Stack

A stack can create an EC2 instance, but the instance's real work, installing and configuring software, happens **inside** the operating system after EC2 reports the instance as launched. Four features cover that gap:

1. **[User data](ec2.md#bootstrapping)** runs a script once at first boot.
2. **[Creation policies and `cfn-signal`](#creation-policies--cfn-signal)** make the stack **wait** until the instance reports that its configuration succeeded. **Wait conditions** do the same as a separate resource that can also carry data.
3. **[`cfn-init`](#cfn-init--desired-state-bootstrapping)** applies a **desired state** described in the template's metadata, instead of a hand-written script.
4. **[`cfn-hup`](#re-running-on-updates--cfn-hup)** watches that metadata and re-runs `cfn-init` when a stack update changes it.

The helper scripts `cfn-init`, `cfn-signal`, and `cfn-hup` come in the `aws-cfn-bootstrap` package under `/opt/aws/bin/`. They call the CloudFormation API, so the instance needs a network path to the CloudFormation endpoint: a public IP with an internet gateway, a NAT gateway, or an interface VPC endpoint ([helper scripts reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/cfn-helper-scripts-reference.html)).

## Creation Policies & cfn-signal

### The problem

By default, **CloudFormation has no idea whether in-instance bootstrapping succeeded.** It asks EC2 for an instance, EC2 reports that the instance launched, and the logical resource moves to `CREATE_COMPLETE`. User data runs after that, and it is opaque to both EC2 and CloudFormation. If the script fails, the stack still reports success. That is the same "[failed bootstrap still passes status checks](ec2.md#bootstrapping)" trap that affects raw user data.

### How a creation policy works

A `CreationPolicy` on a resource tells CloudFormation to **hold** the resource in `CREATE_IN_PROGRESS` until it receives a number of **success signals**, or until a **timeout** expires ([CreationPolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-creationpolicy.html)):

| Setting                                                   | Meaning                                                                         | Default |
| --------------------------------------------------------- | ------------------------------------------------------------------------------- | ------- |
| `ResourceSignal.Count`                                    | Success signals required                                                        | 1       |
| `ResourceSignal.Timeout`                                  | ISO 8601 duration to wait, such as `PT15M`; maximum 12 hours (`PT12H`)          | `PT5M`  |
| `AutoScalingCreationPolicy.MinSuccessfulInstancesPercent` | For an Auto Scaling group, the percentage of instances that must signal success | 100     |

Creation policies are supported on `AWS::EC2::Instance`, `AWS::AutoScaling::AutoScalingGroup`, `AWS::AppStream::Fleet`, and `AWS::CloudFormation::WaitCondition`.

The signal is sent from the instance by **`cfn-signal`**, usually as the last line of user data:

```bash
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource EC2Instance --region ${AWS::Region}
```

- `-e $?` passes the **exit code of the previous command**. `0` sends a success signal; anything else sends a failure signal.
- `--resource` is the **logical ID** of the resource that has the creation policy. CloudFormation fills in `${AWS::StackName}` and `${AWS::Region}` when the template is wrapped in `!Sub`.

| What CloudFormation receives                              | Result                                                           |
| --------------------------------------------------------- | ---------------------------------------------------------------- |
| The required number of success signals within the timeout | The resource moves to `CREATE_COMPLETE` and the stack continues  |
| A failure signal                                          | The resource fails, and the stack rolls back                     |
| Not enough signals before the timeout                     | Treated as failure; the resource fails, and the stack rolls back |

![Creation policy and signals architecture, contrasting two rows. Top row, without a creation policy: Template, Stack, Instance. The instance launches and is marked complete with a green tick, but a yellow question-mark box on the far right shows the problem: CloudFormation has no idea whether the in-instance bootstrapping actually worked; it trusts EC2's provisioned report and moves on. Bottom row, with a creation policy: the EC2Instance resource now carries CreationPolicy, ResourceSignal, Timeout PT15M, shown in a left code block. The Template-Stack-Instance flow is the same, but the stack now waits at a hold point instead of auto-completing. An orange dotted arrow traces the signal from the instance's UserData back to the stack; the cfn-signal -e $? line, highlighted at the bottom, sends the exit code of the preceding cfn-init command. Only on a success signal does the resource reach CREATE_COMPLETE, shown as a green-tick instance at the bottom right](../assets/cfn-init-CreationPolicy-and-signals.png)

The top row is the default: CloudFormation marks the instance complete on EC2's word alone. The bottom row adds a creation policy with a 15-minute timeout, which turns that blind completion into a hold point that only a success signal from the instance releases.

### Example: an Auto Scaling group that must report in

![CloudFormation creation policy on an Auto Scaling group. A code block declares AutoScalingGroup of Type AWS::AutoScaling::AutoScalingGroup with "... (stuff here)", DesiredCapacity 3, MinSize 1, MaxSize 4, and a highlighted CreationPolicy with ResourceSignal Count 3 and Timeout PT15M. A second block declares LaunchTemplate with "... (stuff here)" and UserData using Fn::Base64 and !Sub with #!/bin/bash -xe, yum update -y aws-cfn-bootstrap, some bootstrapping, and a highlighted line /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource AutoScalingGroup --region ${AWS::Region}. The diagram says the CreationPolicy applies a signal requirement (3) and a timeout (15M) to the stack, shown with an hourglass and a 15. The ASG provisions 3 EC2 instances, each signalling once via cfn-signal with a green tick. If all three succeed the stack reaches CREATE_COMPLETE; if one instance has a bug, shown as a red bug among two ticks, the stack goes to CREATE_FAILED](../assets/cfn-creation-policy-asg.png)

The Auto Scaling group creates three instances from a launch template. Each instance signals once, and the group needs three success signals within 15 minutes. The slide elides most of both resources, and its script has a subtle problem. `#!/bin/bash -xe` stops the script at the **first failing command**, which means the `cfn-signal` line at the bottom never runs. The stack then waits the full 15 minutes and fails on the timeout instead of failing at once with a useful signal.

The complete version runs the bootstrap steps in a subshell that stops on the first error, then always signals with the subshell's exit code:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Aus-E-Mart web tier; the stack completes only when all instances report success

Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Subnets for the instances; they need outbound internet access

Resources:
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Ref LatestAmiId
        InstanceType: t3.micro
        MetadataOptions:
          HttpTokens: required
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash -x
            (
              set -e
              dnf install -y aws-cfn-bootstrap httpd
              echo "<h1>Aus-E-Mart web store</h1>" > /var/www/html/index.html
              systemctl enable --now httpd
              curl -sf http://localhost/ > /dev/null
            )
            /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource AutoScalingGroup --region ${AWS::Region}

  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    CreationPolicy:
      ResourceSignal:
        Count: 3
        Timeout: PT15M
    Properties:
      MinSize: "1"
      MaxSize: "4"
      DesiredCapacity: "3"
      VPCZoneIdentifier: !Ref SubnetIds
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
```

Some notes on this template:

- The `curl` check makes "success" mean that the web server actually answers, not just that packages installed. `aws-cfn-bootstrap` may already be on the AMI; installing it again is harmless.
- The subnets need outbound internet access, or a NAT gateway, so that `dnf` and `cfn-signal` can reach their endpoints.
- `Count` should not exceed `DesiredCapacity`, or the stack can never collect enough signals.
- A creation policy applies only when the group is **created**. To make **updates** that replace instances wait for signals too, add an `UpdatePolicy` with `AutoScalingRollingUpdate` and `WaitOnResourceSignals: true` ([UpdatePolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-updatepolicy.html)).

Auto Scaling groups and launch templates are covered on [EC2 Auto Scaling](ec2-auto-scaling.md).

### Wait conditions

A wait condition is the same signalling idea as a **separate resource** instead of an attribute on a resource. Because it is its own logical resource, it can depend on other resources, and other resources can depend on it. That makes it a general **gate** in the middle of a template that nothing can pass until signals arrive. Signals can also carry **data** back into the stack ([AWS::CloudFormation::WaitCondition](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-waitcondition.html)).

![CloudFormation wait conditions. A WaitHandle of Type AWS::CloudFormation::WaitConditionHandle generates a presigned URL for resource signals. A WaitCondition of Type AWS::CloudFormation::WaitCondition has DependsOn "someresource" and Properties Handle !Ref "WaitHandle", Timeout "300" and Count '1'; a note says the WaitCondition can depend on other resources, and other resources can depend on the WaitCondition. On the right, an EC2 instance or an external system sends a JSON document back in the signal response: Status SUCCESS, Reason Something Important, UniqueId ID1337, Data Some amazing thing has happened. At the bottom, !GetAtt WaitCondition.Data, accessible via GetAtt on the WaitCondition, returns {"Signal1":"Some amazing thing has happened."}](../assets/cfn-wait-condition.png)

The two resources work together:

1. A `WaitConditionHandle` does one job: `!Ref WaitHandle` returns a **presigned URL**. Whoever sends the signal needs no AWS credentials, only that URL.
2. A `WaitCondition` waits for `Count` success signals sent to that URL, within `Timeout` **seconds** (maximum 43,200, which is 12 hours).
3. The signaller, whether an EC2 instance or an external system such as a licensing server, sends an HTTP `PUT` with a JSON body containing `Status`, `Reason`, `UniqueId`, and `Data`.
4. `!GetAtt WaitCondition.Data` returns the signals' data as a JSON object keyed by each signal's `UniqueId`.

The slide labels the result `"Signal1"`, but the key is the signal's `UniqueId`. With the JSON shown on the slide, the actual value would be `{"ID1337":"Some amazing thing has happened."}`. The complete template, with the slide's `"someresource"` filled in as an instance:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Wait condition that pauses the stack until an instance reports in

Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  WaitHandle:
    Type: AWS::CloudFormation::WaitConditionHandle

  Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: t3.micro
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -x
          # ... the work the stack is waiting for, such as applying a licence ...
          curl -s -X PUT -H "Content-Type:" \
            --data-binary '{"Status":"SUCCESS","Reason":"Something Important","UniqueId":"ID1337","Data":"Some amazing thing has happened."}' \
            "${WaitHandle}"

  WaitCondition:
    Type: AWS::CloudFormation::WaitCondition
    DependsOn: Instance
    Properties:
      Handle: !Ref WaitHandle
      Timeout: "300"
      Count: 1

Outputs:
  SignalData:
    Value: !GetAtt WaitCondition.Data
```

Three details make this work:

- `"${WaitHandle}"` in `!Sub` is the same as `!Ref WaitHandle`, so the presigned URL is written into the script.
- The empty `Content-Type:` header is needed because the URL was signed without a content type.
- `DependsOn: Instance` starts the 300-second timer only after the instance exists. Without it, the wait condition and the instance would be created in parallel, and slow instance creation would eat into the timeout.

`cfn-signal` can send the same signal: `/opt/aws/bin/cfn-signal -e 0 -i ID1337 -d "Some amazing thing has happened." "${WaitHandle}"`.

|                          | Creation policy                                         | Wait condition                                   |
| ------------------------ | ------------------------------------------------------- | ------------------------------------------------ |
| What it is               | An attribute on the resource being created              | A separate logical resource, plus a handle       |
| Signals go to            | The CloudFormation API, through `cfn-signal --resource` | A presigned URL from the handle                  |
| Can gate other resources | Only through dependencies on that resource              | Yes; anything can depend on the wait condition   |
| Returns data             | No                                                      | Yes, with `!GetAtt WaitCondition.Data`           |
| AWS recommendation       | **Use for EC2 instances and Auto Scaling groups**       | For external systems or when data must come back |

AWS recommends creation policies over wait conditions for EC2 and Auto Scaling resources ([AWS::CloudFormation::WaitCondition](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-waitcondition.html)).

> [!tip] Exam Tip
> "CloudFormation reports `CREATE_COMPLETE` but the application on the instance is not working" points to a **creation policy with `cfn-signal`**. "Pause the stack until an external system finishes, and pass a value back" points to a **wait condition**.

## cfn-init — Desired-State Bootstrapping

`cfn-init` is a helper script that works like a **lightweight configuration-management system**. Instead of writing a script that runs line by line, you describe the state the instance should be in, inside the template, and `cfn-init` makes it so.

### Procedural vs desired state

| Aspect      | User data                                              | cfn-init                                                                       |
| ----------- | ------------------------------------------------------ | ------------------------------------------------------------------------------ |
| **Model**   | **Procedural** only: a script the OS runs line by line | **Procedural** or **desired state**                                            |
| **Meaning** | Does exactly what each line says, in order             | You declare what should be true; cfn-init does whatever is needed to get there |

**Desired-state example.** You tell cfn-init you want Apache version X:

- If version X is already installed, nothing happens.
- If Apache is not installed, cfn-init installs it.
- If an older version is present, cfn-init updates it to X.

> [!tip] One-line mental model
> **User data runs steps; cfn-init ensures a state.** User data is "run this script once and hope it works." cfn-init is "here is the state I want: achieve it, re-apply it when it changes, and, with a creation policy, report back whether it worked."

### How it works

The desired state lives in the resource's `Metadata` under **`AWS::CloudFormation::Init`**. User data still runs once at first boot; its job shrinks to calling `cfn-init`, then `cfn-signal`.

1. The template creates a **stack**, which creates the **EC2 instance**.
2. The instance's **user data** runs `cfn-init`, passing the stack name, the resource's logical ID, and the Region.
3. `cfn-init` calls the **CloudFormation service** and retrieves that resource's `AWS::CloudFormation::Init` metadata.
4. `cfn-init` applies that desired state to the instance and exits with a success or error code, which `cfn-signal` reports.

![CloudFormation cfn-init architecture. The top row shows the high-level flow Template, Stack, Instance connected by solid pink arrows: the template creates the stack, which creates the EC2 instance. On the left, the template's contents: the EC2Instance logical resource carries a CreationPolicy plus Metadata containing AWS::CloudFormation::Init with the configSets and five config keys (install_cfn, software_install, configure_instance, install_wordpress, configure_wordpress), which form the desired-state definition. At the bottom, the UserData block wraps a short script in Fn::Base64 and !Sub that runs cfn-init (highlighted) then cfn-signal; this is the only procedural part, kicking everything off at launch. Numbered dotted arrows trace the runtime: arrow 1 from the launched instance running its cfn-init line, arrow 2 from cfn-init consulting the stack using the substituted StackId and Region to pull the desired-state config, arrow 3 returning the config to be applied. The footer reads WORKS WITH STACK UPDATES](../assets/cfn-init.png)

In the lecture's WordPress demo, the configset `wordpress_install` runs five config keys in order: `install_cfn`, `software_install`, `configure_instance`, `install_wordpress`, and `configure_wordpress`. The footer, **"works with stack updates"**, points at what user data lacks; that is the job of [cfn-hup](#re-running-on-updates--cfn-hup).

### Inside AWS::CloudFormation::Init

The metadata is organized as **configsets**, which are ordered lists of **config keys**. Each config key can contain up to seven sections ([AWS::CloudFormation::Init](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-init.html)):

| Section    | What it manages                                                                                                                                                        |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `packages` | Packages, by package manager: `rpm`, `yum`, `apt`, `python`, `rubygems`, `msi`. An empty version list means latest                                                     |
| `groups`   | Linux groups                                                                                                                                                           |
| `users`    | Linux users                                                                                                                                                            |
| `sources`  | Archives downloaded and extracted into a directory                                                                                                                     |
| `files`    | Files with given content (inline or from a URL), mode, owner, and group                                                                                                |
| `commands` | Commands, with optional `env`, `cwd`, `test` (run only if the test succeeds), and `ignoreErrors`                                                                       |
| `services` | Services to enable and keep running, under `sysvinit`, `systemd`, or `windows`; a service can restart when listed `files`, `sources`, `packages`, or `commands` change |

Within one config key, `cfn-init` always processes the sections in the order shown in the table, whatever order they are written in. Commands run in **alphabetical order of their names**, so prefix them `01_`, `02_` when order matters. Without configsets, `cfn-init` looks for a single config key named `config`. With them, pass `--configsets <name>`.

> [!note] Package manager names on Amazon Linux 2023
> There is no `dnf` key under `packages`. Use `yum`: AL2023 provides the `yum` command as an alias for `dnf`. The `systemd` key under `services` needs `aws-cfn-bootstrap` 2.0-29 or later on Amazon Linux 2 or newer, per the AWS::CloudFormation::Init reference. Older templates, including some course demos, use `sysvinit`.

### Re-running on Updates — cfn-hup

[User data](ec2.md#bootstrapping) runs **only once**, at first launch, and by default so does `cfn-init`. If you change the template's metadata and update the stack, nothing re-applies it. **`cfn-hup`** is a daemon you install and configure on the instance to fix this. It **polls the resource's metadata** and, when the metadata changes, runs a configured **action**, usually `cfn-init` again ([cfn-hup](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-hup.html)).

![CloudFormation cfn-hup example. On the left, a CFN Template whose metadata lists the cfn-init sections packages, groups, users, sources, files, commands and services. Step 1, Change Template, is shown with a refresh icon. Step 2, an UpdateStack arrow, goes to the stack inside a red box. Step 3: cfn-hup, running on an EC2 instance, checks metadata periodically, shown by a two-way arrow between the instance and the stack. Step 4: when updated, cfn-hup calls cfn-init, drawn as a robot with a robotic arm. Step 5: cfn-init applies the new configuration, shown by an orange arrow from the stack's metadata to cfn-init](../assets/cfn-hup-update-flow.png)

The update flow:

1. You change the metadata in the template.
2. You update the stack with that template. A change to metadata only is applied without touching the instance itself.
3. `cfn-hup` notices the change at its next check. The default interval is **15 minutes**, which is why a change can seem not to apply.
4. `cfn-hup` runs its hook action, `cfn-init`.
5. `cfn-init` downloads the new metadata and applies the new desired state on the running instance, with no replacement.

`cfn-hup` is configured by two kinds of files:

| File                      | Contents                                                                                                                                                |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/etc/cfn/cfn-hup.conf`   | A `[main]` section with `stack` (required), plus optional `region`, `interval` (minutes, default 15), `role`, `verbose`, and `umask`                    |
| `/etc/cfn/hooks.d/*.conf` | One section per hook: `triggers` (`post.add`, `post.update`, `post.remove`), `path` (the metadata to watch), `action` (the command to run), and `runas` |

> [!warning] Comparison trap
> "Needs to **re-apply** configuration on a **stack update**, not just at first launch" points to **cfn-init with cfn-hup**, **not** user data. User data is strictly one-shot.

### Example: cfn-init, cfn-hup, and cfn-signal together

The lecture describes these pieces separately. In practice they appear together, as below:

- `cfn-init` installs Apache and writes a page whose text comes from a parameter.
- `cfn-hup` is configured by `cfn-init` itself and re-applies the page when the parameter changes.
- A creation policy holds the stack until the first run succeeds.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Web server configured by cfn-init and kept up to date by cfn-hup

Parameters:
  Message:
    Type: String
    Default: Hello from Aus-E-Mart
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  WebServer:
    Type: AWS::EC2::Instance
    CreationPolicy:
      ResourceSignal:
        Timeout: PT15M
    Metadata:
      AWS::CloudFormation::Init:
        configSets:
          setup:
            - install_cfn
            - install_web
        install_cfn:
          files:
            /etc/cfn/cfn-hup.conf:
              content: !Sub |
                [main]
                stack=${AWS::StackId}
                region=${AWS::Region}
                interval=1
              mode: "000400"
              owner: root
              group: root
            /etc/cfn/hooks.d/cfn-auto-reloader.conf:
              content: !Sub |
                [cfn-auto-reloader-hook]
                triggers=post.update
                path=Resources.WebServer.Metadata.AWS::CloudFormation::Init
                action=/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServer --configsets setup --region ${AWS::Region}
                runas=root
              mode: "000400"
              owner: root
              group: root
            /etc/systemd/system/cfn-hup.service:
              content: |
                [Unit]
                Description=cfn-hup daemon

                [Service]
                Type=simple
                ExecStart=/opt/aws/bin/cfn-hup
                Restart=always

                [Install]
                WantedBy=multi-user.target
          services:
            systemd:
              cfn-hup:
                enabled: true
                ensureRunning: true
                files:
                  - /etc/cfn/cfn-hup.conf
                  - /etc/cfn/hooks.d/cfn-auto-reloader.conf
        install_web:
          packages:
            yum:
              httpd: []
          files:
            /var/www/html/index.html:
              content: !Sub |
                <h1>${Message}</h1>
              mode: "000644"
              owner: root
              group: root
          services:
            systemd:
              httpd:
                enabled: true
                ensureRunning: true
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: t3.micro
      SecurityGroupIds:
        - !GetAtt WebSecurityGroup.GroupId
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash -x
          dnf install -y aws-cfn-bootstrap
          /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource WebServer --configsets setup --region ${AWS::Region}
          /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource WebServer --region ${AWS::Region}

Outputs:
  WebsiteURL:
    Value: !Sub "http://${WebServer.PublicDnsName}"
```

To see `cfn-hup` work, update the stack with a new `Message`. Only the metadata changes, so the instance keeps running. Within about a minute (`interval=1`), `cfn-hup` sees the change, re-runs `cfn-init`, and the page shows the new text.

The systemd unit file follows the shape of the example in the AWS::CloudFormation::Init reference. That page notes that the `cfn-hup` install path varies by platform, so check it on your AMI with `rpm -ql aws-cfn-bootstrap` before relying on `/opt/aws/bin/cfn-hup`. The lecture's own demo predates AL2023 and may configure the service differently.

### Diagnosing on the instance

The user data on a `cfn-init` instance is **short**, only the lines that call `cfn-init` and `cfn-signal`. Reading it back through [IMDS](ec2.md#instance-metadata-service-imds) shows the `${...}` placeholders **already replaced** with real values, proof that CloudFormation substituted them before launch.

```bash
# IMDSv2: get a token first, then read the user data
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/user-data/
```

Relevant logs in `/var/log/`:

| Log file                                   | Source                                                            |
| ------------------------------------------ | ----------------------------------------------------------------- |
| `cloud-init.log` / `cloud-init-output.log` | User-data processing, including the line that launches `cfn-init` |
| `cfn-init.log`                             | `cfn-init`'s own log                                              |
| `cfn-init-cmd.log`                         | Output of each config key and command that `cfn-init` ran         |
| `cfn-hup.log`                              | `cfn-hup` checks and hook runs                                    |

```bash
sudo cat /var/log/cfn-init-cmd.log    # watch each config key run
```

> [!warning] Gotcha
> Exact log file names can vary between helper-script versions. Confirm them on the instance.

### User data vs cfn-init: quick comparison

| Feature                           | **User data**                         | **cfn-init**                                                 |
| --------------------------------- | ------------------------------------- | ------------------------------------------------------------ |
| Configuration model               | Procedural (a script, line by line)   | Procedural **and** desired state                             |
| Execution frequency               | **Once**, at first launch only        | Can **re-run** on stack updates, through `cfn-hup`           |
| Version-aware package install     | No; you script it                     | Yes; declare the version                                     |
| Where the configuration is stored | The user-data block itself            | `AWS::CloudFormation::Init` metadata in the template         |
| How it is invoked                 | The OS runs it at launch              | Invoked **from** user data                                   |
| Knows if configuration succeeded? | No; the resource completes regardless | Yes, when paired with a **creation policy and `cfn-signal`** |
| Complexity                        | Simple                                | More complex, more powerful                                  |
| Exam relevance                    | **Core**                              | Architecture awareness                                       |

For when to bake configuration into an AMI instead of bootstrapping it, see [Boot Time to Service Time](../patterns/boot-time-optimization.md).

## Deletion and Replacement Policies

The next four sections cover operating stacks safely: what CloudFormation may destroy, what you see before a change runs, whose permissions it uses, and how to extend it past what it supports natively.

By default, when a logical resource is removed, CloudFormation **deletes** the physical resource. A resource is removed when you delete it from the template and update the stack, or when you delete the whole stack. For a database or a volume, that means the data is gone. A **`DeletionPolicy`** on the resource changes what happens ([DeletionPolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html)).

![CloudFormation DeletionPolicy example in three columns, each starting from an EC2 instance, an EBS volume and an RDS database created by a stack and then deleted. Delete column: all three become cobwebs, captioned the default process is that when logical resources are removed, individually or via stack delete, the physical resources are also deleted. Retain column: all three remain, captioned physical resources remain untouched when the stack's logical resources are removed. Snapshot column: noted as not supported for EC2; the EC2 instance is deleted, while the EBS volume and the RDS database each leave a snapshot bucket labelled EBS Snapshot and RDS Snapshot](../assets/cfn-deletion-policy.png)

| Policy                 | What happens to the physical resource when its logical resource is removed                                                                                                                  |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Delete`               | Deleted. The default for most resource types                                                                                                                                                |
| `Retain`               | Kept in the account, no longer managed by any stack                                                                                                                                         |
| `RetainExceptOnCreate` | Kept, **except** when the stack's own creation fails and rolls back; then it is deleted, so a failed first deployment does not leave orphans behind                                         |
| `Snapshot`             | A snapshot is taken, then the resource is deleted. Supported only on types that have snapshots, such as EBS volumes, ElastiCache, Neptune, RDS, and Redshift; the reference page lists them |

A few behaviors to know:

- **Defaults are not always `Delete`.** `AWS::RDS::DBCluster`, and `AWS::RDS::DBInstance` without `DBClusterIdentifier`, default to `Snapshot` ([DeletionPolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html)).
- **Snapshots outlive the stack.** They are not deleted with it and keep incurring storage charges until you remove them.
- **EC2 instances cannot be snapshotted** this way. The slide's "EC2 deleted" in the Snapshot column is the instance itself. Volumes defined in its `BlockDeviceMappings` are not separate `AWS::EC2::Volume` resources and get no snapshot.

### Deletion policies do not cover replacement

A `DeletionPolicy` applies only when a logical resource is **removed**. The lecture closes on the important exception: a property change that requires **replacement** also deletes the old physical resource, and `DeletionPolicy` does nothing there. Use **`UpdateReplacePolicy`**, which takes `Delete`, `Retain`, or `Snapshot`, to protect the old resource during a replacement ([UpdateReplacePolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-updatereplacepolicy.html)). Data-bearing resources should normally set both:

```yaml
Resources:
  DataVolume:
    Type: AWS::EC2::Volume
    DeletionPolicy: Snapshot
    UpdateReplacePolicy: Snapshot
    Properties:
      AvailabilityZone: !Select [0, !GetAZs ""]
      Size: 20
      VolumeType: gp3
      Encrypted: true

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

  AuditLogsBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
```

Changing `DataVolume`'s `AvailabilityZone` requires replacement. Without `UpdateReplacePolicy: Snapshot`, the stack update would create a new empty volume and delete the old one with no snapshot. The database uses the account's default VPC when no subnet group is given.

> [!tip] Exam Tip
> "Keep the data when the stack is deleted" means `DeletionPolicy: Retain` or `Snapshot`. "Protect data when an update **replaces** the resource" means `UpdateReplacePolicy`. For "stop anyone deleting the stack at all", the answer is **termination protection** on the stack, which is separate from both.

## Change Sets

A normal stack update applies the new template **immediately**. A **change set** previews instead: you submit the new template, CloudFormation works out what it would change, and nothing happens until you **execute** it ([update stacks using change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)).

![CloudFormation change sets example. Top row: CloudFormation Stack A is created with CreateStack from a template declaring three S3 buckets, koalas, crocs and binchickens. A second template that removes binchickens is applied straight away with UpdateStack, and the binchickens bucket is deleted, marked with a red cross. An ibis raiding a bin sits between the two steps. Bottom row: CloudFormation Stack B is created from the same three-bucket template. The second template is instead submitted as a change set, Stack B ChangeSet Template2, and only when it is executed does Stack B delete binchickens, keeping koalas and crocs](../assets/cfn-change-sets.png)

Both stacks end in the same place. The difference is that the person updating Stack B saw "Remove: binchickens" first and chose to run it. The demo, completed as CLI commands:

```bash
# Template 2 is the original template with the binchickens bucket removed
aws cloudformation create-change-set \
  --stack-name stackb \
  --change-set-name template2 \
  --template-body file://template2.yaml

# Review what would change
aws cloudformation describe-change-set \
  --stack-name stackb \
  --change-set-name template2

# Run it, or delete the change set to discard it
aws cloudformation execute-change-set \
  --stack-name stackb \
  --change-set-name template2
```

`describe-change-set` lists each change. The relevant part of its output, trimmed:

```json
{
  "Changes": [
    {
      "Type": "Resource",
      "ResourceChange": {
        "Action": "Remove",
        "LogicalResourceId": "binchickens",
        "ResourceType": "AWS::S3::Bucket"
      }
    }
  ]
}
```

For a `Modify` action, the `Replacement` field (`True`, `False`, or `Conditional`) tells you whether the change will **replace** the resource. That is the field to read before touching a database.

What change sets add beyond a preview:

- **Many change sets per stack.** Prepare several candidate templates and compare their effects before executing one. Executing one deletes the stack's other change sets.
- **Other change set types.**
  - `--change-set-type CREATE` previews a stack that does not exist yet.
  - `IMPORT` brings existing resources under a stack's management.
  - A **drift-aware** change set (`--deployment-mode REVERT_DRIFT`) also reads the **live** settings of the resources, so it shows, and reverts, changes made outside CloudFormation. See [Drift-aware change sets](#drift-aware-change-sets) below.
- **Validation before deployment.** The lecture presents this as a benefit of change sets: malformed properties and resource name conflicts are caught before anything changes. Since June 2026 these checks run **by default on create stack, update stack, and create change set** operations, and can be turned off per operation with `--disable-validation`. Some extra checks still run **only** when creating a change set, as warnings: whether an S3 bucket to be deleted is empty, service quotas, and AWS Config recorder conflicts ([validate stack deployments](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/validate-stack-deployments.html); [What's New, June 2026](https://aws.amazon.com/about-aws/whats-new/2026/06/aws-cloudformation/)).

> [!tip] Exam Tip
> "Preview the impact of a template change before applying it" or "require approval before production infrastructure changes" points to **change sets**. They are the natural approval step in a CI/CD pipeline.

### Drift-aware change sets

**Drift** is a difference between a resource's live settings and what the stack last deployed. It usually comes from a change made outside CloudFormation, for example in the console, with the CLI, or by another tool. **Drift detection** (`aws cloudformation detect-stack-drift`) reports drift but changes nothing ([detect unmanaged configuration changes](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html)).

A **standard** change set never looks at the resources. It compares the new template with the template from the last deployment, so a change made outside CloudFormation does not appear in the preview. That change also survives the update, unless the new template happens to change the same property ([drift-aware change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/drift-aware-change-sets.html), checked 2026-11-18).

A **drift-aware** change set compares three versions of each resource ([drift-aware change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/drift-aware-change-sets.html)):

1. **Actual state**: the live settings, which CloudFormation reads through the service APIs when you create the change set.
2. **Previous deployment state**: the template from the last deployment.
3. **Desired state**: the new template.

When you execute it, CloudFormation brings every resource in the stack to the desired state, including resources and properties the new template did not change. If the new template is edited to match a resource's live settings, the change set marks the resource as in sync without modifying it.

|                                           | Standard change set                       | Drift-aware change set                                                                                                                                     |
| ----------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Compares                                  | Last template with new template           | Live settings, last template, and new template                                                                                                             |
| Shows changes made outside CloudFormation | No                                        | Yes                                                                                                                                                        |
| On execute                                | Applies only what changed in the template | Also reverts drift on properties the template did not change                                                                                               |
| Rollback after a failed deployment        | To the last deployed template             | To the live settings just before the deployment, so changes made outside CloudFormation are kept                                                           |
| Extra `describe-change-set` fields        | None                                      | `StackDriftStatus`, `ResourceDriftStatus`, `BeforeValueFrom`, `Drift` (`PreviousValue`, `ActualValue`), `ResourceDriftIgnoredProperties`, `DeploymentMode` |
| How to create it                          | Default                                   | `--deployment-mode REVERT_DRIFT`, or **Drift aware change set** as the change set type in the console                                                      |

`StackDriftStatus` is the stack's drift status at the moment the change set was created ([DescribeChangeSet](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_DescribeChangeSet.html)). `BeforeValueFrom` says whether a "before" value came from the live settings (`ACTUAL_STATE`) or the last template (`PREVIOUS_DEPLOYMENT_STATE`).

#### Example: a security group changed in the console

The stack was deployed from this template. It allows HTTPS only:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Web security group (deployed version)

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id

Resources:
  WebSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: HTTPS from anywhere
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Environment
          Value: dev
```

Later, someone adds an inbound SSH rule (TCP 22 from `0.0.0.0/0`) to the group in the console. Then the team deploys `template2.yaml`, which is the same template with only the tag value changed from `dev` to `test`.

```bash
aws cloudformation create-change-set \
  --stack-name web \
  --change-set-name retag \
  --template-body file://template2.yaml \
  --parameters ParameterKey=VpcId,UsePreviousValue=true \
  --deployment-mode REVERT_DRIFT

aws cloudformation describe-change-set \
  --stack-name web \
  --change-set-name retag

aws cloudformation execute-change-set \
  --stack-name web \
  --change-set-name retag
```

- **Without `--deployment-mode REVERT_DRIFT`**, the change set shows only the tag change. After execution, the SSH rule is still open.
- **With it**, the change set also shows `SecurityGroupIngress` as drifted, with the SSH rule in the live settings but not in the template. Executing it changes the tag and removes the SSH rule.

The templates pass cfn-lint 1.60.1. The stack was not deployed, so the described change set output comes from the AWS documentation, not from a live run.

#### What a drift-aware change set does not revert

The [drift-aware change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/drift-aware-change-sets.html) page lists these exceptions (checked 2026-11-18):

- **Properties AWS manages**, when the template does not set them. Examples: RDS `AutoMinorVersionUpgrade`, the capacity of an Application Auto Scaling scalable target, and Auto Scaling group scaling policies. These appear in `ResourceDriftIgnoredProperties` with a reason.
- **Tag keys that are not in the template**. Tags added outside CloudFormation stay.
- **Write-only properties** such as passwords. AWS cannot read them back, so they are compared with the last deployed value, not the live one.
- **Drift on properties that cannot be changed in place**, the ones whose change would replace the resource.
- **About 27 resource types** listed on the page, such as `AWS::MSK::Cluster` and `AWS::RDS::DBProxy`. For these, and for any type that does not support drift detection, the change set compares only the last template with the new one, like a standard change set.

Drift on resources that another stack attaches to this one, such as an `AWS::IAM::Policy` or an `AWS::EC2::SecurityGroupIngress`, is detected, and the change set can revert the attachment.

> [!warning] Gotcha
> The page lists `DELETED` as a possible `ResourceDriftStatus` but does not say what executing the change set does to a resource deleted outside CloudFormation. It also does not cover nested stacks or StackSets. Test those cases before relying on them.

> [!tip] Exam Tip
> "Preview a template change" points to a **change set**. "Find out which resources were changed outside CloudFormation" points to **drift detection**, which only reports. "Bring the resources back in line with the template" points to a **drift-aware change set**.

## Stack Roles

By default, CloudFormation acts **with the permissions of whoever runs the stack operation**. To create a stack containing an EC2 instance, a DynamoDB table, and a Lambda function, you need permission to manage stacks **and** to create all three. That conflicts with a common split of duties, where one team defines infrastructure and another, less privileged team operates it.

A **stack role** (AWS calls it a **service role**) fixes this. CloudFormation **assumes an IAM role** and uses the role's permissions for every operation on the stack. The person running the operation needs only stack permissions plus **`iam:PassRole`** for that role ([CloudFormation service role](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-servicerole.html)).

![CloudFormation stack roles example. On the left, Nat, the account admin, creates an IAM role, drawn as a red helmet, with permissions to create, update and delete AWS resources; a note says Scott has no permissions to assume it or edit it, only to pass it into CFN. In the middle, an AWS account holds an EC2 instance, a DynamoDB table and a Lambda function. On the right, Scott, from the account help desk, only needs permissions to create, update or delete stacks and to pass in the role; he doesn't need permissions to manipulate resources. An orange line carries the role from Nat to Scott's stack, captioned role is attached to the stack and used for any operations, and a red arrow shows the stack acting on the resources in the account](../assets/cfn-stack-roles.png)

1. Nat, an administrator, creates a role that CloudFormation can assume, with permission to manage the resources the stack needs.
2. Scott, on the help desk, can create, update, and delete stacks, and can **pass** that role to CloudFormation. He cannot assume it, cannot edit it, and has no direct permission on EC2, DynamoDB, or Lambda.
3. Scott creates a stack from Nat's template and selects the role. CloudFormation uses the role for that operation and **every later one** on the stack.

The lecture shows this only as a diagram. The IAM side, completed:

The role's **trust policy** lets CloudFormation assume it:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "cloudformation.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Scott's **identity policy** lets him manage stacks only when they use that role, and pass the role only to CloudFormation. The role ARN uses an example account ID:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadStacks",
      "Effect": "Allow",
      "Action": [
        "cloudformation:Describe*",
        "cloudformation:List*",
        "cloudformation:GetTemplate*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ManageStacksOnlyWithTheStackRole",
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DeleteStack",
        "cloudformation:CreateChangeSet"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "cloudformation:RoleArn": "arn:aws:iam::111111111111:role/AEMStackRole"
        }
      }
    },
    {
      "Sid": "RunChangeSets",
      "Effect": "Allow",
      "Action": [
        "cloudformation:ExecuteChangeSet",
        "cloudformation:DeleteChangeSet"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PassOnlyTheStackRoleToCloudFormation",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::111111111111:role/AEMStackRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "cloudformation.amazonaws.com"
        }
      }
    }
  ]
}
```

Scott then deploys with the role:

```bash
aws cloudformation create-stack \
  --stack-name aem-app \
  --template-body file://app.yaml \
  --role-arn arn:aws:iam::111111111111:role/AEMStackRole
```

The `cloudformation:RoleArn` condition stops Scott from creating a stack **without** the role, which would make CloudFormation fall back to his own, smaller permissions. `iam:PassRole` and `iam:PassedToService` are explained on the [IAM page](iam.md) and in [Identity and Access](../concepts/identity-and-access.md).

> [!warning] Gotcha
> The role, not the template, is the real permission boundary. Anyone who can pass the role can deploy **any** template that the role's permissions allow, not only Nat's. Scope the role to what the stacks genuinely need. If the role is later deleted or loses permissions, operations on stacks that use it fail, including deletion.

## Custom Resources

CloudFormation cannot do everything. It can lag behind new AWS features, some actions it will never support, and it knows nothing about systems outside AWS. A **custom resource** fills the gap: a logical resource whose create, update, and delete operations are handled by **your own code** ([custom resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)). Typical uses:

- Populate an S3 bucket with objects when it is created.
- Empty a bucket before CloudFormation deletes it.
- Look up a value CloudFormation cannot, such as configuration from an external system.
- Register the stack with a third-party ticketing or monitoring system.
- Provision non-AWS resources.

### How a custom resource works

1. The template declares a resource of type `AWS::CloudFormation::CustomResource` or `Custom::AnyName`. Its required `ServiceToken` property is the ARN of a **Lambda function** or an **SNS topic** in the **same Region** as the stack.
2. On create, update, or delete, CloudFormation sends that target a JSON event ([request and response reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref.html)) containing:
   - `RequestType`: `Create`, `Update`, or `Delete`.
   - `ResourceProperties`: the properties from the template.
   - `ResponseURL`: a presigned S3 URL to reply to.
   - `StackId`, `RequestId`, `LogicalResourceId`, `PhysicalResourceId` (on update and delete), and `OldResourceProperties` (on update).
3. Your code does the work, then `PUT`s a JSON response to `ResponseURL`. The response contains `Status` (`SUCCESS` or `FAILED`), `Reason`, `PhysicalResourceId`, `StackId`, `RequestId`, `LogicalResourceId`, and optionally `Data`. Other resources can read `Data` values with `!GetAtt`. The response is limited to 4,096 bytes.
4. If no response arrives within `ServiceTimeout` (1 to 3,600 seconds, default 3,600), the operation **fails** ([AWS::CloudFormation::CustomResource](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-customresource.html)).

![CloudFormation custom resources example. Top panel, without custom resource: Template, Create Stack, Stack, Create bucket, Physical Resource; Nat then manually uploads objects, giving a bucket with objects; a Stack Delete then tries to delete the bucket and stops at a barrier, Error - Bucket not empty. Bottom panel, custom resource: the stack creates a bucket and sends an event to CustomLambda, which is invoked, receives the event, downloads objects into the bucket, and replies through the ResponseURL with a green tick. Nat adds more objects to the bucket. On Stack Delete the stack again sends an event to the Lambda function, which removes the objects from the bucket and replies through the ResponseURL, after which the stack deletes the now-empty bucket](../assets/cfn-custom-resource-empty-bucket.png)

### Example: populate a bucket on create, empty it on delete

The lecture's example has two halves:

- **Without** a custom resource, deleting the stack fails because someone uploaded objects and S3 will not delete a non-empty bucket.
- **With** one, a Lambda function copies starter objects into the bucket on create and removes every object on delete, so the stack deletion succeeds.

The slide shows only the flow. A complete template:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Bucket populated on create and emptied on delete by a custom resource

Parameters:
  SourceBucketName:
    Type: String
    Description: Existing bucket in this Region holding the starter objects to copy

Resources:
  AnimalPicsBucket:
    Type: AWS::S3::Bucket

  S3ObjectsFunctionRole:
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
      Policies:
        - PolicyName: ManageObjects
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action:
                  - s3:ListBucket
                  - s3:ListBucketVersions
                Resource: !GetAtt AnimalPicsBucket.Arn
              - Effect: Allow
                Action:
                  - s3:PutObject
                  - s3:DeleteObject
                  - s3:DeleteObjectVersion
                Resource: !Sub "${AnimalPicsBucket.Arn}/*"
              - Effect: Allow
                Action: s3:ListBucket
                Resource: !Sub "arn:${AWS::Partition}:s3:::${SourceBucketName}"
              - Effect: Allow
                Action: s3:GetObject
                Resource: !Sub "arn:${AWS::Partition}:s3:::${SourceBucketName}/*"

  S3ObjectsFunction:
    Type: AWS::Lambda::Function
    Properties:
      Runtime: python3.13
      Handler: index.handler
      Role: !GetAtt S3ObjectsFunctionRole.Arn
      Timeout: 300
      Code:
        ZipFile: |
          import boto3
          import cfnresponse

          s3 = boto3.resource("s3")

          def handler(event, context):
              props = event["ResourceProperties"]
              bucket = s3.Bucket(props["BucketName"])
              try:
                  if event["RequestType"] == "Create":
                      for obj in s3.Bucket(props["SourceBucket"]).objects.all():
                          bucket.copy({"Bucket": obj.bucket_name, "Key": obj.key}, obj.key)
                  elif event["RequestType"] == "Delete":
                      # Deletes every object version and delete marker; also works on unversioned buckets
                      bucket.object_versions.delete()
                  cfnresponse.send(event, context, cfnresponse.SUCCESS, {}, props["BucketName"])
              except Exception as exc:
                  print(exc)
                  cfnresponse.send(event, context, cfnresponse.FAILED, {}, props["BucketName"])

  S3Objects:
    Type: Custom::S3Objects
    Properties:
      ServiceToken: !GetAtt S3ObjectsFunction.Arn
      ServiceTimeout: 300
      BucketName: !Ref AnimalPicsBucket
      SourceBucket: !Ref SourceBucketName
```

Why this works, and the traps it avoids:

- **Order is what makes the delete work.** `S3Objects` references the bucket and the function, so CloudFormation creates it **after** both and deletes it **before** both. The function empties the bucket first, then CloudFormation deletes the empty bucket, the function, and its role.
- **Always respond.** Every code path, including the exception path, calls `cfnresponse.send`. A function that crashes without responding leaves the stack waiting until `ServiceTimeout` expires. With the default timeout, that is a full hour.
- **Keep the physical ID stable.** Returning the same `PhysicalResourceId` (here the bucket name) tells CloudFormation the resource was updated in place. Returning a new one is treated as a **replacement**, and CloudFormation then sends a `Delete` for the old ID, which here would empty the bucket.
- **`cfnresponse` is only available to inline code.** CloudFormation provides the module when the function code is written inline in `ZipFile`. Packaged code must include its own copy or send the HTTPS `PUT` itself.

> [!note] Current AWS behavior
> The lecture's premise still holds: CloudFormation will not delete a non-empty S3 bucket, and pre-deployment validation now warns about it when you create a change set. If a stack is stuck in `DELETE_FAILED` for this reason, you can retry the deletion and keep the bucket with `--retain-resources`. `--deletion-mode FORCE_DELETE_STACK` deletes the stack while leaving the failing resources in place ([delete a stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html)). Neither option empties the bucket.

## Beyond One Stack

A stack is isolated: nothing in it can `Ref` a resource in another stack. When a solution outgrows one stack, CloudFormation offers three designs, compared in detail on [CloudFormation Multi-Stack Architectures](../patterns/cloudformation-multi-stack.md):

| Design                                                                                     | Use when                                                                                                                                             |
| ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Nested stacks](../patterns/cloudformation-multi-stack.md#nested-stacks)                   | One solution, split into reusable templates, with every part sharing **one lifecycle**. Also the workaround for the 500-resources-per-template quota |
| [Cross-stack references](../patterns/cloudformation-multi-stack.md#cross-stack-references) | Independently managed stacks share a long-lived resource such as a central VPC, through Export and `Fn::ImportValue` or `Fn::GetStackOutput`         |
| [StackSets](../patterns/cloudformation-multi-stack.md#stacksets)                           | The same template deployed as many stacks across **many accounts and Regions**, often through [AWS Organizations](organizations.md)                  |

## Exam Recap

- **Templates** declare **logical resources**; a **stack** creates, updates, and deletes the matching **physical resources** and rolls back a failed creation.
- An update to a property causes **no interruption**, **some interruption**, or **replacement**. Replacement creates a new physical resource, with possible data loss.
- **Portable templates** avoid hard-coded names and IDs. Leave out `BucketName`, use **pseudo parameters**, `!GetAZs`, and an **SSM public parameter** for the latest AMI.
- `!Ref` returns a parameter's value or a resource's main ID; `!GetAtt` returns other attributes. Both create **implicit dependencies**.
- `Fn::Base64` is required for user data in templates. `Fn::Sub` cannot reference the resource it is inside; use IMDS and `${!Var}` instead.
- **Mappings** are static lookups with `!FindInMap [Map, TopKey, SecondKey]`. **Conditions** are evaluated first and switch resources or properties on and off.
- `DependsOn` forces order when nothing is referenced, as with an EIP or route after the **internet gateway attachment**.
- A **creation policy with `cfn-signal`** is how CloudFormation learns that in-instance bootstrapping succeeded. No signal within the timeout counts as failure. Use it on EC2 instances and Auto Scaling groups. **Wait conditions** are a separate resource that can gate other resources and return data.
- **cfn-init** applies desired state from `AWS::CloudFormation::Init` metadata; **cfn-hup** re-runs it when a stack update changes the metadata, every 15 minutes by default. User data is one-shot.
- **`DeletionPolicy`** (`Delete`, `Retain`, `RetainExceptOnCreate`, `Snapshot`) covers removal; **`UpdateReplacePolicy`** covers replacement. RDS defaults to `Snapshot`.
- **Change sets** preview a change before execution and show whether a resource will be replaced.
- **Stack roles** let CloudFormation use a role's permissions, so operators need only stack permissions plus `iam:PassRole`.
- **Custom resources** run your Lambda function or SNS-subscribed code for anything CloudFormation cannot do natively, such as emptying a bucket before deletion.
- **Nested stacks** share a lifecycle, **cross-stack references** share resources between independent stacks, and **StackSets** deploy across accounts and Regions.

## References

- Adrian Cantrill, AWS Certified Solutions Architect – Associate (SAA-C03) ([course](https://learn.cantrill.io/)). CloudFormation lessons:
  - "CloudFormation Physical Logical Resources"
  - "CloudFormation Template and Pseudo Parameters"
  - "CloudFormation Intrinsic Functions"
  - "CloudFormation Mappings"
  - "CloudFormation Outputs"
  - "CloudFormation Conditions"
  - "CloudFormation DependsOn"
  - "CloudFormation Wait Conditions cfn-signal"
  - "CloudFormation Init CFN-INIT"
  - "CloudFormation cfn-hup"
  - "CloudFormation Deletion Policy"
  - "CloudFormation Change Sets"
  - "CloudFormation Stack Roles"
  - "CloudFormation Custom Resources"
- Adrian Cantrill, SAA-C03, Advanced EC2 lessons: "Enhanced Bootstrapping with CFN-INIT" and "CFN Creation Policies"
- [AWS: How CloudFormation works](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-overview.html)
- [AWS: Template sections](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html)
- [AWS: Update behaviors of stack resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html)
- [AWS: Parameters section](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html)
- [AWS: Pseudo parameters reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/pseudo-parameter-reference.html)
- [AWS: Dynamic references](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/dynamic-references.html)
- [AWS: Intrinsic function reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html), including [Fn::GetAZs](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-getavailabilityzones.html), [Fn::Sub](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-sub.html), [Fn::Cidr](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-cidr.html), and [Fn::FindInMap](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference-findinmap.html)
- [AWS: AWS::EC2::SecurityGroup return values](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-ec2-securitygroup.html#aws-resource-ec2-securitygroup-return-values)
- [AWS: Conditions section](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/conditions-section-structure.html)
- [AWS: Outputs section](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html)
- [AWS: DependsOn attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-dependson.html)
- [AWS: CreationPolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-creationpolicy.html)
- [AWS: UpdatePolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-updatepolicy.html)
- [AWS: AWS::CloudFormation::WaitCondition](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-waitcondition.html)
- [AWS: CloudFormation helper scripts reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/cfn-helper-scripts-reference.html)
- [AWS: AWS::CloudFormation::Init](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-init.html)
- [AWS: cfn-hup](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-hup.html)
- [AWS: DeletionPolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html)
- [AWS: UpdateReplacePolicy attribute](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-updatereplacepolicy.html)
- [AWS: Update stacks using change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html)
- [AWS: Drift-aware change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/drift-aware-change-sets.html) (checked 2026-11-18)
- [AWS: Detect unmanaged configuration changes to stacks and resources with drift detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html)
- [AWS API Reference: DescribeChangeSet](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_DescribeChangeSet.html)
- [AWS: Validate stack deployments](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/validate-stack-deployments.html) (verified 2026-11-18)
- [AWS What's New: pre-deployment validation on all stack operations (June 2026)](https://aws.amazon.com/about-aws/whats-new/2026/06/aws-cloudformation/)
- [AWS: CloudFormation service role](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-servicerole.html)
- [AWS: Custom resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-custom-resources.html)
- [AWS: Custom resource request and response reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref.html)
- [AWS: AWS::CloudFormation::CustomResource](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-customresource.html)
- [AWS: Delete a stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html)
- [AWS: Amazon Linux 2 FAQs](https://aws.amazon.com/amazon-linux-2/faqs/)
- [AWS: Amazon VPC pricing (public IPv4 addresses)](https://aws.amazon.com/vpc/pricing/)
- Related wiki pages: [EC2: Bootstrapping](ec2.md#bootstrapping), [EC2: Instance Metadata Service](ec2.md#instance-metadata-service-imds), [Boot Time to Service Time](../patterns/boot-time-optimization.md), [CloudFormation Multi-Stack Architectures](../patterns/cloudformation-multi-stack.md)
