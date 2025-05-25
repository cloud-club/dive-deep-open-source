## 3주차 LocalStack

> 목표 : LocalStack 아키텍처 개요 및 S3 실습 내부 구현 분석하기

### LocalStack 아키텍처 개요

- LocalStack은 단일 도커 컨테이너로 동작하는 로컬 AWS 에뮬레이터
- 전체 아키텍처에서 구성 요소는 아래와 같음

**Edge Router**

- 컨테이너 내부에는 **Edge Router** 역할을 하는 입구가 위치
- AWS API 요청은 모두 단일 포트(4566번)으로 취합
- 서비스 이름(ex. `s3`)나 호스트 이름(`<bucket>.s3.localhost.localstack.cloud`) 식의 패턴을 기반으로 요청을 적당한 서비스 프로바이더(ex. S3, Lamdba, DynamoDB etc)로 라우팅하는 역할

**서비스 프로바이더 및 핸들러**

- 각 AWS 서비스들은 LocalStack의 _서비스 프로바이더_ 로 구현이 되어있음
- `localstack/services/서비스명/` 내부에 프로바이더 클래스들과 API 핸들러가 구현되어있음
- AWS API 호출이 일어나면 이를 처리하는 역할 

**데이터 백엔드**
- LocalStack 내에서 AWS 서비스별 데이터를 저장하는 저장소
- 기본적으로 `/tmp/localstack/` 경로에 데이터 저장

**전체 아키텍처 예시**

※ AWS Lambda의 경우 별도의 도커 컨테이너를 띄워서 실행하도록 짜여져있다고 함
```text
                             ┌────────────────────────────┐
                             │      Client (User)         │
                             │ e.g., curl, AWS CLI, SDK   │
                             └────────────┬───────────────┘
                                          │ 
                                          ▼
                             ┌────────────────────────────┐
                             │       Edge Router          │
                             │ (localhost:4566 listener)  │
                             └────────────┬───────────────┘
                                          │ (Host/Path 기반 라우팅)
                                          ▼
                ┌────────────────────────────────────────────────┐
                │              Service Router                    │
                │ (ex. s3 → S3Provider, lambda → LambdaProvider) │
                └────────────┬──────────────────────┬────────────┘
                             │                      │
          ┌──────────────────▼──────────┐  ┌────────▼────────────────┐
          │        S3Provider           │  │      LambdaProvider     │
          │  (S3 서비스 API 처리 로직)    │  │   (Lambda 배포/실행 로직) │
          └────────────┬────────────────┘  └────────────┬────────────┘
                       │                               │
     ┌─────────────────▼────────────┐       ┌──────────▼───────────────┐
     │      API 핸들러들             │       │  Lambda Executor (Docker)│
     │ (put_object, get_object 등)  │       │   → Lambda 컨테이너 실행   │
     └────────────┬─────────────────┘       └────────────┬─────────────┘
                  │                                      │
    ┌─────────────▼─────────────┐           ┌────────────▼─────────────┐
    │      Data Backend         │           │      Lambda Runtime      │
    │ (/tmp/localstack/data/*)  │           │   (컨테이너 내부 실행 환경) │
    └───────────────────────────┘           └──────────────────────────┘

```

## S3 정적 웹사이트 흐름 살펴보기

**1. 클라이언트 요청 수신**

- 클라이언트가 `http://testbucket.s3-website.localhost.localstack.cloud:4566/` 형태로 HTTP GET 요청 전송
- 이러면 LocalStack 컨테이너 4566번 포트로 요청 들어옴

**2. 라우팅 및 버킷 식별**

- Edge Router는 호스트 헤더(`testbucket.s3-website.localhost.localstack.cloud`)을 보고 S3 서비스로 라우팅
- LocalStack은 _가상 호스트 스타일(virtual hosted path)_ 과 _경로 스타일(path style)_ 모두 지원한다고 알려짐 [(관련 내용)](https://docs.localstack.cloud/user-guide/aws/s3/#path-style-and-virtual-hosted-style-requests)

    <details>
    <summary>가상 호스트 vs 경로 스타일</summary> 
    가상 호스트 

    - 호스트 이름에 버킷 이름 포함
    - `bucket-name.s3.localhost.localstack.cloud:4566/key-name`
    - AWS 현재 권장 방식

    경로 스타일
    
    - 요청 URI 경로에 버킷 이름이 포함되는 방식
    - `localhost:4566/bucket-name/key-name`
    - 과거에 많이 사용된 방식

    </details> 


**3. S3 Provider로 전달**

- 요청이 S3 프로바이더로 전달될 경우, S3 프로바이더는 `GetObject`, `ListObjects` 등 적절한 API 핸들러 실행
- 이때 모든 AWS SDK 요청은 `RequestContext`라는 내부적으로 AWS 요청을 처리할 때 사용하는 객체로 변환되어 API 핸들러에게 전달됨

**4. 정적 웹사이트 로직 처리**

- S3 Provider 내부에서 별도 로직으로 정적 웹사이트 호스팅 
- 요청 경로가 비었거나 폴더를 가리키면 `index.html` 파일을 기본적으로 사용
- 객체 존재 시 해당 파일 내용이 호스팅되며, 존재하지 않을 경우 설정된 에러 파일(`error.html`) 내용이 반환되어 HTTP 404 반환

**5. 객체 반환**
- 버킷 저장소에서 객체 데이터 읽어와 HTTP 응답 바디로 반환
- 예를 들어 `awslocal s3api get-object --bucket foo --key index.html` 호출 시 S3 프로바이더는 `tmp/localstack/data/s3/` 등에서 파일을 찾아 반환