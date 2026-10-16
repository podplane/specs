# Podplane RBAC

> **STATUS**: Implemented

## Goal

Give teams sensible access out of the box while keeping the platform out of reach.

- Admins run the cluster. Everyone else works in **tenant namespaces** (namespaces without a `kube-` or `platform-` prefix) and cannot change, or escalate through, the platform.
- Small teams get access to every tenant namespace with no setup.
- Larger organisations can **restrict** a namespace and grant access explicitly, or simply not issue the Podplane standard groups claims and grant access explicitly to custom groups claims.

Non-goal: hard isolation between teams by default. The defaults protect the
platform from users, not teams from each other; use restricted namespaces for that.

## Namespaces

| Kind | Which | Default access |
| --- | --- | --- |
| Platform | names starting `kube-` or `platform-` | admins only |
| Tenant | every other namespace | all Podplane groups, by role |
| Restricted | tenant namespace labelled `rbac.podplane.dev/restricted: "true"` | admins, plus explicit RoleBindings |

Only admins can create namespaces or change their labels, so only admins decide
whether a namespace is restricted.

## Roles

| Group | Tenant namespaces | Platform namespaces |
| --- | --- | --- |
| `podplane:admins` | `cluster-admin` | `cluster-admin` |
| `podplane:managers` | `admin` | none |
| `podplane:operators` | `edit`, with pod shell | none |
| `podplane:editors` | `edit`, without pod shell | none |
| `podplane:viewers` | `view` | none |

Managers can grant access within their namespaces with RoleBindings, as
Kubernetes `admin` allows.

Platform namespaces are not visible to viewers. Admins can grant access to a
specific platform namespace with an explicit RoleBinding.

## How it works

**podplane-operator** watches Namespaces. In each tenant namespace that is not
restricted, it keeps one RoleBinding per group (`podplane-managers`,
`podplane-operators`, `podplane-editors`, `podplane-viewers`), labelled
`rbac.podplane.dev/managed: "true"`. It recreates deleted bindings, removes its
bindings when a namespace becomes restricted, and never touches bindings without
its label. A new namespace has no access for the few seconds before the operator
reconciles it, or until the operator is running.

The operator may create and delete RoleBindings and holds `bind` only on the
`admin`, `edit`, and `view` ClusterRoles. It does not hold those roles itself.

**platform-rbac** provides:

- the `podplane:admins` ClusterRoleBinding, the only cluster-wide grant;
- the policy denying `pods/exec` and `pods/attach` to editors; and
- a policy denying creation of `kube-*` and `platform-*` namespaces except by
  `podplane:admins`, `system:masters`, and Flux's helm-controller, which creates
  platform namespaces.

**Flux**: helm-controller applies every HelmRelease as `cluster-admin`, so tenant
groups get no Flux access (`rbac.createAggregation: false`). Self-service Flux for
tenants needs helm-controller to impersonate a tenant service account first.

## Why RoleBindings, not ClusterRoleBindings plus a deny policy

Admission policies never see reads. Under ClusterRoleBindings, tenant groups could
read Secrets in platform and restricted namespaces regardless of any policy.

## Requirements and limits

- Group access requires `podplane-operator`. Bare and minimal clusters are
  admin-only until it is installed with `podplane install podplane-operator`.
- Pod Security Admission, not this spec, controls what tenant Pods may do.

## Implementation

1. `podplane/operator`: namespace group bindings controller and `rbac.namespace_group_bindings` config.
2. `podplane/components`: operator RBAC and config, platform-rbac bindings and
   namespace prefix policy, Flux aggregation off.
3. `podplane/podplane`: RBAC docs.
4. `podplane/seedgen`, `podplane/seeds`: regenerate seeds.
