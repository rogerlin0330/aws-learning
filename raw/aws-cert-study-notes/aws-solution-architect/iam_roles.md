# Understanding IAM Roles in AWS – Architecture, Functionality, and Usage

![[iam-roles-infographic.png]]

### Introduction: The Significance of IAM Roles in AWS Identity Management  
- [00:00:00 ~ 00:00:48] This chapter delves into **IAM roles**, a fundamental yet often complex concept in **AWS Identity and Access Management (IAM)**. IAM roles represent one of the types of **identities** within an AWS account, distinct from IAM users. Understanding roles is crucial because they govern how access permissions are temporarily granted to various entities, enhancing flexibility and security in AWS environments.  
- **Key terms introduced:**  
  - **Identity:** An entity that can be authenticated and authorized to access AWS resources, such as a **principal** (a physical person, application, device, or process).  
  - **IAM user:** An identity designed for a single principal, typically long-term and directly associated with one person or application.  
  - **IAM role:** An identity used for multiple or unknown principals, often assumed temporarily to grant permissions.  
- The chapter focuses on the architecture of roles, the technical mechanics of assuming them, and their differentiation from IAM users, setting the foundation for practical application and best practices.

---

### Section 1: Defining IAM Roles and Their Purpose  
- [00:00:48 ~ 00:02:07] Roles differ fundamentally from IAM users in terms of their intended usage and flexibility. While IAM users are tailored for **single, identifiable principals**, roles are created for **unknown or multiple principals**. This includes:  
  - Multiple AWS users within the same account.  
  - External entities such as humans, applications, or services, either inside or outside the AWS account.  
- Roles are often chosen when the number of principals is large or undefined, such as when exceeding the IAM user limit of approximately **5,000 principals**.  
- Crucially, roles represent a **level of access**, not a specific individual or entity. They are assumed **temporarily**, providing short-term permissions that a principal can "borrow" to perform tasks.  
- Unlike IAM users, roles do not permanently represent a principal; roles are **transient identities** that facilitate secure, temporary access.

---

### Section 2: The Temporary Nature of IAM Roles  
- [00:02:07 ~ 00:03:47] Once a principal assumes a role, they effectively **become that role** for a limited time, inheriting the permissions attached to it.  
- This temporary assumption is a core conceptual distinction from IAM users, who provide persistent identity and permissions.  
- The temporary credentials generated when assuming a role enable the principal to access AWS resources, but after expiration, the credentials must be renewed by re-assuming the role.  
- This mechanism is particularly useful for external or mobile applications needing controlled and time-limited access to AWS resources without long-term credentials.  
- The speaker emphasizes the common confusion between **logging in as an IAM user** and **assuming a role**, noting that while both grant access based on permissions, the operational and security models differ significantly.

---

### Section 3: Policies Associated with IAM Roles  
- [00:04:10 ~ 00:05:42] IAM roles involve two distinct types of policies:  
  - **Trust policy:** Defines **which principals are allowed to assume the role**. It acts as a gatekeeper by specifying trusted identities that can take on the role.  
    - Can include IAM users, other roles, AWS services (e.g., EC2), identities from other AWS accounts, and even federated identities from external providers such as Facebook, Twitter, or Google.  
    - It can even enable anonymous access under certain configurations.  
  - **Permissions policy:** Specifies **the actions and resources the role can access** once assumed. This governs the actual authorization granted during the temporary session.  
- The example clarifies that only identities explicitly allowed in the trust policy can assume the role; all others are denied by default.  
- This dual-policy architecture ensures strict control over both **who can assume a role** and **what permissions the assumed role provides**.

---

### Section 4: Temporary Security Credentials and Access Control  
- [00:05:42 ~ 00:06:59] When a role is assumed, AWS generates **temporary security credentials** using the **Secure Token Service (STS)**, specifically through the operation **STS:AssumeRole**.  
- These credentials function similarly to access keys but are **time-limited**, expiring after a predefined duration.  
- Once expired, the principal must re-assume the role to obtain new credentials, ensuring that access is continuously controlled and refreshed.  
- The permissions granted by these temporary credentials are dynamically enforced by AWS by checking the associated permissions policy each time the credentials are used.  
- Any changes to the permissions policy immediately affect the abilities of the temporary credentials, providing real-time permission adjustments.  
- Importantly, roles, like IAM users, can be referenced in **resource policies**, such as those granting access to an **S3 bucket**, meaning that permissions can be validated both from the role side and the resource side.

---

### Section 5: Real-World Application: Roles in AWS Organizations  
- [00:06:59 ~ 00:07:43] IAM roles play a vital role in **AWS Organizations**, AWS’s service for managing multiple accounts under a single framework.  
- Roles enable seamless cross-account access without requiring multiple logins, improving efficiency and security in complex, multi-account environments.  
- By assuming roles, users can move between accounts within an organization while maintaining proper access controls.  
- This use case highlights the practical importance of roles in scaling AWS environments and managing permissions centrally across many accounts.

---

### Section 6: Technical Summary and Transition to Usage Scenarios  
- [00:07:43 ~ 00:08:14] The technical foundation covered includes:  
  - The distinction and interplay between **trust policies** and **permissions policies**.  
  - The generation and management of **temporary security credentials** through **STS**.  
  - The conceptual framework for **assuming roles**, emphasizing temporary identity assumption rather than persistent association.  
- The next lesson promises to explore **practical scenarios** for role usage, aiming to clarify when and why roles should be implemented instead of IAM users or other identity mechanisms.  
- This staged approach encourages learners to build conceptual understanding first, then apply knowledge in realistic contexts.

---

### Conclusion: Key Takeaways and Implications  
- IAM roles are a **critical AWS IAM identity type** designed for flexible, temporary access by multiple or unknown principals.  
- Their **temporary nature** and dual-policy management system (trust and permissions policies) provide a robust security model that balances accessibility with control.  
- The generation of **temporary credentials** via STS ensures that access is limited in duration and scope, reducing risk.  
- Roles are especially important in **large-scale AWS environments**, such as those managed by AWS Organizations, enabling efficient cross-account access and streamlined identity management.  
- Understanding roles requires distinguishing them clearly from IAM users, focusing on their transient usage and multi-principal applicability.  
- Mastery of IAM roles is foundational for secure, scalable AWS identity and access management.

---

### Advanced Bullet-Point Summary:

- **Identity Types in AWS IAM:**  
  - IAM users: Single principal, long-term identity.  
  - IAM roles: Multiple/unknown principals, temporary identity.  

- **Principals:** Physical person, application, device, or process requiring authentication.  
- **Authentication vs. Authorization:** Authentication proves identity; authorization grants resource access.  

- **Role Characteristics:**  
  - Represent access level, not a specific individual.  
  - Temporarily assumed by principals.  
  - Suitable for unknown number of principals or large scale (>5,000 users).  
  - Permissions "borrowed" for limited time.  

- **Policy Types for Roles:**  
  - *Trust policy:* Specifies who can assume the role (users, roles, services, external identities).  
  - *Permissions policy:* Defines what actions the assumed role can perform.  

- **Temporary Credentials:**  
  - Generated by AWS STS via **STS:AssumeRole**.  
  - Time-limited, requiring renewal upon expiration.  
  - Permissions dynamically enforced and updated.  

- **Resource Access:** Roles can be referenced in resource policies; access depends on both role and resource permissions.  

- **Use Case – AWS Organizations:**  
  - Roles enable cross-account access without multiple logins.  
  - Essential for multi-account management and security.  

- **Key Conceptual Distinction:**  
  - IAM users = persistent identity.  
  - IAM roles = temporary, assumed identity.  

- **Next Steps:** Practical scenarios and use cases to further clarify role application.  

[00:00:00] This lesson focuses on **when and where to use IAM roles** in AWS, emphasizing practical scenarios to help users understand appropriate role usage both in real-world environments and exam contexts. The goal is to ensure comfort in selecting situations suited for IAM roles and recognizing when not to use them.

[00:00:20] **IAM roles are commonly used within the same AWS account by AWS services themselves.** These services operate on behalf of users and need specific access rights to perform actions. For example, AWS Lambda— a function-as-a-service product—requires permissions to execute its tasks, such as managing EC2 instances or processing data.

[00:01:05] Lambda functions have **no default permissions** because they are not AWS identities but components of AWS services. To perform actions when invoked (executed), Lambda needs a secure way to obtain permissions. This introduces the concept of **Lambda execution roles**.

[00:01:35] For any non-AWS identity entity, such as custom applications or scripts, permissions are typically granted via **access keys**. Instead of hardcoding access keys in Lambda functions—a security risk and maintenance challenge—AWS recommends using an **IAM role with a trust policy** that allows the Lambda service to assume the role during function execution.

[00:02:04] The **Lambda execution role** includes:

- A **trust policy** that trusts the Lambda service.
- A **permissions policy** granting necessary access to AWS resources.

During execution, Lambda uses the **STS (Security Token Service) assume role operation** to acquire **temporary security credentials**. These credentials are used by the Lambda runtime environment to access AWS resources securely.

[00:02:49] The key point is that the **runtime environment assumes the role**, obtains temporary credentials, and the code runs within this secure context. This eliminates the need for static credentials hardcoded in the function.

[00:03:12] **Why use roles here?**

- Avoids storing static access keys, reducing security risks.
- Enables credential rotation without code changes.
- Provides temporary credentials valid only during the task.
- Supports scaling, as Lambda can run multiple concurrent copies (unknown number of principals), matching the rule that roles are ideal when the number of principals is uncertain or multiple.

[00:04:21] **Summary:** Using roles is the preferred method for AWS services acting on your behalf, especially when static credentials are impractical or insecure.

[00:04:51] Another important use case for IAM roles is **emergency or out-of-the-ordinary access scenarios** (often called **break glass scenarios**). The example involves Wayne, a service desk team member with read-only access to a customer’s AWS account.

[00:05:07] Normally, Wayne’s team only has read-only permissions, but in emergencies (e.g., at 3 a.m.) they may need elevated access to stop, start, or terminate EC2 instances. To handle this securely:

- Wayne assumes an **emergency IAM role**.
- This role provides temporary elevated permissions.
- The action is logged, ensuring accountability.
- Wayne’s normal access remains restricted, protecting both him and the customer.

[00:05:58] The **break glass analogy**:

- Physical glass barriers protect keys that only get used intentionally.
- In AWS, roles act as this “glass,” enabling extra permissions only when truly needed.

[00:06:34] This method ensures **controlled, auditable, temporary privilege escalation** without granting permanent high-level permissions.

[00:07:04] Another common scenario is **integrating AWS with an existing corporate identity provider (IdP)**, such as **Microsoft Active Directory**. In such environments:

- Staff use existing credentials to log in to various systems.
- Instead of creating an IAM user for each employee (impractical due to the **5,000 IAM user limit per account**), roles allow identity federation.

[00:07:59] Key facts:

| Concept                            | Details                                      |
|----------------------------------|----------------------------------------------|
| IAM User Limit                   | Hard limit at 5,000 users per AWS account    |
| Identity Federation              | Use roles to allow external identities to access AWS resources |
| Direct Access by External Identities | Not possible (e.g., Active Directory, Facebook) |

[00:08:23] External identities **cannot directly access** AWS resources. Instead, an IAM role is configured to be assumed by these external identities, generating temporary credentials for accessing AWS resources.

[00:09:22] While the user experience is seamless via the AWS console UI, **underlying processes involve using roles for federated access**, which will be covered in detail later in the course.

[00:09:48] **Advantages of using roles for corporate IdPs:**

- Avoids hitting the 5,000 IAM user limit.
- Simplifies management by reducing the number of IAM users.
- Allows external identities to assume a small set of roles for resource access.

[00:10:08] This process is called **Identity Federation** and is essential for large organizations or those using existing corporate identity solutions.

[00:10:37] Another popular use case for roles is in **mobile or web applications with millions of users** (e.g., ride-sharing apps) accessing AWS resources like DynamoDB.

[00:10:55] Challenges:

- Each AWS identity (e.g., IAM user) is needed to interact with AWS resources.
- The 5,000 IAM user limit makes it impossible to create users for all app customers.

[00:11:12] The solution is **Web Identity Federation**, using IAM roles to allow users authenticated by third-party web identities (e.g., Google, Twitter, Facebook) to assume roles and obtain temporary credentials.

[00:11:48] This approach relies on the **role’s trust policy** to permit these web identities to assume the role and interact with AWS resources securely.

[00:12:06] **Advantages of using roles in mobile/web applications:**

- No AWS credentials stored in the application, enhancing security.
- Leverages existing user accounts, avoiding the need for additional account creation.
- Scales seamlessly to hundreds of millions of users.
- Bypasses the 5,000 IAM user limit.

[00:13:09] This architecture is often tested in AWS certification exams, highlighting the importance of understanding **ID Federation with IAM roles** for scalable app designs.

[00:13:34] The final scenario explained is **cross-account access** using roles, a common pattern in multi-account AWS environments.

[00:14:00] Example:

- Two AWS accounts: your own and a partner’s.
- Partner provides an application requiring storage of data in their S3 bucket.
- Partner IT does not want to create IAM users for all your staff.

Solution:

- Use an IAM role in the partner’s account.
- Your users assume that role to get temporary credentials.
- Actions like uploading to S3 are authorized under the partner account’s identity.

[00:14:25] This method ensures resource ownership remains with the partner account and simplifies permissions management.

[00:14:41] Roles can provide **granular access to specific resources** (like an S3 bucket) or **access to entire accounts**. An upcoming AWS Organizations demo will showcase multi-account role usage to simplify cross-account management.

[00:14:57] The lesson concludes by reassuring learners that understanding roles is complex initially but will improve with experience and further course content.

---
### Summary of Key IAM Role Use Cases

![[iam-roles-infographic-cont.png]]

| Use Case                   | Description                                                                                          | Benefits                                                                                           |
|----------------------------|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| AWS Service Access          | AWS services (e.g., Lambda) use roles to obtain temporary credentials when executing tasks.       | Avoids hardcoded keys, supports scaling, enhances security with temporary credentials.            |
| Emergency/Break Glass Access| Temporary elevated permissions for rare emergency situations (e.g., service desk escalation).      | Controlled privilege escalation, auditability, protects normal permissions.                       |
| Corporate Identity Federation | Integrates existing IdPs (e.g., Active Directory) with AWS using roles for single sign-on.           | Circumvents IAM user limit, simplifies user management, leverages existing credentials.          |
| Mobile/Web Application Access| Web Identity Federation allows app users to assume roles via third-party identities (Google, etc.).| Scales to millions of users, no stored credentials in app, utilizes existing user accounts.      |
| Cross-Account Access        | Allows users in one AWS account to assume roles in another account for resource access.            | Simplifies multi-account management, maintains resource ownership, reduces user management overhead.|

---

### Core Concepts

- **IAM Role:** An AWS identity with permission policies and trust policies, assumed temporarily to obtain credentials.
- **Temporary Security Credentials:** Short-lived credentials generated by STS when a role is assumed, enhancing security.
- **Trust Policy:** Defines which entities (users, services, external identities) can assume the role.
- **Permissions Policy:** Defines what actions the role can perform on AWS resources.
- **Identity Federation:** Using external identity providers to authenticate users who then assume IAM roles to access AWS.
- **Break Glass Role:** A role used for emergency access with audit trails and temporary elevated permissions.
- **5,000 IAM User Limit:** A hard AWS limit per account that motivates the use of roles and federation for large user bases.

---

### Important Notes for Exam and Real-World Use

- Always prefer **roles over hardcoded access keys** for service-to-service or application access.
- Use **break glass roles** for emergency or exceptional access instead of granting permanent elevated permissions.
- For large corporate environments or mobile apps, implement **identity federation with roles** to scale beyond IAM user limits.
- **Cross-account roles** simplify permissions management and resource sharing in multi-account AWS architectures.
- Temporary credentials via roles **improve security** by limiting credential exposure and lifespan.

---

This comprehensive overview presents the essential contexts for using IAM roles in AWS, their security and operational benefits, and architectural patterns that leverage them effectively.

### References

- https://docs.aws.amazon.com/wellarchitected/latest/devops-guidance/ag.sad.5-implement-break-glass-procedures.html
- https://github.com/aws-samples/aws-cross-account-break-glass-example#the-break-glass-user