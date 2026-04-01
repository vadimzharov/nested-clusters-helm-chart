# nested-clusters-helm-chart

This Helm chart creates:

- One `Namespace` per item in `.Values.k8snestednamespaces`
- One `ResourceQuota` named `nested-k8s-quota` only when quota settings are available
- One `DataVolume` when `.Values.k8sTemplateParams.nestedK8sDv` is configured
- One `VmTemplate` when `.Values.k8sTemplateParams.k8sVmTemplate` is configured
- One `Role` and one `RoleBinding` for DataVolume clone access in `.Values.k8sTemplateParams.namespace`
- One `Secret` named `v-k8s-cloudinit` in each namespace from `.Values.k8snestednamespaces`

Both resources include `helm.sh/resource-policy: keep`, so Helm will not delete them on uninstall or when they are removed from values during upgrades.

## Values

### Full schema

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

k8snestednamespaces:
  - name: team-a
    quota:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
      storageClassRequests:
        gp3: "200Gi"

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

k8sNodeCustomization:
  password: "spectro"
  edgeHostToken: "NTUXXXXX="
  paletteEndpoint: "api.spectrocloud.com"
```

### Behavior

- If `k8snestednamespaces[].quota` is set, it is used for that namespace.
- If `k8snestednamespaces[].quota` is not set, `defaultQuota` is used.
- If neither `k8snestednamespaces[].quota` nor `defaultQuota` is set, no `ResourceQuota` is created for that namespace.
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

k8snestednamespaces:
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

k8snestednamespaces:
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
k8snestednamespaces:
  - name: team-a
  - name: team-b
```

### Example: only storageclass quota (no requests.storage)

```yaml
defaultQuota:
  storageClassRequests:
    gp3: "250Gi"

k8snestednamespaces:
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

The chart renders `templates/k8s-secret.yaml` once per namespace listed in `k8snestednamespaces`.

Supported fields:

- `k8sNodeCustomization.password`
- `k8sNodeCustomization.edgeHostToken`
- `k8sNodeCustomization.paletteEndpoint`

Example:

```yaml
k8snestednamespaces:
  - name: team-a
  - name: team-b

k8sNodeCustomization:
  password: "spectro"
  edgeHostToken: "NTUXXXXX="
  paletteEndpoint: "api.spectrocloud.com"
```

## DataVolume Clone RBAC

The chart renders `templates/k8s-dv-clone-rbac.yaml` with fixed resource names:

- `Role`: `vmo-role-k8s-nested-images`
- `RoleBinding`: `vmo-rolebinding-k8s-nested-images`

Behavior:

- Both resources are created in `k8sTemplateParams.namespace`.
- `RoleBinding.subjects` is generated from `k8snestednamespaces`.
- For each namespace in `k8snestednamespaces`, a subject is added with:
  - `kind: ServiceAccount`
  - `name: default`
  - `namespace: <k8snestednamespaces[].name>`

## Install

```bash
helm install nested-k8s . -f values.yaml
```

## Upgrade

```bash
helm upgrade nested-k8s . -f values.yaml
```
