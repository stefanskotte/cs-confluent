# Confluent Platform 8.2 (KRaft) on-prem on RHEL 9 with cp-ansible

> **Branch `onprem-rhel9`.** Production layout for delivered RHEL 9 servers: SSH, sudo, Satellite. No Docker.
>
> | Branch | Purpose |
> |---|---|
> | **`onprem-rhel9`** (this) | On-prem deployment, RHEL 9 |
> | [`onprem-rhel8`](https://github.com/stefanskotte/cs-confluent/tree/onprem-rhel8) | On-prem deployment, RHEL 8 |
> | [`main`](https://github.com/stefanskotte/cs-confluent/tree/main) / [`rhel8`](https://github.com/stefanskotte/cs-confluent/tree/rhel8) | Docker lab validation + background on every adjustment |

## Network / firewall (EPG) port map

**→ [PORTS.md](PORTS.md)** (EPGs, contracts, rationale, flow diagram) ·
**[epg-port-map.csv](epg-port-map.csv)** (for the network/ACI order)

| Source | Destination | Port (TCP) | Purpose |
|---|---|---|---|
| Controllers | Controllers | 9093 | KRaft quorum |
| Brokers | Controllers | 9093 | Metadata / registration |
| Brokers | Brokers | 9091 | Replication (inter-broker) |
| Clients | Brokers | 9092 | Produce / consume |
| Ansible control node | All Kafka hosts | 22 | cp-ansible over SSH |
| All Kafka hosts | Satellite | 443 | RHSM + content (incl. CRB) |
| Admin hosts | Brokers | 8090 | Embedded REST / Admin API (recommended) |

`playbooks/prepare.yml` opens the same ports in **firewalld** on the hosts
(controllers 9093; brokers 9091, 9092, 8090).

## Versions

| Component | Version |
|---|---|
| Managed hosts | RHEL 9 (validated on 9.9) |
| Confluent Platform | **8.2.4** (KRaft only) via `confluent.platform` collection **8.2.4** |
| Java | `java-21-openjdk-headless` (RHEL 9 AppStream; validated 21.0.13) |
| Control node | **ansible-core 2.18.x** (not 2.19), Python 3.11+ |
| Python on hosts | `python3.12` from AppStream (installed by `prepare.yml`) |

## Layout

```
ansible.cfg                         hash_behaviour=merge, skip pip-package, inventory path
requirements.yml                    confluent.platform 8.2.4, ansible.posix, community.general
site.yml                            prepare -> confluent.platform.all -> verify
inventories/prod/hosts.yml          controllers + brokers (FQDNs)  <- EDIT
inventories/prod/group_vars/all.yml all settings                   <- EDIT
playbooks/prepare.yml               python3.12, CRB, preflight checks, firewalld
playbooks/verify.yml                non-destructive health check (safe in prod)
playbooks/failover-test.yml         DISRUPTIVE acceptance test (opt-in)
PORTS.md, epg-port-map.csv          network / EPG order
```

## Prerequisites

**Servers (delivered RHEL 9)**
- 3 controllers + 3 brokers (an odd number of controllers; 3+ brokers for RF=3)
- Registered to **Satellite**, with a Content View that contains:
  - `rhel-9-for-<arch>-baseos-rpms`, `rhel-9-for-<arch>-appstream-rpms`
  - `codeready-builder-for-rhel-9-<arch>-rpms`, needed for `python3.12-packaging`
  - **Confluent 8.2** as a custom product (`https://packages.confluent.io/rpm/8.2`,
    GPG key `https://packages.confluent.io/rpm/8.2/archive.key`)
- DNS: every host's FQDN resolves from every other host *and* from the clients.
  Kafka advertises the inventory hostname.
- A deploy user (default `ansible`) with your SSH key and sudo
- Data disks mounted, if you use them (see `log.dirs` in `group_vars/all.yml`)
- EPG contracts in place (see [PORTS.md](PORTS.md))

**Control node**
- Python 3.11+, network access to the hosts on port 22
- Internet access *or* internal mirrors for PyPI (ansible-core) and Galaxy / Automation Hub (collections)

## Install

```bash
git clone -b onprem-rhel9 https://github.com/stefanskotte/cs-confluent.git && cd cs-confluent

# 1. Control node tooling
python3 -m venv .venv && . .venv/bin/activate
pip install "ansible-core>=2.18,<2.19"
ansible-galaxy collection install -r requirements.yml -p ./collections

# 2. Describe the environment
vi inventories/prod/hosts.yml              # real FQDNs
vi inventories/prod/group_vars/all.yml     # ansible_user, repository_configuration, log.dirs, heap

# 3. Trust the host keys (host_key_checking stays on)
ansible-inventory --list | python3 -c 'import json,sys;print("\n".join(json.load(sys.stdin)["_meta"]["hostvars"]))' \
  | xargs -n1 ssh-keyscan -t ed25519 >> ~/.ssh/known_hosts

# 4. Connectivity + sudo (raw: python3.12 isn't on the hosts yet)
ansible all -m raw -a 'sudo -n true && cat /etc/redhat-release'     # add -K if sudo needs a password

# 5. Deploy: prerequisites -> Confluent Platform -> verification
ansible-playbook site.yml                                           # add -K if sudo needs a password

# 6. (optional, BEFORE go-live) disruptive failover acceptance test
ansible-playbook playbooks/failover-test.yml -e confirm_failover_test=yes
```

`site.yml` is idempotent, so it's safe to re-run. `playbooks/verify.yml` can be
run on its own at any time.

## What `playbooks/prepare.yml` does (and checks before anything is installed)

1. Installs `python3.12` (via `raw`). RHEL 9's python 3.9 is below the collection's 3.10 minimum.
2. Asserts the OS is RHEL `rhel_major`.
3. If the host is registered with RHSM: enables the **CodeReady Builder** repo
   (`enable_crb_repo`). It fails if CRB isn't in the Content View.
4. Installs `python3.12-pip`, `-packaging`, `-cryptography`, `-pyyaml` as RPMs, so there is no PyPI access from the hosts.
5. **Preflight:** `java-21-openjdk-headless` is available, plus `confluent-server-8.2.4` when
   `repository_configuration: none`. This catches a missing Satellite Content View before anything is installed.
6. **Preflight:** every Kafka host resolves every other Kafka host.
7. **firewalld:** if it's running and `manage_firewalld: true`, opens the role's ports (permanent + immediate).

## Key settings (`inventories/prod/group_vars/all.yml`)

| Variable | Value | Why |
|---|---|---|
| `confluent_package_version` | `8.2.4` | Required; match the collection version |
| `redhat_java_package_name` | `java-21-openjdk-headless` | The default `java-21-openjdk` pulls in gtk3/X11 |
| `ansible_python_interpreter` | `/usr/bin/python3.12` | The collection needs Python ≥ 3.10 on hosts |
| `repository_configuration` | `none` | Confluent repo comes from Satellite (`custom` / `confluent` are alternatives) |
| `confluent_cli_download_enabled` | `false` | Avoids downloading from S3 |
| `enable_crb_repo` | `true` | `python3.12-packaging` lives in CRB |
| `manage_firewalld` / `firewalld_zone` | `true` / `public` | cp-ansible doesn't open host firewall ports |
| `ansible.cfg` `hash_behaviour` | `merge` | Required by the collection |
| `ansible.cfg` `[tags] skip` | `pip-package` | Python deps come as RPMs; no PyPI |

Commented examples in the file cover data disks (`log.dirs`) and JVM heap.
The collection defaults are broker `-Xms1g -Xmx6g` and controller `-Xms1536M -Xmx4g`.

## Validation of this branch

Run as an operator would. The only difference was a lab inventory with lab
FQDNs and a small overrides file (no Satellite, so `repository_configuration: confluent`;
small heaps). These were 6 RHEL 9.9 hosts reached over **SSH as the non-root
`ansible` user with sudo**, with no python3.12 and no Java preinstalled. The
branch's `group_vars/all.yml` was used unchanged.

| Step | Result |
|---|---|
| `ansible-playbook site.yml` (fresh) | **0 failed** on 6 hosts; python3.12, Java 21 and CP 8.2.4 installed; pip tasks skipped via `ansible.cfg` |
| verify (part of site.yml) | 6/6 nodes RHEL 9.9 / OpenJDK 21.0.13 / `confluent-server-8.2.4`; ports 9093 and 9091 reachable node-to-node; 3 voters + 3 observers, lag 0; 10/10 messages; topic cleaned up |
| `failover-test.yml` without confirmation | Refuses to run ✔ |
| `failover-test.yml -e confirm_failover_test=yes` | Leader 9991 stopped → 9992 elected, broker 3 down, 200/200 messages, back to 0 under-replicated ✔ |
| `site.yml` re-run | **0 failed**, no restarts (only the collection's cosmetic log4j-backup / format-storage tasks + verify's own topic) |
| firewalld tasks (AlmaLinux 9.9, UBI has no firewalld) | Opened 9091/9092/8090 runtime + permanent ✔ |

**Not validated here, so check on the first real server:**
- `rhsm_repository` against your Satellite (the lab hosts were not registered)
- The `repository_configuration: none` path with a real Satellite custom product
- **SELinux enforcing**: the containers run without SELinux
- TLS/SASL/RBAC: this setup is plaintext, as in the lab

Background on every RHEL 9 adjustment (ansible-core 2.19 bug, hash merging,
Python 3.12, CRB, headless JDK) is in the
[`main` branch README](https://github.com/stefanskotte/cs-confluent/tree/main#what-had-to-be-adjusted-to-make-the-collection-work-on-rhel-9).
