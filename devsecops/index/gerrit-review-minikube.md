---
description: Deploy gerrit on kubernetes
---

# Gerrit review minikube

Download git repository with helm charts

```
git clone https://gerrit.googlesource.com/k8s-gerrit
cd k8s-gerrit
```

Create namespace

```
kubectl create ns gerrit-operator
```

Update helm dependencies

```
helm dependency build helm-charts/gerrit-operator/
```

Install k8s operator

```
helm -n gerrit-operator install gerrit-operator helm-charts/gerrit-operator/
```

Install NFS

```
kubectl create ns nfs
helm repo add nfs-ganesha-server-and-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-ganesha-server-and-external-provisioner/
helm upgrade \
  --install nfs \
  nfs-ganesha-server-and-external-provisioner/nfs-server-provisioner \
  -n nfs
```

Create gerrit namespace and sample secrets

```
kubectl create ns gerrit
kubectl apply -f Documentation/examples/gerrit.secret.yaml
```

Create single gerrit cluster

```
kubectl apply -f Documentation/examples/1-gerritcluster.yaml
```

Create gerrit-ingress.yaml file to generate an ingress to the Web UI.

Host IP is the output of minikube ip command

```yaml
cat << EOF > gerrit-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gerrit-ingress
spec:
  rules:
    - host: "gerrit.$(minikube ip).nip.io"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: gerrit-service
                port:
                  number: 80
EOF
```

Create the ingress

```
 kubectl apply -f gerrit-ingress.yaml -n gerrit
```

Create IngressRouteTcp to allow git through SSH

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRouteTCP
metadata:
  labels:
    app: gerrit-service
  name: gerrit-service
  namespace: gerrit
spec:
  entryPoints:
  - gerrit-service
  routes:
  - match: HostSNI(`*`)
    services:
    - name: gerrit-service
      namespace: gerrit
      port: 29418
```
