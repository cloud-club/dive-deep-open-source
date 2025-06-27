## 7주차

### 진행 완료 

provider.py
```python

from localstack.aws.api.ses import (
    Address,
    AddressList,
    AmazonResourceName,
    CloneReceiptRuleSetResponse,
    ConfigurationSetDoesNotExistException,
    ConfigurationSetName,
    CreateConfigurationSetEventDestinationResponse,
    DeleteConfigurationSetEventDestinationResponse,
    DeleteConfigurationSetResponse,
    DeleteTemplateResponse,
    Destination,
    EventDestination,
    EventDestinationDoesNotExistException,
    EventDestinationName,
    GetIdentityVerificationAttributesResponse,
    Identity, # 추가
    IdentityList,
    IdentityVerificationAttributes,
    InvalidSNSDestinationException,
    ListTemplatesResponse,
    MaxItems,
    Message,
    MessageId,
    MessageRejected,
    MessageTagList,
    NextToken,
    RawMessage,
    ReceiptRuleSetName,
    SendEmailResponse,
    SendRawEmailResponse,
    SendTemplatedEmailResponse,
    SesApi,
    SetIdentityHeadersInNotificationsEnabledResponse, # 추가
    TemplateData,
    TemplateName,
    VerificationAttributes,
    VerificationStatus,
)


class SesProvider(SesApi, ServiceLifecycleHook):
    ...
    @handler("SetIdentityHeadersInNotificationsEnabled")
    def set_identity_headers_in_notifications_enabled(
        self,
        context: RequestContext,
        identity: Identity,
        notification_type: str,
        enabled: bool,
        **kwargs,
    ) -> SetIdentityHeadersInNotificationsEnabledResponse:
        """
        Sets whether Amazon SES includes the original email headers in the Amazon SNS notifications 
        for a specified identity and notification type.
        """
        backend = get_ses_backend(context)
        # Store the setting in the backend (moto may not support this, so we use a custom attribute)
        if not hasattr(backend, "identity_headers_in_notifications_enabled"):
            backend.identity_headers_in_notifications_enabled = {}
        backend.identity_headers_in_notifications_enabled.setdefault(identity, {})[notification_type] = enabled
        return SetIdentityHeadersInNotificationsEnabledResponse()
    ...
```

코드 설명 : 특정 이메일 주소 또는 도메인(identity)에 대해 특정 유형의 SNS 알림(notification_type)에 원본 이메일 헤더를 포함할지(enabled) 여부를 설정

---

### 진행 중

- 로컬 테스트 + 통합 테스트 코드 작성 예정!!
