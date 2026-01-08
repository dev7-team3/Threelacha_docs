# lambda 를 통한 인스턴스 기동 자동화

### 구현 배경 및 필요성

- **인프라 운영 정책 대응**
    
    본 프로젝트 환경은 클라우드 자원 비용 최적화를 위해 특정 시간대(21:00, 01:00, 05:00, 09:00)에 인스턴스를 자동으로 정지하는 엄격한 자원 관리 정책아래 운영되고 있음.
    
- **업무 연속성(Continuity) 확보**
    
    자동 정지 정책으로 인해 매일 오전 업무 시작 시점에 인스턴스를 수동으로 기동해야 하는 번거로움이 발생하며, 이는 데이터 파이프라인의 정시 실행과 개발 환경 접근에 차질을 줄 수 있음.
    
- **운영 효율화**
    
    반복적인 수동 개입을 제거하고, 정지 정책 직후인 오전 09:01에 즉시 자원을 가용 상태로 전환함으로써 개발 및 테스트 업무의 공백을 최소화하고자 함.
    

### 구현 방법

> 보안성과 운영 안정성을 최우선으로 고려하여 EventBridge와 Lambda를 연계한 이벤트 구동 방식을 적용.
> 
1. **스케줄링 트리거 구성**(Amazon EventBridge): 데이터 파이프라인이 시작되기 전인 매일 오전09:01(KST)에 Cron 표현식을 사용하여 트리거 이벤트를 발생하도록 설정
    
    <img width="928" height="652" alt="EventBridge" src="https://github.com/user-attachments/assets/105edc90-ee24-45ac-ab41-c018b009dfb6" />

    
2. **인스턴스 제어 로직 구현**(AWS Lambda): Python Boto3 라이브러리를 활용하여 EC2 및 RDS 인스턴스의 상태를 확인하고 start API를 호출하여 인스턴스 수명 주기 제어
    
    <img width="1176" height="772" alt="Lambda" src="https://github.com/user-attachments/assets/ac1b2031-b123-46ea-9087-c6540ce14179" />

    
3. **보안 권한 관리**(IAM Role): Lambda 함수에는 최소 권한 원칙을 적용하여 EC2와 RDS의 상태 변경 및 로그 기록에 필요한 권한만을 부여한 전용 IAM Role을 연결
    
    <img width="1922" height="906" alt="image" src="https://github.com/user-attachments/assets/c9d2611e-c2fd-4f11-846d-c306e242b2b5" />
    <img width="1978" height="616" alt="image" src="https://github.com/user-attachments/assets/34f30530-ec71-4898-8697-a56e4c988da5" />

4. **모니터링**(Amazon CloudWatch): Lambda 실행 로그와 에러 발생 여부를 CloudWatch Logs에 적재하여 실행 이력을 추적할 수 있도록 구성