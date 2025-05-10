# Kafka Cluster in Docker (Confluent + Debezium + Redpanda Console)

This repository contains a full-featured Apache Kafka development setup using Docker Compose. It includes:

- Kafka (3 Brokers)
- Zookeeper
- Schema Registry
- Debezium Kafka Connect
- Kafka REST Proxy
- Redpanda Console (GUI for Kafka)

---

## 🧰 Components Overview

| Service              | Port(s)       | Description                                     |
|----------------------|---------------|-------------------------------------------------|
| Zookeeper            | `2181`        | Kafka's coordination service                    |
| Kafka Broker 1       | `9092`, `29092`| Main Kafka broker                               |
| Kafka Broker 2       | `9093`, `29093`| Secondary Kafka broker                          |
| Kafka Broker 3       | `9094`, `29094`| Tertiary Kafka broker                           |
| Schema Registry      | `8081`        | Avro schema management                          |
| Debezium Connect     | `8083`        | Kafka Connect with Debezium CDC plugin         |
| Kafka REST Proxy     | `8082`        | RESTful interface to Kafka                      |
| Redpanda Console     | `9080`        | UI to browse Kafka topics, schemas, etc.        |

---

# 🗂️ Folder Structure

```text
project-root/
├── docker-compose.yml
├── kafka/
│ ├── kafka-data-broker-1/
│ ├── kafka-data-broker-2/
│ ├── kafka-data-broker-3/
│ ├── connectors/
│ └── plugins/
```

## 🚀 Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/fajarafriad1/kafka-cluster-docker.git
cd kafka-docker-cluster
```

### 2. Prepare the kafka directory
Create and set permissions on the host kafka directory before running docker-compose up -d (required only if the container fails to create it).

```bash
mkdir -p kafka/{kafka-data-broker-1,kafka-data-broker-2,kafka-data-broker-3,connectors,plugins}
chmod -R 777 kafka
```
This ensures each broker’s data directory and the connectors/plugins folders are available and writable by Docker.

#### 3. Launch the Cluster
```bash
docker-compose up -d
```

Docker will:
1. Start Zookeeper
2. Bring up Kafka Brokers (waiting on Zookeeper healthchecks)
3. Initialize Schema Registry
4. Launch Debezium Connect
5. Fire up REST Proxy
6. Spin up Redpanda Console

### 4. Creating Test Topics

#### 1. Using CLI to Create Topics

Run this command from your host (adjust container name or broker endpoint as needed):

```bash
docker exec kafka-broker-1 \
  kafka-topics \
    --bootstrap-server kafka-broker-1:9092 \
    --create \
    --topic test-topic-via-cli \
    --partitions 3 \
    --replication-factor 3
```

- --bootstrap-server kafka-broker-1:9092 points the CLI at Broker 1’s PLAINTEXT listener.
- --partitions 3 and --replication-factor 3 ensure the topic is distributed evenly across all three brokers.
- You can list existing topics with:

```bash
docker exec kafka-broker-1 \
  kafka-topics \
    --bootstrap-server kafka-broker-1:9092 \
    --list
```


#### 2. Using Redpanda Console to Create Topics
Redpanda Console provides a user-friendly web UI for managing your Kafka topics without typing CLI commands. To create a topic via the Console:

1. Open the Console UI
Navigate to http://localhost:9080 in your browser and log in if prompted.

2. Go to the Topics Section
In the left-hand menu, click Topics to view the list of existing topics.

3. Click “Create Topic”
At the top right of the Topics screen, click the Create Topic button. A dialog will appear.

4. Fill in Topic Details
    - Name: Enter your topic name (e.g. test-topic).
    - Partitions: Specify the number of partitions (e.g. 3).
    - Replication Factor: Choose the replication factor (e.g. 3).

    You can also adjust advanced settings (e.g. cleanup policy, retention) under Configuration, or leave them at defaults.

5. Save and Verify
Click Save (or Create). The new topic will appear in the list. You can click on it to inspect its partitions, configurations, and message data in real time.


## 🛠️ Quick Commands

| Command                             | Description                                    |
|-------------------------------------|------------------------------------------------|
| `docker-compose up -d`              | Start all services in detached mode            |
| `docker-compose logs -f`            | Stream logs from all containers                |
| `docker-compose logs <service>`     | Stream logs from a specific service            |
| `docker-compose down`               | Stop and remove containers, networks, volumes  |
| `docker-compose restart`            | Restart all services                           |


## ⚙️ Configuration Details

### Zookeeper
- **Image:** `confluentinc/cp-zookeeper:7.6.1`  
- **Client Port:** `2181`  
- **Environment Variables:**
  ```yaml
  ZOOKEEPER_CLIENT_PORT: 2181
  ZOOKEEPER_TICK_TIME: 2000
  ```

### Kafka Brokers

#### Common settings for each broker
- **Image:** `confluentinc/cp-kafka:7.6.1`
- **Environment Variables:**
  ```yaml
  KAFKA_ZOOKEEPER_CONNECT: kafka-zookeeper:2181
  KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
  KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
  KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
  KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
  ```

### Schema Registry
- **Image:** `confluentinc/cp-schema-registry:7.6.1`
- **Bootstrap Servers:** `PLAINTEXT://kafka-broker-2:9093`
- **Environment Variables:**
  ```yaml
  SCHEMA_REGISTRY_HOST_NAME: kafka-schema-registry
  SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
  ```

### Debezium Kafka Connect

- **Image:** `debezium/connect:2.3`  
- **Bootstrap Servers:**  
  ```text
  kafka-broker-1:19092,kafka-broker-2:19093,kafka-broker-3:19094
  ```
- Internal Topics:
    - connect_configs
    - connect_statuses
    - connect_offsets

- Notes:

    - Place your connector JSON files in ./kafka/connectors/.
    - Add any custom connector JARs under ./kafka/plugins/.
    By default, Debezium will automatically create change‐event topics for each connector. To disable automatic topic creation, set:

    ```yaml
    CONNECT_TOPIC_CREATION_ENABLE: "false"
    ```
    - Adjust other Connect settings via environment variables or a custom connect-distributed.properties file as needed.

### Kafka REST Proxy

- **Image:** `confluentinc/cp-kafka-rest:7.6.1`
- **Bootstrap Servers:** `kafka-broker-1:19092,kafka-broker-2:19093,kafka-broker-3:19094`
- **Environment Variables:**
  ```yaml
  KAFKA_REST_HOST_NAME: kafka-rest-proxy
  KAFKA_REST_BOOTSTRAP_SERVERS: kafka-broker-1:19092,kafka-broker-2:19093,kafka-broker-3:19094
  KAFKA_REST_LISTENERS: http://0.0.0.0:8082
  ```
- Notes:
    - The KAFKA_REST_HOST_NAME sets the hostname used to generate absolute URLs in responses.
    - The KAFKA_REST_BOOTSTRAP_SERVERS specifies the Kafka brokers the REST Proxy connects to.
    - The KAFKA_REST_LISTENERS defines the listener address and port for the REST Proxy service.

### Redpanda Console

- **Image:** `docker.redpanda.com/redpandadata/console:v2.3.8`
- **Entrypoint:** Writes configuration to `/tmp/config.yml` and launches the console.
- **Environment Variable (`CONSOLE_CONFIG_FILE`):**
  ```yaml
  kafka:
    brokers: ["kafka-broker-1:19092", "kafka-broker-2:19093", "kafka-broker-3:19094"]
    schemaRegistry:
      enabled: true
      urls: ["http://kafka-schema-registry:8081"]
  connect:
    enabled: true
    clusters:
      - name: production
        url: http://kafka-debezium:8083
  ```

- The CONSOLE_CONFIG_FILE environment variable allows you to configure Redpanda Console by specifying the Kafka brokers, schema registry, and Kafka Connect clusters.
- Port: 9080 → Access the console at http://localhost:9080


# Kafka-Cluster-in-Docker
