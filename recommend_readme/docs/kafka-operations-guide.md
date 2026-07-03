# ZEST Kafka 운영 설치 가이드

| 항목 | 내용 |
| --- | --- |
| 문서 버전 | v2026.06.12 |
| 기준 코드 | 현재 로컬 구현 기준 |
| 적용 범위 | local Docker Kafka, Amazon Linux 2023 Docker Kafka, `market_tick_data`, `market_tick_data_dlq`, lag dashboard |

## 목적

`market_tick_data`는 한국투자 WebSocket/REST 시세 이벤트를 자동매매 파이프라인으로 전달하는 운영 topic이다. 현재 운영 목표 구조는 EC2 1대 내부 local Kafka(`localhost:9092`)이며, consumer 처리 실패 payload는 `market_tick_data_dlq`에 격리하고 운영 화면 `/admin/operations`에서 lag와 DLQ 처리 실패 지표를 확인한다.

## 로컬 설치

```bash
docker compose -f docker-compose.kafka.yml up -d
docker compose -f docker-compose.kafka.yml ps
docker exec zest-local-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

macOS에서 `docker compose` plugin이 깨져 있거나 없으면 Homebrew의 legacy binary로 같은 검증을 할 수 있다.

```bash
docker-compose -f docker-compose.kafka.yml config
docker-compose -f docker-compose.kafka.yml up -d
```

`Cannot connect to the Docker daemon`이 나오면 Docker Desktop, Colima, Rancher Desktop 중 하나의 Docker daemon을 먼저 시작해야 한다. 단순히 Docker CLI만 설치된 상태로는 Kafka 컨테이너를 띄울 수 없다.

`kafka-init` 컨테이너가 `market_tick_data`, `market_tick_data_dlq` topic을 자동 생성한다. 운영에서는 init 컨테이너가 이미 종료되었거나 재실행되지 않는 경우를 대비해 `scripts/ensure_kafka_topics.sh`를 표준 확인 명령으로 사용한다.

```bash
cd /opt/zest-source
sudo chown root:root scripts/ensure_kafka_topics.sh
sudo chmod 755 scripts/ensure_kafka_topics.sh
scripts/ensure_kafka_topics.sh
```

Spring Boot 실행 시:

```bash
export ZEST_KAFKA_ENABLED=true
export SPRING_KAFKA_BOOTSTRAP_SERVERS=localhost:9092
```

## 운영 topic 기준: EC2 local Kafka

```bash
cd /opt/zest-source
scripts/ensure_kafka_topics.sh
```

운영 권장값:

| 항목 | 권장 |
| --- | --- |
| broker | EC2 내부 local broker 1대 |
| replication-factor | 1 |
| producer ack | 1 |
| `market_tick_data` retention | 7일 |
| `market_tick_data_dlq` retention | 30일 |
| consumer group | `zest-ai-trader` |

트래픽, 장애 허용치, 사용자 수가 증가하면 MSK 또는 3 broker Kafka로 분리한다. 이때는 replication-factor 3, min.insync.replicas 2, producer ack all을 사용한다.

## Amazon Linux 2023 Docker 설치

운영 1차 구조는 EC2 1대 내부 Docker Kafka다. Java Core와 Kafka가 같은 EC2에서 동작하므로 broker는 `localhost:9092`로만 사용하고, Security Group inbound 9092는 열지 않는다.

```bash
sudo dnf update -y
sudo dnf install -y docker git
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
newgrp docker
docker version
```

Docker Compose plugin이 없으면 아래 중 하나를 적용한다.

```bash
docker compose version
sudo dnf install -y docker-compose-plugin
```

배포 디렉터리 예시:

```bash
sudo mkdir -p /opt/zest-stock-ai
sudo chown ec2-user:ec2-user /opt/zest-stock-ai
cd /opt/zest-stock-ai
git clone https://github.com/zestidea77/zestStockAI.git .
docker compose -f docker-compose.kafka.yml up -d
docker compose -f docker-compose.kafka.yml ps
docker exec zest-local-kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic market_tick_data
```

운영 환경 변수:

```bash
export ZEST_KAFKA_ENABLED=true
export SPRING_KAFKA_BOOTSTRAP_SERVERS=localhost:9092
export ZEST_KAFKA_MARKET_TICK_TOPIC=market_tick_data
export ZEST_KAFKA_DLQ_TOPIC=market_tick_data_dlq
```

systemd로 Kafka Docker를 자동 기동하려면 `/etc/systemd/system/zest-kafka.service`를 만든다.

```ini
[Unit]
Description=ZEST local Kafka docker compose
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
WorkingDirectory=/opt/zest-stock-ai
ExecStart=/usr/bin/docker compose -f docker-compose.kafka.yml up -d
ExecStartPost=/opt/zest-source/scripts/ensure_kafka_topics.sh
ExecStop=/usr/bin/docker compose -f docker-compose.kafka.yml down
RemainAfterExit=yes
TimeoutStartSec=180

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now zest-kafka
sudo systemctl status zest-kafka
```

운영 주의점:

- EBS gp3에 Kafka volume이 저장되므로 EBS snapshot 정책을 잡는다.
- 단일 broker이므로 EC2 장애 시 Kafka는 함께 중단된다. Java Core는 `ZEST_KAFKA_ENABLED=false`로 direct queue fallback 가능해야 한다.
- topic partition 수를 늘리면 consumer 병렬성은 좋아지지만 1 broker 환경에서는 장애 내성이 늘지 않는다.
- 10명 규모 초기 운영에서는 `market_tick_data` partition 3개면 충분하다. lag가 지속 증가하면 partition skew와 consumer 처리 시간을 먼저 확인한다.

## DLQ 처리

consumer가 JSON parse 또는 tick 변환에 실패하면 원문 payload를 `market_tick_data_dlq`로 발행하고 `kafka.market_tick.consume.failure` metric을 증가시킨다. DLQ payload는 원문을 보존하므로 운영자는 원본 event schema와 producer 버전을 먼저 확인한다.

## Lag dashboard

운영 화면:

```text
/admin/operations
```

API:

```text
GET /admin/api/operations/kafka-lag
```

`lag > 0`이 3분 이상 지속되면 consumer 처리 지연, broker 장애, partition skew를 확인한다. `status=ERROR:*`는 Spring Kafka bootstrap 설정 또는 ACL 권한 문제를 우선 의심한다.

## 배포 체크리스트

- `ZEST_KAFKA_ENABLED=true`
- `SPRING_KAFKA_BOOTSTRAP_SERVERS=localhost:9092`
- `ZEST_KAFKA_MARKET_TICK_TOPIC=market_tick_data`
- `ZEST_KAFKA_DLQ_TOPIC=market_tick_data_dlq`
- EC2 Security Group inbound 9092 미개방
- topic 생성과 retention 설정을 배포 runbook에 고정
- `/admin/api/operations/kafka-lag`와 `kafka.market_tick.consume.failure` 알림 임계치 등록
