# Multi-tenancy with Config Sync, Capsule and Self Service
This repo is a example of how to setup multitenancy namespace provisioning with Config Sync and Capsule, with self service of creating namespaces

Config sync will sync the Tenants, each Tenant is owned by a maintainer group, the maintainers will have permissions to create and delete namespaces as well as custom access defined in their cluster role (view on most things in this set-up)

## Background and add-ons to install:
- [Config Sync](https://github.com/GoogleContainerTools/kpt-config-sync)
  ```sh
  # Set the release version
  export CS_VERSION=v1.19.2
  # Apply core Config Sync manifests to your cluster
  kubectl apply -f "https://github.com/GoogleContainerTools/kpt-config-sync/releases/download/${CS_VERSION}/config-sync-manifest.yaml"
  ```
- [Capsule](https://github.com/projectcapsule/capsule) - Install with the capsuleUserGroups mapped to your organisations group for RBAC, i.e. `gke-security-groups@yourdomain.dev`
  ```sh
  helm upgrade -i  \
    capsule \
    ./capsule/charts/capsule \
    -n capsule-system \
    --set manager.options.capsuleUserGroups.0="mydomain.dev" \
    --set manager.options.forceTenantPrefix=true \
    --create-namespace
  ```
  - For testing in local clusters i.e. minikube, you can use the [hack script](https://github.com/projectcapsule/capsule/blob/main/hack/create-user.sh) to create a user mapping to a tenant and org group. 
  - i.e. In a separate terminal (after applying set-up RootSyncs below), `./create-user.sh samir tenant-abc123 mydomain.dev` and act as the user to create namespaces in the Tenant - `export KUBECONFIG=samir-tenant-abc123.kubeconfig`

- Apply the RootSyncs:
  ```sh
  kubectl apply -f tenant-rootsync.yaml
  kubectl apply -f global-tenant-resourcs-rootsync.yaml
  ```

## Setup
- Config Sync will act as the robot GitOps account, vending `Tenants`, and `GlobalTenantResources`
- All `Tenants` are handled by a `tenant` `RootSync`
- Each `Tenant` is owned by a Maintainer Group (i.e. a subgroup in your orgs security group used for RBAC)
- Users in a `Tenants` maintainer group can create/delete namespaces, automatically scoped to the `Tenant` - Cluster Roles are synced and refined for maintainers (in this setup, maintainer can create/delete namespaces for their Tenant, and only view objects in their namespace)
- A maintainer group can own more than 1 Tenant, Capsule scopes the Namespaces to a Tenant on create by enforcing a tenant prefix in the Namespace name (uses a Validating Webhook).
- Capsule takes care of the rest - syncing `GlobalTenantResources` that match labels on `Tenant` objects, eliminating the overhead on Config Sync to reconcile all the Namespaced objects.
- Role Bindings for custom access to namespaces are patched in the subteam `patch.yaml`
- The sample files and layout can all be automated using scripts/apis/pipelines and hooked up to an IDP, essentially updating a git repo and for ConfigSync to mange Tenants, empowering team maintainers to manage their namespaces and Capsule to do the rest.

### Global Namespace Resources
- All global resources are handled by a separate `RootSync` `global-tenant-resourcs`
    - This syncs Service Account and Network Policies to all matching Tenants (to all of their namesapces), you can add more custom objects.
    - The global resources have been configured for a blue/green scenario:
    - All Tenants are vended with the label `global-resources: stable`
    - Each subset of resources under `globalTenantResources/base/` has a blue and green copy, admins can test new global resources by patching the `global-resources` label for blue/green `GlobalTenantResources` to be stable/beta in the cluster folder `kustomization.yaml`. i.e. `globalTenantResources/envs/dev/dev-cluster/kustomization.yaml`
    ```yaml
    # Patch the matching colour as the stable version
    - path: patch-stable.yaml
    target:
        kind: GlobalTenantResource
        labelSelector: "colour=green"

    # Patch the matching colour as the beta version
    - path: patch-beta.yaml
    target:
        kind: GlobalTenantResource
        labelSelector: "colour=blue"
    ```

## Scenario: I want to test a new Network Policy:
- Add the new policy into the current beta (`globalTenantResources/base/network-policies/green.yaml`), 
- Update the tenant label for `global-resources` in a test subteam `patch.yaml` to `beta`
- Commit and push the changes, let Capsule sync, and observe the new Network Policy in the test tenants namespace.
- Once satisfied with tests, roll out to all tenants
- Simply switch the label on the cluster kustomization.yaml `globalTenantResources/envs/dev/dev-cluster/kustomization.yaml` for green to be stable (backout would be revering the patch for blue to be stable), or copy the updated policy to the `blue.yaml` under `globalTenantResources/base/network-policies/`
- In this scenario it is important to keep blue and green config consistent after a change is rolled out.

## Resource Quota scoping
Resource Quotas are scoped to `Tenant` level in the `Tenant` spec, patched in the subteams `kustomization.yaml`. All `Namespaces` created in the Tenant will inherit the same Resource Quota, Capsule aggregates the combined usage of all `Namespaces` and enforces the quota accross all the namespaces matching the `Tenant`.
