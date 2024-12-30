---
description: Installation and usage of falco inside kubernetes
---

# Falco real time runtime thread detection on k8s

## Installation



Install falco helm repository

```
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

Deploy falco using helm.

In this demo we also enabled the graphical dashboard UI

```
helm upgrade --install \
  --namespace falco \
  --create-namespace \
  falco falcosecurity/falco \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true
```

Wait for the pods to be ready

Create a demo deployment

```
kubectl create deployment nginx --image=nginx
```

Once pods are ready execute a call in /etc/shadow file which will trigger a warning in falco

```
kubectl exec -it $(kubectl get pods --selector=app=nginx -o name) -- cat /etc/shadow
```

Verify that the warning has been triggered

```
kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco | egrep -i warning
{"hostname":"minikube","output":"15:41:04.268296715: Warning Sensitive file opened for reading by non-trusted program (file=/etc/shadow gparent=systemd ggparent=<NA> gggparent=<NA> evt_type=openat user=root user_uid=0 user_loginuid=-1 process=cat proc_exepath=/usr/bin/cat parent=containerd-shim command=cat /etc/shadow terminal=34816 container_id=ff95ee645d8a container_image=nginx container_image_tag=latest container_name=k8s_nginx_nginx-676b6c5bbc-m86bn_default_fda2eefb-4c21-4a46-ac55-bcdbfc58936b_0 k8s_ns=<NA> k8s_pod_name=<NA>)","output_fields":{"container.id":"ff95ee645d8a","container.image.repository":"nginx","container.image.tag":"latest","container.name":"k8s_nginx_nginx-676b6c5bbc-m86bn_default_fda2eefb-4c21-4a46-ac55-bcdbfc58936b_0","evt.time":1735573264268296715,"evt.type":"openat","fd.name":"/etc/shadow","k8s.ns.name":null,"k8s.pod.name":null,"proc.aname[2]":"systemd","proc.aname[3]":null,"proc.aname[4]":null,"proc.cmdline":"cat /etc/shadow","proc.exepath":"/usr/bin/cat","proc.name":"cat","proc.pname":"containerd-shim","proc.tty":34816,"user.loginuid":-1,"user.name":"root","user.uid":0},"priority":"Warning","rule":"Read sensitive file untrusted","source":"syscall","tags":["T1555","container","filesystem","host","maturity_stable","mitre_credential_access"],"time":"2024-12-30T15:41:04.268296715Z"}
```

## Ingress



To access the web UI you can port-forward the service or create an ingress.

We will create an ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: falco-ingress
spec:
  rules:
    - host: "falco.192.168.39.115.nip.io"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: falco-falcosidekick-ui
                port:
                  number: 2802
```

Create the ingress in the falco namespace

```
kubectl apply -f falco-ingress.yaml -n falco
```

In my example you can access the dashboard using http://falco.192.168.39.115.nip.io

Username admin and password admin by default.

## Custom rules

Create a file with your custom rules `falco_custom_rules_cm.yaml`

Here is an example to trigger alert when a container executes id command

Custom rules documentation [https://falco.org/docs/rules/basic-elements/](https://falco.org/docs/rules/basic-elements/)

```yaml
customRules:
  custom-rules.yaml: |-
    - rule: id_usage
      desc: id usage
      condition: >
        evt.type = execve and
        evt.dir = < and
        container.id != host and 
        proc.name = id    
      output: >
        id command is used 
        (user=%user.name container_id=%container.id container_name=%container.name 
        shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)    
      priority: CRITICAL
```

Update the helm with custom rules

<pre><code>helm upgrade --namespace falco falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
<strong>  --set tty=true \
</strong><strong>  --set falcosidekick.enabled=true \
</strong>  --set falcosidekick.webui.enabled=true \
  -f falco_custom_rules_cm.yaml
</code></pre>

Trigger warning

```
kubectl exec -it $(kubectl get pods --selector=app=nginx -o name) -- id
uid=0(root) gid=0(root) groups=0(root)
```

Check falco logs to verify alert is triggered

```
kubectl logs -l app.kubernetes.io/name=falco -n falco -c falco -f | egrep -i nginx

17:11:28.144420277: Critical id command is used  (user=root container_id=4720698b7671 container_name=k8s_nginx_nginx-676b6c5bbc-2xrkd_default_deae5c43-68c3-49c5-b6fc-15ed60125834_0  shell=id parent=runc cmdline=id) container_id=4720698b7671 container_image=nginx container_image_tag=latest container_name=k8s_nginx_nginx-676b6c5bbc-2xrkd_default_deae5c43-68c3-49c5-b6fc-15ed60125834_0 k8s_ns=<NA> k8s_pod_name=<NA>
```
