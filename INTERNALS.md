# Internals

Deep technical details on how cluster-tool handles snapshot recloning, OVN cleanup, etcd certificate management, and the forked recert image.

## Snapshot Recloning

A snapshot can be booted as a clone, and that clone can be snapshotted again, creating a chain of arbitrary depth (clone-of-clone-of-clone...). Making this work requires handling two subsystems that carry stale state from the previous identity: OVN networking and etcd certificate revisions.

### OVN Networking Cleanup (boot time)

OVN-Kubernetes stores the node's tunnel endpoint IP, connection strings, and system identity in several databases on disk. When a clone boots with a new IP, these stale values prevent OVN from starting — the node stays `NotReady` with "no CNI configuration file."

During boot (after recert, before starting kubelet), the node fix script deletes:
- `/etc/openvswitch/conf.db` and its lock file — the OVS configuration database with the old IP in `ovn-encap-ip`
- `/etc/ovn/ovnsb_db.db` and `ovnnb_db.db` — OVN southbound/northbound databases
- `/var/lib/ovn-ic/etc/` — OVN interconnect state including stale certificates

Then it restarts the `openvswitch` service. On the next boot, `configure-ovs.sh` recreates `br-ex` from scratch, and `ovnkube-node` rebuilds its state with the new IP. The node reaches `Ready` in ~60 seconds.

This follows the same pattern used by the [lifecycle-agent](https://github.com/openshift/lifecycle-agent) (LCA) during Image-Based Install.

### etcd Certificate Revision Hardlinks (snapshot time)

OpenShift's etcd operator uses a revision system for safe rollouts. It keeps two copies of the TLS certificates on disk:

- `etcd-certs/secrets/etcd-all-certs/` — the live certs mounted by the running etcd pod
- `etcd-pod-{rev}/secrets/etcd-all-certs/` — a revision snapshot from the last successful rollout

The operator periodically compares the live certificates (via the Kubernetes API) against the revision snapshot. If they differ, it triggers a full revision rollout — redeploying the etcd static pod, which cascades to kube-apiserver and other components. This takes ~7 minutes.

**The problem with recloning:** recert regenerates all certificates by scanning the filesystem (`--crypto-dir`) and the etcd database (`--etcd-endpoint`) independently. It deduplicates certificates by content — if two files have identical bytes, recert treats them as one certificate and writes the same regenerated output to both. If the bytes differ, it treats them as separate certificates and generates independent replacements with different serial numbers.

On a first-generation clone (from the original installer), the live certs and revision snapshot are byte-identical. Recert deduplicates them and writes matching output. No rollout.

On a second-generation clone (clone-of-clone), the previous recert already wrote different bytes to each location (because it treated them as separate certs — the same problem, recursively). Now recert sees two different certificates and generates two different replacements. The operator detects the mismatch and triggers a 7-minute rollout.

**The fix:** during `snapshot`, before shutting down the VM, the tool replaces the cert files in `etcd-certs/secrets/etcd-all-certs/` with **hardlinks** to the corresponding files in `etcd-pod-{rev}/secrets/etcd-all-certs/`. Hardlinks are filesystem entries that point to the same physical data on disk (same inode). When recert scans both paths, it finds identical bytes (same inode = same data), deduplicates them into one certificate, and writes one regenerated output. Both paths see the same result. The operator finds no mismatch. No rollout.

Hardlinks (not symlinks) are required because the etcd pod mounts `etcd-certs/` as a container volume. A symlink pointing to `../../etcd-pod-24/` would resolve outside the mount boundary inside the container — the target wouldn't exist. Hardlinks have no path to resolve; they are direct inode references that work identically inside and outside containers.

The hardlinks survive `qemu-img convert` (block-level copy preserves filesystem metadata), qcow2 COW overlays (both directory entries reference the same inode through the overlay), and recert's write mechanism (`std::fs::write` uses `O_TRUNC` which modifies the existing inode in-place without breaking the link).

The etcd operator's cert-sync controller could theoretically break hardlinks via `renameat2(RENAME_EXCHANGE)`, but it has a content-equality check that skips the swap when disk content matches the API. Since recert ensures consistency, the check passes and the hardlinks survive.

This step also removes any stale `etcd-pod.yaml` from `etcd-certs/` — a file the etcd operator's installer pod creates during revision rollouts that would cause recert to crash on subsequent boots.

### Standalone etcd for Recert (boot time)

Recert needs a live etcd endpoint to read and write certificate data. During boot, after stopping kubelet and crio, the tool starts a standalone etcd container:

```
podman run -d --name etcd-recert \
  --network host --privileged \
  -v /var/lib/etcd:/store \
  --entrypoint etcd \
  <etcd-image> \
  --name editor --data-dir /store
```

The volume mount uses a different container path (`/store` instead of `/var/lib/etcd`) following the lifecycle-agent convention. The `--name editor` flag is ignored by etcd when existing WAL data is present — etcd loads its identity from the WAL metadata. No `--force-new-cluster` flag is used; that flag is designed for multi-member disaster recovery and is unnecessary on a single-node cluster.

### Forked Recert Image

We use `quay.io/osac-project/recert:latest` with two fixes over upstream:

1. **Binary DER data handling** — upstream crashes on secrets with binary certificate data. The fork returns `None` for non-UTF-8 data instead of erroring.
2. **PersistentVolume node affinity** — upstream doesn't update PV node affinity hostnames during `--hostname`. The fork adds PV support in etcd encoding and hostname replacement.

## Compact clone follow-ups (edge1)

Seen on `ocp-edge126` after `compact-boot` of flavor `mno-3node` as clone `edge1` (subnet 200, domain `test-infra-cluster-edge1.redhat.com`). Recert runs once, on `master-0-0`. The API address load-balances across `master-0-0` (`.69`), `master-0-1` (`.81`), and `master-0-2` (`.89`). These were fixed by hand on that cluster so OSAC could install. They still need to land in automation and in the flavor image.

### API credentials — cluster-tool

Recert rewrites trust files on `master-0-0` only. The other two masters keep the originals. About one request in three hits `master-0-0` and the API answers `the server has asked for the client to provide credentials`.

Copy these from `master-0-1` onto `master-0-0`. Do not copy Recert's new material the other way.

Client CA, so the admin kubeconfig (`system:admin`) is accepted:

- `kube-apiserver-certs/configmaps/client-ca/ca-bundle.crt`
- `kube-apiserver-certs/configmaps/aggregator-client-ca/ca-bundle.crt`
- the same two bundles under `kube-controller-manager-certs/configmaps/`

Service-account signing keys, so in-cluster tokens (Helm hooks, operator jobs) are accepted. The API server reads the public keys at start and does not reload them:

- `kube-apiserver-certs/secrets/bound-service-account-signing-key/service-account.key`
- `kube-apiserver-certs/secrets/bound-service-account-signing-key/service-account.pub`
- `kube-apiserver-pod-*/configmaps/bound-sa-token-signing-certs/service-account-*.pub`
- `kube-apiserver-pod-*/configmaps/sa-token-signing-certs/service-account-*.pub`

Paths are under `/etc/kubernetes/static-pod-resources/`. After the copy, `restorecon` that tree and restart the kube-apiserver sandbox on `master-0-0` (`crictl stopp`). The client CA is picked up without a restart; the service-account keys are not.

The health gate has to ask each node, not only the API address. `/version` and `/readyz` answer without credentials, so a VIP check can pass while Helm gets a 401. Before the run finishes, `oc whoami` with the admin kubeconfig must succeed against `https://<node>:6443` on all three, and one short-lived service-account token must get HTTP 200 from each node's `/readyz`.

The compact heal in this tree copies those files, restarts that API server, and refuses to finish while any node rejects either check. That path has not been booted end to end, and it is not in the flavor image.

### Ingress domain — cluster-tool

DNS `spec.baseDomain` was rewritten to `test-infra-cluster-edge1.redhat.com`. `ingresses.config.openshift.io/cluster` `spec.domain` stayed `apps.test-infra-cluster-mno.redhat.com`. osac-installer reads the ingress domain and publishes Keycloak at `keycloak-keycloak.apps.test-infra-cluster-mno.redhat.com`. In-cluster clients use `keycloak.keycloak.svc.cluster.local`, so the install can proceed, but the route name is still the source cluster. Rewrite `spec.domain` to `apps.test-infra-cluster-<clone>.redhat.com` with the rest of the identity.

### Keycloak storage — image or installer

The booted cluster has no StorageClass. Each master disk is one OS disk (`vda`, xfs on `vda4`), so there is no volume group for LVMS.

osac-installer `charts/osac-prereqs` creates PVC `keycloak/keycloak-database`: 2Gi, `ReadWriteOnce`, no `storageClassName`. The Keycloak wait hook (`oc wait deploy/keycloak-service -n keycloak`) then hits its deadline. The database pod stays Pending with `pod has unbound immediate PersistentVolumeClaims`, and the Keycloak pod sits in `Init:0/2`.

Hand fix on edge1, tied to one node, not the image fix:

- directory `/var/lib/keycloak-database` on `master-0-1`
- PV `keycloak-database`, 2Gi, RWO, `storageClassName: ""`, hostPath of that directory, `nodeAffinity` `kubernetes.io/hostname=master-0-1`, reclaim `Retain`
- with no default StorageClass, the chart's claim (also no class) bound to that PV

The postgres container's fsGroup was `1000810000` (the namespace SCC range), not uid 26. `chown 26:26` and mode `770` still failed with `mkdir: cannot create directory '/var/lib/pgsql/data': Permission denied`. `chmod 777` and `chcon -t container_file_t` let the entrypoint create the data directory. The pod had to be deleted so it remounted the volume. After Postgres logged `accepting connections`, `keycloak-service` became Available.

For the image, ship a default StorageClass (a spare disk plus LVMS), or stop requiring dynamic provisioning for the Keycloak database. A hostPath volume does not belong in the flavor.

The OSAC chart's pre-install validation only warns when no default StorageClass exists. The operator subchart still runs a pre-install job, `osac-operator-label-sc`, when `controllers.storage` is true (the chart default; the edge1 values do not turn it off). That job exits 1 with `ERROR: No default StorageClass found` and Helm reports `BackoffLimitExceeded`. It then labels the default class `osac.openshift.io/tenant=Default` and `osac.openshift.io/storage-tier=default`.

Hand fix on edge1: StorageClass `local-manual`, provisioner `kubernetes.io/no-provisioner`, `volumeBindingMode: WaitForFirstConsumer`, annotated `storageclass.kubernetes.io/is-default-class=true`. That lets the label job succeed. It does not create volumes. A PVC that omits `storageClassName` is now assigned `local-manual` by the default-class admission plugin and stays Pending until a PersistentVolume with that class exists. The image still needs a provisioner.

Phase 3 then waits up to 40 minutes and ends with `context deadline exceeded` while these `osac` pods are not Ready:

- `osac-aap-postgres-15-0` stays Pending. PVC `postgres-15-osac-aap-postgres-15-0` is 8Gi, `ReadWriteOnce`, class `local-manual`. Nothing creates a volume for that class.
- `fulfillment-grpc-server` crash-loops with `invalid vault configuration: flag '--vault-endpoint' is required`. The chart command does not pass that flag. The values pin `ghcr.io/osac-project/fulfillment-service:latest`, which is ahead of this installer checkout. `osac-ui` then exits with `no healthy upstream`.
- `osac-operator` crash-loops with `invalid fulfillment issuer URL`. Its deployment sets the fulfillment address and token file and does not set an issuer URL.
- `bmf-operator` stays ContainerCreating because Secrets `osac-inventory-config` and `osac-management-config` are absent.

### Fulfillment PostgreSQL — image or installer values

`make install` phase 3 runs `check-postgres` before the OSAC chart. With `bundledPostgres.enabled` unset, that check requires Secret `fulfillment-db` and Secret `postgres-client-cert-service` in the `osac` namespace, plus a Service with ready endpoints. Those come from a PostgreSQL operator (CloudNativePG, Crunchy, or Zalando), and `base/osac-fulfillment-service/docs/INSTALL.md` says that path needs dynamic persistent volumes.

This cluster has no StorageClass, so that operator path cannot create its volume. Set `bundledPostgres.enabled: true` in the values file. The chart then creates Postgres itself with `emptyDir` and skips the secret check. The pre-install hook only warns when no default StorageClass exists. The database contents disappear when that pod is rescheduled; a durable database still needs a StorageClass in the image.

### Kubelet serving CA — cluster-tool, if log collection matters

`oc logs` for pods on `master-0-1` failed TLS verification of the kubelet serving certificate. The signer was `kube-csr-signer_@1791570079`, which the client did not trust. Same shape as the API trust split: Recert rotated a CA on one node. The Keycloak install did not need this once the volume existed.
