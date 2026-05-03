---
title: Self-RAG
topics:
  - LLM
  - Retrieval
  - RAG
description: LLM이 검색 필요성과 결과 품질을 reflection token으로 자체 평가하는 RAG 변형
createdAt: 2026-05-03T14:30:00
---
**Self-RAG** (Asai et al., 2023)는 LLM이 reflection token을 통해 검색의 필요성, 검색 결과의 관련성, 답변의 지지도를 **스스로 평가**하도록 학습된 RAG 변형이다.

## 동작

기존 RAG는 모든 질의에 대해 항상 검색을 수행한다. "2 + 2는?" 에 벡터 검색이 필요하지 않은 경우에도 마찬가지다. Self-RAG는 모델이 네 가지 reflection token을 출력해 이 결정을 동적으로 내린다.

- `[Retrieve]` — 검색이 필요한가? (yes/no)
- `[IsRel]` — 검색된 문서가 관련 있는가?
- `[IsSup]` — 검색된 문서가 답변을 뒷받침하는가?
- `[IsUse]` — 답변이 유용한가?

이를 통해 잘 아는 주제는 [[parametric-retrieval]]로, 모르는 주제는 외부 검색으로 동적으로 전환한다.

## 강점

- **불필요한 검색 회피**: 단순 질의에 검색 비용 발생하지 않음
- **Negative rejection 완화**: 검색 결과의 관련성을 자체 평가하므로 노이즈로부터 답변을 만들어내는 경향이 줄어듦
- **RAG와 parametric retrieval의 동적 선택**: 양자택일이 아닌 질의별 결정

[[agentic-ai-stack]]의 Agent Loop(Observe-Plan-Act-Reflect)와 자연스럽게 연결된다 — Reflect 단계에서 검색 결과를 평가하고 필요 시 다시 검색하거나 parametric memory에 의존하는 판단이 이 reflection token의 일반화된 형태다.

## 한계

- **Reflection token 자체가 hallucinate될 수 있음**: 모델이 자신의 무지를 모르는 calibration 문제는 그대로 남는다. `[Retrieve=No]` 가 틀릴 수 있다
- **학습 데이터 의존**: reflection token에 supervised signal이 필요해 표준 RAG보다 학습 복잡도 높음
- **검색 결정이 단일 단계**: 여러 차례 반복적 검색이 필요한 multi-hop 질의에는 [[agentic-retrieval]] 같은 더 일반화된 접근이 더 적합

## 같이 보기

- [[retrieval-augmented-generation]]
- [[agentic-retrieval]] — Self-RAG를 더 일반화한 패턴
- [[agentic-ai-stack]]
- [[llm-as-retrieval]] — index
