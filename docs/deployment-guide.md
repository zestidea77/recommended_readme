# ZEST AI Trader 배포 및 실행 가이드

| 항목 | 내용 |
| --- | --- |
| 문서 버전 | v2026.06.12 |
| 기준 코드 | 현재 로컬 구현 기준 |
| 배포 대상 | 로컬 단독 실행 패키지, 개발 서버, 운영 EC2/RDS/Redis/Kafka 구조 |

<!--
  이 문서는 로컬, 개발, 운영 환경에서 ZEST AI Trader를 실행하고 배포하는 절차를 정리한다.
  자동매매 시스템은 설정 실수가 바로 주문 사고로 이어질 수 있으므로 안전 기본값과 확인 절차를 함께 적는다.
-->

## 1. 공통 구조

ZEST AI Trader는 두 프로세스로 구성된다.

```text
Java Core
- Spring Boot + Thymeleaf
- 한국투자 API adapter
- OMS/RMS
- 전략별 가상 계좌
- gRPC client

Python Brain
- gRPC server
- SCALPING / RL_PORTFOLIO 모델 라우팅
- 추론 결과 반환
```

기본 포트는 아래와 같다.

| 구분 | 기본 포트 | 설명 |
| :--- | :--- | :--- |
| Java Core | `8090` | 웹 화면, REST API, OMS/RMS |
| Python Brain gRPC | `50051` | Java Core가 호출하는 AI 추론 서버 |
| Python Health API | `8001` | 선택 실행하는 Python 상태 확인 HTTP 서버 |

## 2. 필수 준비

### 2.0 설정 파일 구조

환경별 설정은 Spring profile 파일로 분리한다.

| 파일 | 용도 |
| :--- | :--- |
| `application.yml` | 공통 기본값과 기본 active profile |
| `application-local.yml` | 로컬 H2 DB, dry-run, 로컬 Python Brain |
| `application-dev.yml` | 개발 서버 DB, 모의투자, 별도 Python Brain |
| `application-prod.yml` | 운영 서버 DB, 실전투자, 별도 Python Brain |

실행 profile은 아래처럼 선택한다.

```bash
./gradlew bootRun --args='--spring.profiles.active=local'
java -jar zest-ai-trader.jar --spring.profiles.active=dev
java -jar zest-ai-trader.jar --spring.profiles.active=prod
```

### 2.1 Java

JDK 21 권장, Java 컴파일 target은 17이다.

```bash
java -version
./gradlew --version
```

Eclipse에서는 `Gradle wrapper`와 JDK 21을 사용한다.

### 2.2 Python

Python Brain은 Python 3.13.x를 기준으로 실행한다. 2026년 5월 24일 확인 기준 최신 3.13 패치 버전은 3.13.13이다.

```bash
python3.13 --version
```

### 2.3 한국투자 API

한국투자 KIS Developers에서 아래 값을 발급받는다.

| 환경 변수 | 설명 |
| :--- | :--- |
| `KIS_APP_KEY` | 한국투자 Open API app key |
| `KIS_APP_SECRET` | 한국투자 Open API app secret |
| `KIS_ACCOUNT_NO` | 계좌번호 앞 8자리 |
| `KIS_ACCOUNT_PRODUCT_CODE` | 계좌 상품 코드. 보통 `01` |
| `KIS_HTS_ID` | WebSocket 실시간 연동 시 사용할 HTS ID |

처음에는 반드시 모의투자(`paper`)로 시작한다.

## 3. 로컬 실행

로컬은 H2 메모리 DB와 dry-run 기본값으로 실행된다.

### 3.1 Python Brain 준비

```bash
cd /Users/zest/git/stoackAI
python/scripts/setup_local.sh
```

Python Brain을 따로 실행하려면:

```bash
python/scripts/run_brain.sh
```

선택으로 Health API를 띄우려면:

```bash
PYTHONPATH=. uvicorn brain.health_api:app --host 0.0.0.0 --port 8001
```

### 3.2 Java Core 실행

Python Brain을 따로 띄운 경우:

```bash
cd /Users/zest/git/stoackAI
./gradlew bootRun
```

Spring Boot가 Python Brain도 같이 띄우게 하려면:

```bash
cd /Users/zest/git/stoackAI
ZEST_BRAIN_PYTHON=python/.venv/bin/python \
./gradlew bootRun --args='--zest.brain.auto-start=true'
```

접속:

```text
http://localhost:8090/trader
```

### 3.3 로컬 동작 확인

```bash
curl http://localhost:8090/api/status
curl http://localhost:8090/api/accounts
curl -X POST http://localhost:8090/api/ticks \
  -H 'Content-Type: application/json' \
  -d '{"ticker":"005930","price":82000,"volume":3500000,"strategy":"SCALPING","dataType":"TICK"}'
```

기본 설정에서는 `zest.trading.auto-trading-enabled=false`라 주문은 실행되지 않는다.

## 4. 개발 서버 배포

개발 서버는 모의투자, 외부 DB, 별도 Python Brain 프로세스를 권장한다.

### 4.1 서버 디렉터리 예시

```text
/opt/zest-ai-trader
├── app
│   ├── zest-ai-trader.jar
│   └── application-dev.yml
└── python
    ├── .venv
    └── brain source
```

### 4.2 Java 빌드

로컬 또는 CI에서:

```bash
cd /Users/zest/git/stoackAI
./gradlew clean bootJar
```

산출물:

```text
build/libs/zest-ai-trader-0.0.1-SNAPSHOT.jar
```

### 4.3 Python 배포

서버에서:

```bash
cd /opt/zest-ai-trader/python
python3.13 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
./scripts/generate_proto.sh
```

Python Brain 실행:

```bash
cd /opt/zest-ai-trader/python
PYTHONPATH=. BRAIN_PORT=50051 .venv/bin/python -m brain.server
```

### 4.4 Java 개발 서버 실행

환경 변수:

```bash
export KIS_APP_KEY="개발/모의 app key"
export KIS_APP_SECRET="개발/모의 app secret"
export KIS_ACCOUNT_NO="모의계좌번호"
export KIS_ACCOUNT_PRODUCT_CODE="01"
```

실행:

```bash
java -jar /opt/zest-ai-trader/app/zest-ai-trader.jar \
  --spring.profiles.active=dev \
  --server.port=8090 \
  --zest.kis.enabled=true \
  --zest.kis.environment=paper \
  --zest.trading.auto-trading-enabled=false \
  --zest.brain.host=127.0.0.1 \
  --zest.brain.port=50051
```

개발 서버에서도 자동주문은 기본적으로 꺼둔다.

## 5. 운영 서버 배포

운영은 Java Core와 Python Brain을 반드시 분리 실행한다.

운영 권장 원칙:

- Python Brain `auto-start` 사용 금지
- 한국투자 모의투자에서 충분히 검증 후 실전 전환
- `zest.trading.auto-trading-enabled=true`는 마지막 단계에서만 적용
- 주문/체결/잔고 로그를 외부 저장소에 남김
- 서버 시간은 KST 기준으로 NTP 동기화
- 배포 전 당일 휴장일, 장 상태, 계좌 상태 확인

### 5.1 운영 환경 변수

```bash
export KIS_APP_KEY="운영 app key"
export KIS_APP_SECRET="운영 app secret"
export KIS_ACCOUNT_NO="실계좌번호"
export KIS_ACCOUNT_PRODUCT_CODE="01"
export KIS_HTS_ID="HTS ID"
```

### 5.2 운영 Java 실행

```bash
JAVA_OPTS="-Xms2g -Xmx6g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/zest-ai-trader/logs/java -XX:+ExitOnOutOfMemoryError -Dfile.encoding=UTF-8 -Duser.timezone=Asia/Seoul"
java ${JAVA_OPTS} \
  -jar /opt/zest-ai-trader/app/zest-ai-trader.jar \
  --spring.profiles.active=prod \
  --spring.sql.init.mode=never \
  --server.port=8090 \
  --zest.kis.enabled=true \
  --zest.kis.environment=prod \
  --zest.trading.auto-trading-enabled=false \
  --zest.brain.host=127.0.0.1 \
  --zest.brain.port=50051
```

실전 자동주문 활성화는 별도 승인 후 아래처럼 바꾼다.

```bash
--zest.trading.auto-trading-enabled=true
```

네이버 뉴스 수집을 켤 때는 운영 환경변수에 아래 값을 추가한다.

```bash
ZEST_NEWS_ENABLED=true
ZEST_NAVER_NEWS_ENABLED=true
NAVER_CLIENT_ID=<naver-client-id>
NAVER_CLIENT_SECRET=<naver-client-secret>
ZEST_NAVER_NEWS_DISPLAY=10
ZEST_NAVER_NEWS_SORT=date
ZEST_NAVER_NEWS_CRAWL_BODY=false
```

### 5.3 운영 Python 실행

```bash
cd /opt/zest-ai-trader/python
PYTHONPATH=. \
BRAIN_PORT=50051 \
BRAIN_WORKERS=8 \
.venv/bin/python -m brain.server
```

## 6. systemd 예시

### 6.1 Python Brain

`/etc/systemd/system/zest-brain.service`

```ini
[Unit]
Description=ZEST AI Trader Python Brain
After=network.target

[Service]
WorkingDirectory=/opt/zest-ai-trader/python
Environment=PYTHONPATH=/opt/zest-ai-trader/python
Environment=BRAIN_PORT=50051
Environment=BRAIN_WORKERS=8
ExecStart=/opt/zest-ai-trader/python/.venv/bin/python -m brain.server
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 6.2 Java Core

`/etc/systemd/system/zest-core.service`

```ini
[Unit]
Description=ZEST AI Trader Java Core
After=network.target zest-brain.service
Requires=zest-brain.service

[Service]
WorkingDirectory=/opt/zest-ai-trader/app
Environment=KIS_APP_KEY=replace-me
Environment=KIS_APP_SECRET=replace-me
Environment=KIS_ACCOUNT_NO=replace-me
Environment=KIS_ACCOUNT_PRODUCT_CODE=01
Environment="JAVA_OPTS=-Xms2g -Xmx6g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/opt/zest-ai-trader/logs/java -XX:+ExitOnOutOfMemoryError -Dfile.encoding=UTF-8 -Duser.timezone=Asia/Seoul"
ExecStart=/bin/sh -lc 'exec /usr/bin/java ${JAVA_OPTS} -jar /opt/zest-ai-trader/app/zest-ai-trader.jar --spring.profiles.active=prod --spring.sql.init.mode=never --zest.kis.enabled=true --zest.kis.environment=prod --zest.trading.auto-trading-enabled=false --zest.brain.host=127.0.0.1 --zest.brain.port=50051'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

적용:

```bash
sudo systemctl daemon-reload
sudo systemctl enable zest-brain zest-core
sudo systemctl start zest-brain
sudo systemctl start zest-core
sudo systemctl status zest-brain
sudo systemctl status zest-core
```

## 7. Docker Compose 예시

```yaml
services:
  brain:
    image: python:3.13-slim
    working_dir: /app/python
    volumes:
      - .:/app
    command: sh -c "pip install -r requirements.txt && ./scripts/generate_proto.sh && PYTHONPATH=. python -m brain.server"
    environment:
      BRAIN_PORT: 50051
      BRAIN_WORKERS: 8
    ports:
      - "50051:50051"

  core:
    image: eclipse-temurin:21-jre
    working_dir: /app
    volumes:
      - ./build/libs:/app
    command: java -jar zest-ai-trader-0.0.1-SNAPSHOT.jar --zest.brain.host=brain --zest.brain.port=50051 --zest.kis.enabled=true --zest.kis.environment=paper --zest.trading.auto-trading-enabled=false
    environment:
      KIS_APP_KEY: ${KIS_APP_KEY}
      KIS_APP_SECRET: ${KIS_APP_SECRET}
      KIS_ACCOUNT_NO: ${KIS_ACCOUNT_NO}
      KIS_ACCOUNT_PRODUCT_CODE: ${KIS_ACCOUNT_PRODUCT_CODE:-01}
    ports:
      - "8090:8090"
    depends_on:
      - brain
```

## 8. 배포 전 체크리스트

### 로컬

- `./gradlew test` 성공
- Python `pip install -r requirements.txt` 성공
- `./scripts/generate_proto.sh` 성공
- `/api/status` 응답 확인
- `/api/ticks` 샘플 입력 확인

### 개발

- 한국투자 모의투자 키 사용
- `zest.kis.environment=paper`
- `zest.trading.auto-trading-enabled=false`
- Python Brain과 Java Core를 별도 프로세스로 실행
- 로그에서 Brain gRPC 연결 실패가 없는지 확인

### 운영

- 실계좌 번호 재확인
- API key가 운영용인지 확인
- `zest.kis.environment=prod` 확인
- 자동주문 활성화 전 dry-run 주문 이벤트 검증
- 주문 한도, 전략별 가상 계좌, 손실 제한값 확인
- 장애 시 자동주문을 끄는 절차 확인
- 장 시작 전 Python Brain, Java Core, DB, 네트워크 상태 확인

## 9. 장애 대응

### Python Brain 연결 실패

증상:

```text
Brain unavailable
```

대응:

```bash
curl http://localhost:8090/api/status
ps -ef | grep brain.server
```

Python Brain이 내려가면 Java Core는 HOLD로 실패 닫힘 처리한다.

### 한국투자 주문 실패

확인 항목:

- `KIS_APP_KEY`, `KIS_APP_SECRET`
- `KIS_ACCOUNT_NO`, `KIS_ACCOUNT_PRODUCT_CODE`
- `zest.kis.environment`
- 모의/실전 TR ID
- 계좌 예수금
- 한국투자 API 호출 제한

### Tick 폭주

확인 항목:

```bash
curl http://localhost:8090/api/status
```

`droppedTicks`가 증가하면 아래 값을 조정한다.

- `zest.trading.tick-queue-capacity`
- `zest.trading.worker-threads`
- Python Brain worker 수
- Brain deadline

## 10. 추천 운영 순서

1. 로컬 dry-run
2. 개발 서버 모의투자 dry-run
3. 개발 서버 모의투자 자동주문
4. 운영 서버 실계좌 dry-run
5. 운영 서버 소액 자동주문
6. 전략별 한도 확대

각 단계는 최소 하루 이상 로그와 체결 결과를 확인한 뒤 다음 단계로 넘어간다.
