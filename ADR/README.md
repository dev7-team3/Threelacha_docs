## 📘 Architecture Decision Records (ADR)

### 🧭 목적 (Purpose)

이 리포지토리는 우리 팀의 **기술적 아키텍처 결정**을 기록하기 위해 존재합니다.
모든 중요한 기술 결정은 그 당시의 맥락, 고려했던 대안, 근거, 결과 등을 문서화하여
**지속 가능한 협업, 빠른 온보딩, 일관된 기술 판단**을 가능하게 합니다.

### 📚 ADR이란?

**ADR (Architecture Decision Record)** 는 특정 기술적 문제에 대해 **무엇을, 왜, 어떻게 결정했는지를 기록하는 문서**입니다.
짧고 명확한 문서로, 이후에 팀원들이 같은 결정을 반복하거나 질문하지 않도록 도와줍니다.

> 참고: [Architecture Decision Record](https://continuous-architecture.org/docs/practices/architecture-decision-records.html)

### 🔖 ADR 목록

| 번호      | 제목                                                                                  | 상태            | 요약                   |
| ------- | ----------------------------------------------------------------------------------- | ------------- | -------------------- |
| ADR-001 | [Deprecate cron in favor of scheduler](./adr-001-deprecate-cron.md)                 | 🔁 Superseded | cron 대신 Airflow로 전환  |
| ADR-002 | [Use Airflow for DAG orchestration](./adr-002-use-airflow-for-dag-orchestration.md) | ✅ Accepted    | 워크플로우 도구로 Airflow 채택 |
| ADR-003 | [Adopt batch processing for ETL](./adr-003-adopt-batch-processing.md)               | 🟡 Proposed   | 배치 기반 파이프라인 선택       |

> 새로운 ADR을 작성하면 이 표에도 추가해주세요. (상태 이모지는 변경 가능)

#### 📁 디렉토리 구조

```
/adr
  ├──adr-001-deprecate-cron.md
  ├──adr-002-use-airflow-for-dag-orchestration.md
  ├──adr-003-adopt-batch-processing.md
  ├── README.md

#### 🧱 파일 작명 규칙

* `adr-XXX-short-title.md` 형식 (번호는 순차적으로 증가)
* 제목은 \[Verb + 대상 + 맥락] 형식으로 짧게

예시:

* `adr-004-adopt-prefect-for-orchestration.md`

#### ✅ 상태 관리

* 🟡 `Proposed`: 제안된 상태, 아직 검토 중
* ✅ `Accepted`: 채택됨, 실행 중이거나 완료됨
* ❌ `Rejected`: 검토되었지만 채택되지 않음
* 🔁 `Superseded`: 다른 결정에 의해 대체됨
