---
layout: post
title: Kafka Quick Start Guide
description: >
  Apache Kafka Quick Start Guide 입니다.
sitemap: false
hide_last_modified: true
comments: true
---

# Kafka 설치 및 시작 가이드

실시간 스트리밍 데이터 처리 시 사용하는 Kafka의 설치 및 시작 가이드입니다. 본 글에서는 EC2 인스턴스를 생성한 후 그 위에 설치하는 방법을 설명합니다.

## EC2 인스턴스 생성 및 Kafka, Zookeeper 설치 및 실행

### 1. 인바운드 규칙 추가

- 9092 (Kafka broker)
- 2181 (Zookeeper)

### 2. EC2 접속

```bash
chmod 400 test-kafka-server-key.pem
ssh -i test-kafka-server-key.pem ec2-user@xxx.xxx.xxx.xxx
```
### 3. JDK 설치
```bash
sudo yum install -y java-1.8.0-openjdk-devel.x86_64
java -version
```
### 4. Kafka Broker 실행을 위한 Kafka Binary Package 다운로드 (Scala 2.12)
```bash
wget https://archive.apache.org/dist/kafka/2.5.0/kafka_2.12-2.5.0.tgz
tar xvf kafka_2.12-2.5.0.tgz
cd kafka_2.12-2.5.0
```
### 5. Kafka Heap Memory 설정
```bash
export KAFKA_HEAP_OPTS="-Xmx400m -Xms400m"
echo $KAFKA_HEAP_OPTS
```
### 6. 환경변수 설정
```bash
vi ~/.bashrc
# 제일 아래에 다음을 추가
export KAFKA_HEAP_OPTS="-Xmx400m -Xms400m"
```
### 7. Start.sh 파일 확인
```bash
cat bin/kafka-server-start.sh
```
### 8. Kafka Broker 실행 옵션 Properties 설정
```bash
vi config/server.properties
# 다음 부분의 주석을 해제하고 EC2 public IP를 입력
advertised.listeners=PLAINTEXT://your.host.name:9092
```
### 9. 주키퍼 실행
테스트를 위해 1대의 서버에서만 실행, -daemon 옵션으로 백그라운드 실행
jps로 JVM 프로세스 상태를 확인하여 주키퍼 정상 실행 여부 확인
```bash
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
jps -vm
```
### 10. Kafka Broker 실행
```bash
bin/kafka-server-start.sh -daemon config/server.properties
jps -m
```
### 11. 로그 
tail -f logs/server.log
Kafka 다뤄보기
1. 로컬에서 Kafka와 통신 확인 (Kafka Broker 정보 요청)
Kafka 바이너리 패키지 다운로드
bash
코드 복사
curl https://archive.apache.org/dist/kafka/2.5.0/kafka_2.12-2.5.0.tgz --output kafka.tgz
tar -xvf kafka.tgz
cd kafka_2.12-2.5.0
2. Kafka Broker 정보 요청
bash
코드 복사
bin/kafka-broker-api-versions.sh --bootstrap-server your.host.name:9092
3. 로컬에서 Kafka 네이밍 (hosts 파일 설정)
bash
코드 복사
vi /etc/hosts
# 다음을 추가
your.host.name my-kafka
4. 로컬에서 토픽 생성
bash
코드 복사
bin/kafka-topics.sh \
--create \
--bootstrap-server your.host.name:9092 \
--topic hello.kafka
5. 파티션 개수, 복제 개수, 토픽 데이터 유지 기간 옵션 지정하며 토픽 생성
파티션 default: 1
replication-factor 값이 1이면 복제 없음, 2면 1개의 복제본 사용
retention.ms: 토픽의 데이터를 유지하는 기간 (172800000ms = 2일)
bash
코드 복사
bin/kafka-topics.sh \
--create \
--bootstrap-server your.host.name:9092 \
--partitions 3 \
--replication-factor 1 \
--config retention.ms=172800000 \
--topic hello.kafka.2
6. 토픽 리스트 조회
bash
코드 복사
bin/kafka-topics.sh --bootstrap-server your.host.name:9092 --list
Kafka 개요 및 설정
데이터 흐름
mathematica
코드 복사
Source Application ➝ Kafka ➝ Target Application
Producer            Queue 역할 (Topics)       Consumer
Producer: json, tsv, avro 등의 데이터를 전송
Consumer: 데이터를 소비
Producer와 Consumer는 라이브러리로 구현
주요 개념 및 용어
Fault-Tolerant (고가용성)
랙이 내려가더라도 데이터 손실 없이 복구 가능
낮은 지연, 높은 처리량 (low latency, high throughput)
Topic
데이터가 들어가는 공간
일반적인 AMQP와 다름
테이블이나 폴더와 비슷한 개념
하나의 토픽은 여러 개의 파티션으로 구성될 수 있음
Broker, Replication, ISR
Replication (복제): Kafka의 가용성을 보장
Broker: Kafka가 설치된 서버 단위, 보통 3개 이상의 서버로 구성
ISR (In-Sync Replica): 복제된 데이터의 일관성을 유지
Acknowledgment (ack)
ack=0: 속도는 빠르지만 데이터 손실 여부를 알 수 없음
ack=1: 복제 여부를 알 수 없음
ack=all: 데이터 유실은 없지만 속도가 느림
Partitioner
메시지 키 또는 메시지 값에 따라 파티션을 결정
기본값은 UniformStickyPartitioner
Consumer Lag
Producer가 데이터를 빠르게 전송하고 Consumer가 느리게 처리할 때 발생
이 현상은 상황에 따라 좋을 수도, 나쁠 수도 있음
records-lag-max == 31
Burrow
오픈소스 Kafka consumer lag 모니터링 툴
메시지 브로커 플랫폼
메시지 브로커: 이벤트 브로커 역할 불가능
이벤트 브로커: 메시지 브로커 역할 가능
단일 진실 공급원: 장애 발생 시 해당 시점으로 복구 가능
스트림 데이터 처리: 많은 양의 데이터를 효과적으로 처리
MSA에서 중요한 역할:
메시지 브로커 예: RedisQ, RabbitMQ
이벤트 브로커 예: Kafka, AWS Kinesis
참고 자료
아파치 카프카 애플리케이션 프로그래밍 with 자바 (최원영)
```
