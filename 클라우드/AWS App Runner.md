# AWS App Runner

- 완전관리형 서버리스 컨테이너 웹 애플리케이션 서비스
- 도메인 구성, TLS 설정, 무중단 배포, 로드밸런싱, 오토스케일링, 지표 및 로그, 자동배포 등의 기능을 복잡한 설정 없이 사용 가능. 관리 포인트가 매우 줄어들음
  - 커스텀이 어려운 것이 단점
  - 내부 동작 파악이 어려움 (문서에 모두 나와있지 않음, 개발자 코멘트 등 신뢰하기 어려운 정보 참고 필요)
- Heroku랑 딱 비슷한 PaaS 서비스. 같은 PaaS인 Elastic Beanstalk보다는 좀더 인프라가 추상화된 서비스
  - EB는 EC2를 직접 건들 수 있지만 App Runner는 인프라 관리 필요 없는 서버리스임
  - App Runner는 정말 설정할 게 거의 없고, CPU, Memory 용량 설정 정도만 중요함
- Node.js, Java (17은 안됨 ??) 등 런타임 지원하고 Docker Image 지원
- [Concurrency Compared: AWS Lambda, AWS App Runner, and AWS Fargate](https://nathanpeck.com/concurrency-compared-lambda-fargate-app-runner/)

## 빌드

- Code 기반 배포, Image 기반 배포가 있음
  - Code는 Git Repository에 직접 연동해서 네이티브 런타임으로 배포, Image는 Amazon ECR 연동
  - Image 기반 배포 사용하고 CI/CD로 Amazon ECR에 Image 푸시하는게 다른 클라우드 서비스 고려할때 확장성이 좋을듯

## 오토스케일링

- 로드밸런서에서 바로 애플리케이션으로 Request를 보내지 않고, Overflow Queue에 일단 보내서 Request 대기열을 두고 순차적으로 애플리케이션에 보냄 (메세지큐 같은거 잘 활용한듯)
- Queue에 설정된 Request 제한 개수를 넘으면 인스턴스를 오토스케일링으로 추가하는 방식. Max 인스턴스 개수까지 꽉차면 HTTP 429 Too Many Requests 오류 반환
  - 근데 한번에 Request가 들어오면 Max 인스턴스까지 다 추가는 되는데, 인스턴스에서 서버 Listen 준비가 안되어서 Request 처리가 안 돼 429 오류 발생 가능 (Listen한 이후에는 충분히 처리 가능한 Request 개수인데도)
    - Queue 별로 절대적인 Request 개수 제한을 두는 방식이라 유동적인 대응이 어려움 (인스턴스 추가를 하는 임계값과 429 오류를 발생시키는 임계값을 각각 설정할 수 있으면 좋을듯)
  - [Traffic drops due to error 429 Too Many Requests when App Runner is scaling out](https://github.com/aws/apprunner-roadmap/issues/224)
- Request 개수가 줄어들면 인스턴스 개수도 줄어들음
- 개발자가 Queue까지 붙이면서 직접 구현하는건 상당히 까다로운 방식인데, 스마트하게 다 구현되어 있고 별거 설정할 것 없이 Request 임계값만 정하면 됨. 그런데도 꽤 프로덕션에 적합해서 좋음
  - EB에서는 CPU 로드, 패킷 개수 등 물리적인 리소스 기준으로 로드밸런싱을 함. 기준잡기가 어려움
- 다만 이런 기본 동작을 다른 방식으로 커스텀이 불가능함. 무조건 HTTP Request 개수 기반 오토스케일링임


## 무중단 배포

- Blue-Green 배포 방식
  - 기존 서버(Blue)가 돌아가는 상태에서 신규 서버(Green)을 배포하고 Health Check가 완료되면 로드밸런서 라우팅을 신규 서버(Green)으로 변경, 기존 서버(Blue) 제거
  - Health Check 실패시 롤백
- 기본적으로 설정된 Health Check는 TCP와 HTTP. 선택의 폭이 넓지는 않음
  - HTTP는 Allow 상태 코드 커스텀이 안되는게 단점. 아마 200~399 대역까지 OK 판정인듯
- 가장 큰 단점: 배포가 상당히 오래 걸림. 전체적인 과정이 느리고 Health Check도 오래하는 것 같음 (최종 배포까지 3분쯤)

## 문제

### 임시 스토리지 용량 작음
- AWS App Runner는 서버리스답게 영구 스토리지는 없음
- 서비스 배포시 임시 스토리지는 주어지나 3GB에 불과함
- 파일 업로드 등 구현시 용량 부족할 수 있으니 구현 방법 고민 필요 (스트리밍, S3에 직접 업로드 등)
  - [Spring Boot에서 S3에 파일을 업로드하는 세 가지 방법](https://techblog.woowahan.com/11392/)

### amazon-app-runner-deploy GitHub Actions 사용시 API 주의
- StartDeployment API를 호출하지 않음. CreateService/UpdateService API만 호출
  - 단순히 설정 정보만 갱신하는 것. 이때 설정 정보가 기존과 달라지면 배포 등 알맞은 작업이 자동 진행됨
  - Image 배포를 위해서는 Image 이름 혹은 태그가 달라져야 배포가 진행됨
  - CI/CD 파이프라인 구성시 빌드마다 유니크한 Image 태그를 갖도록 구성

### 리전 지원 한정
- 현재 서울 리전 지원을 안함
- 서울 리전의 EventBridge Scheduler에서 타 리전의 AWS App Runner 대상으로 API 호출 시도시 404 오류 발생 (ServiceArn을 맞는 값으로 넘겼는데도)
  - ServiceArn 상관없이, 같은 리전의 AWS App Runner API 서버를 부를려고 해서 그런 것. 서울 리전에는 AWS App Runner API 서버가 없기에 404 오류가 발생함
  - AWS App Runner 지원하는 리전에서 API 호출해야 됨
