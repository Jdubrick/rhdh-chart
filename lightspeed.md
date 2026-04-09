# Lightspeed in `rhdh-chart`

This document describes how the built-in Lightspeed support is wired into the `charts/backstage` chart, which values are intentionally exposed for users, which pieces remain chart-managed, and how to enable the testing-only guest access override used for RBAC verification.

## Overview

When `global.lightspeed.enabled` is `true`, the chart wires in a built-in Lightspeed deployment model instead of expecting you to manage all of the pieces yourself.

At a high level, enabling Lightspeed causes the chart to:

- Add the default Lightspeed frontend and backend dynamic plugin packages to the rendered `dynamic-plugins.yaml`
- Create three Lightspeed ConfigMaps from vendored files:
  - `lightspeed-stack.yaml`
  - `config.yaml`
  - `rhdh-profile.py`
- Create a Lightspeed Secret from a vendored `secret.yaml` payload
- Add a Lightspeed RAG bootstrap init container
- Add a Lightspeed Core sidecar container listening on port `8080`
- Add a writable runtime volume mounted at `/tmp`
- Add a chart-managed RAG volume mounted at `/rag-content`
- Restart the Backstage Pod when the generated Lightspeed ConfigMaps or Secret content changes

The public values surface for this feature lives under `global.lightspeed` in `charts/backstage/values.yaml`.

## How values are resolved

There are two layers involved:

- `charts/backstage/values.yaml` defines the supported user-facing defaults and drives the generated README/schema
- `charts/backstage/templates/_helpers.tpl` contains an internal upgrade-safe default object used when a release is upgraded from an older chart that did not yet have `global.lightspeed`

Those two default sets are intentionally aligned for the exposed fields. The helper still contains additional internal fields that are not part of the supported values API, such as the generated ConfigMap definitions, Secret wiring, init container defaults, sidecar defaults, and RAG volume settings.

In practice, the supported shape is the one documented in `values.yaml` and enforced by `values.schema.json`. Treat other internal helper fields as implementation details rather than stable user-facing inputs.

## Supported user-facing fields

These are the fields that are intentionally exposed today under `global.lightspeed`.

### `global.lightspeed.enabled`

Turns the built-in Lightspeed feature on or off.

Default:

```yaml
global:
  lightspeed:
    enabled: true
```

When disabled:

- No Lightspeed plugin packages are added to `dynamic-plugins.yaml`
- No Lightspeed ConfigMaps or Secret are rendered
- No Lightspeed init container or sidecar is added to the Backstage Deployment

Example:

```yaml
global:
  lightspeed:
    enabled: false
```

### `global.lightspeed.plugins`

Controls the built-in Lightspeed dynamic plugin package list.

Default behavior:

- Installs the Lightspeed frontend plugin package
- Installs the Lightspeed backend plugin package
- Configures the frontend plugin to:
  - add a `/lightspeed` route
  - mount the drawer-based UI components through `mountPoints`
  - register translation resources

Default shape:

```yaml
global:
  lightspeed:
    plugins:
      - package: oci://ghcr.io/redhat-developer/rhdh-plugin-export-overlays/red-hat-developer-hub-backstage-plugin-lightspeed:bs_1.45.3__1.4.0!red-hat-developer-hub-backstage-plugin-lightspeed
        disabled: false
        pluginConfig:
          dynamicPlugins:
            frontend:
              red-hat-developer-hub.backstage-plugin-lightspeed:
                translationResources:
                  - importName: lightspeedTranslations
                    module: Alpha
                    ref: lightspeedTranslationRef
                dynamicRoutes:
                  - path: /lightspeed
                    importName: LightspeedPage
                mountPoints:
                  - mountPoint: application/listener
                    importName: LightspeedFAB
                  - mountPoint: application/provider
                    importName: LightspeedDrawerProvider
                  - mountPoint: application/internal/drawer-state
                    importName: LightspeedDrawerStateExposer
                    config:
                      id: lightspeed
                  - mountPoint: application/internal/drawer-content
                    importName: LightspeedChatContainer
                    config:
                      id: lightspeed
                      priority: 100
      - package: oci://ghcr.io/redhat-developer/rhdh-plugin-export-overlays/red-hat-developer-hub-backstage-plugin-lightspeed-backend:bs_1.45.3__1.4.0!red-hat-developer-hub-backstage-plugin-lightspeed-backend
        disabled: false
```

Important:

- `plugins` is a list, so overriding it replaces the whole list rather than merging individual entries
- If you override `global.lightspeed.plugins`, copy forward any default entries you still want
- Do not also keep the same Lightspeed packages in `global.dynamic.plugins`, or the rendered `dynamic-plugins.yaml` will contain duplicates

Example: replace the package references for a disconnected environment while keeping the built-in drawer-style config:

```yaml
global:
  lightspeed:
    plugins:
      - package: oci://registry.example.com/rhdh/lightspeed-frontend:custom-tag!red-hat-developer-hub-backstage-plugin-lightspeed
        disabled: false
        pluginConfig:
          dynamicPlugins:
            frontend:
              red-hat-developer-hub.backstage-plugin-lightspeed:
                translationResources:
                  - importName: lightspeedTranslations
                    module: Alpha
                    ref: lightspeedTranslationRef
                dynamicRoutes:
                  - path: /lightspeed
                    importName: LightspeedPage
                mountPoints:
                  - mountPoint: application/listener
                    importName: LightspeedFAB
                  - mountPoint: application/provider
                    importName: LightspeedDrawerProvider
                  - mountPoint: application/internal/drawer-state
                    importName: LightspeedDrawerStateExposer
                    config:
                      id: lightspeed
                  - mountPoint: application/internal/drawer-content
                    importName: LightspeedChatContainer
                    config:
                      id: lightspeed
                      priority: 100
      - package: oci://registry.example.com/rhdh/lightspeed-backend:custom-tag!red-hat-developer-hub-backstage-plugin-lightspeed-backend
        disabled: false
```

### `global.lightspeed.images`

Controls the two container images used by the chart-managed Lightspeed pod wiring:

- `ragInit`: the RAG bootstrap init container image
- `lightspeedCore`: the sidecar image

Default:

```yaml
global:
  lightspeed:
    images:
      ragInit: quay.io/redhat-ai-dev/rag-content:release-1.9-lls-0.5.0-642c567fe10a62b5ff711654306b72912f341e05
      lightspeedCore: quay.io/lightspeed-core/lightspeed-stack:0.5.0
```

Example:

```yaml
global:
  lightspeed:
    images:
      ragInit: registry.example.com/lightspeed/rag-content:custom
      lightspeedCore: registry.example.com/lightspeed/lightspeed-stack:custom
```

### `global.lightspeed.resources`

Lets you set Kubernetes resource requests and limits independently for the two Lightspeed containers:

- `ragInit`
- `lightspeedCore`

Default:

```yaml
global:
  lightspeed:
    resources:
      ragInit: {}
      lightspeedCore: {}
```

Example:

```yaml
global:
  lightspeed:
    resources:
      ragInit:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          memory: 512Mi
      lightspeedCore:
        requests:
          cpu: 250m
          memory: 512Mi
        limits:
          memory: 1Gi
```

### `global.lightspeed.runtimeVolume`

Controls the writable runtime storage mounted at `/tmp` inside the Lightspeed containers.

Supported fields:

- `type`
  - `emptyDir`
  - `persistentVolumeClaim`
- `emptyDir`
- `persistentVolumeClaim.claimName`
- `persistentVolumeClaim.readOnly`

Default:

```yaml
global:
  lightspeed:
    runtimeVolume:
      type: emptyDir
      emptyDir: {}
      persistentVolumeClaim: {}
```

Important:

- If `type: persistentVolumeClaim`, then `persistentVolumeClaim.claimName` must be set
- The chart validates this during rendering

Example using a PVC:

```yaml
global:
  lightspeed:
    runtimeVolume:
      type: persistentVolumeClaim
      persistentVolumeClaim:
        claimName: lightspeed-runtime
        readOnly: false
```

## Chart-managed internal pieces

The chart also manages several Lightspeed-specific defaults internally. These are not currently part of the documented public values API.

That includes:

- The chart-managed RAG volume mounted at `/rag-content`
- The exact generated ConfigMap list and mount paths
- The exact generated Secret name/create policy
- The default init container command
- The sidecar name, port, and security context

Today those internals live in `charts/backstage/templates/_helpers.tpl` so the chart can render upgrade-safe defaults even when an older release has no `global.lightspeed` block yet.

### Generated ConfigMaps

When Lightspeed is enabled, the chart renders three ConfigMaps from vendored files under `charts/backstage/vendor/backstage/charts/backstage/files/lightspeed`:

- `lightspeed-stack.yaml`
- `config.yaml`
- `rhdh-profile.py`

They are mounted read-only into the Lightspeed sidecar.

### Generated Secret

When Lightspeed is enabled, the chart renders a Secret from the vendored file `charts/backstage/vendor/backstage/charts/backstage/files/lightspeed/secret.yaml`.

The Lightspeed Core sidecar consumes that Secret through `envFrom`, so those keys become container environment variables.

The placeholder keys currently include:

- `ENABLE_VLLM`
- `VLLM_URL`
- `VLLM_API_KEY`
- `VLLM_MAX_TOKENS`
- `VLLM_TLS_VERIFY`
- `ENABLE_OLLAMA`
- `OLLAMA_URL`
- `ENABLE_OPENAI`
- `OPENAI_API_KEY`
- `ENABLE_VERTEX_AI`
- `VERTEX_AI_PROJECT`
- `VERTEX_AI_LOCATION`
- `ENABLE_VALIDATION`
- `VALIDATION_PROVIDER`
- `VALIDATION_MODEL_NAME`

At the moment, those keys are chart-managed rather than exposed as supported Helm values.

Practical implication:

- If you need to change the set of placeholder keys or the default payload in the repo, update the vendored file
- If you patch the generated Secret directly in the cluster for ad hoc testing, restart the Backstage Deployment afterward
- Changes made directly to the cluster Secret are outside Helm values management and may be overwritten by later upgrades

## Common ways to set Lightspeed values

The most common pattern is to put your overrides in a values file and pass it to Helm:

```bash
helm upgrade --install <release> charts/backstage -f my-values.yaml
```

You can also combine multiple files:

```bash
helm upgrade --install <release> charts/backstage -f my-values.yaml -f another-override.yaml
```

Or use simple CLI overrides for scalar values:

```bash
helm upgrade --install <release> charts/backstage \
  --set global.lightspeed.enabled=false
```

For anything nested or list-shaped, prefer a values file over `--set`.

## Maintainer note: vendored Lightspeed config sync

The vendored Lightspeed config payloads are synced separately from the Backstage subtree by:

```bash
./hack/sync-lightspeed-configs.sh
```

Examples:

```bash
./hack/sync-lightspeed-configs.sh
./hack/sync-lightspeed-configs.sh --ref release-1.9
./hack/sync-lightspeed-configs.sh --check
```

That script updates:

- `lightspeed-stack.yaml`
- `config.yaml`
- `rhdh-profile.py`

The secret payload is still maintained locally in this repo.

## Testing-only: guest access and RBAC override

This section describes the testing-only override used to let a guest user exercise Lightspeed chat permissions without having to set up a full identity provider and RBAC source.

Use:

- `charts/backstage/ci/with-lightspeed-guest-rbac-values.yaml`

### What the override does

The file makes the following changes:

1. Creates an additional ConfigMap containing a tiny RBAC policy file
2. Enables the Backstage guest auth provider
3. Forces guest auth to work outside development-only defaults by setting `dangerouslyAllowOutsideDevelopment: true`
4. Enables the Backstage permission framework and file-based RBAC policy loading
5. Mounts the RBAC policy ConfigMap into the Backstage container at `/opt/app-root/src/rbac`
6. Grants the guest user the minimal Lightspeed chat permissions used during testing

The policy entries in that file are:

```text
p, role:default/team_a, lightspeed.chat.read, read, allow
p, role:default/team_a, lightspeed.chat.create, create, allow
p, role:default/team_a, lightspeed.chat.delete, delete, allow
g, user:default/guest, role:default/team_a
```

### Create the file locally

If the file is not already present in your checkout, create it from the repository root with:

```bash
mkdir -p charts/backstage/ci

cat > charts/backstage/ci/with-lightspeed-guest-rbac-values.yaml <<'EOF'
# Temporary override for testing the Lightspeed plugin with guest auth and RBAC.
# Stop passing this file to Helm when testing is complete.
upstream:
  extraDeploy:
    - |
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: {{ printf "%s-lightspeed-guest-rbac" .Release.Name }}
      data:
        rbac-policies.csv: |
          p, role:default/team_a, lightspeed.chat.read, read, allow
          p, role:default/team_a, lightspeed.chat.create, create, allow
          p, role:default/team_a, lightspeed.chat.delete, delete, allow
          g, user:default/guest, role:default/team_a
  backstage:
    appConfig:
      auth:
        environment: development
        providers:
          guest:
            userEntityRef: user:default/guest
            dangerouslyAllowOutsideDevelopment: true
      permission:
        enabled: true
        rbac:
          policies-csv-file: /opt/app-root/src/rbac/rbac-policies.csv
          policyFileReload: true
    # These lists intentionally mirror the chart defaults and add one RBAC test mount.
    extraVolumeMounts:
      - name: dynamic-plugins-root
        mountPath: /opt/app-root/src/dynamic-plugins-root
      - name: extensions-catalog
        mountPath: /extensions
      - name: temp
        mountPath: /tmp
      - name: lightspeed-guest-rbac
        mountPath: /opt/app-root/src/rbac
        readOnly: true
    extraVolumes:
      - name: dynamic-plugins-root
        ephemeral:
          volumeClaimTemplate:
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 5Gi
      - name: dynamic-plugins
        configMap:
          defaultMode: 420
          name: '{{ printf "%s-dynamic-plugins" .Release.Name }}'
          optional: true
      - name: dynamic-plugins-npmrc
        secret:
          defaultMode: 420
          optional: true
          secretName: '{{ printf "%s-dynamic-plugins-npmrc" .Release.Name }}'
      - name: dynamic-plugins-registry-auth
        secret:
          defaultMode: 416
          optional: true
          secretName: '{{ printf "%s-dynamic-plugins-registry-auth" .Release.Name }}'
      - name: npmcacache
        emptyDir: {}
      - name: extensions-catalog
        emptyDir: {}
      - name: temp
        emptyDir: {}
      - name: lightspeed-guest-rbac
        configMap:
          defaultMode: 420
          name: '{{ printf "%s-lightspeed-guest-rbac" .Release.Name }}'
EOF
```

### How to use it

Pass the override file during install or upgrade:

```bash
helm upgrade --install <release> charts/backstage \
  -f my-values.yaml \
  -f charts/backstage/ci/with-lightspeed-guest-rbac-values.yaml
```

If you do not already have a separate values file, you can test with just the guest override:

```bash
helm upgrade --install <release> charts/backstage \
  -f charts/backstage/ci/with-lightspeed-guest-rbac-values.yaml
```

After deployment:

- open Backstage
- sign in with the guest provider
- exercise the Lightspeed UI as the guest user

### How to remove it

Stop passing the override file to Helm and run another upgrade:

```bash
helm upgrade --install <release> charts/backstage -f my-values.yaml
```

### Important caveats

- This is for testing only
- Do not use this guest-auth setup in production
- The file intentionally mirrors the chart's default `extraVolumes` and `extraVolumeMounts` lists, then adds one extra RBAC mount
- Because Helm replaces lists rather than merging them item-by-item, this file may need to be refreshed if the chart defaults for those lists change in the future
- If your release already customizes `upstream.backstage.extraVolumes` or `upstream.backstage.extraVolumeMounts`, merge those changes carefully instead of blindly stacking this file on top
