# Kafka 3-Node Cluster in Docker (KRaft Mode)

This repository contains a **real, production‑style Kafka 3‑node cluster** running in **KRaft mode (no Zookeeper)** using Docker Compose.

> This project is intentionally designed for **learning by debugging real failures** — not just a happy‑path demo.
> It is the companion repository for a YouTube video where the cluster is built, broken, and fixed step by step.

---

## 🎯 What This Project Covers

* Kafka **KRaft mode** (no Zookeeper)
* 3 nodes acting as **broker + controller**
* Proper **internal vs external listeners**
* Real **replication & ISR behavior**
* **Prometheus JMX exporter** per broker
* Debugging common Kafka‑in‑Docker failures

This is **not** a toy example. Every problem shown here is something people hit in real life.

---

## 📦 Repository Structure

```
.
├── docker-compose.yml          # Final working baseline
├── steps/                      # Intentionally broken configs (used in the video)
│   ├── step-01-no-cluster-id.yml
│   ├── step-02-bad-metrics-port.yml
│   ├── step-03-bad-jar.yml
│   └── step-04-permission.yml
├── data/                       # Kafka data directories (gitignored)
│   ├── kafka1/
│   ├── kafka2/
│   └── kafka3/
├── jmx-exporter/
│   ├── jmx_prometheus_javaagent.jar
│   └── kafka.yml
└── README.md
```

---

## 🚀 Prerequisites

* Docker
* Docker Compose (v2)
* Linux / macOS (tested on Linux)
* `wget` installed

---

## 📥 JMX Exporter Setup

The Prometheus JMX exporter **must be a real JAR file**.

Download and rename it exactly like this:

```bash
cd jmx-exporter
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.20.0/jmx_prometheus_javaagent-0.20.0.jar
mv jmx_prometheus_javaagent-0.20.0.jar jmx_prometheus_javaagent.jar
```

If this file is missing, corrupted, or renamed incorrectly, **Kafka will crash at startup**.

---

## 🆔 Generate the KRaft Cluster ID

Kafka in KRaft mode **will not start without a cluster ID**.

Generate one UUID **once**, and reuse it for all brokers:

```bash
docker run --rm confluentinc/cp-kafka:7.6.1 bash -lc "kafka-storage random-uuid"
```

Copy the output and place it into **`CLUSTER_ID` for all three services** in `docker-compose.yml`.

> ⚠️ Using different IDs means you do **not** have a cluster.

---

## 🔐 Fix Data Directory Permissions

The Kafka container runs as **UID 1000**.
Your local data directories must be writable by this user:

```bash
sudo chown -R 1000:1000 ./data
sudo chmod -R u+rwX,g+rwX ./data
```

If you skip this step, Kafka will fail with a *data directory not writable* error.

---

## ▶️ Start the Final Working Cluster

Always start clean when testing:

```bash
docker compose down -v
rm -rf ./data
mkdir -p ./data/kafka1 ./data/kafka2 ./data/kafka3
sudo chown -R 1000:1000 ./data
docker compose up -d
```

Check status:

```bash
docker compose ps
```

---

## 📊 Verify JMX Metrics

Each broker exposes metrics on its own port:

* Broker 1 → `http://localhost:19101/metrics`
* Broker 2 → `http://localhost:19102/metrics`
* Broker 3 → `http://localhost:19103/metrics`

Test:

```bash
curl -s http://localhost:19101/metrics | head
curl -s http://localhost:19102/metrics | head
curl -s http://localhost:19103/metrics | head
```

> If `curl` prints a warning when piped to `head`, that is normal.

---

## ⚠️ Important: Running Kafka CLI Commands

Because `KAFKA_OPTS` includes the JMX Java agent, **Kafka CLI tools may crash** if they inherit it.

Always clear `KAFKA_OPTS` when running CLI commands inside containers:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-topics --bootstrap-server kafka1:19092 --list"
```

This is a **very common and very confusing issue**.

---

## 🧠 Verify KRaft Quorum

Check controller quorum status:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-metadata-quorum --bootstrap-server kafka1:19092 describe --status"
```

Check replication state:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-metadata-quorum --bootstrap-server kafka1:19092 describe --replication"
```

You should see:

* 3 voters
* 1 leader
* 0 lag

---

## 🧪 Test Real Replication & Failover

Create a replicated topic:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-topics --bootstrap-server kafka1:19092 --create --topic test --partitions 3 --replication-factor 3"
```

Describe it:

```bash
docker exec -it -e KAFKA_OPTS= kafka1 bash -lc "kafka-topics --bootstrap-server kafka1:19092 --describe --topic test"
```

Simulate a broker failure:

```bash
docker stop kafka2
```

ISR should shrink but remain available (`min.insync.replicas=2`).

Bring it back:

```bash
docker start kafka2
```

After catch‑up, ISR should return to all three brokers.

---

## 🧨 The `steps/` Directory (Intentional Failures)

The `steps/` folder contains **broken docker‑compose files** used in the video:

| Step    | Problem Demonstrated       |
| ------- | -------------------------- |
| step‑01 | Missing CLUSTER_ID         |
| step‑02 | JMX metrics port conflict  |
| step‑03 | Invalid or missing JAR     |
| step‑04 | Data directory permissions |

Each step demonstrates:

* The exact error
* Why it happens
* How to fix it properly

---

## 📌 Who This Repo Is For

* Backend engineers
* DevOps engineers
* Anyone who tried Kafka in Docker and got stuck

If you’re looking for a **copy‑paste demo**, this is not it.
If you want to understand **why Kafka breaks and how to debug it**, this repo is for you.

---

## 📺 Video & Next Steps

This repository is used in a YouTube walkthrough.

Planned follow‑ups:

* Prometheus + Grafana dashboards
* ISR lag alerts
* SASL/SCRAM security
* Dedicated KRaft controllers

---

## ✅ License

MIT (use it, break it, learn from it)
