---
type: note
title: Transformer
description: Attention is All You Need
topics:
  - AI
  - Transformer
createdAt:
---
트렌스포머는 [[RNN]]과 비교를 많이 한다. 트렌스포머는 번역을 위해 만들어진 시퀀스 모델인데, 이전에 번역은 RNN으로 접근했기 때문이다. RNN은 시퀀스를 순차적으로 처리해 다음과 같은 한계가 있었다.
- 병렬처리를 할 수 없어 느렸다.
- 거리가 먼 단어의 관계를 잘 잡을 수 없었다.
트렌스포머는 이러한 문제를 Attention으로 해결했다. 우리가 단어를 순서대로 읽는 것이 아닌, 모든 단어가 서로를 직접 마주보게 하는 것이다.