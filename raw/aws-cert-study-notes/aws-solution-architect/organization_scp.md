[00:00:00] **Introduction to Service Control Policies (SCPs) in AWS Organizations**  
- Service Control Policies (SCPs) are a feature within AWS Organizations used to **restrict and manage permissions across multiple AWS accounts**.  
- SCPs are essential for designing and implementing **larger AWS environments**, especially when managing multiple accounts under a single organization.  
- This lesson introduces SCPs, their purpose, and how they fit into the AWS Organizations structure.

[00:00:25] **Current AWS Organization Setup: Animals for Life**  
- An AWS Organization named *Animals for Life* has been created.  
- The organization contains:  
  - A **management account** (referred to as the general account).  
  - Two member accounts: **Production (prod)** and **Development (dev)**.  
- All accounts currently reside directly under the **root container** of the organization, meaning they are **not yet placed inside Organizational Units (OUs)**.

[00:00:57] **Future Structure: Organizational Units Addition**
- Planned introduction of two OUs:  
  - One for **Production**.  
  - One for **Development**.  
- The member accounts (prod and dev) will be moved into their respective OUs in the next demo lesson.

[00:01:21] **Overview of Service Control Policies (SCPs)**  
- SCPs are **JSON policy documents** that:  
  - Can be attached to the **root container** (affecting the entire organization).  
  - Can be attached to one or more **Organizational Units** (affecting all accounts within those OUs, including nested OUs).  
  - Can be attached to individual **AWS accounts** directly.  
- SCPs **inherit down the organizational hierarchy**, meaning policies applied at higher levels propagate to all child OUs and accounts.

[00:01:46] **SCP Inheritance and Scope**  
- **Attachment scopes:**  
  - Root container → affects all accounts in the organization.  
  - OUs → affects all accounts within that OU and any nested OUs.  
  - Individual accounts → affects only that account.  
- SCPs enable centralized control over account permissions at various levels of the organization.

[00:02:33] **Special Status of the Management Account**  
- The **management account** is **exempt from SCP restrictions**:  
  - SCPs attached to the root container, OUs, or directly to the management account **do not restrict the management account’s permissions**.  
- This behavior can be both beneficial (ensuring management account always has full control) and a limitation (cannot be locked down via SCPs).  
- **Best practice:** Avoid using the management account for routine AWS resource deployments in production environments since it cannot be restricted by SCPs.

[00:03:41] **SCPs as Account Permission Boundaries**  
- SCPs act as **permission boundaries** on the AWS account level, limiting what the account and identities inside it can do.  
- They **restrict the effective permissions** of all identities within an account, including the account root user, but:  
  - The **account root user itself cannot be directly restricted** by identity policies.  
  - SCPs restrict the *account’s allowed actions* and thus indirectly limit what the root user can perform.  
- **Key distinction:** SCPs do *not* grant permissions; they only define the *maximum permissions* that can be granted.

[00:04:56] **Examples of SCP Usage**  
- Restrict usage of AWS resources to specific conditions, e.g.:  
  - Restrict operations to a certain AWS region (e.g., **US East 1**).  
  - Limit EC2 instance sizes that can be launched.  
- SCPs are valuable in **large, complex AWS environments** to enforce organizational security and operational policies.

[00:05:23] **SCPs Do Not Grant Permissions**  
- SCPs **define limits** on permissions but do **not grant any permissions themselves**.  
- Actual permissions must be granted through **identity-based policies** (IAM policies attached to users, groups, or roles).  
- SCPs **restrict the maximum permissions** that can be granted by these identity policies.

[00:06:01] **SCP Usage Models: Allow List vs. Deny List**  
- Two primary approaches to SCPs:  
  1. **Deny List (default in AWS):**  
     - Start with a default **Full AWS Access SCP** that allows all actions.  
     - Add explicit **deny policies** to block specific services or actions.  
  2. **Allow List:**  
     - Remove the default Full AWS Access SCP.  
     - Define explicit policies that **allow only certain services**.  
- The default SCP is a **deny list model**, where everything is allowed unless explicitly denied.

[00:06:27] **Default SCP: Full AWS Access**  
- When SCPs are enabled, AWS applies a **default SCP called Full AWS Access** to the entire organization and OUs.  
- This policy effectively imposes **no restrictions**, allowing all services.  
- This default ensures SCPs operate on a **deny list basis**, where you add explicit denies to restrict resources.

[00:07:14] **Policy Evaluation Logic in SCPs**  
- SCP evaluation follows these rules:  
  - **Explicit deny always overrides allow**.  
  - Explicit allows in SCPs permit permissions to identities if no deny exists.  
  - In the absence of any explicit allow or deny, an **implicit deny** applies.  
- SCPs control which services can be granted permissions by identity policies within the account.

[00:08:01] **Advantages of Deny List Architecture**  
- Since the default policy allows all services, AWS automatically includes new services as they are released.  
- This reduces administrative overhead compared to allow lists, because:  
  - You only need to add denies for unwanted services.  
  - Access to new AWS services granted by identity policies is automatically permitted unless explicitly denied.

[00:08:40] **Allow List Architecture: More Restrictive but Higher Admin Overhead**  
- To implement allow lists:  
  - Remove the default Full AWS Access SCP.  
  - Add policies explicitly allowing only certain services (e.g., S3, EC2).  
- This model:  
  - Is **more secure** since only explicitly allowed services are accessible.  
  - Carries **higher administrative overhead** because every required service must be explicitly added.  
  - Increases risk of unintentionally blocking access to needed services.

[00:10:08] **Recommendation**  
- The instructor generally recommends the **deny list model** due to its **lower administrative overhead** and ease of maintenance.

[00:10:28] **Visual Explanation of SCPs’ Effect on Permissions**  
- Permissions effective for identities are the **intersection** of:  
  1. **Identity policies** granting permissions (orange circle).  
  2. **SCPs allowing services** (red circle).  
- Only permissions **allowed by both** identity policies and SCPs are effective.  
- Permissions allowed by identity policies but restricted by SCPs are denied.  
- Permissions allowed by SCPs but not granted by identity policies have no effect.

[00:12:00] **Exam-Level Understanding of SCPs**  
- For certification, it is critical to grasp that **effective permissions = intersection of identity policies and SCPs**.  
- This concept highlights that SCPs act as a **permission boundary** on the maximum allowed actions within an account.

[00:12:22] **Next Steps: Practical Demo**  
- The upcoming lesson will provide a **hands-on demo** to implement SCPs in the Animals for Life AWS Organization.  
- The demo will build on the theory presented and illustrate SCP application and effects in practice.

---

### **Summary Table: SCP Attachment Scopes and Effects**

| Attachment Point       | Affected Entities                 | Effect on Permissions                               |
|-----------------------|---------------------------------|----------------------------------------------------|
| Root Container        | All accounts in the organization | SCP applies to all accounts (except management)    |
| Organizational Unit   | All accounts in OU and nested OUs| SCP applies to all member accounts under OU        |
| Individual Account    | Specific AWS account             | SCP applies only to that account                     |
| Management Account    | Special case                    | SCPs do **not** restrict management account         |

---

### **Key Concepts and Definitions**

| Term                     | Definition                                                                                       |
|--------------------------|------------------------------------------------------------------------------------------------|
| Service Control Policy (SCP) | A JSON policy attached at organization, OU, or account level that defines permission boundaries but does not grant permissions. |
| Identity Policy          | IAM policies attached to users, groups, or roles that grant permissions within an AWS account.  |
| Management Account       | The AWS account that manages the organization; cannot be restricted by SCPs.                   |
| Deny List SCP Model      | Default SCP mode where all permissions are allowed unless explicitly denied.                    |
| Allow List SCP Model     | SCP model where no permissions are granted unless explicitly allowed; higher security and overhead. |

![[organization-scp-infographic.png]]

---

### **Important Security Insights**

- **Management account cannot be restricted by SCPs**; avoid using it for production resources.  
- **SCPs act as permission boundaries**: they limit what permissions can be granted but do not grant permissions themselves.  
- Explicit **deny in SCPs always overrides allow**.  
- Effective permissions require both SCP allowance and identity policy allowance.  
- Default SCP is Full AWS Access, enabling a deny list architecture that reduces admin overhead.  

---

### **Conclusion**  

Service Control Policies are a fundamental tool in AWS Organizations for centrally managing and constraining permissions across multiple accounts. Understanding SCPs’ hierarchical application, the special role of the management account, and the distinction between permission boundaries and permission grants is crucial for designing secure, scalable AWS environments. The deny list approach is generally preferred for its balance of security and manageability, while the allow list approach offers stricter control at the cost of increased complexity. Practical experience with SCPs will solidify these concepts, as demonstrated in the forthcoming demo.