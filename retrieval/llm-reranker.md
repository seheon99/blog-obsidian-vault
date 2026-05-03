---
title: LLM Reranker
topics:
  - LLM
  - Retrieval
  - RAG
description: 검색된 후보 문서를 LLM이 재정렬하는 RAG 강화 기법
createdAt: 2026-05-03T14:30:00
---
**LLM Reranker**는 검색된 top-k 후보 문서를 LLM이 재정렬하는 방식이다. RAG 파이프라인의 retrieval 단계와 generation 단계 **사이**에 위치한다.

## 위치

Retrieval은 **recall**을 위해 후보를 넓게 가져온다 (top-50, top-100). Reranker는 **precision**을 위해 그중 진짜 관련 있는 top-N (보통 5-10)을 골라낸다. 기존 BERT 기반 cross-encoder를 대체하거나 보완한다.

## 동작

두 가지 접근이 있다.

- **Pointwise**: LLM이 각 (query, document) 쌍에 독립적으로 관련도 점수를 부여한다. 병렬화 가능. 정확도는 listwise보다 낮은 편
- **Listwise**: LLM이 전체 후보 목록을 받아 한 번에 재배열한다. 더 정확하지만 컨텍스트 윈도우 제약 안에서만 동작

## 강점

- **추론 능력 활용**: 단순 임베딩 유사도가 아닌 의미적 관련성 판단
- **Retrieval noise 완화**: hard negative를 효과적으로 걸러냄
- **기존 RAG에 추가 가능**: retrieval 단계와 generation 단계 사이에 끼워넣으면 됨

## 한계

- **Latency 증가**: 매 질의마다 추가 LLM 호출
- **비용**: top-k가 클수록 호출 횟수 또는 입력 토큰 증가
- **Listwise는 컨텍스트 윈도우 제약**: 후보 문서가 많으면 한 번에 못 봄

## 같이 보기

- [[retrieval-augmented-generation]]
- [[hyde]] — 검색 *전* 단계에서 LLM이 개입하는 변형
- [[llm-embedding-models]] — 검색 단계 자체에 LLM이 들어간 형태
- [[llm-as-retrieval]] — index
