# KafkaClusterSetup

# Forcing Kafka to Run with ZooKeeper (Disabling KRaft Completely)

Since you're still getting KRaft-related issues with Apache Kafka 3.9.0, here's how to explicitly force ZooKeeper mode by modifying the container's entrypoint:

## Solution: Override Entrypoint to Disable KRaft

The official Apache Kafka image automatically detects configuration, but we can force ZooKeeper mode by:

1. **Creating a custom startup script**
2. **Mounting a proper server.properties file**
3. **Explicitly disabling KRaft**

### Modified Docker Compose Example (for each node)

```yaml
version: '3.8'

services:
  zookeeper:
    image: zookeeper:latest
    hostname: zookeeper
    ports:
      - "2181:2181"
      - "2888:2888"
      - "3888:3888"
    environment:
      ZOO_MY_ID: 1  # Change per node (1, 2, 3)
      ZOO_SERVERS: "server.1=10.7.200.238:2888:3888;server.2=10.7.200.237:2888:3888;server.3=10.7.200.236:2888:3888"
    volumes:
      - zk_data:/data
      - zk_log:/datalog

  kafka:
    image: apache/kafka:3.9.0
    hostname: kafka1  # Change per node (kafka1, kafka2, kafka3)
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1  # Change per node (1, 2, 3)
    volumes:
      - ./server.properties:/kafka/config/server.properties
      - ./start-kafka.sh:/start-kafka.sh
      - kafka_data:/tmp/kafka-logs
    entrypoint: ["/start-kafka.sh"]
    depends_on:
      - zookeeper

volumes:
  zk_data:
  zk_log:
  kafka_data:
```

## Required External Files

1. **server.properties** (place in same directory as docker-compose.yml):
```properties
# Core configuration
broker.id=1  # Change per node
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://10.7.200.238:9092  # Change IP per node

# ZooKeeper configuration
zookeeper.connect=10.7.200.238:2181,10.7.200.237:2181,10.7.200.236:2181

# Disable KRaft explicitly
process.roles=null
inter.broker.listener.name=PLAINTEXT

# Replication settings
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
default.replication.factor=3
min.insync.replicas=2
```

2. **start-kafka.sh** (make executable with `chmod +x start-kafka.sh`):
```bash
#!/bin/bash

# Explicitly disable KRaft
export KAFKA_KRAFT_ENABLED="false"

# Wait for ZooKeeper to be ready
while ! nc -z zookeeper 2181; do
  echo "Waiting for ZooKeeper..."
  sleep 1
done

# Start Kafka in ZooKeeper mode
exec /kafka/bin/kafka-server-start.sh /kafka/config/server.properties
```

## Key Changes That Force ZooKeeper Mode:

1. **Explicit KRaft Disable**:
   - `process.roles=null` in server.properties
   - `KAFKA_KRAFT_ENABLED="false"` in startup script

2. **Direct Properties File**:
   - Bypasses environment variable conversion that might enable KRaft

3. **Custom Entrypoint**:
   - Ensures proper startup sequence
   - Adds explicit ZooKeeper dependency check

## Deployment Steps:

1. Create the two files (`server.properties` and `start-kafka.sh`) on each host
2. Adjust the broker.id, advertised.listeners, and hostnames for each node
3. Make the script executable: `chmod +x start-kafka.sh`
4. Deploy with `docker-compose up -d`

## Verification:

Check logs to confirm ZooKeeper mode:
```bash
docker logs kafka1 | grep -i "Kafka Server started"
```

You should see messages about:
- Connecting to ZooKeeper
- Registering broker with ZooKeeper
- No mention of "KRaft" or "Raft" in startup logs

## Alternative Approach (If Still Having Issues)

If you continue to have problems, you can try using an older Kafka version that doesn't default to KRaft:

```yaml
kafka:
  image: apache/kafka:3.4.0  # Version before KRaft became default
  # ... rest of configuration ...
```

This more aggressive approach should completely eliminate any KRaft-related behavior and force pure ZooKeeper operation.
