
## Run compose

```
docker-compose up
```

## Exec in container

```
docker-compose exec kafka /bin/bash
```

## Create a topic

```
kafka-topics \
    --create \
    --topic foo \
    --partitions 1 \
    --replication-factor 1 \
    --if-not-exists \
    --zookeeper zookeeper:2181
```

## List topics

```
$ kafka-topics \
    --list \
    --zookeeper zookeeper:2181
__confluent.support.metrics
foo
```

## Describe the topic

```
kafka-topics --describe --topic foo --zookeeper zookeeper:2181
Topic:foo	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: foo	Partition: 0	Leader: 1001	Replicas: 1001	Isr: 1001
```

## Produce messages

```
bash -c "seq 42 | kafka-console-producer --request-required-acks 1 \
--broker-list kafka:9092 --topic foo && echo 'Produced 42 messages.'"
```

## Consume messages

```
kafka-console-consumer --bootstrap-server kafka:9092 --topic foo --from-beginning --max-messages 42
```

## Creating a new go module

```
go mod init com.github.dnvriend/kafka-golang
```

## Building binaries

```
go build produce.go
go build consume.go
```

## Kafka client poll()
After subscribing to a set of topics, the consumer will automatically join the group when poll(long) is invoked. The poll API is designed to ensure consumer liveness. As long as you continue to call poll, the consumer will stay in the group and continue to receive messages from the partitions it was assigned. Underneath the covers, the consumer sends periodic heartbeats to the server. If the consumer crashes or is unable to send heartbeats for a duration of session.timeout.ms, then the consumer will be considered dead and its partitions will be reassigned.
It is also possible that the consumer could encounter a "livelock" situation where it is continuing to send heartbeats, but no progress is being made. To prevent the consumer from holding onto its partitions indefinitely in this case, we provide a liveness detection mechanism using the max.poll.interval.ms setting. Basically if you don't call poll at least as frequently as the configured max interval, then the client will proactively leave the group so that another consumer can take over its partitions. When this happens, you may see an offset commit failure (as indicated by a CommitFailedException thrown from a call to commitSync()). This is a safety mechanism which guarantees that only active members of the group are able to commit offsets. So to stay in the group, you must continue to call poll.

The consumer provides two configuration settings to control the behavior of the poll loop:

max.poll.interval.ms: By increasing the interval between expected polls, you can give the consumer more time to handle a batch of records returned from poll(long). The drawback is that increasing this value may delay a group rebalance since the consumer will only join the rebalance inside the call to poll. You can use this setting to bound the time to finish a rebalance, but you risk slower progress if the consumer cannot actually call poll often enough.
max.poll.records: Use this setting to limit the total records returned from a single call to poll. This can make it easier to predict the maximum that must be handled within each poll interval. By tuning this value, you may be able to reduce the poll interval, which will reduce the impact of group rebalancing.
For use cases where message processing time varies unpredictably, neither of these options may be sufficient. The recommended way to handle these cases is to move message processing to another thread, which allows the consumer to continue calling poll while the processor is still working. Some care must be taken to ensure that committed offsets do not get ahead of the actual position. Typically, you must disable automatic commits and manually commit processed offsets for records only after the thread has finished handling them (depending on the delivery semantics you need). Note also that you will need to pause the partition so that no new records are received from poll until after thread has finished handling those previously returned.

## Resources

- [kafka-single-node-client](https://docs.confluent.io/5.0.0/installation/docker/docs/installation/single-node-client.html)
- [Kafka Consumer Doc](https://kafka.apache.org/10/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html)
- [hadoop-in-docker](https://clubhouse.io/developer-how-to/how-to-set-up-a-hadoop-cluster-in-docker/)