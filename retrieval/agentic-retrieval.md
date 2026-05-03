---
title: Agentic Retrieval
topics:
  - LLM
  - Retrieval
  - Agent
description: LLM이 결정론적 도구를 runtime에 선택해 호출하는 방식의 검색
createdAt: 2026-05-03T14:30:00
---
**Agentic Retrieval**은 LLM이 결정론적 도구(grep, glob, file read, SQL 등)를 runtime에 선택해 호출하는 방식의 검색이다. 비결정성은 **tool 선택**에 위치하고, 각 tool 호출 자체는 재현 가능하다.

## 동작

1. LLM이 사용 가능한 tool 집합과 그 사용법을 받음
2. 질의의 성격에 따라 어떤 tool을 어떤 인자로 호출할지 결정
3. Tool은 deterministic (예: `grep "useAuthContext" --type rust`)
4. 결과를 받아 다음 행동을 결정하거나 답변 생성
5. 필요 시 N+1 round trip으로 반복적 정제

## 강점

- **재현 가능성**: 사용자가 동일한 tool 호출을 직접 재현해 결과를 검증할 수 있음. RAG의 silent retrieval miss와 대비됨
- **Zero indexing cost**: 임베딩 생성, 벡터 저장, 인덱스 갱신 부담 없음
- **운영 부담 낮음**: 임베딩 모델 호스팅, reranker 튜닝, 청크 크기 실험 불필요
- **식별자가 의미인 도메인에 강력**: 코드(`useAuthContext`)처럼 토큰 자체가 의미를 인코딩하는 도메인에서 grep이 embedding을 자주 이긴다
- **디버깅 용이**: 어떤 tool이 어떤 인자로 무엇을 반환했는지 trace로 명확

## 한계

- **Per-query 비용 높음**: 매 round trip마다 LLM 호출 발생
- **Paraphrase 도메인에 약함**: 법률, 의료처럼 의미가 어휘 일치보다 paraphrase에 있는 경우 lexical tool이 부족
- **Latency 증가**: 직렬 round trip이 누적됨
- **Tool 집합 설계 의존**: 좋은 tool이 없으면 모델이 우회를 시도하다 실패함

## 실제 사례

Warp는 코드 검색에서 이 패턴을 명시적으로 채택했다. `SearchCodebase` 한 곳만 RAG이고, `Grep`, `FileGlob`, `ReadFiles`, `ReadDocuments` 등 다른 모든 검색은 agentic — LLM이 결정론적 tool을 직접 호출한다. 코드 식별자가 토큰 자체에 의미를 담기 때문에 embedding 검색보다 grep이 더 정확한 케이스다.

## 같이 보기

- [[retrieval-augmented-generation]] — 대안적 검색 모달리티
- [[agentic-ai-stack]] — 에이전트 시스템의 일반적 맥락
- [[self-rag]] — agentic retrieval의 좁은 변형
- [[llm-as-retrieval]] — index
