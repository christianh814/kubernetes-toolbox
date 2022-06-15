# Ingress install with Helm

This is a QnD method but with Helm

You need to first decide where to put it, label your nodes something that you can use in your selector

```
# kubectl label node dhcp-host-8.cloud.chx nginx=ingresshost
node/dhcp-host-8.cloud.chx labeled

# kubectl get nodes -l nginx=ingresshost
NAME                    STATUS   ROLES    AGE   VERSION
dhcp-host-8.cloud.chx   Ready    <none>   10d   v1.13.1
```

The IP for this node is `192.168.1.8`

```
# dig dhcp-host-8.cloud.chx +short
192.168.1.8
```

I want to install this in the `ingress` namespace; so I'll create it

```
kubectl create namespace ingress
```

If you haven't already, this is where you would ini helm. [Instuctions here](../../README.md#helm)

**NOW...**

Using this information; I will deploy the `nginx-ingress` helm chart; giving it the name `nginx-ingress`. List of all options can be found on the [github page](https://github.com/helm/charts/tree/master/stable/nginx-ingress#configuration)

```
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress \
--set rbac.create=true --set controller.image.pullPolicy="Always" --set controller.extraArgs.enable-ssl-passthrough="" \
--set controller.nodeSelector.nginx="ingresshost" --set controller.stats.enabled=true \
--set controller.service.externalIPs={192.168.1.8} --set controller.service.type="ClusterIP"
```

^ Pay close attention to `controller.nodeSelector`...the syntax is `controller.nodeSelector.<key>="<value>"` ...note the use of the dot instead of an `=`. Also note that `controller.service.externalIPs` is an array



Export the stats page if you wish (make sure the `svc` name and the `port`are right)

```
cat <<EOF | kubectl apply -n ingress -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: nginx.192.168.1.8.nip.io
    http:
      paths:
      - backend:
          serviceName: nginx-ingress-controller-stats
          servicePort: 18080
        path: /nginx_status
EOF
```

# TLS

The following sets up `nginx-ingress` with `cert-bot` for getting/updating ssl certs using [letsencrypt](https://letsencrypt.org/). Note the following two "gotchas"

* These steps worked on a cluster installed with [capi](https://cluster-api.sigs.k8s.io/user/quick-start.html)
* These steps worked on an "internal" cluster with the "infra" node exposed to the internet on `80` and `443`

# Install Nginx Ingress

**LOCAL**

If you're installing nginx "internall" the [above installation with helm](#ingress-install-with-helm) should work just fine. Just pay attention to your `nodeSelector` and you're targeting your "exposed" node.

**CLOUD** 

After installing with [capi](https://cluster-api.sigs.k8s.io/user/quick-start.html), I did an `init` with [helm](../../README.md#helm). Then I did a simple install like this

```
helm install nginx-ingress ingress-nginx/ingress-nginx \
--namespace ingress --set rbac.create=true \
--set controller.extraArgs.enable-ssl-passthrough="" --set controller.image.pullPolicy="Always"
```

The above automatically creates a Network `ELB` on `80` and `443` passing through to the NGINX ingress controller (that then sends that traffic to the pods). I took the `ELB` DNS name...

```
$ kubectl get svc nginx-ingress-controller -n ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}{"\n"}'
```
Then I created a Route53 wildcard entry `*.apps.my.cluster.com` that CNAMEs to that ELB.

## Cert Manager

First thing to do, is use `helm` to install it

```
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
$ helm install cert-manager jetstack/cert-manager --create-namespace --namespace cert-manager \
--set installCRDs=true \
--set ingressShim.defaultIssuerName=letsencrypt-prod \
--set ingressShim.defaultIssuerKind=ClusterIssuer \
--set ingressShim.defaultIssuerGroup=cert-manager.io
```

> FOLLOW THIS: https://cert-manager.io/docs/installation/kubernetes/

Create a  `cluster issuer` yaml. Replace the email with a valid email address

```
# cat <<EOF > clusterissuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: christian@ctest.io
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

Create this in your env

```
kubectl create -f clusterissuer.yaml
```

Create your ingress config yaml

```
# cat <<EOF > welcome-php-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/tls-acme: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  name: welcome-php-ingress
  namespace: test
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - welcome-php.test.domain
    secretName: welcome-php-tls
  rules:
  - host: welcome-php.test.domain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: welcome-php
            port:
              number: 8080
EOF
```

^ **Note** the annotations! Get the `cluster-issuer` name with `kubectl get clusterissuer`

Create the ingress object for the app

```
# kubectl create -n test -f welcome-php-ingress.yaml
```

Viola! You have a valid cert with letsencrypt (that will autorenew too!)
