# 지난 시간에…

빌드가 안 된다고 했는데

go build를 처음 써봐서 잘 몰랐는데

build 파일을 확인해보니 빌드가 되어 있었다!!

go 빌드는 완료되면 딱히 별다른 output이 없다는 걸 처음 알았다.

이후 테스트를 위해 minikube + kubectl 설치

![alt text](alloy_console-1.png)

이전에 빌드해두었던 파일 실행

`./alloy/build/alloy run ./test_app/test_tailFromEnd.alloy`

근데 생각보다 실행이 잘 안됨ㅠㅠ

```hcl
loki.source.file "test_file_logs" 
  targets = [
    { __path__ = "/tmp/alloy_test_log.log", job = "tail_from_end_test" },
  ]

  tail_from_end = false
  tail_lines    = 100
  forward_to    = [loki.write.debug.receiver]
}

// 디버그용 loki.write 컴포넌트 (콘솔에 로그를 출력)
loki.write "debug" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"
  }
}
```

pod도 정상 작동을 확인 했고, 이론 상으로는 config 파일도 잘 적었다고 생각했는데, 콘솔에 로그가 안 찍힌다..

내부 로직 수정이 덜 된 것 같아서 internal/component/loki/file/file.go 부분을 확인하니

Argument 부분에서 컴포넌트 누락을 확인!

그래서 TailFromEnd, TailLines 추가

```hcl

// Arguments holds values which are used to configure the loki.source.file
// component.
type Arguments struct {
	Targets             []discovery.Target  `alloy:"targets,attr"`
	ForwardTo           []loki.LogsReceiver `alloy:"forward_to,attr"`
	Encoding            string              `alloy:"encoding,attr,optional"`
	DecompressionConfig DecompressionConfig `alloy:"decompression,block,optional"`
	FileWatch           FileWatch           `alloy:"file_watch,block,optional"`
	TailFromEnd         bool                `alloy:"tail_from_end,attr,optional"`
	TailLines					int                 `alloy:"tail_lines,attr,optional"`
	LegacyPositionsFile string              `alloy:"legacy_positions_file,attr,optional"`
}

```

근데도 안된다.. 정상 실행은 되는데 로그가 확인되지 않고 있음

/tmp/alloy_test_log.log 파일에 로그로 생성해뒀는데 연결이 뭔가 잘못 된 것 같다.

loki 연결 부분을 좀 더 확인해 봐야 할 것 같다.

생각보다 수정해야 할 파일들이 많다.
