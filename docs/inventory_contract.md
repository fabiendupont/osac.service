# OSAC Host Inventory Contract

This document defines the host_vars contract that all OSAC inventory sources
must satisfy for host selection and state management to work correctly.

## Required host_vars

| Variable    | Type   | Description                                      |
|-------------|--------|--------------------------------------------------|
| `hostclass` | string | ComputeInstanceClass name (e.g. `gpu-h100-8x`)   |
| `state`     | string | Current host state (see State Values below)       |
| `site`      | string | Region or site identifier (e.g. `us-east-1`)     |

## Optional host_vars

| Variable                 | Type   | Description                                        |
|--------------------------|--------|----------------------------------------------------|
| `tenant`                 | string | Tenant ID when host is allocated                   |
| `nvlink_domain`          | string | NVLink domain for GPU affinity grouping            |
| `rack`                   | string | Physical rack identifier                           |
| `bmc_address`            | string | BMC/IPMI endpoint address                          |
| `bmc_credentials_secret` | string | Kubernetes secret name for BMC credentials         |
| `boot_mac_address`       | string | MAC address for PXE boot                           |
| `gpu_uuids`              | list   | GPU device UUIDs                                   |
| `ib_guids`               | list   | InfiniBand port GUIDs                              |
| `root_device`            | string | Root disk device path or serial for provisioning   |
| `inventory_source`       | string | Backend type: `netbox`, `nico`, `ironic`, `static` |

## State Values

| State         | Description                                          |
|---------------|------------------------------------------------------|
| `available`   | Host is ready for allocation                         |
| `allocated`   | Host is assigned to a tenant                         |
| `maintenance` | Host is undergoing maintenance, not available        |
| `error`       | Host is in an error state, requires investigation    |

## Inventory Source Mapping

The `update_host_state` role maps OSAC states to backend-native values:

### NetBox (`netbox.netbox.netbox_device`)

| OSAC State    | NetBox Status |
|---------------|---------------|
| `available`   | `active`      |
| `allocated`   | `active`      |
| `maintenance` | `planned`     |
| `error`       | `failed`      |

Tenant assignment is handled via the NetBox `tenant` field.

### NICo (`nvidia.bare_metal.machine`)

| OSAC State    | NICo Label `state` |
|---------------|---------------------|
| `available`   | `available`         |
| `allocated`   | `allocated`         |
| `maintenance` | `maintenance`       |
| `error`       | `error`             |

State and tenant are stored as machine labels.

### Ironic (`openstack.cloud.baremetal_node`)

| OSAC State    | Ironic `extra.state` |
|---------------|----------------------|
| `available`   | `available`          |
| `allocated`   | `allocated`          |
| `maintenance` | `maintenance`        |
| `error`       | `error`              |

State and tenant are stored in the node's `extra` metadata.

### Static AAP Inventory (`ansible.controller.host`)

| OSAC State    | host_vars `state` |
|---------------|-------------------|
| `available`   | `available`       |
| `allocated`   | `allocated`       |
| `maintenance` | `maintenance`     |
| `error`       | `error`           |

State and tenant are stored directly as host variables in the AAP inventory.

## Usage with select_hosts

The `osac.service.select_hosts` role queries hosts from AAP inventory
and filters by `hostclass`, `state == 'available'`, and optionally `site`.
It supports two placement strategies:

- **pack**: Fill groups by affinity key before spilling (default)
- **spread**: Round-robin across affinity groups for fault tolerance

After selection, use `osac.service.update_host_state` to transition
selected hosts to `allocated` state with the appropriate tenant ID.

## Example Workflow

```yaml
# 1. Select hosts matching a class
- name: Select GPU hosts
  ansible.builtin.include_role:
    name: osac.service.select_hosts
  vars:
    select_hosts_class: "gpu-h100-8x"
    select_hosts_count: 4
    select_hosts_region: "us-east-1"
    select_hosts_placement:
      strategy: "spread"
      affinity_key: "nvlink_domain"

# 2. Mark selected hosts as allocated
- name: Update host state to allocated
  ansible.builtin.include_role:
    name: osac.service.update_host_state
  vars:
    update_host_names: "{{ select_hosts_result | map(attribute='inventory_hostname') | list }}"
    update_host_state: "allocated"
    update_host_tenant: "tenant-abc-123"
```
