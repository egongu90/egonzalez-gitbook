---
description: Vault integration with Gitalb CI to retrieve secrets in jobs pipelines
---

# DRAFT: Vault integration with gitlab CI

Install consul helm

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install consul hashicorp/consul
```

Install vault

```
helm install vault hashicorp/vault
```

