# Port map / EPG contract request – Confluent Platform 8.2.4 (KRaft) via cp-ansible

Applies to **both** branches (`main` = RHEL 9, `rhel8` = RHEL 8). The ports come
from the `confluent.platform` 8.2.4 collection defaults and do not depend on the
OS version:
- Listener config was checked on both the RHEL 9.9 and RHEL 8.10 labs.
- Live TCP flows (`ss -tnp`) were captured on the RHEL 8.10 3+3 lab.

Machine-readable version for the network order: [`epg-port-map.csv`](epg-port-map.csv).

All ports are **TCP**. The current setup is **PLAINTEXT** (no TLS/SASL). If
TLS/SASL is enabled later, the port numbers stay the same unless they are
explicitly overridden.

## Proposed EPGs

| EPG | Members | Count (prod) |
|---|---|---|
| `EPG-KAFKA-CTRL` | KRaft controllers (`confluent-kcontroller`, node.id 9991–9993) | 3 |
| `EPG-KAFKA-BRK` | Kafka brokers (`confluent-server`, node.id 1..n) | 3+ |
| `EPG-KAFKA-CLIENTS` | Producer/consumer applications | – |
| `EPG-ANSIBLE` | Ansible control node(s) running cp-ansible | 1+ |
| `EPG-KAFKA-ADMIN` | Ops/admin hosts (CLI tools, REST admin, monitoring) | – |
| `EPG-SATELLITE` | Red Hat Satellite / Capsule | existing |

## Required contracts (validated: the cluster does not work without these)

| # | Consumer (source) | Provider (destination) | Port | Purpose | Observed in lab |
|---|---|---|---|---|---|
| 1 | `EPG-KAFKA-CTRL` | `EPG-KAFKA-CTRL` | **9093** | KRaft quorum (Raft vote / fetch between controllers) | ✔ controller↔controller |
| 2 | `EPG-KAFKA-BRK` | `EPG-KAFKA-CTRL` | **9093** | Brokers fetch metadata / register / heartbeat with the controller quorum | ✔ every broker → every controller |
| 3 | `EPG-KAFKA-BRK` | `EPG-KAFKA-BRK` | **9091** | Inter-broker listener `BROKER` (replication, cluster-internal) | ✔ broker↔broker |
| 4 | `EPG-KAFKA-CLIENTS` | `EPG-KAFKA-BRK` | **9092** | Client listener `INTERNAL` (produce / consume / admin) | ✔ test clients |
| 5 | `EPG-ANSIBLE` | `EPG-KAFKA-CTRL`, `EPG-KAFKA-BRK` | **22** | SSH for cp-ansible | (lab used docker exec) |
| 6 | `EPG-KAFKA-CTRL`, `EPG-KAFKA-BRK` | `EPG-SATELLITE` | **443** | RHSM + content (BaseOS, AppStream, **CRB**, Confluent mirror) | – |

Not required:
- **Controllers → brokers.** Controllers make **no** outbound connections to
  brokers; in the lab, controllers had only inbound connections on 9093.
- **Brokers → controllers on 9091/9092.** Controllers don't listen there.

ACI note: rules 1 and 3 are **intra-EPG** traffic. If the EPGs are configured
with *intra-EPG isolation = enforced*, add explicit intra-EPG contracts for
9093 (controllers) and 9091 (brokers).

## Recommended / operational

| # | Consumer (source) | Provider (destination) | Port | Purpose |
|---|---|---|---|---|
| 7 | `EPG-KAFKA-ADMIN` | `EPG-KAFKA-BRK` | **9092** | Kafka CLI (`kafka-topics`, `kafka-metadata-quorum`, …) |
| 8 | `EPG-KAFKA-ADMIN` | `EPG-KAFKA-BRK` | **8090** | Confluent Server embedded REST / Admin API v3 (`confluent.http.server.listeners=http://0.0.0.0:8090`; listening by default) |
| 9 | `EPG-KAFKA-ADMIN` | `EPG-KAFKA-CTRL`, `EPG-KAFKA-BRK` | **22** | Ops SSH |
| 10 | `EPG-SATELLITE` | `EPG-KAFKA-CTRL`, `EPG-KAFKA-BRK` | **22** | Satellite Remote Execution (only if used) |

## Outbound internet (only if NOT mirrored via Satellite)

| Source | Destination | Port | Purpose | Avoid with |
|---|---|---|---|---|
| CTRL, BRK | `packages.confluent.io` | 443 | Confluent RPM repo `/rpm/8.2` | Satellite custom product + `repository_configuration: custom`/`none` |
| CTRL, BRK | `s3-us-west-2.amazonaws.com` | 443 | Confluent CLI tarball | `confluent_cli_download_enabled: false` |
| CTRL, BRK | `pypi.org`, `files.pythonhosted.org` | 443 | collection's "Upgrade pip"/`cryptography` | `--skip-tags pip-package` (validated) |
| `EPG-ANSIBLE` | `galaxy.ansible.com`, `pypi.org` | 443 | Install ansible-core + collections | Internal Automation Hub / PyPI mirror |

## Optional – only if the feature is enabled later (closed by default, not listening)

| Feature (var) | Destination EPG | Port | Typical source |
|---|---|---|---|
| Jolokia (`jolokia_enabled: true`) | CTRL / BRK | **7770** (controller) / **7771** (broker) | Monitoring |
| Prometheus JMX exporter (`jmxexporter_enabled: true`) | CTRL / BRK | **8079** (controller) / **8080** (broker) | Prometheus |
| Schema Registry | SR hosts | 8081 (Jolokia 7772) | Clients |
| Kafka REST Proxy (standalone) | REST hosts | 8082 | Clients |
| Kafka Connect | Connect hosts | 8083 | Admin |
| ksqlDB | ksqlDB hosts | 8088 | Clients |
| Control Center (next gen) | C3 host | 9021 (+ its Prometheus 9090) | Admin browsers |

## Not to be opened
- **Ephemeral JVM port** (e.g. `*:39827`, changes on every restart). This is the
  JVM's local JMX connector (`-Dcom.sun.management.jmxremote` without a port).
  Nothing remote uses it, so it must **not** be in any contract.
- Baseline services (DNS 53, NTP 123, syslog, etc.) are assumed to be covered by
  the existing infrastructure EPG/contracts.

## Flow diagram
```
             9092                       9091 (intra-EPG)
 CLIENTS ─────────────► BROKERS ◄───────────────► BROKERS
 ADMIN ──9092/8090/22─►    │
                           │ 9093
                           ▼
                      CONTROLLERS ◄──── 9093 (intra-EPG) ────► CONTROLLERS
 ANSIBLE ──22──► CONTROLLERS + BROKERS ──443──► SATELLITE
```
