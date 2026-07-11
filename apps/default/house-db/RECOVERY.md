# house-db recovery runbook

Disaster-recovery / point-in-time-recovery for the `house-db` CloudNativePG cluster.

Backups are written to Backblaze B2 (`s3://hef-restic/cnpg/house-db`) by the
**Barman Cloud Plugin** (`barman-cloud.cloudnative-pg.io`). The plugin replaced
the deprecated in-tree `spec.backup.barmanObjectStore` integration, so recovery
now goes through an `ObjectStore` CR + an `externalClusters[].plugin` reference —
**not** the old `externalClusters[].barmanObjectStore` form.

## Prerequisites

Before a restore can succeed, these must already be reconciled in the cluster:

1. The `cnpg` operator (`apps/cnpg-system/cnpg`).
2. The `barman-cloud` plugin (`apps/cnpg-system/barman-cloud`) — provides the
   `barmancloud.cnpg.io/v1 ObjectStore` CRD and the plugin sidecar.
3. The `house-db-backup-creds` Secret (B2 `ACCESS_KEY_ID` / `ACCESS_SECRET_KEY`)
   in the target namespace — shipped via `apps/default/house-db/app/secret.sops.yaml`.
4. An `ObjectStore` named `house-db` pointing at the backup bucket
   (`apps/default/house-db/app/objectstore.yaml`).

The recovery source `ObjectStore` can reuse the existing `house-db` `ObjectStore`
as long as its `destinationPath` still points at the backups you want to restore.

## Restore into a NEW cluster (recommended)

Restoring into a *new* Cluster object (leaving backups intact) is the safest
path. Point a bootstrap `recovery` block at an `externalCluster` that references
the plugin.

```yaml
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: house-db-restore
spec:
  instances: 3
  storage:
    size: 10Gi
    storageClass: host-zfs-postgres
  # The plugin must be declared so the restored cluster can read WAL / resume archiving.
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      isWALArchiver: true
      parameters:
        barmanObjectName: house-db        # ObjectStore used for ongoing archiving
  bootstrap:
    recovery:
      source: house-db-backup             # name of the externalCluster below
      # Optional PITR — omit the whole recoveryTarget block to restore to the latest WAL:
      # recoveryTarget:
      #   targetTime: "2026-07-11 12:00:00+00"
  externalClusters:
    - name: house-db-backup
      plugin:
        name: barman-cloud.cloudnative-pg.io
        parameters:
          barmanObjectName: house-db      # ObjectStore that holds the backups to restore FROM
          serverName: house-db            # original cluster name inside the bucket path
```

Notes:

- `serverName` must match the folder the backups were written under
  (`s3://hef-restic/cnpg/house-db/<serverName>`). For an in-place restore of the
  same cluster this is just `house-db`.
- If the recovery source bucket differs from where the new cluster should
  *archive* to, create a second `ObjectStore` for the source and reference it via
  the externalCluster's `barmanObjectName` while keeping a different
  `ObjectStore` in `spec.plugins`.
- Do **not** use `externalClusters[].barmanObjectStore:` — that is the removed
  in-tree form and will not work with the plugin.

## Point-in-time recovery (PITR)

Add a `recoveryTarget` under `bootstrap.recovery`:

```yaml
  bootstrap:
    recovery:
      source: house-db-backup
      recoveryTarget:
        targetTime: "2026-07-11 12:00:00+00"   # RFC3339 / Postgres timestamp
```

Other targets are supported: `targetLSN`, `targetXID`, `targetName` (restore
point). Only set one.

## In-place restore (rebuild house-db itself)

1. Delete the broken `Cluster` (this does **not** delete the B2 backups):
   `kubectl delete cluster house-db -n default`.
   Also remove its PVCs if you want fresh storage:
   `kubectl delete pvc -l cnpg.io/cluster=house-db -n default`.
2. Temporarily edit `cluster.yaml` to add the `bootstrap.recovery` +
   `externalClusters` blocks shown above (keep `metadata.name: house-db`), commit
   and let Flux reconcile — or `kubectl apply` for an out-of-band recovery.
3. Once the cluster reports `phase=Cluster in healthy state` and
   `ContinuousArchiving=True`, **revert** the `bootstrap` change back to the
   normal `initdb` bootstrap in git so future reconciles don't attempt to
   re-bootstrap. CNPG ignores `bootstrap` after the cluster exists, but keeping
   git clean avoids surprises on a full rebuild.

## Verify after restore

```bash
export KUBECONFIG=./kubeconfig
# cluster healthy + primary elected
kubectl get cluster house-db-restore -n default \
  -o jsonpath='ready={.status.readyInstances}/{.status.instances} phase={.status.phase} primary={.status.currentPrimary}{"\n"}'
# plugin loaded and archiving
kubectl get cluster house-db-restore -n default \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {end}{"\n"}'
# take a fresh plugin backup to confirm write path
kubectl cnpg backup house-db-restore -n default --method plugin
```

## Full-cluster bootstrap ordering

On a from-scratch `task bootstrap:talos`, Flux brings these up in dependency
order automatically: `cnpg` → `barman-cloud` → `house-db` (see the `dependsOn`
in `apps/default/house-db/ks.yaml`). A recovery `Cluster` therefore only works
once `barman-cloud` is `Ready`.
