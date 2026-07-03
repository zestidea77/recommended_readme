# ZEST AI Trader Python Brain 로컬 환경 구성 가이드

| 항목 | 내용 |
| --- | --- |
| 문서 버전 | v2026.06.12 |
| 기준 코드 | 현재 로컬 구현 기준 |
| 적용 범위 | gRPC Brain, 자동 proto 생성, model registry latest load, CNN/LSTM 및 PPO/SAC adapter, ONNX artifact |

이 문서는 로컬 Mac에서 Python Brain을 띄우고 Spring Boot Java Core가 gRPC로 AI 추론을 호출할 수 있게 만드는 절차를 정리한다.  
Python 버전은 `3.13.x`를 사용한다. 2026년 5월 24일 기준 Python 3.13 최신 패치 버전은 `3.13.13`이다.  
로컬 기본 포트는 Python Brain `50051`, Spring Boot `8090`이다.

## 1. 구성 개요

Python Brain은 Java Core와 같은 `src/main/proto/aitrader.proto` 계약을 사용한다.

- Java Core: tick 수신, RMS 검증, 주문 처리, 화면/API 제공
- Python Brain: 전략별 AI 모델 라우팅, 추론 응답 생성
- 통신 방식: Protobuf 기반 gRPC
- 로컬 DB: `application-local.yml` 기준 MariaDB `jdbc:log4jdbc:mariadb://3.37.143.246:3306/aitrader`

로컬에서는 두 가지 방식 중 하나로 실행한다.

| 방식 | 용도 | 특징 |
| --- | --- | --- |
| Python Brain 단독 실행 | AI 서버 로그를 따로 보며 개발 | 가장 권장되는 로컬 개발 방식 |
| Spring Boot 자동 실행 | Java 서버 하나만 실행하고 싶을 때 | `zest.brain.auto-start=true` 필요 |

## 2. 최초 1회 환경 구성

먼저 Python 3.13을 설치한다.

```bash
brew install python@3.13
```

Homebrew를 쓰지 않는다면 Python 공식 다운로드 페이지에서 Python `3.13.13` macOS installer를 설치한다.

```text
https://www.python.org/downloads/latest/python3.13/
```

프로젝트 루트에서 실행한다.

```bash
cd /Users/zest/git/stoackAI
python/scripts/setup_local.sh
```

이 script가 수행하는 일은 다음과 같다.

- `python/.venv` 가상환경 생성
- `python/requirements.txt` 의존성 설치
- `src/main/proto/aitrader.proto` 기준 Python gRPC 코드 생성
- 생성 위치: `python/generated`

Python 버전을 직접 지정하려면 다음처럼 실행한다.

```bash
PYTHON_BIN=/opt/homebrew/bin/python3.13 python/scripts/setup_local.sh
```

기존 `python/.venv`가 Python 3.13이 아니면 setup script가 기존 venv를 백업 폴더로 옮긴 뒤 새로 만든다.

## 3. Python Brain 단독 실행

터미널 1에서 Python Brain을 실행한다.

```bash
cd /Users/zest/git/stoackAI
python/scripts/run_brain.sh
```

정상 시작 로그 예시는 다음과 같다.

```text
[ZEST Brain] starting gRPC server on 127.0.0.1:50051
INFO:root:ZEST Brain gRPC server started on 127.0.0.1:50051
```

포트나 worker 수를 바꾸려면 환경변수를 넘긴다.

```bash
BRAIN_HOST=127.0.0.1 BRAIN_PORT=50052 BRAIN_WORKERS=4 python/scripts/run_brain.sh
```

## 4. Python Brain 응답 확인

터미널 2에서 gRPC smoke test를 실행한다.

```bash
cd /Users/zest/git/stoackAI
python/.venv/bin/python python/scripts/smoke_test.py
```

다른 포트로 띄운 경우:

```bash
python/.venv/bin/python python/scripts/smoke_test.py --host 127.0.0.1 --port 50052
```

정상 응답 예시는 다음과 같다.

```json
{
  "strategy": "SCALPING",
  "action": "HOLD",
  "weight": 0.0,
  "confidence": 0.55,
  "reason": "momentum below threshold"
}
```

## 5. Spring Boot와 연결 실행

Python Brain을 단독으로 띄운 상태에서 Java Core를 실행한다.

```bash
cd /Users/zest/git/stoackAI
./gradlew bootRun
```

`application-local.yml`의 기본 Brain 연결값은 다음과 같다.

```yaml
zest:
  brain:
    enabled: true
    auto-start: false
    host: 127.0.0.1
    port: 50051
```

Python Brain 포트를 바꿨다면 Java Core도 같은 포트를 보게 실행한다.

```bash
./gradlew bootRun --args='--zest.brain.port=50052'
```

## 6. Spring Boot가 Python Brain도 같이 실행하도록 설정

Python 가상환경을 먼저 준비한다.

```bash
cd /Users/zest/git/stoackAI
python/scripts/setup_local.sh
```

그 다음 Spring Boot를 다음처럼 실행한다.

```bash
./gradlew bootRun --args='--zest.brain.auto-start=true'
```

이 방식에서는 Spring Boot 시작 시 다음 작업이 자동으로 수행된다.

- `python/scripts/generate_proto.sh` 실행
- `python -m brain.server` 실행
- Spring Boot 종료 시 같이 띄운 Python process 종료

로컬 편의 기능이므로 운영에서는 Python Brain과 Java Core를 별도 프로세스로 분리한다.

## 7. Java Core에서 AI 호출 확인

Spring Boot가 떠 있으면 tick API로 Brain 호출을 확인할 수 있다.

```bash
curl -X POST http://localhost:8090/api/ticks \
  -H 'Content-Type: application/json' \
  -d '{
    "ticker": "005930",
    "price": 82000,
    "volume": 1200,
    "strategy": "SCALPING",
    "dataType": "TICK"
  }'
```

대시보드는 다음 주소에서 확인한다.

```text
http://localhost:8090/trader
```

로그에서 `Brain unavailable`이 나오지 않으면 Java Core가 Python Brain과 통신하고 있는 상태다.

## 8. 현재 모델 구조

현재 Python Brain은 실제 모델 파일을 넣기 전의 adapter 구조로 구성되어 있다.

| 파일 | 역할 |
| --- | --- |
| `python/brain/server.py` | gRPC 서버와 전략 라우터 |
| `python/brain/models/scalping.py` | tick 기반 SCALPING placeholder 모델 |
| `python/brain/models/rl_portfolio.py` | portfolio state 기반 RL_PORTFOLIO placeholder 모델 |
| `python/brain/health_api.py` | 선택 실행 가능한 HTTP health endpoint |

실제 모델을 붙일 때는 gRPC 계약을 바꾸지 말고 각 모델 class의 `predict` 내부를 교체한다.  
이렇게 하면 Java Core, RMS, 주문 처리 코드는 그대로 두고 Python 추론만 교체할 수 있다.

## 9. 문제 해결

### generated protobuf files are missing

Python gRPC 코드가 생성되지 않은 상태다.

```bash
cd /Users/zest/git/stoackAI
PYTHON_EXECUTABLE=python/.venv/bin/python python/scripts/generate_proto.sh
```

### No module named grpc_tools

가상환경 의존성이 설치되지 않았거나, Python 3.13이 아닌 interpreter로 proto 생성을 실행한 상태다.

```bash
python/scripts/setup_local.sh
```

정상 로그에는 다음 줄이 보여야 한다.

```text
[ZEST Brain] generating proto with /Users/zest/git/stoackAI/python/.venv/bin/python
```

그리고 버전은 다음처럼 `3.13.x`여야 한다.

```bash
python/.venv/bin/python --version
```

### Brain unavailable

Java Core가 Brain gRPC 서버에 연결하지 못한 상태다.

확인 순서:

```bash
python/scripts/run_brain.sh
python/.venv/bin/python python/scripts/smoke_test.py
curl http://localhost:8090/api/status
```

확인할 설정:

- Python Brain 포트와 `zest.brain.port`가 같은지
- `zest.brain.enabled=true`인지
- 방화벽이나 이미 사용 중인 포트가 없는지

### address already in use

이미 `50051` 포트를 쓰는 Python Brain이 떠 있는 상태다.

다른 포트로 실행한다.

```bash
BRAIN_HOST=127.0.0.1 BRAIN_PORT=50052 python/scripts/run_brain.sh
./gradlew bootRun --args='--zest.brain.port=50052'
```

## 10. 운영 전환 시 주의

로컬에서는 placeholder 모델과 dry-run 기본값으로 개발한다.  
실제 투자 전에는 반드시 다음을 분리해서 점검한다.

- Python Brain process supervisor 적용
- model artifact 경로와 버전 관리
- gRPC timeout과 worker 수 부하 테스트
- KIS 실전 설정과 `zest.trading.auto-trading-enabled` 별도 승인 절차
- Brain 장애 시 HOLD fail-closed 동작 확인
