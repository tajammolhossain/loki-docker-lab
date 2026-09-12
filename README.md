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
Log Collection             ⏳ Planned
Grafana Alloy              ⏳ Planned
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

* Install Grafana Alloy
* Configure log collection
* Send Linux system logs to Loki
* Collect Docker container logs
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
