# t3.medium 환경에서의 Airflow 운영 안정화 및 리소스 최적화 전략

> 💡 요약 <br><br>
EC2(t3.medium)의 자원 한계로 발생한 시스템 프리징의 원인을 커널 로그 분석으로 규명하고, Swap 메모리 확보 및 Airflow 환경 변수 최적화를 통해 서비스 안정성을 확보함.

# 1. 문제 상황

로컬 환경에서 검증을 마친 Airflow Docker Compose 환경을 AWS EC2(t3.medium)로 이관하여 배포하였으나, airflow 동작 수행 시 인스턴스가 응답 불능(Freezing) 상태에 빠지는 현상이 발생.

- **UI 기반 작업 중 장애:** Airflow Web UI에 접속하여 DB 및 API 통신을 위한 커넥션 정보를 등록하던 중, 페이지 로딩이 무한히 지속되며 인스턴스가 응답 불능 상태에 빠짐.
- **CLI 접근 시도:** 인스턴스 강제 재시작 후, UI 부하를 피하고자 CLI(명령행 인터페이스)를 통해 직접 등록을 시도하였으나 동일하게 시스템이 멈추는 현상이 재현됨.

# 2. 원인 분석

## 2.1 메모리 사용 현황 확인

```bash
$ free -h
```

> <img width="581" height="59" alt="1_free-h" src="https://github.com/user-attachments/assets/51c90ed6-00bb-4459-9565-a5e4b4e8fb0a" /> <br>💡전체 3.7GiB의 메모리 중 78%(2.9GiB)를 이미 점유 중이며, 여유 메모리(available)가 약 577MiB에 불과함 확인


## 2.2 시스템 로그 분석을 통한 메모리 부족(OOM) 현상 점검

```python
# oom 관련 시스템 에러로그 수집
sudo grep -i "out of memory \|oom\|kill" /var/log/syslog > /tmp/oom_check.log
# oom 관련 커널레벨 에러로그 수집
sudo grep -i "out of memory \|oom\|kill" /var/log/kern.log >> /tmp/oom_check.log
# 현재 커널 베시지 버퍼의 실시간 기록 중 검색
sudo dmesg | grep -i "out of memory \|oom\|kill" >> /tmp/oom_check.log

# 에러로그 확인
grep -i "killed process" /tmp/oom_check.log
grep -B 5 -A 5 "Out of memory" /tmp/oom_check.log
```
<img width="1754" height="843" alt="2_oom" src="https://github.com/user-attachments/assets/6a87c6ad-cfde-48ad-8f7b-d2294481ea89" />

# 3. 해결 방안

## 3.1 **Swap Memory** 2G 할당

### **3.1.1 할당 과정**

```bash
# 1. 시스템에 2GB 크기의 빈 파일을 생성하여 Swap 공간으로 사용할 기반 마련
sudo fallocate -l 2G /swapfile 

# 2. 생성된 Swap 파일의 권한을 600(소유자 읽기/쓰기만 허용)으로 제한하여 보안 강화
sudo chmod 600 /swapfile 

# 3. 해당 파일을 리눅스 스왑(Swap) 구역으로 초기화 및 포맷 수행
sudo mkswap /swapfile 

# 4. 구성된 스왑 파일을 활성화하여 시스템이 즉시 가상 메모리로 사용할 수 있도록 설정
sudo swapon /swapfile 

# 5. 재부팅 후에도 스왑 설정이 자동으로 유지되도록 /etc/fstab 설정 파일에 영구 등록
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 6. 할당 결과 확인
free -h
```
<img width="604" height="172" alt="3_swap_memory" src="https://github.com/user-attachments/assets/13d1cce3-ddd9-4a8e-b5b5-95f29e61a559" />

### **3.1.2 결과  ⚠️**

- 앞서 시도했던 airflow 커넥션 등록은 성공
- 하지만 dag 구동 후, 36개의 workflow가 생성되고 동작하지 않은 현상 발생
    
    <img width="1734" height="892" alt="4_airflow_workflow" src="https://github.com/user-attachments/assets/09681eec-2668-4ce3-8067-860877fec567" />

    
- worker 컴포넌트 로그 확인 → 워커 프로세스 강제 종료(`Process 'ForkPoolWorker-1' exited with 'signal 9 (SIGKILL)'`) 현상 발생함 확인
    
    ```bash
    docker logs threelacha_airflow_dbt-airflow-worker-1 --tail=50 
    ```
    
    <img width="1260" height="524" alt="5_docker_logs_airflow" src="https://github.com/user-attachments/assets/000179d0-0271-408a-a0da-567afe7bec97" />
    
    > **Swap Memory** 할당으로 airflow UI 접근은 원활해졌으나, airflow의 기본 설정값(`concurrency: 16 (prefork)`)으로는 t3.medium 사양의 ec2 내에서 정상적으로 workflow를 구동하기에 적합한 환경은 아님 확인
    
    

## 3.2 t3.medium 사양을 고려한 Airflow 리소스 제한 및 설정 최적화

### 3.2.1 docker compse 파일 수정

```yaml
  # Celery Worker 실행 안정성 관련 설정
  airflow-worker:
      environment:
	      <<: *airflow-common-env
	      AIRFLOW__CELERY__WORKER_CONCURRENCY: 2         # 동시에 실행할 수 있는 Celery task 수

```

### 3.2.2 결과  🟢
<img width="1588" height="770" alt="6_airflow_success" src="https://github.com/user-attachments/assets/afcad22a-42c7-43d1-aba7-2d32bdde15d3" />


## 4. 기존 구조 → 변경된 구조

| **구분** | **항목** | **AS-IS (기존 구조)** | **TO-BE (개선 구조)** |
| --- | --- | --- | --- |
| **EC2 OS** | **Swap Memory** | X | 2G |
| **docker-compose.yml** | **AIRFLOW__CELERY__WORKER_CONCURRENCY** | 16 | 2 |

## 5. 변경으로 인해 개선된 부분

- **시스템 가용성 및 인프라 안정성 확보**
    - 제한된 리소스(`t3.medium`) 환경에서 발생하던 **OOM(Out Of Memory) Killer에 의한 프로세스 강제 종료 현상을 근본적으로 해결**함.
    - 물리 메모리 한계를 Swap 메모리(2GB)로 보완하고, 애플리케이션의 자원 점유율을 하향 조정함으로써 시스템 프리징 없이 안정적인 서비스 유지가 가능해짐.
- **워크플로우 실행의 예측 가능성 증대**
    - 과도한 병렬 처리 설정(`Concurrency`)을 실제 하드웨어 사양에 맞춰 최적화함으로써, 태스크 간 자원 경합을 방지하고 **데이터 파이프라인의 실행 성공률을 제고**함.
- **비용 효율적인 운영 환경 구축**
    - 상위 인스턴스(t3.large 등)로의 즉각적인 사양 업그레이드 대신, **소프트웨어 최적화**를 통해 주어진 환경 내에서 최선의 성능을 이끌어냄으로써 프로젝트 운영 비용을 절감함.

## 6. 향후 개선 방안

### 6-1. Local Executor로의 전환

- 현재와 같이 단일 노드 운영 체제라면, 별도의 메시지 브로커(Redis)가 필요 없는 **Local Executor로 전환**하여 아키텍처를 단순화하고 추가적인 메모리 여유 공간을 확보 운영

### 6-2. 서비스 컴포넌트 분리 및 스케일 아웃 (Scale-out)

- 부하가 집중되는 **Worker 컴포넌트를 별도의 EC2 인스턴스로 분리**하거나, AWS의 관리형 서비스인 **MWAA(Managed Workflows for Apache Airflow)** 도입을 검토하여 관리 부담을 줄이고 확장성을 확보