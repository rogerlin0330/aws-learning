# IAM

## IAM Identity Policies

- Every IAM policy is compose of a IAM policy document (JSON). The structure of a policy document is:

  ```json
  {
    "version": "2012-10-17",
    "Statement": [
      {
        "Sid": "FullAccess",
        "Effect": "Allow",
        "Action": ["s3:*"],
        "Resource": ["*"]
      },
      {
        "Sid": "DenyAccessToCatgrifsBucket",
        "Effect": "Deny",
        "Action": ["s3:*"],
        "Resource": ["arn:aws:s3:::catgrifs", "arn:aws:s3:::catgrifs/*"]
      }
    ]
  }  
  ```

  Each statement has the following fields:
  - `Sid`: Optional field to identify a statement, recommended to include it for every statement
  - `Effect`: "Allow" or "Deny"
  - `Action`: The action to allow or deny, can be a single action (string) or a list of actions (array of strings)
  - `Resource`: The resource to allow or deny the action on, can be a single resource (string) or a list of resources (array of strings)
- Overlapping rules priority
  - Explicit deny
  - Explicit allow
  - Implicit deny (default)
  ![[iam-policy-rule-priority.png]]
- When multiple policies are applied (e.g. user, group, or even resource-based policies), the same priority rules apply. AWS evaluates all policies together to determine the final permission.
  ![[iam-multiple-policies.png]]
	- Managed policies vs inline policies
  - Managed policies benefits
    - Reusable across multiple identities
    - Can be attached to multiple identities
    - Can be updated in one place
    - Low management overhead
  - There are two types of managed policies:
    - AWS managed policies
    - Customer managed policies

## IAM Users and ARNs

- IAM Users are an identity used for anything requiring **long-term** AWS access, e.g., Humans, Applications or Service Accounts.
- ARN uniquely identifies resources within any AWS account
