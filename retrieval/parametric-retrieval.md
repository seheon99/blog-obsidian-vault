---
title: Parametric Retrieval
topics:
  - LLM
  - Retrieval
description: 외부 검색 없이 LLM의 가중치 자체를 지식 베이스로 사용하는 방식
createdAt: 2026-05-03T14:30:00
---
**Parametric Retrieval**은 외부 검색 시스템 없이 **LLM의 가중치(parametric memory)를 직접 활용해 답변을 생성**하는 방식이다. 사용자가 ChatGPT에 질문을 던지고 답을 받는 가장 단순한 흐름이 여기에 해당한다.

## 동작

- 질의 → LLM이 자신의 학습 데이터에서 압축된 지식을 사용해 직접 응답
- 벡터 DB, 임베딩 모델, 검색 인프라 일체 없음
- 모델의 parametric memory가 유일한 지식 출처

## 강점

- **Zero infrastructure**: 인덱싱, 임베딩 모델 운영, 벡터 저장소 불필요
- **Retrieval noise 부재**: 검색된 문서가 없으므로 hard negative나 lost-in-the-middle 문제가 발생하지 않음
- **추론 중심 작업에 강함**: 수학, 논리, 일반 개념 설명, 코드 패턴 같이 *지식 검색이 아닌 추론*이 핵심인 작업
- **Latency 낮음**: 검색 round trip 없음

최근 reasoning 모델(o1, Claude 3.7 이후)에서 이 방식의 신뢰도가 크게 올라갔다.

## 한계

- **Knowledge cutoff**: 학습 시점 이후 정보 부재
- **비공개 데이터 사용 불가**: 사내 문서, 고객 데이터 등은 모델에 존재하지 않음
- **출처 추적 불가**: 답변의 근거를 명시할 수 없음 — 법률, 의료, 규제 도메인에서 사용 불가
- **Confabulation**: 모르는 사실에 대해 그럴듯한 거짓을 자신감 있게 생성
- **Calibration 실패**: 모델이 자신이 모른다는 것을 모름. "I don't know" 라는 응답이 자연스럽지 않음
- **Sycophancy**: 사용자의 잘못된 전제에 동조하는 경향

## 같이 보기

- [[gen-read]] — parametric memory를 명시적 배경 문서 형태로 변환하는 변형
- [[retrieval-augmented-generation]] — 외부 검색을 결합한 보완적 접근
- [[llm-as-retrieval]] — index
