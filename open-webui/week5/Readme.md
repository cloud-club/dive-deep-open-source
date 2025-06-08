## 새로운 오픈소스
open web ui 를 파다가 최근 ai 에이전트를 만들면서 langchain 을 사용했을 때 gpt 모델과 달리 로컬 llm을 사용하는 경우 json parsing 및 tool calling이 제대로 안되는 이슈를 경험함

이를 해결해보는 방식으로 기여해볼 수 있지 않을까 하는 생각에 langchain을 보니 역시 해당 이슈들이 제법 많았고 이들 중 선택을 해보고자 하였다


## 주된 이슈

주된 이슈 중 하나가 pydantic문제 gpt 와 다른 로컬 모델간 동일 코드이지만 다른 결과 이슈가 주로 있었던 것을 확인함

## 이슈 내역

llm.with_structured_output(json_schema, method="json_schema") returns an empty JSON object.

https://github.com/langchain-ai/langchain/issues/31423 



Issue with Ollama Function – Only Agent Response, No Tool Calls

https://github.com/langchain-ai/langchain/issues/28110


DOC: PydanticDeprecatedSince20: The schema method is deprecated; use model_json_schema instead.

https://github.com/langchain-ai/langchain/issues/31458

## 시도 내역
위에 docs 를 수정하려고 했지만 알고보니 다른 이슈...

