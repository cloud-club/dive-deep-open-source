## What did we do this week?

- 요청 사항 기반 실행 환경 구축
- 기능 작업

## Key Takeaways / Learnings

### 1. 이슈 내용

> [#15574 Filter by text string](https://github.com/jestjs/jest/issues/15574) <br/> Jest CLI의 -t 옵션에서 정규식이 아닌 순수 문자열 기반 필터링을 지원해달라는 기능 요청

- 현재 -t 옵션은 테스트 이름을 정규식으로 필터링함. 따라서, 테스트 이름에 (), +, \* 등 정규식 특수문자가 포함되면 일일이 escape 해야 함
- 예상 구현 방법

  - -t 옵션에서 text: 접두어가 붙으면 정규식이 아닌 문자열 그대로 비교<br/>
    `예: jest -t "text:should handle errors (edge cases)"`
  - 또는 새로운 옵션 -T를 도입하여 순수 문자열 비교용으로 사용<br/>
    `예: jest -T "should handle errors (edge cases)"`

### 2. 구현

> 새로운 옵션 -T를 추가하려 했지만! 아래 플로우 상 기존 옵션 -t에 정규식 처리 여부만 분기해도 될 것 같아서, 문자열 리터럴을 나타내는 프리픽스를 처리하는 방식으로 결정!

1.  기존 `-t` 옵션에서 description 수정 `packages/jest-cli/src/args.ts`

    ```ts
    testNamePattern: {
      alias: 't',
      description:
        'Run only tests with a name that matches the regex pattern. ' +
        'Prefix with "text:" to match literal string instead of regex.',
      requiresArg: true,
      type: 'string',
    },
    ```

2.  기존 정규식 처리 코드 확인 `packages/jest-circus/src/run.ts`

- testNamePattern이 설정돼 있으면 해당 테스트의 이름 (getTestID(test))이 정규식에 매칭되지 않으면 이 테스트는 건너뜀

  ```ts
  const isSkipped =
    parentSkipped ||
    test.mode === "skip" ||
    (hasFocusedTests && test.mode === undefined) ||
    (testNamePattern && !testNamePattern.test(getTestID(test)));
  ```

3.  testNamePattern이 RegExp로 초기화되는 부분 확인 `packages/jest-circus/src/eventHandler.ts`

    ```ts
    if (event.testNamePattern) {
      state.testNamePattern = new RegExp(event.testNamePattern, "i");
    }
    ```

4.  escape하는 `text:` 접두어 분기 추가

- CLI 옵션은 globalConfig.testNamePattern에 들어감. `jest-cli/src/args.ts, runCLI.ts`
- 그리고 내부적으로 TestRunner가 실행될 때 `event.testNamePattern`의 형태로 전달됨. 이게 `eventHandler.ts`에서 `"setup"` 케이스임.
- `state.testNamePattern = new RegExp(event.testNamePattern, 'i');` CLI에서 입력한 `-t` 값은 `event.testNamePattern`으로 들어오는 거고, 여기서 RegExp(...)로 변환됨.

  ```ts
  if (event.testNamePattern) {
    if (event.testNamePattern.startsWith("text:")) {
      const raw = event.testNamePattern.slice(5);
      state.testNamePattern = new RegExp(escapeStrForRegex(raw), "i");
    } else {
      state.testNamePattern = new RegExp(event.testNamePattern, "i");
    }
  }
  ```

5. 로컬 테스트 완료!

- <img width="769" alt="Image" src="https://github.com/user-attachments/assets/f43cba11-373c-4432-bcab-030f9905be65" />

## What will we do next week?

- Jest(e2e 디렉토리) 통합 테스트 확인하기
- PR 올리기!
