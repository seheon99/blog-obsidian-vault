---
title: LLM as Retrieval
type: write-up
topics:
  - LLM
  - RAG
  - AI
  - Information Retrieval
description: 검색 시스템에서 비결정성을 어디에 둘 것인가 — RAG, agentic retrieval, parametric retrieval의 세 축
createdAt: 2026-05-03T14:00:00
---
검색은 모델이 답을 만들기 위해 필요한 정보를 어디에서 가져올지 결정하는 문제다. [[retrieval-augmented-generation|RAG]]는 외부 벡터 저장소에서 가져왔다. 하지만 이것이 유일한 답은 아니다. 모델 가중치 자체에서 가져올 수도 있고, 결정론적 도구를 호출해 가져올 수도 있다.

세 가지 접근의 본질적 차이는 **비결정성을 어디에 두는가**에 있다. 비결정성의 위치가 비용의 모양, 디버깅 가능성, 그리고 어떤 도메인에 적합한가를 결정한다.

## 비결정성의 위치

세 가지 아키텍처가 있고, 각각 다른 위치에 비결정성을 둔다.

### Vector-space retrieval (RAG)

비결정성이 **벡터 공간**에 있다. 임베딩 모델이 semantic similarity를 계산하고, ANN 인덱스가 top-k를 가져온다. 어떤 문서가 검색되고 어떤 문서가 누락되는지는 임베딩 분포에 의해 암묵적으로 결정된다.

- 디버깅이 어렵다 — silent retrieval miss는 vector-space 도구 없이는 보이지 않음
- Paraphrase가 의미인 도메인(법률, 의료, 자연어 문서)에 강함
- 자세한 내용: [[retrieval-augmented-generation]]

### Agentic retrieval

비결정성이 **tool 선택**에 있다. LLM이 grep, glob, file read, SQL 같은 결정론적 도구를 runtime에 선택해 호출한다. Tool 호출 자체는 결정론적이고 재현 가능하다.

- 사용자가 동일한 호출을 직접 재현해 검증 가능
- 식별자가 의미를 인코딩하는 도메인(코드)에서 강력
- 자세한 내용: [[agentic-retrieval]]

### Parametric retrieval

비결정성이 **모델 가중치**에 있다. 외부 검색 자체가 없고, 모델이 학습 시점에 흡수한 지식을 직접 출력한다.

- Reasoning 중심 작업에 강함, infrastructure 부담 zero
- Knowledge cutoff, 출처 추적 불가, confabulation 위험
- 자세한 내용: [[parametric-retrieval]]

## 비용의 모양

세 접근의 비용 곡선은 **모양 자체가 다르다.**

|접근|Upfront 비용|Per-query 비용|
|---|---|---|
|RAG|높음 (인덱싱, 임베딩 생성, 인프라)|낮음 (벡터 검색)|
|Agentic|0|높음 (round trip마다 LLM 호출)|
|Parametric|0|중간 (단일 LLM 호출)|

손익분기점은 대략 `query frequency × corpus stability`로 결정된다.

- **Frequent queries on a stable corpus**: RAG가 이긴다. Upfront 비용을 많은 질의에 분산시킬 수 있고, 인덱스 갱신 비용이 낮다
- **One-shot or low-frequency tasks**: Agentic이 이긴다. 인덱싱 비용을 회수할 시간이 없음
- **Reasoning-heavy, knowledge-static**: Parametric이 이긴다. 검색 단계 자체가 불필요

> [!note]
> Reasoning 모델(o1, Claude 3.7 이후) 시대에 들어 parametric retrieval의 신뢰도가 올라갔지만, 추론 토큰이 폭증하면서 per-query 비용도 동시에 증가했다. "단일 LLM 호출"이라는 가정이 점점 깨지고 있다.

## 결정 변수

비용 외에 세 가지 축이 더 작동한다.

**Paraphrase intensity** — 의미가 어휘에 있는가, paraphrase에 있는가?

- 코드 식별자(`useAuthContext`)는 토큰 자체가 의미. → Agentic + grep
- 법률·의료 문서는 paraphrase가 substance. → RAG (LLM 기반 임베딩과 reranker로 강화)

**Citation as system requirement** — 출처가 시스템 요구사항인가?

- 법률, 의료, 규제 금융처럼 인용이 *법적 요구사항*인 경우 → RAG (외부 grounding 필수)
- 일반 Q&A, 기술 설명 → 출처 선택사항, parametric 또는 agentic도 충분

**Determinism / debuggability** — 결과가 재현 가능해야 하는가?

- 운영 중 디버깅이 중요한 시스템(코드 어시스턴트, 보안 분석) → Agentic 우위
- 콘텐츠 추천처럼 결과의 미세한 흔들림이 허용되는 경우 → RAG도 충분

## 하이브리드가 현실

순수 아키텍처는 production에서 드물다. 실제 시스템 대부분이 hybrid다.

**Warp** — 코드 어시스턴트로 진화 중인 터미널이다. 주 검색 패턴은 [[agentic-retrieval]]이고, RAG는 `SearchCodebase`라는 *하나의 tool*로만 노출된다. Grep, FileGlob, ReadFiles 같은 결정론적 tool이 기본이고, semantic search가 필요한 경우에만 RAG를 호출한다. 인덱싱 부담을 매번 지불하지 않으면서, 필요한 곳에서만 paraphrase 검색을 쓴다.

**Deep research 시스템** — 반대 패턴이다. RAG로 초기 seed를 가져와 폭을 확보하고, agentic loop으로 깊이를 확장한다.

**Compliance / 법률 RAG** — RAG가 핵심이지만, 그 위에 [[hyde]]로 query를 보강하고 [[llm-reranker]]로 결과를 정제한다.

설계 질문은 "RAG인가 agentic인가?" 가 아니라 **"질의 유형마다 어떤 retrieval modality가 필요하고, 누가 runtime에 그걸 결정하는가?"** 다.

## LLM이 RAG에 통합되는 방법

LLM이 RAG를 *대체*하는 길만 있는 게 아니다. RAG 파이프라인 안에 LLM이 들어가 검색 품질을 높이는 방법들도 빠르게 발전하고 있다. 각각이 서로 다른 위치에서 LLM을 끼워넣는다.

- [[hyde]] — 검색 *전* 단계, query를 LLM이 보강
- [[self-rag]] — 검색 *결정* 자체를 LLM이 동적으로 판단
- [[llm-reranker]] — 검색 *후* 단계, 결과를 LLM이 정제
- [[llm-embedding-models]] — 검색의 *기반*인 임베딩 모델 자체가 LLM
- [[gen-read]] — 외부 검색 자리에 LLM의 가상 문서 생성을 끼움

## Forward outlook

Context window가 자라고 reasoning이 깊어지면 두 곡선 모두 [[agentic-retrieval]] 쪽으로 휘어진다. RAG의 가치는 corpus를 k개 청크로 압축하는 데 있는데, 모델이 1M 토큰을 들고 50턴을 추론하면 그 압축의 가치가 줄어든다.

Sutton의 "bitter lesson"이 부분적으로 적용된다. 청크 크기 튜닝, reranker 선택, 임베딩 모델 fine-tuning은 expert-designed structure이고, 모델이 retrieval을 내재화하면 sunk cost가 된다.

RAG가 살아남는 위치는 두 곳으로 좁혀진다.

- 인용이 시스템 요구사항인 도메인 (법률, 의료, 규제 금융) — 검색이 명시적·추적 가능한 단계여야 함
- Context window를 초과하는 corpus (수십 GB+) — 압축이 본질적으로 필요
