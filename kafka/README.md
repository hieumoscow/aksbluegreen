


```bash


mkdir kafka/kafka-bin && cd kafka/kafka-bin
wget https://archive.apache.org/dist/kafka/2.7.0/kafka_2.13-2.7.0.tgz
tar -xvzf kafka_2.13-2.7.0.tgz --strip 1

for x in {1..200}; do echo $x; sleep 2; done | bin/kafka-console-producer.sh --broker-list 20.198.137.129:9092 --topic test

bin/kafka-console-consumer.sh --topic test --from-beginning --bootstrap-server 20.198.137.129:9092

```