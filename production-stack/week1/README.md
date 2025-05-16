# 자기소개

https://github.com/falconlee236

- 이름 - 이상윤
- 소속 - 계약직 백수 (ML엔지니어 인턴중)
- 관심분야 - 모델 최적화, ML 클라우드 인프라, 모델 서빙

# vLLM - Production Stack

### 오픈소스 선정 이유

모델 최적화와 서빙을 동시에 할 수 있는 매우매우매우 유명한 프레임워크인 vLLM이라는 오픈소스의 하위 프로젝트인 production-stack에 기여할 생각입니다

> https://www.redhat.com/ko/topics/aiml/what-vllm
> 
> 
> vLLM, 즉 가상 대규모 언어 모델(Virtual Large Language Model)은 [vLLM 커뮤니티](https://github.com/vllm-project)에 의해 유지 관리되는 오픈소스 코드 라이브러리입니다. vLLM은 [대규모 언어 모델(Large Language Model, LLM)](https://www.redhat.com/ko/topics/ai/what-are-large-language-models)이 계산을 더욱 효율적이고 대규모로 수행할 수 있도록 돕습니다.
> 
> 구체적으로 설명하면 vLLM은 GPU 메모리를 더욱 효율적으로 활용하여 [생성형 AI](https://www.redhat.com/ko/topics/ai/what-is-generative-ai) 애플리케이션의 출력을 가속화하는 [추론](https://www.redhat.com/ko/topics/ai/what-is-ai-inference) 서버입니다.
> 

### Production Stack


https://github.com/vllm-project/production-stack

vLLM Production Stack은 vLLM 위에 추론 스택을 구축하는 참조 구현을 제공하는 프로젝트로, 다음과 같은 이점이 있습니다:

- 단일 vLLM 인스턴스에서 분산 vLLM 배포로 애플리케이션 코드 변경 없이 확장 가능
- 웹 대시보드를 통한 모니터링
- 요청 라우팅 및 KV 캐시 오프로딩으로 성능 향상

**주요 기능**

- Helm을 사용한 간편한 배포
- Kubernetes 환경 지원
- OpenAI API 호환 인터페이스
- 다양한 LLM 모델 지원
- 모니터링 및 메트릭 시각화

**아키텍처 구성요소**

- 서빙 엔진: 다양한 LLM 모델을 실행하는 vLLM 엔진
- 요청 라우터: 라우팅 키나 세션 ID를 기반으로 KV 캐시 재사용을 최대화하여 적절한 백엔드로 요청 전달
- 관측성 스택: Prometheus + Grafana를 통한 백엔드 메트릭 모니터링

**설치 및 배포**

- 사전 요구사항: GPU가 있는 Kubernetes 환경
- Helm 차트를 통한 간편한 배포 방법
- 설치 검증 및 쿼리 전송 방법

**로드맵**

- vLLM 특화 메트릭 기반 자동 확장
- 분리된 프리필 지원
- 라우터 개선 (비-Python 언어를 사용한 성능 향상, KV-캐시 인식 라우팅 알고리즘, 향상된 장애 허용성 등)

**커뮤니티**

- 주간 커뮤니티 미팅 (시간대 교대로 진행)
- 공식 문서, 배포 튜토리얼, 슬랙 채널 등의 리소스 제공

# 선정 이유

- 기존에 해당 프로젝트의 아키텍쳐를 테라폼을 사용해서 GCP 형태로 배포하는 과정을 튜토리얼로 올렸고 머지에 성공했습니다
    
    https://github.com/vllm-project/production-stack/pull/250
    
- 그러고나서 MS Azure 환경에서도 테라폼을 사용해서 배포하는 과정을 만들어보고 싶다고 issue에 올렸고, 긍정적인 반응을 보였습니다.
    
    https://github.com/vllm-project/production-stack/issues/271
    
- 시간이 없어서 2달동안 방치했지만 이제는 시작할 때가 된 것 같아서 선정했습니다.

# 6주간 학습 및 분석 계획

1. Azure 무료 크레딧 받기
2. 테라폼으로 간단한 Azure 서버 생성하기
3. 테라폼으로 AKS 클러스터 생성하기
4. 테라폼으로 GPU 클러스터 생성하기
5. 최종 아키텍쳐 코드 작성
6. 튜토리얼 문서 작성
