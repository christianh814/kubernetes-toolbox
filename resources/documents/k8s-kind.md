# KIND

Kind stands for "Kubernetes In Docker" and it's a [SIG](kind.sigs.k8s.io/) that was originally intended to run tests.

In order to run kind you need...

* Docker (I've tested on 1.13.1 and newer)
  * Podman only works in rootful mode with version 1.8.1 (or newer) and kind v0.8 
* The [kind binary](https://github.com/kubernetes-sigs/kind/releases)
* Go version 1.11 or newer
* And [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-on-linux)

# Running a cluster

The [quickstart](https://kind.sigs.k8s.io/docs/user/quick-start/) can help you get started...below is MY notes for my use case.

To create a cluster, you can simply run

```
kind create cluster
```

Here is an example mulimaster setup (opening ports to use in case I want to install ingress). I disabled the CNI, and changed the podsubnet and servicesubnet (because I'm using calico).

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: True
  podSubnet: "10.254.0.0/16"
  serviceSubnet: "172.30.0.0/16"
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    listenAddress: 0.0.0.0
  - containerPort: 443
    hostPort: 443
    listenAddress: 0.0.0.0
- role: worker
- role: worker
```

Then you'd just run...

```
kind create cluster --config=config.yaml
```

:warning: In my above yaml example I disabled the default CNI, so you need to install one after the cluster starts. Example:

```shell
curl -so manifests/calico.yaml https://docs.projectcalico.org/v3.11/manifests/calico.yaml
sed -i 's/192\.168/10\.254/g' manifests/calico.yaml
kubectl apply -f manifests/calico.yaml
kubectl rollout status ds calico-node -n kube-system
kubectl rollout status deploy calico-kube-controllers -n kubesystem
```

Worker lables don't get set, do it with this "one-liner"

```shell
kubectl  get nodes --no-headers -l '!node-role.kubernetes.io/master' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}' | xargs -I{} kubectl label node {} node-role.kubernetes.io/worker=''
```

Visit [the official doc](https://kind.sigs.k8s.io/docs/user/configuration/) for more information about configuration

# Ingress

Using the above `config.yaml` file, I installed `nginx` controller (after installing calico) using `helm` v3

```
kubectl label node kind-worker nginx=ingresshost
kubectl create ns ingress-nginx
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update 
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --set controller.nodeSelector.nginx="ingresshost" \
--set rbac.create=true --set controller.image.pullPolicy="Always" --set controller.extraArgs.enable-ssl-passthrough="" \
--set controller.stats.enabled=true --set controller.service.type="ClusterIP" \
--set controller.kind="DaemonSet" --set controller.daemonset.useHostPort=true
kubectl rollout status ds ingress-nginx-nginx-ingress-controller -n ingress-nginx
kubectl rollout status deploy ingress-nginx-nginx-ingress-default-backend -n ingress-nginx
```

:warning: Note, that I labeled the worker based on the one I did the portmappings on in the `config.yaml` file. In my case it was the first node.
