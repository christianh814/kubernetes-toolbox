# Store

Notes specific to storage will be here

[Rook](#rook)
[Glusterfs](#glusterfs)

## Rook

Quick notes until I have more time.

First, install via helm

```
helm repo add rook-stable https://charts.rook.io/stable
helm install --namespace rook-ceph-system rook-stable/rook-ceph
```

Next, Create a rook cluster with the [rook-cluster.yaml](../examples/rook-cluster.yaml) (found in this repo)

```
kubectl create -f rook-cluster.yaml
```

## Glusterfs

tk
