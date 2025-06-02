# Podlog 개선 시도

지난 주차에 찾아둔 PR에 적합한 솔루션을 구현하기 위해 주요 컴포넌트 파일을 확인했다.

[Issue #3550](https://github.com/grafana/alloy/issues/3550)

## 컴포넌트

1. loki.source.podlogs
    1. path : \internal\component\loki\source\podlogs\

### **1. 컴포넌트 설정 확장 (Arguments 구조체)**

```go
// internal/component/loki/source/podlogs/podlogs.go
type Arguments struct {
    ForwardTo    []storage.LogsReceiver `alloy:"forward_to,attr"`
    TailFromEnd  bool                   `alloy:"tail_from_end,attr,optional"` // 새로운 옵션
    TailLines int `alloy:"tail_lines,attr,optional"`
    // 기존 필드 유지
}

```

- TailFromEnd 옵션을 추가하여 최신 로그를 가져올 수 있는 선택지를 제공
- TailLines : tail_from_end=true 시에 읽을 최신 로그 라인 수 (default는 1000줄로 해두었음)
  - 왜 1000줄?
    - kubernetes api의 기본 taillines 값이 1000줄이어서

path : \internal\component\loki\source\kubernetes\kubetail\kubetail.go

### 2. **kubetail.Options에 옵션 추가**

```go
type Options struct {
    // ...
    TailFromEnd bool // 추가
    TailLines int // 추가
}

```

- loki.source.podlogs는 내부적으로 kubetail 라이브러리를 이용하여 k8s api와 상호작용함
  - 따라서 options 구조체를 통해 k8s에 옵션을 전파

### 3. **tail() 함수에서 조건 분기 추가**

아래와 같이 **PodLogOptions 생성 직전에 조건 분기** 추가

```go
var podLogOpts *corev1.PodLogOptions

if t.opts.TailFromEnd {
// 최신 로그 N줄만 가져오고 이후부터 follow
    tailLines := int64(t.opts.TailLines)
    podLogOpts = &corev1.PodLogOptions{
        Follow:     true,
        Container:  containerName,
        Timestamps: true,
        TailLines:  &tailLines,
        // SinceTime은 nil로 둠
    }
} else {
    podLogOpts = &corev1.PodLogOptions{
        Follow:     true,
        Container:  containerName,
        SinceTime:  offsetTime,
        Timestamps: true,
    }
}

req := t.opts.Client.CoreV1().Pods(key.Namespace).GetLogs(key.Name, podLogOpts)

```

- TailFromEnd이 true일 때만 TailLines 사용 (TailLines와 SinceTime은 상호 배타적임)
- false면 기존 동작(SinceTime) 그대로 유지

### **4. Manager와 상위 코드에서 옵션 전달**

이미 Manager가 Options를 tailerTask에 넘기고 있으니

상위 컴포넌트(예: podlogs.go)에서 Options를 만들 때

**`TailFromEnd: args.TailFromEnd`**로 값을 넣어주면 됨

```markdown
managerOpts := &kubetail.Options{
    Client:      clientSet,
    Handler:     entryHandler,
    Positions:   positions,
    TailFromEnd: args.TailFromEnd, // 추가
    TailLines: args.TailLines, // 추가
}

```

### **5. SinceTime, SinceSeconds 검증 로직**

```markdown
// podlogs.go
func (a *Arguments) Validate() error {
    if a.TailFromEnd && (a.SinceSeconds != nil || a.SinceTime != nil) {
        return fmt.Errorf("tail_from_end cannot be used with since_seconds or since_time")
    }
    return nil
}

```

- 잘못된 설정을 예방하기 위함
- tail_from_end와 since_seconds가 동시에 사용되지 않도록 함

### 6. default 값 추가 & 유효성 검증 추가

```markdown
// DefaultArguments holds default settings for loki.source.kubernetes.
var DefaultArguments = Arguments{
    Client: commonk8s.DefaultClientArguments,
    TailLines: 1000,  //추가
}

// SetToDefault implements syntax.Defaulter.
func (a *Arguments) Validate() error {
    if a.TailFromEnd {
        if a.TailLines < 1 {
            return fmt.Errorf("tail_lines must be ≥1 when tail_from_end=true")
        }
    } else if a.TailLines != 0 {
        return fmt.Errorf("tail_lines can only be set when tail_from_end=true")
    }
    return nil
}
```

- tail_from_end가 true일 때, tail_lines를 1줄 이상 사용하도록 유효성 검증

### 7. 포지션 업데이트 로직 수정

```go
select {
case <-ctx.Done():
    return nil
case ch <- entry:
    // TailFromEnd=false일 때만 포지션 업데이트
    if !t.opts.TailFromEnd {
        t.opts.Positions.Put(positionsEnt.Path, positionsEnt.Labels, entryTimestamp.UnixMicro())
    }
    t.target.Report(entryTimestamp, nil)
}

```

```go
// internal/component/loki/source/podlogs/podlogs.go
func (c *Component) updateTailer(args Arguments) error {
    // ... 기존 코드

    // TailFromEnd 활성화 시 기존 포지션 삭제
    if args.TailFromEnd {
        for _, target := range c.tailer.Targets() {
            entry := entryForTarget(target)
            c.positions.Remove(entry.Path, entry.Labels)
        }
    }

    _ = c.tailer.UpdateOptions(context.Background(), managerOpts)
    return nil
}
```

- tail_from_end=true 시에는 최신 로그만 수집하므로 포지션 추적이 불필요함
  - k8s api에서 tail 시에 포지션 파일을 사용하지 않는 것에 착안하였음

## 테스트

실제로 생성한 옵션이 잘 작동하는지 테스트하기 위한 환경 세팅

### 1. kind 클러스터 생성

```markdown
# mac
brew install kind && kind create cluster
```

하고 수정한 alloy를 빌드하는 도중 **디스크 부족** 에러 발생..

k8s 파드 생성하고 alloy로 로그 수집하여 확인 필요

config.alloy

```go
loki.source.podlogs "test" {
  tail_from_end = true
  selector = "run=logtest"
  forward_to = [loki.write.local.receiver]
}
```

이런식으로 tail_from_end 옵션을 true로 한 것과, 비활성화 한 config를 사용하여 비교해봐야 함

## TODO

1. 빌드 중 디스크 부족 해결 방안 생각 → 클라우드 or 집에 있는 홈서버
2. 수정사항 테스트
