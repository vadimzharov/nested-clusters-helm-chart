# nested-clusters-helm-chart

## Overview

This Helm chart prepares a Spectro Cloud Virtual Machine Orchestrator (VMO) cluster to deploy Nested Kubernetes Clusters based on Edge native architecture. It automates the creation of infrastructure and configuration required to run multiple Nested K8S cluster nodes on the VMO platform.

## What This Chart Creates

The chart creates the following resources on your Spectro Cloud VMO cluster:

- **DataVolume**: Based on Ubuntu 22.04 image, used as the source for Nested K8S cluster node VMs
- **VmTemplates**: One or multiple templates configured to run Nested Kubernetes cluster nodes (can be customized for different node sizes: small, medium, etc.)
- **Namespaces**: Dedicated namespaces to host VMs with Nested K8S cluster nodes
- **ResourceQuotas**: CPU, memory, and storage quotas for each namespace to enforce resource limits
- **Secrets**: Cloud-init configuration secrets per namespace with Spectro Palette agent settings and credentials
- **RBAC**: Role and RoleBinding to enable DataVolume clone operations from the golden image namespace to nested cluster namespaces

### Specific Resources Generated

- One `Namespace` per item in `.Values.k8snamespaces`
- One `ResourceQuota` named `nested-k8s-quota` per namespace (when quota settings are available)
- One `DataVolume` when `.Values.k8sTemplateParams.nestedK8sDv` is configured
- Multiple `VmTemplate` resources from the list in `.Values.k8sTemplateParams.k8sVmTemplate` (supports small, medium, large node configurations)
- One `Role` and one `RoleBinding` for DataVolume clone access in `.Values.k8sTemplateParams.namespace`
- One `Secret` named `v-k8s-cloudinit` in each namespace from `.Values.k8snamespaces`

### Resource Protection

Resources include `helm.sh/resource-policy: keep`, so Helm will not delete them on uninstall or when they are removed from values during upgrades, protecting your infrastructure from accidental deletion.

## Values

### Full schema

```yaml
k8snamespaces:
  - name: team-a
    edgeHostToken: "NTUXXXXX="
    quota:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
      storageClassRequests:
        gp3: "200Gi"

defaultQuota:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
  storageClassRequests:
    gp3: "500Gi"

k8sTemplateParams:
  namespace: "vmo-golden-images"
  nestedK8sDv:
    name: "nested-k8s-ubuntu-2204"
    accessModes:
      - ReadWriteMany
    storage: "50Gi"
    volumeMode: "Block"
    storageClassName: "freenas-iscsi-csi"
    source:
      registry:
        url: "docker://us-docker.pkg.dev/palette-images/palette/virtual-machine-orchestrator/os/ubuntu-container-disk:22.04"
  k8sVmTemplate:
    name: "ubuntu-2204-k8s"
    description: "Virtual Kubernetes node (Ubuntu)"
    runStrategy: "Always"
    networkName: "default/home-vlan-90"
    storage: "50Gi"
    cpu: 4
    memory: "6Gi"
    password: "spectro"
    paletteEndpoint: "api.spectrocloud.com"
```

### Behavior

- If `k8snamespaces[].quota` is set, it is used for that namespace.
- If `k8snamespaces[].quota` is not set, `defaultQuota` is used.
- If neither `k8snamespaces[].quota` nor `defaultQuota` is set, no `ResourceQuota` is created for that namespace.
- Storage quota can be set globally (`requests.storage`) and/or per storage class (`storageClassRequests`).
- A `ResourceQuota` is still created when only `storageClassRequests` is set (without `requests.storage`).

### Example: default quota for all namespaces

```yaml
defaultQuota:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
  storageClassRequests:
    gp3: "500Gi"

k8snamespaces:
  - name: team-a
  - name: team-b
```

### Example: per-namespace override

```yaml
defaultQuota:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
  storageClassRequests:
    gp3: "500Gi"

k8snamespaces:
  - name: team-a
  - name: team-b
    quota:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
      storageClassRequests:
        gp3: "100Gi"
        archive-hdd: "2Ti"
```

### Example: no ResourceQuota creation

```yaml
k8snamespaces:
  - name: team-a
  - name: team-b
```

### Example: only storageclass quota (no requests.storage)

```yaml
defaultQuota:
  storageClassRequests:
    gp3: "250Gi"

k8snamespaces:
  - name: team-a
  - name: team-b
    quota:
      storageClassRequests:
        gp3: "50Gi"
        fast-ssd: "20Gi"
```

Quota mapping:

- `quota.requests.cpu` -> `requests.cpu`
- `quota.requests.memory` -> `requests.memory`
- `quota.requests.storage` -> `requests.storage`
- `quota.limits.cpu` -> `limits.cpu`
- `quota.limits.memory` -> `limits.memory`
- `quota.limits.storage` -> `limits.ephemeral-storage`
- `quota.storageClassRequests.<storageClassName>` -> `<storageClassName>.storageclass.storage.k8s.io/requests.storage`

The same mapping applies to `defaultQuota`.

## DataVolume

The chart renders `templates/nested-k8s-dv.yaml` only when `k8sTemplateParams.nestedK8sDv` is set in values.

Supported fields:

- `k8sTemplateParams.namespace`
- `k8sTemplateParams.nestedK8sDv.name`
- `k8sTemplateParams.nestedK8sDv.accessModes`
- `k8sTemplateParams.nestedK8sDv.storage`
- `k8sTemplateParams.nestedK8sDv.volumeMode`
- `k8sTemplateParams.nestedK8sDv.storageClassName`
- `k8sTemplateParams.nestedK8sDv.source.registry.url`

Example:

```yaml
k8sTemplateParams:
  namespace: "vmo-golden-images"
  nestedK8sDv:
    name: "nested-k8s-ubuntu-2204"
    accessModes:
      - ReadWriteMany
    storage: "50Gi"
    volumeMode: "Block"
    storageClassName: "freenas-iscsi-csi"
    source:
      registry:
        url: "docker://us-docker.pkg.dev/palette-images/palette/virtual-machine-orchestrator/os/ubuntu-container-disk:22.04"
```

## VmTemplate

The chart renders `templates/k8s-vm-template.yaml` only when `k8sTemplateParams.k8sVmTemplate` is set in values.

Supported fields:

- `k8sTemplateParams.namespace`
- `k8sTemplateParams.k8sVmTemplate.name`
- `k8sTemplateParams.k8sVmTemplate.description` (used for both `spec.description` and `spec.displayName`)
- `k8sTemplateParams.k8sVmTemplate.runStrategy`
- `k8sTemplateParams.k8sVmTemplate.networkName` (used in `spec.template.spec.networks[].multus.networkName`)
- `k8sTemplateParams.k8sVmTemplate.storage`
- `k8sTemplateParams.k8sVmTemplate.cpu`
- `k8sTemplateParams.k8sVmTemplate.memory`
- `k8sTemplateParams.k8sVmTemplate.password` (used by cloud-init secret)
- `k8sTemplateParams.k8sVmTemplate.paletteEndpoint` (used by cloud-init secret)

Notes:

- Interface name is hardcoded to `default`.
- Network entry name is hardcoded to `default`.
- Users can set only the Multus `networkName` value.

Example:

```yaml
k8sTemplateParams:
  namespace: "vmo-golden-images"
  k8sVmTemplate:
    name: "ubuntu-2204-k8s"
    description: "Virtual Kubernetes node (Ubuntu)"
    runStrategy: "Always"
    networkName: "default/home-vlan-90"
    storage: "50Gi"
    cpu: 4
    memory: "6Gi"
```

## Secret

The chart renders `templates/k8s-secret.yaml` once per namespace listed in `k8snamespaces`.

Supported fields:

- `k8sTemplateParams.k8sVmTemplate.password`
- `k8sTemplateParams.k8sVmTemplate.paletteEndpoint`
- `k8snamespaces[].edgeHostToken`

Example:

```yaml
k8snamespaces:
  - name: team-a
    edgeHostToken: "NTUXXXXX="
  - name: team-b
    edgeHostToken: "NTUYYYYY="

k8sTemplateParams:
  namespace: "vmo-golden-images"
  k8sVmTemplate:
    name: "ubuntu-2204-k8s"
    description: "Virtual Kubernetes node (Ubuntu)"
    runStrategy: "Always"
    networkName: "default/home-vlan-90"
    storage: "50Gi"
    cpu: 4
    memory: "6Gi"
    password: "spectro"
    paletteEndpoint: "api.spectrocloud.com"
```

## DataVolume Clone RBAC

The chart renders `templates/k8s-dv-clone-rbac.yaml` with fixed resource names:

- `Role`: `vmo-role-k8s-nested-images`
- `RoleBinding`: `vmo-rolebinding-k8s-nested-images`

Behavior:

- Both resources are created in `k8sTemplateParams.namespace`.
- `RoleBinding.subjects` is generated from `k8snamespaces`.
- For each namespace in `k8snamespaces`, a subject is added with:
  - `kind: ServiceAccount`
  - `name: default`
  - `namespace: <k8snamespaces[].name>`

## Install

```bash
helm install nested-k8s . -f values.yaml
```

## Upgrade

```bash
helm upgrade nested-k8s . -f values.yaml
```
