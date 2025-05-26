## What did we do this week?

- /jest-runtime, /jest-transform 에서 캐싱 전략 분석해보기

## Key Takeaways / Learnings

1. `/jest-runtime`

- 테스트 모듈을 실행, 캐시 관리 담당. 모듈 캐시를 관리하여 이미 처리된 모듈을 다시 로드하지 않도록 하는 방식.
- 모듈 캐싱은 동일한 모듈을 여러 번 변환하고 로드하는 것을 방지하여 테스트 실행 속도를 높이는 데 중요한 역할.

```ts
class Runtime {
  private readonly _cacheFS: Map<string, string>;
  private readonly _cacheFSBuffer = new Map<string, Buffer>();

  constructor() {
    this._cacheFS = new Map(); // 파일을 문자열로 캐시
    this._cacheFSBuffer = new Map(); // 파일을 버퍼로 캐시
  }

  // 파일을 읽을 때 캐시를 먼저 확인
  private readFileBuffer(filename: string): Buffer {
    let source = this._cacheFSBuffer.get(filename);
    if (!source) {
      source = fs.readFileSync(filename); // 파일을 캐시가 없으면 읽기
      this._cacheFSBuffer.set(filename, source);
    }
    return source;
  }

  // 변환된 파일을 캐시로 저장
  private readFile(filename: string): string {
    let source = this._cacheFS.get(filename);
    if (!source) {
      const buffer = this.readFileBuffer(filename);
      source = buffer.toString("utf8");
      this._cacheFS.set(filename, source);
    }
    return source;
  }
}
```

2. `jest-transform`

- 파일을 실행하기 전에 변환
- Jest는 코드 변경을 빠르게 감지하고, 파일을 재변환할 때 캐시 무효화 로직을 고려해야 함.(작은 설정 변경에도 전체 변환이 다시 이루어지는 위험 존재)
- 캐시 무효화 관련 코드

```ts
class ScriptTransformer {
  private _fileTransforms: Map<string, RuntimeTransformResult>;

  constructor() {
    this._fileTransforms = new Map();
  }

  transform(filename: string, options: TransformationOptions) {
    // 변환된 파일이 캐시에 있는지 확인
    const cachedTransform = this._fileTransforms.get(filename);
    if (cachedTransform) {
      return cachedTransform.code;
    }

    const transformedCode = this.transformFile(filename, options);
    this._fileTransforms.set(filename, transformedCode);
    return transformedCode.code;
  }

  private transformFile(filename: string, options: TransformationOptions) {
    // 파일 변환 처리
    const source = this.readFile(filename);
    const transformedCode = this.applyTransformation(source, options);
    return transformedCode;
  }
}
```

3. 캐시 단계

- 파일 시스템 캐시 (jest-haste-map): 테스트할 파일을 찾는 첫 번째 단계에서 사용
- 모듈 캐시 (jest-runtime): 파일을 찾은 후 모듈을 로드하는 단계에서 사용
- 변환 캐시 (jest-transform): 모듈이 로드되면 코드 변환이 필요한 경우가 많음. 이때 코드 변환 결과를 캐시하여, 변환을 반복하지 않게 함.
- 모의(Mock) 캐시: Jest는 테스트 중에 사용하는 모의 객체나 함수를 캐시. 이것은 모의 객체가 여러 번 사용될 때 성능을 최적화하는 데 사용됨.
- 워커 캐시 (jest-worker): 여러 테스트를 병렬로 실행할 때, 워커 간에 데이터를 공유하는 캐시를 사용하여 병렬 처리 성능을 최적화 함.

4. 공유 캐시

- Jest에서 공유 캐시는 병렬 테스트 환경에서 각 워커가 동일한 데이터를 접근할 수 있도록 하는 메커니즘.
- 예를 들어, 여러 워커 프로세스가 동일한 모듈을 캐시하고 이를 재사용하는 방식.

## What will we do next week?

- 공유 캐시 관련 코드 살펴보기
- 이슈 아이디어 찾기
