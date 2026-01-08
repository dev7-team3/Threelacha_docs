# Threelacha_docs

> Threelacha 프로젝트의 문서화 레포지토리입니다. 아키텍처 결정 기록(ADR), 프로젝트 기획서, 중간 보고서 등 프로젝트 전반에 걸친 문서를 관리합니다.

## 📚 문서 구조

### 📄 보고서
프로젝트 진행 과정의 보고서 문서

- [프로젝트 기획서](./보고서/프로젝트_기획서.md) : 프로젝트의 기획 문서
- [중간 현황](./보고서/중간_현황.md) : 멘토님과의 중간 현황 발표
- [프로젝트 최종보고서](./보고서/프로젝트_최종보고서.md) : 프로젝트 최종 결과 보고서
- [대시보드 시연](./보고서/대시보드_시연.md) : Streamlit 대시보드 시연 자료

### 📘 ADR (Architecture Decision Records)
아키텍처 결정 기록 문서 모음

| 번호 | 제목 | 설명 |
| --- | --- | --- |
| ADR-001 | [프로덕션 DB 사용여부 판단](./ADR/adr-001-production-db-use-status.md) | Athena 쿼리엔진으로 S3 단일저장소 유지 결정 |
| ADR-002 | [데이터 수집 파이프라인 아키텍처 결정](./ADR/adr-002-data-ingestion-pipeline.md) | EC2 + Docker + Airflow 채택 |
| ADR-003 | [Airflow Executor 선택](./ADR/adr-003-airflow-executor.md) | Celery Executor 채택 |
| ADR-004 | [RDS 도입 결정](./ADR/adr-004-rds-adoption-for-dashboard-performance.md) | 대시보드 조회 성능 개선을 위한 RDS 도입 |


### 🔧 Troubleshooting
프로젝트 진행 중 발생한 문제 해결 과정

- 🚨 [Athena 워크그룹 설정 충돌에 따른 dbt 모델 S3 저장 경로 미적용 문제](./troubleshooting/athena_work_group_conflict_in_dbt_s3_location.md) : Athena 워크그룹의 설정 재정의 옵션으로 인한 dbt 모델 S3 저장 경로 미적용 문제 해결
- 🚨 [DooD 환경의 Airflow-dbt 컨테이너 간 Docker 소켓 권한 장애](./troubleshooting/dood_airflow_dbt_container_socket_chmod.md) : EC2 환경에서 DooD 아키텍처 구성 시 발생한 Docker 소켓 권한 문제 해결
- 🚨 [Parquet 내부 컬럼과 파티션 메타데이터 간 정합성 오류 해결](./troubleshooting/parquet_columns_partition_metadata_error.md) : Athena 파티션 메타데이터와 Parquet 파일 내부 컬럼 간 정합성 오류 해결
- 🚨 [t3.medium 환경에서의 Airflow 운영 안정화 및 리소스 최적화 전략](./troubleshooting/t3_medium_ec2_airflow_resource.md) : EC2 t3.medium 환경에서의 Airflow 운영 안정화 및 리소스 최적화 전략

## 🔗 관련 링크

- [ADR 디렉토리 README](./ADR/README.md)