# Kuberentes Notes

These are notes in no paticular oder (it is mostly focused on installation)

* [Installing with Kubeadm](#kubernetes-installation)
* [Installing in the cloud with ClusterAPI](https://cluster-api.sigs.k8s.io/user/quick-start.html)
* [Ingress](#ingress)
* [Kubernetes Authentication](resources/documents/k8s-auth.md#kubernetes-authentication)
* [Storage](resources/documents/k8s-store.md#storage)
* [Miscellaneous](#miscellaneous-notes)

# Kubernetes Installation

This guide will show you how to "easily" (as easy as possible) install a Multimaster system. You can also modify this to have a similer one contoller/one node setup (steps are almost exactly the same).

Also, this is here mainly for my notes; so I may be brief in some sections. I TRY to keep this up to date but you're better off looking at the [official documentation](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

This is broken off into sections

* [Prerequisites And Assumptions](#prerequisites-and-assumptions)
* [Infrastructure](#infrastructure)
* [Installing Kubeadm](#installing-kubeadm)
* [Initialize the Control Plane](#initialize-the-control-plane)
* [Bootstrap Remaining Controllers](#bootstrap-remaining-controllers)
* [Bootstrap Worker Nodes](#bootstrap-worker-nodes)
* [Smoke Test](#smoke-test)

# Prerequisites And Assumptions

Out of all sections, this is probably the most important one. Here I will list prereqs and assumptions. This list isn't complete and I assume you know your way around Linux and are familiar with containers and Kubernetes already. Thanks to [this blog](https://octetz.com/posts/ha-control-plane-k8s-kubeadm) for updating some of this howto.

Prereqs:

* You have control of DNS
  * Also that DNS is in place for the hosts (forward and reverse)
  * You can use [nip.io](http://nip.io) if you wish
* You are using Tmux and/or Ansible
  * A lot of these tasks are done on all machines at the same time
  * I used mostly Ansible with some Tmux here and there (where it made sense)
* All hosts are on one network segment
  * All my VMs were on the same vlan
  * I didn't have to create firewall rules on a router/firewall


Assumptions:

* You are well vested in the art of Linux
* You're running this on virtual machines
* You either have preassinged DHCP or static ips
* DNS is in place (forward and reverse)

# Infrastructure

I have spun up 7 VMs

* 1 LB VM
* 3 VMs for the controllers (will also run etcd)
* 3 VMs for the workers


The contollers/workers are set up as follows

* Latest CentOS 7 (FULLY updated with `yum -y update` and `systemctl reboot`-ed)
* 50GB HD for the OS
* 4 GB of RAM
* 4 vCPUs
* Swap has been disabled
* SELinux has been disabled (part of the howto)
* Firewall has been disabled (part of the howto)

## HAProxy Setup

Although not 100% within the scope of this document (really any LB will do); this is how I set up my LB on my LB server. These are "high level" notes.

Install HAProxy package

```
yum -y install haproxy
```

Then I enable `6443` (kube api) and `9000` (haproxy stats) on the firewall

```
firewall-cmd --add-port=6443/tcp --add-port=6443/udp --add-port=9000/tcp --add-port=9000/udp --permanent
firewall-cmd --reload
```

If using selinux, make sure "connect any" is on

```
setsebool -P haproxy_connect_any=on
```

Edit the `/etc/haproxy/haproxy.cfg` file and add the following

```
frontend  kube-api
    bind *:6443
    default_backend k8s-api
    mode tcp
    option tcplog

backend k8s-api
    balance source
    mode tcp
    server      controller-0 192.168.1.98:6443 check
    server      controller-1 192.168.1.99:6443 check
    server      controller-2 192.168.1.5:6443 check
```

If you want a copy of the whole file, you can find an example [here](resources/examples/k8s-haproxy.cfg)

Now you can start/enable the haproxy service

```
systemctl enable --now haproxy
```

I like to keep `<ip of lb>:9000` up so I can monitor when my control plane servers has come up

# Installing Kubeadm

On ALL servers, install these packages:

* `kubeadm`: the command to bootstrap the cluster.
* `kubelet`: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
* `kubectl`: the command line util to talk to your cluster.
* `containerd`: Container runtime.

You should also disable the firewall and SELinux at this point as well

```
ansible all -m shell -a "sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config"
ansible all -m shell -a "setenforce 0"
ansible all -m shell -a "systemctl stop firewalld"
ansible all -m shell -a "systemctl disable firewalld"
ansible all -m shell -a "modprobe overlay"
ansible all -m shell -a "modprobe br_netfilter"
ansible all -m copy -a "src=files/containerd.conf dest=/etc/modules-load.d/containerd.conf"
ansible all -m copy -a "src=files/k8s.repo dest=/etc/yum.repos.d/kubernetes.repo"
ansible all -m copy -a "src=files/sysctl_k8s.conf dest=/etc/sysctl.d/k8s.conf"
ansible all -m shell -a "sysctl --system"
ansible all -m shell -a "yum install -y yum-utils device-mapper-persistent-data lvm2"
ansible all -m shell -a "yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo"
ansible all -m shell -a "yum install -y containerd.io"
ansible all -m shell -a "mkdir -p /etc/containerd"
ansible all -m shell -a "containerd config default > /etc/containerd/config.toml"
ansible all -m shell -a "systemctl restart containerd"
ansible all -m shell -a "yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes"
ansible all -m shell -a "systemctl enable --now kubelet"
```

(Note: A copy of [k8s.repo](resources/examples/k8s.repo), [sysctl_k8s.conf](resources/examples/sysctl_k8s.conf), and [containerd.conf](resources/examples/containerd.conf) are found in this repo)

If you run `systemctl status kubelet` you will see it error out becuase it's trying to connect to a control plane that's not there. This is a normal error.

At this point, "pre-pull" the required images on ONLY the controllers to run the control plane.

```
ansible controllers -m shell -a "kubeadm config images pull"
```

# Initialize the Control Plane

Pick one of the controllers (doesn't matter which one) and initialize the control plane with `kubeadm`. I used a config file since I was going through a loadbalancer. (Also note, if you're using Calico (like I was) you have to specify a `podSubnet` that's something other than `192.168.0.0/16`)

I did the following on my first controller (in the below example; the IP of my LB was `192.168.1.97`)

```
LBIP=192.168.1.97
cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - ${LBIP}
controlPlaneEndpoint: ${LBIP}:6443
networking:
  podSubnet: 10.254.0.0/16
  serviceSubnet: 172.30.0.0/16
EOF
```

Use this `kubeadm-config.yaml` in install the first controller. On the controller you created the file, run the `kubeadm init` command...


```
kubeadm init --config=kubeadm-config.yaml --upload-certs
```

> The `--upload-certs` uploads certificates to etcd (encrypted) so you can bootstrap your other masters

At this point, if you reload your LB's status page, you should see the first controller going green.

When the bootstrapping finishes; you should see a message like the following. SAVE THIS MESSAGE! You'll need this to join the other 2 controllers. 

```
kubeadm join <loadbalancer>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
```

It will also output a message for the woker nodes

```
kubeadm join <loadbalancer>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

Next, install a CNI compliant SDN. I used Calico since it was the easiest (always look to [the doc](https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises) for the latest yamls). First wget them

```
curl -O https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```

Edit the `calico.yaml` file to use the `podSubnet` you defined in the `kubeadm-config.yaml` file

```
sed -i 's/192\.168/10\.254/g' calico.yaml
```

Now load these into kubernetes

```
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl apply -f calico.yaml
```

Wait a few minutes and you should see CoreDNS and the Calico pods up

```
kubectl get pods --all-namespaces
NAMESPACE     NAME                                             READY   STATUS    RESTARTS   AGE
kube-system   calico-node-g6s98                                2/2     Running   0          33s
kube-system   coredns-86c58d9df4-gq6n8                         1/1     Running   0          15m
kube-system   coredns-86c58d9df4-pl6m6                         1/1     Running   0          15m
kube-system   etcd-dhcp-host-98.cloud.chx                      1/1     Running   0          14m
kube-system   kube-apiserver-dhcp-host-98.cloud.chx            1/1     Running   0          14m
kube-system   kube-controller-manager-dhcp-host-98.cloud.chx   1/1     Running   0          14m
kube-system   kube-proxy-6dn5z                                 1/1     Running   0          15m
kube-system   kube-scheduler-dhcp-host-98.cloud.chx            1/1     Running   0          13m
```
# Bootstrap Remaining Controllers

After the SDN is installed; this is the point you can boostrap the remaining controllers. Use the `kubeadm join ...` command that you saved before. (on hosts called `controller-1` and `controller-2`)

```
ansible controllers -l contoller-X -m shell -a "kubeadm join <lb>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <key>"
```

Once that finishes; you can see them listed with `kubectl`

```
kubectl get nodes
NAME                     STATUS   ROLES    AGE     VERSION
dhcp-host-5.cloud.chx    Ready    master   3m34s   v1.13.1
dhcp-host-98.cloud.chx   Ready    master   34m     v1.13.1
dhcp-host-99.cloud.chx   Ready    master   4m52s   v1.13.1
```

# Bootstrap Worker Nodes

This part is the easiest; you just run the `kubeadm join ...` command WITHOUT the `--control-plane` flag


On the 3 workers run the following
```
kubeadm join <lb ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Once they have joined; get it with `kubectl get nodes`


```
[root@jumpbox k8s-with-kubeadm]# kubectl get nodes
NAME                     STATUS   ROLES    AGE     VERSION
dhcp-host-5.cloud.chx    Ready    master   9m32s   v1.13.1
dhcp-host-6.cloud.chx    Ready    <none>   53s     v1.13.1
dhcp-host-7.cloud.chx    Ready    <none>   53s     v1.13.1
dhcp-host-8.cloud.chx    Ready    <none>   54s     v1.13.1
dhcp-host-98.cloud.chx   Ready    master   40m     v1.13.1
dhcp-host-99.cloud.chx   Ready    master   10m     v1.13.1
```

Label them as workers with this handy "one liner"

```
kubectl  get nodes --no-headers -l '!node-role.kubernetes.io/master' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}' | xargs -I{} kubectl label node {} node-role.kubernetes.io/worker=''
```

# Smoke Test

Test your cluster by deploying a pod with a `NodePort` definition.

First create a namespace

```
kubectl create namespace test
```

Next, create a deployment with a test image

```
kubectl create deployment welcome-php -n test --image=quay.io/redhatworkshops/welcome-php
```

Expose the deployment to create a service to this pod

```
kubectl expose deployment/welcome-php -n test --port=8080 --target-port=8080
```

Check to see if your pods and svc is up

```
$ kubectl get pods -n test
NAME                           READY   STATUS    RESTARTS   AGE
welcome-php-77f6d8845b-n2g92   1/1     Running   0          12s

$ kubectl get svc -n test
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
welcome-php   ClusterIP   172.30.110.181   <none>        8080/TCP   38s
```

Curling the svc address should get you that 200

```
$ curl -sI 172.30.110.181:8080
HTTP/1.1 200 OK
Date: Tue, 15 Jan 2019 16:10:11 GMT
Server: Apache/2.4.27 (Red Hat) OpenSSL/1.0.1e-fips
Content-Type: text/html; charset=UTF-8
```

Congrats! You have an HA k8s cluster up and running!


# Ingress

This is "high level" notes. I did the instlal with `helm`

```shell
! which helm >/dev/null 2>&1 && echo "ERROR: Helm is not installed"
```

First, I initialized `helm`

```shell
helm repo add haproxy-ingress https://haproxy-ingress.github.io/charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
```

Then I labeled one of my nodes as the "ingress host" (can be any worker)

```shell
kubectl label node dhcp-host-5.cloud.chx haproxy=ingresshost
```
Next, I installed using `helm`

```shell
helm install haproxy-ingress haproxy-ingress/haproxy-ingress --create-namespace --namespace=ingress-controller \
--set controller.hostNetwork=true --set=controller.nodeSelector.haproxy=ingresshost
```

You can wait for ingress rollout

```shell
kubectl rollout status deploy haproxy-ingress -n ingress-controller
```

Deploy a sample application with an ingress (I have an example [here](https://raw.githubusercontent.com/christianh814/kubernetes-toolbox/master/resources/examples/welcome-php.yaml) that you can modify and use)

```shell
kubectl apply -f welcome-php.yaml
```

Wait for the rollout

```shell
kubectl rollout status deployment welcome-php -n test
```

Sample welcome-php ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations: {}
  name: welcome-php-ingress
  namespace: test
spec:
  rules:
  - host: welcome-php.127.0.0.1.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: welcome-php
            port:
              number: 8080
```

What I needed for argocd

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    ingress.kubernetes.io/force-ssl-redirect: "true"
    ingress.kubernetes.io/ssl-redirect: "true"
    ingress.kubernetes.io/ssl-passthrough: "true"
  name: argocd-ingress
  namespace: argocd
spec:
  rules:
  - host: argocd.127.0.0.1.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 8080
```

# Miscellaneous Notes

Notes in no paticular order

## Get join token

This is specifically for kubeadm, First create the token

```
kubeadm token create --ttl 90m --print-join-command
```

If you need to, get the hash from the ca cert (the `--print-join-command` should have done this for you however)

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

This is all you need in order to join using the `kubeadm join --token <token> <lb>:<port> --discovery-token-ca-cert-hash sha256:<hash>` syntax. You can list "join tokens" with

```
kubeadm token list
```

If you want to create one for a master run this on your first master (the one where you ran `kubeadm` the first time)

```
kubeadm init phase upload-certs --upload-certs
```

This will output the following

```
[upload-certs] Storing the certificates in ConfigMap "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
9555b74008f24687eb964bd90a164ecb5760a89481d9c55a77c129b7db438168
```

Then you can use the combination of the both outputs to join a new master

## Using nodeSelector

I used `nodeSelector` under the `.spec.template.spec` path on the `ingress` namespace. Below is an example using my ingress controller but you can do this for any app.

```
# kubectl get pods nginx-ingress-controller-59bcb6c455-bjvn7 -n ingress -o wide
NAME                                        READY   STATUS    RESTARTS   AGE   IP           NODE                    NOMINATED NODE   READINESS GATES
nginx-ingress-controller-59bcb6c455-bjvn7   1/1     Running   0          15m   10.254.3.7   dhcp-host-8.cloud.chx   <none>           <none>
```

^ my controller is running on `dhcp-host-8.cloud.chx` let's keep it there...


```
# kubectl get nodes -l kubernetes.io/hostname=dhcp-host-8.cloud.chx
NAME                    STATUS   ROLES    AGE    VERSION
dhcp-host-8.cloud.chx   Ready    <none>   2d1h   v1.13.1
```

^ Based on this... `kubernetes.io/hostname=dhcp-host-8.cloud.chx` seems like a good choice for the `key=value` to use.


I edited the `deployment` and added `nodeSelector` under `.spec.template.spec` snippet below...

```
[root@jumpbox ~]# kubectl get deployments nginx-ingress-controller -n ingress -o yaml | grep -C 10 containers
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx-ingress-lb
    spec:
      #-- I added this section here -- #
      nodeSelector:
        kubernetes.io/hostname: dhcp-host-8.cloud.chx
      # ----------------------------- #
      containers:
      - args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/default-backend
        - --configmap=$(POD_NAMESPACE)/nginx-ingress-controller-conf
        - --v=2
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
```

Now this pod will always be on this host that's my "infra" host

```
# kubectl get pods -n ingress nginx-ingress-controller-59bcb6c455-bjvn7 -o yaml | grep -A 1 nodeSelect
  nodeSelector:
    kubernetes.io/hostname: dhcp-host-8.cloud.chx
```

## Switching context

To see what context you're using (which "namespace" you're on) run

```
# kubectl config get-contexts 
CURRENT   NAME                          CLUSTER             AUTHINFO             NAMESPACE
*         default/192-168-1-97:6443/    192-168-1-97:6443   /192-168-1-97:6443   default
          ingress/192-168-1-97:6443/    192-168-1-97:6443   /192-168-1-97:6443   ingress
          kubernetes-admin@kubernetes   kubernetes          kubernetes-admin     
          test/192-168-1-97:6443/       192-168-1-97:6443   /192-168-1-97:6443   test
```

This shows me the current context I'm in. I am using server `default/192-168-1-97:6443/` and on namespace `default`. To switch to another project (say the `ingress` project)...

```
# kubectl config set-context --current --namespace=ingress
Context "default/192-168-1-97:6443/" modified.
```

Now every `kubectl` command will be in the context of the `ingress` namespace. To verify this run

```
# kubectl config get-contexts 
CURRENT   NAME                          CLUSTER             AUTHINFO             NAMESPACE
*         default/192-168-1-97:6443/    192-168-1-97:6443   /192-168-1-97:6443   ingress
          ingress/192-168-1-97:6443/    192-168-1-97:6443   /192-168-1-97:6443   ingress
          kubernetes-admin@kubernetes   kubernetes          kubernetes-admin     
          test/192-168-1-97:6443/       192-168-1-97:6443   /192-168-1-97:6443   test
```

As you see the `NAMESPACE` feild says I'm in `ingress` now.

To switch back and verify

```
# kubectl config set-context --current --namespace=default
Context "default/192-168-1-97:6443/" modified.

# kubectl config get-contexts 
CURRENT   NAME                          CLUSTER             AUTHINFO             NAMESPACE
*         default/192-168-1-97:6443/    192-168-1-97:6443   /192-168-1-97:6443   default
          ingress/192-168-1-97:6443/    192-168-1-97:6443   /192-168-1-97:6443   ingress
          kubernetes-admin@kubernetes   kubernetes          kubernetes-admin     
          test/192-168-1-97:6443/       192-168-1-97:6443   /192-168-1-97:6443   test
```

## Create Users

This is HIGH high level (more to come soon). First create a ns for this user

```
kubectl create ns development
```

View the clusters and contexts. This allows you to config the cluster, namespace and user for `kubectl` commands

```
kubectl config get-contexts
```

Generate a private key and a CSR for the user

```
openssl genrsa -out devman.key 2048
openssl req -new -key devman.key -out devman.csr -subj "/CN=devman/O=development"
```

Use the CA keys for the Kubernetes cluster and set a 45 day expiration

```
openssl x509 -req -in devman.csr -CA ./ca.crt -CAkey ./ca.key -CAcreateserial -out devman.crt -days 45
```

Set the `~/.kube/config` file for the user

```
kubectl config set-credentials devman --client-certificate=devman.crt --client-key=devman.key
kubectl config set-context devman --cluster=kluster.chx.osecloud.com --namespace=development --user=devman
```

Check the access

```
kubectl --context=devman get pods
```

Output should be
> Error from server (Forbidden): pods is forbidden: User "devman" cannot list pods in the namespace "development"

Now create a YAML file to associate RBAC

```
cat <<EOF > role-dev.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: [ "" , "extensions" , "apps" ]
  resources: [ "deployments" , "replicasets" , "pods" ]
  verbs: [ "*" ]
EOF
```

Create the RBAC object

```
kubectl create -f role-dev.yaml
```

Create the rolebinding yaml

```
cat <<EOF > rolebind.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: developer-role-binding
  namespace: development
subjects:
- kind: User
  name: devman
  apiGroup: ""
roleRef:
  kind: Role
  name: developer
  apiGroup: ""
EOF
```

Create the rolebinding object

```
kubectl create -f rolebind.yaml
```

Test the binding/rbac

```
kubectl --context=devman create deployment nginx --image=nginx
kubectl --context=devman get pods
```

**NOTE** I lifted and created a shellscript to do this "automagically" for me [here](resources/examples/mk-k8s-user.sh)

More notes tk

## Helm

Helm helps you manage Kubernetes applications. Helm Charts helps you define, install, and upgrade complex Kubernetes applications.

This is a "quick start" (for more complex/secure setups visit the [official docs](https://docs.helm.sh/using_helm))


First install the latest [stable release](https://github.com/helm/helm/releases)


```
wget https://raw.githubusercontent.com/helm/helm/master/scripts/get -O get-helm.sh
bash get-helm.sh
```

Set up bash completion for easier use

```
source <(helm completion bash)
```

Verify it's installed

```
helm version
```

Now install a sample chart. First, get a fresh list of the repo

```
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Now let's install a sample helm chart of mysql

```
kubectl create namespace test-mysql
helm install stable/mysql --namespace test-mysql
```

See the status of what you have deployed

```
helm  ls
```

Deleting is done by name so first get the name and delete it based on that name

```
helm ls --short
helm delete deadly-dachshund
```

Get the status

```
helm status deadly-dachshund
```

## Taint

You `taint` a node in order to contol how pods get deployed

* `<key>=value:PreferNoSchedule` - a node tainted `PreferNoSchedule` will only schedule pods when other nodes can't
* `<key>=value:NoSchedule` - will never schedule a pod unless a `toleration` is in place
* `<key>=value:NoExecute` - Pods that do not tolerate the taint are evicted immediately. Pods that **DO** tolerate the taint without specifying `tolerationSeconds` in their toleration specification remain bound forever And pods that **DO** tolerate the taint **BUT** with a specified `tolerationSeconds` remain bound for the specified amount of time
* In all instances `<key>` can be whatever you want

Examples:

```
kubectl taint node mynode.example.com foo=value:PreferNoSchedule
kubectl taint node mynode.example.com foo=value:NoSchedule
kubectl taint node mynode.example.com foo=value:NoExecute
```

To remove a taint

```
kubectl taint node mynode.example.com foo-
```

## Monitoring

> This assumes that you have a dynamic storageclass created

Again, helm is your buddy here, first create a namespace for your monitoring stack

```
kubectl create ns monitoring
```

Install Prometheus

```
helm install prometheus stable/prometheus --version 6.7.4 --namespace monitoring
```

Next, I create a config file for grafana (based on the above prometheus install...my service name will be `prometheus-server`...yours may differ if you're not following allong exactly)

```
cat <<EOF > grafana-values.yaml
persistence:
  enabled: true
  accessModes:
    - ReadWriteOnce
  size: 5Gi

datasources: 
 datasources.yaml:
   apiVersion: 1
   datasources:
   - name: Prometheus
     type: prometheus
     url: http://prometheus-server
     access: proxy
     isDefault: true

dashboards:
    kube-dash:
      gnetId: 6663
      revision: 1
      datasource: Prometheus
    kube-official-dash:
      gnetId: 2
      revision: 1
      datasource: Prometheus

dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
    - name: 'default'
      orgId: 1
      folder: ''
      type: file
      disableDeletion: false
      editable: true
      options:
        path: /var/lib/grafana/dashboards
EOF
```

Once I have that, I install grafana

```
helm install  grafana stable/grafana --version 1.11.6 --namespace monitoring -f grafana-values.yaml
```

Create an ingress for grafana (note I'm using [TLS](resources/documents/k8s-ingress-helm.md#tls) here...you don't NEED to but it's recommended)

```
cat <<EOF | kubectl -n monitoring create -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/tls-acme: "true"
    certmanager.k8s.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: "nginx"
  name: grafana
spec:
  tls:
  - hosts:
    - grafana.apps.example.com
    secretName: grafana-tls
  rules:
  - host: grafana.apps.example.com
    http:
      paths:
      - backend:
          serviceName: grafana
          servicePort: 80
        path: /
EOF
```

Log in with the following creds...

* `admin` - this is the default username
*  This is how you get the password

```
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Success! See [this blog post](https://rohanc.me/monitoring-kubernetes-prometheus-grafana/) for more info

## K8S Oneliners

Quick notes that don't require it's own category

__Create a pod ONLY (i.e no deployment/ds/etc) with nodeport only__

```
kubectl run --generator=run-pod/v1  nginx --image=nginx  -l app=nginx
kubectl create service nodeport nginx --node-port=32000 --tcp=80
```

^ Sometimes the port mapping doesn't work right...just `kubectl edit svc...` to change it to what you want.

__Init Containers__

This example is, in part, taken from the [official documentation](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#init-containers-in-use). Init containers basically run and exit before the actual containers run

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: alpine
    command: ['/bin/sh', '-c', '[[ -f /workdir/gentle.txt ]] && sleep 36000 || exit 1']
    volumeMounts:
    - mountPath: /workdir
      name: workdir
  initContainers:
  - name: init-myservice
    image: alpine
    command: ['/bin/sh', '-c', 'touch -f /workdir/gentle.txt']
    volumeMounts:
    - mountPath: /workdir
      name: workdir
  volumes:
  - name: workdir
    emptyDir: {}
```

This example also uses a volume (note; the volume needs to exists on BOTH if you want this paticular example to work for you)

__Pod DNS__

Create a deployment like this

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      hostname: nginx-1
      subdomain: nginx-svc
      containers:
      - image: nginx
        name: nginx
        resources: {}
```

Next after you `kubectl -f...` that bad boy; create a service name with the **SAME** name as the `subdomain` as up top

```
kubectl expose deployment nginx --name=nginx-svc --port=80 --target-port=80
```

Now you can nslookup both `<svc>.<namespace>.svc.cluster.local` and `<pod hostname from yaml>.<subdomain defined in yaml>.<namespace>.svc.cluster.local`

__Rollback Deployments__

Create your deployment

```
kubectl create deployment nginx --image=nginx:1.7.9
```

Make a change, making sure to `--record` it

```
kubectl set image deploy/nginx nginx=nginx:1.9.1 --record=true
```

Make sure your change rolledouot

```
kubectl describe pods
```

List your rollouts

```
kubectl rollout history deploy/nginx
```

Rollback to the version you want

```
kubectl rollout undo deploy/nginx --to-revision=1
```

__Gain Root__

Test your PSP with the following

```
kubectl run r00t --restart=Never -ti --rm --image lol --overrides '{"spec":{"hostPID": true, "containers":[{"name":"1","image":"alpine","command":["nsenter","--mount=/proc/1/ns/mnt","--","/bin/bash"],"stdin": true,"tty":true,"securityContext":{"privileged":true}}]}}'
```

