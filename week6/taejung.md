# 1~5주차 회고 - taejung (허태정 / Aqudi)

## 1. 목표와 진행 과정

### 처음 세웠던 목표

평소 공부하거나 일하면서 배운 것·느낀 것에서 **글감을 자동으로 추출**하고, 이를 바탕으로 **글을 쓰는 과정을 도와주는 도구**를 만드는 것이 출발점이었다.

### 진행 흐름

1. **자동 작업 기록부터 시작 (Week 1~2)**
   - "내가 뭘 하는지"를 자동으로 남기는 게 먼저라고 판단.
   - Obsidian에 자동화 기능을 붙여 **Claude Code 세션 로그를 LLM으로 요약 → 데일리 업무일지**를 작성하는 파이프라인을 구축.
   - 1주차 요약은 너무 피상적이어서, 2주차에는 git log를 추가하고 사용자 입력 비중을 높여 "내가 뭘 했는지/왜 했는지"가 드러나도록 데이터·프롬프트를 개선.

2. **노트 정리 방법론 적용 (Week 3 전반)**
   - 쌓이는 노트를 효과적으로 관리하기 위해 **Andrej Karpathy의 LLM Wiki 방법론**을 적용.
   - `wiki-tools` Claude Code 플러그인을 직접 만들어 ingest / tag-classify / lint / query 4개 스킬로 옵시디언 vault를 구조화. 원본은 사람 영역, Wiki는 AI 영역으로 경계를 명확히 분리.

3. **블로그 발행 자동화로 확장 (Week 3 후반 ~ Week 4)**
   - 처음에는 LLM이 쓴 글을 기존 블로그(`blog.aqudi.me`)에 바로 올릴 생각이었지만, **아예 처음부터 자동 발행을 전제로 한 별도 공간**을 만드는 쪽으로 방향 전환.
   - 평소 써보고 싶었던 디지털 가든 템플릿을 활용해 노트 저장소를 새로 셋업.
   - **해커뉴스/긱뉴스 RSS 수집 → 트렌드 분석 → 주제 선정 → 리서치 → 초안 작성 → 리뷰** 일련의 과정을 수행하는 **CLI 도구(`blog-automation`)** 를 구현.

4. **실사용 가능한 수준으로 마감 (Week 5)**
   - Quartz v4로 `Aqudi/notes` 디지털 가든 셋업, AI 글쓰기 프롬프트에 `[[wikilink]]` 자동 삽입 가이드라인 추가.
   - 스케줄러를 GitHub Actions에서 **로컬 Mac Mini의 launchd**로 변경(항상 켜진 머신 활용 + 무료 플랜 제약 회피).
   - **fetch → analyze → draft → revise → publish** end-to-end Dry-run 검증 완료.
   - 이후 웹 GUI와 Obsidian 노트를 소스로 붙이는 기능을 추가 작업 중.

### 어려웠던 점 / 예상과 달랐던 부분

- **바이브코딩으로 완성도를 끌어올리는 게 생각보다 어려웠다.** 특히 UI를 "이쁘게" 만드는 게 가장 큰 벽이었고, `ui ux pro max` 스킬의 도움을 많이 받았다.
- **LLM의 치팅 문제.** 자료를 조사한 뒤 그걸 기반으로 글을 쓰게 시키면, 베이스 자료를 단순 요약/번역하는 수준에 그치는 경우가 많았다. 프롬프트 작성에 더 신경 써야 한다는 걸 체감. 처음에는 "결국 사람 검수가 필요하겠다"고 생각했지만, 정신 차리고 **이것조차 LLM이 리뷰하게 만드는 방향**(원문과 초안의 유사도 검사 등)으로 개선 중.
- **데이터 품질이 결과 품질을 지배한다.** 1주차 → 2주차 요약 개선, RSS 소스 큐레이션 → 글 품질 향상 모두 같은 패턴이었다. 결국 파이프라인 첫 단계가 전체를 결정한다.

### 공유할 만한 인사이트

- **인간 영역 / AI 영역을 명확히 분리하면 작업이 훨씬 편해진다.** wiki-tools에서 "원본은 사람, Wiki는 AI", 블로그에서 "초안은 AI, 리뷰는 사람"으로 경계를 그으니 마음 편하게 위임할 수 있었다.
- **AI에게는 AI가 잘하는 일을 시켜야 한다.** 요약, 분류, 점검처럼 사람이 귀찮아서 안 하는 작업을 LLM은 불평 없이 해준다 ("LLM은 따분함을 모른다" - Karpathy).
- **AI 작업에는 판단 근거 로그가 필수.** 결과물만 보면 왜 그런 선택을 했는지 알 수 없어 개선이 어렵다. 그래서 각 단계마다 `.trace.json`을 남기도록 설계.
- **Z.AI의 GLM 5.1이 한국어 글쓰기 품질이 생각보다 괜찮다.** 단, max token을 너무 짧게 잡으면 글을 쓰다 만다.

---

## 2. 완성한 파이프라인의 기술 스택

### 전체 흐름

```mermaid
flowchart LR
    A[launchd 매일 09:00] --> B[RSS 수집<br/>HN Best · GeekNews]
    B --> C[트렌드 분석<br/>주제 후보 추출]
    C --> D[리서치 + Draft 작성<br/>wikilink 자동 삽입]
    D --> E[Revise<br/>피드백 반영]
    E --> F[Publish<br/>Aqudi/notes 푸시]
    F --> G[GitHub Pages<br/>Quartz v4 자동 배포]
    D -.trace.json.-> H[(판단 근거 로그)]
```

### 기술 스택

| 영역 | 사용 기술 |
| --- | --- |
| **CLI / 코어** | Python 3.11+, `click`, `pydantic`, `rich`, `feedparser` |
| **LLM 연동** | `anthropic`, `openai`, Codex CLI, Z.AI(GLM 5.1) — provider 교체 가능한 구조 |
| **웹 GUI** | FastAPI + Uvicorn (백엔드), Vite 기반 frontend, `openapi-typescript`로 타입 자동 생성 |
| **노트 저장소 / 발행** | Quartz v4 디지털 가든(`Aqudi/notes`), GitHub Pages 배포, `[[wikilink]]` + 백링크 + 그래프 뷰 |
| **저장소 연동** | `PyGithub`, `gh` CLI (`gh api ... --method PUT`로 글 업로드) |
| **스케줄링** | macOS `launchd` (Mac Mini 24/7), `pm2` (`ecosystem.config.cjs`로 웹 데몬 관리) |
| **알림 / 휴먼 인 더 루프** | Discord Webhook (예정), Happy Code 통한 로컬 리뷰 흐름 검토 중 |
| **패키지 관리** | `uv` (Python), `npm` (frontend) |
| **배포 환경** | 로컬 Mac Mini + Tailscale |

### 주요 CLI

```bash
uv run blog-automation fetch     # RSS 수집
uv run blog-automation analyze   # 트렌드 분석 → 주제 후보
uv run blog-automation draft "<주제>"      # 리서치 + 초안 작성
uv run blog-automation revise <파일> --feedback "<피드백>"
uv run blog-automation publish   # Aqudi/notes 푸시 → Pages 자동 배포
uv run blog-automation run       # 전체 파이프라인 한 번에
```

### 주변 도구

- **`wiki-tools` Claude Code 플러그인**: `wiki-ingest`, `tag-classify`, `wiki-lint`, `wiki-query` 4개 스킬로 옵시디언 vault를 LLM Wiki 방법론으로 운영.
- **`session-summary` 스킬**: Claude Code 세션 + git log를 묶어 옵시디언 데일리 노트에 자동 적재.

---

## 3. 시각 자료

> 추후 추가 예정 — 플레이스홀더

- [ ] CLI 실행 화면 (fetch → analyze → draft 흐름)
- [ ] 웹 GUI 대시보드 스크린샷
- [ ] Quartz v4 디지털 가든 (생성된 글 + 그래프 뷰)
- [ ] Discord 알림 데모 GIF
- [ ] launchd 자동 실행 → 발행까지의 end-to-end 동영상

---

## 부록: 주차별 산출물 한 줄 정리

| 주차 | 핵심 산출물 |
| --- | --- |
| Week 1 | 파이프라인 4단계(수집→정제→학습→산출물) 설계, Claude Code 세션 → Obsidian Daily Note 적재 스킬 |
| Week 2 | `session-summary` 리팩터링(데이터 품질·프롬프트 개선), 옵시디언 버튼 자동화, headless CLI 노하우 |
| Week 3 | LLM Wiki 방법론 적용(`wiki-tools` 플러그인), `blog-automation` CLI 초기 버전 |
| Week 4 | 블로그 자동화 파이프라인 안정화 및 provider 추상화 |
| Week 5 | Quartz v4 디지털 가든 셋업, launchd 스케줄러, end-to-end Dry-run 검증, wikilink 자동 삽입 |
