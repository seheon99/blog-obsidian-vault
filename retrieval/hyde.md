---
title: HyDE
topics:
  - LLM
  - Retrieval
  - RAG
description: 가상 답변 문서를 임베딩해 검색 정확도를 높이는 RAG 변형
createdAt: 2026-05-03T14:30:00
---
**HyDE** (Hypothetical Document Embeddings)는 짧은 질의를 직접 임베딩하는 대신, **LLM이 생성한 가상의 답변 문서를 임베딩해 검색에 사용**하는 기법이다.

## 핵심 통찰

질의와 답변 문서는 **서로 다른 분포**에 존재한다. "OAuth 토큰 만료 처리 방법" 이라는 짧은 질의는 해당 답변이 담긴 실제 문서 단락과 어휘적·구조적으로 다르게 생겼다. 하지만 LLM이 생성한 가상 답변은 실제 답변 문서와 비슷한 분포를 가진다. 그래서 가상 답변을 임베딩으로 사용하면 query-document 분포 격차가 해소되고 검색 정확도가 올라간다.

## 동작

1. Query → LLM이 가상 답변 문서 생성
2. 가상 답변을 임베딩
3. 가상 답변 임베딩으로 외부 벡터 DB 검색
4. 검색된 **실제** 문서를 컨텍스트로 답변 생성

[[gen-read]]와 달리 외부 검색 단계가 유지된다. LLM은 검색 쿼리를 보강하는 역할이고, 답변의 근거는 여전히 외부 문서다.

## 강점

- **Query-document 분포 격차 해소**: 짧고 모호한 질의에서 검색 recall 향상
- **외부 grounding 유지**: 답변은 실제 검색된 문서에 기반
- **기존 RAG 인프라에 추가 가능**: 임베딩 단계 앞에 LLM 호출만 끼워넣으면 됨

## 한계

- **잘못된 가상 답변은 검색을 망친다**: hallucinate된 가상 답변이 엉뚱한 방향으로 검색을 유도할 수 있음
- **LLM 호출 비용 추가**: 매 질의마다 가상 답변 생성 비용 발생
- **Latency 증가**: 검색 전 LLM round trip이 추가됨

## 같이 보기

- [[retrieval-augmented-generation]] — HyDE는 RAG의 query 단계 강화
- [[gen-read]] — 외부 검색 없이 가상 문서만 사용하는 변형
- [[llm-reranker]] — 검색 후 단계에서 LLM이 개입하는 변형
- [[llm-as-retrieval]] — index
