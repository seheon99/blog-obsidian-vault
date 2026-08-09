---
title: LLM as Retrieval
type: write-up
topics:
  - LLM
  - RAG
  - AI
  - Information Retrieval
description: 검색 시스템에서 비결정성을 어디에 둘 것인가
createdAt: 2026-05-03T14:00:00
---
RAG에서 Retrieval은 모델이 답을 만들기 위해 필요한 정보를 어디에서 가져올지 결정하는 문제다. 외부 벡터 저장소에서 가져올 수도 있고, 모델의 가중치 자체에서 가져올 수도 있고, 그 외 결정론적 도구를 호출해 가져올 수도 있다. 세 가지 접근의 본질적 차이는 **비결정성을 어디에 두는가**에 있다.

## 비결정성의 위치
다음 세 가지 아키텍처는 서로 다른 위치에 비결정성을 가져다놓는다.
### Vector-space retrieval
가장 보편적으로 사용되는 방법이다. 모든 문서를 나누어 한 단위를 대표하는 의미 벡터를 만들고, 검색 시 쿼리의 벡터와 가장 유사한 벡터를 검색해 연관된 문서를 가져오는 방식이다. 임베딩 모델이 의미론적 유사성을 계산하고 상위 문서를 언어 모델에게 전달해 답을 만든다. 검색의 품질은 임베딩 모델이 생성한 벡터가 문서를 얼마나 잘 대표하는지, 문서가 얼마나 잘 구분되어 있는지 등으로 나눌 수 있다.

### Agentic retrieval
에이전트 모델이 grep, glob, file read, SQL 같은 도구를 선택해 호출하는 방식이다. 도구를 사용할 때마다 결과는 정해져있지만, 도구 선택을 에이전트 모델이 선택해야 하므로, 검색의 품질이 에이전트 모델 성능에 달렸다.

### Parametric retrieval
모델이 학습 시점에 흡수한 지식을 직접 출력하는 방식이다. 데이터베이스나 도구 등 외부 의존성 없이 실행될 수 있는 가장 가벼운 형태이지만, 학습 시점 이후 발생한 지식에는 접근할 수 없고, 할루시네이션 문제도 있어 사용하기 까다로운 방법이다.

## 비용
세 가지 방법은 서로 다른 비용 모델을 갖고있다. Vector-space retrieval 은 검색 전 인덱싱하고 임베딩을 생성하는 등 비용이 추가로 필요한 반면 agentic retrieval 과 parametric retrieval 은 별도의 비용이 필요하지 않다.

하지만 검색 당 비용은 vector-space retrieval 은 벡터 검색으로 빠르게 이루어지는 반면, agentic retrieval과 parametric retrieval 은 추가로 언어 모델을 호출해야 하기 때문에 검색 당 비용이 추가로 발생한다. 특히 agentic retrieval 은 도구 호출 마다 추가로 언어 모델을 호출해야 하기 때문에 적지 않은 비용이 발생한다.

> [!note]
> Reasoning 모델(o1, Claude 3.7 이후) 이후 parametric retrieval의 신뢰도가 이전에 비해 높아졌지만, 추론 토큰이 적지 않게 필요해 per-query 비용도 증가했다.

> [!NOTE]
> 대부분의 시스템에서는 위 아키텍처를 섞어서 사용한다. 코드 어시스턴트로 확장 중인 Warp는 주 검색은 agentic retrieval 을 사용하고, 의미 검색이 필요할 때만 하나의 도구로 제공된 vector-space retrieval 을 사용한다.