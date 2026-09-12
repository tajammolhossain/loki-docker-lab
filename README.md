# Loki Docker Lab

A practical Grafana Loki logging lab deployed using Docker Compose on Ubuntu Server 24.04.4 LTS.

This lab deploys Grafana Loki as a centralized log storage and querying backend and integrates it with Grafana for log visualization and exploration.

## Author

**S.M. Tajammol Hossain**

---

## Lab Environment

| Component        | Details                   |
| ---------------- | ------------------------- |
| Hostname         | `monitoring-vm`           |
| IP Address       | `192.168.0.196`           |
| Operating System | Ubuntu Server 24.04.4 LTS |
| Loki             | 3.7.0                     |
| Grafana          | 13.2.1                    |
| Deployment       | Docker Compose            |
| Loki Port        | `3100`                    |
| Grafana Port     | `3000`                    |

---

## Architecture

```text
                    monitoring-vm
                    192.168.0.196
                         |
          +--------------+--------------+
          |                             |
          v                             v
     +---------+                  +-------------+
     | Grafana |                  |    Loki     |
     |  :3000  |----------------->|    :3100    |
     +---------+     HTTP API     +-------------+
          |                             |
          |                             |
          v                             v
     Grafana Explore              Log Storage
```

### Current Data Flow

```text
Loki
  |
  | HTTP API
  v
Grafana
  |
  v
Explore
```

Log collection using Grafana Alloy is planned as the next stage.

---

## Directory Structure

```text
/opt/monitoring/
└── loki/
    ├── compose.yml
    └── loki-config.yaml
```

---

## Loki Configuration

See [`loki-config.yaml`](./loki-config.yaml) for the full configuration.

---

## Docker Compose Configuration

See [`compose.yml`](./compose.yml) for the full configuration.

---

## Deployment

Create the Loki directory:

```bash
sudo mkdir -p /opt/monitoring/loki
sudo chown -R $USER:$USER /opt/monitoring/loki
cd /opt/monitoring/loki
```

Place `loki-config.yaml` and `compose.yml` in this directory.

---

## Validate Docker Compose Configuration

```bash
sudo docker compose -f compose.yml config
```

The configuration was successfully validated before deployment.

---

## Start Loki

```bash
sudo docker compose -f compose.yml up -d
```

Check the container:

```bash
sudo docker compose -f compose.yml ps
```

Expected:

```text
NAME   IMAGE                STATUS
loki   grafana/loki:3.7.0  Up
```

---

## Troubleshooting

During the initial deployment, Loki entered a restart loop because retention was enabled without configuring the required delete request store.

The error was:

```text
CONFIG ERROR: invalid compactor config:
compactor.delete-request-store should be configured when retention is enabled
```

For this lab, retention was disabled:

```yaml
compactor:
  working_directory: /loki/compactor
  retention_enabled: false
```

After changing the configuration, Loki started successfully.

---

## Loki Health Check

Check the readiness endpoint:

```bash
curl http://127.0.0.1:3100/ready
```

Expected:

```text
ready
```

During initial startup, Loki may temporarily return:

```text
Ingester not ready: waiting for 15s after being ready
```

After the startup period, the endpoint should return:

```text
ready
```

---

## Loki Metrics

Verify that Loki is exposing Prometheus-style metrics:

```bash
curl -s http://127.0.0.1:3100/metrics | head -n 10
```

Example:

```text
# HELP deprecated_flags_inuse_total The number of deprecated flags currently set.
# TYPE deprecated_flags_inuse_total counter
deprecated_flags_inuse_total 0
```

---

## Loki Logs

Check Loki container logs:

```bash
sudo docker logs --tail 50 loki
```

A successful startup contains:

```text
Loki started
```

and:

```text
compactor is ACTIVE in the ring
```

---

## Docker Network Integration

Grafana and Loki were initially running on separate Docker networks:

```text
grafana_default
loki_default
```

Loki was connected to the Grafana network:

```bash
sudo docker network connect grafana_default loki
```

This allows the Grafana container to communicate with Loki using:

```text
http://loki:3100
```

Connectivity was verified with:

```bash
sudo docker exec grafana curl -s http://loki:3100/ready
```

Expected result:

```text
ready
```

---

## Grafana Integration

In Grafana:

```text
Connections
    ↓
Data Sources
    ↓
Add data source
    ↓
Loki
```

Loki URL:

```text
http://loki:3100
```

The Loki datasource was successfully connected to Grafana and opened in Grafana Explore.

---

## Log Collection with Grafana Alloy

Grafana Alloy is used on the Linux host to collect system (`systemd-journal`) and Docker container logs and forward them to this Loki instance.

See [`config.alloy`](./config.alloy) for the full Alloy configuration used in this lab.

### Install Grafana Alloy

```bash
sudo apt update
sudo apt install gpg

sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc \
https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc

echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | \
sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install alloy

alloy --version
```

### Alloy Service

```bash
sudo systemctl status alloy
sudo systemctl start alloy
sudo systemctl enable alloy
sudo systemctl restart alloy
sudo systemctl stop alloy
```

The configuration file lives at `/etc/alloy/config.alloy` on the host — use the [`config.alloy`](./config.alloy) file in this repository as its content. It forwards logs to this Loki instance at `http://127.0.0.1:3100/loki/api/v1/push`, collects the systemd journal, and collects Docker container logs via the Docker socket.

### Docker Permissions

Alloy's service user needs access to the Docker socket to collect container logs:

```bash
id alloy
sudo -u alloy docker ps
```

### Validate and Monitor

```bash
sudo alloy fmt /etc/alloy/config.alloy
sudo systemctl status alloy
sudo journalctl -u alloy -f
```

Check ingestion metrics:

```bash
curl -s http://127.0.0.1:12345/metrics | grep 'loki_write_sent_entries_total'
curl -s http://127.0.0.1:12345/metrics | grep 'loki_write_dropped_entries_total'
```

### Troubleshooting: `entry too far behind`

If Loki rejects entries with:

```text
entry too far behind
```

it means a log entry is older than the timestamp window Loki currently accepts for a stream. This lab's `loki-config.yaml` sets:

```yaml
ingester:
  max_chunk_age: 24h
```

to provide a larger out-of-order ingestion window. Validate any Loki config change before restarting the container:

```bash
docker run --rm \
  -v /opt/monitoring/loki/loki-config.yaml:/etc/loki/config.yaml:ro \
  grafana/loki:3.7.0 \
  -config.file=/etc/loki/config.yaml \
  -verify-config=true

sudo docker restart loki
curl -s http://127.0.0.1:3100/ready
```

> Increasing `max_chunk_age` can increase memory usage and should be evaluated carefully for production deployments.

### Security Notes

* The Docker socket (`/var/run/docker.sock`) grants powerful access to the Docker daemon — restrict permissions appropriately.
* Protect Loki from unauthorized network access.
* Use authentication and TLS when logs leave the local host.
* Monitor Alloy and Loki health, and watch for dropped entries.
* Avoid unnecessarily large `max_chunk_age` values.

---

## Current Status

```text
Loki 3.7.0                 ✅
Docker Compose             ✅
Loki Container             ✅
Loki /ready Endpoint       ✅
Loki Metrics Endpoint      ✅
Filesystem Storage         ✅
Compactor                  ✅
Grafana Integration        ✅
Grafana Explore            ✅
Grafana Alloy              ✅
Log Collection             ✅
```

---

## Monitoring Stack

Current monitoring VM stack:

```text
monitoring-vm
192.168.0.196

├── Zabbix 7.4.14
│   └── PostgreSQL 17
│
├── Prometheus 3.14.0
│
├── Node Exporter
│
├── Grafana 13.2.1
│
├── Loki 3.7.0
│
└── Uptime Kuma
    └── Planned
```

---

## Next Steps

* Query logs using LogQL
* Build Grafana log dashboards
* Configure log-based alerts
* Integrate Loki with the wider monitoring stack
* Deploy Uptime Kuma

---

## Learning Objectives

This lab demonstrates practical implementation of:

* Grafana Loki deployment using Docker Compose
* Loki TSDB storage
* Filesystem-based log storage
* Loki health monitoring
* Loki HTTP API
* Docker network integration
* Grafana Loki datasource integration
* Grafana Explore
* Centralized log management
* Log querying with LogQL
* Grafana Alloy deployment and configuration
* systemd journal log collection
* Docker container log collection via Docker service discovery
* Loki ingestion troubleshooting and tuning
