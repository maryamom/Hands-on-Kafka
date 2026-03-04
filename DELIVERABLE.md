# Hands-On Kafka — Deliverable

## Commands Used and Observations

### Task 1 — Start Kafka

**Commands:**
```bash
docker compose up -d
docker ps
docker exec kafka kafka-broker-api-versions --bootstrap-server localhost:9092
```

**Observation:** After `docker compose up -d`, both Zookeeper and Kafka containers run. `docker ps` shows the two services. The broker API versions command confirms the Kafka broker is reachable on port 9092.

---

### Task 2 — Topics

**Commands:**
```bash
# Create topic 'transactions'
docker exec kafka kafka-topics --create --topic transactions --bootstrap-server localhost:9092

# Inspect topic configuration and see partitions
docker exec kafka kafka-topics --describe --topic transactions --bootstrap-server localhost:9092

# List all topics
docker exec kafka kafka-topics --list --bootstrap-server localhost:9092
```

**Observation:** The topic `transactions` is created with a default number of partitions (e.g. 1). The describe output shows partition count, replication factor, and leader broker. Topics are the logical channels where messages are stored.

---

### Task 3 — Producers

**Commands:**
```bash
# Start console producer (interactive: type messages, Ctrl+C to exit)
docker exec -it kafka kafka-console-producer --topic transactions --bootstrap-server localhost:9092
```
Then type several messages line by line (e.g. "msg1", "msg2", "msg3") and exit.

**Observation:** Each line sent is one message. Messages are written to the topic and persisted. The producer does not need to know about partitions; the broker (or key) decides assignment.

---

### Task 4 — Consumers

**Commands:**
```bash
# Read all messages from the beginning
docker exec -it kafka kafka-console-consumer --topic transactions --from-beginning --bootstrap-server localhost:9092
```
Exit with Ctrl+C, then run again without `--from-beginning`:

```bash
docker exec -it kafka kafka-console-consumer --topic transactions --bootstrap-server localhost:9092
```

**Observation:** With `--from-beginning`, all existing messages are read. Without it, the consumer only sees new messages produced after it started. Restarting the consumer (without `--from-beginning`) continues from the last committed offset, so old messages are not shown again.

---

### Task 5 — Offsets

**Commands:**
```bash
# List consumer groups
docker exec kafka kafka-consumer-groups --list --bootstrap-server localhost:9092

# Describe a consumer group to see offsets (group name appears when using console-consumer)
docker exec kafka kafka-consumer-groups --describe --group <group-name> --bootstrap-server localhost:9092
```

**Observation:** Kafka tracks progress per partition via offsets. Each partition has a current offset for the consumer group. With `--from-beginning`, consumption starts at offset 0. Without it, consumption starts at the latest offset. Restarting the consumer resumes from the last committed offset, so offsets are the “bookmark” for consumption.

---

### Task 6 — Consumer Groups

**Commands:**
- Terminal 1 — Consumer 1 (same group):
```bash
docker exec -it kafka kafka-console-consumer --topic transactions --bootstrap-server localhost:9092 --consumer-property group.id=my-group
```
- Terminal 2 — Consumer 2 (same group):
```bash
docker exec -it kafka kafka-console-consumer --topic transactions --bootstrap-server localhost:9092 --consumer-property group.id=my-group
```
- Terminal 3 — Producer: send several messages via console producer.

**Observation:** Messages are distributed across consumers in the same group: each partition is assigned to only one consumer in the group. So with one partition, only one consumer receives messages; with multiple partitions, messages are split between consumers. Work is shared within the group.

---

## Reflection Questions — Answers

**Why are topics considered logical streams?**  
A topic is a named, ordered log of messages. It behaves like a stream: producers append messages, consumers read in order. It is “logical” because it is a single name (e.g. `transactions`) that can be backed by one or more partitions, which together form the stream.

**Why does Kafka use partitions?**  
Partitions allow parallelism: different partitions can be written and read in parallel by different producers/consumers. They also allow scaling and load distribution across brokers and consumer instances.

**What is the role of offsets?**  
Offsets are the position of each consumer within a partition. They let Kafka know which messages have been consumed so that (1) consumers can resume from where they left off after a restart, and (2) the same message is not replayed unnecessarily (unless the consumer resets to an earlier offset).

**Why does each partition only allow one consumer per group?**  
So that each message in a partition is processed by exactly one consumer in the group. This avoids duplicate processing and keeps ordering per partition. Multiple consumers in the same group share partitions among themselves.

**What happens if there are more consumers than partitions?**  
Some consumers in the group stay idle. Each partition is assigned to at most one consumer in the group, so the number of active consumers is at most the number of partitions. Extra consumers get no partition assignment until a rebalance occurs (e.g. a consumer leaves).
