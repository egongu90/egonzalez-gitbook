---
description: Installation an example usages
---

# OWASP Dependency-track

Add dependency-track helm repository

```
helm repo add dependency-track https://dependencytrack.github.io/helm-charts
```

Deploy depencency-track helm chart, hostname is my minikube instance. Use the appropiate hostname for your environment

```
helm upgrade --install dtrack dependency-track/dependency-track \
  --set ingress.enabled=true \
  --set ingress.hostname=dtrack.$(minikube ip).nip.io
```

Browse to your ingress `kubectl get ingress -o yaml | awk '/host/ {print$3}'`

Default username and password are admin/admin, you must change them before do anything

Create a new team at \<URL>/admin/accessManagement/teams

On the new team generate an API KEY

Add required permissions, at least BOM upload

Download an example git repository

```
git clone https://github.com/xNaaro/vulnerable_python.git
cd vulnerable_python
```

Install syft to create an example SBOM of the above repository

```
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /tmp/
```

Create an SBOM of the example repository

```
/tmp/syft . -o cyclonedx-json > cyclonedx.json
 ✔ Indexed file system                                                                     .
 ✔ Cataloged contents              cdb4ee2aea69cc6a83331bbe96dc2caa9a299d21329efb0336fc02a82  
   ├── ✔ Packages                        [1 packages]  
   ├── ✔ File digests                    [1 files]  
   ├── ✔ File metadata                   [1 locations]  
   └── ✔ Executables                     [0 executables]  
[0000]  WARN no explicit name and version provided for directory source, deriving artifact ID 
```

Upload SBOM to dependency-check

```
curl -X "POST" "http://dtrack.192.168.39.47.nip.io/api/v1/bom" \
    -H "Content-Type: multipart/form-data" \
    -H "X-API-Key: odt_Wrz6kL1YBcw3tyrCDT7oJvHcYrFaPACV" \
    -F "autoCreate=true" \
    -F "projectName=vulnerablepython" \
    -F 'bom=@./cyclonedx.json'
```

Now in your frontend server should have a project called vulnerablepython with flask as vulnerable package

First installation of dependency-track may take for a while to update vulnerabilities lists
