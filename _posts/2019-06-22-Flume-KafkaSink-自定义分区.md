---
layout:     post
title:      "Flume之KafkaSink的自定义分区写入"
subtitle:   "相同key之写入到同一个kafka分区"
date:       2019-06-22
author:     "Woods"
header-img: "img/post-bg-js-version.jpg"
tags:
    - Flume
    - Kafka
    - 大数据
    
---

### 场景
Kafka接收MySQL BinLog日志，同一个表的同一个主键需要按照顺序来消费。

如果数据一条数据实际顺序是先create,再delete，消费是也必须按照这个顺序。

但是kafka只保证了同一分区内的数据是有序的。

所以需要将同一个主键的数据放到一个Kafka分区中。

可以按照`表名.主键值`作为Kafka的分区key。

下面使用flume模拟数据发送到Kafka。

### Flume Source的配置
```
kafka_key.sources = sources1
kafka_key.channels = channel1
kafka_key.sinks = sink1
##source 配置
kafka_key.sources.sources1.type = TAILDIR
kafka_key.sources.sources1.positionFile = /var/log/flume/taildir_position.json
kafka_key.sources.sources1.filegroups = f1
kafka_key.sources.sources1.filegroups.f1 = /home/data/kafkaKey/test.log
kafka_key.sources.sources1.batchSize = 100
kafka_key.sources.sources1.backoffSleepIncrement = 1000
kafka_key.sources.sources1.maxBackoffSleep = 5000
kafka_key.sources.sources1.channels = memorychannel
```

### Flume Source拦截器配置
```
# source 拦截器
kafka_key.sources.sources1.interceptors = i1
kafka_key.sources.sources1.interceptors.i1.type = regex_extractor
kafka_key.sources.sources1.interceptors.i1.regex = .*?\\|(.*?)\\|.*
kafka_key.sources.sources1.interceptors.i1.serializers = s1
kafka_key.sources.sources1.interceptors.i1.serializers.s1.name = key
```
该拦截器（Regex Extractor Interceptor）用于从原始日志中抽取key值，访问到events header中，header名字为key。

### Flume channel 配置
```
kafka_key.channels.channel1.type = memory
kafka_key.channels.channel1.capacity = 1000
kafka_key.channels.channel1.transactionCapacity = 100
```

### Flume Kafka Sink配置

```
# sink 1 配置
kafka_key.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
kafka_key.sinks.sink1.brokerList = bigdata-001:9092,bigdata-002:9092,bigdata-003:9092
kafka_key.sinks.sink1.topic = test_key
kafka_key.sinks.sink1.channel = channel1
kafka_key.sinks.sink1.batch-size = 100
kafka_key.sinks.sink1.requiredAcks = -1
# kafka_key.sinks.sink1.kafka.partitioner.class = com.lxw1234.flume17.SimplePartitioner
```

### 数据
```
time1|key1|ip1
time2|key2|ip2
time3|key3|ip3
time4|key1|ip4
time5|key3|ip5
time6|key3|ip6
time7|key1|ip7
time8|key2|ip8
time9|key2|ip9
time10|key2|ip10
time11|key1|ip11
time12|key3|ip12
```

### Java消费代码
```java
package com.woods.kafka;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Properties;

import kafka.consumer.Consumer;
import kafka.consumer.ConsumerConfig;
import kafka.consumer.ConsumerIterator;
import kafka.consumer.KafkaStream;
import kafka.javaapi.consumer.ConsumerConnector;
import kafka.message.MessageAndMetadata;
import org.apache.kafka.common.utils.Utils;

public class MyConsumer {
    public static void main(String[] args) {
        String topic = "test_key";
        ConsumerConnector consumer = Consumer.createJavaConsumerConnector(createConsumerConfig());
        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
        topicCountMap.put(topic, new Integer(1));
        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
        KafkaStream<byte[], byte[]> stream = consumerMap.get(topic).get(0);
        ConsumerIterator<byte[], byte[]> it = stream.iterator();
        while(it.hasNext()) {
            MessageAndMetadata<byte[], byte[]> mam = it.next();
            String msg = new String(mam.message());
            if(msg.length()<=0) continue;
            String key = msg.split("\\|")[1];
            //int testPartition = Math.abs(key.hashCode()) % 3;
            int testPartition = Utils.toPositive(Utils.murmur2(key.getBytes())) % 3;
            System.out.println("consume: Partition [" + mam.partition() + "] testPartition [" + testPartition + "] Message: [" + new String(mam.message()) + "] ..");
        }
    }
    private static ConsumerConfig createConsumerConfig() {
        Properties props = new Properties();
        props.put("group.id","group1");
        props.put("zookeeper.connect","bigdata-001:2181,bigdata-002:2181,bigdata-003:2181/kafka");
        props.put("zookeeper.session.timeout.ms", "4000");
        props.put("zookeeper.sync.time.ms", "200");
        props.put("auto.commit.interval.ms", "1000");
        //props.put("enable.auto.commit", "false");
        props.put("auto.offset.reset", "smallest");
        return new ConsumerConfig(props);
    }
}
```

### 结果
key1、key2 都在分区2中，key3在分区1中。
和testPartition是相同的。
```
consume: Partition [1] testPartition [1] Message: [time3|key3|ip3] ..
consume: Partition [1] testPartition [1] Message: [time5|key3|ip5] ..
consume: Partition [1] testPartition [1] Message: [time6|key3|ip6] ..
consume: Partition [2] testPartition [2] Message: [time1|key1|ip1] ..
consume: Partition [2] testPartition [2] Message: [time2|key2|ip2] ..
consume: Partition [1] testPartition [1] Message: [time12|key3|ip12] ..
consume: Partition [2] testPartition [2] Message: [time4|key1|ip4] ..
consume: Partition [2] testPartition [2] Message: [time7|key1|ip7] ..
```

### 总结
kafka sink会自动从header中获取key的值，并以此值最为key发送到broker中。

kafka的分区规则是：
1. 如果key为空，则随机发送到各个分区中。
2. key不为空，则根据`Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions`，类似于hash(keyBytes)对分区数取摸，来发送到对应的分区。

所以相同的key肯定会在同一个分区中。
    
源码如下：    
```java
public class DefaultPartitioner implements Partitioner {

    private final ConcurrentMap<String, AtomicInteger> topicCounterMap = new ConcurrentHashMap<>();

    public void configure(Map<String, ?> configs) {}

    /**
     * Compute the partition for the given record.
     *
     * @param topic The topic name
     * @param key The key to partition on (or null if no key)
     * @param keyBytes serialized key to partition on (or null if no key)
     * @param value The value to partition on or null
     * @param valueBytes serialized value to partition on or null
     * @param cluster The current cluster metadata
     */
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();
        if (keyBytes == null) {
            int nextValue = nextValue(topic);
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                // no partitions are available, give a non-available partition
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {
            // hash the keyBytes to choose a partition
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }
```

---
欢迎访问[个人博客](https://wsjwoods.github.io/)！



