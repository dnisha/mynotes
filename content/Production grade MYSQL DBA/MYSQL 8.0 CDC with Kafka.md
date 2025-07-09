---
longform:
  format: single
  title: MYSQL 8.0 CDC with Kafka
title: MYSQL 8.0 CDC with Kafka
---
# Step-by-Step Guide to Implement CDC with Kafka and Debezium in Docker

This guide will walk you through implementing a Change Data Capture (CDC) pipeline using Kafka, Debezium, and MySQL in a Docker environment.

## Prerequisites
- Docker installed on your machine
- Docker Compose installed
- Basic understanding of Docker concepts
- Basic familiarity with Kafka and databases

## Step 1: Set Up Project Structure

1. Create a new directory for your project:
   ```bash
   mkdir kafka-cdc-pipeline
   cd kafka-cdc-pipeline
   ```

2. Create a `conf` directory for configuration files:
   ```bash
   mkdir conf
   ```

## Step 2: Create Docker Compose File

1. Create a `docker-compose.yml` file in your project directory with the following content:

```yaml
services:
  mysql:
    image: debezium/example-mysql:1.8
    container_name: mysql
    hostname: mysql
    ports:
      - '3306:3306'
    environment:
      MYSQL_ROOT_PASSWORD: debezium
      MYSQL_USER: mysqluser
      MYSQL_PASSWORD: mysqlpw

  zookeeper:
    image: confluentinc/cp-zookeeper:5.5.3
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - '2181:2181'

  kafka:
    image: confluentinc/cp-enterprise-kafka:5.5.3
    container_name: kafka
    depends_on:
      - zookeeper
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    links:
      - zookeeper
    ports:
      - '9092:9092'

  schema-registry:
    container_name: schema-registry
    image: confluentinc/cp-schema-registry:4.0.3
    environment:
      - SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=zookeeper:2181
      - SCHEMA_REGISTRY_HOST_NAME=schema-registry
      - SCHEMA_REGISTRY_LISTENERS=http://schema-registry:8081
    ports:
      - '8081:8081'

  kafka-connect:
    container_name: kafka-connect
    image: debezium/connect:1.8
    ports:
      - '8083:8083'
    links:
      - kafka
      - zookeeper
    environment:
      - BOOTSTRAP_SERVERS=kafka:9092
      - GROUP_ID=medium_debezium
      - CONFIG_STORAGE_TOPIC=my_connect_configs
      - CONFIG_STORAGE_REPLICATION_FACTOR=1
      - OFFSET_STORAGE_TOPIC=my_connect_offsets
      - OFFSET_STORAGE_REPLICATION_FACTOR=1
      - STATUS_STORAGE_TOPIC=my_connect_statuses
      - STATUS_STORAGE_REPLICATION_FACTOR=1
      - REST_ADVERTISED_HOST_NAME=medium_debezium

  connector-sender:
    container_name: connector-sender
    image: curlimages/curl
    volumes:
      - ./conf:/conf
    depends_on:
      - kafka-connect
    command:
      - sh
      - -c
      - |
        echo "Waiting for kafka connect to start..."
        until curl -s -o /dev/null -w "%{http_code}" http://kafka-connect:8083/connectors | grep -q "200"; do
          sleep 1
        done
        echo -e "Sending connector"
        curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://kafka-connect:8083/connectors/ -d @/conf/mysql_config.json
```

## Step 3: Create MySQL Connector Configuration

1. In the `conf` directory, create a file named `mysql_config.json` with the following content:

```json
{
    "name": "medium_debezium",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "mysql",
        "database.port": "3306",
        "database.user": "root",
        "database.password": "debezium",
        "database.server.id": "184054",
        "database.server.name": "dbserver1",
        "database.include.list": "inventory",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.history.kafka.topic": "schema-changes.inventory",
        "table.include.list": "inventory.customers, inventory.orders",
        "column.exclude.list": "inventory.customers.email",
        "snapshot.mode": "initial",
        "snapshot.locking.mode": "none"
    }
}
```

## Step 4: Start the Services

1. Run the following command to start all services:
   ```bash
   docker-compose up -d
   ```

2. Monitor the logs to ensure all services start correctly:
   ```bash
   docker-compose logs -f
   ```

## Step 5: Verify the Setup

1. Check if the connector was created successfully:
   ```bash
   curl -s http://localhost:8083/connectors | jq
   ```

   You should see `["medium_debezium"]` in the output.

2. Check the status of your connector:
   ```bash
   curl -s http://localhost:8083/connectors/medium_debezium/status | jq
   ```

## Step 6: Explore Kafka Topics

1. Enter the Kafka container:
   ```bash
   docker exec -it kafka bash
   ```

2. List all Kafka topics:
   ```bash
   kafka-topics --bootstrap-server=localhost:9092 --list
   ```

   You should see topics like `dbserver1.inventory.customers` and `dbserver1.inventory.orders`.

3. Consume messages from a topic:
   ```bash
   kafka-console-consumer --bootstrap-server localhost:9092 --topic dbserver1.inventory.customers --from-beginning
   ```

## Step 7: Test CDC Functionality

1. Connect to the MySQL database:
   ```bash
   docker exec -it mysql mysql -u root -pdebezium
   ```

2. Use the inventory database:
   ```sql
   USE inventory;
   ```

3. Make some changes to the data:
   ```sql
   INSERT INTO customers (first_name, last_name, email) VALUES ('John', 'Doe', 'john.doe@example.com');
   UPDATE customers SET first_name = 'Jane' WHERE first_name = 'John';
   DELETE FROM customers WHERE first_name = 'Jane';
   ```

4. Observe the changes in the Kafka consumer you started earlier.

## Step 8: Clean Up

When you're done testing, you can stop and remove all containers:
```bash
docker-compose down
```

## Additional Notes

1. **Schema Registry**: The schema registry is running on port 8081 if you need to access it.

2. **Kafka Connect UI**: You can access the Kafka Connect REST API at http://localhost:8083.

3. **Customizing the Setup**:
   - To monitor the pipeline, consider adding Kafka UI tools like AKHQ or Conduktor.
   - For production use, you'll want to add proper security configurations.

4. **Troubleshooting**:
   - If services don't start properly, check logs with `docker-compose logs [service_name]`.
   - Ensure ports 3306, 2181, 9092, 8081, and 8083 are available on your host machine.

This setup provides a complete CDC pipeline that captures changes from MySQL and streams them to Kafka topics in real-time. You can now extend this by adding sinks to other systems or building stream processing applications.