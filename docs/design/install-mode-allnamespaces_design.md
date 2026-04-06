# AllNamespaces Install Mode for OADP Operator

## Abstract

This proposal describes converting the OADP operator from `OwnNamespace` install mode to `AllNamespaces` install mode.
Currently the operator watches and manages resources only within its own installation namespace.
AllNamespaces mode enables a single operator deployment to manage DPA instances across multiple namespaces, improving multi-tenancy and reducing operational overhead.

## Background

OADP currently runs in `OwnNamespace` install mode, meaning the operator is restricted to watching and reconciling resources in the namespace where it is installed (typically `openshift-adp`).
This is enforced by the `WATCH_NAMESPACE` environment variable (set via downward API to the pod's own namespace) and the manager's `cache.Options.DefaultNamespaces` configuration.

The OLM CSV explicitly marks only `OwnNamespace` as supported:

```yaml
installModes:
- supported: true
  type: OwnNamespace
- supported: false
  type: SingleNamespace
- supported: false
  type: MultiNamespace
- supported: false
  type: AllNamespaces
```

Despite this single-namespace operational mode, OADP's RBAC is already configured with ClusterRole and ClusterRoleBinding resources, granting the operator's service account permissions across all namespaces.
This means the permission model is already compatible with AllNamespaces mode.

In an AllNamespaces deployment, a single OADP operator instance watches for DPA resources in all namespaces and deploys Velero stacks per-namespace as needed.
This is a common pattern for operators managing infrastructure services across an OpenShift cluster.

## Goals

- Enable the OADP operator to run in `AllNamespaces` install mode, watching and reconciling DPA resources across all namespaces from a single operator deployment.
- Maintain backward compatibility with `OwnNamespace` install mode for existing deployments.
- Ensure each namespace with a DPA gets its own isolated Velero stack (deployment, BSLs, VSLs, node agent, etc.).

## Non Goals

- Introducing a cluster-scoped DPA CRD. The DPA will remain a namespaced resource.
- Cross-namespace backup/restore orchestration. Each DPA/Velero instance operates independently within its namespace.
- Changing the Non-Admin controller's multi-tenancy model. NonAdmin resources remain scoped to their respective namespaces.
- Removing support for `OwnNamespace` install mode.

## High-Level Design

The conversion involves three categories of changes:

1. **Manager cache configuration**: Remove the single-namespace restriction on the manager's cache so that controllers watch all namespaces for DPA and related resources.
2. **Resource naming and scoping**: Audit and fix hardcoded resource name prefixes and namespace assumptions so that resources created per-DPA are unique and correctly scoped.
3. **OLM/CSV and deployment configuration**: Update the CSV install modes, the `WATCH_NAMESPACE` environment variable handling, and any deployment manifests to support AllNamespaces mode.

The operator will continue to create all managed resources (Velero deployment, BSLs, VSLs, services, etc.) in the same namespace as the DPA that triggered their creation.
This per-namespace isolation is already the current behavior and requires no fundamental changes.

## Detailed Design

### 1. Manager Cache Configuration

**Current state** (`cmd/main.go:204-208`):
```go
Cache: cache.Options{
    DefaultNamespaces: map[string]cache.Config{
        watchNamespace: {},
    },
},
```

**Proposed change**: When `WATCH_NAMESPACE` is empty (or a new configuration flag is set), configure the manager cache without namespace restrictions:

```go
cacheOpts := cache.Options{}
if watchNamespace != "" {
    cacheOpts.DefaultNamespaces = map[string]cache.Config{
        watchNamespace: {},
    }
}

mgr, err := ctrl.NewManager(kubeconf, ctrl.Options{
    // ...
    Cache: cacheOpts,
    // ...
})
```

**`WATCH_NAMESPACE` handling** (`cmd/main.go:343-355`): The `getWatchNamespace()` function currently returns an error if `WATCH_NAMESPACE` is unset.
This will be changed to allow an empty value, which signals AllNamespaces mode:

```go
func getWatchNamespace() (string, error) {
    var watchNamespaceEnvVar = "WATCH_NAMESPACE"
    ns, found := os.LookupEnv(watchNamespaceEnvVar)
    if !found {
        return "", nil // AllNamespaces mode
    }
    return ns, nil
}
```

### 2. DPA Validation: One DPA Per Namespace

**Current state** (`internal/controller/validator.go:28-35`):
The operator enforces that only one DPA can exist per namespace.
This validation already scopes the check to the DPA's namespace using `client.ListOptions{Namespace: r.NamespacedName.Namespace}`.

**No change required.** This validation naturally extends to AllNamespaces mode: each namespace is allowed one DPA, and each DPA gets its own Velero stack.

### 3. Managed Resource Creation

**Current state**: All managed resources (Velero deployment, node agent, BSLs, VSLs, services, configmaps) are created in the DPA's namespace using `dpa.Namespace` or `r.NamespacedName.Namespace`.

Key files:
- `internal/controller/velero.go:78-87` - Velero Deployment
- `internal/controller/bsl.go:183-186` - BackupStorageLocations
- `internal/controller/nonadmin_controller.go:51-56` - Non-Admin Deployment
- `internal/controller/monitor.go:13-19` - Metrics Service

**No fundamental change required.** Resources are already scoped to the DPA namespace.

### 4. Hardcoded Name Prefixes

Several resources use the hardcoded prefix `openshift-adp`:

| Resource | Location | Current Name |
|----------|----------|-------------|
| Velero Metrics Service | `internal/controller/monitor.go:16` | `openshift-adp-velero-metrics-svc` |
| CLI Server Deployment | `internal/controller/cli_download_controller.go:25` | `openshift-adp-oadp-cli-server` |
| CLI Server Service | `internal/controller/cli_download_controller.go:26` | `openshift-adp-cli-server` |
| CLI Download | `internal/controller/cli_download_controller.go:28` | `openshift-adp-oadp-cli` |
| Label Selector | `internal/controller/velero.go:59` | `k8s-app: openshift-adp` |
| Operator Name | `cmd/main.go:308` | `openshift-adp-controller-manager` |

**Assessment**: These hardcoded prefixes are largely cosmetic/naming conventions and do not prevent multi-namespace operation, since the named resources are created within each DPA's namespace.
However, cluster-scoped resources like `ConsoleCLIDownload` need special handling (see section 6).

### 5. RBAC

**Current state**: The operator uses ClusterRole and ClusterRoleBinding (`config/rbac/role.yaml`, `config/rbac/role_binding.yaml`), granting cluster-wide permissions.

**No change required.** The RBAC model already supports AllNamespaces mode.

### 6. Cluster-Scoped Resources

Some resources managed by the operator are cluster-scoped and require special handling in AllNamespaces mode:

#### ConsoleCLIDownload / ConsoleVMDPDownload

**Current state** (`cmd/main.go:303-325`): These are set up once at operator startup with the operator's namespace.

**Proposed change**: In AllNamespaces mode, these cluster-scoped resources should be created once during operator startup (not per-DPA).
The CLI download server deployment should remain in the operator's own namespace.
The `ConsoleCLIDownload` resource points to a route, which references a specific namespace — this should use the operator's deployment namespace.

#### SecurityContextConstraints

**Current state**: SCCs are cluster-scoped and owned by the DPA.
In AllNamespaces mode, multiple DPAs could attempt to create/manage the same SCC.

**Proposed change**: SCCs should be managed by the operator at startup or reconciled with conflict resolution.
The operator should use a single shared SCC across all namespaces, with appropriate labels and ownership tracking.

### 7. Leader Election

**Current state** (`cmd/main.go`): Leader election is configured with a lease in the operator's namespace.

**No change required.** Leader election will continue to use a single lease since only one operator instance runs.

### 8. Non-Admin Controller

**Current state**: The non-admin controller deployment is created per-DPA in the DPA's namespace.
Its `WATCH_NAMESPACE` is set to the DPA's namespace.

**No change required.** Each DPA will deploy its own non-admin controller scoped to that namespace, which is the correct behavior.

### 9. OLM CSV Changes

**File**: `config/manifests/bases/oadp-operator.clusterserviceversion.yaml`

```yaml
installModes:
- supported: true
  type: OwnNamespace
- supported: false
  type: SingleNamespace
- supported: false
  type: MultiNamespace
- supported: true
  type: AllNamespaces
```

### 10. Manager Deployment Manifest

**File**: `config/manager/manager.yaml`

The `WATCH_NAMESPACE` environment variable configuration needs to support both modes:

For AllNamespaces mode, `WATCH_NAMESPACE` should either be omitted or set to an empty string.
This can be controlled via OLM subscription configuration or a kustomize overlay.

### 11. Secret Watching in AllNamespaces Mode

**Current state**: Secrets are watched via labels (`oadpApi.OadpOperatorLabel: "True"`) and the DPA name is embedded in a label.
The `labelHandler` (`internal/controller/dataprotectionapplication_controller.go:182-250`) extracts the DPA name from the secret's labels and enqueues reconciliation.

**Assessment**: This label-based mechanism already supports multiple namespaces — the handler uses the secret's own namespace to locate its parent DPA.
No change is required for the secret watch mechanism.

## Alternatives Considered

### Separate Operator Instance Per Namespace
Continue with OwnNamespace mode and deploy a separate operator instance per namespace.
Rejected because this increases resource usage, management overhead, and does not align with OpenShift operator best practices for infrastructure services.

### Cluster-Scoped DPA CRD
Make the DPA a cluster-scoped resource that references a target namespace.
Rejected because the existing namespaced DPA model naturally provides isolation and is simpler.
A cluster-scoped DPA would require reworking ownership, RBAC, and multi-tenant isolation.

## Security Considerations

- **Namespace isolation**: Each DPA/Velero stack operates independently within its namespace. Credentials in one namespace's BSL are not accessible to Velero instances in other namespaces.
- **RBAC**: The operator service account already has cluster-wide permissions. No escalation of privileges is required.
- **SCC management**: Moving to a shared SCC model requires careful handling to prevent one namespace from modifying SCCs used by another. The SCC should be immutable once created, with the operator as sole owner.
- **Non-Admin**: The non-admin controller per-namespace deployment model preserves tenant isolation — each controller only watches its own namespace.

## Compatibility

- **Upgrade path**: Existing OwnNamespace deployments must continue to function without disruption. The operator should detect the mode at startup and behave accordingly.
- **Downgrade**: Reverting to OwnNamespace mode should be possible by setting `WATCH_NAMESPACE` back to a specific namespace value.
- **OLM**: Both `OwnNamespace` and `AllNamespaces` install modes will be marked as supported in the CSV, allowing operators to choose during subscription creation.
- **Multiple DPAs**: AllNamespaces mode supports one DPA per namespace. The existing single-DPA-per-namespace validation remains in effect.

## Implementation

### Phase 1: Core Manager Changes
1. Update `getWatchNamespace()` to allow empty values.
2. Conditionally configure the manager cache based on watch namespace.
3. Update CSV install modes.
4. Update manager deployment manifest with kustomize overlay for AllNamespaces mode.

### Phase 2: Cluster-Scoped Resource Handling
1. Refactor ConsoleCLIDownload / ConsoleVMDPDownload to be managed independently from DPA lifecycle.
2. Implement shared SCC management with proper ownership and conflict resolution.

### Phase 3: Testing and Validation
1. Add unit tests for multi-namespace DPA reconciliation.
2. Add E2E tests deploying DPAs in multiple namespaces with a single operator.
3. Validate upgrade from OwnNamespace to AllNamespaces mode.
4. Test interaction between multiple Velero instances across namespaces.

## Open Issues

1. **SCC ownership model**: When multiple DPAs exist, which DPA "owns" the shared SCC? Should the operator itself own it outside of any DPA's lifecycle?
2. **Operator namespace for cluster-scoped resources**: In AllNamespaces mode, which namespace hosts the CLI download server deployment? Should this be the operator's own deployment namespace?
3. **Velero server flags**: Some Velero server flags may assume a single namespace. Need to verify that per-namespace Velero instances can coexist without conflict (e.g., restic/kopia repository locks, global configmaps).
4. **Resource limits**: Should there be a maximum number of DPAs/namespaces that can be managed by a single operator instance? What are the performance implications?
5. **Monitoring and metrics**: With multiple Velero instances, how should metrics be aggregated? Should ServiceMonitors be namespace-scoped or should there be a single aggregation point?
