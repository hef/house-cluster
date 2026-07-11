# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Overview

This is a GitOps home Kubernetes cluster configuration. A single-node Talos Linux cluster (node `k8s-4` at `192.168.1.22`, VIP `192.168.1.200`) is managed via FluxCD. All cluster state is declared here and reconciled automatically.

## Common Commands

```bash
# List all available tasks
task

# Generate Talos machine configs from talconfig.yaml
task talos:genconfig

# Apply updated Talos config to the node (with reboot)
task talos:apply

# Upgrade Talos on the node
task talos:upgrade

# Bootstrap a fresh cluster (full sequence: generate config → apply → bootstrap etcd → get kubeconfig → install prometheus-operator CRDs → helmfile (installs Flux) → apply SOPS secret → install external-dns/gateway-api CRDs)
task bootstrap:talos

# Reset the cluster back to maintenance mode
task talos:reset

# Print the postgres superuser password for house-db
task cnpg:password

# Connect to house-db via psql as the postgres superuser
task cnpg:psql

# Connect to the house-db app database via psql as the grafana user
task cnpg:psql-grafana

# Update the Grafana admin password in both the SOPS secret and the live cluster
task grafana:update-password

# Remove stale restic locks from volsync backup repositories
task volsync:unlock

# Delete all volsync cache PVCs so they get recreated with correct ownership
task volsync:delete-cache-pvcs
```

## Architecture

### Bootstrap vs. Runtime

There are two layers of cluster setup:

1. **`helmfile.yaml` (bootstrap-only)**: Installs the minimum set of charts needed before Flux exists — Cilium, CoreDNS, Spegel, cert-manager, flux-operator, flux-instance — in strict dependency order. This is only run once during `task bootstrap:talos`.

2. **FluxCD (runtime)**: Once Flux is running, it watches this repo and reconciles everything under `apps/`. The entry point is `flux/cluster/ks.yaml`, which creates a Flux `Kustomization` pointing at `./apps`.

### Directory Layout

- `apps/<namespace>/`: One directory per Kubernetes namespace. Each app inside follows the pattern:
  - `ks.yaml` — Flux `Kustomization` CRD that tells Flux what to deploy and where
  - `app/` — The actual Kubernetes manifests (HelmRelease, Deployment, PVC, secrets, etc.)
- `components/common/` — Kustomize Component included in every namespace-level `kustomization.yaml`. Provides shared cluster secrets, SOPS decryption config, and namespace creation.
- `components/volsync/` — Kustomize Component included in any app that needs PVC backups. Adds a Restic-backed `ReplicationSource` (runs every 4h) and a `ReplicationDestination` for restore.
- `patches/` — Talos machine config patches applied via `talconfig.yaml`.
- `clusterconfig/` — Generated Talos configs (git-ignored except the talosconfig file).
- `helmfile.yaml` — Bootstrap-only helm chart installs.
- `talconfig.yaml` — Node definitions and schematic for `talhelper`.
- `.taskfiles/` — Task definitions grouped by domain (bootstrap, cnpg, grafana, talos, volsync), included from the root `Taskfile.yaml`.

### How a New App Is Added

1. Create `apps/<namespace>/<appname>/ks.yaml` — the Flux Kustomization pointing at `./apps/<namespace>/<appname>/app`.
2. Create `apps/<namespace>/<appname>/app/kustomization.yaml` listing all resources.
3. Add the `ks.yaml` to the namespace-level `apps/<namespace>/kustomization.yaml`.
4. If the app needs PVC backups, add `components/volsync` to the Flux `Kustomization` in `ks.yaml` and set `CLAIM`/`CAPACITY` substitution vars.

### Secrets

Secrets are encrypted with SOPS + age. The `.sops.yaml` rules:
- `talsecret.sops.yaml` — fully encrypted
- `apps/**/*.sops.yaml` and `components/**/*.sops.yaml` — only `data` and `stringData` fields are encrypted

Flux decrypts secrets at runtime using the `sops-age` secret in `flux-system`. To edit an encrypted file: `sops <file>`.

Flux variable substitution (`postBuild.substituteFrom`) pulls values from the `cluster-secrets` Secret (provided by `components/common`) into manifests using `${VARIABLE_NAME}` syntax.

### Networking

- **CNI**: Cilium in gateway-api mode
- **Ingress**: Gateway API with two Gateways — `external` (internet-facing via Cloudflare tunnel, IP `192.168.1.3`) and `internal` (LAN only)
- **DNS**: `cloudflare-dns` (external-dns managing Cloudflare) and `unifi-dns` (internal DNS via Unifi controller)
- **Multi-homed pods**: Multus with a `house` NetworkAttachmentDefinition for pods that need direct LAN access (e.g. zigbee, hardware)

### Storage

- **OpenEBS hostpath** (`host-zfs-standard` StorageClass): Primary PVC storage backed by a ZFS pool named `zfspv-pool` on the NVMe drive. The cluster requires this ZFS pool to exist.
- **Volsync**: PVC backups via Restic to an external repository. Apps opt in by including `components/volsync` and providing a `${CLAIM}-backup-config` Secret.
- **Snapshot**: `host-zfs-snapshot` VolumeSnapshotClass used by Volsync for point-in-time copies.
- **Volsync maintenance**: Use `task volsync:unlock` to clear stale restic locks and `task volsync:delete-cache-pvcs` to recreate cache PVCs with correct ownership.

### CI

On pull requests touching `apps/`, `components/`, or `flux/`, GitHub Actions runs `flux-local`:
- **test**: Validates all Flux Kustomizations and HelmReleases render correctly
- **diff**: Posts a comment on the PR showing the rendered manifest diff vs. main

## Key Tools Required

- `talhelper` — generates Talos machine configs from `talconfig.yaml`
- `task` — task runner
- `sops` — decrypt/edit secrets
- `helmfile` — bootstrap helm installs
- `kubectl` — cluster interaction
- `flux` — Flux CLI for troubleshooting reconciliation
