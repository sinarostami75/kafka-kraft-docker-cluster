# Kafka 3-Node Cluster in Docker (KRaft Mode)

This repository provides a **clean, production-style Kafka cluster** running in **KRaft mode (no Zookeeper)** using **Docker Compose**.

The goal of this project is to offer a **stable reference implementation** for running Kafka locally with:

* Three nodes
* Proper listener separation
* Metadata quorum
* Replication enabled
* Prometheus-ready JMX metrics

This repository focuses **only on the final, working configuration**.

---
## 🎥 Video Walkthrough

A complete step-by-step walkthrough of this project is available on YouTube:
👉 https://www.youtube.com/watch?v=XXXXXXXX

The video explains:
- Cluster architecture
- KRaft quorum behavior
- Metrics configuration
- How to run and verify the setup

---
## 📦 Project Overview

* Kafka version: **7.6.1 (Confluent Platform)**
* Mode: **KRaft (Kafka Raft Metadata mode)**
* Nodes: **3 (broker + controller)**
* Zookeeper: **Not used**
* Orchestration: **Docker Compose**
* Metrics: **Prometheus JMX Exporter**

---

## 📁 Repository Structure

```
.
├── docker-compose.yml          # Final, working Kafka cluster
├── jmx-exporter/               # Prometheus JMX exporter files
│   ├── jmx_prometheus_javaagent.jar
│   └── kafka.yml
├── data/                       # Kafka data directories (not committed)
│   ├── kafka1/
│   ├── kafka2/
│   └── kafka3/
└── README.md
```

> The `data/` directory should be added to `.gitignore`.

---

## 🔧 Prerequisites

* Docker
* Docker Compose (v2)
* Linux or macOS (tested on Linux)
* `wget` installed

---

## 📥 Download Prometheus JMX Exporter

Kafka metrics are exposed using the Prometheus JMX Java Agent.

Download and rename the agent exactly as shown below:

```bash
cd jmx-exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar
mv jmx_prometheus_javaagent-0.20.0.jar jmx_prometheus_javaagent.jar
```

The file name **must match** the one referenced in `docker-compose.yml`.

---

## 🆔 Generate the KRaft Cluster ID

Kafka running in KRaft mode requires a **cluster ID**.

Generate it once using the following command:

```bash
docker run --rm confluentinc/cp-kafka:7.6.1 bash -lc "kafka-storage random-uuid"
```

Copy the generated UUID and set it as the value of `CLUSTER_ID` **for all three services** in `docker-compose.yml`.

> All brokers must share the **same** cluster ID.

---

## 🔐 Fix Data Directory Permissions

The Kafka container runs as user **UID 1000**.

Ensure the local data directories are writable:

```bash
sudo chown -R 1000:1000 ./data
sudo chmod -R u+rwX,g+rwX ./data
```

---

## ▶️ Start the Kafka Cluster

Always start with a clean state when running the cluster for the first time:

```bash
docker compose down -v
rm -rf ./data
mkdir -p ./data/kafka1 ./data/kafka2 ./data/kafka3
sudo chown -R 1000:1000 ./data
```

Start the cluster:

```bash
docker compose up -d
```

Verify that all containers are running:

```bash
docker compose ps
```

---

## 📊 Verify Metrics Endpoints

Each broker exposes Prometheus metrics on a dedicated port:

* Broker 1 → `http://localhost:19101/metrics`
* Broker 2 → `http://localhost:19102/metrics`
* Broker 3 → `http://localhost:19103/metrics`

Test using `curl`:

```bash
curl -s http://localhost:19101/metrics | head
curl -s http://localhost:19102/metrics | head
curl -s http://localhost:19103/metrics | head
```

---

## 🧠 Verify KRaft Metadata Quorum

Check quorum status:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-metadata-quorum --bootstrap-server kafka1:19092 describe --status"
```

Check replication state:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-metadata-quorum --bootstrap-server kafka1:19092 describe --replication"
```

Expected state:

* One leader
* Three voters
* Zero replication lag

---

## 📌 Running Kafka CLI Commands

Kafka CLI tools inherit environment variables from the container.

To avoid conflicts with the JMX Java agent, always clear `KAFKA_OPTS` when running CLI commands:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-topics --bootstrap-server kafka1:19092 --list"
```

---

## 🧪 Create and Verify a Replicated Topic

Create a topic with replication factor 3:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-topics --bootstrap-server kafka1:19092 --create --topic test --partitions 3 --replication-factor 3"
```

Describe the topic:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-topics --bootstrap-server kafka1:19092 --describe --topic test"
```

---

## 📄 Notes

* This repository provides a **baseline Kafka cluster configuration**.
* It is suitable for local development, testing, and learning KRaft-based deployments.
* For production usage, additional considerations are required (security, monitoring, backups).

---

## ✅ License

MIT
