# Confluent Platform 8.2 (KRaft) on RHEL 8 with cp-ansible – validated setup

> **Branch `rhel8`.** This branch covers RHEL 8.
> For RHEL 9 see the [`main` branch](https://github.com/stefanskotte/cs-confluent/tree/main).
> Both use the same Confluent versions; the main difference is the **ansible-core version on the control node**.

Validated on 2026-11-17 in RHEL 8.10 (UBI 8 + systemd) containers with
**3 controllers + 3 brokers**. The `confluent.platform` collection did the whole
install, including Java. The run covered a fresh install, verify, a failover
test and an idempotency rerun; see [3+3 results](#33-cluster-results).
>
> **On-prem deployment (no Docker):** [`onprem-rhel9`](https://github.com/stefanskotte/cs-confluent/tree/onprem-rhel9) ·
> [`onprem-rhel8`](https://github.com/stefanskotte/cs-confluent/tree/onprem-rhel8). These branches (main/rhel8)
> are the Docker lab validation and background.

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

The same ports apply to RHEL 8 and RHEL 9. See PORTS.md for optional ports,
internet egress, and ports that must *not* be opened.

| Component | Version |
|---|---|
| OS (managed hosts) | RHEL 8.10 |
| Ansible collection | `confluent.platform` **8.2.4** (newest 8.2.x on Galaxy) |
| Confluent Platform | **8.2.4** (`confluent-server-8.2.4-1`, Kafka 4.2 line, KRaft only) |
| Java | `java-21-openjdk-headless` **21.0.13** (RHEL 8 AppStream) |
| ansible-core (control node) | **2.16.x** (2.17+ **cannot** manage RHEL 8, see below) |
| Supporting collections | `community.general` **11.4.10**, `community.docker` 4.8.9 (lab only) |
| Python (managed hosts) | **3.12** (`/usr/bin/python3.12`), not the RHEL 8 system 3.6 |

Note: CP **8.2.5** RPMs are already in the Confluent repo, but there is no
8.2.5 collection yet. Keep the collection and the package version at the same
patch level (8.2.4/8.2.4).

## RHEL 8 vs RHEL 9 at a glance

| | RHEL 8 (this branch) | RHEL 9 (`main`) |
|---|---|---|
| ansible-core on control node | **2.16.x** | 2.18.x (not 2.19) |
| Control-node Python | 3.10–3.12 (2.16 doesn't support newer) | 3.11+ |
| `community.general` | pin **11.4.10** (12+ needs 2.17) | latest |
| System python on hosts | 3.6 | 3.9 |
| Everything else (vars, `prepare.yml`, RPM names, CRB, run order) | identical | identical |

## What had to be adjusted to make the collection work on RHEL 8

| # | Problem hit | Fix |
|---|---|---|
| 1 | ansible-core 2.17+/2.18: every `dnf` task fails with `Could not import the dnf python module using /usr/bin/python3.12 ... attempted ['/usr/libexec/platform-python', ...]`. On RHEL 8 the dnf bindings only exist for platform-python **3.6**, and ansible-core ≥ 2.17 can't run modules on Python 3.6 | Use **ansible-core 2.16.x** on the control node. It's the collection's minimum and also the version Red Hat ships and supports in RHEL 8 AppStream |
| 2 | ansible-core 2.19 fails in `roles/variables/tasks/mtls.yml` (`combine ... '_AnsibleTaggedStr'`) | Covered by #1 (stay on 2.16) |
| 3 | `Collection community.general does not support Ansible version 2.16` | Pin `community.general` **11.4.10** (and `community.docker` 4.8.9 for the lab) |
| 4 | `Hash Merging must be enabled in ansible.cfg` | `hash_behaviour = merge` in `ansible.cfg` |
| 5 | `You must upgrade Python to at least 3.10` (RHEL 8 python3 = 3.6.8) | Install `python3.12`, set `ansible_python_interpreter: /usr/bin/python3.12` |
| 6 | `Upgrade pip`: `No module named 'packaging'` | RPMs `python3.12-pip`, `python3.12-packaging` |
| 7 | log4j task: `The python PyYAML module is required` | RPM `python3.12-pyyaml` |
| 8 | Collection pip-installs `cryptography` from PyPI | RPM `python3.12-cryptography`, then run with `--skip-tags pip-package` (no PyPI needed) |
| 9 | Default `java-21-openjdk` pulls in gtk3/X11 desktop deps | `redhat_java_package_name: java-21-openjdk-headless` |

Items 5–8 are handled by `prepare.yml`, which runs before the collection. All
`python3.12*` RPMs and `java-21-openjdk-headless` are in RHEL 8.10 AppStream,
except `python3.12-packaging` (CRB, see below).

### RHSM / Satellite requirements
- `python3.12-packaging` is **only in CodeReady Builder**:
  `codeready-builder-for-rhel-8-<arch>-rpms` must be synced in Satellite and
  included in the hosts' Content View / activation key. `prepare.yml` enables
  it with `rhsm_repository` when the host is registered (the repo name is
  derived from the OS major version). That fails if the repo is not in the
  Content View.
- Everything else (`python3.12*`, `java-21-openjdk-headless`) comes from
  BaseOS/AppStream. This was validated against RHEL **8.10** content; make sure the Content View is at 8.10 (older minors may lack `python3.12*`).
- Confluent RPMs come from `packages.confluent.io/rpm/8.2` by default (noarch,
  same repo as RHEL 9). If the hosts can't reach it, mirror it in Satellite
  (custom product, GPG key `https://packages.confluent.io/rpm/8.2/archive.key`)
  and set `repository_configuration: custom` (+ `custom_yum_repofile_filepath`) or `none`.
- The Confluent CLI is downloaded from S3. Set `confluent_cli_download_enabled: false` if there's no internet.
- Use `--skip-tags pip-package` to avoid PyPI entirely. This was validated on RHEL 8.

## Vars for the existing playbook
See **`customer-vars.yml`**. These are the same as RHEL 9:

```yaml
confluent_package_version: 8.2.4
install_java: true
redhat_java_package_name: java-21-openjdk-headless
ansible_python_interpreter: /usr/bin/python3.12
```
plus in `ansible.cfg`:
```ini
[defaults]
hash_behaviour = merge
```

## Control node
ansible-core 2.16 needs Python **3.10–3.12** on the control node:
```bash
python3.12 -m venv .venv216 && . .venv216/bin/activate
pip install "ansible-core>=2.16,<2.17"
ansible-galaxy collection install -r requirements.yml -p ./collections
#   requirements.yml pins: confluent.platform 8.2.4, community.general 11.4.10, community.docker 4.8.9
```
A RHEL 8 control node can use the Red Hat package instead: `dnf install ansible-core` (2.16 in RHEL 8.10 AppStream).

## Run order
```bash
ansible-playbook prepare.yml                                   # python3.12 + deps (CRB)
ansible-playbook confluent.platform.all --skip-tags pip-package
ansible-playbook verify.yml                                    # checks below (set rhel_major: '8')
```

## 3+3 cluster results

Topology: `kafka-controller1..3` (node.id 9991–9993, `process.roles=controller`) and
`kafka-broker1..3` (node.id 1–3, `process.roles=broker`), on 6 separate RHEL 8.10 hosts.

| Step | Result |
|---|---|
| `prepare.yml` (ansible-core 2.16.20) | 6/6 ok (python3.12 + pip/packaging/cryptography/pyyaml RPMs) |
| `confluent.platform.all --skip-tags pip-package` (fresh) | **0 failed** on 6 hosts, no PyPI access needed |
| Per-node checks | 6/6: RHEL 8.10, `openjdk 21.0.13 LTS`, `confluent-server-8.2.4-1`, `java-21-openjdk-headless-21.0.13.0.9-1.1.el8`, service active |
| KRaft quorum | **3 voters** (9991/9992/9993), **3 observers** (brokers 1–3), MaxFollowerLag 0 |
| Brokers | 3/3 registered, `isFenced: false` |
| Topic `smoke-test` | 6 partitions, **RF=3**, `min.insync.replicas=2`, 0 under-replicated |
| Produce batch A | 100 msgs, `acks=all` ✔ |
| **Failover**: stopped controller *leader* (9993 / kafka-controller3) **and** kafka-broker3 | New controller leader elected (**9991**) |
| Produce batch B while degraded | 100 msgs, `acks=all` succeeded with 2/3 ISR ✔ |
| Consume while degraded | **200/200** messages (A+B) ✔ |
| Restart stopped nodes | Rejoined; **0 under-replicated partitions**; all 6 nodes lag 0 |
| Idempotency rerun of the collection | **0 failed**, no service restarts (only cosmetic "changed" tasks: CLI symlink, log4j backup, `Format Storage Directory` on already-formatted dirs); no collection-version warnings with the pinned collections |

Note: cp-ansible 8.2.4 configures a **static** KRaft quorum
(`controller.quorum.voters=9991@...,9992@...,9993@...`; controller directory IDs
show as `AAAA…`). It does not use the dynamic KIP-853 quorum, so adding or
replacing controllers later is a planned operation, not a hot add.

## What `verify.yml` checks
- Every node: RHEL `rhel_major` (inventory var, default 9), OpenJDK 21, `confluent-server-8.2.4`, systemd service active
- KRaft: #voters == #controllers, #observers == #brokers, follower lag 0
- All brokers unfenced; RF=min(3, brokers) topic with minISR 2; no under-replicated partitions
- Failover: stop the controller leader + the last broker → new leader elected →
  produce (`acks=all`) and consume succeed → restart → back to fully in sync

## Lab files (this directory)
- `Dockerfile.rhel8`, `docker-compose.yml`: 6× RHEL 8.10 systemd containers (3 controllers + 3 brokers, no Java preinstalled)
- `inventory.yml`: lab inventory (docker connection; use SSH for real hosts).
  It contains **lab-only** small heaps (`*_service_environment_overrides.KAFKA_HEAP_OPTS`)
  to fit 6 JVMs in 8 GB. Do **not** copy those to production; the collection
  defaults are broker `-Xms1g -Xmx6g` and controller `-Xms1536M -Xmx4g`.
- `ansible.cfg`, `requirements.yml`, `prepare.yml`, `verify.yml`, `customer-vars.yml`
- `install.log`, `install-rerun.log`, `prepare.log`, `verify.log`: run evidence (RHEL 8 runs)
- `PORTS.md`, `epg-port-map.csv`: port map / EPG contract request
- Tear down: `docker compose down`
