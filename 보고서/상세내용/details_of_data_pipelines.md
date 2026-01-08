# 데이터 파이프라인 단계별 상세 정의

## 전체 파이프라인 실행 흐름

- **실행 주기**
    - 매일 UTC 01:00 (KST 10:00)
- **실행 순서**
    - Raw 수집 (API 1 → 10 → 17)
    - Silver 변환 (API별 순차 실행)
    - DBT Gold 변환 (Silver 완료 후)
    - RDS 적재 (Gold 완료 후)
- **의존성 관리**
    - `Master Pipeline` DAG에서 각 단계를 순차 트리거

### 1. Raw 레이어: 원천 데이터 수집

- **목적**
    - 외부 공공 API(KAMIS)로부터 **일별 원천 데이터 수집**
- **데이터 형식**
    - JSON
- **저장 위치**
    - `s3://raw/api-{번호}/dt={날짜}/product_cls={분류}/country={지역}/category={카테고리}/data.json`
- **처리 내용**
    - API 호출 및 응답 데이터 유효성 검증
    - 메타데이터 추출 후 S3 파티션 경로 동적 생성
    - 원본 JSON 데이터 무가공 저장 (재처리 및 추적 가능성 확보)
- **실행 주기**
    - 일별 실행 (전일 데이터 기준)
- **주요 DAG**
    - `raw_api1_collect_daily` , `raw_api10_collect_daily` , `raw_api17_collect_daily`

---

### 2. Silver 레이어: 데이터 정제 및 Parquet 변환

- **목적**
    - Raw JSON 데이터를 **정제·표준화하여 분석 가능한 형태로 변환**
- **저장 위치**
    - `s3://silver/api-{번호}/year={연도}/month={월}/data_{날짜}.parquet`
- **처리 내용**
    - JSON 파싱 및 레코드 단위 변환
    - 코드 값 → 명칭 매핑 (상품 분류, 카테고리, 지역 등)
    - 파생 컬럼 생성 (요일, 주차, 주말 여부 등)
    - 데이터 품질 검증 (NULL 제거, 타입 캐스팅)
- **저장 전략**
    - Parquet 파일(Snappy 압축) 분리 저장
    - 월 단위 Hive format 방식으로 파티셔닝
    - 재처리 시 기존 데이터 영향 최소화
- **특징**
    - Athena에서 즉시 쿼리 가능한 구조
    - 정제 단계와 비즈니스 로직 분리
- **주요 DAG**
    - `silver_api1_transform_daily` , `silver_api10_transform_daily` , `silver_api17_transform_daily`

---

### 3. DBT 변환: Gold 마트 생성

- **목적**
    - Silver 데이터를 **비즈니스 요구사항에 맞는 Gold 마트로 변환**
- **실행 환경**
    - Docker 컨테이너 기반 dbt 실행
- **처리 단계**
    - Athena 파티션 동기화 (`MSCK REPAIR TABLE`)
    - `dbt run`을 통한 Gold 모델 생성
    - `dbt test`를 통한 데이터 품질 검증 (not_null, not_empty, 커스텀 테스트)
- **모델 유형**
    - **Ephemeral 모델**
        - 물리적 테이블 없이 CTE 기반 중간 변환
        - 최신 데이터 필터링 및 데이터 타입 정제
        - 예: `stg_daily_product_price`, `stg_eco_data_with_market_category`
    - **Seed 데이터**
        - CSV 기반 참조 데이터 테이블
        - 예: `seasonal_info.csv`, `country_coordinates.csv`
        - 메타데이터 스키마에 적재
- **주요 DAG**
    - `run_dbt_transform_gold`

---

### 4. Gold 레이어: 데이터 마트

- **목적**
    - 분석 및 대시보드 활용을 위한 **최종 비즈니스 마트 제공**
- **데이터 형식**
    - Parquet (Table materialization)
- **저장 위치**
    - `s3://gold/mart_{마트명}/`
- **마트 종류**
    - `mart_retail_channel_comparison` : 유통/전통 채널 가격 비교
    - `mart_retail_region_comparison` : 지역별 가격 비교
    - `mart_price_rise_top3` : 지역별 가격 상승 Top3
    - `mart_price_drop_top3` : 지역별 가격 하락 Top3
    - `mart_season_region_product` : 계절·지역별 상품 가격 분석
    - `mart_eco_price_statistics_by_category` : 카테고리별 가격 통계
- **처리 내용**
    - 집계 및 그룹 연산 (AVG, MIN, MAX, COUNT)
    - 비즈니스 로직 적용 (채널 분류, 계절 매핑)
    - 데이터 품질 검증 포함
- **특징**
    - 각 마트는 독립적인 분석 목적을 가진 최종 산출물

---

### 5. RDS 적재: PostgreSQL 동기화

- **목적**
    - Gold 마트 데이터를 **RDS PostgreSQL로 적재하여 BI·대시보드 조회 성능 확보**
- **대상 DB**
    - PostgreSQL / `mart` 스키마
- **처리 내용**
    - S3 Gold 디렉토리 기반 마트 테이블 자동 탐색
    - Parquet 스키마 분석 후 RDS 테이블 자동 생성
    - Full Load 방식 (TRUNCATE 후 COPY)
    - 트랜잭션 처리 (실패 시 롤백)
    - 테이블 통계 정보 갱신 (`ANALYZE`)
- **데이터 타입 처리**
    - Arrow 타입 → PostgreSQL 타입 자동 매핑
- **특징**
    - 신규 마트 추가 시 코드 변경 없이 자동 반영
    - 트랜잭션 기반 데이터 일관성 보장
- **주요 DAG**
    - `load_s3_gold_to_rds_mart`