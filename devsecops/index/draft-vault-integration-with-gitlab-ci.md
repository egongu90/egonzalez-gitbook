---
description: Vault integration with Gitalb CI to retrieve secrets in job pipelines
---

# Vault integration with Gitlab CI

Installation is done in kubernetes, if already have gitlab and vault running ignore helm and kubectl steps.

Install consul helm (optional if doing allinone with minikube)

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install consul hashicorp/consul
```

Install vault

```
helm install vault hashicorp/vault
```

Verify vault installation, for now vault is not initialized so pods are 0/1

```
kubectl exec vault-0 -- vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        false
Sealed             true
Total Shares       0
Threshold          0
Unseal Progress    0/0
Unseal Nonce       n/a
Version            1.18.1
Build Date         2024-10-29T14:21:31Z
Storage Type       file
HA Enabled         false
command terminated with exit code 2
```

Initialize vault

```
 kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
```

```
cat cluster-keys.json
{
  "unseal_keys_b64": [
    "EDRPrduCa/VZbBKYEChAk82FxGBfpTw8rYecy24UrwM="
  ],
  "unseal_keys_hex": [
    "10344faddb826bf5596c129810284093cd85c4605fa53c3cad879ccb6e14af03"
  ],
  "unseal_shares": 1,
  "unseal_threshold": 1,
  "recovery_keys_b64": [],
  "recovery_keys_hex": [],
  "recovery_keys_shares": 0,
  "recovery_keys_threshold": 0,
  "root_token": "hvs.Uqwd3Pb6kgi4sMD4L0bnBqRj"
}
```

Note the root\_token for the initial web ui login

Unseal vault

```
kubectl exec vault-0 -- vault operator unseal $(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.18.1
Build Date      2024-10-29T14:21:31Z
Storage Type    file
Cluster Name    vault-cluster-27427646
Cluster ID      b4f4ad08-c258-93b4-c368-3b182ffab753
HA Enabled      false
```

Repeat the same for the other vault-\* pods if existing

Exec into the pod to initialize vault config, use root token from the vault keys file we generated

```
kubectl exec --stdin=true --tty=true vault-0 -- /bin/sh
vault login
```

Enable kv-v2 engine

```
vault secrets enable -path=secret kv-v2
Success! Enabled the kv-v2 secrets engine at: secret/
```

Create demo password

```
vault kv put secret/gitlab/auth username="demo" password="testpass"
===== Secret Path =====
secret/data/gitlab/auth

======= Metadata =======
Key                Value
---                -----
created_time       2024-12-26T13:04:34.241453844Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

Verify secret retrieval

```
vault kv get secret/gitlab/auth
===== Secret Path =====
secret/data/gitlab/auth

======= Metadata =======
Key                Value
---                -----
created_time       2024-12-26T13:04:34.241453844Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key         Value
---         -----
password    testpass
username    demo
```

Create ingress for web access

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-ingress
spec:
  rules:
    - host: "vault.192.168.39.66.nip.io"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: vault
                port:
                  number: 8200
```

```
kubectl apply -f vault-ingress.yaml
```

### Gitlab CI integration

Enable jwt

```
vault auth enable jwt
Success! Enabled jwt auth method at: jwt/
```

Download certificates from gitlab if are self signed

```
openssl s_client -showcerts \
    -connect gitlab.192.168.39.66.nip.io:443 \
    -servername gitlab.192.168.39.66.nip.io < /dev/null 2>/dev/null \
     | openssl x509 -outform PEM > gitlab.crt
```

Generate jwt connection to gitlab

```
vault write -tls-skip-verify auth/jwt/config \
    oidc_discovery_url="https://gitlab.192.168.39.66.nip.io" \
    bound_issuer="gitlab.192.168.39.66.nip.io" \
    oidc_discovery_ca_pem="$(cat gitlab.crt)" \
    jwks_ca_pem="$(cat ./gitlab.crt)"
Success! Data written to: auth/jwt/config
```

Create a policy to read secrets

```
vault policy write demo - <<EOF
# Read-only permission on 'secret/data/project/*' path

path "secret/data/project/*" {
  capabilities = [ "read" ]
}
EOF
```

Create demo role

```
vault write auth/jwt/role/demo - <<EOF
{
  "role_type": "jwt",
  "policies": ["demo"],
  "token_explicit_max_ttl": 60,
  "user_claim": "user_email",
  "bound_audiences": "http://vault.192.168.39.66.nip.io",
  "bound_claims_type": "glob",
  "bound_claims": {
    "namespace_path": "root"
  }
}
EOF
```

Configure in gitlab CI variables

* VAULT\_SERVER\_URL: http://vault-server:8200
* VAULT\_AUTH\_ROLE: demo

Create the demo secret

```
vault kv put secret/project/demo username="demo" password="testpass"
====== Secret Path ======
secret/data/project/demo

======= Metadata =======
Key                Value
---                -----
created_time       2024-12-26T17:12:05.530301004Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

Example gitlab pipeline

```yaml
stages:
  - build
demo-job-with-secrets:
  variables:
    VAULT_SERVER_URL: http://vault.192.168.39.66.nip.io:8200
    VAULT_AUTH_ROLE: demo
  stage: build
  id_tokens:
    VAULT_ID_TOKEN:
      aud: http://vault.192.168.39.66.nip.io
  secrets:
    STAGING_DB_PASSWORD:
      vault: project/demo/password@secret
      file: false
  script:
    - echo $STAGING_DB_PASSWORD > test.txt
    - cat test.txt
```

Output from the job

```
Executing "step_script" stage of the job script 00:01
$ echo $STAGING_DB_PASSWORD > test.txt
$ cat test.txt
[MASKED]
```
