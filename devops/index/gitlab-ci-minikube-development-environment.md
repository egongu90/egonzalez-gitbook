# Gitlab CI minikube development environment

Install minikube

```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

Create minikube machine

```
minikube start --cpus 4 --memory 8192 --addons ingress
```

Download gitlab helm charts repository

```
git clone https://gitlab.com/gitlab-org/charts/gitlab.git
cd gitlab
```

Install Gitlab Helm charts

```
helm dependency update
helm upgrade --install gitlab . \
  --timeout 600s \
  --set global.hosts.domain=$(minikube ip).nip.io \
  --set global.hosts.externalIP=$(minikube ip) \
  -f https://gitlab.com/gitlab-org/charts/gitlab/raw/master/examples/values-minikube-minimum.yaml
```

Default user is root and password can be get with the following command

```
kubectl get secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo
```

Install gitlab helm repository

```
helm repo add gitlab https://charts.gitlab.io
helm repo update
```

Generate `values.yml` with gitlab runner contents.

Registration token can be made in the admin user interface at  `https://$(minikube ip)/admin/runners/new`

Certificate is created by default with the helm deployment name, otherwise download and create a secret or find whats the secret name in k8s

```yaml
gitlabUrl: https://gitlab.192.168.49.2.nip.io
runnerRegistrationToken: "glrt-t1_P1oviNSAj83aiiKXr4UQ"
rbac:
    create: true
runners:
    privileged: true
certsSecretName: gitlab-wildcard-tls-chain
```

Deploy gitlab runner helm

```
helm install -f values.yml gitlab-runner gitlab/gitlab-runner
```

Create a file`.gitlab-ci.yml` in a new project to verify  CI jobs

```yaml
stages:
  - build

image-build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.2-debug
    entrypoint: [""]
  script:
    - |
      cat <<EOF > Dockerfile
      FROM alpine:latest
      RUN echo "Hello World from CI"
      EOF
    - /kaniko/executor
      --context "${CI_PROJECT_DIR}"
      --dockerfile "${CI_PROJECT_DIR}/Dockerfile"
      --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHORT}"
      --no-push
```
