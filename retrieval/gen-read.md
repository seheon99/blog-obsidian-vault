---
title: GenRead
topics:
  - LLM
  - Retrieval
description: LLM이 가상 배경 문서를 생성한 뒤 그 문서를 읽어 답변하는 방식
createdAt: 2026-05-03T14:30:00
---
**GenRead** ("Generate rather than Retrieve")는 사용자 질의에 대해 LLM이 먼저 가상의 배경 문서를 생성하고, 그 문서를 컨텍스트로 사용해 답변을 생성하는 방식이다. 외부 검색을 LLM의 parametric memory로 대체한다.

## 동작

기존 RAG: query → 문서 검색 → 검색된 문서 읽기 → 답변 생성

GenRead: query → **LLM이 배경 문서 생성** → 생성된 문서 읽기 → 답변 생성

LLM이 학습 데이터에서 흡수한 지식을 압축해 단일 배경 문서로 재구성하고, 그 문서를 명시적인 reasoning context로 사용한다.

## 강점

- **외부 검색 인프라 불필요**: 벡터 DB나 임베딩 모델 없이 동작
- **Multi-hop 추론에 자연스러움**: 모델이 필요한 맥락을 능동적으로 구성. RAG처럼 "첫 청크를 읽어야 다음 질문을 알 수 있는" 구조적 제약이 없음
- **Retrieval noise 회피**: hard negative 없음

## 한계

- **Parametric memory 오류 증폭**: 잘못된 배경 문서를 생성하면 그 위에서 만든 답변도 틀린다. 외부 grounding이 없으므로 교정 수단 부재
- **출처 추적 불가**: 가상 문서는 진짜 출처가 아님
- **컨텍스트 길이 비용**: 생성된 배경 문서가 길어지면 EMNLP 2025의 "input length 자체가 성능을 저하시킴" 영향을 받음
- **Hallucination이 한 단계 더 들어감**: 답변뿐 아니라 배경 문서 자체도 hallucinate될 수 있음

## 같이 보기

- [[parametric-retrieval]] — 가상 문서를 만들지 않고 직접 응답
- [[hyde]] — 가상 문서를 검색 쿼리로 사용하는 변형 (외부 검색 유지)
- [[llm-as-retrieval]] — index
