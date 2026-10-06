# Talos

Machine configs are assembled by [TOPF](https://postfinance.github.io/topf/) from
`topf.yaml` plus the strategic merge patches in this directory.

Day-2 operations stay behind the existing `task talos:*` / `task bootstrap:talos`
wrappers.

## Patch directories

Patches merge in this order, alphabetically within each directory, with later
patches taking precedence:

- `all/`: applied to every node
- `control-plane/`: applied to control-plane nodes
- `worker/`: applied to worker nodes
- `node/${hostname}/`: applied to the node with the specified name

Files ending in `.yaml.tpl` are Go-templated per node.

Talos 1.14 splits Kubernetes, DNS, and install settings into dedicated
documents. Patches in this directory use those kinds (`Kube*Config`,
`ResolverConfig`, `UnattendedInstallConfig`) so they do not conflict with
the documents TOPF emits. etcd extra args remain on the v1alpha1 `cluster:`
document.
