# Week 3 - kyungseop

## 이번 주 진행 내용

week2에서 설계한 파이프라인을 실제로 구현하고 launchd로 자동화까지 완성했다.

### 1. Outliner / Writer 에이전트 구현

- `pipeline/prompts/outliner.md` — 시드 → 구조화된 개요
- `pipeline/prompts/writer.md` — 개요 → 블로그 초안
- Discord DM에서 `캐싱 초안 써줘` 입력 → Notion 블로그 DB에 `상태: 초안`으로 저장까지 자동화
- 흐름: Notion 시드 조회 → Outliner(Sonnet) → Writer(Sonnet) → Notion 저장 → Discord 알림

### 2. launchd 자동화

두 개의 launchd 서비스 등록:

| 서비스 | 역할 |
|--------|------|
| `com.sup.blog-bot` | Discord 봇 상시 실행 (KeepAlive) |
| `com.sup.blog-nightly` | 매일 23시 시드 추출 + Notion 등록 + Discord 알림 |

`~/Library/LaunchAgents/`에 등록해뒀으니 재부팅 후에도 유지됨.

### 3. seed-processor.sh 버그 수정

nightly를 실제로 돌려보면서 버그 3개 발견 + 수정:

1. **`SCRIPT_DIR` 오염**: `discord.sh`를 source할 때 변수가 덮어써짐 → source 전에 `PIPELINE_DIR`로 저장
2. **백틱 bash 해석**: python3 -c 인라인 스크립트 안의 `` ``` ``가 bash에서 명령어로 해석됨 → `parse_seeds.py` 외부 스크립트로 분리
3. **MCP 환경변수 미상속**: launchd는 쉘 환경변수를 상속 안 해서 `OPENAPI_MCP_HEADERS` 없음 → Notion API httpx 직접 호출로 교체

## 구현 중 막힌 것 / 해결한 것

| 문제 | 해결 여부 | 메모 |
|------|-----------|------|
| launchd에서 Desktop 폴더 접근 `Operation not permitted` | ✅ | `/bin/zsh`에 전체 디스크 접근 권한 부여 |
| Claude CLI MCP가 launchd에서 인증 실패 | ✅ | httpx 직접 호출로 전환 |
| python3 바이너리에 FDA 직접 부여 불가 | ✅ | zsh 래퍼로 우회 |
| nightly 실제 실행 결과: 4개 시드 추출 + Notion 등록 완료 | ✅ | |

## 인사이트 / 배운 것

- **launchd는 쉘 환경변수를 상속하지 않는다.** 터미널에서 동작하던 게 launchd에서 안 되는 이유가 대부분 이거다. 비밀키나 토큰을 환경변수로 쓰는 게 있다면 plist의 `EnvironmentVariables`나 config 파일로 직접 주입해야 함.
- **macOS TCC는 symlink를 실제 경로로 추적한다.** `~/.local/share/blog-bot → ~/Desktop/...` 심링크를 만들어도 소용없었다. Desktop 접근 권한이 필요하면 해당 바이너리(또는 셸)에 직접 FDA를 줘야 함.
- **bash 스크립트에서 source는 전역 네임스페이스를 오염시킨다.** `SCRIPT_DIR` 같은 흔한 이름은 라이브러리 파일에서도 쓸 수 있으니, 중요한 경로 변수는 source 전에 따로 저장해두는 게 안전함.

## 다음 주 계획

- `rewrite_handler` 구현 — 피드백 반영 재작성 루프
- Publisher 에이전트 — Notion 초안 → MDX 변환 + kskim.dev PR 생성
- Qwen3 VL OCR 블로그 초안 검토 + 발행
