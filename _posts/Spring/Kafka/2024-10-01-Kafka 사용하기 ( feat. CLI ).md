---
title: Kafka 사용하기 (feat. CLI)
categories: [스프링, Kafka]
tags: [대용량 트래픽]
description: ""
---

<span class="md-title">1. kafka란?</span>   
<span class="md-content">kafka는 분산 이벤트 스트리밍 플랫폼으로써, linked in에서 마이크로 서비스간 복잡한 종속성을 해결하기 위해 등장한 메시지 큐이다. 이는 상대적으로 오버헤드가 큰 AMPQ 프로토콜을 사용하는 RabbitMQ에 비해 독자적인 TCP 바이너리 프로토콜을 사용함으로써 오버헤드를 줄이고 대용량의 이벤트를 실시간으로 처리할 수 있다는 특징이 있다. 대규모 트래픽을 처리하기 위한 필수 스택이기 때문에 이번 글에서는 가장 기본적인 수단인 CLI를 통해 kafka를 사용해본다.</span>   
<span class="md-title">2. 목표</span>   
<span class="md-content">다음은 kafka를 사용하여 구현하고자 하는 메시징 구조를 나타낸다.</span>   

<div class="img-box">
<img src="/assets/img/Kafka/kafka_message_queue_architecture.png" width="90%" height="90%" alt="kafkaResult">
</div>   

<span class="md-title"> 3. 구현 </span>   
<span class="md-sub"> 1) kafka cluster 설정하기 </span>   
<span class="md-content"> 먼저 zookeeper를 실행시킨다. </span>   

```powershell
bin/zookeeper-server-start.sh config/zookeeper.properties
```

<span class="md-content"> 좀 더 현실적인 환경을 구성하기 위해 3개의 분산 브로커를 사용하였다. </span>   

```powershell
bin/kafka-server-start.sh config/server0.properties
bin/kafka-server-start.sh config/server1.properties
bin/kafka-server-start.sh config/server0.properties
```

<span class="md-content"> 각각의 브로커에 id(0, 1, 2)와 9092, 9093, 9094 포트를 할당하고 같은 주키퍼를 가리키도록 설정하였다. </span>   

```properties
broker.id=0
listeners=PLAINTEXT://localhost:9092
log.dirs=/tmp/kafkainaction/kafka-logs-0
zookeeper.connect=localhost:2181
```
{:file="server0.properties"}

```properties
broker.id=1
listeners=PLAINTEXT://localhost:9093
log.dirs=/tmp/kafkainaction/kafka-logs-1
zookeeper.connect=localhost:2181
```
{:file="server1.properties"}

```properties
broker.id=2
listeners=PLAINTEXT://localhost:9094
log.dirs=/tmp/kafkainaction/kafka-logs-2
zookeeper.connect=localhost:2181
```
{:file="server2.properties"}

<span class="md-sub"> 2) kafka topic 생성하기 </span>   
<span class="md-content"> 한개의 topic에 대해 3개의 파티션을 생성 및 replication factor를 사용하여 브로커가 각각 다른 리더 파티션을 가지고, 남은 파티션에 대해 팔로워 파티션을 가지도록 구성하였다. </span>   

```powershell
bin/kafka-topics.sh --create --bootstrap-server localhost:9094 --topic kinaction_helloworld --partitions 3 --replication-factor 3
```

<div class="img-box">
<img src="/assets/img/Kafka/kafka_temp_topic.png" alt="kafkaResult">
</div>

<span class="md-content"> 위의 이미지를 보면 kafka_temp_topic이 성공적으로 구성된 것을 확인할 수 있다.   
여기까지의 아키텍처를 그려보면 다음과 같다. </span>   

<div class="img-box">
<img src="/assets/img/Kafka/kafka_topic_architecture.png" width="90%" height="90%" alt="kafkaResult">
</div>

<span class="md-sub"> 1) kafka producer 생성하기 </span>   
<span class="md-content"> kafka-console-producer.sh을 실행하여 kafka_temp_topic 토픽에 대해 3개의 메시지를 생성해준다. </span>   

<div class="img-box">
<img src="/assets/img/Kafka/kafka-console-producer.png" alt="kafkaResult">
</div>

<span class="md-content"> kafka_temp_topic 토픽을 가진 각각의 파티션별로 한 개의 메시지를 할당하였다.
( 메시지 할당 방식을 따로 지정해주지 않았으므로 라운드 로빈 방식이 지정된다. ) </span>   
<span class="md-sub"> 1) kafka consumer-group 생성하기 </span>   
<span class="md-content"> kafka-console-consumer.sh를 실행하여 kafka_temp_topic 토픽 에 대해 메시지를 소비할 경우의 그림은 다음과 같다. </span>   

<div class="img-box">
<img src="/assets/img/Kafka/kafka-console-consumer.png" alt="kafkaResult">
</div>

<span class="md-content"> 위의 이미지는 consumer를 한 개 생성하여 메시지를 소비한다. 이때 서로 다른 파티션에 있는 메시지를 소비하는 순서는 순서를 보장하지 않는다는 것을 확인할 수 있다.
3개의 파티션으로 나누어 토픽을 분리하였기 때문에 consumer group을 통해 메시지를 병렬적으로 소비할 수 있다.
이를 확인하기 위해 우선 같은 consumer group을 가진 3개의 consumer를 할당하였다. </span>   

<div class="img-box">
<img src="/assets/img/Kafka/kafka-console-consumer1.png" alt="kafkaResult">
</div>

<div class="img-box">
<img src="/assets/img/Kafka/kafka-console-consumer2.png" alt="kafkaResult">
</div>

<div class="img-box">
<img src="/assets/img/Kafka/kafka-console-consumer3.png" alt="kafkaResult">
</div>

<span class="md-content"> kafka_temp_group에 생성된 consumer들을 확인하면 다음과 같다. </span>   

<div class="img-box">
<img src="/assets/img/Kafka/kafka_temp_group.png" alt="kafkaResult">
</div>

<span class="md-content"> 각각의 파티션을 서로 다른 comsumer id를 가진 consumer가 담당하고 있는 것을 확인할 수 있다.
kafka producer를 통해 메시지를 생성하여 consumer group이 소비한 결과는 다음과 같다. </span>   

<div class="img-box">
<img src="/assets/img/Kafka/kafka_tmp_producer.png" alt="kafkaResult">
</div>

<div class="img-box">
<img src="/assets/img/Kafka/kafka_tmp_1.png" alt="kafkaResult">
</div>

<div class="img-box">
<img src="/assets/img/Kafka/kafka_tmp_2.png" alt="kafkaResult">
</div>

<div class="img-box">
<img src="/assets/img/Kafka/kafka_tmp_3.png" alt="kafkaResult">
</div>

<span class="md-content"> 서로 다른 파티션에 할당된 consumer가 각각의 파티션에 존재하는 메시지를 소비하는 것을 확인할 수 있다. </span>   

<span class="md-refs"> refs:   
"백엔드 개발자를 위한 한 번에 끝내는 대용량 데이터 & 트래픽 처리 초격차 패키지"  [\< fastCampus \>](https://fastcampus.co.kr/)   
"kafka in action" by Dylan Scott   
"apache kafka 공식 문서" [\< kafka docs \>](https://kafka.apache.org/documentation/)
</span>
