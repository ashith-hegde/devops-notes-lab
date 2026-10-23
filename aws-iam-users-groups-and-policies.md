# AWS IAM Users, Groups, and Policies

## IAM Overview

**AWS Identity and Access Management (IAM)** is a global AWS service used to manage identities and control access to AWS resources.

IAM determines who can access AWS services and which actions they can perform.

## Root User and IAM Users

- **Root user:** Created when an AWS account is created and has extensive privileges. Secure it carefully and avoid routine use or sharing its credentials.
- **IAM user:** An identity that can represent a person or workload needing AWS access.
- Use root-level access only when required. For workforce access, consider federation with IAM Identity Center; for AWS workloads, IAM roles are often preferable to long-lived IAM users.

## IAM Groups

Groups simplify permission management for users with similar responsibilities.

- Groups can contain users, but cannot contain other groups.
- A user can belong to multiple groups.
- A user does not have to belong to a group.
- Users receive permissions through policies attached to their groups, as well as applicable policies attached directly to them.

Example:
- Alice, Bob, and Charles belong to the Developers group.
- David and Edward belong to the Operations group.
- Charles and David also belong to an Audit group.
- Fred does not belong to a group.

A policy attached to a group applies to its members. A user in multiple groups can receive permissions through each group. For example, Charles can inherit permissions from both Developers and Audit, while David can inherit permissions from Operations and Audit.

### How Effective Permissions Add Up

A user's effective permissions can come from multiple places, including policies attached directly to the user and policies inherited through every group they belong to. For example, a user could receive `AdministratorAccess` from an Admin group, a service-specific policy from a Developers group, and `IAMReadOnlyAccess` directly. These policy attachments all contribute to the user's permissions. Removing a user from a group removes the permissions provided by that group's policies, though permissions from other attachments may remain.

**Practical lesson:** Test permissions after changing group membership or policy attachments. A user who loses administrative access may receive `AccessDenied` for actions such as `iam:ListUsers`. Read-only permissions can allow viewing IAM information without allowing changes such as creating a user group.

### Inline Policies

An **inline policy** is embedded directly in an individual IAM user, group, or role rather than managed as a separate standalone policy. It is useful to recognize the term for exams, but reusable managed policies are generally easier to maintain consistently across identities.

## IAM Policies and Permissions

An **IAM policy** is a JSON document defining which actions are allowed or denied and which resources those actions apply to. Policies can be attached to users or groups; AWS also supports resource-based policies attached to resources.

Example: allow EC2 describe actions.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    }
  ]
}
```

In this example, `ec2:Describe*` matches EC2 API actions beginning with `Describe`. `"Resource": "*"` applies to all resources supported by the specified actions; it does not allow every AWS action.

### Policy Structure

| Field | Meaning |
|---|---|
| `Version` | Policy language version; commonly `2012-10-17` |
| `Id` | Optional identifier for the policy document |
| `Statement` | One or more permission statements |
| `Sid` | Optional identifier for a statement |
| `Effect` | `Allow` or `Deny` |
| `Principal` | In a resource-based policy, identifies the account, user, role, or other principal the policy applies to |
| `Action` | API actions that the statement allows or denies |
| `Resource` | Resources to which the actions apply |
| `Condition` | Optional conditions that determine when the statement applies |

**Important distinction:** A typical identity-based policy attached to a user or group does not specify `Principal`; the identity to which the policy is attached provides that context. Resource-based policies use `Principal` to identify who can access the resource.

## Principle of Least Privilege

Grant only the permissions required to perform a task. Avoid broad permissions when narrower permissions will work. Excessive permissions increase the risk of accidental changes, security incidents, and unexpected costs.

Examples:
- A monitoring user may need permission to read metrics and describe resources without being allowed to delete or modify infrastructure.
- Assign `AdministratorAccess` only when broad administrative permissions are genuinely required.

## Creating Users and Assigning Permissions

In the AWS Management Console, users can be created through **IAM → Users**.

- Permissions can be attached directly to a user or granted through group membership.
- Assigning permissions through groups usually simplifies ongoing administration.
- During user creation, you can configure console access and credentials.
- For workforce access, consider IAM Identity Center and federation where appropriate instead of creating long-lived IAM users for every employee.

### Tags

**Tags** are optional key-value metadata attached to AWS resources. For example, `Department: Engineering`. Tags can help organize resources, report on costs, and support governance.

## Signing In to the AWS Console

- IAM users can sign in using the AWS account ID or account alias, IAM username, and password.
- An **account alias** provides a more memorable sign-in URL and must be unique across AWS account aliases.
- AWS Console multi-session support can allow multiple accounts or identities to be open in separate sessions within the same browser.
- A private browser window is another option for keeping separate sessions open.

Each session has its own account and identity context. Resources created in one account are not automatically visible in another account. Before making changes, especially in production, verify the active account and identity.

Keep root credentials secure and ensure root-account access is retained for tasks that specifically require it.

## Password Policy and Multi-Factor Authentication (MFA)

### IAM Password Policy

A password policy strengthens password-based sign-in for IAM users. Configure it in **IAM → Account settings → Password policy**.

Available settings include:
- Minimum password length.
- Required character types: uppercase, lowercase, numbers, and non-alphanumeric characters.
- Whether IAM users can change their own passwords.
- Password expiration after a configured period, if required by your policy.
- Whether users can reuse previous passwords.
- Whether expired passwords require an administrator reset (where configured).

A strong password policy helps defend against password guessing and brute-force attempts, but it does not replace MFA.

### What MFA Means

**Multi-factor authentication (MFA)** adds a second proof of identity to a password. In the course's simplified model, this is:

- **Something you know:** your password.
- **Something you have:** an authenticator or security device.

If a password is stolen, MFA makes account takeover harder because the attacker also needs the second factor. Enable MFA for the **root user** and IAM users, especially privileged users, wherever applicable.

### MFA Device Types

| Type | Examples / notes |
|---|---|
| Virtual authenticator app | Google Authenticator or Authy; one device can hold tokens for multiple accounts/identities. |
| U2F/FIDO security key | A physical key such as YubiKey; one key may support multiple root and IAM identities. |
| Hardware TOTP token / key fob | A dedicated physical token; the slides mention third-party providers such as Gemalto. |
| AWS GovCloud (US) hardware token | The slides mention a GovCloud-specific key fob from SurePassID. |

Device types and supported providers can change; use the options currently shown in the AWS console for your account.

### Enabling MFA: Hands-on Flow

1. For the root user, sign in as root and open the account menu → **Security credentials**. For IAM users, use the relevant IAM security-credentials workflow and permissions.
2. Choose **Assign MFA device** and give the device a recognizable name.
3. Select the supported MFA type, such as an authenticator app, security key, or hardware TOTP token.
4. For an authenticator app, display the QR code and scan it with the app.
5. Enter the consecutive MFA codes requested by AWS to verify that the device is synchronized correctly, then complete setup.
6. Sign out and sign in again to confirm that the password plus MFA code/key is required.

**Avoid lockout:** Before enabling MFA on a critical account, make sure you can retain access to the device and understand the account's recovery process. Securely maintain any supported backup/recovery options. Losing the only usable MFA method can make account recovery difficult.

### Security Takeaways

- Protect the root user with MFA and avoid using root for routine work.
- Enforce MFA for IAM users, especially administrators, where applicable.
- A password policy and MFA are complementary controls: use both.
- Console sign-in uses the password and MFA; programmatic access through CLI/SDK uses credentials such as access keys or role-based credentials, and should be secured separately. Do not assume console MFA automatically protects exposed access keys.

## Quick Revision

| Concept | Remember |
|---|---|
| IAM | Global identity and access management service |
| Root user | Secure carefully; avoid routine use |
| IAM user | Identity with AWS access |
| IAM group | Organizes users with similar permissions |
| Managed policy | Separate policy that can be reused across identities; inspect its JSON to understand its permissions |
| Inline policy | Policy embedded directly in one identity |
| IAM policy | JSON document defining permissions |
| `Effect` | Allows or denies actions |
| `Action` | API actions covered; wildcards such as `iam:List*` can match multiple actions |
| `Resource` | Resources those actions apply to |
| `Principal` | Who a resource-based policy applies to |
| Effective permissions | Can come from directly attached policies and policies inherited through multiple groups |
| Wildcard (`*`) | Matches multiple actions or resources; scope carefully |
| Least privilege | Grant only the permissions required |
| Password policy | Controls IAM user password strength, changes, expiration, and reuse |
| MFA | Adds a second factor to password-based sign-in |
| Root MFA | High-priority protection for the AWS account root user |
| Virtual MFA | Authenticator app that generates time-based codes |
| Security key / hardware token | Physical MFA option; supported choices may vary |
| Account alias | Human-readable alternative for the account sign-in URL |

**Interview takeaway:** Explain how IAM identities, groups, policies, and least privilege work together to control access to AWS infrastructure. Know the distinction between identity-based and resource-based policies.

## References

- [AWS Identity and Access Management](https://aws.amazon.com/iam/)
- [IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/)
- [IAM JSON policy elements](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html)
- [Managing passwords for IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_passwords.html)
- [Multi-factor authentication (MFA) in AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html)

