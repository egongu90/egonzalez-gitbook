# Zarf - Airgap deployment in kubernetes

> Zarf is a free and open-source tool that enables _**declarative creation & distribution of software into air-gapped/constrained/standalone environments**_.
>
> Zarf provides a way to package and deploy software in a way that is **repeatable**, **secure**, and **reliable**.



Install Zarf CLI

```sh
ZARF_VERSION=$(curl -sIX HEAD https://github.com/zarf-dev/zarf/releases/latest | grep -i ^location: | grep -Eo 'v[0-9]+.[0-9]+.[0-9]+')

curl -sL "https://github.com/zarf-dev/zarf/releases/download/${ZARF_VERSION}/zarf_${ZARF_VERSION}_Linux_amd64" -o zarf
chmod +x zarf
```

Download init package

When init this will deploy a registry and a couple more pods into the destination cluster

```sh
zarf tools download-init
zarf init --confirm
```

In this guide we will deploy falco for real time threat detection in kuberentes, config files are an example. Adapt to your needs.

Create a file `zarf.yaml` with the following data, images can be found with a command later on this guide

```yaml
kind: ZarfPackageConfig
metadata:
  name: falco
  version: 4.20.1
  description: |
    "A Zarf Package that deploys Falco Security for real time runtime threat detection"
components:
  - name: falco
    description: |
      "Deploys the falcosecurity falco chart into the cluster"
    required: true
    charts:
      - name: falco
        url: https://falcosecurity.github.io/charts
        version: 4.20.1
        namespace: falco
        valuesFiles:
          - values.yaml
    images:
      - docker.io/falcosecurity/falco-driver-loader:0.40.0
      - docker.io/falcosecurity/falco:0.40.0-debian
      - docker.io/falcosecurity/falcoctl:0.11.0
      # Cosign artifacts for images - falco - falco
      - index.docker.io/falcosecurity/falco-driver-loader:sha256-8bb7b51adf6598c5d9c90d2f3e55724212e6282afbd26f0ba428db9c0c417fbf.sig
      - index.docker.io/falcosecurity/falco:sha256-bfa486ca137359e90401f6121e52065e99bff44a949c02229fd0df467386fcaa.sig
      - index.docker.io/falcosecurity/falcoctl:sha256-4b590b9c49a881a55f6c3121c235057951418d726a9c43c4e1dbe3a5fcf358d3.sig
      - index.docker.io/falcosecurity/falcoctl:sha256-4b590b9c49a881a55f6c3121c235057951418d726a9c43c4e1dbe3a5fcf358d3.att
      
```

This command will output the list of images to include into `zarf.yaml`

```sh
zarf dev find-images
```

Generate a `values.yml` with the configuration you need, in this example I'm adding a custom rule for testing

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

Validate there is no errors in the config files.

```sh
zarf dev lint .
```

Generate a tar file with the images and config

```sh
zarf package create . --confirm
```

Deploy the package into the cluster, this will push images into local registry and invoke helm to deploy the resources in the chart.

```sh
zarf package deploy zarf-package-falco-amd64-4.20.1.tar.zst  --confirm
```
