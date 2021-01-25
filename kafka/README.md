


```bash


for x in {1..200}; do echo $x; sleep 2; done | kafka/kafka_2.13-2.7.0/bin/kafka-console-producer.sh --broker-list 20.198.137.129:9092 --topic test

kafka/kafka_2.13-2.7.0/bin/kafka-console-consumer.sh --topic test --from-beginning --bootstrap-server 20.198.137.129:9092

```