## 4주차 LocalStack

> 목표 : 기여할 수 있는 이슈 정리해보기

- 기존에는 지난 주에 진행했던 서비스 흐름도를 코드 단에서 분석하는 것을 목표로 두었으나, 이슈를 하나 정하고 관련 코드를 파악해보는 것이 더 좋을 것 같다고 판단해 이슈 우선 탐색

### 이슈 탐색 기준

1. 친숙한 AWS 서비스일 것
2. 오픈 소스 기여가 처음이므로, 난이도가 적절해야 할 것

**문제점**

- Community 버전에서 지원하는 AWS 서비스가 제한적 ⇒ 기여할 수 있는 부분이 적음
- `labeling`이 많아서 파악이 쉽지 않다….

## 이슈 후보

> https://github.com/localstack/localstack/issues/12457

**문제 상황**

- 원래 AWS SQS FIFO 큐의 경우 메시지 그룹 ID가 동일한 메시지들이라면(= `MessageGroupId`가 동일) 메시지 전송 순서를 보장해야 함.
- 그러나 LocalStack에서는 특정 그룹의 첫 번째 메시지가 보이지 않을 경우(=visibility timeout 중일 때), 그 다음 메시지가 꺼내져 순서 보장이 깨짐
- SQS Visibility Timeout이란
    - SQS 메시지를 누군가가 받으면 (receive-message 호출 시) 해당 메시지는 잠시 ‘숨겨져서(보이지 않게)’ 다른 소비자에게 안 보이게 됨. 이 숨겨진 기간을 가시성 타임아웃(Visibility Timeout)이라고 함
    - 이는 메시지를 받은 소비자(consumer)가 메시지를 처리하는 동안 다른 소비자로 하여금 그 메시지를 처리하지 못하도록 막기 위한 목적 (=중복 처리 금지)


**해결 방향**

- 메시지 그룹의 첫 번째 메시지가 보이지 않을 경우, 해당 그룹의 다른 메시지도 반환되지 않도록 로직을 수정해야 할 듯

---

<br>

> https://github.com/localstack/localstack/issues/12516



**문제 상황**

- LocalStack에서는 `SetIdentityHeadersInNotificationsEnabled` API를 구현하지 않아 이메일 헤더 정보를 SNS 이벤트 메시지에 포함하지 못하며, 시도할 경우 internal failure 발생
- SetIdentityHeadersInNotificationsEnabled 헤더란
    - 특정 이메일에 대해 SES가 생성하는 이벤트 알림 메시지에 이메일 헤더 정보를 포함할지 말지 유무를 설정하는 데 사용이 되는 API
    - 헤더 정보 포함 시 이메일에 대한 추가 정보를 쉽게 얻을 수 있기 때문에 사용

**참고 자료**

https://docs.localstack.cloud/references/coverage/coverage_ses/

https://docs.aws.amazon.com/ses/latest/APIReference/API_SetIdentityHeadersInNotificationsEnabled.html

**해결 방안**

실제 API를 추가하는 과정이 필요함

1. Provider에 SES API 호출하는 로직 추가해야 함
2. SNS 알림 구성 시 헤더 포함하는 로직 추가 필요

---
## 컨트리뷰션 가이드

https://github.com/localstack/localstack/blob/master/docs/CONTRIBUTING.md