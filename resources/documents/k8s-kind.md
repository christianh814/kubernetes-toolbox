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

It uses `kubeadmin` so you can provide it customizations if you wish. Here is an example mulimaster setup (opening ports to use in case I want to install ingress)

```yaml
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
networking:
  disableDefaultCNI: True
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
kubeadmConfigPatches:
- |
  apiVersion: kubeadm.k8s.io/v1beta2
  kind: ClusterConfiguration
  metadata:
    name: config
  networking:
    serviceSubnet: "172.30.0.0/16"
    podSubnet: "10.254.0.0/16"
```

Then you'd just run...

```
kind create cluster --config=config.yaml
```

:warning: In my above yaml example I disabled the default CNI, so you need to install one after the cluster starts. Example:

```shell
curl -so calico.yaml https://docs.projectcalico.org/v3.11/manifests/calico.yaml
sed -i 's/192\.168/10\.254/g' calico.yaml
kubectl apply -f calico.yaml
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
helm repo update 
helm install ingress-nginx stable/nginx-ingress --namespace ingress-nginx --set controller.nodeSelector.nginx="ingresshost" \
--set rbac.create=true --set controller.image.pullPolicy="Always" --set controller.extraArgs.enable-ssl-passthrough="" \
--set controller.stats.enabled=true --set controller.service.type="ClusterIP" \
--set controller.kind="DaemonSet" --set controller.daemonset.useHostPort=true 
```

:warning: Note, that I labeled the worker based on the one I did the portmappings on in the `config.yaml` file. In my case it was the first node.
