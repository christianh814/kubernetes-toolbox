# ArgoCD

> Quicknotes for now from [official site](https://argoproj.github.io/argo-cd/getting_started/)

## K8S

1.Install

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2.[Download the latest Argo CD version](https://github.com/argoproj/argo-cd/releases/latest)

3.Create ingress for argo (I used nginx that was installed with helm)


```
kubectl create -n argocd -f manifests/argo-ingress.yaml
```

Contents of `arog-ingress.yaml`

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    # needed for SSL passthrough
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
  name: argocd-ingress
  namespace: argocd
spec:
  rules:
  - host: argocd.127.0.0.1.nip.io
    http:
      paths:
      - backend:
          serviceName: argocd-server
          servicePort: 443
        path: /
```

4.Login Using The CLI

Get the pod name (that's the default admin password)

```
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

Then login 

```
argocd login argocd.127.0.0.1.nip.io
```

change the password

```
argocd account update-password
```

5.Add your clusters

List your clusters in your context

```
argocd cluster add
```

add the cluster by name

```
argocd cluster add kind-kind
argocd cluster list
```

6.Create your app

add your repo and create your app

> note: you can add `--sync-policy automated`

```
argocd repo add https://github.com/christianh814/gitops-examples

argocd app create --project default --name bgd --repo https://github.com/christianh814/gitops-examples --path bgd/ --dest-server  https://kubernetes.default.svc --dest-namespace bgd --revision master
```

7.Get status of oyur app


List your apps

```
argocd app list
```

Status

```
argocd app get bgd
```

8.Sync if you see descrepencies

Patch the app

```
kubectl -n bgd patch deploy/bgd --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/env/0/value", "value":"green"}]'
```

Sync

```
argocd app sync bgd
```

delete unexpected resources

```
argocd app sync --prune bgd
```

9.Kustomize

Based on repo https://github.com/christianh814/gitops-examples under bgdk

add your clusters..

```
oc login --insecure-skip-tls-verify https://api.openshift4.cloud.chx:6443
oc login --insecure-skip-tls-verify https://api.ocp4.cloud.chx:6443
argocd cluster add  default/api-openshift4-cloud-chx:6443/ocp-admin
argocd cluster add  default/api-ocp4-cloud-chx:6443/ocp-admin
```

Verify with

```
argocd cluster list
```

Add your repo if you haven't already

```
argocd repo add https://github.com/christianh814/gitops-examples
```

Add the app to your first cluster...specifying the overlay you want.

```
argocd app create --project default --name bgdk1 --repo https://github.com/christianh814/gitops-examples --path bgdk/overlays/cluster1 --dest-server https://api.openshift4.cloud.chx:6443 --dest-namespace bgd --revision master
```

Now do the same for cluster 2

```
argocd app create --project default --name bgdk2 --repo https://github.com/christianh814/gitops-examples --path bgdk/overlays/cluster2 --dest-server https://api.ocp4.cloud.chx:6443 --dest-namespace bgd --revision master
```

Now sync them

```
argocd app sync bgdk1
argocd app sync bgdk2
```

## OCP

On OpenShift, once you've installed the operator, just load [this CR](../examples/argocd.yaml)

```
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
  namespace: argocd
spec:
  server:
    route: true
  dex:
    openShiftOAuth: true
    image: quay.io/redhat-cop/dex
    version: v2.22.0-openshift
  rbac:
    defaultPolicy: 'role:readonly'
    policy: |
      g, system:cluster-admins, role:admin
    scopes: '[groups]'
```
