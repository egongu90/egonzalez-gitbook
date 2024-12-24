# Gerrit and gitlab replication and CI job hooks on k8s

Create repos at both gitlab and gerrit

Generate a token (admin) in gitlab (not sure if this is needed at all since the hook is with a different token)

Create secret for gerrit replication

```
secure.txt                 
[remote "gitlab"]
password = "GY50BPkhK1YVV4ND0PSPQAhPJ7FqIsc4EI7YveoivLVI00tqj7bYiIYYbot0ZovF"



cat secure.txt| base64 ; echo
W3JlbW90ZSAiZ2l0bGFiIl0KcGFzc3dvcmQgPSAiR1k1MEJQa2hLMVlWVjRORDBQU1BRQWhQSjdGcUlzYzRFSTdZdmVvaXZMVkkwMHRxajdiWWlJWVlib3QwWm92RiIK

```

Contents of the secret yaml file, replace with your own secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name:  gerrit-secure-config
  namespace: gerrit
  labels:
    app: gerrit
data:
  ssh_host_ecdsa_key: |
    LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFBQUFBQkc1dmJtVUFBQUFFYm05dVpRQUFBQUFBQUFBQkFBQUFhQUFBQUJObFkyUnpZUwoxemFHRXlMVzVwYzNSd01qVTJBQUFBQ0c1cGMzUndNalUyQUFBQVFRUmNZWUNCZnVxczd3d2Q2amN5a0J4NXZ0QjRrSkp2CmxtbnlMS2EwbEZ1L1BpbVNUbmdUcXBRM3d5bHFsWEtLZ2ZsbzJyWkQzRCtkZGRFNUNxRXBTZDVOQUFBQXNFUGsvY0ZENVAKM0JBQUFBRTJWalpITmhMWE5vWVRJdGJtbHpkSEF5TlRZQUFBQUlibWx6ZEhBeU5UWUFBQUJCQkZ4aGdJRis2cXp2REIzcQpOektRSEhtKzBIaVFrbStXYWZJc3ByU1VXNzgrS1pKT2VCT3FsRGZES1dxVmNvcUIrV2phdGtQY1A1MTEwVGtLb1NsSjNrCjBBQUFBaEFLZC9IY3g4RlZkM3JPQ2J4ODFmWUxYeGFKOWc2dk1QWXRNdUFRb3E2YkI0QUFBQUVXczRjMmRsY25KcGRDMWwKZUdGdGNHeGxBUUlEQkFVRwotLS0tLUVORCBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0=
  ssh_host_ecdsa_key.pub: |
    ZWNkc2Etc2hhMi1uaXN0cDI1NiBBQUFBRTJWalpITmhMWE5vWVRJdGJtbHpkSEF5TlRZQUFBQUlibWx6ZEhBeU5UWUFBQUJCQkZ4aGdJRis2cXp2REIzcU56S1FISG0rMEhpUWttK1dhZklzcHJTVVc3OCtLWkpPZUJPcWxEZkRLV3FWY29xQitXamF0a1BjUDUxMTBUa0tvU2xKM2swPSBrOHNnZXJyaXQtZXhhbXBsZQ==
  secure.txt: |
    W3JlbW90ZSAiZ2l0bGFiIl0KcGFzc3dvcmQgPSAiSDdaY3lyWEhJa1ZXMHZCcElxTjFGVVhXRmhnQ2Y5alM3aTdCNEdzUzFKYWJuSTdmcTZWblpvWXhmb1I4dVV3biIK
type: Opaque
```

Update cluster deployment with the follow config files and include replication plugin

```ini
      plugins:
        - name: download-commands
        - name: delete-project
        - name: replication
      configFiles:
          gitlab.config: |-
            [gitlab]
            url = https://gitlab.192.168.39.219.nip.io
            gerritUser = admin
            token = glpat-kvN7HW9x5RFjntkhw-zC
            recheckCommand = recheck
        replication.config: |-
            [remote "gitlab"]
              projects = testrepo
              url = https://root@gitlab.192.168.39.219.nip.io/root/${name}.git 
              push = +refs/heads/*:refs/heads/*
              push = +refs/tags/*:refs/tags/*
              timeout = 30
              threads = 3
              mirror = true
              replicatePermissions = false
              rescheduleDelay = 15
            [replication]
              lockErrorMaxRetries = 5
              maxRetries = 5
```

Create .gitreview for new repo, example for https, if have ssh better

```
[gerrit]
host=gerrit.192.168.39.219.nip.io
port=80
project=root/testrepo.git
defaultbranch=master
```

Configure remote if http errors

```ini
[remote "gerrit"]$
     url = http://admin@gerrit.192.168.39.219.nip.io/a/testrepo$
     fetch = +refs/heads/*:refs/remotes/origin/*$
```

WIP: Configure ssh key to gerrit push to gitlab (NOTE: pass ssh config as volume)

```
kubectl exec -n gerrit gerrit-0 -- /bin/sh -c 'echo "-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
...
hKtHHsTTdhvlQDAAAAHnJvY2t5QG5zMzEwMzc2NC5pcC01NC0zNy04NC5ldQECAwQ=
-----END OPENSSH PRIVATE KEY-----" > /home/gerrit/.ssh/id_rsa'
```

```
kubectl exec -n gerrit gerrit-0 -- /bin/sh -c "echo 'StrictHostKeyChecking no' > /home/gerrit/.ssh/config"
```

Replication logs

```
kubectl exec -n gerrit gerrit-0 -- tail -f -n100 /var/gerrit/logs/replication_log
```

For the hooks to work firsts create a secret with the code of the hooks, here is an example of the hook with curl. Do any language you want.

Get the token at `https://gitlab.192.168.39.102.nip.io/root/test/-/settings/ci_cd#js-pipeline-triggers`

```
cat hook.txt
#!/bin/sh
curl -k -s -X POST --fail \
-F token=glptt-f56abb7c111f3a12cb832676170594397fcddd40 \
-F ref=main https://gitlab.192.168.39.102.nip.io/api/v4/projects/1/trigger/pipeline
```

Base64 the script

```
cat hook.txt | base64 ; echo
```

Copy contents of the base64 encoded object and place it into a secret, name should match one of the gerrit supported hooks name, in this case comment-added, change-merged or all the supported hooks you want to add

```
apiVersion: v1
kind: Secret
metadata:
  name:  hooks
  namespace: gerrit
  labels:
    app: gerrit
data:
  comment-added: |
      IyEvYmluL3NoCmN1cmwgLWsgLXMgLVggUE9TVCAtLWZhaWwgLUYgdG9rZW49Z2xwdHQtZjU2YWJiN2MxMTFmM2ExMmNiODMyNjc2MTcwNTk0Mzk3ZmNkZGQ0MCAtRiByZWY9bWFpbiBodHRwczovL2dpdGxhYi4xOTIuMTY4LjM5LjEwMi5uaXAuaW8vYXBpL3Y0L3Byb2plY3RzLzEvdHJpZ2dlci9waXBlbGluZQo=
type: Opaque
```

Apply the secret and configure the cluster yaml definition.

Add a new plugin with the hooks

<pre class="language-yaml"><code class="lang-yaml"><strong>- name: hooks
</strong>  data:
    secretRef: hooks
    executable: true
</code></pre>

Add into gerrit.config the hook path

```ini
[hooks]
  path = /var/mnt/data/hooks
```

Apply the cluster change, wait for the pod to spawn and test the integration adding a comment in the gerrit review patch.

In gitlab a new pipeline will be triggered
