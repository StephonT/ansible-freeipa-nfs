# ansible-freeipa-nfs

**Build FreeIPA Server and NFS Server**

Automate a FreeIPA identity management environment with NFS roaming home directories using Ansible. This project deploys a FreeIPA server, enrolls three client machines into the domain, provisions users, and configures every client to export and mount home directories over NFS using the `/net -hosts` autofs wildcard map.

---

## Architecture

```
                        ┌─────────────────────┐
                        │   Control Node       │
                        │   (Ansible)          │
                        └────────┬────────────┘
                                 │ SSH
              ┌──────────────────┼──────────────────┐
              │                  │                   │
     ┌────────▼───────┐ ┌────────▼───────┐ ┌────────▼───────┐
     │  ipaserver     │ │  ipaclient1    │ │  ipaclient2    │
     │                │ │                │ │                │
     │  FreeIPA       │ │  FreeIPA Client│ │  FreeIPA Client│
     │  Kerberos      │ │  NFS Export    │ │  NFS Export    │
     │  DNS           │ │  autofs /net   │ │  autofs /net   │
     └────────────────┘ └────────────────┘ └────────────────┘
                                                    │
                                           ┌────────▼───────┐
                                           │  ipaclient3    │
                                           │                │
                                           │  FreeIPA Client│
                                           │  NFS Export    │
                                           │  autofs /net   │
                                           └────────────────┘

  Roaming home directory path:  /net/<home_host>/home/<username>
  Example (alice on ipaclient2): /net/ipaclient1/home/alice
                                  └── autofs mounts ipaclient1:/
                                      NFS resolves to /home/alice on ipaclient1
```

Every client exports its own `/home` over NFS **and** mounts every other client's home via autofs. A user's `homedirectory` attribute in FreeIPA is set to `/net/<home_host>/home/<username>` so the correct physical directory is always found regardless of which client they log into.

---

## Prerequisites

| Requirement | Details |
|---|---|
| OS | RHEL 9 (required — CentOS Stream 9 is not supported by `freeipa.ansible_freeipa`) |
| Subscription | Red Hat Individual Developer subscription (free at https://developers.redhat.com) |
| Control node | Ansible installed, SSH key access to all managed nodes |
| IPA server | 2 vCPU, 2 GB RAM minimum |
| IPA clients | 1 vCPU, 1 GB RAM minimum each |
| Python | 3.9+ on all managed nodes |

---

## Project Structure

```
ansible-freeipa-nfs/
├── ansible.cfg                  # Project-local config — collections and roles resolved here first
├── requirements.yml             # Collection dependencies
├── inventory/
│   └── hosts.ini                # ipaserver, ipaclients, lab groups
├── group_vars/
│   ├── all.yml                  # FreeIPA domain, realm, NFS variables
│   ├── ipaclients.yml           # Client enrollment variables
│   └── users_vault.yml          # Vault-encrypted user credentials
├── collections/                 # ansible-galaxy installs here (gitignored)
│   └── ansible_collections/
│       └── freeipa/
│           └── ansible_freeipa/
├── roles/
│   └── nfs_server/              # Custom role — NFS export, autofs, SELinux, firewall
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── templates/
│       │   ├── exports.j2
│       │   └── auto.master.j2
│       └── vars/main.yml
├── playbooks/
│   ├── site.yml                 # Master playbook — chains all plays in order
│   └── create_users.yml         # FreeIPA user provisioning + home dir creation
└── docs/
    └── test_results.txt
```

---

## Collection Dependencies

Collections are declared in `requirements.yml` and installed into the project-local `collections/` directory:

```yaml
---
collections:
  - name: freeipa.ansible_freeipa
  - name: ansible.posix
    version: "1.5.4"
```

Install:

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
```

---

## Configuration

### ansible.cfg

```ini
[defaults]
inventory         = inventory/hosts.ini
remote_user       = your_ssh_user
host_key_checking  = false
collections_paths = ./collections:~/.ansible/collections:/usr/share/ansible/collections
roles_path        = ./roles:~/.ansible/roles:/usr/share/ansible/roles
```

The project-local paths are listed first so collections and roles installed here take precedence over anything in `~/.ansible` or system directories.

### Key Variables (group_vars/all.yml)

```yaml
ipaserver_domain:             example.com
ipaserver_realm:              EXAMPLE.COM
ipaserver_setup_dns:          true
ipaserver_auto_forwarders:    true
ipaserver_no_dnssec_validation: true
ipaserver_auto_reverse:       true
nfs_server_export_homedir:    "/home  *.example.com(rw,no_root_squash,no_subtree_check)"
```

> **Note:** Set `ipaserver_auto_reverse: true` to have FreeIPA create the reverse DNS zone automatically. If you have already deployed the server without it, create the zone manually: `ipa dnszone-add <reverse-zone>`

---

## Usage

### 1. Install collections

```bash
ansible-galaxy collection install -r requirements.yml -p ./collections
```

### 2. Run the full stack

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
```

### 3. Run individual stages

```bash
# Provision IPA server only
ansible-playbook collections/ansible_collections/freeipa/ansible_freeipa/playbooks/install-server.yml

# Enroll clients only
ansible-playbook collections/ansible_collections/freeipa/ansible_freeipa/playbooks/install-client.yml --limit ipaclients

# Configure NFS on all clients
ansible-playbook playbooks/site.yml --tags nfs --limit ipaclients

# Create users and home directories
ansible-playbook playbooks/create_users.yml --ask-vault-pass
```

---

## Roaming Home Directory Design

This project uses the `/net -hosts` autofs wildcard map rather than static NFS mounts or FreeIPA LDAP automount maps. The key design decisions are:

- **Every client exports its own `/home`** — each machine is both an NFS exporter and consumer
- **`/net -hosts` is configured on every client** — accessing `/net/<hostname>` triggers an on-demand NFS mount of that host's root filesystem
- **Each FreeIPA user's `homedirectory` attribute** is set to `/net/<home_host>/home/<username>`
- **Physical home directories are pre-created** on the designated host using `delegate_to` in the user provisioning playbook — this is required because `mkhomedir` only creates the directory on the machine the user logs *into*, not on the designated home host

### Known Gotchas

| Issue | Cause | Fix |
|---|---|---|
| Login fails: "no such file or directory" | Physical `/home/<user>` missing on `home_host` | Run `create_users.yml` — `delegate_to` creates it on the correct host |
| NFS mounts denied by SELinux | `use_nfs_home_dirs` boolean not set before autofs starts | Role sets `setsebool -P use_nfs_home_dirs on` before service start |
| User ownership wrong on remote mounts | `idmapd.conf` Domain not set | Role fixes with `lineinfile` using regexp `^[#]?Domain\s*=` |
| Reverse DNS zone error during IPA install | Reverse zone not created | Set `ipaserver_auto_reverse: true` in `group_vars/all.yml` |

---

## Security Notes

- Admin and Directory Manager passwords are stored in `group_vars/users_vault.yml` encrypted with Ansible Vault
- Never commit unencrypted vault files — add `*vault*` to `.gitignore` or ensure files are always encrypted before committing
- The NFS export uses a domain wildcard (`*.example.com`) rather than a subnet CIDR — only enrolled IPA clients in your domain can mount

---

## Technologies Used

| Technology | Purpose |
|---|---|
| Ansible | Automation engine |
| `freeipa.ansible_freeipa` | FreeIPA server and client deployment |
| `ansible.posix` | `firewalld` module for NFS firewall rules |
| FreeIPA / 389-DS / Kerberos | Identity management, authentication, DNS |
| NFSv4 | Home directory file sharing |
| autofs (`/net -hosts`) | On-demand NFS mount of roaming home directories |
| Ansible Vault | Credential encryption |
| Jinja2 | `exports.j2`, `auto.master.j2` templates |
| RHEL 9 | Target OS |

---

## License

MIT