---
title: "Terraforming the Cluster"
tags:
- terraform
- kubernetes
- homelab
- cluster
---

Terraform is a configuration management program which
enables creation of cloud-based resources using specifications
written in a domain-specific language (DSL) called the
Hashicorp Configuration Language (HCL).

# Installing Terraform

Terraform can be downloaded from the
[Terraform website](https://www.terraform.io/downloads.html).

### Mac install

```bash
brew install terraform
```

### Windows Install

```powershell
choco install terraform
```

### Linux Install

```bash
wget https://releases.hashicorp.com/terraform/0.12.26/terraform_0.12.26_linux_amd64.zip
unzip terraform_0.12.26_linux_amd64.zipterraform_0.12.26_linux_amd64.zip
sudo mv terraform /usr/local/bin
sudo chmod +x /usr/local/bin/terraform
```

Additionally, when setting up Terraform in bash, an
alias can be set to shorten the Terraform command.

```bash
alias tf=/usr/bin/tf
```

# Setting up Cloud Access

Terraform configures access to different clouds using plugins known
as providers. For the purpose of this tutorial, the
[kubernetes](https://www.terraform.io/docs/providers/kubernetes/index.html)
and
[kubectl](https://gavinbunney.github.io/terraform-provider-kubectl/docs/provider.html).

The Kubernetes provider is built into Terraform.

## Installing kubectl

The kubectl provider is a 3rd party Terraform provider which uses kubectl to provision YAML charts for Kubernetes.

The kubectl provider needs to be manually installed
through an external plugin.

```bash
$ mkdir -p ~/.terraform.d/plugins && \
    curl -Ls https://api.github.com/repos/gavinbunney/terraform-provider-kubectl/releases/latest \
    | jq -r ".assets[] | select(.browser_download_url | contains(\"$(uname -s | tr A-Z a-z)\")) | select(.browser_download_url | contains(\"amd64\")) | .browser_download_url" \
    | xargs -n 1 curl -Lo ~/.terraform.d/plugins/terraform-provider-kubectl && \
    chmod +x ~/.terraform.d/plugins/terraform-provider-kubectl
```
