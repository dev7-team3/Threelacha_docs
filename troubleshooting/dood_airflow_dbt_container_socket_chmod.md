# DooD 환경의 Airflow-dbt 컨테이너 간 Docker 소켓 권한 장애

> 💡 요약 <br><br>로컬 환경과 다른 EC2(Ubuntu)의 엄격한 보안 정책으로 인해 발생한 Docker 소켓 권한 문제를 GID 매핑 시행착오를 거쳐 최적의 권한 조정(chmod 666)으로 해결.


# 1. 문제 상황

- 동일한 EC2 인스턴스 내에서 Airflow와 dbt를 각각 독립된 컨테이너로 구동하고, Airflow DAG가 dbt 컨테이너를 제어(Trigger)하는 **DooD(Docker-out-of-Docker)** 아키텍처를 구성함.
- dbt 컨테이너 단독 실행 시에는 모델이 정상 동작하나, Airflow를 통해 원격 호출할 경우에만 권한 오류(Permission Denied)가 발생하며 작업이 실패함.

**상세 오류 메세지)**

```prolog
permission denied while trying to connect to the docker API at unix:///var/run/docker.sock
```

로컬 환경(Windows/macOS)에서는 아래와 같이 `docker-compose.yaml` 설정에 호스트의 Docker 소켓을 공유하는 볼륨 마운트 설정만으로 정상 동작하였으나, EC2 이관 후에는 오류 발생.

```yaml
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

- 참고) dbt 컨테이너 내 모델 동작 정상 구동 화면
    
    <img width="864" height="870" alt="docker_img_normal" src="https://github.com/user-attachments/assets/ff5ec535-771d-4a71-8a9f-cc227f090ecf" />

    

# 2. 원인 분석

**운영체제별 Docker 권한 관리 차이**

- **Windows/macOS:** Docker Desktop이 내부적으로 권한을 중계하므로, 일반 사용자 계정에서도 소켓 접근이 용이함.
- **Linux (EC2 Ubuntu):** 보안을 위해 `/var/run/docker.sock` 파일의 소유권이 `root:docker`로 설정되어 있으며, 기본적으로 **660 권한**(소유자와 그룹만 읽기/쓰기 가능)을 가짐.

>💡 Airflow 컨테이너 내부의 실행 유저(`airflow`)가 호스트 시스템의 `docker` 그룹에 속해 있지 않거나, 해당 그룹의 GID(Group ID)가 호스트와 일치하지 않아 발생하는 접근 제어 현상으로 판명됨.


# 3. 해결 방안

## 3.1 Airflow GID를 docker GID와 동일하게 부여

### 3.1.1. 적용 과정

① 도커 GID 확인

```bash
getent group docker
```

<img width="486" height="31" alt="image" src="https://github.com/user-attachments/assets/16c5098e-535a-4ed3-aabf-5c11e49724b8" />


② `docker-compose.yaml`파일 반영

|  | 변경 전 | 변경 후 |
| --- | --- | --- |
| 설정 값 | user: "${AIRFLOW_UID:-1000}:0" | user: "${AIRFLOW_UID:-1000}:999" |

### 3.1.2 결과  ❌

- airflow 컴포넌트 컨테이너가 계속해서 재실행하는 현상 반복(`CrashLoopBackOff`)
    
    <img width="1260" height="161" alt="image" src="https://github.com/user-attachments/assets/de345ac3-5abc-4387-8de7-c9f8cd447d23" />

    
- apiserver  에러 로그 확인
    
    <img width="738" height="220" alt="image" src="https://github.com/user-attachments/assets/239e6f7d-47bc-4e80-9b31-a21c421f0172" />

    

>💡 로그 확인 결과, Airflow 이미지는 **반드시 `GID=0` 값**으로 구동 가능함 확인 <br> 공식문서 확인 결과, Airflow 2.2 버전 이후 `AIRFLOW_GID` 매개변수는 더 이상 사용되지 않음 확인<img width="789" height="70" alt="image" src="https://github.com/user-attachments/assets/b940df67-1074-4276-bc34-fabe84eec0d2" />



- [Airflow공식문서_Running Airflow in Docker](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)

## 3.2 원복 후, Docker 소켓 권한 변경

### 3.2.1 변경 과정 및 결과 🟢

**권한 변경 전)  airflow 컨테이너 내부에서 docker 명령어 수행 안됨 확인**

```bash
# 1. Docker 소켓 권한 확인
echo "=== Docker Socket 권한 (변경 전) ==="
ls -lh /var/run/docker.sock
stat /var/run/docker.sock

# 2. 현재 상태 스크린샷 저장
ls -lh /var/run/docker.sock > ~/docker_sock_before.txt

# 3. Docker 그룹 ID 확인
echo "=== Docker 그룹 정보 ==="
getent group docker
stat -c '%g' /var/run/docker.sock

# 4. Airflow Worker 컨테이너에서 Docker 접근 테스트
echo "=== Airflow Worker에서 Docker 접근 테스트 (변경 전) ==="
docker exec airflow-worker bash -c "docker ps" 2>&1
echo "Exit code: $?"
```

<img width="800" height="341" alt="image" src="https://github.com/user-attachments/assets/fd939bea-1e17-40b4-8158-1bef7a8d9ad4" />


**권한 변경 후) airflow 컨테이너 내부에서 docker 명령어 수행 성공**

```bash
# 1. 권한 변경 실행
echo "=== 권한 변경 실행 ==="
sudo chmod 666 /var/run/docker.sock

# 2. 변경 직후 확인
echo "=== Docker Socket 권한 (변경 후) ==="
ls -lh /var/run/docker.sock

# 3. Airflow Worker에서 Docker 접근 재테스트
echo "=== Airflow Worker에서 Docker 접근 테스트 (변경 후) ==="
docker exec airflow-worker bash -c "docker ps" 2>&1
echo "Exit code: $?"
```

<img width="1451" height="372" alt="image" src="https://github.com/user-attachments/assets/e85b55f5-9062-4010-b7d6-420a35b27909" />


# 4. 기존 구조 → 변경된 구조

| **구분** | **항목** | **AS-IS (기존 구조)** | **TO-BE (개선 구조)** |
| --- | --- | --- | --- |
| **EC2 OS** | **Docker Socket 권한** | **660** (root:docker 전용) | **666** (모든 유저 허용) |

# 5. 변경으로 인해 개선된 부분

- **DooD(Docker-out-of-Docker) 환경의 권한 병목 해결**
    - 보안 정책이 엄격한 리눅스(EC2) 환경에서 발생하는 Docker 소켓 접근 권한 문제를 해결하여, Airflow가 dbt 컨테이너를 정상적으로 트리거할 수 있는 **컨테이너 간 통신 환경을 완성함**.
- **시시각각 변하는 GID 설정으로부터의 독립성 확보**
    - 호스트 OS마다 상이할 수 있는 `docker` 그룹의 GID를 컨테이너 설정에 매번 매핑하는 대신, 소켓 권한 자체를 조정함으로써 **인프라 환경 변화에 유연하게 대응**할 수 있는 구조를 마련함.
- **Airflow 공식 이미지의 실행 무결성 유지**
    - GID를 강제로 변경할 때 발생하는 Airflow 공식 이미지의 엔트리포인트 실행 오류 및 무한 재시작 현상을 방지하고, **이미지 본래의 보안 설계(GID=0 기반)를 유지**하며 문제를 해결함.

# 6. 현재 구성 문제점 및 개선 방안

## **6.1 현재 구성의 문제점**

- **보안 취약성 노출**: `chmod 666` 설정은 호스트 시스템의 모든 유저에게 Docker 데몬 제어권을 부여하므로, 비인가 사용자에 의한 컨테이너 변조 위험이 존재함.
- **운영 관제 및 제어력 한계**: `BashOperator`를 통한 단순 쉘 실행 방식은 태스크 단위의 세밀한 리소스 제한이 불가능하며, 장애 발생 시 원인 파악을 위한 로깅 가시성이 낮음.

## **6.2 향후 고려 가능 개선 방안**

- **보안 계층 강화**: `docker-socket-proxy` 컨테이너를 배치하여 Airflow가 Docker API에 접근할 때 필요한 최소 권한(Read/Write)만 선별적으로 허용하는 보안 필터링 구현 고려
- **표준 오퍼레이터 도입**: `DockerOperator` 또는 dbt 전용 라이브러리(`Cosmos` 등)를 도입하여 워크플로우를 객체화하고, 인스턴스 사양에 최적화된 태스크 단위 리소스 관리 실현