# TP3 — HDFS Monitoring & Alerting with Docker

## 📋 Description

This project deploys a complete **Hadoop HDFS cluster** with a full **monitoring and alerting stack** using Docker Compose. It demonstrates how to collect JVM/HDFS metrics, visualize them in Grafana, and trigger alerts when critical thresholds are exceeded.

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Docker Network: hadoopnet                     │
│                                                                      │
│  ┌─────────────┐       ┌─────────────┐                               │
│  │  NameNode    │       │  DataNode    │                              │
│  │  :9870 (UI)  │       │  :9864 (UI)  │        HDFS Cluster         │
│  │  :8020 (RPC) │       │              │                              │
│  │  :7000 (JMX) │       │  :7001 (JMX) │                             │
│  └──────┬───────┘       └──────┬───────┘                             │
│         │   JMX metrics        │   JMX metrics                       │
│         └──────────┬───────────┘                                     │
│                    ▼                                                  │
│  ┌──────────────────────────────┐                                    │
│  │        Prometheus            │                                    │
│  │        :9090                 │──── evaluates ───► alert_rules.yml │
│  │  scrapes metrics every 10s  │                                     │
│  └──────────────┬───────────────┘                                    │
│                 │ fires alerts                                        │
│                 ▼                                                     │
│  ┌──────────────────────────────┐                                    │
│  │       Alertmanager           │                                    │
│  │       :9093                  │                                    │
│  │  routes by severity          │                                    │
│  │  (critical / warning)        │                                    │
│  └──────────────┬───────────────┘                                    │
│                 │                                                     │
│                 ▼                                                     │
│  ┌──────────────────────────────┐                                    │
│  │         Grafana              │                                    │
│  │         :3000                │                                    │
│  │  - HDFS Mini Dashboard       │                                    │
│  │  - HDFS Alerts Dashboard     │                                    │
│  └──────────────────────────────┘                                    │
│                                                                      │
│  ┌─────────────────┐   ┌─────────────────┐                           │
│  │ ResourceManager  │   │  NodeManager     │    YARN Cluster         │
│  │ :8088            │   │                  │                          │
│  └─────────────────┘   └─────────────────┘                           │
└──────────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **JMX Exporter** (java agent) exposes Hadoop/JVM metrics as Prometheus endpoints on NameNode (`:7000`) and DataNode (`:7001`)
2. **Prometheus** scrapes these endpoints every 10 seconds and evaluates alert rules
3. When a rule threshold is breached, Prometheus sends the alert to **Alertmanager**
4. **Alertmanager** routes alerts by severity (critical vs warning) with configurable grouping, inhibition, and repeat intervals
5. **Grafana** visualizes all metrics and alerts via pre-provisioned dashboards and datasources

---

## 📁 Project Structure

```
tp3/
├── docker-compose.yaml                          # All services definition
├── config                                       # Hadoop configuration (env vars)
├── hdfs_dashboard.json                          # Grafana HDFS dashboard (importable)
├── README.md                                    # This file
│
└── monitoring/
    ├── prometheus.yml                           # Prometheus config (scrape + alerting)
    ├── alert_rules.yml                          # Prometheus alerting rules
    ├── alertmanager.yml                         # Alertmanager routing config
    │
    ├── jmx/
    │   ├── jmx_prometheus_javaagent-0.20.0.jar  # JMX Exporter agent
    │   └── hadoop.yml                           # JMX metric mapping rules
    │
    └── grafana/
        └── provisioning/
            ├── datasources/
            │   └── datasources.yml              # Auto-provisions Prometheus + Alertmanager
            └── dashboards/
                ├── dashboards.yml               # Dashboard provider config
                └── json/
                    └── hdfs_alerts_dashboard.json  # Alerts visualization dashboard
```

---

## 🚀 Services

| Service | Image | Port | Description |
|---------|-------|------|-------------|
| **NameNode** | `apache/hadoop:3.3.6` | 9870, 8020, 7000 | HDFS master node (metadata, namespace) |
| **DataNode** | `apache/hadoop:3.3.6` | 9864, 7001 | HDFS worker node (block storage) |
| **ResourceManager** | `apache/hadoop:3.3.6` | 8088 | YARN resource scheduler |
| **NodeManager** | `apache/hadoop:3.3.6` | — | YARN task executor |
| **Prometheus** | `prom/prometheus` | 9090 | Metrics collection & alert evaluation |
| **Alertmanager** | `prom/alertmanager` | 9093 | Alert routing, grouping & deduplication |
| **Grafana** | `grafana/grafana` | 3000 | Visualization dashboards |

---

## ⚠️ Alert Rules

### HDFS Alerts

| Alert | Severity | Condition | Duration |
|-------|----------|-----------|----------|
| `NameNodeDown` | 🔴 Critical | `up{job="namenode"} == 0` | 1 min |
| `DataNodeDown` | 🔴 Critical | `up{job="datanode"} == 0` | 30 sec |
| `HDFSCapacityWarning` | 🟡 Warning | Capacity used > 80% | 5 min |
| `HDFSCapacityCritical` | 🔴 Critical | Capacity used > 90% | 5 min |

### JVM Alerts

| Alert | Severity | Condition | Duration |
|-------|----------|-----------|----------|
| `JVMHeapUsageHigh` | 🟡 Warning | Heap usage > 80% | 5 min |
| `JVMHeapUsageCritical` | 🔴 Critical | Heap usage > 95% | 2 min |

### Alert Routing (Alertmanager)

- **Critical alerts** → grouped, repeat every **1 hour**, wait 10s before first notification
- **Warning alerts** → grouped, repeat every **4 hours**
- **Inhibit rule** → critical alerts suppress warning alerts for the same `alertname` + `instance`

---

## 🖥️ Web Interfaces

| Interface | URL | Credentials |
|-----------|-----|-------------|
| NameNode UI | http://localhost:9870 | — |
| DataNode UI | http://localhost:9864 | — |
| YARN ResourceManager | http://localhost:8088 | — |
| Prometheus | http://localhost:9090 | — |
| Prometheus Alerts | http://localhost:9090/alerts | — |
| Alertmanager | http://localhost:9093 | — |
| Grafana | http://localhost:3000 | `admin` / `admin` |

---

## 📖 Usage

### Start the cluster

```bash
docker compose up -d
```

### Verify all services are running

```bash
docker compose ps
```

### Test alerting (simulate DataNode failure)

```bash
# Stop the DataNode
docker stop datanode

# After ~1 minute, check alerts:
# → http://localhost:9090/alerts   (Prometheus)
# → http://localhost:9093          (Alertmanager)
# → http://localhost:3000          (Grafana dashboard)

# Restart to resolve
docker start datanode
```

### Stop and clean up

```bash
# Stop all containers and remove volumes
docker compose down -v

# Remove all unused Docker images to free disk space
docker system prune -a --volumes -f
```

---

## 🔧 JMX Metrics Exposed

The JMX Exporter translates Hadoop MBeans into Prometheus metrics:

| Metric | Description |
|--------|-------------|
| `hadoop_namenode_fs_namesystem_capacitytotal` | Total HDFS capacity (bytes) |
| `hadoop_namenode_fs_namesystem_capacityused` | Used HDFS capacity (bytes) |
| `hadoop_namenode_fs_namesystem_capacityremaining` | Remaining HDFS capacity (bytes) |
| `hadoop_namenode_fs_namesystem_blockstotal` | Total number of HDFS blocks |
| `hadoop_namenode_fs_namesystem_filestotal` | Total number of files in HDFS |
| `jvm_memory_bytes_used` | JVM heap memory used |
| `jvm_memory_bytes_max` | JVM heap memory max |
| `jvm_threads_*` | JVM thread metrics |

---

## 📊 Grafana Dashboards

### 1. HDFS Mini Dashboard (`hdfs_dashboard.json`)
- HDFS Capacity Used / Remaining
- Blocks & Files Total
- JVM Heap Usage & Threads

### 2. HDFS Alerts Dashboard (auto-provisioned)
- Active Alerts count (stat panel)
- NameNode / DataNode status indicators (UP/DOWN)
- HDFS Capacity Used % gauge
- Current Firing Alerts table
- Alert History timeline
- HDFS Capacity & JVM Heap with threshold lines

---

## 🛠️ Technologies

- **Apache Hadoop 3.3.6** — Distributed storage (HDFS) & processing (YARN)
- **JMX Prometheus Exporter 0.20.0** — Java agent for metric exposure
- **Prometheus** — Time-series metrics collection & alerting engine
- **Alertmanager** — Alert routing, grouping & notification management
- **Grafana** — Metrics visualization & dashboarding
- **Docker Compose** — Container orchestration
