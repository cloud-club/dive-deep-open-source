## What did we do this week?

- PR 검토

## Key Takeaways / Learnings

### 1. PR 올리기

1. `CHANGELOG.md` 수정 사항 반영하기
2. `e2e/literal-pattern/__tests__/literal_pattern.test.js` e2e 테스트 코드 작성하기
3. 변경된 코드 반영

- <img width="1187" alt="Image" src="https://github.com/user-attachments/assets/7565d21d-c471-4138-b8be-43f2647bbded" />

### 2. 레퍼런스 검토

- `--retries=N` : 테스트가 실패했을 때 최대 N회까지 자동으로 재시도함.

  > Mocha는 --retries, Playwright는 retries 설정을 통해 flaky test 재시도를 기본 지원함. Vitest도 test.retry 방식으로 지원하고 있음.

- `--tag=xxx` : 특정 태그가 붙은 테스트만 선택적으로 실행함.

  > Vitest는 테스트 함수에 { tags: [...] } 형태로 태그를 붙이고 CLI에서 --tag로 필터링 가능. Jasmine과 Mocha에서도 커뮤니티 플러그인을 통해 유사한 기능을 구현할 수 있음.

- `--shard=1/3` : 전체 테스트 파일을 N개로 분할하여 그 중 일부만 실행함.
  > Playwright는 --shard 옵션으로 테스트 분산 실행을 기본 제공하며, CircleCI 등과 통합 시 테스트 시간을 줄이는 데 매우 효과적임. Vitest도 테스트 파일 병렬 분산 로직을 config에서 제어할 수 있음.

## What will we do next week?

- 레퍼런스 재검토하고 결정하기
- 코드 수정해서 PR 올리기
