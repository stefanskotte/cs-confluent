# Confluent Platform 8.2 (KRaft) on RHEL 9 with cp-ansible – validated setup

> **Branch `main`.** This branch covers RHEL 9.
> For RHEL 8 see the [`rhel8` branch](https://github.com/stefanskotte/cs-confluent/tree/rhel8).
> Both use the same Confluent versions and vars. RHEL 8 needs **ansible-core 2.16** on the
> control node (2.17+ can't drive `dnf` on RHEL 8's Python 3.6) and `community.general` 11.4.10.

Validated on 2026-11-16 in RHEL 9.9 (UBI 9 + systemd) containers. The
`confluent.platform` collection did the whole install, including Java:
- **1 controller + 1 broker**: install, verify and smoke test
- **3 controllers + 3 brokers** (production-shaped KRaft): fresh install, verify,
  failover test and idempotency rerun. See [3+3 results](#33-cluster-results).

| Component | Version |
|---|---|
| OS (managed hosts) | RHEL 9.9 |
| Ansible collection | `confluent.platform` **8.2.4** (newest 8.2.x on Galaxy) |
| Confluent Platform | **8.2.4** (`confluent-server-8.2.4-1`, Kafka 4.2 line, KRaft only) |
| Java | `java-21-openjdk-headless` **21.0.13** (RHEL AppStream) |
| ansible-core (control node) | **2.18.x** (2.19 does **not** work, see below) |
| Python (managed hosts) | **3.12** (`/usr/bin/python3.12`), not the RHEL 9 system 3.9 |

Note: CP **8.2.5** RPMs are already in the Confluent repo, but there is no
8.2.5 collection yet. Keep the collection and the package version at the same
patch level (8.2.4/8.2.4).

## What had to be adjusted to make the collection work on RHEL 9

| # | Problem hit | Fix |
|---|---|---|
| 1 | ansible-core **2.19** fails in `roles/variables/tasks/mtls.yml`: `combine ... expected dicts but got '_AnsibleTaggedStr'` (2.19 data-tagging) | Run from a venv with `ansible-core>=2.18,<2.19` |
| 2 | `Hash Merging must be enabled in ansible.cfg` | `hash_behaviour = merge` in `ansible.cfg` |
| 3 | `You must upgrade Python to at least 3.10` (RHEL 9 python3 = 3.9.25) | Install `python3.12`, set `ansible_python_interpreter: /usr/bin/python3.12` |
| 4 | `Upgrade pip`: `No module named 'packaging'` (the collection only installs it for RHEL 10) | RPMs `python3.12-pip`, `python3.12-packaging` |
| 5 | log4j task: `The python PyYAML module is required` (collection installs the 3.9 `python3-pyyaml`) | RPM `python3.12-pyyaml` |
| 6 | Collection pip-installs `cryptography` from PyPI | RPM `python3.12-cryptography`, then run with `--skip-tags pip-package` (no PyPI needed) |
| 7 | Default `java-21-openjdk` pulls in gtk3/tracker/X11 desktop deps | `redhat_java_package_name: java-21-openjdk-headless` |

Items 3–6 are handled by `prepare.yml`, which runs before the collection.

### RHSM / Satellite requirements
- `python3.12-packaging` is **only in CodeReady Builder**:
  `codeready-builder-for-rhel-9-<arch>-rpms` must be synced in Satellite and
  included in the hosts' Content View / activation key. `prepare.yml` enables
  it with `rhsm_repository` when the host is registered. That fails if the repo
  is not in the Content View.
- Everything else (`python3.12*`, `java-21-openjdk-headless`) comes from
  BaseOS/AppStream.
- Confluent RPMs come from `packages.confluent.io/rpm/8.2` by default. If the
  hosts can't reach it, mirror it in Satellite (custom product, GPG key
  `https://packages.confluent.io/rpm/8.2/archive.key`) and set
  `repository_configuration: custom` (+ `custom_yum_repofile_filepath`) or `none`.
- The Confluent CLI is downloaded from S3. Set `confluent_cli_download_enabled: false` if there's no internet.
- Use `--skip-tags pip-package` to avoid PyPI entirely. This was validated: the
  rerun had 0 failures and no restarts.

## Vars for the existing playbook
See **`customer-vars.yml`**. The core settings:

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
```bash
python3 -m venv .venv && . .venv/bin/activate
pip install "ansible-core>=2.18,<2.19"
ansible-galaxy collection install -r requirements.yml -p ./collections
#   requirements.yml: confluent.platform 8.2.4 (+ community.general for rhsm_repository)
```

## Run order
```bash
ansible-playbook prepare.yml                                   # python3.12 + deps (CRB)
ansible-playbook confluent.platform.all --skip-tags pip-package
ansible-playbook verify.yml                                    # checks below
```

## 3+3 cluster results

Topology: `kafka-controller1..3` (node.id 9991–9993, `process.roles=controller`) and
`kafka-broker1..3` (node.id 1–3, `process.roles=broker`), on 6 separate RHEL 9.9 hosts.

| Step | Result |
|---|---|
| `prepare.yml` | 6/6 ok (python3.12 + pip/packaging/cryptography/pyyaml RPMs) |
| `confluent.platform.all --skip-tags pip-package` (fresh) | **0 failed** on 6 hosts, no PyPI access needed |
| Per-node checks | 6/6: RHEL 9.9, `openjdk 21.0.13 LTS`, `confluent-server-8.2.4-1`, service active |
| KRaft quorum | **3 voters** (9991/9992/9993), **3 observers** (brokers 1–3), MaxFollowerLag 0 |
| Brokers | 3/3 registered, `isFenced: false` |
| Topic `smoke-test` | 6 partitions, **RF=3**, `min.insync.replicas=2`, leaders spread over 1/2/3, ISR full, 0 under-replicated |
| Produce batch A | 100 msgs, `acks=all` ✔ |
| **Failover**: stopped controller *leader* (9992 / kafka-controller2) **and** kafka-broker3 | New controller leader elected (**9991**) |
| Produce batch B while degraded | 100 msgs, `acks=all` succeeded with 2/3 ISR ✔ |
| Consume while degraded | **200/200** messages (A+B) ✔ |
| Restart stopped nodes | Rejoined; **0 under-replicated partitions**; all 6 nodes lag 0 at offset 1004 |
| Idempotency rerun of the collection | **0 failed**, no service restarts (only cosmetic "changed" tasks: CLI symlink, log4j backup, `Format Storage Directory` on already-formatted dirs) |

Note: cp-ansible 8.2.4 configures a **static** KRaft quorum
(`controller.quorum.voters=9991@...,9992@...,9993@...`; controller directory IDs
show as `AAAA…`). It does not use the dynamic KIP-853 quorum, so adding or
replacing controllers later is a planned operation, not a hot add.

## What `verify.yml` checks
- Every node: RHEL 9, OpenJDK 21, `confluent-server-8.2.4`, systemd service active
- KRaft: #voters == #controllers, #observers == #brokers, follower lag 0
- All brokers unfenced; RF=min(3, brokers) topic with minISR 2; no under-replicated partitions
- Failover: stop the controller leader + the last broker → new leader elected →
  produce (`acks=all`) and consume succeed → restart → back to fully in sync

## Lab files (this directory)
- `Dockerfile.rhel9`, `docker-compose.yml`: 6× RHEL 9 systemd containers (3 controllers + 3 brokers, no Java preinstalled)
- `inventory.yml`: lab inventory (docker connection; use SSH for real hosts).
  It contains **lab-only** small heaps (`*_service_environment_overrides.KAFKA_HEAP_OPTS`)
  to fit 6 JVMs in 8 GB. Do **not** copy those to production; the collection
  defaults are broker `-Xms1g -Xmx6g` and controller `-Xms1536M -Xmx4g`.
- `ansible.cfg`, `requirements.yml`, `prepare.yml`, `verify.yml`, `customer-vars.yml`
- `install.log`, `install-rerun.log`, `verify.log`: run evidence
- Tear down: `docker compose down`
