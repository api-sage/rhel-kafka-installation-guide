# Kafka Installation & Provisioning Guide (RHEL 9 / KRaft Mode)

**Project:** NPS Migration  
**Target:** Test/Production Environments

This document outlines the end-to-end process for provisioning an Apache Kafka node on a Red Hat Enterprise Linux (RHEL) 9 server.

---

## Step 1: System Updates & Prerequisites

Ensure the system is up to date and Java 17 (required for Kafka 3.x) is installed.

**1. Update package repositories:**

```bash
sudo dnf update -y
```

**2. Install OpenJDK 17:**

```bash
sudo dnf install java-17-openjdk-devel -y
```

**3. Verify Java version** (should reflect 17.x.x):

```bash
java -version
```

---

## Step 2: Download and Prepare Kafka Binaries

Kafka is extracted and moved directly to `/kafka` for a consistent install path across environments.

**1. Download the Kafka binary** (check for the latest version at [kafka.apache.org](https://kafka.apache.org)):

```bash
wget https://dlcdn.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
```

**2. Extract and move to `/kafka`:**

```bash
tar -xzf kafka_2.13-4.0.0.tgz
sudo mv kafka_2.13-4.0.0 /kafka
```

---

## Step 3: KRaft Configuration (No ZooKeeper)

KRaft mode uses a Cluster UUID to manage metadata.

**1. Update `advertised.listeners` in `server.properties`:**

Locate `/kafka/config/kraft/server.properties` and replace `localhost` with the server's actual IP address:

```properties
advertised.listeners=PLAINTEXT://XX.YY.ZZ.AA:9092
```

**2. Generate a unique Cluster ID:**

```bash
KAFKA_CLUSTER_ID=$(/kafka/bin/kafka-storage.sh random-uuid)
```

**3. Format the log directories:**

```bash
/kafka/bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c /kafka/config/kraft/server.properties
```

---

## Step 4: Systemd Service Creation

To ensure Kafka runs as a background service and starts on boot.

**1. Create the service file:**

```bash
sudo vi /etc/systemd/system/kafka.service
```

**2. Paste the following configuration:**

```ini
[Unit]
Description=Apache Kafka Server
Documentation=http://kafka.apache.org/documentation.html
After=network.target

[Service]
Type=simple
User=kafka
WorkingDirectory=/kafka
ExecStart=/bin/bash /kafka/bin/kafka-server-start.sh /kafka/config/kraft/server.properties
ExecStop=/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Security note:** It is strongly recommended to run Kafka as a dedicated non-root user (e.g. `kafka`). Ensure that user owns `/kafka` before starting the service:
> ```bash
> sudo useradd -r -s /sbin/nologin kafka
> sudo chown -R kafka:kafka /kafka
> ```

**3. Reload and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kafka
sudo systemctl status kafka
```

---

## Step 5: Firewall & Network Security

RHEL blocks port 9092 by default. Open it for producer/consumer connectivity.

```bash
sudo firewall-cmd --permanent --add-port=9092/tcp
sudo firewall-cmd --reload
```

---

## Step 6: Smoke Test (Verification)

**1. Confirm Kafka is listening on port 9092:**

```bash
sudo ss -tulpn | grep 9092
```

**2. Create a test topic** (replace `XX.YY.ZZ.AA` with the server's actual IP):

```bash
/kafka/bin/kafka-topics.sh --create --topic nps-test --bootstrap-server XX.YY.ZZ.AA:9092
```

**3. List topics to verify:**

```bash
/kafka/bin/kafka-topics.sh --list --bootstrap-server XX.YY.ZZ.AA:9092
```

---

## Pro-Tip: Scaling to a Production Cluster

This guide covers a **standalone (single-node)** setup. For a data-intensive production environment processing 10 M+ daily transactions, a **3-node (or more) cluster** is mandatory. The following changes must be applied across all nodes:

| Setting | Single-Node | Multi-Node (3 brokers) |
|---|---|---|
| `node.id` | `1` | Unique per node: `1`, `2`, `3` |
| `controller.quorum.voters` | `1@XX.YY.ZZ.AA:9093` | `1@IP1:9093,2@IP2:9093,3@IP3:9093` |
| `offsets.topic.replication.factor` | `1` | `3` |
| `transaction.state.log.replication.factor` | `1` | `3` |
| `advertised.listeners` | Single IP | Each node's own unique IP |

- **Unique Identity:** Each server must have a unique `node.id` (e.g., `1`, `2`, `3`).
- **Controller Quorum:** `controller.quorum.voters` must list all controllers on every node.
- **High Availability:** Replication factors must be raised to the number of brokers (≥ 3) to prevent data loss during a node failure.
- **Network Visibility:** Each server's `advertised.listeners` must point to its own unique IP so producers and consumers can locate the correct partition leader.
