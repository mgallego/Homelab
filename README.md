# 🏠 Homelab

> **One repository to rule them all.** GitOps-style provisioning for my homelab — so that when disaster strikes (yes, even a dead SD card 🥲), the whole setup can be rebuilt from scratch in minutes, not days.

## 🎯 What is this?

This is the central repository that manages my home infrastructure. Instead of relying on backups of a single broken machine, the entire homelab state lives here as **code**, ready to be re-applied to any (or every) machine that needs to come back to life.

It currently starts small: provisioning the base system for an **Orange Pi 5 Plus** (the machine whose SD card just gave up). The plan is to grow it into a full homelab recovery tool covering every machine and service I run.

## ✨ Why this exists

- 💥 **The trigger:** The Orange Pi's SD card died. Nothing was documented, nothing was automated — rebuilding would have been painful.
- 🛟 **The idea:** If it happens again, I want to say "one command" and have the machine come back: user, packages, Docker, disks, and network shares.
- 🤖 **The playground:** This project is also my sandbox for practicing with AI agents — they write the code, but every single change is reviewed by me before it lands.
- 🚀 **The vision:** Go from single-device provisioning to provisioning the **entire homelab**, whatever it may contain in the future.

## 🧩 What it provisions today

| Layer | What it does |
| --- | --- |
| 👤 **Users** | Creates the `moises` user (uid 1000), sudo + docker groups, SSH keys, and authorized keys for `moises` and `root`. |
| 📦 **Packages** | Installs base tools (`tmux`, `git`, `vim`, `rsync`). |
| 🧠 **System** | Reduces SD writes: journald in RAM, apt cache in tmpfs. |
| 🚫 **Pi-hole** | Frees port 53 (disables systemd-resolved) for Pi-hole; static `/etc/resolv.conf`. Run only on Pi-hole hosts (`--tags pihole`). |
| 🔗 **Symlinks** | Links SD paths to M2 folders (SD → M2), sparing the SD card. |
| 🐳 **Docker** | Adds the official Docker apt repo and installs `docker-ce`, `containerd`, Buildx, and Compose. `data-root` and containerd's root live on the M.2 disk to keep writes off the SD card; container logs are capped. Registry login (e.g. ghcr.io) is written to `~moises/.docker/config.json` from `docker_registries` (vault). |
| 💾 **Mounts** | Mounts the M2 NVMe disk by UUID under `/home/moises/mnt/M2`. |
| 🌐 **NFS** | Mounts NFS shares from the NAS machines (192.168.1.20 / .245) under `/home/moises/mnt/`. |
| 🗄️ **NFS Server** | Shares the M2 disk (`/home/moises/mnt/M2`) to other LAN machines via NFS. |
| 📦 **IAC** | Clones the infrastructure-as-code repository into `/home/moises/mnt/M2/iac` using the SSH key from the `users` role. |
| 🐙 **Stacks** | Deploys Docker Compose stacks (e.g. Portainer) from the infra repo. |

## 📁 Layout

```
provisioning/
└── ansible/
    ├── site.yml                    # Playbook entrypoint (roles in order)
    ├── inventory.ini               # Single host: orangepi (192.168.1.241)
    ├── group_vars/orangepi/
    │   ├── vars.yml                # Non-sensitive variables
    │   └── vault.yml               # Secrets (gitignored — never commit!)
    └── roles/
        ├── users/                  # User, SSH keys, sudo/docker groups
        ├── packages/               # Base apt packages
        ├── system/                 # Journald in RAM, apt cache tmpfs
        ├── pihole/                 # Frees port 53 for Pi-hole
        ├── mounts/                 # M2 disk mount (before docker)
        ├── docker/                 # Docker engine + data-root on M2
        ├── nfs/                    # NFS client + shares
        ├── nfs_server/             # Serves the M2 disk to other machines
        ├── symlinks/               # SD -> M2/NFS symlinks
        ├── iac/                    # Clones the infra-as-code repository
        └── stacks/                 # Deploys Docker Compose stacks
```

## 🚀 Getting started

From `provisioning/ansible/`:

```sh
# 1. Make sure the ansible.posix collection is installed (used for mounts)
ansible-galaxy collection install ansible.posix

# 2. Provision the Orange Pi
ansible-playbook -i inventory.ini site.yml
```

### 🔍 Verification

There are no dry-run/lint wrappers — use Ansible's built-in flags:

```sh
ansible-playbook --syntax-check -i inventory.ini site.yml
ansible-playbook --check -i inventory.ini site.yml
```

### 📋 Prerequisites

- Network access from the machine running the playbook to `192.168.1.241`.
- The playbook connects as `root` and uses `become: true`.
- `infra_path` (`vars.yml`) is the repo clone at `/home/moises/mnt/M2/iac`; container data/configs stay in `/home/moises/mnt/M2/Infra`.
- The host must expose `/usr/bin/python3.14` as its Python interpreter (hardcoded in `inventory.ini`).

## 🔐 Secrets

Sensitive variables live in `group_vars/orangepi/vault.yml`. That file is:

- **Plaintext YAML** — not actually ansible-vault encrypted.
- **Gitignored** — it never touches the repository, so nothing secret is ever committed.

It holds: the SSH public key (`my_ssh_pubkey`), the git keypair for the `iac` role (`git_ssh_private_key` / `git_ssh_public_key`), the infra repository URL (`infra_repository`), and Docker registry credentials (`docker_registries`, e.g. ghcr.io).

If a new secret is ever needed, it must be added by the repo owner — never auto-generated or committed.

## 🗺️ Roadmap

- [x] Provision the Orange Pi 5 Plus base system
- [x] Move Docker data-root and infra configs to the M.2 disk (spare the SD card)
- [ ] Add application/services deployment on top of the base system
- [ ] Expand to provisioning the rest of the homelab (NAS, other SBCs, VMs)
- [ ] One-command full-homelab disaster recovery

### ⏰ TODO — before the next SD card format

- [ ] Set a **static/fixed IP** on the Orange Pi so a re-provision always lands on the same address.
- [ ] Set up **fixed SSH keys** (host + user) so the machine keeps its identity across formats.

## 📝 Contributing

This is a personal project, but feedback and ideas are welcome. If you fork it for your own homelab, remember to swap the IPs, UUIDs, and user names in `inventory.ini` and `group_vars/orangepi/vars.yml`.

---

Made with ❤️ and a lot of YAML.