# PR 분석

## good first issue

### 1. **Add agent level tags to telemetry spans - Flow Mode #445**

[issue #445](https://github.com/grafana/alloy/issues/445)

open : 2023.11

```markdown
Request
Add agent level tags to telemetry spans. The function would just need to append the defined tags to outgoing spans.
Same functionality is used by Jaeger agents https://www.jaegertracing.io/docs/1.45/deployment/#agent-level-tags.

Use case
When running multiple Kubernetes clusters, separating spans based on tags is necessary. Without them, there is no way to seperate same service spans across multiple clusters. Example tags are environment=uat, account=xxx
```

<요약>

쿠버네티스 운영 편의를 위하여 telemetry span에 agent-level tag 등록을 요청하고 있다.

- telemetry span : 작업 단위(span)에 시간, 결과, 태그, 오류 정보 등(telemetry)이 포함되어 있는 것
  - trace가 요청이 처리되는 과정의 기록을 의미하는데, span은 trace 안의 세부 조각을 의미하게 됨
  - 즉 trace는 여러 개의 span으로 구성되어 있고, span은 telemtry를 지니고 있음
- agent-level tag : 에이전트가 자동적으로 부여하는 태그로 작업자가 일일이 태그를 추가하지 않아도 됨
  - 일관성 확보 가능
  - 예를 들어, 서비스에서 나오는 데이터 구조가 같을 때, 에이전트가 dev, prod 등으로 태그를 부여하여 구별 가능

***⇒ 즉 ,이 pr은 trace 정보를 span 별로 세세하게 모니터링 하기 위해서 agent-level tag 기능을 추가해달라는 의미임***

<답변 분석>

이에 대해 아래의 답변이 달림

```markdown
I'm trying to identify where the best place to configure and apply these would be.

Add agent level tags to telemetry spans.

Are you talking about just spans that the agent generates itself, or all spans, including spams received from the network (e.g., via otelcol.receiver.otlp)?
```

- agent-level tag 적용 범위에 대해 묻는 질문이다.
- agent가 tag를 override 할 것인지, skip 할 것인지 그 범위에 대한 질문
  - 기존에 있던 태그를 override 하면 정보에 오염이 발생할 수 있기 때문에 목적을 확실히 하기 위한 질문으로 보인다.
  - 질문자에 따르면, Jaeger agent의 agent level tag 기능을 생각해서 요청했다고 한다.
    - [issue #833](https://github.com/jaegertracing/jaeger/issues/833)
  
  - 2018년 경 Jaeger agnet에 agent level tag 기능 추가를 요청하면서 굉장히 많은 토론이 오고 갔다.
  - 불필요한 복잡성인가, 적용 범위, 대체 가능한 기술 등에 대한 얘기가 많았다.
  - 결과적으로는, gRPC reporting path에 agent level tag 기능을 제공하게 되었다고 한다.

    ```yaml
    Jaeger supports agent level tags, that can be added to the process tags of all spans passing through the agent.
    This is supported through the command line flag --jaeger.tags=key1=value1,key2=value2,...,keyn=valuen.
    Tags can also be set through an environment flag like so - --jaeger.tags=key=${envFlag:defaultValue} - The tag value will be set to the value of the envFlag environment key and defaultValue if not set.
    This feature is not supported for the tchannel reporter, enabled using the flags --collector. host-port or --reporter.tchannel.host-port.
    ```

    - 참고로 Jaeger는 기본적으로 태그를 덮어쓴다.
      - [Jaeger Deployment](https://www.jaegertracing.io/docs/1.15/deployment/)

<총평>

- 기능 구현을 요구하는 단순한 PR인 줄 알았는데……
- Alloy 측에서 이에 대해 어떻게 생각할지가 중요한 부분인 것 같다.
- Alloy의 컨셉에서 어느 정도를 override 하고 skip할 것인지, 어느정도 정책을 정해야 하는 부분인 것 같아서 내가 임의로 기준을 정해서 구현하기가 다소 까다로워 보인다.
- 조사하다 보니 opentelemtry가 trace를 위해 span을 생성하고 OTLP에 전송하는 보다 더 고도화된 작업을하고, Alloy는 OTLP에서 이를 수신해서 트레이스 정보를 종합하고, 가공한다는 것을 알았다. 목적에 따라 각각 구분해서 사용해야 겠다.
  - 참고로 Alloy로 trace를 사용할거면 애플리케이션이 알아서 span을 생성해야 한다고 한다. alloy는 span 생성 기능을 제공하지 않는다.
    - [Grafana Alloy | Tempo Documentation](https://grafana.com/docs/tempo/latest/configuration/grafana-alloy/)

---

## 2. **loki.source.podlogs allow tail_from_end style configuration #3550**

[issue #3550](https://github.com/grafana/alloy/issues/3550)

open: 2 weeks ago (Grafana Labs 관계자 분이 최근에 열어주신 이슈)

```markdown
Request
loki.source.podlogs will always try to read all logs from the kubernetes API available for the pods identified. However, if there are long-running pods this can be a large number of logs and many of them will be outside the acceptable windows for a destination like Loki. We should allow users to configure the component to only request new logs, similar to the tail_from_end configuration in other log components.

Use case
Avoid processing various logs that will just be rejected by destination and span alloy's logs with errors.
```

<요약>

파드의 로그를 일부만 받아올 수 있는 옵션 구현 요청

- loki.source.podlogs같은 컴포넌트는 k8s 클러스터 내의 파드의 로그를 k8s api를 통해 가져온다.
  - 근데 이 과정에서 이전부터의 모든 로그를 전부 읽어오려 한다.
  - 오래된 파드일수록 이에 대한 부담이 커지기에 이를 피하기 위해 최신의 로그를 가져오는 등의 방법이 필요하다. ex) tail_from_end

<총평>

- 다른 pr들에 비해 기능 요구가 보다 더 명확하고 어느정도 레퍼런스가 있을 것으로 예상돼서 이 PR에 대해 중점적으로 구현해보려고 한다.

## TO-DO

1. Alloy log 수집 부분 확인
2. log의 최신 데이터만 수집할 수 있는 옵션을 제공하는 다른 기술 찾아보기
3. Alloy log 수집 부분에 이를 적용시킬 수 있는 방법 생각하기
4. “최신”이라 함은 어느 정도 기간을 의미하고, 얼마 만큼의 기간, 작업 등의 범위 설정 옵션을 제공할 것인지 생각하기
