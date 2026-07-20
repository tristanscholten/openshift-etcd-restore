# OpenShift etcd Restore Automation

![Ansible](https://img.shields.io/badge/Ansible-2.15%2B-EE0000?logo=ansible&logoColor=white)
![OpenShift](https://img.shields.io/badge/OpenShift-4.x-red?logo=redhatopenshift&logoColor=white)
![Kasten Kanister](https://img.shields.io/badge/Kasten-Kanister-00AEEF)
![License](https://img.shields.io/badge/License-MIT-blue.svg)

Ansible automation for restoring an OpenShift etcd backup produced with the Veeam Kasten Kanister etcd Blueprint for OpenShift.

The flow follows the Kasten documentation:

<https://docs.kasten.io/latest/kanister/etcd/ocp/install>

> [!WARNING]
> etcd restore is a disaster-recovery operation. It intentionally disrupts the OpenShift control plane. Read this README completely, test in a non-production cluster, and keep console/BMC/cloud access to every control-plane node before running the destructive playbook.

## What this repository automates

| Phase | Playbook | What it does |
|---|---|---|
| Preflight | `playbooks/preflight.yml` | Verifies required local tools, OpenShift admin access, Kasten namespace, etcd pods, control-plane nodes, restore node selection, optional SSH reachability, and inventory shape. |
| Prepare Kasten restore target | `playbooks/prepare-kasten-restore.yml` | Creates the restore namespace, PV/PVC used by the Kanister restore action, labels the chosen restore node with `etcd-restore=true`, and waits for PVC binding. |
| Wait for downloaded snapshot | `playbooks/wait-for-snapshot.yml` | Verifies the Kasten restore action has placed the etcd snapshot on the restore node, usually `/mnt/data/etcd-backup.db`. |
| Execute host restore | `playbooks/restore-etcd.yml` | Stops static pods on non-restore control-plane nodes, moves their old etcd data aside, copies/runs `cluster-ocp-restore.sh` on the restore node, and restarts kubelet. Guarded by `restore_i_understand_this_is_destructive=true`. |
| Post-restore recovery | `playbooks/post-restore.yml` | Checks cluster status, optionally approves pending CSRs, and forces redeployment of etcd/API server/controller manager/scheduler. |

## Manual steps that still remain

Some restore steps are deliberately left manual because they require operator judgement or provider-specific replacement workflows:

1. **Choose the restore point in the Veeam Kasten dashboard.** This automation prepares the namespace/PVC/node label, but you must click the restore option for the selected etcd restore point in the Veeam dashboard.
2. **Use the prepared restore node and namespace in Kasten.** The playbook labels one master/control-plane node with `etcd-restore=true`. In the Veeam restore wizard, restore into the target namespace, by default `etcd-restore`. The Kanister restore action schedules on that labeled master node and places the etcd backup on the node at `/mnt/data`, typically `/mnt/data/etcd-backup.db`. The later restore playbook uses that file.
3. **Provide the modified `cluster-ocp-restore.sh`.** Kasten documents a modified OpenShift restore script that skips static pod manifest backup assumptions. Put that script at `files/cluster-ocp-restore.sh` or pass `-e restore_script_local_path=/path/to/cluster-ocp-restore.sh`.
4. **Delete/recreate lost control-plane machines one by one.** On installer-provisioned infrastructure or Machine API clusters this means exporting a Machine object, sanitising fields, deleting/recreating the lost machines, and never deleting the restore node. The README includes the exact command outline below.
5. **Review CSRs before approving.** `post-restore.yml` can approve pending CSRs automatically, but the default is review-only.
6. **Provider-specific node recovery.** Bare metal, VMware, AWS, Azure, and agent-based installs all differ. This repo automates the common OpenShift host actions, not provider-specific machine lifecycle.

## Repository layout

```text
.
├── ansible.cfg
├── inventories/example/
│   ├── hosts.yml
│   └── group_vars/all.yml
├── playbooks/
│   ├── preflight.yml
│   ├── prepare-kasten-restore.yml
│   ├── wait-for-snapshot.yml
│   ├── restore-etcd.yml
│   └── post-restore.yml
├── roles/
│   ├── preflight/
│   ├── kasten_restore_target/
│   ├── wait_for_snapshot/
│   ├── etcd_host_restore/
│   └── post_restore/
└── files/
    └── .gitkeep
```

## Prerequisites

### Operator workstation

- `ansible-playbook` 2.15 or newer
- `oc` CLI authenticated as a user with cluster-admin privileges
- SSH access with sudo to all OpenShift control-plane nodes
- A copy of the modified `cluster-ocp-restore.sh`
- Network access to the OpenShift API and the control-plane hosts

### OpenShift/Kasten

- OpenShift 4.x cluster where etcd static pods run on control-plane nodes
- Kasten installed, commonly in namespace `kasten-io`
- Kanister Blueprint for OpenShift etcd backup already applied
- A successful Kasten restore point containing the `etcdBackup` artifact
- A chosen control-plane restore node

## Quick start

Clone the repo and install Ansible if needed:

```bash
git clone https://github.com/tristanscholten/openshift-etcd-restore.git
cd openshift-etcd-restore
ansible-playbook --version
oc whoami
```

Copy the example inventory:

```bash
cp -R inventories/example inventories/prod
```

Edit `inventories/prod/hosts.yml`:

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

Edit `inventories/prod/group_vars/all.yml` and at minimum set:

```yaml
restore_node_name: master-0.example.com
restore_script_local_path: files/cluster-ocp-restore.sh
```

Then run the safe preflight checks:

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/preflight.yml
```

Prepare the restore namespace/PV/PVC/node label:

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/prepare-kasten-restore.yml
```

## Veeam Kasten dashboard restore step

This part is intentionally manual and important:

1. Open the Veeam Kasten dashboard.
2. Locate the restore point for the OpenShift etcd backup created with the Kanister etcd Blueprint.
3. Make sure the chosen master/control-plane node has the label `etcd-restore=true`. The `prepare-kasten-restore.yml` playbook applies this label automatically.
4. Click the **Restore** option for that restore point.
5. Choose the prepared target namespace, by default `etcd-restore`.
6. Start the restore.
7. Kasten/Kanister schedules the restore pod on the labeled master node and downloads the etcd backup to the restore PV mounted at `/mnt/data`.
8. The expected file is `/mnt/data/etcd-backup.db` on the restore node. This file is what `cluster-ocp-restore.sh` consumes in the later host restore steps.

Verify that the snapshot is present:

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/wait-for-snapshot.yml
```

Do not continue to the destructive host restore until this playbook confirms that `/mnt/data/etcd-backup.db` exists and has non-zero size on the restore node.

## Destructive restore execution

> [!CAUTION]
> This stops static etcd and API server pods on non-restore control-plane nodes and moves old etcd data directories aside. Do not run this unless the cluster is in an etcd restore scenario and you have out-of-band access to all control-plane nodes.

Dry-run-ish preview of host targeting:

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/restore-etcd.yml --list-hosts
```

Execute the restore:

```bash
ansible-playbook \
  -i inventories/prod/hosts.yml \
  playbooks/restore-etcd.yml \
  -e restore_i_understand_this_is_destructive=true
```

The playbook performs the common Kasten/OpenShift host-side sequence:

1. Confirms `restore_node_name` is one of the inventory control-plane hosts.
2. Confirms the etcd snapshot exists on the restore node.
3. Copies the modified `cluster-ocp-restore.sh` to the restore node.
4. On every non-restore control-plane node:
   - moves `/etc/kubernetes/manifests/etcd-pod.yaml` out of the static pod path
   - waits for etcd static pod containers to stop
   - moves `/etc/kubernetes/manifests/kube-apiserver-pod.yaml` out of the static pod path
   - waits for kube-apiserver static pod containers to stop
   - moves `/var/lib/etcd` aside to a timestamped path
5. On the restore node:
   - runs `sudo ./cluster-ocp-restore.sh /mnt/data`
6. Restarts kubelet on all control-plane nodes.

## Post-restore recovery

Run the post-restore checks and force redeployments:

```bash
ansible-playbook -i inventories/prod/hosts.yml playbooks/post-restore.yml
```

By default this shows pending CSRs but does not approve them. To approve pending CSRs automatically:

```bash
ansible-playbook \
  -i inventories/prod/hosts.yml \
  playbooks/post-restore.yml \
  -e approve_pending_csrs=true
```

## Manual machine replacement outline

Do not delete or recreate the restore-node Machine. For each lost non-restore control-plane machine, one at a time:

```bash
oc get machines -n openshift-machine-api -o wide
oc get machine <old-master-machine> -n openshift-machine-api -o yaml > new-master-machine.yaml
```

Edit `new-master-machine.yaml`:

- remove the entire `status` section
- set a new `metadata.name`
- remove `spec.providerID`
- remove `metadata.annotations`
- remove `metadata.generation`
- remove `metadata.resourceVersion`
- remove `metadata.uid`

Then recreate:

```bash
oc delete machine -n openshift-machine-api <old-master-machine>
oc get machines -n openshift-machine-api -o wide
oc apply -f new-master-machine.yaml
oc get machines -n openshift-machine-api -o wide
```

Wait until the replacement machine reaches `Running` and the Node joins before replacing the next one.

## Useful verification commands

```bash
oc get nodes -w
oc get pods -n openshift-etcd | grep -v etcd-quorum-guard | grep etcd
oc get csr
oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
oc get kubeapiserver -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
oc get kubecontrollermanager -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
oc get kubescheduler -o=jsonpath='{range .items[0].status.conditions[?(@.type=="NodeInstallerProgressing")]}{.reason}{"\n"}{.message}{"\n"}'
```

## Safety defaults

- Destructive host restore tasks do nothing unless `restore_i_understand_this_is_destructive=true`.
- CSR approval is review-only unless `approve_pending_csrs=true`.
- Static pod manifests and `/var/lib/etcd` are moved to timestamped backup paths, not deleted.
- The restore node is never included in the non-restore etcd data move.
- Preflight validates OpenShift API access, Kasten namespace, etcd pods, control-plane nodes, and the chosen restore node before changes.

## License

MIT
