# Personal Ansible Playbooks

Collection of Ansible playbooks to automate personal instances.

## Repository layout

```
.
├── kubernetes-alpine.yml      # Main playbook (k8s cluster on Alpine)
├── group_vars/                # Inventory group variables
│   ├── masters.yml            # kubernetes_node_type: master
│   └── workers.yml            # kubernetes_node_type: worker
├── roles/
│   └── kubernetes-alpine/     # Role: kubeadm + containerd + Flannel
├── standalone/                # Standalone playbooks (no roles)
│   ├── DmorgamShellEnv.yml    # Personal shell env (zsh, tmux, SpaceVim)
│   └── OCIactualizarPaquetes.yml  # Package updates on OCI hosts
├── requirements.txt           # Python deps (ansible, jmespath, cryptography)
├── requirements.yml           # Galaxy collections (community.general, community.crypto, kubernetes.core)
└── inventory.yml              # (git-ignored) local inventory
```

## Requirements

```bash
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

## Main playbook: `kubernetes-alpine.yml`

Deploys a `kubeadm`-based Kubernetes cluster on Alpine Linux nodes with optional HA across multiple masters and Flannel CNI.

### Play variables

| Variable | Description |
|----------|-------------|
| `kubernetes_first_master` | `inventory_hostname` of the primary control plane (the one running `kubeadm init`). |
| `kubernetes_apiserver_extra_sans` | Extra SAN for the API server certificate. Defaults to `k8s-cluster.local`. |
| `kubernetes_apiserver_ha_endpoint` | HA endpoint (DNS or VIP) pointing at the API server. Defaults to `k8s.local`. |

`become_method: su` — Alpine nodes use `su` instead of `sudo`.

### Expected inventory

Two groups:

```yaml
all:
  children:
    masters:
      hosts:
        master1:
        master2:
        master3:
    workers:
      hosts:
        worker1:
        worker2:
```

`group_vars/masters.yml` and `group_vars/workers.yml` tag each node with `kubernetes_node_type`.

### Run

```bash
ansible-playbook -i inventory.yml kubernetes-alpine.yml
```

## Role `kubernetes-alpine`

`roles/kubernetes-alpine/tasks/main.yml` orchestrates six phases:

1. **`core.yml`** — common prep on every node:
   - Loads kernel modules `br_netfilter`, `overlay`.
   - Applies sysctl: `net.bridge.bridge-nf-call-iptables`, `net.ipv4.ip_forward`, `net.bridge.bridge-nf-call-ip6tables`.
   - Disables swap.
   - Enables the Alpine community repo.
   - Installs `cni-plugins`, `cni-plugin-flannel`, `containerd`, `uuidgen`, `nfs-utils`.
   - Generates `/etc/machine-id` if missing.
   - Starts the `containerd` service.
   - Installs `kubelet`, `kubectl`, `kubeadm`, `py3-cryptography`, pinned to an exact minor version.
   - Enables the `kubelet` service.

2. **`master.yml`** — only nodes with `kubernetes_node_type: master`:
   - **First master** (`inventory_hostname == kubernetes_first_master`):
     - Checks API server health at `localhost:6443/livez`.
     - Adds `kubernetes_apiserver_ha_endpoint` to `/etc/hosts` if it does not resolve.
     - Runs `kubeadm init` with `--pod-network-cidr`, `--service-cidr`, `--apiserver-advertise-address`, `--apiserver-cert-extra-sans`, `--control-plane-endpoint`, `--upload-certs`.
     - Downloads the Flannel manifest, patches the pod CIDR, applies it with `kubectl --kubeconfig admin.conf`.
     - If there are more nodes, generates join token, CA hash, certificate-key. Exposes them as facts.
   - **Secondary masters**: run `kubeadm join --control-plane` reusing the first master's token + CA hash + certificate-key.

3. **`worker.yml`** — only nodes with `kubernetes_node_type: worker`:
   - Checks `kubelet` at `localhost:10248/healthz`.
   - If not joined, runs `kubeadm join` with token and CA hash (no `--control-plane`).

4. **`traefik.yml`** — first master only: installs Traefik ingress with Helm.

5. **`storage.yml`** — first master only: installs local-path-provisioner as default StorageClass.

6. **`metrics.yml`** — first master only: installs metrics-server with Helm.

### Relevant defaults (`roles/kubernetes-alpine/defaults/main.yml`)

| Variable | Default |
|----------|---------|
| `kubernetes_node_type` | `none` (must be overridden) |
| `kubernetes_first_master` | `none` |
| `kubernetes_pods_cidr` | `172.18.0.0/17` |
| `kubernetes_service_cidr` | `172.18.128.0/17` |
| `kubernetes_apiserver_extra_sans` | `k8s-apiserver.local` |
| `kubernetes_apiserver_ha_endpoint` | `kubernetes-apiserver.local` |
| `kubernetes_cni_yaml_url` | Flannel official manifest on GitHub |

## Standalone playbooks

### `standalone/DmorgamShellEnv.yml`

Sets up a personal shell environment on Debian/Ubuntu and CentOS:

- Installs `git`, `neovim`, `tmux`, `zsh`, `powerline`.
- Downloads and runs the OhMyZsh and SpaceVim installers.
- Configures `tmux` with powerline bindings.
- Enables zsh with vi mode and autosuggestions.

Run (prompts for sudo password):

```bash
ansible-playbook -i inventory.yml standalone/DmorgamShellEnv.yml --ask-become-pass
```

### `standalone/OCIactualizarPaquetes.yml`

Updates packages on hosts in the `oci` group. Uses `apt` on Debian/Ubuntu, `yum` on CentOS/OracleLinux.

```bash
ansible-playbook -i inventory.yml standalone/OCIactualizarPaquetes.yml
```

## License

GNU GPLv3 — see `LICENSE`.
