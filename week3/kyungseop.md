# Week 3 - kyungseop

## 아웃풋 목표

> week2에서 설계한 파이프라인을 실제로 돌아가는 상태까지 올리기

- 매일 쌓이는 Claude Code 세션, 커밋 자동 수집 및 Discord DM을 통해 직접 아이디어 시드 제공 → 시드 추출 → Notion 등록 → Discord 알림까지 자동화
- Discord DM으로 시드 선택 → Outliner → Writer → Notion 초안 저장까지 봇으로 처리
- 최종 발행처: [kskim.dev](https://kskim.dev)

## 파이프라인 설계

> 전체 흐름 (✅ 구현 완료 / 🔲 미구현)

```mermaid
flowchart LR
    A["✅ 수집\n(세션, 커밋,\n직접입력)"] --> B["✅ 시드 처리\n(Sonnet)"]
    B --> C["✅ Blog Seeds DB"]
    C --> N1["✅ Discord 알림"]
    N1 --> D["✅ 주제 선택\n(사람)"]
    D --> E["✅ 개요 + 초안 작성\n(Sonnet)"]
    E --> F["✅ 블로그 DB\n(상태: 초안)"]
    F --> N2["✅ Discord 알림"]
    N2 --> G["✅ 검토\n(사람)"]
    G -->|Rewriter| E
    G -->|Publish| H["🔲 발행\n(Sonnet)"]
    H --> I["🔲 kskim.dev PR"]
```

- **수집 소스**: Claude Code 세션 로그, Git commits, Discord DM 직접 입력
- **사용 툴/프레임워크**: Claude Code (Sonnet), Notion API (httpx 직접), Discord 봇 (discord.py), launchd
- **발행 채널**: kskim.dev (🔲 미구현)

### 에이전트 구성

| 에이전트 | 트리거 | 역할 |
|---------|--------|------|
| Seed Processor | nightly launchd (매일 23시) | 세션 로그 + 커밋 → 시드 추출 + Notion 등록 |
| Outliner | Discord DM (`초안 써줘`) | 시드 + 방향 메모 → 구조화된 개요 |
| Writer | 개요 완성 후 | 개요 → 블로그 초안 |
| Publisher | 발행 결정 시 (🔲 미구현) | Notion → MDX + PR 생성 |

### week2 대비 설계 변경

- **Claude CLI + MCP → httpx 직접 호출**: launchd 환경에서 MCP 인증에 필요한 환경변수(`OPENAPI_MCP_HEADERS`)가 상속이 안 됨. 어차피 Notion API가 REST라서 직접 호출하는 게 더 단순하고 빠름
- **python3 -c 인라인 → 외부 스크립트 분리**: bash `$()` 안에서 백틱(` ``` `)이 명령어로 해석되는 버그 때문에 `lib/parse_seeds.py`로 분리

## 이번 주 진행 내용

- Outliner/Writer 에이전트 프롬프트 작성 후 Discord 봇에 `write_handler` 붙임
  - DM으로 `캐싱 초안 써줘` 보내면 → Notion 시드 조회 → Outliner(Sonnet) → Writer(Sonnet) → Notion 블로그 DB에 `상태: 초안`으로 저장 → Discord 알림
  - 쉼표로 방향 메모 추가 가능: `Redis 초안 써줘, 삽질 위주로`
- Discord 봇이랑 nightly seed-processor를 launchd에 올렸는데 둘 다 처음엔 안 됨
  - 봇은 Desktop 폴더 접근이 launchd에서 막혀 있었고, nightly는 MCP 인증 실패에 bash 버그에 경로 버그까지 3개가 한꺼번에 나왔음
  - 하나씩 고쳐서 결국 nightly 실행 성공: 시드 4개 추출 + Notion 등록 + Discord 알림 확인

## 이후 개선 사항

- **Discord DM 봇 대화 히스토리** — 매 메시지가 독립 프로세스(`claude --print`)라 이전 대화를 전혀 몰랐음. Discord 채널 히스토리 최근 10개를 읽어서 프롬프트에 포함하는 방식으로 해결
  - 토큰 낭비 문제: 10개 원문을 그대로 Sonnet에 넘기는 건 비효율적. 그래서 Haiku로 1~2문장 요약 후 Sonnet에 전달
- **목록 조회 시 Notion 링크** — `page["url"]`이 API 응답에 이미 있었는데 item에 안 담고 있었음. 추가해서 Discord에서 바로 클릭 가능
- **웹훅 알림 개선**
  - 시드 제목 목록 미리보기 추가 — 몇 개 추출됐는지만 알려주던 것에서 제목 전체 나열로 변경
  - 실패 알림 추가 — `trap ERR` + `CURRENT_STEP` 변수로 어느 단계에서 실패했는지 Discord로 전송
  - JSON 인젝션 수정 — `$message`를 JSON에 직접 박던 방식 → python3 `json.dumps`로 안전하게 직렬화

## 구현 중 막힌 것 / 해결한 것

| 문제 | 해결 여부 | 메모 |
|------|-----------|------|
| launchd에서 Desktop 폴더 접근 `Operation not permitted` | ✅ | python3 바이너리에 FDA 직접 부여가 안 됨 → `/bin/zsh`에 부여 후 zsh 래퍼로 실행 |
| Claude CLI + MCP가 launchd에서 인증 실패 | ✅ | `OPENAPI_MCP_HEADERS` 미상속 → Notion API httpx 직접 호출로 교체 |
| `discord.sh` source 후 `SCRIPT_DIR`이 `lib/`로 덮어써짐 | ✅ | source 전에 `PIPELINE_DIR`로 따로 저장 |
| python3 -c 스크립트 안 백틱이 bash에서 명령어로 해석됨 | ✅ | `lib/parse_seeds.py` 외부 스크립트로 분리 |
| launchd가 macOS symlink를 실제 경로로 추적 | ✅ | 심링크 우회 시도했다가 안 됨 확인. zsh FDA 방식으로 해결 |
| Discord DM 봇이 이전 대화를 모름 | ✅ | Discord 채널 히스토리 읽어서 Haiku 요약 후 Sonnet에 전달 |
| 리라이트 루프 실제 품질 | 미검증 | 검증 예정 |

## 인사이트 / 배운 것

- **launchd는 터미널이랑 다른 환경이다.** 터미널에서 멀쩡히 돌던 스크립트가 launchd에서 안 되는 경우가 많은데, 이번에 나도 만났다. 환경변수 상속 안 됨, Desktop 폴더 TCC 제한, symlink 실제 경로 추적까지. 진화 누나도 비슷한 문제를 겪었던 거 같은데, launchd 자동화는 항상 터미널 밖에서도 되는가를 따로 검증해야 되는 거 같다.
- **설계에서 안 보이던 버그가 실행해보니 다 나왔다.** 2주차에서 `seed-processor.sh`를 다 짜놓고 완성됐다고 생각했는데, 실제로 돌려보니 버그 3개가 한꺼번에 나왔다. bash source의 전역 네임스페이스, python3 -c 안의 백틱, MCP 환경변수 상속 — 전부 코드 읽어서는 잡기 어려운 것들이었다. 빠르게 실패를 경험하는 전략이 필요하다.
- **컨텍스트 비용을 의식해야 한다.** 히스토리 10개를 그대로 Sonnet에 넘기는 건 매 메시지마다 수천 토큰 낭비다. Haiku로 요약하면 Sonnet 입력 토큰을 크게 줄이면서 비용 대비 효과가 훨씬 좋다. 100불의 가치를 소중히 하자.

## 다음 주 계획

- `rewrite_handler` 구현 — 피드백 메모 남기면 Writer가 다시 쓰는 루프
- Publisher 에이전트 — 테스트
- Qwen3 VL OCR 게시물 초안 검토 + 발행
