# Lab 1 — Cloud Account Security, Identity & Access Management

**Course:** IKB42603 Cloud Computing Security Essentials  
**Lab period:** Weeks 1–2  
**Platforms:** LocalStack IAM (AWS-compatible) and Kubernetes (kind)

## Aim

This lab applied identity governance, least privilege, credential hygiene, and Kubernetes Role-Based Access Control (RBAC). The work replaced routine root-level activity with a named administrator, created a read-only analyst identity, demonstrated access-key rotation, and enforced namespace-scoped permissions in Kubernetes.

## Evidence index

All screenshots used for this report are stored in the `Evidence` and `Evidence part b` folders.

| Evidence File | Purpose |
|---|---|
| [Evidence/2.1createGroup.png](<Evidence/2.1createGroup.png>) | Creation of the `Admins` IAM group. |
| [Evidence/2.2createUserAdmin.png](<Evidence/2.2createUserAdmin.png>) | Creation of the `CloudAdmin_ASRA` administrator user. |
| [Evidence/2.3.verifyMembership.png](<Evidence/2.3.verifyMembership.png>) | Verification that `CloudAdmin_ASRA` belongs to `Admins`. |
| [Evidence/3.1createUserRead.png](<Evidence/3.1createUserRead.png>) | Creation of the `Analyst_AMIR` analyst user. |
| [Evidence/3.2verifyPolicyUserRead.png](<Evidence/3.2verifyPolicyUserRead.png>) | Verification of `AmazonS3ReadOnlyAccess` for `Analyst_AMIR`. |
| [Evidence/4.1createAccessKey.png](<Evidence/4.1createAccessKey.png>) | Access-key creation evidence for `Analyst_AMIR`. **Redact the secret access key before submitting.** |
| [Evidence/4.2listAccessKey.png](<Evidence/4.2listAccessKey.png>) | Access-key metadata listing for `Analyst_AMIR`. |
| [Evidence/4.3rotateDeactivateOldKey.png](<Evidence/4.3rotateDeactivateOldKey.png>) | Access-key deactivation command. |
| [Evidence part b/setupKubernetesCluster.png](<Evidence part b/setupKubernetesCluster.png>) | Kubernetes cluster and node verification. |
| [Evidence part b/5.0listNamespace.png](<Evidence part b/5.0listNamespace.png>) | `dev` and `prod` namespace verification. |
| [Evidence part b/6.0roleBind.png](<Evidence part b/6.0roleBind.png>) | Service account, Role, and RoleBinding creation. |
| [Evidence part b/7.0test.png](<Evidence part b/7.0test.png>) | RBAC authorization tests. |
| [Evidence part b/Verification.png](<Evidence part b/Verification.png>) | RoleBinding YAML verification. |

## Task 1 — Map the cloud identity landscape

| Concept | AWS term | Purpose |
|---|---|---|
| All-powerful owner | Root user | The account’s highest-privileged owner identity. It controls account-level settings and must not be used for routine work. |
| Human/application identity | IAM User | A persistent identity for a person or application, which receives credentials and permissions. |
| Permission bundle | IAM Policy | A JSON permission document that states which actions and resources are allowed or denied. |
| Collection of users | IAM Group | A manageable collection of IAM users. Policies attached to the group apply to all members. |
| Temporary identity | IAM Role | An assumable identity with temporary credentials, used by people, services, or workloads without sharing long-lived credentials. |

## Session A — LocalStack IAM

### Environment setup

The lab guide specifies starting LocalStack, configuring dummy AWS CLI credentials, and verifying the current LocalStack identity with:

```powershell
aws --endpoint-url=http://localhost:4566 sts get-caller-identity
```

**Status:** The supplied folder does not contain the required `sts get-caller-identity` output. Capture and add this screenshot before final submission.

### Task 2 — Create a least-privilege administrator

1. Created the `Admins` IAM group.
2. Attached `AdministratorAccess` to the group, so administrative permissions are managed centrally rather than per user.
3. Created personal administrator user `CloudAdmin_ASRA`.
4. Added the user to `Admins`.
5. Verified membership with `aws $EP iam get-group --group-name Admins`.

The verification output shows `CloudAdmin_ASRA` as the member of `Admins`. This establishes a dedicated daily-use administrator and supports avoiding routine root-user access.

### Task 3 — Enforce least privilege with a scoped policy

1. Created the analyst user `Analyst_AMIR`.
2. Attached `AmazonS3ReadOnlyAccess` to that user.
3. Verified the attachment with:

```powershell
aws $EP iam list-attached-user-policies --user-name Analyst_AMIR
```

The result lists only `AmazonS3ReadOnlyAccess`. Therefore, the analyst can read S3 data but is not granted write, deletion, IAM-administration, or broader administrative permissions.

**Blast-radius explanation:** If the analyst credentials were compromised, an attacker could at most access data allowed by the read-only S3 policy. They could not use that identity to alter or delete resources, create new identities, attach policies, or administer the account. Restricting permissions this way reduces the scope and impact of a compromise.

### Task 4 — Credential hygiene and access keys

1. Created an access key for `Analyst_AMIR`.
2. Listed the user’s access keys and confirmed the key was initially `Active`.
3. Rotated the credential by changing the old key’s status to `Inactive`:

```powershell
aws $EP iam update-access-key --user-name Analyst_AMIR `
  --access-key-id <ACCESS_KEY_ID> --status Inactive
```

This demonstrates the key-lifecycle control required by the lab. In a production AWS environment, access-key secrets must never be stored in source control or reports; prefer short-lived role credentials and deactivate or remove keys promptly when rotating them.

## Session B — Kubernetes RBAC

### Setup — Create the local Kubernetes cluster

1. Created the kind cluster `ccse-lab1`.
2. Verified the control plane and CoreDNS with `kubectl cluster-info --context kind-ccse-lab1`.
3. Confirmed node `ccse-lab1-control-plane` was `Ready` using `kubectl get nodes`.

### Task 5 — Separate environments with namespaces

1. Created the `dev` namespace.
2. Created the `prod` namespace.
3. Listed namespaces to verify that both were `Active`.

Namespaces provide a scope boundary within one Kubernetes cluster. They allow permissions and workloads for development and production to be separated.

### Task 6 — Define a role and bind it

1. Created service account `dev-user` in the `dev` namespace.
2. Created namespaced Role `pod-reader` in `dev` with only `get`, `list`, and `watch` verbs on `pods`.
3. Created RoleBinding `dev-user-binding` in `dev`, which binds `pod-reader` to `system:serviceaccount:dev:dev-user`.

The RoleBinding YAML confirms the binding references the `pod-reader` Role and the `dev-user` service account in namespace `dev`.

### Task 7 — Test that access control works

The tested identity was:

```powershell
$SA = 'system:serviceaccount:dev:dev-user'
```

| Verification command | Result | Interpretation |
|---|---|---|
| `kubectl auth can-i list pods -n dev --as=$SA` | `yes` | The Role allows listing pods in `dev`. |
| `kubectl auth can-i delete pods -n dev --as=$SA` | `no` | Delete was not granted by the Role. |
| `kubectl auth can-i list pods -n prod --as=$SA` | `no` | The namespaced Role does not apply to `prod`. |

Authentication identifies the request as the `dev-user` service account. Authorization then evaluates the requested verb, resource, and namespace against Kubernetes RBAC. The service account is authenticated in all three tests, but authorization allows only the first request and blocks deletion and access to `prod`.

## Required RoleBinding verification output

The lab requires the output of:

```powershell
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

The supplied output verifies:

- `kind: RoleBinding`
- namespace: `dev`
- `roleRef.kind: Role`
- `roleRef.name: pod-reader`
- subject: service account `dev-user` in namespace `dev`

## Short-answer questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?

Group-based policy assignment centralizes access management. Administrators change the group policy or membership once and the correct permissions apply consistently to every member. This is easier to audit, avoids duplicated policy assignments, and reduces configuration drift.

### Q2. What is the difference between an IAM User and an IAM Role?

An IAM User is a persistent identity for a specific person or application and can have long-term credentials. An IAM Role is an assumable identity intended for temporary use; it provides temporary credentials and can be used by trusted users, services, or workloads.

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.

`Analyst_AMIR` has `AmazonS3ReadOnlyAccess`, not administrative permissions. The account can perform only the read operations necessary for analysis. A compromise is therefore contained: the attacker cannot modify data or manage cloud identities and resources using that account.

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?

A Role defines a set of allowed actions on resources within a namespace. A RoleBinding assigns that Role to one or more subjects, such as users, groups, or service accounts. The Role states *what* is allowed; the RoleBinding states *who* receives it.

### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?

`dev-user` was bound to a namespaced Role in `dev`; it has no binding or permissions in `prod`. Kubernetes therefore denies the request to list pods in `prod`. This demonstrates least privilege and environment segregation through namespace-scoped RBAC.

## Security best-practices checklist

- [x] A dedicated administrator identity (`CloudAdmin_ASRA`) was created for daily administrative work.
- [x] Administrative permissions are assigned through the `Admins` group.
- [x] A least-privilege read-only analyst identity was created and verified.
- [x] An analyst access key was listed and deactivated as a rotation demonstration.
- [x] Kubernetes RBAC denied an unauthorised delete request and cross-namespace request.
- [ ] Capture the outstanding `sts get-caller-identity` screenshot before submission.

## Conclusion

The lab demonstrated identity separation and least privilege in two environments. LocalStack IAM was used to create group-managed administrative access, a scoped analyst account, and an access-key rotation workflow. Kubernetes RBAC then enforced the same principle operationally: `dev-user` could list pods only in `dev`, while destructive actions and `prod` access were denied.
