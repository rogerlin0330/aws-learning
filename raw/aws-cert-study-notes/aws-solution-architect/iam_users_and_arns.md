## Summary of IAM Users and Related Concepts in AWS

This lesson expands on the concept of **IAM users**, providing critical details needed for AWS certification exams and practical AWS account management.

![[iam-users-and-arns-infographic.png]]

---

## Core Concepts

- **IAM Users**: Identities in AWS used for **long-term access** by entities such as humans, applications, or service accounts. Examples include named individuals (e.g., James from accounts) or applications running on laptops.
- **Principals**: Entities (people, computers, services, or groups) attempting to access AWS resources. Initially unidentified, they must authenticate and then be authorized to perform actions.
- **Authentication**: The process where a principal proves its claimed identity to AWS using:
  - **Username and password** (typically for humans via AWS Console)
  - **Access keys** (for applications or command-line interface users)
- **Authorization**: After authentication, AWS checks policies attached to the IAM user to determine if the requested action on a resource is permitted.

---

## Authentication vs. Authorization

| Aspect            | Description                                              |
|-------------------|----------------------------------------------------------|
| Authentication    | Validates identity using credentials (username/password or access keys) |
| Authorization     | Verifies permissions based on policies linked to the IAM user |

---

## Amazon Resource Names (ARNs)

- **Purpose**: ARNs uniquely identify AWS resources across all accounts and regions.
- **Usage**: Required in IAM policies to specify resources precisely.
- **Format**: Composed of fields separated by colons, typically:

  $$ \text{arn} : \text{partition} : \text{service} : \text{region} : \text{account-id} : \text{resource} $$

- **Key ARN Fields**:
  - **Partition**: Usually `aws` for standard regions; can be `aws-cn` for China regions.
  - **Service**: AWS service namespace (e.g., `s3`, `iam`, `rds`).
  - **Region**: AWS region; may be omitted or wildcarded depending on the service.
  - **Account ID**: AWS account number owning the resource; omitted if globally unique (e.g., S3 buckets).
  - **Resource**: Resource identifier or path (e.g., bucket name, user name).

- **Important ARN Distinctions**:
  - Example for S3 bucket ARN:

    | ARN Example                                   | Meaning                          |
    |----------------------------------------------|---------------------------------|
    | `arn:aws:s3:::cat-gifts`                      | The bucket itself               |
    | `arn:aws:s3:::cat-gifts/*`                    | All objects inside the bucket  |

  - **Bucket ARN** refers only to the bucket, not objects.
  - **Bucket objects ARN** refers only to objects, not the bucket.
  - Policies may need both ARNs to allow operations on both bucket and objects.

- **Wildcards vs. Omission**:
  - Using `*` is a wildcard to match multiple resources.
  - Double colons `::` indicate omitted fields (not the same as wildcard).
  - Wildcards are used to refer to all regions or multiple resources.

---

## IAM User Limits and Design Considerations

| Limit                      | Description                                     |
|----------------------------|------------------------------------------------|
| Maximum IAM users per account | **5,000 IAM users** (global per account, not per region) |
| Maximum IAM groups per user   | **10 groups**                                   |

- **Implications**:
  - Systems requiring more than 5,000 distinct identities should **not use IAM users** individually.
  - Large-scale applications (e.g., internet-scale with millions of users) or large organizations (with >5,000 staff) need alternative identity management.
- **Alternatives**:
  - **IAM roles**
  - **Identity federation** (using existing external identities)
- These alternatives allow scalable, flexible identity solutions beyond IAM user limits.

---

## Key Takeaways

- **IAM users represent long-term, named identities** used for accessing AWS resources.
- Authentication and authorization are **distinct steps**: verifying identity and verifying permissions.
- **ARNs are critical** for uniquely identifying AWS resources; understanding their structure is essential.
- The **difference between bucket-level and object-level ARNs in S3 is crucial** to avoid policy errors.
- **IAM user limits (5,000 users per account)** impose architectural decisions; large-scale deployments require roles or federation.
- Understanding these concepts is vital for passing the AWS certification exam and for designing effective AWS security architectures.

---

## Summary Timeline (Conceptual Flow)

| Step                     | Description                                            |
|--------------------------|--------------------------------------------------------|
| IAM User Creation        | Create IAM users for humans, applications, or services needing access |
| Authentication           | User proves identity using credentials (password or access key) |
| Authorization            | AWS checks policies to allow or deny requested actions |
| ARN Usage                | Specify resources in policies using ARNs               |
| Policy Application       | Attach policies referencing ARNs to IAM users          |
| Recognize Limits         | Understand 5,000 user limit and 10 groups per user limit |
| Alternative Identities   | Use IAM roles or federation for large-scale identity management |

---

This summary covers foundational and advanced aspects of IAM users, authentication, authorization, ARNs, and account limits, strictly based on the source content.