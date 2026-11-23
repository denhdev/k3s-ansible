# Build a Kubernetes cluster using K3s via Ansible

Ansible playbooks that bring up a K3s cluster, then install
[MetalLB](https://metallb.io/) for LoadBalancer services and
[ArgoCD](https://argo-cd.readthedocs.io/) for GitOps.

## Command Quick Reference

```bash
ansible all -m ping                        # test SSH access to the inventory
ansible-playbook -K playbooks/site.yml     # install k3s, MetalLB, ArgoCD
ansible-playbook -K playbooks/upgrade.yml  # upgrade k3s to k3s_version
ansible-playbook -K playbooks/reset.yml    # uninstall k3s from all nodes
```

`ansible.cfg` points at `./inventory.yml`, so `-i inventory.yml` is optional.
`-K` prompts for the sudo password on the nodes.

## Requirements

The control node (where you run Ansible) needs Ansible 8.0+ (ansible-core 2.15+).

All managed nodes need:

- Passwordless SSH access (key-based)
- A user with sudo rights

Supported node OSes: Debian, Ubuntu, Raspberry Pi OS, RHEL family, SUSE family,
and ArchLinux, on x64, arm64, or armhf. Disabling firewalls and swap is
recommended; see the [K3s requirements](https://docs.k3s.io/installation/requirements).

## Setup

Install Ansible into a virtualenv (`.venv` is gitignored):

```bash
python3 -m venv .venv
.venv/bin/pip install ansible
```

Copy the sample inventory and edit it for your cluster:

```bash
cp inventory-sample.yml inventory.yml
```

```yaml
k3s_cluster:
  children:
    server:
      hosts:
        <HOSTNAME>:
          ansible_host: 10.0.0.xxx
    agent:
      hosts:
        <HOSTNAME>:
          ansible_host: 10.0.0.xxx
        <HOSTNAME>:
          ansible_host: 10.0.0.xxx
```

In the `vars` section, set at least `ansible_user`, `token`, and
`metallb_ip_range` (a block of free IPs on your LAN, outside the DHCP range).
`inventory.yml` is gitignored, since it holds the cluster token.

Then test SSH and run the playbook:

```bash
.venv/bin/ansible all -m ping
.venv/bin/ansible-playbook -K playbooks/site.yml
```

### High availability

If the `server` group has more than one host, the playbook sets up K3s in HA
mode with embedded etcd. Use an odd number of servers (3, 5, 7). See the
[K3s HA docs](https://docs.k3s.io/datastore/ha-embedded). A load balancer or VIP
in front of the API is possible but not covered here.

### External database

To use an external datastore instead of etcd, set `use_external_database` and
pass the endpoint as a server argument:

```yaml
  vars:
    use_external_database: true
    extra_server_args: "--disable servicelb --datastore-endpoint=postgres://username:password@hostname:port/database-name"
```

See the [datastore endpoint format](https://docs.k3s.io/datastore#datastore-endpoint-format-and-functionality).

## Kubeconfig

After bringup, the cluster's kubeconfig is copied to the control node and merged
into `~/.kube/config` under the `k3s-ansible` context:

```bash
kubectl config use-context k3s-ansible
kubectl get nodes
```

Set `cluster_context` to change the context name, or `kubeconfig` to write it to
a separate file instead of merging.

## MetalLB

K3s's built-in `servicelb` is disabled (`extra_server_args: "--disable servicelb"`)
so MetalLB can assign LoadBalancer IPs from `metallb_ip_range`. Find assigned IPs
with:

```bash
kubectl get svc -A | grep LoadBalancer
```

## ArgoCD

ArgoCD is installed from the upstream `stable` manifest into the `argocd`
namespace, and `argocd-server` is exposed as a LoadBalancer service:

```bash
kubectl -n argocd get svc argocd-server   # UI address (EXTERNAL-IP)
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Log in as `admin` with that password.

## Upgrading

Update `k3s_version` in `inventory.yml` (see the
[K3s releases](https://github.com/k3s-io/k3s/releases)), then:

```bash
ansible-playbook -K playbooks/upgrade.yml
```

## Resetting

Uninstall K3s from all nodes:

```bash
ansible-playbook -K playbooks/reset.yml
```

## Airgap install

Set `airgap_dir` to a directory containing the K3s binary and the airgap images
for your architecture, from the [K3s releases](https://github.com/k3s-io/k3s/releases):

```bash
$ ls ./playbooks/my-airgap/
k3s
k3s-airgap-images-amd64.tar.gz
```

```yaml
airgap_dir: ./my-airgap  # relative to the playbooks directory
```

On SELinux systems, also add the latest
[k3s-selinux RPM](https://github.com/k3s-io/k3s-selinux/releases/latest) to that
directory. The control node still needs internet access to download the K3s
install script, which it distributes to the nodes.
