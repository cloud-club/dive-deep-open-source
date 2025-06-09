## 5주차 LocalStack

> 목표 : 이슈 확정 및 환경 테스트

### 이슈 선정

https://github.com/localstack/localstack/issues/12516


## 문제 재현

프로젝트 폴더에서

```python
project-root/
├── docker-compose.yml
└── localstack/
    ├── data/                     # 데이터 볼륨 (자동 생성됨)
    └── bootstrap/
        └── init.sh              # 초기화 스크립트
```

docker-compose.yml

```yaml
services:
  localstack:
    image: localstack/localstack:latest
    container_name: localstack
    restart: unless-stopped
    ports:
      - "4566:4566"             # LocalStack Gateway
      - "4510-4559:4510-4559"   # External services
    environment:
      - SERVICES=iam,ses,sqs,sns
      - DEFAULT_REGION=us-east-1
      - PERSISTENCE=1
      - DEBUG=1
      - QUEUE_NAME=app
      - QUEUE_EMAILS_NAME=app_emails
      - QUEUE_FAILED_NAME=app_failed_jobs
      - QUEUE_SES_EVENT_NAME=app_ses_events
      - AWS_VERIFIED_EMAIL=app@local.com
      - AWS_VERIFIED_DOMAIN=local.com
    volumes:
      - ./localstack/data:/var/lib/localstack
      - ./localstack/bootstrap:/etc/localstack/init/ready.d
      - /var/run/docker.sock:/var/run/docker.sock

```

init.sh

```bash
#!/bin/bash
set -e

echo "▶ Initializing LocalStack SES, SNS, and SQS..."

# 환경변수 불러오기
QUEUE_NAME=${QUEUE_NAME:-app}
QUEUE_EMAILS_NAME=${QUEUE_EMAILS_NAME:-app_emails}
QUEUE_FAILED_NAME=${QUEUE_FAILED_NAME:-app_failed_jobs}
QUEUE_SES_EVENT_NAME=${QUEUE_SES_EVENT_NAME:-app_ses_events}
AWS_VERIFIED_EMAIL=${AWS_VERIFIED_EMAIL:-app@local.com}
AWS_VERIFIED_DOMAIN=${AWS_VERIFIED_DOMAIN:-local.com}

# SQS 생성
awslocal sqs create-queue --queue-name "$QUEUE_NAME"
awslocal sqs create-queue --queue-name "$QUEUE_EMAILS_NAME"
awslocal sqs create-queue --queue-name "$QUEUE_FAILED_NAME"
awslocal sqs create-queue --queue-name "$QUEUE_SES_EVENT_NAME"

# SNS Topic 생성
TOPIC_ARN=$(awslocal sns create-topic --name "$QUEUE_SES_EVENT_NAME" --query 'TopicArn' --output text)

# SQS ARN 가져오기
QUEUE_URL="http://localhost:4566/000000000000/$QUEUE_SES_EVENT_NAME"
QUEUE_ARN=$(awslocal sqs get-queue-attributes --queue-url "$QUEUE_URL" --attribute-name QueueArn --query "Attributes.QueueArn" --output text)

# SNS -> SQS 구독 연결
awslocal sns subscribe --topic-arn "$TOPIC_ARN" --protocol sqs --notification-endpoint "$QUEUE_ARN"

# SES 구성 셋 생성 및 이벤트 설정
awslocal ses create-configuration-set --configuration-set '{"Name": "SesConfigSet"}'

for TYPE in "send" "delivery" "bounce" "complaint"; do
  awslocal ses create-configuration-set-event-destination --configuration-set-name "SesConfigSet" --event-destination "{
    \"Name\": \"${TYPE^}Event\",
    \"Enabled\": true,
    \"MatchingEventTypes\": [\"$TYPE\"],
    \"SNSDestination\": {
      \"TopicARN\": \"$TOPIC_ARN\"
    }
  }"
done

# 이메일/도메인 검증 및 설정
awslocal ses verify-domain-identity --domain "$AWS_VERIFIED_DOMAIN"
awslocal ses verify-email-identity --email-address "$AWS_VERIFIED_EMAIL"

awslocal ses set-identity-configuration-set --identity "$AWS_VERIFIED_DOMAIN" --configuration-set-name "SesConfigSet"
awslocal ses set-identity-configuration-set --identity "$AWS_VERIFIED_EMAIL" --configuration-set-name "SesConfigSet"

# 피드백 전달 비활성화
awslocal ses set-identity-feedback-forwarding-enabled --identity "$AWS_VERIFIED_EMAIL" --no-forwarding-enabled

# (❗실제 LocalStack에서 미구현 상태인 API들)
for TYPE in "Send" "Delivery" "Bounce" "Complaint"; do
  echo "Attempting to enable headers for $TYPE (may not be supported)..."
  awslocal ses set-identity-headers-in-notifications-enabled --identity "$AWS_VERIFIED_EMAIL" --notification-type "$TYPE" --enabled || true
done

# 확인용 출력
awslocal ses get-identity-notification-attributes --identities "$AWS_VERIFIED_EMAIL"

echo "✅ Initialization complete."

```

아래 명령어 실행 → 문제 재현 완료

```bash
$ docker-compose up -d
[+] Running 2/2
 ✔ Network cloudclub_opensource_default  Created                                                                                                                        0.1s 
 ✔ Container localstack                  Started   
 
$ awslocal ses set-identity-headers-in-notifications-enabled \
>   --identity "app@local.com" \
>   --notification-type Send \
>   --enabled

**An error occurred (InternalFailure) when calling the SetIdentityHeadersInNotificationsEnabled operation: The set_identity_headers_in_notifications_enabled action has not been implemented**
 
```


## 개발 환경 셋업

https://github.com/localstack/localstack/blob/master/docs/CONTRIBUTING.md

https://github.com/localstack/localstack/blob/master/docs/development-environment-setup/README.md

<details>
<summary> make install 과정에서 에러가 많이 났다.</summary>

- venv 설정 → source .venv/bin/activate → 종료할 경우 deactivate (WSL 환경)
- python 버전이 낮아서 에러가 나기도 했음 → `.python-version`에 명시된 걸로 변경 + nodejs도 마찬가지
    - 가상 환경 생성 시점의 Python 버전을 기준으로 가상 환경이 만들어지기 때문에, 새로운 파이썬 버전을 사용할 경우 새로 가상 환경을 만들어줘야 함
- WSL 환경 내 로컬 디렉토리 이용

    ```bash
    subprocess.TimeoutExpired: Command '['git', '--git-dir', '/[파일경로].git', 'status', '--porcelain', '--untracked-files=no']' timed out after 40 seconds
    ```

    - `setuptools_scm`은 소스 트리 상태를 보고 버전을 자동으로 추출하는데, 이 과정에서 `git status` 명령을 호출 → 근데 WSL에서 windows 경로를 마운트해서 사용하게 되면 git 명령이 오래걸림
    - WSL 리눅스 홈 디렉터리로 Localstack 프로젝트 복사

        ```bash
        cp -r [파일경로]
        cd ~/localstack
        make install
        ```

- 시스템 내부에서 관리되는 라이브러리의 경우 가상 환경 내부로 끌어오지 못하는 에러가 있었음
- 가상 환경을 나갔다가 다시 들어와도 설치해놓은 패키지들은 영구적으로 유지가 됨 (삭제하지 않는 이상 by `rm -rf .venv`)
- `ls -al .venv/bin` : 가상 환경에 설치된 실행 가능한 파일들 목록 보여줌

</details>
<br>
Consider running `make install-dev-types` development도 해줌

<br>

`make start`하니 잘 실행이 됨

---

### Makefile이란

`Makefile`은 **자동화된 명령어 집합을 정의한 파일**입니다. 일종의 작업 레시피로, 프로젝트의 **설치, 테스트, 빌드, 정리, 배포** 등을 한 줄 명령으로 쉽게 실행할 수 있게 도와줍니다.

**기본 개념**

**📌 타겟(target)**

```makefile
install: install-dev entrypoints
```

`install`이라는 작업을 실행하면 `install-dev`와 `entrypoints`라는 하위 작업도 차례로 실행됩니다.

**📌 명령(command)**

```makefile
$(VENV_RUN); $(PIP_CMD) install -r requirements-dev.txt
```

이건 실제로 쉘에서 실행할 명령어입니다.

**📌 의존성(dependency)**

`install`은 `install-dev`, `entrypoints`가 먼저 실행돼야 완료됩니다.
