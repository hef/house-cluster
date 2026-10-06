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

This cluster is still on Talos 1.13, so install disk and cluster settings stay
in the `machine:` / `cluster:` documents. Do not add `UnattendedInstallConfig`
until the OS is on 1.14.
