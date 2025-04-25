--- 
layout: single
classes: wide
title: "[Kafka] Kafka Connect Monitoring"
header:
  overlay_image: /img/kafka-bg.jpg
excerpt: 'Kafka Connect 를 운영하며 모니터링을 위한 환경을 구축하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Kafka
tags:
    - Practice
    - Kafka
    - Kafka Connect
    - Monitoring
    - JMX
    - Prometheus
    - Grafana
    - Docker
    - Docker Compose
    - JMX Exporter
    - Kafka Connector
    - FileStreamSourceConnector
    - FileStreamSinkConnector
    - Kafka Connect REST API
toc: true
use_math: true
---  

## Kafka Connect Monitoring
`Kafka Connect` 를 모니터링 하는 것은 `Kafka` 기반 데이터 통합 파이프라인의 
안전성, 성능, 신뢰성을 보장하는데 필요한 작업이다. 
모니터링을 통해 `Kafka Connector` 들의 성능과 상태를 추적하고 관리할 수 있다. 
이를 위해 `Kafka Connect` 에서 제공하는 `JMX Metrics` 를 수집해 이를 식각화 하는 환경을 구축하는 방법에 대해 알아보고자 한다.  

`Kafka Connect` 의 모니터링을 구축하기 위해 `JMX` 로 메트릭을 `Export` 하고 이를 `Prometheus` 를 통해 수집한 후 `Grafana` 를 통해 
모니터링 대시보드를 구축한다. 
이를 도식화해 그리면 아래와 같다.  

구축의 상세 내용이 있는 예제는 [여기](https://github.com/windowforsun/kafka-connect-monitoring-exam)
에서 확인 할 수 있다.  

### Kafka Connect JMX Metrics
`Kafka Connect` 에서는 `JMX` 를 통해 다양한 메트릭을 제공한다. 
관련해 제공되는 메트릭의 종류는 [여기](https://docs.confluent.io/platform/current/connect/monitoring.html#use-jmx-to-monitor-kconnect)
에서 확인 할 수 있다.  

`Kafka Connect` 의 `JMX` 메트릭을 수집하기 위해서는 `JMX Exporter` 설치가 필요하다. 
`Docker` 환경에서 사용하기 위해 `Dockerfile` 에 아래와 같이 설치 스크립트를 추가한다.  

```dockerfile
FROM confluentinc/cp-kafka-connect-base:7.0.10

ENV CONNECT_PLUGIN_PATH="/usr/share/java,/usr/share/confluent-hub-components,/usr/share/filestream-connectors" \
    CUSTOM_SMT_PATH="/usr/share/java/custom-smt" \
    CUSTOM_CONNECTOR_MYSQL_PATH="/usr/share/java/custom-connector-mysql"
ENV TZ=Asia/Seoul
ENV KAFKA_OPTS="-Duser.timezone=Asia/Seoul"

ARG CONNECT_TRANSFORM_VERSION=1.4.4

# Download Using confluent-hub
RUN confluent-hub install --no-prompt confluentinc/connect-transforms:$CONNECT_TRANSFORM_VERSION

user root

RUN chmod -R 755 /usr/share/java

# Install tzdata package
RUN yum update -y && yum install -y tzdata

# Set the timezone to Asia/Seoul
RUN ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    echo "Asia/Seoul" > /etc/timezone

ENV TZ=Asia/Seoul
ENV KAFKA_OPTS="-Duser.timezone=Asia/Seoul"

# Set JVM timezone through JAVA_OPTS
ENV JAVA_OPTS="-Duser.timezone=Asia/Seoul"

# Download JMX Exporter
ADD https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.16.1/jmx_prometheus_javaagent-0.16.1.jar /opt/jmx_prometheus_javaagent.jar

# Add JMX Exporter configuration file
COPY config.yml /opt/config.yml

# Expose JMX port
EXPOSE 5556

CMD ["/bin/bash", "-c", "/usr/local/bin/init-connect.sh & /etc/confluent/docker/run"]
```  

그리고 `config.yaml` 파일을 아래와 같이 작성해 `JMX Exporter` 에서 메트릭을 공개할 룰을 정의한다. 
아래는 전체 메트릭을 공개하는 내용이다. 

```yaml
# config.yml
rules:
  - pattern: ".*"
```  

위 설정 내용에서 알 수 있듯이 `JMX` 를 수집할 때 사용할 포트는 `5556` 이다.  

### Prometheus
`Kafka Connect` 에서 `JMX Exporter` 를 사용해 메트릭을 수집할 수 있도록 공개했다면, 
메트릭을 수집하는 과정이 필요하다. 
이때 사용할 것이 바로 `Prometheus` 로 `Prometheus` 의 설정 파일인 `prometheus.yml` 파일에 
아래와 같이 `Kafka Connect` 에서 메트릭을 수집하도록 추가한다.  

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']


  - job_name: 'grafana'
    scrape_interval: 5s
    static_configs:
      - targets: ['grafana:3000']

  - job_name: 'kafka-connect'
    static_configs:
      - targets: [ 'exam-connect:5556' ]
```  



### 데모 환경 구성
이제 `docker-compose.yaml` 파일을 통해 테스트에 필요한 전체 구성을 작성하고 실행한다. 
`docker-compose.yaml` 파일 내용은 아래와 같이 `kafka`, `zookeeper`, `kafka-connect`, `prometheus`, `grafana` 로 구성돼있다.  

```yaml
version: '3'

services:
  zookeeper:
    container_name: myZookeeper
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
    networks:
      - exam-net

  kafka:
    container_name: myKafka
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    depends_on:
      - zookeeper
    networks:
      - exam-net

  exam-connect:
    container_name: exam-connect
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8083:8083"
      - "5556:5556"
    command: sh -c "
      echo 'waiting 10s' &&
      sleep 10 &&
      /etc/confluent/docker/run"
    environment:
      CONNECT_GROUP_ID: 'exam-connect'
      CONNECT_BOOTSTRAP_SERVERS: kafka:9092
      CONNECT_REST_PORT: 8083
      CONNECT_CONFIG_STORAGE_TOPIC: 'file-source-config'
      CONNECT_OFFSET_STORAGE_TOPIC: 'file-source-offset'
      CONNECT_STATUS_STORAGE_TOPIC: 'file-source-status'
      CONNECT_KEY_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_VALUE_CONVERTER: "org.apache.kafka.connect.storage.StringConverter"
      CONNECT_REST_ADVERTISED_HOST_NAME: exam-connect
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: '1'
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: '1'
      KAFKA_JMX_OPTS: "-javaagent:/opt/jmx_prometheus_javaagent.jar=5556:/opt/config.yml -Dcom.sun.management.jmxremote.authenticate=false"
      TZ: 'Asia/Seoul'
      JAVA_OPTS: "-Duser.timezone=Asia/Seoul"
    volumes:
      - ./data:/data
      - ./config.yml:/opt/config.yml
    networks:
      - exam-net

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - exam-net

  # id : admin, pw : admin
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - exam-net

networks:
  exam-net:
```  

`Kafka Connect` 쪽 `environment` 를 보면 `KAFKA_JMX_OPTS` 환경변수를 통해 
다운로드 받은 `JMX Exporter` 실행 파일과 사용할 포트 그리고 설정 파일인 `config.yml` 지정했다.  

```yaml
KAFKA_JMX_OPTS: "-javaagent:/opt/jmx_prometheus_javaagent.jar=5556:/opt/config.yml -Dcom.sun.management.jmxremote.authenticate=false"
```

모든 구성은 `docker-compose up --build` 명령으로 실행할 수 있다.  

```bash
$ docker-compose up --build                                                   0.0s
[+] Running 6/0
 ✔ Container docker-grafana-1                                                                                                                               Created0.0s 
 ✔ Container myZookeeper                                                                                                                                    Created0.0s 
 ✔ Container exam-connect                                                                                                                                   Created0.0s 
 ✔ Container docker-prometheus-1                                                                                                                            Created0.0s 
 ! zookeeper The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested 0.0s 
 ✔ Container myKafka                                                                                                                                        Created0.0s 
Attaching to docker-grafana-1, docker-prometheus-1, exam-connect, myKafka, myZookeeper

...

```  

그리고 데모 테스트를 위해 `FileStreamSourceConnector` 를 사용하는 `Kafka Connector` 를 아래 요청으로 등록한다.  

```bash
$ curl -X POST -H "Content-Type: application/json" \
--data '{
  "name": "exam-file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/file-source-input.txt",
    "topic" : "file-source-topic"
  }
}' \
http://localhost:8083/connectors | jq

{
  "name": "exam-file-source",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "tasks.max": "1",
    "file": "/data/file-source-input.txt",
    "topic": "file-source-topic",
    "name": "exam-file-source"
  },
  "tasks": [],
  "type": "source"
}
```  

그리고 `Kafka` 에서 `file-source-topic` 컨슘하면 파일의 내용이 토픽에 추가가 된 것으로 
`Kafka Connect` 및 `Kafka Connector` 실행이 정상적으로 된 것을 확인 할 수 있다.  

```bash
$ docker exec -it myKafka kafka-console-consumer.sh \
--bootstrap-server localhost:9092 \
--topic file-source-topic \
--property print.key=true \
--property print.headers=true \
--from-beginning 
NO_HEADERS      null    111
NO_HEADERS      null    222
```  

### JMX 메트릭 확인
`Kafka Connect` 의 `JMX Exporter` 를 통해 공개되는 메트릭은 `5556` 포트를 통해 조회할 수 있다. 
아래와 같이 요청을 하면 수집 가능한 메트릭 확인이 가능하다.  

```bash
$  curl localhost:5556/metrics | grep 'kafka_connect'
# TYPE kafka_connect_source_task_metrics_source_record_active_count_avg untyped
kafka_connect_source_task_metrics_source_record_active_count_avg{connector="exam-file-source",task="0",} NaN
# HELP kafka_connect_connect_metrics_failed_authentication_rate The number of connections with failed authentication per second (kafka.connect<type=connect-metrics, client-id=connect-1><>failed-authentication-rate)
# TYPE kafka_connect_connect_metrics_failed_authentication_rate untyped
kafka_connect_connect_metrics_failed_authentication_rate{client_id="connect-1",} 0.0
# HELP kafka_connect_connect_metrics_failed_reauthentication_rate The number of failed re-authentication of connections per second (kafka.connect<type=connect-metrics, client-id=connect-1><>failed-reauthentication-rate)
# TYPE kafka_connect_connect_metrics_failed_reauthentication_rate untyped
kafka_connect_connect_metrics_failed_reauthentication_rate{client_id="connect-1",} 0.0
# HELP kafka_connect_connect_metrics_request_rate The number of requests sent per second (kafka.connect<type=connect-metrics, client-id=connect-1><>request-rate)
# TYPE kafka_connect_connect_metrics_request_rate untyped
kafka_connect_connect_metrics_request_rate{client_id="connect-1",} 0.36672014668805863
# HELP kafka_connect_connect_coordinator_metrics_assigned_connectors The number of connector instances currently assigned to this consumer (kafka.connect<type=connect-coordinator-metrics, client-id=connect-1><>assigned-connectors)
# TYPE kafka_connect_connect_coordinator_metrics_assigned_connectors untyped
kafka_connect_connect_coordinator_metrics_assigned_connectors{client_id="connect-1",} 1.0
# HELP kafka_connect_connect_metrics_outgoing_byte_rate The number of outgoing bytes sent to all servers per second (kafka.connect<type=connect-metrics, client-id=connect-1><>outgoing-byte-rate)
# TYPE kafka_connect_connect_metrics_outgoing_byte_rate untyped
kafka_connect_connect_metrics_outgoing_byte_rate{client_id="connect-1",} 31.583772633509053
# HELP kafka_connect_connect_worker_metrics_task_startup_success_percentage The average percentage of this worker's tasks starts that succeeded. (kafka.connect<type=connect-worker-metrics><>task-startup-success-percentage)
# TYPE kafka_connect_connect_worker_metrics_task_startup_success_percentage untyped
kafka_connect_connect_worker_metrics_task_startup_success_percentage 0.0
# HELP kafka_connect_task_error_metrics_deadletterqueue_produce_requests The number of attempted writes to the dead letter queue. (kafka.connect<type=task-error-metrics, connector=exam-file-source, task=0><>deadletterqueue-produce-requests)
# TYPE kafka_connect_task_error_metrics_deadletterqueue_produce_requests untyped
kafka_connect_task_error_metrics_deadletterqueue_produce_requests{connector="exam-file-source",task="0",} 0.0
# HELP kafka_connect_connect_metrics_select_rate The number of times the I/O layer checked for new I/O to perform per second (kafka.connect<type=connect-metrics, client-id=connect-1><>select-rate)
# TYPE kafka_connect_connect_metrics_select_rate untyped
kafka_connect_connect_metrics_select_rate{client_id="connect-1",} 55.57974834406729
# HELP kafka_connect_connect_worker_rebalance_metrics_rebalancing Whether this worker is currently rebalancing. (kafka.connect<type=connect-worker-rebalance-metrics><>rebalancing)
# TYPE kafka_connect_connect_worker_rebalance_metrics_rebalancing untyped
kafka_connect_connect_worker_rebalance_metrics_rebalancing 0.0
# HELP kafka_connect_connect_metrics_connection_creation_rate The number of new connections established per second (kafka.connect<type=connect-metrics, client-id=connect-1><>connection-creation-rate)
# TYPE kafka_connect_connect_metrics_connection_creation_rate untyped
kafka_connect_connect_metrics_connection_creation_rate{client_id="connect-1",} 0.0
# HELP kafka_connect_connect_metrics_network_io_rate The number of network operations (reads or writes) on all connections per second (kafka.connect<type=connect-metrics, client-id=connect-1><>network-io-rate)
# TYPE kafka_connect_connect_metrics_network_io_rate untyped
kafka_connect_connect_metrics_network_io_rate{client_id="connect-1",} 0.7334402933761173
# HELP kafka_connect_connect_metrics_reauthentication_latency_avg The average latency observed due to re-authentication (kafka.connect<type=connect-metrics, client-id=connect-1><>reauthentication-latency-avg)
# TYPE kafka_connect_connect_metrics_reauthentication_latency_avg untyped
kafka_connect_connect_metrics_reauthentication_latency_avg{client_id="connect-1",} NaN
# HELP kafka_connect_kafka_metrics_count_count total number of registered metrics (kafka.connect<type=kafka-metrics-count><>count)
# TYPE kafka_connect_kafka_metrics_count_count untyped
kafka_connect_kafka_metrics_count_count 61.0
kafka_connect_kafka_metrics_count_count{client_id="connect-1",} 98.0
# HELP kafka_connect_source_task_metrics_poll_batch_avg_time_ms The average time in milliseconds taken by this task to poll for a batch of source records. (kafka.connect<type=source-task-metrics, connector=exam-file-source, task=0><>poll-batch-avg-time-ms)
# TYPE kafka_connect_source_task_metrics_poll_batch_avg_time_ms untyped
kafka_connect_source_task_metrics_poll_batch_avg_time_ms{connector="exam-file-source",task="0",} NaN
# HELP kafka_connect_connect_coordinator_metrics_last_heartbeat_seconds_ago The number of seconds since the last coordinator heartbeat was sent (kafka.connect<type=connect-coordinator-metrics, client-id=connect-1><>last-heartbeat-seconds-ago)
# TYPE kafka_connect_connect_coordinator_metrics_last_heartbeat_seconds_ago untyped
kafka_connect_connect_coordinator_metrics_last_heartbeat_seconds_ago{client_id="connect-1",} 1.0
# HELP kafka_connect_task_error_metrics_total_errors_logged The number of errors that were logged. (kafka.connect<type=task-error-metrics, connector=exam-file-source, task=0><>total-errors-logged)
# TYPE kafka_connect_task_error_metrics_total_errors_logged untyped
kafka_connect_task_error_metrics_total_errors_logged{connector="exam-file-source",task="0",} 0.0

...
```  

그리고 이번에는 `Prometheus` 가 `Kafka Connect` 의 `JMX` 메트릭을 정상 수집이 가능한지 확인을 위해 웹 브라우저를 통해 
`localhost:9090` 에 접속해 나오는 화면에서 확인이 필요한 메트릭 명을 입력해 조회할 수 있다. 
그리고 아래와 같이 `Prometheus` 에서 제공하는 `REST API` 를 사용해 확인 하는 것도 가능하다.  

```bash
$  curl "http://localhost:9090/api/v1/query?query=kafka_connect_connect_worker_metrics_task_count" | jq
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "kafka_connect_connect_worker_metrics_task_count",
          "instance": "exam-connect:5556",
          "job": "kafka-connect"
        },
        "value": [
          1723284580.359,
          "1"
        ]
      }
    ]
  }
}
```  

만약 정상적으로 조회되지 않으면 `Prometheus` 에서 수집 타겟 상태를 확인해 볼 수 있다.  

```bash
$  curl "http://localhost:9090/api/v1/targets?state=active&scrapePool=kafka-connect" | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   618  100   618    0     0  49527      0 --:--:-- --:--:-- --:--:-- 68666
{
  "status": "success",
  "data": {
    "activeTargets": [
      {
        "discoveredLabels": {
          "__address__": "exam-connect:5556",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "__scrape_interval__": "15s",
          "__scrape_timeout__": "10s",
          "job": "kafka-connect"
        },
        "labels": {
          "instance": "exam-connect:5556",
          "job": "kafka-connect"
        },
        "scrapePool": "kafka-connect",
        "scrapeUrl": "http://exam-connect:5556/metrics",
        "globalUrl": "http://exam-connect:5556/metrics",
        "lastError": "",
        "lastScrape": "2024-08-10T10:24:00.857769124Z",
        "lastScrapeDuration": 0.042762583,
        "health": "up",
        "scrapeInterval": "15s",
        "scrapeTimeout": "10s"
      }
    ],
    "droppedTargets": [],
    "droppedTargetCounts": null
  }
}
```


### Grafana 대시보드 구성
`Prometheus` 정상 수집까지 확인이 됐으면 이제 이를 `Grafana` 와 연동하고 메트릭을 사용해서 대시보드를 구성하면 된다. 
우선 웹 브라우저로 `localhost:3000` 에 접속하고 초기 `id` 와 `pw` 인 `admin` 을 입력해 로그인한다. 
그리고 `Prometheus` 를 `Grafana` 의 `Data Source` 로 등록하는 방법은 아래와 같다.  

1. 좌측 상단 `Configuration` > `Data Sources` 이동
2. `Add data source` 클릭
3. `Prometheus` 클릭
4. `URL` 에 `http://prometheus:9090` 입력
5. `Save & Test` 클릭해 연결 확인

그리고 대시 보드를 추가하는 방법은 아래와 같다.  

1. 좌측 상단 `Dashboards` 클릭
2. `Create dashboard` 클릭
3. `Import dashborad` 클릭
4. 아래 `JSON` 내용 붙여 넣기
5. `Load` 클릭
6. `Import` 클릭

`Grafana` 에 `Import` 할 `JSON` 내용은 [여기](https://github.com/windowforsun/kafka-connect-monitoring-exam/blob/master/docker/grafana-dashboard.json)
에서 확인 할 수 있다.
테스트 삼아 구성해본 내용이므로 리얼환경에서는 좀 더 모니터링하고자 하는 목적에 맞는 차트 구성과 메트릭 선택이 필요하다.  


---  
## Reference
[Use JMX to Monitor Connect](https://docs.confluent.io/platform/current/connect/monitoring.html#use-jmx-to-monitor-kconnect)  
[Prometheus HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)  


