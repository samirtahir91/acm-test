# Multitenantcy with Config Sync and Capsule
This repo is a example of how to setup multitenancy namespace provisioning with Config Sync and Capsule

- Config Sync will act as the robot GitOps account, vending Tenants and Namespaces (1:1 relationship), and `GlobalTenantResources`
- All `Tenants` and `Namespaces` are handled by a `namespace` `RootSync`
- All global resources are handled by a separate `RootSync` `global-tenant-resourcs`
    - This syncs Service Account, Network Policies and Resource Quotas, you can add more custom objects.
    - The global resources has been configured for a blue/green scenario:
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

### Scenario: I want to test a new Network Policy:
- Add the new policy into the current beta (`globalTenantResources/base/network-policies/green.yaml`), 
- Update my tenant label for `global-resources` in a test subteam `patch.yaml` to `beta`
- Commit and push the changes, let Capsule sync, and observe the new Network Policy in the test tenants namespace.
- Once satisfied with tests, roll out to all tenants
- Simply switch the label on the cluster kustomization.yaml `globalTenantResources/envs/dev/dev-cluster/kustomization.yaml` for green to be stable, or copy the updated policy to the `blue.yaml` under `globalTenantResources/base/network-policies/`

### ResourceQuotas scoping
ResourceQuotas are scoped to tenant namespaces using the `quota` label in the subteam `patch.yaml` (patches the Tenant with quota label)

The ResourceQuotas are maintained in `globalTenantResources/base/resource-quotas`, each having the tenantSelector to match Tenants for scoping, i.e.
```yaml
  tenantSelector:
    matchLabels:
      quota: low
```
