 

## :wave:&nbsp; Introduction

The following components will be installed in your [k3s](https://k3s.io/) cluster by default.
Feel free to read up on any of these technologies before you get started to be more familiar with them.

- [cert-manager](https://cert-manager.io/) - SSL certificates - with Cloudflare DNS challenge
- [calico](https://www.tigera.io/project-calico/) - CNI (container network interface)
- [flux](https://toolkit.fluxcd.io/) - GitOps tool for deploying manifests from the `cluster` directory
- [hajimari](https://github.com/toboshii/hajimari) - start page with ingress discovery
- [kube-vip](https://kube-vip.io/) - layer 2 load balancer for the Kubernetes control plane
- [local-path-provisioner](https://github.com/rancher/local-path-provisioner) - default storage class provided by k3s
- [metallb](https://metallb.universe.tf/) - bare metal load balancer
- [reloader](https://github.com/stakater/Reloader) - restart pods when Kubernetes `configmap` or `secret` changes
- [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller) - upgrade k3s
- [traefik](https://traefik.io) - ingress controller
- [velero](https://velero.io/) - backup system
- [rook-ceph](https://rook.github.io/) - setup volumes and claims

For provisioning the following tools will be used:

- [Ubuntu](https://ubuntu.com/download/server) - this is a pretty universal operating system that supports running all kinds of home related workloads in Kubernetes
- [Ansible](https://www.ansible.com) - this will be used to provision the Ubuntu operating system to be ready for Kubernetes and also to install k3s
- [Terraform](https://www.terraform.io) - in order to help with the DNS settings this will be used to provision an already existing Cloudflare domain and DNS settings

## :memo:&nbsp; Prerequisites

### :computer:&nbsp; Systems

- One or mote nodes with a fresh install of [Ubuntu Server 20.04](https://ubuntu.com/download/server). These nodes can be bare metal or VMs.
- A [Cloudflare](https://www.cloudflare.com/) account with a domain, this will be managed by Terraform.
- Some experience in debugging problems and a positive attitude ;)

### :wrench:&nbsp; Tools

:round_pushpin: You should install the below CLI tools on your workstation. Make sure you pull in the latest versions.

#### Required

| Tool                                                               | Purpose                                                             |
|--------------------------------------------------------------------|---------------------------------------------------------------------|
| [ansible](https://www.ansible.com)                                 | Preparing Ubuntu for Kubernetes and installing k3s                  |
| [direnv](https://github.com/direnv/direnv)                         | Exports env vars based on present working directory                 |
| [flux](https://toolkit.fluxcd.io/)                                 | Operator that manages your k8s cluster based on your Git repository |
| [gnupg](https://gnupg.org/)                                        | Encrypts and signs your data                                        |
| [go-task](https://github.com/go-task/task)                         | A task runner / simpler Make alternative written in Go              |
| [ipcalc](http://jodies.de/ipcalc)                                  | Used to verify settings in the configure script                     |
| [jq](https://stedolan.github.io/jq/)                               | Used to verify settings in the configure script                     |
| [kubectl](https://kubernetes.io/docs/tasks/tools/)                 | Allows you to run commands against Kubernetes clusters              |
| [pinentry](https://gnupg.org/related_software/pinentry/index.html) | Allows GnuPG to read passphrases and PIN numbers                    |
| [sops](https://github.com/mozilla/sops)                            | Encrypts k8s secrets with GnuPG                                     |
| [terraform](https://www.terraform.io)                              | Prepare a Cloudflare domain to be used with the cluster             |
| [helm](https://helm.sh/)                                           | Manage Kubernetes applications                                      |
| [kustomize](https://kustomize.io/)                                 | Template-free way to customize application configuration            |

## :open_file_folder:&nbsp; Repository structure

The Git repository contains the following directories under `cluster` and are ordered below by how Flux will apply them.

- **base** directory is the entrypoint to Flux
- **crds** directory contains custom resource definitions (CRDs) that need to exist globally in your cluster before anything else exists
- **core** directory (depends on **crds**) are important infrastructure applications (grouped by namespace) that should never be pruned by Flux
- **apps** directory (depends on **core**) is where your common applications (grouped by namespace) could be placed, Flux will prune resources here if they are not tracked by Git anymore

```
cluster
├── apps
│   ├── home
│   ├── home-automation
│   ├── monitoring 
│   ├── networking
│   ├── velero
│   └── system-upgrade
├── base
│   └── flux-system
├── core
│   ├── cert-manager
│   ├── metallb-system
│   ├── namespaces
│   ├── rook-ceph
│   └── system-upgrade
└── crds
    └── cert-manager
```

### :closed_lock_with_key:&nbsp; Setting up GnuPG keys

:round_pushpin: Here we will create a personal and a Flux GPG key. Using SOPS with GnuPG allows us to encrypt and decrypt secrets.

1. Create a Personal GPG Key, password protected, and export the fingerprint. It's **strongly encouraged** to back up this key somewhere safe so you don't lose it.

```sh
export GPG_TTY=$(tty)
export PERSONAL_KEY_NAME="First name Last name (location) <email>"

gpg --batch --full-generate-key <<EOF
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Real: ${PERSONAL_KEY_NAME}
EOF

gpg --list-secret-keys "${PERSONAL_KEY_NAME}"
# pub   rsa4096 2021-03-11 [SC]
#       772154FFF783DE317KLCA0EC77149AC618D75581
# uid           [ultimate] k8s@home (Macbook) <k8s-at-home@gmail.com>
# sub   rsa4096 2021-03-11 [E]
```

2. Create a Flux GPG Key and export the fingerprint

```sh
export GPG_TTY=$(tty)
export FLUX_KEY_NAME="Cluster name (Flux) <email>"

gpg --batch --full-generate-key <<EOF
%no-protection
Key-Type: 1
Key-Length: 4096
Subkey-Type: 1
Subkey-Length: 4096
Expire-Date: 0
Name-Real: ${FLUX_KEY_NAME}
EOF

gpg --list-secret-keys "${FLUX_KEY_NAME}"
# pub   rsa4096 2021-03-11 [SC]
#       AB675CE4CC64251G3S9AE1DAA88ARRTY2C009E2D
# uid           [ultimate] Home cluster (Flux) <k8s-at-home@gmail.com>
# sub   rsa4096 2021-03-11 [E]
```

### :cloud:&nbsp; Global Cloudflare API Key

In order to use Terraform and `cert-manager` with the Cloudflare DNS challenge you will need to create a API key.

1. Head over to Cloudflare and create a API key by going [here](https://dash.cloudflare.com/profile/api-tokens).

2. Under the `API Keys` section, create a global API Key.

3. Use the API Key in the configuration section below.

### :page_facing_up:&nbsp; Configuration

:round_pushpin: The `.config.env` file contains necessary configuration files that are needed by Ansible, Terraform and Flux.

1. Copy the `.config.sample.env` to `.config.env` and start filling out all the environment variables. **All are required** and read the comments they will explain further what is required.

2. Once that is done, verify the configuration is correct by running `./configure.sh --verify`

3. If you do not encounter any errors run `./configure.sh` to start having the script wire up the templated files and place them where they need to be.

### :zap:&nbsp; Preparing Ubuntu with Ansible

:round_pushpin: Here we will be running a Ansible Playbook to prepare Ubuntu for running a Kubernetes cluster.

1. Ensure you are able to SSH into you nodes from your workstation with using your private ssh key. This is how Ansible is able to connect to your remote nodes.

2. Install the deps by running `task ansible:deps`

3. Verify Ansible can view your config by running `task ansible:list`

4. Verify Ansible can ping your nodes by running `task ansible:adhoc:ping`

5. Finally, run the Ubuntu Prepare playbook by running `task ansible:playbook:ubuntu-prepare`

6. If everything goes as planned you should see Ansible running the Ubuntu Prepare Playbook against your nodes.

7. If needed use `task ansible:playbook:rook-ceph-nuke` to clean up the rook volumes

### :sailboat:&nbsp; Installing k3s with Ansible

:round_pushpin: Here we will be running a Ansible Playbook to install [k3s](https://k3s.io/) with [this](https://galaxy.ansible.com/xanmanning/k3s) wonderful k3s Ansible galaxy role. After completion, Ansible will drop a `kubeconfig` in `./provision/kubeconfig` for use with interacting with your cluster with `kubectl`.

1. Verify Ansible can view your config by running `task ansible:list`

2. Verify Ansible can ping your nodes by running `task ansible:adhoc:ping`

3. Run the k3s install playbook by running `task ansible:playbook:k3s-install`

4. If everything goes as planned you should see Ansible running the k3s install Playbook against your nodes.

5. Verify the nodes are online
   
```sh
kubectl --kubeconfig=./provision/kubeconfig get nodes
# NAME           STATUS   ROLES                       AGE     VERSION
# k8s-0          Ready    control-plane,master      4d20h   v1.21.5+k3s1
# k8s-1          Ready    worker                    4d20h   v1.21.5+k3s1
```

### :small_blue_diamond:&nbsp; GitOps with Flux

:round_pushpin: Here we will be installing [flux](https://toolkit.fluxcd.io/) after some quick bootstrap steps.

1. Verify Flux can be installed

```sh
flux --kubeconfig=./provision/kubeconfig check --pre
# ► checking prerequisites
# ✔ kubectl 1.21.5 >=1.18.0-0
# ✔ Kubernetes 1.21.5+k3s1 >=1.16.0-0
# ✔ prerequisites checks passed
```

2. Pre-create the `flux-system` namespace

```sh
kubectl --kubeconfig=./provision/kubeconfig create namespace flux-system --dry-run=client -o yaml | kubectl --kubeconfig=./provision/kubeconfig apply -f -
```

3. Add the Flux GPG key in-order for Flux to decrypt SOPS secrets

```sh
source .config.env
gpg --export-secret-keys --armor "${BOOTSTRAP_FLUX_KEY_FP}" |
kubectl --kubeconfig=./provision/kubeconfig create secret generic sops-gpg \
    --namespace=flux-system \
    --from-file=sops.asc=/dev/stdin
```

:round_pushpin: Variables defined in `./cluster/base/cluster-secrets.sops.yaml` and `./cluster/base/cluster-settings.sops.yaml` will be usable anywhere in your YAML manifests under `./cluster`


4.  Push you changes to git

```sh
git add -A
git commit -m "initial commit"
git push
```

5. Install Flux

:round_pushpin: Due to race conditions with the Flux CRDs you will have to run the below command twice. There should be no errors on this second run.

```sh
kubectl --kubeconfig=./provision/kubeconfig apply --kustomize=./cluster/base/flux-system
# namespace/flux-system configured
# customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io created
# ...
# unable to recognize "./cluster/base/flux-system": no matches for kind "Kustomization" in version "kustomize.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "GitRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
# unable to recognize "./cluster/base/flux-system": no matches for kind "HelmRepository" in version "source.toolkit.fluxcd.io/v1beta1"
```

6. Verify Flux components are running in the cluster

```sh
kubectl --kubeconfig=./provision/kubeconfig get pods -n flux-system
# NAME                                       READY   STATUS    RESTARTS   AGE
# helm-controller-5bbd94c75-89sb4            1/1     Running   0          1h
# kustomize-controller-7b67b6b77d-nqc67      1/1     Running   0          1h
# notification-controller-7c46575844-k4bvr   1/1     Running   0          1h
# source-controller-7d6875bcb4-zqw9f         1/1     Running   0          1h
```

7. nuke rook-ceph cluster and extra commands:

```sh
# kubectl label node k8s-0 node-role.kubernetes.io/worker=worker
# kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
# -------NUKE-------
# rm -rf /var/lib/rook
# sudo sgdisk --zap-all /dev/sdb
# sudo fuser --kill ceph--78d128a9--5575--4a6c--b53f--bff7cebb2372-osd--block--851bf7db--37a9--491e--b6f1--ba84dbc78ba4
# sudo dmsetup clear ceph--78d128a9--5575--4a6c--b53f--bff7cebb2372-osd--block--851bf7db--37a9--491e--b6f1--ba84dbc78ba4
# sudo dmsetup remove -f ceph--78d128a9--5575--4a6c--b53f--bff7cebb2372-osd--block--851bf7db--37a9--491e--b6f1--ba84dbc78ba4
# sudo rm -rf ceph--78d128a9--5575--4a6c--b53f--bff7cebb2372-osd--block--851bf7db--37a9--491e--b6f1--ba84dbc78ba4
# sudo wipefs -af /dev/sdb
```

:tada: **Congratulations** you have a Kubernetes cluster managed by Flux, your Git repository is driving the state of your cluster.

### :cloud:&nbsp; Configure Cloudflare DNS with Terraform

:round_pushpin: Review the Terraform scripts under `./terraform/cloudflare/` and make sure you understand what it's doing (no really review it). If your domain already has existing DNS records be sure to export those DNS settings before you continue. Ideally you can update the terraform script to manage DNS for all records if you so choose to.

1. Pull in the Terraform deps by running `task terraform:init:cloudflare`

2. Review the changes Terraform will make to your Cloudflare domain by running `task terraform:plan:cloudflare`

3. Finally have Terraform execute the task by running `task terraform:apply:cloudflare`

If Terraform was ran successfully head over to your browser and you _should_ be able to access `https://hajimari.${BOOTSTRAP_CLOUDFLARE_DOMAIN}`

#### All thanks to:

https://k8s-at-home.com/
