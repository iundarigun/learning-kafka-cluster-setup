# Learning Kafka cluster Setup

## Zookeeper

See more info the repo [learning-zookeeper](https://github.com/iundarigun/learning-zookeeper)

### Why kafka uses Zookeeper

- Kafka uses for brokers registrations, and keeping a heartbeat to keep the list updated
- Maintaining a list of topics alongside
    - Their configuration (partitions, replication factor, additional configurations...) 
    - The list of ISRs (in sync replicas) for partitions 
- Performing leader elections in case some brokers go down
- Storing the Kafka cluster id (randomly generated at 1st startup of cluster) 
- Storing ACLs (Access Control Lists) if security is enabled 
    - Topics
    - Consumer Groups
    - Users 
- Quotas configuration if enabled

> Time ago, used to store the consumer offsets (deprecated)

    It is recomended to use an odd number of servers (1, 3, 5, etc), but also is not common have more than 3 servers, because requires a lot of network to coordinate themself, so can produce lags.

To connect to zookeeper:
```shell
docker run -it --net=host zookeeper bash 
bin/zkCli.sh -server localhost:2181
```

## Kafka setup
Replication factor more than 3 is not recomended, even for big cluster (100 brokers)

Running and connecting to kafka cluster:
```shell
cd docker-compose
docker-compose up -d
docker exec -it docker-compose-kafka1-1 bash
kafka-topics.sh --bootstrap-server kafka1:9192 --create --topic first_topic
## (optional)
kafka-topics.sh --bootstrap-server kafka1:9192 --create --topic second_topic --replication-factor 1 --partitions 4
```

Publishing messages:
```shell
docker exec -it docker-compose-kafka1-1 bash
kafka-console-producer.sh --bootstrap-server kafka1:9192 --topic first_topic
> hi
> hello
```

Consuming messages:
```shell
docker exec -it docker-compose-kafka1-1 bash
kafka-console-consumer.sh --bootstrap-server kafka1:9192 --topic first_topic --group sample-consumer --from-beginning
```

List topics
```shell
docker exec -it docker-compose-kafka1-1 bash
kafka-topics.sh --bootstrap-server kafka1:9192 --list
```

> Since we have the kafka-ui in the docker-compose, we can just go to the http://localhost:9999 and be happy

Leader election after broker problems (for example, stop broker1 and broker2, publish data and start again):
```shell
docker exec -it docker-compose-kafka1-1 bash
kafka-leader-election.sh --all-topic-partitions --bootstrap-server kafka1:9192 --election-type PREFERRED
```

## kafka performance

### Disk I/O
- Reads are done sequentially (as in not randomly), therefore make sure you should a disk type that corresponds well to the requirements 
- Format your drives as XFS (easiest, no tuning required) 
- If read / write throughput is your bottleneck
    - it is possible to mount multiple disks in parallel for Kafka 
- Kafka performance is constant with regards to the amount of data stored in Kafka. 
    - Make sure you expire data fast enough (default is one week) 
    - Make sure you monitor disk performance

### Network
- Latency is key in Kafka 
    - Ensure your Kafka instances and your Zookeeper instances are geographically close  
    - Having two brokers live on the same rack is good for performance, but a big risk if the rack goes down. 
- Network bandwidth is key in Kafka 
    - Network will be your bottleneck. 
    - Make sure you have enough bandwidth to handle many connections, and TCP requests. 
    -  Make sure your network is high performance 
- Monitor network usage to understand when it becomes a bottleneck

### RAM
- Kafka has amazing performance thanks to the page cache which utilizes your RAM 
- Understanding RAM in Kafka means understanding two parts: 
    - The Java HEAP for the Kafka process 
    - The rest of the RAM used by the OS page cache
- Kafka should keep a low heap usage over time, and heap should increase only if you have more partitions in your broker.
- The remaining RAM will be used automatically for the Linux **OS Page Cache**. 
- This is used to buffer data to the disk and this is what gives Kafka an amazing performance and you don't have to specify anything! 
- Any un-used memory will automatically be leveraged by the Linux Operating System and assign memory to the page cache
- Note: Make sure swapping is disabled for Kafka entirely `vm.swappiness=0` or `vm.swappiness=1` (default is 60 on Linux)

> Overall, your Kafka production machines should have at least 8GB of RAM to them (the more the better - it's common to have 16GB or 32GB per broker)

### CPU
- CPU is usually not a performance bottle neck in Kafka because Kafka does not parse any messages, but can become one in some situations 
- If you have SSL enabled, Kafka has to encrypt and decrypt every payload, which adds load on the CPU 
- Compression can be CPU bound if you force Kafka to do it. Instead, if you send compressed data, make sure your producer and consumers are the ones doing the compression work (that's the default setting anyway) 
- Make sure you monitor Garbage Collection over time to ensure the pauses are not too long

### OS
- Use Linux or Solaris, running production Kafka clusters on Windows is not recommended. 
- Increase the file descriptor limits (at least 100,000 as a starting point)
    - https://unix.stackexchange.com/a/8949/170887 
-  Make sure only Kafka is running on your Operating System. Anything else will just slow the machine down.

### Others
- Make sure you have enough file handles opened on your servers, as Kafka opens 3 file descriptor for each topic-partition-segment that lives on the Broker. 
- You may want to tune the GC implementation
- Set Kafka quotas in order to prevent unexpected spikes in usage

--- 

> Folder `code` contains the helper scripts from the Udemy course

> Folder `docker-compose` has a docker based kafka cluster

---
## References
- Kafka udemy course: https://www.udemy.com/course/kafka-cluster-setup/
- Broker properties: https://kafka.apache.org/documentation/#brokerconfigs
