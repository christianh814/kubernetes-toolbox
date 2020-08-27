# Create a kind config that exposes the proper ports

```
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
nodes:
- role: control-plane
- role: worker
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    listenAddress: 0.0.0.0
  - containerPort: 443
    hostPort: 443
    listenAddress: 0.0.0.0
```

# Start kind with this config

```
kind create cluster --config=config.yaml
```

# Make sure any workers are properly labeled

```
kubectl  get nodes --no-headers -l '!node-role.kubernetes.io/master' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}' | xargs -I{} kubectl label node {} node-role.kubernetes.io/worker=''
```

# Label the worker to be able to run the nginx ingress

```
kubectl label node kind-worker nginx=ingresshost
```

# Create a namespace for the ingress controller

```
kubectl create ns ingress-nginx
```

# Set up helm

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

# Install ingress with helm

```
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --set controller.nodeSelector.nginx="ingresshost" \
--set rbac.create=true --set controller.image.pullPolicy="Always" --set controller.extraArgs.enable-ssl-passthrough="" \
--set controller.stats.enabled=true --set controller.service.type="ClusterIP" \
--set controller.kind="DaemonSet" --set controller.daemonset.useHostPort=true
```

# Wait for ingress rollout

```
kubectl rollout status ds ingress-nginx-nginx-ingress-controller -n ingress-nginx
kubectl rollout status deploy ingress-nginx-nginx-ingress-default-backend -n ingress-nginx
```

# Create a namespace to test

```
kubectl create ns test
```

# Create deployment in this namespace

```
kubectl create deployment welcome-php --image=quay.io/redhatworkshops/welcome-php:latest -n test
```

# Create a service for the deployment

```
kubectl expose deployment welcome-php --port=8080 --target-port=8080 -n test
```

# Create the ingress yaml

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: welcome-php-ingress
  namespace: test
spec:
  rules:
  - host: welcome-php.127.0.0.1.nip.io
    http:
      paths:
      - backend:
          serviceName: welcome-php
          servicePort: 8080
        path: /
```

# Create the ingress object

```
kubectl create -f ing.yaml -n test
```

# Test

```
firefox welcome-php.127.0.0.1.nip.io
```
