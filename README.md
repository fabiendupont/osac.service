# osac.service

Shared utility roles for OSAC provider collections.

## Roles

| Role | Purpose |
|---|---|
| `common` | Kubeconfig retrieval for remote clusters |
| `lease` | Distributed locking via K8s Lease |
| `retrieve_kubeconfig` | Multi-cluster auth |
| `select_hosts` | Bare-metal host selection from AAP inventory |
| `wait_for` | Generic wait for K8s resources |
| `workflow_helpers` | Noop hook points for test overrides |
