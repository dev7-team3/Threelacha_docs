# RDS 도입 배경 - Athena 쿼리 지연 문제

### 초기 시스템 구조

- 초기 데이터 조회 아키텍처는 **S3에 Parquet 포맷으로 데이터를 저장**하고, **Athena를 조회 엔진으로 사용하는 구조**로 설계
- S3에서 S3로 데이터를 집계한 데이터를 저장하였고, 대시보드에서 데이터를 조회할 때도 SQL로 가능하였기 때문에 문제가 없을 것이라고 예상

### 문제

- 대시보드 조회 시 **응답 시간이 수 초 이상 소요되어 소량, 반복조회임에도 체감 지연이 발생**
- Athena는 소량의 데이터를 빈번하게 조회하는 환경에서는 매 쿼리 실행 시마다 객체 스캔과 메타데이터 로딩이 반복되면서, 조회량 대비 과도한 오버헤드가 발생
- 동일한 쿼리를 log를 찍어서 비교해 보았고 평균적으로 약 10배가 넘는 시간차이가 발생하는 것을 확인
    - `postrgreSQL` : 약 0.1초 이내
        <img width="907" height="52" alt="rds_log" src="https://github.com/user-attachments/assets/effe2fc4-8440-4039-a3a3-147797889c90" />

    - `athena` : 약 1.5초 이내
        <img width="1024" height="52" alt="athena_log" src="https://github.com/user-attachments/assets/2de136c0-e179-4525-aecd-0460faebd99a" />

### 결정

- `Athena`가 디스크 기반 오브젝트 스토리지를 스캔하는 구조인 반면, `PostgreSQL`은 빠른 응답이 가능한 OLTP 성향의 엔진이기 때문인 것으로 판단되어 RDS 도입을 결정