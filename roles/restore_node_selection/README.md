# Role: restore_node_selection

Derives the OpenShift etcd restore node from inventory.

Set `openshift_etcd_restore_node: true` on exactly one host in the `control_plane` inventory group. The role exposes:

- `restore_node_inventory_hostname`
- `restore_node_name`
- `restore_node_is_current_host`

This keeps restore-node selection close to the host definition instead of hiding it in `group_vars`.
