## 6주차 LocalStack


## 이슈 선정

https://github.com/localstack/localstack/issues/12516

- AWS SES API 중 `set-identity-headers-in-notifications-enabled` 가 추가되어 있지 않아 internal server error 발생하는 버그

※ AWS SES([Amazon Simple Email Service](https://aws.amazon.com/ses)) : 사용자의 이메일 주소와 도메인을 사용해 이메일을 보내고 받기 위한 경제적이고 손쉬운 방법을 제공하는 이메일 서비스

## 코드 분석

> `localstack/localstack-core/localstack/services/ses` 에 있는 파일들 참고
> 
- 실제 SES 핵심 로직이 작성되어있는 파일은 `provider.py` → 해당 파일 집중

### 코드 주요 구성

1. **함수**
- `save_for_retrospection(sent_email: SentEmail)`
    - **역할**: 발송된 이메일을 파일시스템과 메모리에 저장
    - 이메일을 JSON 파일로 저장 (`/data/ses/{message_id}.json`)
- `recipients_from_destination(destination: Destination)`
    - **역할**: Destination 객체에서 모든 수신자 추출
    - ToAddresses, CcAddresses, BccAddresses 모두 함께 반환
- `get_ses_backend(context: RequestContext)`
    - 백엔드 역할을 하는 Moto 백엔드 인스턴스 불러오기

1. **메인 프로바이더 → `SesProvider`**
    
    ✅ 이 부분이 AWS SES의 모든 API 요청을 처리하는 핵심 클래스 모음
    
    - AWS SES API 호출을 가로채고 처리하는 역할
    - SES의 여러 API를 `@handler`로 구현
        - 현재 지원하는 ses api
            - **SendEmail**: Send a standard email with subject, body (text/html), and recipients.
            - **SendRawEmail**: Send a raw MIME email, allowing custom headers and attachments.
            - **SendTemplatedEmail**: Send an email using a pre-defined template and template data.
            - **CreateConfigurationSetEventDestination**: Add an event destination (like SNS) to a configuration set.
            - **DeleteConfigurationSet**: Remove a configuration set.
            - **DeleteConfigurationSetEventDestination**: Remove an event destination from a configuration set.
            - **ListTemplates**: List all email templates.
            - **DeleteTemplate**: Delete a specific email template.
            - **GetIdentityVerificationAttributes**: Get verification status for email identities.
            - **CloneReceiptRuleSet**: Clone an existing receipt rule set.
        
        ```python
        # ex) ListTemplates : AWS 리전의 AWS 계정에 있는 이메일 템플릿 나열
        @handler("ListTemplates")
        def list_templates(
            self,
            context: RequestContext,
            next_token: NextToken = None,
            max_items: MaxItems = None,
            **kwargs,
        ) -> ListTemplatesResponse:
            backend = get_ses_backend(context)
            for template in backend.list_templates():
                if isinstance(template["Timestamp"], (date, datetime)):
                    template["Timestamp"] = timestamp_millis(template["Timestamp"])
            return call_moto(context)
        ```
        
    - 여기에 핸들러를 추가하면 되지 않을까 생각

---

### 코드 흐름 정리

**ex) 클라이언트에서 이메일 보낼 때(`sendEmail`)**

1. **클라이언트 요청**
    
    ```bash
    ses_client.send_email(
        Source='marketing@company.com',
        Destination={
            'ToAddresses': ['user1@example.com', 'user2@example.com']
        },
        Message={
            'Subject': {'Data': 'Summer Campaign 2024'},
            'Body': {
                'Text': {'Data': 'Check out our summer deals!'},
                'Html': {'Data': '<h1>Summer Deals!</h1>'}
            }
        },
        Tags=[
            {'Name': 'campaign_id', 'Value': 'summer-2024'},
            {'Name': 'segment', 'Value': 'premium'}
        ]
    )
    ```
    
2. **요청 파라미터 검증**
    1. 요청에서 tag가 유효한지 검증
    
    ```bash
    # 올바른 태그들
    good_tags = [
        {"Name": "campaign_id", "Value": "summer-2024"},      # 캠페인 식별
        {"Name": "user_type", "Value": "premium"},            # 사용자 분류
        {"Name": "department", "Value": "marketing@team"}     # 부서 정보
    ]
    
    # 거부되는 태그들 예시
    bad_tags = [
        {"Name": "campaign#id", "Value": "test"},           # '#' 특수문자 불허
        {"Name": "", "Value": "empty-name"},                # 빈 이름 불허
        {"Name": "too_long" * 50, "Value": "test"},         # 255자 초과
        {"Name": "한글이름", "Value": "test"},              # ASCII 문자 아님
    ]
    ```
    
3. **Moto에서 처리 (`call_moto`)**
    1. Moto : 실제 대부분의 AWS 서비스들을 파이썬 가상의 환경에서 구현한 라이브러리
    2. Localstack : HTTP 요청 받기 / 태그 형식 및 권한 체크 / 부가 기능 연결
    
    ```python
    # moto.py
    def call_moto(context: RequestContext, include_response_metadata=False) -> ServiceResponse:
        """
        Call moto with the given request context and receive a parsed ServiceResponse.
    
        :param context: the request context
        :param include_response_metadata: whether to include botocore's "ResponseMetadata" attribute
        :return: a serialized AWS ServiceResponse (same as boto3 would return)
        """
        return dispatch_to_backend(context, dispatch_to_moto, include_response_metadata)
    ```
    
    ```bash
    # LocalStack에서 Moto 호출
    response = call_moto(context)
    
    # Moto 내부에서 실제로 하는 일:
    # 1. 메시지 ID 생성 (예: "0000014a-f896-4c07-b8ff-fxxxxxxxxxxxxx")
    # 2. 이메일 형식 검증 (source, destination 유효성)
    # 3. 메시지 구조 검증 (Subject, Body 필수 필드)
    # 4. 가상 이메일 발송 처리 (실제로는 메모리에 저장)
    # 5. AWS 형식의 응답 생성
    ```
    
4. **SNS Payload 구성**
    
    ```python
    # 수신자 목록 추출
    recipients = recipients_from_destination(destination)
    # destination에서 ToAddresses + CcAddresses + BccAddresses 모두 합침
    
    # SNS 이벤트용 페이로드 생성
    payload = SNSPayload(
        message_id=response["MessageId"],           # "0000014a-f896-4c07..."
        sender_email=source,                       # "sender@example.com"  
        destination_addresses=recipients,          # ["user1@test.com", "user2@test.com"]
        tags=tags                                 # [{"Name": "campaign", "Value": "summer"}]
    )
    ```
    
5. **SNS 이벤트 발행 (send/delivery)**
6. **메일 내용 JSON 저장**
    - 보낸 메일은 메모리에 저장하기 위해 JSON으로 변환
        
        ```python
        {
          "Id": "0000014a-f896-4c07-b8ff-fxxxxxxxxxxxxx",
          "Region": "us-east-1", 
          "Source": "marketing@company.com",
          "Destination": {
            "ToAddresses": ["user1@example.com", "user2@example.com"],
            "CcAddresses": [],
            "BccAddresses": []
          },
          "Subject": "Welcome to our summer campaign!",
          "Body": {
            "text_part": "Hello! Welcome to our amazing summer campaign...",
            "html_part": "<h1>Hello!</h1><p>Welcome to our amazing summer campaign...</p>"
          },
          "Timestamp": "2024-06-10T14:30:00.123Z"
        }
        ```
        
7. **최종 SendEmailResponse 반환**
    - 클라이언트에게 응답 반환
        
        ```python
        {
          "MessageId": "0000014a-f896-4c07-b8ff-fxxxxxxxxxxxxx",
          "ResponseMetadata": {
            "RequestId": "12345678-1234-1234-1234-123456789012",
            "HTTPStatusCode": 200,
            "HTTPHeaders": {
              "content-type": "application/x-amz-json-1.0",
              "date": "Mon, 10 Jun 2024 14:30:00 GMT"
            }
          }
        }
        ```