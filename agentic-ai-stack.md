---
title: Agentic AI Stack
type: write-up
topics:
  - AI
  - Agent
description: AI 에이전트 시스템을 구성하는 기술 스택을 알아보자
createdAt: 2026-03-30T10:21:00
---
AI 에이전트가 빠르게 발전하는 것 같다. 에이전트가 스스로 판단하고 코드를 실행하는 환경을 더 안전하고 효율적이게 만드는 분야도 커지는 것 같다. 

![[agentic-ai-stack.svg|650]]

## Infrastructure Layer (#1)
- 에이전트의 물리적 논리적 기반이며, 가장 기초가 되는 레이어
- 컴퓨터, 컨테이너 오캐스트레이션, 스토리지, 네트워킹으로 구성됨
### Compute
- CPU, GPU/TPU 등으로 구성됨
- CPU는 오케스트레이션, API 서빙에 사용되고, GPU/TPU는 학습, 추론에 사용됨
- AWS EC2, GCP GCE, Azure VMs 가 주요 제공자
### Container Orchestration
- 쿠버네티스가 유일한 선택지
	- AWS EKS, GCP GKE, Azure AKS
- 하지만 쿠버네티스는 에이전트의 워크로드에 맞지 않음
	- 쿠버네티스는 `Deployment`, `StatefulSet` 두 개의 요소가 있음
		- `Deployment`: 동일한 Pod을 여러개 복제해서 돌리자(stateless)
		- `StatefulSet`: Pod에 아이디를 주고 각각 스토리지와 네트워크를 부여하자(stateful)
	- 하지만 에이전트는 둘 다 아님
		- 에이전트는 각각 고유의 컨텍스트와 메모리를 갖기 때문에 고유의 상태를 갖는다
		- 에이전트는 클러스터 멤버가 아니라 독립된 싱글톤이므로 `StatefulSet` 도 아님
- 이를 해결하기 위해 여러 솔루션이 등장하는 중
### Storage
- 에이전트 시스템에서 저장해야 하는 데이터는 크게 세 가지 종류
- **Object Storage:** 에이전트가 생성한 파일, 로그, 대화 이력 등 비정형 데이터를 저장하는 스토리지
- **Block Storage:** 컨테이너에 마운트되는, 컨테이너가 죽어도 유지되는 디스크
- **Vector DB:** RAG에서 사용되는 임베딩 벡터를 저장하고 검색하는 데이터베이스
### Networking
- 네트워크는 크게 두 가지 종류
- **외부 인터넷으로의 네트워크:** 피해 영역 최소화가 목표
	- 에이전트가 API를 호출하거나 웹 검색을 하는 통신
	- 코드를 LLM이 실행하므로 아웃바운드를 제어해야 함
	- 화이트리스트, 프록시, DNS 제어 등을 통해 제어함
- **에이전트 간 네트워크:** **trust propagation** 방지가 목표
	- 멀티 에이전트 시스템에서 에이전트 간 이루어지는 통신
	- 에이전트 찾기, 찾은 에이전트를 믿을 수 있는지, 누구에게 통신할 수 있는지 등을 제어
	- 누가 누구에게 통신할 수 있는지를 제어함으로 성능과 안정성이 결정됨
		- Hub-and-spoke: 중앙 에이전트가 모든 서브 에이전트에게 작업을 분배하고 결과를 수집함
		- Peer-to-peer: 에이전트끼리 직접 메시지를 주고 받음
		- Message queue: 에이전트가 직접 통신하지 않고 메시지큐를 통해 통신함

> [!NOTE]
> **Trust Propagation** 이란? 영화 기생충에 나온 것과 같이 과외 선생님 - 미술 선생님 - 운전 기사 - 가정부로 최초의 신뢰를 악용해 악의적인 의도가 전체 네트워크에 전파되는 것

## Foundation Models Layer (#2)

### Adaptable intelligence core
- 모델 그 자체로, 다양한 모델로 교체 가능한 컴포넌트임
- 하나의 모델에 종속되지 않고 여러 모델을 어떻게 조합하고 라우팅하느냐가 중요해짐
- 간단한 작업에 강력한 모델을 사용하거나 복잡한 추론에 가벼운 모델을 사용할 이유가 없기 때문
### Inference economics
- 핵심 문제가 2020년 초 학습 비용에서 2026년 추론 비용으로 이동함
- 에이전트의 워크플로우는 하나의 작업에 수백번의 LLM 호출을 발생시키기 때문에 추론 비용이 중요함
- **Quantization / Distillation**: 모델의 정밀도를 줄이거나 경량 모델로 지식을 증류해 많은 추론을 수행
- **Smart routing / caching**: 반복되는 쿼리 패턴을 캐싱하고 복잡도에 따라 모델을 동적 라우팅함
- **Speculative decoding**: 경량 모델이 여러 토큰의 초안을 먼저 생성하고, 강력한 모델이 병렬로 검증해 틀린 지점부터 재생성함으로써 반응 속도를 절감함
- 하드웨어 차원에서는 최신 세대 GPU를 학습에 사용하고 이전 세대 GPU를 추론과 경량 모델 학습에 재배치하는 패턴이 일반적임
### Serving infrastructure
- 사용자에게 강력한 모델을 빠르고 안정적으로 제공하기 위한 레이어
- **Inference engine**: vLLM, SGLang, TensorRT-LLM, NVIDIA Triton
- **Cloud managed serving**: AWS Bedrock, Google Vertex AI, Azure AI Studio, Databricks Model Serving
- **Self-hosted serving**: Ollama, Together AI, Fireworks AI
- **Custom silicon**: Google TPU, Amazon Inferentia, Amazon Trainium 등 GPU 의존도를 줄이면서 추론 비용을 최적화하는 방향으로 발전 중

> [!note]
> **"모델을 만드는 건 소수의 일이지만, 모델을 잘 쓰는 건 모든 엔지니어의 일이다."**
> Foundation Model Layer는 모델의 성능을 논하는 것이 아닌, 모델을 교체 가능한 컴포넌트로 보고 비용-성능-가용성을 고려해 시스템을 설계하는 관점이다.

## Sandbox Layer (#3)

![[agentic-ai-stack-sandbox_runtime_architecture.svg|650]]
- 에이전트가 생성한 코드를 어디서, 어떻게 실행할 지 담당하는 레이어
- 에이전트가 동적으로 생성한 코드는 어떤 **사람**도 리뷰하지 않은 코드
- 그렇기 때문에 존재하지 않는 패키지명을 만들기도 하고 악의적인 코드가 생성될 수도 있음
- 호스트 머신에서 그대로 실행하면 다양한 보안 사고가 발생할 수 있음
- **"AI가 생성한 코드를 믿지 않는다"** 에서 시작한 레이어
- 샌드박스는 4가지의 격리 경계를 제공함
	- **Compute isolation**
		- 커널을 격리하기 위한 경계
		- 격리 기술이 3-tier 로 나뉨
		- **Container (Process Isolation)**: Linux Namespace + `cgroup` 을 사용해 환경을 격리하지만, 호스트 커널을 공유하기 때문에 완벽한 격리는 불가능
		- **gVisor (Kernel Isolation)**: 컨테이너의 모든 시스템 콜을 호스트 커널에 직접 전달하지 않고 유저 공간에서 재구현함
		- **MicroVM (Hardware Isolation)**: 워크로드당 전용 커널을 부팅해 완전한 하드웨어 레벨의 커널 분리를 구현함
	- **Filesystem boundary**: Linux Namespace, 가상 VFS, 또는 독립 커널 파일 시스템을 사용해 호스트 정보 격리, 다른 에이전트 세션과의 정보 격리를 구현함
	- **Network control**: `iptables`, `nftables` 또는 네트워크 정책으로 코드가 접근할 수 있는 호스트를 제한함
	- **Resource limit and TTL**
		- 무한 루프를 제한하거나 무한한 리소스 사용을 제한하기 위해 `cgroup` 을 사용함
		- 또한 TTL을 설정해 종료 후 자동 파기되도록 설정함

## Orchestration Layer (#4)
- 에이전트의 작업을 스케줄링하는 레이어

![[agentic-ai-stack-orchestration_layer_anatomy.svg|650]]

- **Agent Loop**: Observe(상태 인식) - Plan(다음 행동 결정) - Act(툴 호출, 코드 실행) - Reflect(평가, 종료 판단)
	- **Observe**: 유저의 입력, 대화 기록, 실행 결과, 외부 데이터 등을 하나의 Context Window에 모아 현재 상태를 인식하는 단계
	- **Plan**: LLM이 Context를 받아서 자연어로 추론한 뒤 구조화된 행동을 출력함
	- **Act**: 계획 단계의 행동을 Sandbox Layer에서 실제로 실행하는 단계
	- **Reflect**: 실행 결과를 보고 목표 달성 여부를 판단해 Observe로 돌아가거나 최종 답변을 생성하는 단계
- **Planning**: 에이전트의 작업을 스케줄링
	- **ReAct (Reason + Act)**
		- LLM이 한 단계씩 생각하고 행동하고 관찰하는 것을 반복하는 패턴
		- 단순하고 디버깅이 쉽지만 장기 계획에 약함
	- **Plan and Execute**
		- Divide and Conquer 전략
		- 에이전트 작업을 세부 작업으로 분해 후 순서대로 실행
		- 장기 계획에 강하지만
		- 초기 계획이 잘못되면 모든 결과물이 잘못됨
	- **LangGraph**
		- Agent Loop를 directed graph로 정의
			- 노드가 하나의 실행 단위
			- 엣지가 상태 전이 조건
		- Finite State Machine 이지만 각 전이 조건이 LLM으로 동적 생성될 수 있음
- **Memory**: 에이전트에게 주어지는 상태
	- LLM 자체는 매 호출마다 별도의 Context Window가 구성되므로 Stateless
	- 별도의 기억(memory, state)를 명시적으로 관리해야 함
	- **Working Memory**: 직접 Context Window 안에 들어가는 기억
	- **Short-term Memory**: 세션 내 누적되는 기억
		- 오래된 대화를 요약해 압축하거나
		- 슬라이딩 윈도우로 최근 몇 개의 대화만 유지하는 전략을 가짐
	- **Long-term Memory**: 세션 간 누적되는 기억
		- 에이전트가 Observe 단계에서 Vector Search를 통해 관련 과거 경험을 상기하는 등
- **Tool Registry**: 에이전트가 사용할 수 있는 도구
	- LLM이 생성한 텍스트를 실행하는 단계
	- LLM이 생성한 구조화된 행동을 오케스트레이터가 파싱해 실행하고, 그 결과를 다시 LLM에게 전달하는 구조
	- MCP가 이를 표준화한 프로토콜임
- **Multi-agent coordination**: 여러 에이전트가 협력하는 것
	- 에이전트가 하위 에이전트에게 작업을 할당하고 결과를 수집하는 구조
	- 한 에이전트의 결과물이 다음 에이전트의 입력이 되는 구조
	- 에이전트끼리 메시지를 주고 받으며 자율적으로 협력함
		- 디버깅이 매우 어려움
		- A2A가 이를 표준화한 프로토콜임

## Application Layer (#5)
- 에이전트가 다른 시스템 또는 사용자와 만나는 접점
- Infrastructure - Foundation Model - Sandbox - Orchestration 이 제공하는 능력을 실제 문제 해결에 바인딩하는 계층
- 세 가지 특징을 가짐
	- **Non-deterministic execution**: 같은 입력에 같은 출력을 보장하지 않음
	- **Loop-based execution model**: Request - Process - Response 의 선형적 파이프라인이 아닌 Observe - Plan - Act - Reflect 의 루프를 반복함
	- **Tool-calling as first-class operation**: 모델이 추론 시점에 동적으로 어떤 도구를 호출할지 결정함
- 세 가지 패턴이 있음
	- **Copilot Pattern**: 사람이 주도하고 에이전트가 보조하는 구조
	- **Single Agent Pattern**: 하나의 에이전트가 목표를 받아 루프를 돌며 완수하는 구조
	- **Multi Agent Pattern**: 여러 에이전트가 하나의 목표를 받아 각각 다른 역할을 맡고 협업하는 구조

## Observability & Governance

### Observability: 무슨 일이 일어나고 있는가
- **Distributed Tracing**: 실행 경로 추적
	- HTTP Request - Service A - Service B - Database - Response 와 같이
	- User Prompt - LLM Reasoning - Tool Selection - Tool Execution - LLM Reasoning - Response 구조를 추적하는 방법
	- LLM 호출, Tool 호출, 메모리 검색, 다른 에이전트 통신 등을 Session - Trace - Span 계층으로 나누어 기록하고 추적함
	- Span에는 Input Prompt, Output, Token 수, Latency, 사용한 모델 등으로 기록함
- **Evaluation**: 품질 측정
	- Non-deterministic 특성때문에 테스트 코드가 아닌 새로운 품질 특정 방법이 생김
	- **Offline eval**: 배포 전 Golden Dataset으로 에이전트를 시뮬레이션하고 품질을 측정함
	- **Online eval**: 배포 후 실제 트레픽을 기반으로 실시간 품질을 측정함
	- **LLM-as-a-judge**: LLM이 에이전트의 결과를 평가하는 패턴
- **Cost-Performance Monitoring**: 비용 감시
	- 에이전트 시스템은 실행마다 비용이 발생하는 구조
	- 불필요한 추론 루프를 돌거나 과도한 도구 호출이 발생하면 많은 비용이 발생함
	- GPU 사용량 또는 LLM API 호출량을 관찰하고 동적으로 스케일링해 수익성을 유지해야 함

### Governance: 무슨 일이 일어나도 되는가
- **Access Control and Permission Scoping**: 에이전트를 사람처럼 인증-정책 기반 접근 관리를 해야 함
- **Tiered Autonomy**: 위험하지 않은 루틴 액션은 자동화, 위험한 운영 결정은 반자동화, 매우 위험한 활동은 반드시 사람에게 허가를 받아야 하는 모델
- **Audit Trail and Compliance**: 에이전트의 모든 결과물, 판단 컨텍스트, 접근한 데이터, 어떤 옵션을 고려해 어떤 판단을 내렸는지 등에 대한 변경할 수 없는 감사 로그가 있어야 함

> [!note]
> Agentic AI 스택은 LLM을 통해 신입사원을 만들고자 하는 문제에서 발전한 기술인 것 같다. 그 안에서 에이전트가 동적으로 생성한 코드를 실행하고자 하는 필요가 생겼고, Sandbox 레이어가 최근 분리되기 시작한 것 같다.
