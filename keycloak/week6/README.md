# Keycloak - ID token에 custom attribute 주입 과정

경로:
services/src/main/java/org/keycloak/protocol/oidc/mappers/UserAttributeMapper.java

### 📌 핵심 메서드: 사용자 속성을 토큰에 Claim으로 주입
```java
protected void setClaim(IDToken token, ProtocolMapperModel mappingModel, UserSessionModel userSession) {

    UserModel user = userSession.getUser();
    String attributeName = mappingModel.getConfig().get(ProtocolMapperUtils.USER_ATTRIBUTE);
    boolean aggregateAttrs = Boolean.valueOf(mappingModel.getConfig().get(ProtocolMapperUtils.AGGREGATE_ATTRS));
    Collection<String> attributeValue = KeycloakModelUtils.resolveAttribute(user, attributeName, aggregateAttrs);
    if (attributeValue == null) return;
    OIDCAttributeMapperHelper.mapClaim(token, mappingModel, attributeValue);
}
```
### 토큰 생성 시 Mapper 적용 흐름
```java
    public AccessToken transformAccessToken(KeycloakSession session, AccessToken token,
                                            UserSessionModel userSession, ClientSessionContext clientSessionCtx) {
        AccessToken accessToken = ProtocolMapperUtils.getSortedProtocolMappers(session, clientSessionCtx, mapper -> mapper.getValue() instanceof OIDCAccessTokenMapper)
                .collect(new TokenCollector<AccessToken>(token) {
                    @Override
                    protected AccessToken applyMapper(AccessToken token, Map.Entry<ProtocolMapperModel, ProtocolMapper> mapper) {
                        return ((OIDCAccessTokenMapper) mapper.getValue()).**transformAccessToken**(token, mapper.getKey(), session, userSession, clientSessionCtx);
                    }
                });
        final ClientModel[] requestedAudienceClients = clientSessionCtx.getAttribute(Constants.REQUESTED_AUDIENCE_CLIENTS, ClientModel[].class);
        if (requestedAudienceClients != null) {
            restrictRequestedAudience(accessToken, Arrays.stream(requestedAudienceClients)
                    .map(ClientModel::getClientId)
                    .collect(Collectors.toSet()));
        }
        return accessToken;
    }

```
### 호출 흐름
> TokenManager.transformAccessToken() <br>
    └─> filter OIDCAccessTokenMapper <br>
        └─> call UserAttributeMapper.transformAccessToken <br>└─> call setClaim(...)
> 

### 🔁 동작 흐름 요약

1. `ProtocolMapperUtils.getSortedProtocolMappers...)`
    
    → 등록된 모든 Mapper 중 **`OIDCAccessTokenMapper`** 구현체만 필터링해서 순회
    
2. 각 Mapper에 대해 아래 코드 실행:
    
    ```java
    ((OIDCAccessTokenMapper) mapper.getValue())
        .transformAccessToken(token, mapper.getKey(), session, userSession, clientSessionCtx);
    ```
    
3. → 이때 `UserAttributeMapper`는 `OIDCAccessTokenMapper`를 구현하므로 위에서 호출.
4. → `UserAttributeMapper.transformAccessToken(...)` 내부에서는 `setClaim(...)` 호출
5. → `setClaim(...)`에서 사용자 attribute 값을 추출해 `OIDCAttributeMapperHelper.mapClaim(...)`을 통해 Access Token에 포함