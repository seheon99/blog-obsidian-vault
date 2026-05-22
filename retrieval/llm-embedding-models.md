---
title: LLM-based Embedding Models
type: note
topics:
  - LLM
  - Retrieval
  - RAG
description: LLM 아키텍처 위에 구축된 임베딩 모델
createdAt: 2026-05-03T14:30:00
---
**LLM-based Embedding Models**는 기존 LLM 아키텍처를 기반으로 만들어진 임베딩 모델이다. E5-Mistral, GTE-Qwen, Gecko, NV-Embed 등이 대표적이다.

## 동작

기존 LLM(예: Mistral-7B, Qwen)을 contrastive learning이나 fine-tuning으로 임베딩 전용 모델로 변환한다.

- 입력 텍스트 → LLM 인코더 통과 → 고차원 벡터(보통 512-4096차원)
- 학습은 (query, positive document, hard negatives) 트리플로 contrastive loss 사용
- RAG의 검색 단계에서 query/document 임베딩 생성에 사용

## 위치

이것은 RAG 안에서 가장 널리 퍼져 있지만 **가장 눈에 띄지 않는 형태**의 LLM 통합이다. Retrieval 파이프라인을 "외부 검색 시스템" 으로 인식하지만, 그 안의 임베딩 모델 자체가 이미 LLM이다.

## 강점

- **의미 표현력**: BERT 기반 임베딩보다 paraphrase, 다중 문장 의미 표현에 강함
- **멀티링구얼**: 다국어 도메인에서 dense embedding의 표준이 됨
- **Long-context 임베딩**: 긴 문서를 그대로 임베딩하는 모델도 등장 (예: voyage-3.5, 32K input)

## 한계

- **임베딩 비용**: BERT 기반 임베딩보다 높음 — 인덱싱 시점 비용 증가
- **차원 증가로 저장/검색 비용 상승**: ANN 인덱스 크기, 메모리 사용량 증가
- **Fine-tuning 데이터 의존**: 도메인 특화 성능은 학습 데이터 품질에 좌우됨
