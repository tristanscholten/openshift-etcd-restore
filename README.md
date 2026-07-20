# OpenShift etcd Restore

![Ansible](https://img.shields.io/badge/Ansible-2.15%2B-EE0000?logo=ansible&logoColor=white)
![OpenShift](https://img.shields.io/badge/OpenShift-4.x-red?logo=redhatopenshift&logoColor=white)
![Kasten Kanister](https://img.shields.io/badge/Kasten-Kanister-00AEEF)
![License](https://img.shields.io/badge/License-MIT-blue.svg)

A small Ansible wrapper around the Veeam Kasten Kanister OpenShift etcd restore flow:

<https://docs.kasten.io/latest/kanister/etcd/ocp/install>

> [!CAUTION]
> This is disaster-recovery automation. It stops static control-plane pods, moves etcd data on non-restore control-plane nodes, runs the bundled OpenShift `cluster-restore.sh`, handles CSRs, and forces control-plane redeployments. Do not run it casually.

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

The restore node is selected in inventory, not in `group_vars`:

```yaml
all:
  children:
    control_plane:
      hosts:
        master-0.example.com:
          ansible_host: 10.0.0.10
          openshift_node_name: master-0.example.com
          openshift_etcd_restore_node: true
        master-1.example.com:
          ansible_host: 10.0.0.11
          openshift_node_name: master-1.example.com
          openshift_etcd_restore_node: false
        master-2.example.com:
          ansible_host: 10.0.0.12
          openshift_node_name: master-2.example.com
          openshift_etcd_restore_node: false
```

Exactly one `control_plane` host must have `openshift_etcd_restore_node: true`.

## Prerequisites

Workstation:

- `ansible-playbook`
- `oc` authenticated as cluster-admin
- SSH access with passwordless sudo to all control-plane nodes
- the bundled OpenShift `cluster-restore.sh` from `roles/execute_restore/files/`
  - source: <https://github.com/openshift/cluster-etcd-operator/blob/main/bindata/etcd/cluster-restore.sh>
- optional: set `restore_script_local_path` if you need to override the bundled script with a custom controller-local copy

Cluster/Kasten:

- Kasten installed, default namespace `kasten-io`
- OpenShift etcd pods in `openshift-etcd`
- Kasten Kanister etcd Blueprint applied
- a successful etcd restore point
- a selected restore control-plane node

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
restore_script_source: cluster-restore.sh
restore_script_remote_path: /root/cluster-restore.sh
restore_host_path: /mnt/data
restore_snapshot_file: etcd-backup.db
```

## Kasten restore-download phase

The playbook prepares the target namespace/PV/PVC and labels the restore node. The actual Kasten restore selection remains a manual dashboard action because you must choose the correct restore point.

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
   - verifies Kasten and etcd namespaces
   - verifies etcd pods are discoverable
   - verifies SSH/sudo/crictl/static pod access on all control-plane hosts
   - ensures the restore namespace, PV, PVC, and restore-node label exist
   - verifies the Kasten restore has downloaded the etcd backup to the restore node

2. `execute_restore`
   - requires `i_understand_this_is_destructive=true`
   - stops etcd and kube-apiserver static pods on non-restore control-plane nodes
   - moves old `/var/lib/etcd` aside on non-restore nodes
   - copies the bundled `cluster-restore.sh` to the restore node over SSH
   - runs `/root/cluster-restore.sh /mnt/data` on the restore node
   - restarts kubelet on all control-plane nodes

3. `approve_csrs`
   - lists pending CSRs
   - approves them when `auto_approve_csrs=true`
   - otherwise prints manual approval commands and stops if pending CSRs exist

4. `finish_restore`
   - verifies etcd container/pods
   - prints machine replacement instructions for lost non-restore control-plane machines
   - forces redeployment of etcd, kube-apiserver, kube-controller-manager, and kube-scheduler
   - prints final verification commands

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
