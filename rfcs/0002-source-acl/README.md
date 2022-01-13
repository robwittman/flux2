# RFC-0002 Access control for source references

## Summary

Cross-namespace references to Flux sources should be subject to
Access Control Lists (ACLs) as defined by the owner of a particular source.

Similar to [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/),
Flux ACLs define policies for restricting the access to the source artifact server based on the
caller's namespace.

## Motivation

This proposal tries to solve the "cross-namespace references side-step namespace isolation" issue (explained in
[RFC-0001](https://github.com/fluxcd/flux2/tree/main/rfcs/0001-authorization#cross-namespace-references-side-step-namespace-isolation)).

As of [version 0.25](https://github.com/fluxcd/flux2/releases/tag/v0.25.0) (Ian 2022),
Flux allows for `Kustomizations` and `HelmReleases` to reference sources in different namespaces.
This poses a serious security risk for multi-tenant environments as Flux does not prevent tenants from accessing
known sources outside of their namespaces.

Flux does not allow for an `ImageUpdateAutomation` to reference a `GitRepository` in a different namespace.
This means users have to copy the Git auth secret and the `GitRepository` object in all namespaces
where `ImageUpdateAutomations` are used. This poses a serious security risk for multi-tenant environments,
as tenants could use the Git secret to push changes to repositories by impersonating Flux.

Flux allows for `ImagePolicies` to reference `ImageRepositories` in a different namespace only 
if the ACL present on the `ImageRepository` grants access to the namespace where the `ImagePolicy` is.
This has been implemented in
[fluxcd/image-reflector-controller#162](https://github.com/fluxcd/image-reflector-controller/pull/162).

Flux should be consistent when dealing with cross-namespace references by extending the
Image Policy/Repository approach to all the other APIs.

### Goals

- Allow source owners to choose which sources are shared and with which namespaces.
- Allow cluster admins to enforce source ACLs.

### Non-Goals

- Enforce source ACLs by default. 

## Proposal

Extend the current Image Policy/Repository ACL implementation to all the others Flux resources
as described in [flux2#1704](https://github.com/fluxcd/flux2/issues/1704).

When a Flux resource (`Kustomization`, `HelmRelease` or `ImageUpdateAutomation`)
refers to a source (`GitRepository`, `HelmRepository` or `Bucket`) in a different namespace,
access is granted based on the source ACL.

The ACL check is performed only if `--enable-source-acl` flag is set to `true` for the following controllers:

- kustomize-controller
- helm-controller
- image-automation-controller

### User Stories

#### Story 1

> As a cluster admin, I want to share Helm Repositories approved by the platform team with all tenants.

If the owner of a Flux `HelmRepository` wants to grant access to the repository for all namespaces in a cluster,
an empty `matchLabels` can be used:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  url: https://charts.bitnami.com/bitnami
  accessFrom:
    namespaceSelectors:
      - matchLabels: {}
```

If the `accessFrom` field is not present and `--enable-source-acl` is set to `true`,
means that a source can't be accessed from any other namespace but the one where it currently resides.

#### Story 2

> As a tenant, I want to share my app repository with another tenant
> so that they can deploy the application in their own namespace.

If `dev-team1` wants to grant read access to their repository to `dev-team2`,
a `matchLabels` that selects the namespace owned by `dev-team2` can be used:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: app1
  namespace: dev-team1
spec:
  url: ssh://git@github.com/<org>/app1-deploy
  secretRef:
    name: app1-ro-ssh-key
  accessFrom:
    namespaceSelectors:
      - matchLabels:
          kubernetes.io/metadata.name: dev-team2
```

#### Story 3

> As a cluster admin, I want to let tenants configure image automation in their namespaces by
> referring to a Git repository managed by the platform team.

If the owner of a Flux `GitRepository` wants to grant write access to `ImageUpdateAutomations` in a different namespace,
a `matchLabels` that selects the image automation namespace can be used:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: cluster-config
  namespace: flux-system
spec:
  url: ssh://git@github.com/<org>/cluster-config
  secretRef:
    name: read-write-ssh-key
  accessFrom:
    namespaceSelectors:
      - matchLabels:
          kubernetes.io/metadata.name: dev-team1
```

The `dev-team1` can refer to the `cluster-config` repository in their image automation config:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: app1
  namespace: dev-team1
spec:
  sourceRef:
    kind: GitRepository
    name: cluster-config
    namespace: flux-system
```

### Alternatives

#### Admission controllers

An alternative solution to source ACLs is showcased in the current multi-tenancy example, where an
admission controller such as Kyverno or OPA Gatekeeper is used to block cross-namespace access to sources.

The current proposal offers the same feature but without the need to manage yet another controller to guard
sources.

#### Kubernetes RBAC

Another alternative is to rely on impersonation and create a `ClusterRoleBinding` per named source and tenant account
as described in [fluxcd/flux2#582](https://github.com/fluxcd/flux2/pull/582). 

The current proposal is more flexible than RBAC and implies less work for Flux users. ALCs act more like
Kubernetes Network Policies where access is define based on labels, with RBAC every time a namespace is added,
the platform admins have to create new RBAC rules to target that namespace.

#### Source reflection CRD

Yet another alternative is to introduce a new API kind `SourceReflection` as described in
[fluxcd/flux2#582-821027543](https://github.com/fluxcd/flux2/pull/582#issuecomment-821027543).

The current proposal allows the owner to define the access control list on the source object, instead
of creating objects in namespaces where it has no control over.

#### Remove cross-namespace refs

An alternative is to simply remove cross-namespace references from the Flux API.

This would break with current behavior, and users would have to make substantial changes to their
repository structure and workflow. In cases where e.g. a resource is common (across many namespaces),
this would mean the source-controller would use way more memory and network bandwidth that grows with
each namespace that uses the same Git or Helm repository due to the requirement of having to duplicate
"common" resources.

## Implementation History

- ACL support for allowing cross-namespace access to `ImageRepositories` was first released in flux2 **v0.23.0**.
