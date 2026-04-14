# DPA v1 API Design

## Abstract

The DataProtectionApplication (DPA) CRD has accumulated inconsistencies in field naming and typing since `v1alpha1`.
This document catalogs every field in the current API and identifies changes needed for a version bump.

## Background

Two Jira issues drive this work:
- **OADP-3692**: `defaultVolumesToFSBackup` uses capital `S`, while the upstream Velero Backup spec uses `defaultVolumesToFsBackup` (lowercase `s`). This confuses users.
- **OADP-3471**: Several timeout and duration fields use raw `string` type instead of `metav1.Duration`, diverging from upstream Velero's format (e.g. `"4h0m0s"`).

Both are breaking API changes that cannot be made in-place without a version bump.

## Goals

- Catalog every field in the current `v1alpha1` DPA API.
- Identify fields that need renaming, retyping, or removal.
- Provide a reference for designing the new API version.

## Non Goals

- Defining the conversion webhook implementation (separate design).
- Adding new features to the API.

---

## Current API Field Reference (`v1alpha1`)

### DataProtectionApplicationSpec

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `BackupLocations` | `[]BackupLocation` | `backupLocations` | |
| `SnapshotLocations` | `[]SnapshotLocation` | `snapshotLocations` | |
| `UnsupportedOverrides` | `map[UnsupportedImageKey]string` | `unsupportedOverrides` | Image FQIN overrides |
| `PodAnnotations` | `map[string]string` | `podAnnotations` | **Deprecated** — use PodConfig |
| `ResourceLabels` | `map[string]string` | `resourceLabels` | |
| `ResourceAnnotations` | `map[string]string` | `resourceAnnotations` | |
| `PodDnsPolicy` | `corev1.DNSPolicy` | `podDnsPolicy` | |
| `PodDnsConfig` | `corev1.PodDNSConfig` | `podDnsConfig` | |
| `BackupImages` | `*bool` | `backupImages` | |
| `Configuration` | `*ApplicationConfig` | `configuration` | Required |
| `Features` | `*Features` | `features` | |
| `ImagePullPolicy` | `*corev1.PullPolicy` | `imagePullPolicy` | |
| `NonAdmin` | `*NonAdmin` | `nonAdmin` | |
| `VMFileRestore` | `*VMFileRestore` | `vmFileRestore` | |
| `LogFormat` | `LogFormat` | `logFormat` | Enum: text, json |

### ApplicationConfig

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Velero` | `*VeleroConfig` | `velero` | |
| `Restic` | `*ResticConfig` | `restic` | **Deprecated** — use nodeAgent |
| `NodeAgent` | `*NodeAgentConfig` | `nodeAgent` | |
| `RepositoryMaintenance` | `map[string]RepositoryMaintenanceConfig` | `repositoryMaintenance` | |

### VeleroConfig

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `FeatureFlags` | `[]string` | `featureFlags` | |
| `DefaultPlugins` | `[]DefaultPlugin` | `defaultPlugins` | |
| `CustomPlugins` | `[]CustomPlugin` | `customPlugins` | |
| `RestoreResourcesVersionPriority` | `string` | `restoreResourcesVersionPriority` | |
| `NoDefaultBackupLocation` | `bool` | `noDefaultBackupLocation` | |
| `PodConfig` | `*PodConfig` | `podConfig` | |
| `LogLevel` | `string` | `logLevel` | Enum: trace..panic |
| `ItemOperationSyncFrequency` | `string` | `itemOperationSyncFrequency` | **CHANGE: string -> *metav1.Duration** |
| `DefaultItemOperationTimeout` | `string` | `defaultItemOperationTimeout` | **CHANGE: string -> *metav1.Duration** |
| `DefaultVolumesToFSBackup` | `*bool` | `defaultVolumesToFSBackup` | **CHANGE: rename -> defaultVolumesToFsBackup** |
| `DisableFsBackup` | `*bool` | `disableFsBackup` | |
| `DefaultSnapshotMoveData` | `*bool` | `defaultSnapshotMoveData` | |
| `DisableInformerCache` | `*bool` | `disableInformerCache` | |
| `ItemBlockWorkerCount` | `int` | `itemBlockWorkerCount` | |
| `ConcurrentBackups` | `int` | `concurrentBackups` | |
| `ResourceTimeout` | `string` | `resourceTimeout` | **CHANGE: string -> *metav1.Duration** |
| `ClientBurst` | `*int` | `client-burst` | |
| `ClientQPS` | `*int` | `client-qps` | |
| `Args` | `*VeleroServerArgs` | `args` | Direct server flag overrides |
| `LoadAffinityConfig` | `[]*LoadAffinity` | `loadAffinity` | |

### NodeAgentCommonFields

Embedded in both `NodeAgentConfig` and `ResticConfig`.

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Enable` | `*bool` | `enable` | |
| `SupplementalGroups` | `[]int64` | `supplementalGroups` | |
| `Timeout` | `string` | `timeout` | **CHANGE: string -> *metav1.Duration** |
| `PodConfig` | `*PodConfig` | `podConfig` | |

### NodeAgentConfig

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| *(inline)* | `NodeAgentCommonFields` | | See above |
| `DataMoverPrepareTimeout` | `*metav1.Duration` | `dataMoverPrepareTimeout` | Already correct type |
| `ResourceTimeout` | `*metav1.Duration` | `resourceTimeout` | Already correct type |
| `UploaderType` | `string` | `uploaderType` | Enum: restic, kopia. **restic value deprecated** |
| *(inline)* | `NodeAgentConfigMapSettings` | | See below |
| *(inline)* | `KopiaRepoOptions` | | See below |

### NodeAgentConfigMapSettings

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `LoadConcurrency` | `*LoadConcurrency` | `loadConcurrency` | |
| `LoadAffinityConfig` | `[]*LoadAffinity` | `loadAffinity` | |
| `BackupPVCConfig` | `map[string]types.BackupPVC` | `backupPVC` | Manual DeepCopy |
| `RestorePVCConfig` | `*types.RestorePVC` | `restorePVC` | |
| `PodResources` | `*kube.PodResources` | `podResources` | |
| `CachePVCConfig` | `*types.CachePVC` | `cachePVC` | |

### KopiaRepoOptions

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `CacheLimitMB` | `*int64` | `cacheLimitMB` | |
| `FullMaintenanceInterval` | `FullMaintenanceInterval` | `fullMaintenanceInterval` | Enum: normalGC, fastGC, eagerGC |

### ResticConfig (Deprecated)

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| *(inline)* | `NodeAgentCommonFields` | | Same fields as above |

### RepositoryMaintenanceConfig

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `LoadAffinityConfig` | `[]*LoadAffinity` | `loadAffinity` | |
| `PodResources` | `*kube.PodResources` | `podResources` | |

### PodConfig

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Labels` | `map[string]string` | `labels` | |
| `Annotations` | `map[string]string` | `annotations` | |
| `NodeSelector` | `map[string]string` | `nodeSelector` | |
| `Tolerations` | `[]corev1.Toleration` | `tolerations` | |
| `ResourceAllocations` | `corev1.ResourceRequirements` | `resourceAllocations` | |
| `Env` | `[]corev1.EnvVar` | `env` | |
| `PriorityClassName` | `string` | `priorityClassName` | |

### BackupLocation

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Name` | `string` | `name` | |
| `Velero` | `*velero.BackupStorageLocationSpec` | `velero` | |
| `CloudStorage` | `*CloudStorageLocation` | `bucket` | |

### CloudStorageLocation

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `CloudStorageRef` | `corev1.LocalObjectReference` | `cloudStorageRef` | |
| `Config` | `map[string]string` | `config` | |
| `Credential` | `*corev1.SecretKeySelector` | `credential` | |
| `Default` | `bool` | `default` | |
| `BackupSyncPeriod` | `*metav1.Duration` | `backupSyncPeriod` | Already correct type |
| `Prefix` | `string` | `prefix` | |
| `CACert` | `[]byte` | `caCert` | |

### SnapshotLocation

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Name` | `string` | `name` | |
| `Velero` | `*velero.VolumeSnapshotLocationSpec` | `velero` | |

### NonAdmin

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Enable` | `*bool` | `enable` | |
| `EnforceBackupSpec` | `*velero.BackupSpec` | `enforceBackupSpec` | |
| `EnforceRestoreSpec` | `*velero.RestoreSpec` | `enforceRestoreSpec` | |
| `EnforceBSLSpec` | `*EnforceBackupStorageLocationSpec` | `enforceBSLSpec` | |
| `RequireApprovalForBSL` | `*bool` | `requireApprovalForBSL` | |
| `GarbageCollectionPeriod` | `*metav1.Duration` | `garbageCollectionPeriod` | Already correct type |
| `BackupSyncPeriod` | `*metav1.Duration` | `backupSyncPeriod` | Already correct type |

### EnforceBackupStorageLocationSpec

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Provider` | `string` | `provider` | |
| `Config` | `map[string]string` | `config` | |
| `Credential` | `*corev1.SecretKeySelector` | `credential` | |
| *(inline)* | `StorageType` | | Contains ObjectStorage |
| `AccessMode` | `velero.BackupStorageLocationAccessMode` | `accessMode` | |
| `BackupSyncPeriod` | `*metav1.Duration` | `backupSyncPeriod` | Already correct type |
| `ValidationFrequency` | `*metav1.Duration` | `validationFrequency` | Already correct type |

### VMFileRestore

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Enable` | `*bool` | `enable` | |
| `Resources` | `*corev1.ResourceRequirements` | `resources` | |

### Features

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `DataMover` | `*DataMover` | `dataMover` | **Deprecated** — use Velero built-in data mover |

### DataMover (Deprecated)

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `Enable` | `bool` | `enable` | |
| `CredentialName` | `string` | `credentialName` | |
| `Timeout` | `string` | `timeout` | **CHANGE: string -> *metav1.Duration** |
| `MaxConcurrentBackupVolumes` | `string` | `maxConcurrentBackupVolumes` | |
| `MaxConcurrentRestoreVolumes` | `string` | `maxConcurrentRestoreVolumes` | |
| `PruneInterval` | `string` | `pruneInterval` | |
| `VolumeOptionsForStorageClasses` | `map[string]DataMoverVolumeOptions` | `volumeOptionsForStorageClasses` | |
| `SnapshotRetainPolicy` | `*RetainPolicy` | `snapshotRetainPolicy` | |
| `Schedule` | `string` | `schedule` | Cron expression |

### VeleroServerArgs (CLI flag overrides)

These are direct Velero server flags using `time.Duration` (nanosecond precision).
Users rarely set these directly; they exist for advanced overrides.

| Field | Type | JSON Tag | Notes |
|-------|------|----------|-------|
| `MetricsAddress` | `string` | `metrics-address` | |
| `BackupSyncPeriod` | `*time.Duration` | `backup-sync-period` | |
| `PodVolumeOperationTimeout` | `*time.Duration` | `fs-backup-timeout` | |
| `ResourceTerminatingTimeout` | `*time.Duration` | `terminating-resource-timeout` | |
| `DefaultBackupTTL` | `*time.Duration` | `default-backup-ttl` | |
| `StoreValidationFrequency` | `*time.Duration` | `store-validation-frequency` | |
| `RestoreResourcePriorities` | `string` | `restore-resource-priorities` | |
| `DisabledControllers` | `[]string` | `disabled-controllers` | |
| `ClientQPS` | `*string` | `client-qps` | |
| `ClientBurst` | `*int` | `client-burst` | |
| `ClientPageSize` | `*int` | `client-page-size` | |
| `ProfilerAddress` | `string` | `profiler-address` | |
| `ItemOperationSyncFrequency` | `*time.Duration` | `item-operation-sync-frequency` | |
| `FormatFlag` | `string` | `log-format` | |
| `RepoMaintenanceFrequency` | `*time.Duration` | `default-repo-maintain-frequency` | |
| `GarbageCollectionFrequency` | `*time.Duration` | `garbage-collection-frequency` | |
| `DefaultVolumesToFsBackup` | `*bool` | `default-volumes-to-fs-backup` | Note: already lowercase `s` here |
| `DefaultItemOperationTimeout` | `*time.Duration` | `default-item-operation-timeout` | |
| `ResourceTimeout` | `*time.Duration` | `resource-timeout` | |
| `MaxConcurrentK8SConnections` | `*int` | `max-concurrent-k8s-connections` | |
| `Colorized` | `*bool` | `colorized` | |
| *(inline)* | `LoggingFlags` | | klog flags |

---

## Summary of Required Changes

### Field Renames (OADP-3692)

| Struct | Current Field | Current JSON Tag | Proposed JSON Tag |
|--------|--------------|------------------|-------------------|
| `VeleroConfig` | `DefaultVolumesToFSBackup` | `defaultVolumesToFSBackup` | `defaultVolumesToFsBackup` |

### Type Changes (OADP-3471)

| Struct | Field | Current Type | Proposed Type |
|--------|-------|-------------|---------------|
| `VeleroConfig` | `ItemOperationSyncFrequency` | `string` | `*metav1.Duration` |
| `VeleroConfig` | `DefaultItemOperationTimeout` | `string` | `*metav1.Duration` |
| `VeleroConfig` | `ResourceTimeout` | `string` | `*metav1.Duration` |
| `NodeAgentCommonFields` | `Timeout` | `string` | `*metav1.Duration` |
| `DataMover` | `Timeout` | `string` | `*metav1.Duration` |

### Deprecated Fields to Consider Removing

| Struct | Field | Reason |
|--------|-------|--------|
| `DataProtectionApplicationSpec` | `PodAnnotations` | Superseded by PodConfig |
| `ApplicationConfig` | `Restic` | Superseded by NodeAgent |
| `Features` | `DataMover` | Superseded by Velero built-in data mover |
| `NodeAgentConfig.UploaderType` | `restic` enum value | Comment says remove in v2 |

## Open Issues

- Target API version name (`v1alpha2`, `v1beta1`, `v1`)?
- Conversion webhook design (hub-and-spoke model, which version is hub)?
- Should deprecated fields be removed outright or retained with stronger deprecation markers?
- Are there additional inconsistencies to fix in the same bump?
