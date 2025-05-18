## What did we do this week?

- How to Contribute 문서 읽기
- JEST 실행 환경 구축
- 구조 분석 및 코드 트레이싱

## Key Takeaways / Learnings

1. jest는 [50개의 packages](https://github.com/jestjs/jest/tree/main/packages)로 구성됨.
2. 전체 코드 베이스 트레이싱

   - jest-cli (bin/jest.js)
     → 터미널에서 jest 명령어를 실행하면 가장 먼저 실행되는 CLI 엔트리 포인트
   - jest-core (runCLI)
     → 내부적으로 runCLI() 함수가 호출되어 전체 테스트 흐름을 시작
   - jest-runner (runTests)
     → 테스트 파일을 워커 프로세스에 분배하고 실행
   - jest-runner/testWorker.ts → runTest.ts
     → 각 테스트 파일은 독립 워커에서 실행되며, runTest()에서 실질적인 테스트 수행
   - runTest.ts 내부에서 필요한 핵심 컴포넌트들 불러옴:

     - @jest/runtime: 테스트 모듈 실행 엔진
     - @jest/transform: 코드 변환기
     - @jest/environment: 실행 환경(jsdom, node 등)
     - @jest/console: 콘솔 출력 관리

3. 주요 패키지 분석

   | 패키지         | 설명                    | 성능 최적화와 관련하여                                                       |
   | -------------- | ----------------------- | ---------------------------------------------------------------------------- |
   | jest-worker    | 워커 프로세스 풀 관리   | 현재는 실행 후 `.end()`로 종료함. 재사용 풀 도입 시 초기화 비용 줄일 수 있음 |
   | jest-runtime   | 모듈 캐시 정책 개선     | 병렬 환경에서도 일부 캐시 공유 가능성 탐색                                   |
   | jest-haste-map | 파일 시스템 스캔 병렬화 | 현재 watchman 의존, 내부 처리 병렬화 가능성 검토                             |
   | jest-transform | 캐시 무효화 로직 분석   | 작은 config 변경에도 전체 다시 트랜스파일 되는 문제                          |

4. jest-worker를 활용한 프로세스 병렬화
   : Jest는 실제로 어떻게 테스트를 병렬로 실행하는가? [참고 가이드 문서](https://cpojer.net/posts/building-a-javascript-testing-framework)

   > Node.js는 기본적으로 단일 스레드. 테스트 파일이 많아지면 → 순차 실행 시 시간이 기하급수적으로 늘어남. 자바스크립트는 비동기 처리(Promise, async/await)는 잘하지만, CPU-bound 작업을 병렬로 처리할 수 없음

   - 테스트 파일 탐색:
     jest-haste-map을 이용해 프로젝트 내 테스트 파일을 수집.

   - 병렬 분산 실행:
     jest-worker를 이용해 수집된 테스트 파일들을 작업 단위로 쪼개고,
     각 워커 프로세스에서 병렬로 실행.

   - 테스트 실행 엔진:
     각 워커는 jest-circus 또는 jest-jasmine2와 같은 러너를 통해
     독립된 테스트 환경을 구성하고 실행.

   - 결과 집계 및 리포트:
     워커들로부터 결과를 수집해 전체 테스트 리포트를 생성.

   | 개념             | 예제 구현                        | Jest 내부 구조                                      |
   | ---------------- | -------------------------------- | --------------------------------------------------- |
   | 테스트 파일 탐색 | `haste-map`으로 `*.test.js` 탐색 | `jest-haste-map`                                    |
   | 병렬 처리 분배   | `jest-worker` + `worker.js`      | `jest-worker` + TestRunnerWorker                    |
   | 테스트 실행 환경 | `worker.js`에서 파일 읽고 실행   | `jest-runtime`, `jest-config`, `jest-environment-*` |
   | CPU 활용 최적화  | `os.cpus().length` 사용          | `maxWorkers`, `watchman`                            |

## What will we do next week?

- `/jest-runtime`, `/jest-transform` 에서 캐싱 전략 분석해보기
