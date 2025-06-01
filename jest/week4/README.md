## What did we do this week?

- 공유 캐시 관련 코드 살펴보기
- 이슈 아이디어 찾기

## Key Takeaways / Learnings

### 1. 공유 캐시 관련

- 공유 캐시(shared cache)라는 말은 상황에 따라 다르게 쓰이지만, 일반적으로는 여러 실행 단위(예: 워커, 프로세스, 서버 인스턴스 등) 가 같은 캐시 데이터를 함께 사용할 수 있는 구조를 의미. 이때 공유 방법은 크게 `메모리 기반 공유캐시`, `디스크 기반 공유캐시`, `IPC 기반 공유캐시` 세 가지로 나눌 수 있음.

- 메모리 공유 캐시 vs 디스크 기반 공유 캐시

  | 항목                | 메모리 공유 캐시           | 디스크 기반 공유 캐시          |
  | ------------------- | -------------------------- | ------------------------------ |
  | 공유 대상           | 같은 프로세스/스레드 내    | 다른 프로세스 포함 전체 시스템 |
  | 속도                | 매우 빠름                  | 느리지만 안정적                |
  | 제약                | 멀티 프로세스 불가         | 모든 환경에서 사용 가능        |
  | Jest 사용 여부      | X                          | 사용함 (`cacheDirectory`)      |
  | 병렬 테스트 시 효과 | 없음 (각 워커 메모리 독립) | 있음 (캐시된 결과 재사용 가능) |

* Jest는 공유 캐시를 “디스크 기반”으로 구현하여, 멀티 프로세스 환경에서도 transform 결과 등을 재사용할 수 있게 하고 있음. 이는 메모리 기반 공유 캐시와는 방식이 다르며, 안정성과 범용성 측면에서 더 적합한 방식임.
* 트레이싱 요약

  - jest-worker/src/Farm.ts의 워커 생성/데이터 전달 방식 확인
  - jest-transform/src/ScriptTransformer.ts에서 transform 결과가 공유되는 방식 확인
  - jest-config에서 캐시 옵션이 ScriptTransformer에 어떻게 전달되는지 추적

* 워커 간 transform 캐시 재사용 방식

  - jest-transform의 ScriptTransformer는 내부적으로 변환된 코드의 결과를 디스크에 저장함. 이 디스크 경로는 Jest 설정(cacheDirectory)을 기준으로 모든 워커가 공통 접근 가능. 그래서 transform 캐시는 다음과 같은 방식으로 작동함.
    ```
    [워커 A]파일 A.js를 처음 transform → cacheDirectory에 저장
    [워커 B]같은 A.js를 transform하려 할 때, 캐시 존재 확인 → 그대로 사용
    ```

* `jest-transform/src/ScriptTransformer.ts`

  ```
  this.\_config.cacheDirectory // 설정값으로 공유 디렉토리 사용
  this.\_getCacheKey(fileContent, filename, instrument)
  ```

  - `_getCacheKey()`로 각 파일의 고유 캐시 키를 생성함
  - transform 결과는 이 키를 기반으로 cacheDirectory에 파일로 저장됨
  - 이후 다른 워커에서도 같은 캐시 키가 생성되면 디스크에서 읽어옴

### 2. 이슈 결정 🤩🤩🤩

> [#15574 Filter by text string](https://github.com/jestjs/jest/issues/15574) <br/> Jest CLI의 -t 옵션에서 정규식이 아닌 순수 문자열 기반 필터링을 지원해달라는 기능 요청

- 이슈 내용

  - 현재 -t 옵션은 테스트 이름을 정규식으로 필터링함
  - 테스트 이름에 (), +, \* 등 정규식 특수문자가 포함되면 일일이 escape 해야 함
  - 특히 실패한 테스트 이름을 복사해서 붙여넣을 때 불편함이 큼
  - 제안된 해결 방법
    - -t 옵션에서 text: 접두어가 붙으면 정규식이 아닌 문자열 그대로 비교<br/>
      `예: jest -t "text:should handle errors (edge cases)"`
    - 또는 새로운 옵션 -T를 도입하여 순수 문자열 비교용으로 사용<br/>
      `예: jest -T "should handle errors (edge cases)"`

- 목적

  - 테스트 이름 복사 후 바로 실행 가능하게 하여 사용성을 개선
  - 정규식 escape 부담 해소
  - 기존 -t는 그대로 유지되어 호환성 문제 없음

- 예상 구현 방향

  - jest-cli에서 --testNamePattern 처리 시 text: 접두어 조건 분기
  - 또는 -T 옵션 추가하여 별도로 처리
  - 문자열 비교는 testName === 입력값 형태로 처리 가능

## What will we do next week?

- 요청 사항 기반 실행 환경 구축
- 기능 작업

## Etc

> 📌 [Feature: Provide the ability to cut-off files from "finding more tests" (#15642)](https://github.com/jestjs/jest/issues/15642)  
> 테스트 속도 개선 관련 이슈가 재밌어보였으나 너무... 코드베이스가 방대했다... 다음을 기약...
