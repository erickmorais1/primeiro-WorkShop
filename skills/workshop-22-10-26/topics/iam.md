# IAM — Identity and Access Management

> **In one sentence:** IAM answers two questions on every call to AWS: **who are you?** (authentication) and **what are you allowed to do?** (authorization).

- **Global** service (not per region) and **free**.
- Everything in AWS goes through it: console, CLI, SDK, and even one service calling another.
- Scope of the workshop: **user, group, role and policy**. Organizations and Permissions Boundaries are only mentioned.

> **Docs:** [What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) · [Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

---

## 1. Analogy to remember

Think of an **office building**:

| Building | AWS |
|---|---|
| Building owner (holds the master key) | **Root user** |
| A named employee badge | **IAM User** |
| Department (IT, Finance) that defines which doors open | **IAM Group** |
| Visitor/temporary badge that any authorized person can "wear" | **IAM Role** |
| Written rules: "this badge opens door X, not door Y" | **Policy** |

---

## 2. The concepts

### 2.1 Root user
- It is the e-mail used to **create** the account. It has **total, unrestricted** power (not even a policy can limit root).
- **Best practices:**
  - Turn on **MFA** for root.
  - **Do not** create access keys for root.
  - **Do not** use it day to day. Use it only for the few tasks that require root (e.g. closing the account, changing the support plan, some billing settings).
  - Never share its password.
- For daily work, create an administrative user/identity and use that instead.

> **Docs:** [Root user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html) · [AWS account root user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html) · [MFA in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html)

### 2.2 IAM User
- Represents **one person or application** with **long-term** credentials:
  - a password (console) and/or an **access key** (ID + secret) for CLI/SDK.
- It is born with **no permissions at all**. Everything is denied until someone attaches a policy.
- Modern recommendation: for people, prefer **IAM Identity Center (SSO)** with temporary credentials instead of many IAM users. Worth **mentioning**, without going deeper.

> **Docs:** [IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html) · [What is IAM Identity Center?](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)

### 2.3 IAM Group
- A **set of users** to apply permissions in bulk ("Devs", "Finance").
- You attach the policy to the **group**, and every member inherits it.
- Points of attention:
  - A group **contains users**, it does **not** contain other groups (no nesting).
  - A group **is not an identity** that "logs in" and cannot be used as a `Principal` in a resource policy.
  - A user can belong to **several** groups.

> **Docs:** [IAM user groups](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups.html)

### 2.4 IAM Role
- An **identity with permissions but no fixed credentials**. Whoever **assumes** it receives **temporary credentials** (via STS).
- Who can assume it? Defined in the **trust policy**.
- What can it do? Defined by the attached **permission policies**.
- Typical use cases:
  - **An AWS service accessing another service**: Lambda → S3, EC2 → DynamoDB.
  - Access **across** AWS accounts.
  - Federated users (SSO / external identity providers).
- **In practice:** a Lambda function accesses S3 without credentials in the code by **having an IAM Role associated with the function**.

> **Two policies on a role, do not mix them up:**
> - **Trust policy:** *who* can assume the role.
> - **Permission policy:** *what* the role can do once assumed.

> **Docs:** [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) · [Temporary security credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html) · [Create a role for an AWS service](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html)

### 2.5 Policy (the "law")
- A **JSON** document that defines permissions. It is the element that says what someone can or cannot do.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-store-bucket/*"
    }
  ]
}
```

Reading in plain English: *"ALLOW the action of **reading objects** in the **bucket my-store-bucket**"*.

| Field | Meaning |
|---|---|
| `Effect` | `Allow` or `Deny` |
| `Action` | Which API calls (`s3:GetObject`, `s3:PutObject`...) |
| `Resource` | On which resources (identified by **ARN**) |
| `Principal` | *Who* (only appears in **resource** policies, such as a bucket policy or a trust policy) |
| `Condition` | Optional conditions (source IP, MFA active, tag...) |

**Two types you need to tell apart:**
- **Identity-based:** attached to a user/group/role ("this person can...").
- **Resource-based:** attached to the resource, e.g. a **bucket policy** ("this bucket accepts access from...").

**Managed vs inline:**
- *AWS managed:* ready-made and maintained by AWS (e.g. `AmazonS3ReadOnlyAccess`).
- *Customer managed:* you create it and reuse it.
- *Inline:* embedded in a single identity, for one-off cases.

> **Docs:** [Policies and permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) · [JSON policy elements reference](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html)

### 2.6 How AWS decides (golden rule)
1. Everything starts **denied** (*implicit deny*).
2. An explicit `Allow` grants access.
3. An explicit `Deny` **always wins**, even if there is an `Allow` somewhere else.

> **Docs:** [Policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)

### 2.7 Principle of least privilege
Grant **only** what is needed, nothing more. Avoid `"Action": "*"` with `"Resource": "*"`, the famous "admin for everything".

> **Docs:** [Apply least-privilege permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege)

---

## 3. Just mention (do not go deeper)

- **AWS Organizations:** manages **several accounts** together (consolidated billing, OU structure). Uses **SCPs** (Service Control Policies) to set maximum limits for the accounts.
- **Permissions Boundary:** a **permissions ceiling** you set on a user/role. The effective permission is the **intersection** of the boundary and the identity's policies.
- **IAM Identity Center:** single sign-on (SSO) across several accounts.

In short: there are layers above this, such as Organizations and boundaries, for governance at scale; here the focus is the basics.

> **Docs:** [What is AWS Organizations?](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html) · [Permissions boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) · [What is IAM Identity Center?](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)

---

## 4. Where IAM shows up in our static site

```
You (user/SSO) --> create the S3 bucket and the CloudFront distribution
CloudFront --(OAC + bucket policy)--> reads objects from the private S3 bucket
```

- The **bucket policy** is a resource policy with `Principal: cloudfront.amazonaws.com` that only accepts requests coming from **your** distribution (details in `cloudfront.md`).
- With automated deploys (CI/CD), the pipeline uses a **role** with permission for `s3:PutObject` and for creating CloudFront invalidations, and **not** fixed access keys.

> **Docs:** [Bucket policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html) · [Restrict access to an S3 origin (OAC)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)

---

## 5. Common beginner pitfalls

- Using root "just to test" and forgetting to switch.
- Committing an access key to Git (AWS detects it and warns you, but the damage is quick).
- Giving `AdministratorAccess` to everything.
- Confusing **Role** with **User**: a role has no password and no permanent key.
- Confusing a **trust policy** with a **permission policy**.

---

## 6. Hands-on practice (study account, all free)

1. Turn on **MFA** for root and sign out of root.
2. Create a **group** `s3-readers` with the managed policy `AmazonS3ReadOnlyAccess`.
3. Create a **user** and add it to the group. Sign in and try to **create** a bucket (it should fail: read-only).
4. Create a **role** for the **Lambda** service with read permission on S3. Open the *trust policy* and identify the `Principal`.
5. Open the **IAM Policy Simulator** and test actions against the policy.

> **Docs:** [Testing IAM policies with the policy simulator](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html)

---

## 7. Self-check questions

- What is the difference between a user and a role?
- Why can't a group be a `Principal`?
- If one policy gives `Allow` and another gives `Deny` for the same action, which one wins?
- What is in a trust policy?
- Why not create access keys for root?

---

## References

- IAM product page: https://aws.amazon.com/iam
- What is IAM?: https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- Security best practices in IAM: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- Root user best practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html
- MFA in IAM: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html
- IAM users: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users.html
- IAM user groups: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_groups.html
- IAM roles: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html
- Temporary security credentials: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html
- Policies and permissions: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
- JSON policy elements reference: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html
- Policy evaluation logic: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
- Policy simulator: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_testing-policies.html
- IAM Identity Center: https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html
- AWS Organizations: https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html
- Permissions boundaries: https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html
