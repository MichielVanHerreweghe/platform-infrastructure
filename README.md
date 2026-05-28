# platform-infrastructure

Ansible automation for provisioning and hardening the VMs that host the K3s
cluster.

## Layout

```
src/ansible/
  bootstrap.yml               # one-shot: creates the deployer account
  cluster-initialization.yml  # full host configuration (hardening + K3s prereqs)
  group_vars/all.yml          # tunables for every role
  inventory.yml
  roles/
    common              # timezone, NTP, journald, base packages
    user_management     # accounts, sudo, authorized_keys
    kernel_hardening    # sysctl, login.defs, module blacklist, coredumps
    k3s_prereqs         # swap off, BPF fs, K3s+Cilium sysctl/modules, ulimits
    auto_updates        # unattended-upgrades (security only, no auto-reboot)
    fail2ban            # SSH brute-force protection
    firewall            # ufw, default-deny, peer-aware cluster rules
    ssh_hardening       # custom port, modern crypto, AllowUsers, banner
```

## First-time provisioning

```sh
cd src/ansible
ansible-galaxy collection install -r requirements.yml

# 1. Create the deployer account on the fresh VM. Defaults to root; pass
#    bootstrap_user=ubuntu (or similar) if the provider account is named.
ansible-playbook bootstrap.yml --ask-pass --ask-become-pass

# 2. Apply hardening + K3s prereqs. The very first run still connects on
#    port 22, so override ansible_port for this run only:
ansible-playbook cluster-initialization.yml -e ansible_port=22

# 3. From this point on the host listens on ssh_port (2222) and ansible_port
#    in group_vars/all.yml resolves to ssh_port automatically:
ansible-playbook cluster-initialization.yml
```

The firewall role keeps the migration port open during step 2 (it allows both
`ansible_port` and `ssh_port` whenever they differ), so a mid-run failure
cannot strand the host behind a closed port.

## What gets hardened

- **SSH**: port 2222, key-only auth, no root, AllowUsers whitelist, modern
  KEX/cipher/MAC list, login grace 30s, max 3 auth tries, legal banner.
- **Firewall (ufw)**: default deny incoming, allow outgoing. Public TCP 80/443
  open; cluster-internal ports (K3s API, etcd, kubelet, Cilium VXLAN, Hubble)
  are only opened to other nodes in the `cluster_servers` inventory group.
- **fail2ban**: sshd jail on the custom port, aggressive mode.
- **Auto-updates**: security archive only, no auto-reboot, kernel/K3s
  packages explicitly blacklisted.
- **Kernel**: hardening sysctls (kptr_restrict, ptrace_scope, redirects off,
  rp_filter on by default), legacy filesystem and protocol modules blocked,
  core dumps disabled, login.defs password aging and umask tightened.
- **K3s + Cilium prereqs**: swap permanently off, `ip_forward=1`, BPF
  filesystem mounted at `/sys/fs/bpf`, raised inotify/file limits, kernel
  modules loaded at boot. `br_netfilter` intentionally NOT loaded — Cilium
  with kube-proxy replacement uses eBPF, not the bridge netfilter path.
- **Base**: timezone, chrony NTP (tight clock sync for etcd quorum),
  persistent journald with size caps.

## Cluster topology

`firewall_cluster_*_ports` in `group_vars/all.yml` are only opened to peers
listed under the `cluster_servers` inventory group. Adding a node is a matter
of adding it to that group and re-running the playbook across all hosts so
peer IPs propagate via gathered facts.
