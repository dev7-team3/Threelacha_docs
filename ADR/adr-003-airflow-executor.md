# Airflow Executor 선택: Celery Executor 채택
<div align="right">📅 2025-12-10</div>

> 💡 요약 <br><br>
> 본 프로젝트는 로컬 개발 환경에서 Airflow 3.1.4를 학습하고 검증하기 위해 **Celery Executor**를 채택하였다. 초기에는 구성의 단순함을 고려하여 Local Executor 사용을 검토했으나, Airflow 3.1.4에서 공식적으로 제공하는 Docker Compose 예제가 Celery Executor 기반이라는 점을 확인하면서, 로컬 환경에서도 운영 환경과 유사한 구성을 가져가는 것이 더 적절하다고 판단하였다. 이를 통해 Executor 변경 없이 운영 환경으로 확장 가능하고, 병렬 처리 및 워커 확장 구조를 로컬에서 실습할 수 있게 되었다.

## 1. Context (배경 및 문제 정의)

본 프로젝트에서는 Airflow 3.1.4를 학습하고 검증하기 위해 Docker 기반의 로컬 개발 환경을 구성하고자 하였다. Airflow의 Executor는 태스크 실행 방식을 결정하는 핵심 구성 요소로, 선택에 따라 시스템 아키텍처와 확장성이 달라진다.

초기에는 구성의 단순함을 고려하여 `Local Executor` 사용을 검토했으나, 실제로 필요한 구성 요소(메타스토어 DB, 스케줄러, 웹서버 등)를 하나씩 확인하며 직접 구성하는 과정에서 설정 복잡도가 예상보다 높아졌다.

또한 Airflow 3.1.4에서 공식적으로 제공하는 `docker-compose.yml` 예제가 Celery Executor 기반이라는 점을 확인하면서, 로컬 환경에서도 운영 환경과 유사한 구성을 가져가는 것이 더 적절하다고 판단하게 되었다.



## 2. Decision (결정)

- Celery Executor를 채택
    - 로컬 Docker 환경에서 Celery Executor 사용
    - Airflow 공식 Docker Compose 예제 기반 구성

하는 **Celery Executor 기반 Airflow 구성**을 채택하였다.

## 3. Alternatives Considered (고려했던 대안들)

### 3.1 Local Executor 사용

- 단일 프로세스에서 모든 태스크를 순차적으로 실행
- 별도의 메시지 브로커나 워커 프로세스 불필요

**장점**

- 구성 요소가 상대적으로 적고 초기 설정이 단순함
- 메시지 브로커(Redis/RabbitMQ) 및 워커 컨테이너 불필요로 리소스 사용량이 적음

**단점**

- Airflow 3.1.4에서 공식 Docker 예제가 제공되지 않음
- 직접 구성해야 하므로 설정 복잡도가 높아짐
- 병렬 처리 및 워커 확장 구조를 실습하기 어려움
- 운영 환경으로 확장 시 Executor 변경 필요

### 1.2 Celery Executor 사용

- 분산 태스크 실행을 위한 메시지 브로커 기반 구조
- 여러 워커를 통한 병렬 처리 지원

**장점**

- Airflow 3.1.4에서 공식적으로 제공되는 Docker Compose 예제 활용 가능
- 검증된 구성 방식을 따를 수 있어 안정적
- 병렬 처리 및 워커 확장 구조를 로컬에서 실습 가능


**단점**

- Redis(또는 RabbitMQ), Worker 등으로 인한 컨테이너 수 증가
- Local Executor 대비 초기 구성 및 리소스 사용량 증가


## 4. Rationale (결정의 이유)

- **공식 예제 활용 및 안정성 확보**
    - Airflow 3.1.4에서 공식적으로 제공되는 Docker Compose 예제가 Celery Executor 기반이므로, 검증된 구성 방식을 따르는 것이 안정적이라고 판단함
    - 공식 문서와 예제를 참고하여 문제 해결이 용이함
- **운영 환경과의 일관성**
    - 로컬 환경에서도 운영 환경과 유사한 구성을 가져감으로써 개발-운영 환경 간 일관성 확보
    - Executor 변경 없이 운영 환경으로 확장 가능하여 마이그레이션 리스크 최소화
- **장기적 확장성 고려**
    - 병렬 처리 및 워커 확장 구조를 로컬에서 실습할 수 있어 Airflow의 분산 처리 특성을 이해하기 용이함
    - 초기 설정은 다소 복잡하지만, 장기적으로 재사용 가능하고 확장성을 고려한 선택이라고 판단함
    - 프로젝트 규모가 커져도 동일한 Executor 구조를 유지할 수 있음

## 5. Consequences (영향 및 결과)

### 긍정적 영향

- Airflow 공식 예제 기반 구성으로 안정적인 로컬 개발 환경 구축
- Executor 변경 없이 운영 환경으로 확장 가능하여 마이그레이션 리스크 최소화
- 병렬 처리 및 워커 확장 구조를 로컬에서 실습 가능하여 학습 효과 향상
- 로컬 환경과 운영 환경의 일관성 유지로 개발 효율성 향상

### 한계 및 트레이드오프

- Redis, Worker 등으로 인한 컨테이너 수 증가로 리소스 사용량 증가
- Local Executor 대비 초기 구성 복잡도 및 리소스 사용량 증가


## 6. Related Decisions (관련 ADR 문서)

- [ADR-002: 데이터 수집 파이프라인 아키텍처 결정](./adr-002-data-ingestion-pipeline.md)

## 7. References (참고 자료)

- [Airflow Docker Compose 공식 문서](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)
- [(3.1) 로컬 환경 구성 및 데이터 수집](https://www.notion.so/3-1-2c26e9180a9680099bc7fb5fc1629e84?pvs=21)
