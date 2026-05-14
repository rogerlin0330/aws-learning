## Summary Infographic

![[iam-groups-infographic.png]]

## Summary Table: IAM Groups Key Facts

| Feature                             | Description/Limitations                             |
|-----------------------------------|---------------------------------------------------|
| Purpose                           | Organize IAM users for easier management          |
| Credentials                      | None; groups cannot be logged into                 |
| Membership                       | Users only; no nesting of groups                   |
| User Group Membership Limit      | Max 10 groups per user (hard limit)                |
| Account User Limit               | Max 5,000 IAM users per account (hard limit)       |
| Group Membership Size            | No limit on number of users per group               |
| Group Count per Account          | Default 300 groups; increase via support ticket    |
| Policy Attachment                | Inline and managed policies can be attached         |
| Policy Inheritance               | Users inherit policies from all their groups + own  |
| Resource Policy Reference        | Cannot reference groups; only users and roles       |
| Built-in "All-users" Group       | None; must be created manually if needed            |

---

## Key Insights

- **IAM groups are containers, not identities.**  
- **Groups simplify permission management by allowing policy attachment at the group level.**  
- **Users’ effective permissions are cumulative from their group memberships and individual policies.**  
- **Groups cannot be used in resource-based policies (e.g., S3 bucket policies).**  
- **IAM user limits and group membership limits are hard limits and cannot be changed.**  
- **There is no nesting of groups, which simplifies but limits organizational structures.**  
- **Misunderstanding group capabilities (e.g., login or resource policy referencing) is a common exam trap.**

---

This professional summary is fully grounded in the provided lesson content on IAM groups, capturing all critical concepts, limitations, and use cases.