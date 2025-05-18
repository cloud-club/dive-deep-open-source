### 오픈소스 프로젝트 선정 이유 및 개요

[JEST](https://github.com/jestjs/jest): 자바스크립트 테스팅 프레임워크

주요 특징은 다음과 같습니다.

- Zero config: Jest는 대부분의 자바스크립트 프로젝트에서 별도의 설정 없이 바로 사용할 수 있도록 설계되었습니다.
- Snapshots: 큰 객체의 상태를 쉽게 추적할 수 있는 테스트를 작성할 수 있습니다.
- Isolated: 테스트는 각각 독립된 프로세스에서 병렬로 실행되어, 성능을 극대화합니다.
- Great API: it, expect와 같은 명령어부터 시작해서, 테스트를 작성하는 데 필요한 모든 기능이 한곳에 잘 갖춰져 있습니다.

실제 프로덕션 환경에서 Jest를 테스트 자동화 도구로 자주 활용하고 있습니다. 따라서 테스트 속도는 CI/CD에서 배포 품질을 결정짓는 핵심 요소 중 하나입니다. 그런데, JEST는 빠른 속도를 보장한다는 공식문서에 비해 규모가 조금만 커져도 실제 6초~에 느린 속도를 자주 경헙합니다.

따라서 이번 오픈소스 프로젝트 분석 스터디의 목표는 다음과 같습니다.

1. 해당 도구의 내부의 동작 원리를 학습/파악하고자 합니다.
2. 테스트 실행 시간에 영향을 미치는 코드 베이스를 중심으로 동작 구조를 분석합니다.
3. 성능 병목 지점을 중심으로 오픈소스 기여 포인트를 찾습니다.

---

### 프로젝트 구조 및 주요 특징

> _Jest is designed in a way that makes memory leaks likely if you’re not actively trying to squash them. For many test suites this isn’t a problem because even if tests leak memory, the tests don’t use enough memory to actually cause a crash
> 애초에 Jest는 테스트를 한 곳에 때려넣는게 아니라면 메모리 누수가 발생할 수밖에 없는 구조로 설계되었습니다._

Jest 실행이 느린 이유에 대한 초기 분석 _(only 가설 ~ㅎㅎ)_

1. TypeScript 트랜스파일링은 제한적인 영향
   - 초기 실행 시 TypeScript → JavaScript로 변환하는 데 상당한 시간이 소요됨.
   - 그러나 이후에는 `.cache/jest`를 기반으로 캐시를 적극 활용하므로, 변경된 일부 파일만 재트랜스파일링 됨.
   - 즉, 트랜스파일링은 초기 실행에서만 병목이 되며, 지속적인 실행 속도 저하의 핵심 원인이라고 보긴 어려움.
2. 의심되는 병목 요소
   - Jest는 내부적으로 `ts-jest`, `babel-jest` 등을 통해 AST 수준의 정적 분석 및 변환을 반복 수행함.
   - 테스트 파일마다 독립된 워커 환경이 생성되며, 이 안에서 동일한 초기화 로직(트랜스파일러 로딩, 캐시 검증, 테스트 환경 구성 등)이 반복됨.
   - 이 구조는 리소스 낭비 및 GC가 즉각 동작하지 않는 상황에서 메모리 누수 발생 → 실행 지연으로 이어질 수 있음.
3. 결론
   - 다음 단계로는 `jest-runtime`, `jest-worker`, `jest-config`, `jest-haste-map` 등의 모듈을 중심으로 아래 항목들을 추적해보고자 함
     - 트랜스파일링 시점과 해당 코드 베이스
     - 캐시 키 생성 및 무효화 조건

레퍼런스

1. https://blog.bitsrc.io/why-is-my-jest-suite-so-slow-2a4859bb9ac0
2. https://blog.bitsrc.io/why-is-my-jest-suite-so-slow-2a4859bb9ac0
   https://chanind.github.io/javascript/2019/10/12/jest-tests-memory-leak.html?source=post_page-----243596864c40---------------------------------------

---

### 향후 6주간의 학습/분석 계획

| 주차 | 내용                   | 핵심 목표 요약                          |
| ---- | ---------------------- | --------------------------------------- |
| 1주  | 프로젝트 구조 분석     | 디렉토리 구조 및 주요 모듈 이해         |
| 2주  | 실행 흐름 분석         | CLI → 내부 실행 흐름 트레이싱           |
| 3주  | 테스트 시스템 이해     | `expect`, `it`, 스냅샷 테스트 구조 분석 |
| 4주  | 이슈 탐색 및 계획 수립 | 기여 가능한 이슈 선정 및 작업 계획 작성 |
| 5주  | 첫 PR 시도             | 코드 수정 및 PR 제출, 리뷰 대응         |
| 6주  | 리뷰 반영 및 회고      | 피드백 반영, 정리 문서 작성 및 마무리   |
