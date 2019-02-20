# Store

Notes specific to storage will be here

* [Rook](#rook)
* [Glusterfs](#glusterfs)

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

Next, create the [rook-storageclass.yaml](../examples/rook-storageclass.yaml) file and apply it to the k8s cluster.

```
kubectl create -f rook-storageclass.yaml
```

Next, test it by creating a `pvc` (reference the storageclass and RWO since it's block); and it will automatically create a block storage for you

```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: ceph-block-pvc0001
 annotations:
   volume.beta.kubernetes.io/storage-class: rook-ceph-block
spec:
 accessModes:
  - ReadWriteOnce
 resources:
   requests:
     storage: 1Gi
EOF
```

Check your work

```
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                     STORAGECLASS      REASON   AGE
pvc-ca1b04ab-353d-11e9-92c1-42010a8e002f   1Gi        RWO            Retain           Bound    test/ceph-block-pvc0001   rook-ceph-block            1h

$ kubectl get pvc --all-namespaces 
NAMESPACE   NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
test        ceph-block-pvc0001   Bound    pvc-ca1b04ab-353d-11e9-92c1-42010a8e002f   1Gi        RWO            rook-ceph-block   1h
```

## Glusterfs

tk
