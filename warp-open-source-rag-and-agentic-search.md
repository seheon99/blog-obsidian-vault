---
title: Warp 코드 살펴보기
topics:
  - RAG
  - LLM
  - Agent
  - Code Search
  - Warp
  - Merkle Tree
description: Warp가 AGPLv3로 공개한 클라이언트 코드베이스(github.com/warpdotdev/warp)를 직접 읽고, Merkle 트리 기반 증분 인덱싱 RAG 파이프라인과 28개의 Action으로 구성된 agentic 도구 시스템을 분석한다.
createdAt: 2026-05-03T15:00:00
---

2025년 후반, Warp는 자사 터미널 클라이언트의 전체 소스 코드를 [github.com/warpdotdev/warp](https://github.com/warpdotdev/warp)에 AGPLv3로 공개했다. 이 코드를 살펴보며 Cursor 같은 현대 코딩 도구들이 어떻게 설계되었는지 알아보자.

Warp의 코드 중 두 가지 서브시스템을 자세히 살펴봤다. 사용자의 코드를 어떻게 인덱싱하고 검색하는지 알아보기 위해 **RAG 파이프라인**과 **agentic search** 설계를 위주로 살펴봤다. 이를 위해 `crates/ai/`, `crates/graphql/`, `crates/warp_ripgrep/` 위주로 읽었다.

## 공개된 코드는 Warp 클라이언트

**Warp가 공개한 것은 클라이언트 코드뿐이다.** RAG 파이프라인의 임베딩 모델, 벡터 DB, 리랭커 모델, 추론 LLM은 모두 서버에 있고, 그 서버는 비공개다. 클라이언트는 GraphQL을 통해 이 서버와 통신한다.

Warp의 RAG가 어떻게 동작하는지 알아보기 위해 질문을 다음 두 질문으로 나누어보았다.

1. **클라이언트가 어떤 책임을 지는가** — 파일 변경 감지, Merkle 트리 유지, 청킹, 증분 동기화, 도구 실행
2. **서버가 어떤 책임을 지는가** — 실제 임베딩 생성, 벡터 검색, 리랭킹, 에이전트 루프 자체

이 글은 1번에 대해서는 코드 단위로 답할 수 있다. 2번에 대해서는 클라이언트가 호출하는 GraphQL 스키마와 응답 구조를 보고 추론할 수 있을 뿐이다. 그 이상을 단정하지 않는 게 이 글의 원칙이다.

저장소 루트의 `crates/`를 펼치면 63개 크레이트 중 RAG·에이전트 관련 핵심은 네 곳이다.

```
crates/ai/                            ← 본진. RAG 인덱싱 + 에이전트 액션 정의
crates/graphql/                       ← 서버 API 경계 (cynic으로 정의된 쿼리/뮤테이션)
crates/warp_ripgrep/                  ← Grep 도구 구현 (ripgrep 래핑)
crates/computer_use/                  ← Computer Use 도구
```

`crates/ai`만 68개 파일이고 그중 절반 이상이 `index/full_source_code_embedding/`에 있다. Warp가 이 모듈에 얼마나 큰 비중을 뒀는지 짐작할 수 있다.

## Part 1. RAG 파이프라인 — Merkle 트리 기반 증분 인덱싱

### 큰 그림부터

Warp의 RAG 파이프라인은 **클라이언트가 코드베이스의 Merkle 트리를 유지**하고, **서버가 보유한 트리와의 차이만 동기화**하는 구조다. 한 줄짜리 수정이 전체 재인덱싱을 일으키지 않는다. 데이터 흐름은 다음과 같다.

```
파일 변경
   ↓
ChangedFiles { deletions, upsertions } 누적 (1시간 윈도우 기본)
   ↓
MerkleTree.upsert_files() / .remove_files()
   ↓
변경된 노드만 ancestor chain까지 해시 재계산
   ↓
SyncMerkleTree 쿼리로 서버와 차이 노드 식별
   ↓
GenerateCodeEmbeddings 뮤테이션 (변경된 fragment만 4MB 단위로 배치 전송)
   ↓
서버: 임베딩 생성 → 벡터 DB 저장
   ↓
[검색 시]
   ↓
GetRelevantFragments(query, root_hash) → 후보 fragment 해시
   ↓
RerankFragments(query, candidates) → 정렬된 fragment
   ↓
에이전트의 SearchCodebase 도구 결과로 반환
```

이 흐름이 머릿속에 그려진 상태로 코드를 따라가면 훨씬 편하다.

### Merkle 노드의 3종 구조

`crates/ai/src/index/full_source_code_embedding/merkle_tree/node.rs`에서 노드는 세 종류로 나뉜다.

```rust
pub(crate) enum NodeId {
    /// A file node that contains fragment children
    File {
        absolute_path: PathBuf,
        file_size: usize,
        fs_modified_time: DateTime<Utc>,
        file_contents_hash: String,
    },
    /// A directory node that contains file and directory children
    Directory { absolute_path: PathBuf },
    /// A leaf node representing a code fragment
    Fragment {
        absolute_path: PathBuf,
        content_range: Range<ByteOffset>,
    },
}
```

**Directory → File → Fragment**의 3단 위계다. 여기서 주목할 설계 결정 두 개가 있다.

첫째, **File 노드에 `file_contents_hash`를 들고 있다.** 이는 fragment 단위 해시와 별개로, 파일 전체 내용의 SHA-256이다. 파일 단위 빠른 변경 감지가 가능하고 — `fs_modified_time`만 믿으면 거짓 양성이 너무 많다 — fragment 분해 없이도 무결성을 확인할 수 있다.

둘째, **Fragment는 byte range로만 정의되어 콘텐츠를 들고 있지 않다.** 트리 자체는 메모리 효율적이고, 실제 콘텐츠는 청킹 시점에 파일에서 잘라 쓴다. snapshot 직렬화 시점에도 트리 구조만 디스크에 보관되어 warm start가 빠르다.

해시는 SHA-256, `sha2` 크레이트를 사용한다(`merkle_tree/hash.rs`).

```rust
pub(crate) struct MerkleHash(Arc<GenericArray<u8, <Sha256 as OutputSizeUser>::OutputSize>>);
```

`Arc`로 감싼 이유는 부모 노드가 자식 해시를 참조할 때 복제 비용을 없애기 위해서다. 20만 노드짜리 트리에서 32바이트 해시 복사가 누적되면 무시할 수 없다.

### 청킹: Tree-sitter 시멘틱 청킹과 라인 기반 fallback

청킹은 두 단계 fallback이다 (`chunker.rs`).

```rust
const LINES_PER_CHUNK: usize = 200;
const AVG_CHAR_PER_LINE: usize = 60;
const MAX_BYTES_PER_CHUNK: usize = LINES_PER_CHUNK * AVG_CHAR_PER_LINE;  // 12,000

pub fn chunk_code<'a>(code: &'a str, path: &'a Path) -> Vec<Fragment<'a>> {
    if let Some(fragments) = try_chunk_code_semantically(code, path) {
        return fragments;
    }
    naive::chunk_code(code, path, MAX_BYTES_PER_CHUNK, LINES_PER_CHUNK)
}
```

기본 청크 크기는 **200줄 또는 12KB**. 200 × 60자라는 산술이 직접 박혀 있는 게 흥미롭다. 평균 60자/줄이 안 되는 코드(LISP 같은 기능형 언어, 압축된 한 줄 함수형 코드)에서는 청크가 작아지고, 60자를 크게 넘는 코드(긴 문자열, 미니파이된 JS)에서는 200줄짜리 청크가 12KB를 넘으므로 byte 기준으로 다시 잘린다.

#### 시멘틱 청킹

`chunker/semantic.rs`는 tree-sitter의 syntax 트리를 따라 재귀적으로 노드를 쪼갠다.

```rust
const MAX_TRAVERSAL_DEPTH: usize = 200;

fn split_node<'a, 'b>(
    node: Node<'b>,
    code: &'a str,
    max_bytes_per_chunk: usize,
    path: &'a Path,
    cursor: &mut TreeCursor<'b>,
    depth: usize,
) -> anyhow::Result<Vec<Fragment<'a>>> {
    if depth > MAX_TRAVERSAL_DEPTH {
        return Err(anyhow!("Max depth {} exceeded", MAX_TRAVERSAL_DEPTH));
    }
    // max_bytes_per_chunk 이상이면 자식으로 내려가서 분할
    // 자식에서도 분할이 안 되면 그 노드 자체로 fragment 종료
}
```

핵심 휴리스틱: **fragment는 절대 12KB를 넘지 않는다.** 임베딩 모델의 토큰 한도를 초과하지 않기 위함이다. 함수가 12KB를 넘으면 함수 본문 안의 statement 단위로 더 들어간다. 200 깊이 제한은 병적으로 깊게 nested된 표현식에서 무한 재귀를 막는다. 분할 후에는 `coalesce_fragments`가 너무 작은 인접 fragment를 다시 합쳐 의미적 응집을 회복시킨다.

#### Naive fallback

지원하지 않는 언어 또는 파싱 실패 시 `chunker/naive.rs`로 폴백한다. 200줄로 자르되, 그 청크가 12KB를 넘으면 줄 단위로 쪼개고, 한 줄이 12KB를 넘으면 byte 경계(UTF-8 안전)에서 자른다. 미니파이된 vendored JS 한 줄을 마주쳐도 죽지 않게 하려는 방어다.

### 변경 감지: ChangedFiles 누적과 흡수

파일 시스템 워처가 만들어내는 변경은 곧바로 동기화되지 않고 `ChangedFiles`에 누적된다(`changed_files.rs`).

```rust
pub(super) struct ChangedFiles {
    pub(super) deletions: HashSet<PathBuf>,
    pub(super) upsertions: HashSet<PathBuf>,
}

impl ChangedFiles {
    pub(super) fn merge_subsequent(&mut self, mut subsequent_changes: Self) {
        for path in subsequent_changes.deletions.drain() {
            if self.upsertions.contains(&path) {
                self.upsertions.remove(&path);
            }
            self.deletions.insert(path);
        }
        for path in subsequent_changes.upsertions.drain() {
            if self.deletions.contains(&path) {
                self.deletions.remove(&path);
            }
            self.upsertions.insert(path);
        }
    }
}
```

**상쇄 로직**이 인상적이다. "파일 추가 → 삭제"는 결국 삭제만 남기고, "삭제 → 다시 추가"는 추가만 남긴다. brew install로 vendor dir이 잠깐 사라졌다 돌아오는 IDE 환경에서, 이 흡수 없이 모든 이벤트를 서버로 보냈다면 벡터 DB가 시간당 수만 번 깜빡였을 것이다.

플러시 정책은 다음 상수에서 보인다(`codebase_index.rs`).

```rust
const REINDEX_INTERVAL: Duration = Duration::from_secs(20 * 60);                // 20분 전체 재인덱싱
const DEFAULT_INCREMENAL_SYNC_FLUSH_INTERVAL: Duration = Duration::from_secs(60 * 60); // 1시간 증분 플러시
const REPO_WATCHER_DEBOUNCE_DURATION: Duration = Duration::from_secs(10);       // 10초 디바운스
const REPO_SNAPSHOT_PERSISTENCE_MINUTES: u64 = 10;                              // 10분 디스크 스냅샷
```

기본 1시간 윈도우는 보수적이다. 활발히 코딩 중일 때 변경 사항을 1시간 묵혀두면 검색 쿼리에 신선한 코드가 안 잡힐 수 있다. 그래서 `force_flush` 경로가 별도로 존재한다 — 사용자가 명시적으로 검색을 실행할 때 강제 플러시되는 식으로 보인다.

### 노드 갱신과 정렬 전략

`upsert_files`와 `remove_files`는 트리를 mutating하면서 변경된 노드 집합을 `NodeMask`에 기록한다. 디렉토리 노드의 자식은 **해시 순으로 정렬**되어 저장된다.

```rust
let new_children_with_idx = self
    .children
    .drain(..)
    .enumerate()
    .sorted_by(|(_, a), (_, b)| a.hash.cmp(&b.hash));
```

왜 이렇게 했는가. 디렉토리 트래버설 순서는 파일시스템과 OS에 따라 달라진다. 자식을 해시 순으로 정렬해 부모 해시를 계산하면, 동일한 파일 집합이라면 디렉토리 워킹 순서와 무관하게 동일한 부모 해시가 나온다. **머신 간 캐시 호환성이 보장**된다.

반면 File 노드의 자식 fragment들은 정렬하지 않는다. fragment의 byte range 순서 자체가 의미를 가지므로 — 파일 처음의 import 블록과 끝의 export 블록을 뒤섞으면 의미가 달라진다.

### 동기화: 풀 싱크와 증분 싱크

`sync_client.rs`는 두 모드를 가진다.

**Full sync**는 트리 전체를 BFS 형태로 순회하며 서버와의 차이를 좁혀간다.

```rust
let mut nodes_pending_check = vec![root_node];
loop {
    nodes_pending_check = operation
        .check_if_nodes_synced(&nodes_pending_check, sync_progress_tx.clone())
        .await?;
    if nodes_pending_check.is_empty() {
        break;
    }
}
```

각 라운드에서 클라이언트는 노드 해시 묶음을 서버에 보내고, 서버는 "이 중 어떤 게 우리 쪽과 다른지" 알려준다. 다른 노드만 자식까지 내려가서 다시 비교한다. **Merkle proof와 본질적으로 같은 패턴**이다 — 변경된 가지에 한해서만 트래버설이 일어난다.

**Incremental sync**는 트리 갱신 시점에 변경된 노드 리스트를 받아 그것만 동기화한다.

배치 전략은 두 종류 한도가 같이 걸린다.

```rust
const SYNC_NODE_BATCH_SIZE: usize = 500;
const MIN_UPDATE_NODE_BATCH_SIZE: usize = 100;
const MAX_BATCH_CONTENT_BYTES: usize = 4_000_000;  // 4 MB
```

`SYNC_NODE_BATCH_SIZE`는 한 요청당 최대 500개 노드, `MAX_BATCH_CONTENT_BYTES`는 4MB. 둘 중 먼저 걸리는 게 한도다. **4MB라는 숫자가 흥미롭다.** Cloud Armor — Google Cloud의 WAF — 기본 요청 크기 한도가 5MB다. JSON 직렬화 오버헤드를 빼면 4MB가 실용적인 안전 한도다. Warp가 GCP를 쓴다는 사실이 이 상수에서 새어나온다.

### 상태 머신: 동기화 실패의 우아한 처리

```rust
enum TreeSourceSyncState {
    Synced {
        tree: MerkleTree,
        server_sync_result: ServerSyncResult,
    },
    Syncing {
        last_server_synced_root_node: Option<NodeHash>,
        abort_handle: Option<AbortHandle>,
        sync_progress: Option<SyncProgress>,
    },
    InitializeTreeFailure(Error),
}

pub(super) enum ServerSyncResult {
    Success,
    Failed {
        error: Error,
        last_server_synced_root_node: Option<NodeHash>,
    },
}
```

`last_server_synced_root_node`가 핵심이다. **동기화가 실패해도 마지막 성공 시점의 root hash가 보존된다.** 검색 쿼리는 이 해시로 서버에 묻는다 — 결과는 stale일 수 있어도 일관된 시점의 결과다. 동기화 진행 중에도 검색이 동작하고, 실패 시점에는 신선도를 잃지만 맞는 답을 준다. **freshness vs availability 트레이드오프를 명시적으로 코드에 박은** 사례다.

또 하나, 다음 증분 플러시는 직전 동기화가 성공한 경우에만 실행된다.

```rust
move |me, _, ctx| {
    if me.codebase_index_status().last_sync_successful() == Some(true) {
        me.flush_incremental_update(store_client, ctx);
    }
}
```

서버가 일시적으로 죽었을 때 클라이언트가 30초마다 같은 요청을 폭탄처럼 던지지 않는다. 실패가 있으면 backoff에 가까운 지연이 자연스럽게 발생한다.

### 네트워크 경계: GraphQL 스키마

`crates/graphql/src/api/`에 cynic으로 정의된 GraphQL 스키마가 있다. RAG 관련 엔드포인트는 5개다.

| 종류 | 엔드포인트 | 용도 |
| --- | --- | --- |
| Query | `codebase_context_config` | 서버가 사용하는 임베딩 모델, 토큰 한도 등 설정을 받아온다 |
| Query | `sync_merkle_tree` | 클라이언트의 노드 해시 리스트를 보내 차이만 받아온다 |
| Mutation | `generate_code_embeddings` | fragment 콘텐츠를 보내 임베딩을 생성한다 |
| Mutation | `update_merkle_tree` / `populate_merkle_tree_cache` | 중간 노드 트리 구조를 서버에 반영 |
| Query | `get_relevant_fragments` | 자연어 쿼리를 보내 후보 fragment 해시를 받는다 |
| Query | `rerank_fragments` | 후보를 콘텐츠 포함해 보내 재정렬된 결과를 받는다 |

쿼리·뮤테이션 분리가 깨끗하다. **임베딩 생성과 검색이 동일 트랜잭션이 아니라**는 점이 중요하다 — fragment를 인덱싱한 뒤 즉시 검색했을 때 인덱스에 잡히지 않을 수 있다. 이건 사용자 경험에 미묘한 영향을 준다.

`get_relevant_fragments`의 응답이 **콘텐츠가 아니라 `Vec<ContentHash>`만** 돌려준다는 점도 눈여겨볼 만하다.

```rust
pub struct GetRelevantFragmentsOutput {
    pub candidate_hashes: Vec<ContentHash>,
}
```

서버는 누가 어떤 코드를 들고 있는지 모르거나, 적어도 응답에는 담지 않는다. 클라이언트가 hash로부터 로컬 파일 시스템에서 fragment를 다시 읽어 — 또는 서버의 fragment store에서 별도 호출로 — 가져온다. **개인정보 측면에서는 안전한 설계**다. 그리고 `rerank_fragments`는 콘텐츠를 다시 보낸다 — 이쪽은 cross-encoder 같은 콘텐츠 기반 리랭커를 돌려야 하기 때문이다.

마지막으로, `RETRIEVE_FRAGMENT_CONTEXT_LENGTH = 0`이라는 상수가 `codebase_index.rs`에 있다.

```rust
/// We are not attaching any context lines right now since it does not seem to improve quality
/// and is causing extra token efficiency issues.
const RETRIEVE_FRAGMENT_CONTEXT_LENGTH: usize = 0;
```

검색 결과로 fragment 자체만 반환하지, 위·아래 N줄 컨텍스트를 붙이지 않는다. 코멘트가 솔직하다 — "별로 도움 안 되고 토큰만 낭비"였다는 것. 많은 RAG 구현체가 무비판적으로 chunk + N줄 패딩을 디폴트로 두는데, Warp는 측정 결과 기반으로 0으로 내렸다.

## Part 2. Agentic Search — 실제 구현은 "에이전트 도구로서의 검색"

### "agentic"이라는 단어는 코드에 없다

먼저 분명히 해두자. `crates/`에서 `grep -r "agentic"`을 돌리면 **단 한 건도 나오지 않는다.** "agentic search"는 마케팅 용어다. 코드에서는 단순히 `agent`, `action`, `SearchCodebase`로 나뉘어 있다.

그럼 실제로 무엇이 agentic한가? 답은 명확하다. **에이전트는 도구를 가지고 있고, 그 도구 중 하나가 시멘틱 코드 검색이다.** 검색은 항상 켜져 있는 컨텍스트 윈도우 자동 주입(auto-context)이 아니라, 모델이 명시적으로 호출하는 함수 콜이다.

이건 Cursor의 초기 "@codebase" 또는 GitHub Copilot Chat의 자동 컨텍스트 주입과 다른 철학이다. 자동 주입은 토큰 예산을 모델에게서 빼앗고, 잘못 가져온 컨텍스트가 답을 왜곡시킨다. Warp의 선택은 **"모델이 필요할 때 명시적으로 부른다"** — 더 간결하고 디버깅하기 쉽다. 이걸 "agentic search"라고 부르는 게 이상하지 않다.

### AIAgentActionType: 28개의 도구

`crates/ai/src/agent/action/mod.rs`의 `AIAgentActionType` enum이 에이전트가 호출 가능한 도구의 전수 목록이다. 직접 세어 본 결과 **28개**다(서브에이전트가 33개라고 했지만 그건 sub-enum의 variant까지 잘못 포함한 숫자였다). 카테고리로 묶어 정리한다.

**파일 읽기·탐색 (5개)**
- `ReadFiles` — 라인 범위 옵션을 가진 파일 읽기
- `Grep` — 정규식 검색 (warp_ripgrep으로 구현, 후술)
- `FileGlob` / `FileGlobV2` — 글롭 패턴 검색
- `SearchCodebase` — 임베딩 기반 시멘틱 검색 (RAG 진입점)

**코드 수정 (1개 + 문서 도구 3개)**
- `RequestFileEdits` — diff 적용 (str-replace 또는 V4A hunk 형식)
- `ReadDocuments` / `EditDocuments` / `CreateDocuments` — Warp Drive 문서 다루기

**셸 실행 (4개)**
- `RequestCommandOutput` — 셸 명령 실행. `is_read_only`, `is_risky`, `uses_pager`, `wait_until_completion`, `rationale`, `citations` 메타데이터를 포함
- `WriteToLongRunningShellCommand` — 실행 중인 PTY에 입력 전달
- `ReadShellCommandOutput` — 이전에 시작한 long-running 명령의 출력 회수
- `TransferShellCommandControlToUser` — 사용자에게 셸 제어 이양 (이유 첨부)

**확장성·통합 (4개)**
- `ReadSkill` — Skill 로드
- `CallMCPTool` — MCP 도구 호출 (서버 ID + 이름 + JSON 입력)
- `ReadMCPResource` — MCP 리소스 URI로 읽기
- `UploadArtifact` — 로컬 파일을 대화 아티팩트로 업로드

**에이전트 오케스트레이션 (3개)**
- `StartAgent` — 자식 에이전트 시작 (로컬 임베디드 또는 원격 작업자)
- `SendMessageToAgent` — 다른 에이전트에게 비동기 메시지
- `FetchConversation` — 이전 대화의 태스크 상태 로드

**코드 리뷰·UX (5개)**
- `InsertCodeReviewComments`, `OpenCodeReview`, `InitProject`, `SuggestPrompt`, `SuggestNewConversation`

**컴퓨터 사용 (2개)**
- `UseComputer` — 직접 GUI 제어 (마우스/키보드/스크린샷)
- `RequestComputerUse` — 컴퓨터 사용 권한 요청

**사용자 상호작용 (1개)**
- `AskUserQuestion` — 다중 선택 질문

이 목록을 처음 봤을 때 인상은 분명했다. **셸 실행 도구의 정교함과 에이전트 오케스트레이션 도구의 존재감이 다른 코드 에이전트들과 구별된다.**

`RequestCommandOutput`만 봐도 그렇다. `is_read_only`, `is_risky`, `wait_until_completion` 플래그가 LLM이 채워야 하는 인자로 노출되어 있다. 단순히 셸을 실행하는 게 아니라, **모델에게 "이 명령은 읽기 전용인지, 위험한지, 끝까지 기다려야 하는지를 판단하라"고 강제**한다. 이 메타데이터가 클라이언트 측 정책 — 위험한 명령은 사용자 확인을 요구하기, 읽기 전용 명령은 자동 승인하기 — 의 입력이 된다.

### SearchCodebase 도구의 시그니처

```rust
pub struct SearchCodebaseRequest {
    pub query: String,
    /// Optional list of file paths to search through. This is used to narrow down the search scope.
    /// Files are searched if any of the partial paths are a substring of the file path.
    pub partial_paths: Option<Vec<String>>,
    /// Optional absolute path to the codebase that we want to search.
    pub codebase_path: Option<String>,
}

pub enum SearchCodebaseResult {
    Success { files: Vec<FileContext> },
    Failed { reason: SearchCodebaseFailureReason, message: String },
    Cancelled,
}
```

자연어 쿼리, 선택적 경로 필터, 선택적 코드베이스 경로. 단순하다. 결과는 `FileContext` 리스트로, 각 항목은 파일 경로·콘텐츠·라인 범위를 포함한다. **에이전트는 이 결과를 그대로 다음 추론 턴의 컨텍스트로 받는다.**

### 에이전트 루프는 어디에 있는가

`crates/ai/src/agent/`을 아무리 뒤져도 `fn run_agent(...)`나 `loop { ... tool_call ... }` 같은 루프 구현이 없다. **에이전트 루프는 서버에 있다.** 클라이언트는 다음 역할만 한다.

1. 서버로부터 `AIAgentActionType`을 받는다 (protobuf over WebSocket으로 보임 — `warp_multi_agent_api` 의존성이 그 단서).
2. 액션을 로컬에서 실행한다.
3. 결과를 `AIAgentActionResultType`으로 변환해 서버에 돌려준다.
4. 서버가 다음 LLM 턴을 돌려 새 액션을 결정한다.

이는 두 가지 의미다. 첫째, **클라이언트는 LLM의 비밀을 모른다.** 어떤 모델이, 어떤 시스템 프롬프트로, 어떤 temperature로 추론하는지 클라이언트 코드에는 없다. 둘째, **클라이언트는 OS 권한을 가진 도구 실행자이고, 서버는 결정자**다. 권한 분리 측면에서 깔끔하다 — 모델이 환각해서 `rm -rf /` 같은 명령을 만들어도 클라이언트의 정책 게이트(is_risky 플래그, 사용자 확인)에서 막힌다.

### Action ↔ ActionResult 변환

`agent/action/convert.rs`에서 protobuf wire format이 Rust enum으로 들어오는 모습을 볼 수 있다.

```rust
impl From<api::message::tool_call::SearchCodebase> for AIAgentActionType {
    fn from(value: api::message::tool_call::SearchCodebase) -> Self {
        AIAgentActionType::SearchCodebase(SearchCodebaseRequest {
            query: value.query,
            partial_paths: if !value.path_filters.is_empty() {
                Some(value.path_filters)
            } else {
                None
            },
            codebase_path: if !value.codebase_path.is_empty() {
                Some(value.codebase_path)
            } else {
                None
            },
        })
    }
}
```

빈 `Vec`을 `None`으로 해석해서 의미적 옵션성을 보존한다. 사소하지만 정확한 매핑이다.

### Skills: YAML front-matter markdown

Warp의 Skill 시스템은 코드 한가운데 박혀 있는 흥미로운 결정이다. `SkillProvider` enum을 그대로 옮기면 다음과 같다.

```rust
pub enum SkillProvider {
    Warp,
    Agents,
    Claude,
    Codex,
    Cursor,
    Gemini,
    Copilot,
    Droid,
    Github,
    OpenCode,
}
```

**Warp가 자기 외에도 9개 다른 코드 에이전트의 skill 디렉토리를 인식한다.** `~/.claude/skills`, `~/.cursor/skills`, `.copilot/skills`, `.codex/skills`. 이는 시장 포지셔닝 측면에서 강력하다. "Cursor에서 만들어둔 skill을 Warp에서 그대로 쓸 수 있다." 사용자를 락인하는 대신 **공통 형식의 흡수자**가 되겠다는 선언이다. 이 결정은 다른 IDE/터미널 AI 제품들이 따라해야 할 패턴이라 본다.

Skill은 YAML front matter가 있는 markdown 파일이다.

```markdown
---
name: "Debug TypeScript"
description: "Add console.log statements for variable inspection"
---

# Markdown content describing the skill
When debugging TypeScript:
1. Inspect variables with console.log
...
```

파서는 `---`...`---`로 둘러싸인 YAML 블록을 추출하고, 그 아래 markdown을 본문으로 취급한다. 에이전트는 `ReadSkill` 도구로 path 또는 bundled-id를 통해 skill을 로드해서, 그 콘텐츠를 다음 턴의 컨텍스트로 흡수한다. 본질적으로 **"prompt as a file"** — 프롬프트 엔지니어링을 코드처럼 버전 관리하기 위한 단순한 포맷이다.

### Project Rules: WARP.md / AGENTS.md

`crates/ai/src/project_context/model.rs`의 핵심 상수.

```rust
const RULES_FILE_PATTERN: [&str; 2] = ["WARP.md", "AGENTS.md"];
const MAX_SCAN_DEPTH: usize = 3;
const MAX_FILES_TO_SCAN: usize = 5000;
```

`WARP.md`와 `AGENTS.md` 파일을 ancestor 디렉토리에서 발견하면 system context로 포함시킨다. 파이썬 프로젝트면 "PEP 8을 따르라", 회사 코드면 "이 모듈에는 절대 mutation을 만들지 말라" 같은 규칙을 코드와 함께 git에 두는 패턴이다. 깊이 3까지만 스캔, 파일 5,000개 한도 — 모노레포에서도 안전하다.

`AGENTS.md`라는 이름은 명시적으로 "agent-agnostic" 의도다. Warp뿐 아니라 어느 에이전트가 와도 이 파일을 읽도록 하자는 사실상 표준화 시도. Skill 시스템과 같은 철학이다.

### Diff Validation: 모델이 만든 패치를 신뢰하지 않는다

`crates/ai/src/diff_validation/mod.rs`는 LLM이 생성한 코드 패치를 적용 전에 검증한다. 두 형식을 지원한다.

```rust
pub enum ParsedDiff {
    StrReplaceEdit {
        file: Option<String>,
        search: Option<String>,
        replace: Option<String>,
    },
    V4AEdit {
        file: Option<String>,
        move_to: Option<String>,
        hunks: Vec<V4AHunk>,
    },
}
```

`StrReplaceEdit`는 단순한 search-replace, `V4AEdit`는 OpenAI의 V4A 형식(hunk 단위 컨텍스트 매칭). 검증 흐름:

1. 원본 파일을 읽는다.
2. `search` 패턴을 정확 매칭 또는 fuzzy 매칭으로 찾는다.
3. 찾았으면 `replace`로 교체한다.
4. 매칭 실패, no-op 패치, 라인 번호 어긋남을 별도로 추적한다.

```rust
pub fn warrants_failure(&self) -> bool {
    // Checks if fuzzy match failures or noop deltas justify retry
}
```

이 메서드는 **에이전트에게 재시도 신호를 보내는 진입점**이다. Fuzzy 매칭이 너무 약하면 — 즉 LLM이 search 문자열을 잘못 만들었거나 컨텍스트가 stale이면 — 클라이언트가 서버에 "이 패치는 신뢰할 수 없으니 다시 만들어달라"고 요청한다. **"LLM의 출력은 검증되지 않은 가설"**로 다루는 보수적 설계다.

### Grep: ripgrep 서브프로세스

`crates/warp_ripgrep/src/search.rs`는 자체 ripgrep 래퍼다.

```rust
fn spawn_search_process(...) -> Result<async_process::Child, std::io::Error> {
    let mut cmd = command::r#async::Command::new(current_exe);
    cmd.arg(warp_cli::ripgrep_search_subcommand())
        .arg(warp_cli::parent_flag());
    // Parallel walk + regex matching
}
```

흥미로운 디테일: **별도 ripgrep 바이너리를 깔지 않고 자기 자신(`current_exe`)을 다시 호출한다.** Warp 바이너리에 `ripgrep-search` 서브커맨드가 박혀 있다. 이는 (1) 사용자 시스템에 ripgrep 설치를 강제하지 않고, (2) Cargo `grep` 크레이트를 직접 임베딩해서 wire 호환성을 보장한다. 그리고 `parent_flag`로 부모 PID를 monitor — 부모(Warp 프로세스)가 죽으면 grep 서브프로세스도 자동 종료된다. **고아 프로세스 방지** 패턴이다.

결과는 JSON 스트리밍으로 오고, 메인 프로세스가 파싱해 다음과 같이 구조화한다.

```rust
pub struct Match {
    pub file_path: PathBuf,
    pub line_number: u32,
    pub line_text: String,
    pub submatches: Vec<Submatch>,  // byte ranges of matched text
}
```

### File Outline: 임베딩 없이도 가능한 빠른 코드 이해

`crates/ai/src/index/file_outline/native.rs`는 tree-sitter로 파일에서 심볼만 뽑아 outline을 만든다.

```rust
pub struct Symbol {
    pub name: String,
    pub type_prefix: Option<String>,  // "fn", "class", etc.
    pub comment: Option<Vec<String>>,
    pub line_number: usize,
}
```

직접 LLM에게 노출되는 도구는 아니지만, 검색 결과에 메타정보를 첨부하거나 RAG 컨텍스트로 outline을 사용하는 경로가 있을 것으로 추정된다(`crates/ai/src/index/`에 위치한다는 것 자체가 단서). **"임베딩 없이 가능한 빠른 코드 구조 이해"** — 모든 RAG가 임베딩으로 풀려고 하지 않아도 된다는 좋은 기억재생기.

## 설계 차원에서 본 통찰

코드를 다 읽고 나니 몇 가지 강한 의견이 생겼다. 순서대로 풀어 본다.

**첫째, Merkle 트리가 정공법이다.** 여러 사내 프로젝트에서 codebase RAG를 시도해 본 적이 있는 엔지니어라면 모두 한 번씩 마주쳤을 문제 — "파일이 바뀌었는지 어떻게 효율적으로 알지?"에 대한 답은 결국 (1) 파일 단위 해시, (2) 트리 단위 해시 두 가지로 수렴한다. Warp가 둘 다 한다. Fragment 해시로 임베딩 캐시 키를, 디렉토리 해시로 동기화 차이를 풀었다. 기성 솔루션 — Sourcegraph zoekt의 commit-based 인덱싱, Cursor의 자체 indexer — 도 비슷한 방향이지만, **Merkle 트리를 명시적으로 메인 자료구조로 두고 BFS 동기화를 하는 패턴**은 학습 가치가 크다. AGPL이라 그대로 못 베껴도 설계는 카피프리다.

**둘째, "Agentic search"라는 단어와 실제 구현의 거리.** 코드에는 그 단어가 없다. 있는 건 `SearchCodebase` 도구 하나와, 그 도구를 부를지 말지 결정하는 서버측 LLM이다. 마케팅에서 "agentic"이라는 형용사가 약속하는 것 — "에이전트가 알아서 검색을 결정한다" — 은 결국 **모델이 함수 콜을 할 줄 안다**는 사실의 재포장이다. 이게 나쁘다는 게 아니다. 오히려 깔끔하다. 자동 컨텍스트 주입(Cursor 초기형, Copilot Chat 일부 모드)보다 디버깅하기 쉽고 토큰 예산을 모델이 직접 관리한다. 다만 우리는 이걸 더 정직하게 부를 필요가 있다 — **"function calling으로서의 코드 검색"**, 또는 더 간결하게 **"검색을 도구로"**.

**셋째, 클라이언트와 서버의 책임 분리가 매우 단호하다.** 클라이언트는 (1) Merkle 트리 유지, (2) 도구 실행, (3) 권한 게이팅을 한다. 서버는 (1) 임베딩 생성, (2) 벡터 검색, (3) 리랭킹, (4) LLM 추론, (5) 에이전트 루프 — 즉 **돈이 드는 모든 것**을 한다. AGPL로 클라이언트만 공개한 결정이 합리적인 이유가 여기 있다. **클라이언트 코드의 가치는 ~10%이고, moat는 서버측 모델·인프라·평가 데이터에 있다.** OSS화는 커뮤니티 기여를 받아 클라이언트 품질을 끌어올리고, 동시에 서버는 닫아둠으로써 비즈니스 모델을 보존한다. **앞으로 AI 도구 회사들이 따라야 할 OSS 전략의 모범**이라 본다.

**넷째, 다른 에이전트들의 skill·rules 형식을 흡수하려는 의지.** 10개 SkillProvider, `WARP.md`와 `AGENTS.md` 동시 지원. 이건 단순한 호환성 제스처가 아니다. **"이 분야는 곧 표준화될 것이고, 우리는 가장 폭넓게 흡수하는 클라이언트가 되겠다"**는 베팅이다. AGENTS.md를 표준화하려는 흐름은 이미 OpenAI Codex, GitHub 등에서도 보이는데, Warp가 가장 적극적이다. 이 베팅이 맞다면 — 즉 사용자들이 도구 락인을 거부하고 portable한 prompt artifact를 선호한다면 — Warp는 큰 보상을 받는다.

**다섯째, 검증 레이어의 존재가 성숙한 신호다.** `diff_validation`, `is_risky` 플래그, `last_server_synced_root_node`. 이 모든 게 **"LLM의 출력은 검증되지 않은 가설"**이라는 전제를 코드에 반영한다. 초기 코드 에이전트들은 모델 출력을 그대로 실행했다. 다음 세대는 검증 레이어를 표준화한다. Warp는 이미 그 단계에 있다. 우리가 만드는 에이전트도 같은 길을 가야 한다.

**여섯째, 4MB Cloud Armor 한도, REINDEX_INTERVAL 20분, RETRIEVE_FRAGMENT_CONTEXT_LENGTH=0 같은 상수들.** 이런 숫자들은 학회 논문에 안 실린다. 이 숫자들이 기록된 이유는 **운영 중에 누군가 디버깅하다 알아냈기 때문**이다. 우리가 RAG 시스템을 직접 만든다면 같은 발견들을 비슷한 시간에 우리도 마주칠 것이다. Warp가 공개해준 덕에 그 발견을 일부 건너뛸 수 있다는 건 적지 않은 선물이다.

## 결론

Warp의 OSS 클라이언트는 코드 인지 AI를 만드는 사람이라면 한 번은 정독할 가치가 있다. RAG 측면에서는 **Merkle 트리 + SHA-256 + 시멘틱 청킹 + 4MB 배치 동기화**라는 정공법의 견본이다. Agentic 측면에서는 **"검색은 도구다"**라는 철학과, 클라이언트를 도구 실행자로, 서버를 결정자로 명확히 가르는 구조의 좋은 예다.

물론 이 글이 다루지 못한 부분이 많다. 임베딩 모델이 무엇인지, 리랭커가 어떤 모델인지, 벡터 DB가 무엇인지 — 이건 서버에 있고 비공개다. 추측은 가능하지만 추측은 글에서 제외했다. 또한 `crates/computer_use/`, `crates/onboarding/`, `crates/voice_input/` 같은 다른 흥미로운 모듈도 다루지 않았다. 다음에 시간이 되면 별도 글로 풀어볼 가치가 있다.

마지막으로 한 가지 실용적인 권고. **자기 코드베이스에 RAG를 붙이려고 한다면, Cursor·Continue 위에 얹지 말고 Warp의 인덱싱 구조 — Merkle 트리, 배치 정책, 상태 머신 — 를 그대로 베껴라.** AGPL을 직접 차용할 수 없다 해도 설계는 자유다. 1년쯤 운영하다가 재발견할 패턴들을 미리 가져갈 수 있다.

---

## 참고

- [warpdotdev/warp](https://github.com/warpdotdev/warp) — 분석 대상 저장소 (AGPLv3 / 일부 MIT)
- [Warp is now open source](https://www.warp.dev/blog/warp-is-now-open-source) — 공식 발표
- [warpdotdev/docs](https://github.com/warpdotdev/docs) — 사용자 문서 (별도 저장소)
- 본문에 인용된 코드 라인 번호와 상수 값은 분석 시점(2026-05-03)의 main 브랜치 기준이다. 이후 리팩토링되었을 수 있다.
