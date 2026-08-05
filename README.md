# OpenShift etcd Restore

![Ansible](https://img.shields.io/badge/Ansible-2.15%2B-EE0000?logo=ansible&logoColor=white)
![OpenShift](https://img.shields.io/badge/OpenShift-4.x-red?logo=redhatopenshift&logoColor=white)
![Kasten Kanister](https://img.shields.io/badge/Kasten-Kanister-00AEEF)
![License](https://img.shields.io/badge/License-MIT-blue.svg)

A small Ansible wrapper around the Veeam Kasten Kanister OpenShift etcd restore flow:

<https://docs.kasten.io/latest/kanister/etcd/ocp/install>

> [!CAUTION]
> This is disaster-recovery automation. It stops static control-plane pods, moves etcd data on non-restore control-plane nodes, runs the bundled OpenShift `cluster-restore.sh`, handles CSRs, and forces control-plane redeployments. Do not run it casually.

## Out-of-box reality

This repository is not a one-command restore for a random cluster. It is an
automation wrapper around the documented Veeam Kasten/OpenShift flow. It expects
the cluster and inventory to be prepared exactly:

- one, and only one, control-plane node is labeled `etcd-restore=true`
- that same node has `node-role.kubernetes.io/control-plane`
- the inventory `control_plane` group maps SSH hosts to OpenShift node names
- the Kasten etcd Blueprint, policy, restore point, and restore namespace exist
- the operator has validated that this is the correct recovery host and backup

The playbook now creates the restore host path, verifies the pre-existing
PV/PVC, waits longer for static pods and etcd recovery, and waits for stable
ClusterOperators after forced redeployments. Human judgement is still required
for Kasten restore point selection, CSR validation, and lost-machine
replacement.

## Shape

One playbook:

```text
playbooks/restore.yml
```

Four roles:

```text
roles/check_prerequisites   # prepare/check Kasten restore target and backup file
roles/execute_restore       # destructive host restore up to CSR approval
roles/approve_csrs          # auto-approve CSRs or print manual commands and stop
roles/finish_restore        # force remaining control-plane redeployments and print final checks
```

The restore node is selected from the OpenShift node label
`etcd-restore=true`. The inventory only maps SSH hosts to OpenShift node names:

```yaml
all:
  children:
    control_plane:
      hosts:
        master-0.example.com:
          ansible_host: 10.0.0.10
          openshift_node_name: master-0.example.com
        master-1.example.com:
          ansible_host: 10.0.0.11
          openshift_node_name: master-1.example.com
        master-2.example.com:
          ansible_host: 10.0.0.12
          openshift_node_name: master-2.example.com
```

Exactly one control-plane node must have the label:

```bash
oc label node master-0.example.com etcd-restore=true --overwrite
```

If zero or multiple nodes have the label, the playbook stops before restore
work. The labeled node must also carry the OpenShift 4.12+ control-plane role
label `node-role.kubernetes.io/control-plane`. If the SSH inventory hostname
differs from the OpenShift node name, set `openshift_node_name` on that
inventory host.

## Prerequisites

Workstation:

- `ansible-playbook`
- `oc` authenticated as cluster-admin
- SSH access with passwordless sudo to all control-plane nodes
- the bundled OpenShift `cluster-restore.sh` from `roles/execute_restore/files/`
  - source: <https://github.com/openshift/cluster-etcd-operator/blob/main/bindata/etcd/cluster-restore.sh>

Control-plane hosts:

- `crictl` available in the standard command path
- standard OpenShift paths/services available: `/etc/kubernetes/manifests`,
  `/var/lib/etcd`, and `kubelet.service`

Cluster/Kasten:

- Kasten installed, default namespace `kasten-io`
- OpenShift etcd pods in `openshift-etcd`
- Kasten Kanister etcd Blueprint applied
- a successful etcd restore point
- exactly one control-plane node labeled `etcd-restore=true`

## Configure

Copy and edit the example inventory:

```bash
cp -R inventories/example inventories/prod
vi inventories/prod/hosts.yml
vi inventories/prod/group_vars/all.yml
```

Important variables in `inventories/prod/group_vars/all.yml`:

```yaml
i_understand_this_is_destructive: false
auto_approve_csrs: false
kasten_namespace: kasten-io
etcd_namespace: openshift-etcd
etcd_restore_namespace: etcd-restore
restore_require_kasten_download_prerequisites: true
restore_snapshot_file: etcd-backup.db
restore_pv_name: pv-etcd
restore_pvc_name: pvc-etcd
restore_use_etcdctl_restore: false
```

## Variables and defaults

Inventory variables live in `inventories/<name>/group_vars/all.yml`. Role
defaults live under `roles/*/defaults/main.yml`; override them in inventory or
with `-e` only when your restore shape differs from the default Kasten flow.

| Variable | Default | Change when |
|---|---:|---|
| `i_understand_this_is_destructive` | `false` | Set `true` only for the actual destructive restore run. Leave `false` for syntax checks, inventory validation, and reviews. |
| `auto_approve_csrs` | `false` | Set `true` in controlled lab/test clusters when you want the playbook to approve pending CSRs automatically after restore. Leave `false` in production unless an operator validates each CSR manually. |
| `kasten_namespace` | `kasten-io` | Change only if Kasten is installed in a different namespace. Used when Kasten prerequisite checks are enabled. |
| `etcd_namespace` | `openshift-etcd` | Normally do not change. Change only for non-standard OpenShift layouts. |
| `etcd_restore_namespace` | `etcd-restore` | Change if your Kasten Kanister restore namespace uses another name. Used when Kasten prerequisite checks are enabled. |
| `restore_require_kasten_download_prerequisites` | `true` | Keep `true` for the normal Kasten/Kanister workflow so the playbook verifies Kasten namespace, restore namespace, PV, and PVC. Set `false` when the backup is already staged on `restore_host_path` outside of Kasten, for example a CRC/SNO lab where you copied the backup to `/mnt/data` yourself. |
| `restore_snapshot_file` | `etcd-backup.db` | Set to the exact snapshot filename present under `restore_host_path` on the selected restore node, for example `snapshot_2026-08-05_084046.db`. |
| `restore_pv_name` | `pv-etcd` | Change if your Kasten restore PV has another name. Used when Kasten prerequisite checks are enabled. |
| `restore_pvc_name` | `pvc-etcd` | Change if your Kasten restore PVC has another name. Used when Kasten prerequisite checks are enabled. |
| `restore_node_label_key` | `etcd-restore` | Rarely change. Must match the label key on exactly one control-plane OpenShift node. |
| `restore_node_label_value` | `"true"` | Rarely change. Must match the label value on exactly one control-plane OpenShift node. |
| `restore_host_path` | `/mnt/data` | Change only when your Kasten PV or pre-staged backup uses another host path on the restore node. |
| `restore_script_path` | `/usr/local/bin/cluster-restore.sh` | Rarely change. Remote path where the bundled restore script is copied and run. |
| `restore_use_etcdctl_restore` | `false` | Set `true` for single-node OpenShift, CRC, or SNO restores so `cluster-restore.sh` runs the documented `ETCD_ETCDCTL_RESTORE=1` path. Leave `false` for standard multi-node restore-pod flow. |
| `restore_wait_retries` | `90` | Increase for slow or resource-constrained clusters. With default delay this is 15 minutes. |
| `restore_wait_delay` | `10` | Increase/decrease polling interval in seconds. |
| `finish_restore_force_redeployment` | `true` | Keep `true` for a real restore so control-plane components redeploy. Set `false` only for idempotent finish-role validation on an already-restored cluster. |
| `finish_restore_wait_for_stable_cluster` | `true` | Keep `true` unless your `oc`/cluster cannot support stable-cluster waiting and you intentionally use external validation. |
| `finish_restore_stable_cluster_minimum_period` | `1m` | Change if you need a longer continuous healthy period before declaring success. |
| `finish_restore_stable_cluster_timeout` | `30m` | Increase for very slow recovery. |
| `finish_restore_repair_etcd_endpoint_addresses` | `true` | Keep enabled, especially for CRC/SNO. It repairs restored `localhost` etcd peer URLs and `openshift-etcd/etcd-endpoints` values to the restore node InternalIP. |

CRC/SNO local backup example:

```yaml
i_understand_this_is_destructive: true
auto_approve_csrs: true
restore_require_kasten_download_prerequisites: false
restore_host_path: /mnt/data
restore_snapshot_file: snapshot_2026-08-05_084046.db
restore_use_etcdctl_restore: true
```

This shape assumes you already copied the snapshot and
`static_kuberesources_*.tar.gz` to `/mnt/data` on the restore node. The playbook
still verifies the snapshot exists before destructive restore tasks run.

The restore namespace, PV, and PVC are prerequisites for the Kasten workflow.
The playbook verifies that they exist and that the PVC is already `Bound`; it
does not create or mutate them. The Kasten documentation names these resources
`pv-etcd` and `pvc-etcd`. If your manifests use different names, update
`restore_pv_name` and `restore_pvc_name`.

If `restore_require_kasten_download_prerequisites=false`, the playbook skips the
Kasten namespace/PV/PVC checks and verifies only the pre-staged snapshot under
`restore_host_path`. Use this for CRC/SNO/local lab restores, not for the normal
Kasten workflow.

Use an empty `storageClassName: ""` on static PV/PVC manifests unless you
intentionally provision dynamic storage. If omitted, OpenShift/Kubernetes can
apply the cluster default StorageClass, which may prevent binding to the static
hostPath PV at `/mnt/data`.

Operational constants live in role defaults, not inventory:

| Role | Default |
|---|---|
| `check_prerequisites` | `restore_node_label_key: etcd-restore` |
| `check_prerequisites` | `restore_node_label_value: "true"` |
| `check_prerequisites` | `restore_host_path: /mnt/data` |
| `execute_restore` | `restore_script_path: /usr/local/bin/cluster-restore.sh` |
| `execute_restore` | `restore_host_path: /mnt/data` |
| `execute_restore` / `finish_restore` | `restore_wait_retries: 90` |
| `execute_restore` / `finish_restore` | `restore_wait_delay: 10` |
| `finish_restore` | `finish_restore_force_redeployment: true` |
| `finish_restore` | `finish_restore_wait_for_stable_cluster: true` |
| `finish_restore` | `finish_restore_stable_cluster_minimum_period: 1m` |
| `finish_restore` | `finish_restore_stable_cluster_timeout: 30m` |
| `finish_restore` | `finish_restore_repair_etcd_endpoint_addresses: true` |

OpenShift host paths and services used by the restore procedure are intentionally
hardcoded to the standard locations: `/etc/kubernetes/manifests`,
`/var/lib/etcd`, and `kubelet.service`.

## Kasten restore-download phase

The playbook verifies the target namespace/PV/PVC and uses the existing
`etcd-restore=true` node label to identify the restore node. The actual Kasten
restore selection remains a manual dashboard action because you must choose the
correct restore point.

The relevant restore phase from the Kasten Blueprint is:

```yaml
restore:
  # This phase is not actualy performing restore of the etcd data store but is used
  # to copy backup data to one of the leader nodes. It spins a pod on a leader node
  # having label etcd-restore. The pod is used to download the backup file from the
  # object store and copy it to the /mnt/data location of the PV mapped to PVC pvc-etcd.
  # The PV's mount path is /mnt/data on leader node where the cluster-ocp-restore.sh
  # script would be executed.
```

When the playbook stops because `/mnt/data/etcd-backup.db` is missing:

1. Open the Veeam Kasten dashboard.
2. Select the wanted etcd restore point.
3. Restore into the namespace configured by `etcd_restore_namespace`, default `etcd-restore`.
4. Confirm the Kanister restore pod runs on the node labeled `etcd-restore=true`.
5. Confirm the backup file exists on the restore node at `/mnt/data/etcd-backup.db`.
6. Re-run the playbook.

## Run

Safe syntax check:

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/restore.yml --syntax-check
```

Run the full restore only after setting the destructive confirmation:

```bash
ansible-playbook \
  -i inventories/prod/hosts.yml \
  playbooks/restore.yml \
  -e i_understand_this_is_destructive=true
```

To automatically approve pending CSRs:

```bash
ansible-playbook \
  -i inventories/prod/hosts.yml \
  playbooks/restore.yml \
  -e i_understand_this_is_destructive=true \
  -e auto_approve_csrs=true
```

With `auto_approve_csrs=true`, the playbook also attempts the common post-restore
kubelet recovery path automatically: if a control-plane node remains `NotReady`
after kubelet restart, it removes `/var/lib/kubelet/pki/*.pem`, restarts
`kubelet.service`, waits briefly for new CSRs, and then the `approve_csrs` role
approves pending CSRs. This covers the normal kubelet client certificate recovery
case. It also detects a stopped kubelet and tries to start/enable it before
falling back to manual diagnostics.

Single-node/CRC restores can restore etcd member peer URLs or the
`openshift-etcd/etcd-endpoints` ConfigMap with `localhost`. The etcd operator
rejects that value because it is not a routable node IP. The finish role now
detects the restore node's InternalIP, reads the restored etcd member ID/name
from the running etcd pod, updates the member peer URL when needed, patches the
ConfigMap when needed, and restarts the etcd operator only after a repair.

If `auto_approve_csrs=false` and pending CSRs exist, the playbook prints:

```bash
oc get csr
oc describe csr <csr_name>
oc adm certificate approve <csr_name>
```

Approve valid CSRs manually, then re-run the playbook.

## What the playbook does

1. `check_prerequisites`
   - verifies local tools and OpenShift admin access
   - verifies the OpenShift etcd namespace
   - verifies Kasten namespace and restore PV/PVC when `restore_require_kasten_download_prerequisites=true`
   - verifies etcd pods are discoverable
   - verifies SSH/sudo/crictl/static pod access on all control-plane hosts
   - verifies exactly one control-plane node has the restore-node label
   - verifies the snapshot file exists under `restore_host_path` on the restore node

2. `execute_restore`
   - requires `i_understand_this_is_destructive=true`
   - stops etcd and kube-apiserver static pods on non-restore control-plane nodes
   - waits up to 15 minutes for those static pod containers to stop
   - moves old `/var/lib/etcd` aside on non-restore nodes
   - copies the bundled `cluster-restore.sh` to `/usr/local/bin/cluster-restore.sh` on the restore node over SSH
   - runs `cluster-restore.sh /mnt/data` on the restore node
   - restarts kubelet on all control-plane nodes
   - verifies kubelet is running and tries to start/enable it when it is stopped
   - automatically performs kubelet certificate recovery for nodes that stay `NotReady`: remove `/var/lib/kubelet/pki/*.pem`, restart kubelet, then let the CSR role approve or print pending CSRs

3. `approve_csrs`
   - lists pending CSRs
   - approves them when `auto_approve_csrs=true`
   - otherwise prints manual approval commands and stops if pending CSRs exist

4. `finish_restore`
   - waits for the restored etcd container and OpenShift etcd pod to appear
   - prints machine replacement instructions for lost non-restore control-plane machines
   - waits for control-plane nodes to become `Ready`; if any stay `NotReady`, prints diagnostic commands and stops for manual intervention
   - forces redeployment of etcd, kube-apiserver, kube-controller-manager, and kube-scheduler
   - waits for ClusterOperators with `oc adm wait-for-stable-cluster`
   - prints final verification commands

If nodes remain `NotReady` after automatic kubelet start, kubelet certificate
recovery, and CSR handling, diagnose manually before continuing:

```bash
oc get nodes -o wide
oc describe node <node>
oc get csr
oc get co
ssh <node>
sudo systemctl status kubelet.service
sudo journalctl -u kubelet.service -b --no-pager | tail -200
sudo crictl ps -a
```

At that point the likely causes are outside automatic CSR/kubelet certificate
recovery: host networking/DNS/runtime health, a kubelet that cannot start, or a
lost Machine that must be replaced.

## Updating the bundled restore script

The playbook always uses the bundled script at:

```text
roles/execute_restore/files/cluster-restore.sh
```

Before using this repository on a real restore, check whether OpenShift changed the upstream script:

```bash
curl -fsSL \
  https://raw.githubusercontent.com/openshift/cluster-etcd-operator/main/bindata/etcd/cluster-restore.sh \
  -o /tmp/cluster-restore.sh

sha256sum roles/execute_restore/files/cluster-restore.sh /tmp/cluster-restore.sh
diff -u roles/execute_restore/files/cluster-restore.sh /tmp/cluster-restore.sh
```

If the upstream version is newer and appropriate for your OpenShift version, update the bundled copy:

```bash
cp /tmp/cluster-restore.sh roles/execute_restore/files/cluster-restore.sh
ansible-playbook -i inventories/example/hosts.yml playbooks/restore.yml --syntax-check
ansible-lint --profile moderate .
```

## Manual machine replacement

Do not delete or recreate the restore-node Machine. For each lost non-restore control-plane Machine, one at a time:

```bash
oc get machines -n openshift-machine-api -o wide
oc get machine <old-master-machine> -n openshift-machine-api -o yaml > new-master-machine.yaml
```

Edit `new-master-machine.yaml`:

- remove `status`
- set a new `metadata.name`
- remove `spec.providerID`
- remove `metadata.annotations`
- remove `metadata.generation`
- remove `metadata.resourceVersion`
- remove `metadata.uid`

Then:

```bash
oc delete machine -n openshift-machine-api <old-master-machine>
oc apply -f new-master-machine.yaml
oc get machines -n openshift-machine-api -o wide
```

Wait for the replacement node before recreating the next one.

## Final checks

```bash
oc get nodes -w
oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd
oc get csr
oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
oc get kubecontrollermanager -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
oc get kubescheduler -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
```

## License

MIT
