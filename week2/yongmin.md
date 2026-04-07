# Week 2 - Yongmin

## 아웃풋 목표

Slack DM으로 일상의 텍스트/이미지/URL을 던지면, AI가 자동으로 **일기 / 일정 / 인사이트** 중 하나로 분류하여 각각의 저장소(Notion, Google Calendar 등)에 자동 기록해주는 **개인 AI 비서 시스템** 구현

## 파이프라인

```
Slack DM (텍스트 / 이미지 / URL)
    ↓
API Gateway (AWS)
    ↓
Lambda + FastAPI
    ↓
UnifiedInput 파싱 (콘텐츠 유형 판별: text / url / image / mixed)
    ↓
LangGraph 라우팅
    ├── [image 단독] → Vision 분석 (Claude Sonnet) → Intent 분류 (Claude Haiku) → END
    └── [text / url / mixed] → Intent 분류 (Claude Haiku)
                                    ↓ [needs_vision=True인 경우]
                               Vision 분석 (Claude Sonnet) → END
    ↓
Intent 결과 (diary / schedule / insight)
    ↓
(다음 단계) Notion / Google Calendar / Notion DB 저장
```

## 이번 주 진행 내용

### 1. 기술 스택 재검토 및 변경

프로젝트 문서 전체를 분석하고, 초기 스펙 대비 아래 항목들을 실용적 이유로 변경하였다.

| 항목        | 변경 전             | 변경 후                         | 이유                               |
| ----------- | ------------------- | ------------------------------- | ---------------------------------- |
| 패키지 관리 | Poetry              | `uv`                            | 10~100배 빠름                      |
| LLM 벤더    | Claude + GPT-4o     | Anthropic 단일 (Sonnet + Haiku) | API 키 이원화 복잡도 제거          |
| Slack 연동  | Socket Mode         | Webhook + API Gateway           | 비동기 구조에 Socket Mode 부적합   |
| 클라우드    | GCP Cloud Run       | AWS Lambda                      | 이벤트 기반 구조에 적합, 무료 티어 |
| 운영 DB     | Notion (주)         | Supabase + Notion (뷰어)        | Notion API Rate Limit 문제         |
| Vector DB   | ChromaDB / Pinecone | pgvector                        | 별도 인프라 불필요                 |

### 2. FastAPI 서버 + Slack Webhook 연동 (Priority 1)

- FastAPI 기반 서버 구조 설계 및 구현 (`main.py`, `config.py`, `api/slack.py`, `services/slack_handler.py`)
- Mangum으로 FastAPI ↔ AWS Lambda 어댑터 연결
- Slack 서명 검증 (`X-Slack-Signature`) 및 URL Verification 핸드셰이크 처리

### 3. AWS 인프라 구성

Docker 이미지 빌드 → ECR 푸시 → Lambda 생성 → API Gateway 순으로 구성하였으며, 세 가지 트러블슈팅이 있었다.

- **Docker 멀티플랫폼 이슈:** Apple Silicon에서 빌드 시 manifest list가 생성되어 Lambda가 거부함 → `--provenance=false` 옵션으로 해결
- **Lambda Function URL 403 Forbidden:** 신규 AWS 계정의 "Block Public Access" 기본 활성화로 퍼블릭 접근 차단 → Function URL 포기, **API Gateway HTTP API**로 교체
- **`asyncio.create_task()` 미작동:** Lambda는 응답 반환 즉시 실행 환경을 동결하므로 백그라운드 태스크가 실행되지 않음 → `await handle_event()` 직접 호출로 변경

**최종 엔드포인트:** `https://0q2m2z0sni.execute-api.ap-northeast-2.amazonaws.com/api/slack/events`

### 4. UnifiedInput 통합 데이터 모델 (Priority 2)

모든 유입 소스(Slack, Android 등)를 단일 스키마로 정규화하는 `UnifiedInput` 모델 구현.

- `InputSource`: `slack` / `android`
- `ContentType`: `text` / `url` / `image` / `mixed` (자동 판별)
- `Intent`: `diary` / `schedule` / `insight` / `unknown` (LangGraph가 채움)
- URL 자동 추출, 이미지 URL 파싱, 시스템 이벤트 필터링(`message_changed`, `bot_message` 등) 포함

**트러블슈팅:**

- `KeyError: 'user'` → `event.get("user", "unknown")`으로 처리 + `SKIP_SUBTYPES` 화이트리스트 필터링
- Lambda에서 `logger.info` 미출력 → `logging.basicConfig(force=True)`로 강제 재설정

### 5. LangGraph 3-Way Intent 라우팅 (Priority 3)

- `classify_node`: `claude-haiku-4-5` 호출, `max_tokens=10`, `temperature=0`으로 비용/속도 최소화
- 단일 노드 그래프(`classify → END`)로 시작, 추후 intent별 처리 노드 확장 예정

**실제 동작 확인:**

| 입력                                       | intent       |
| ------------------------------------------ | ------------ |
| "오늘 낮잠잠 개꿀"                         | `diary` ✅    |
| "이번주 금요일에 회사 점심 약속"           | `schedule` ✅ |
| YouTube URL + "이거 재밌겠다 나중에봐야지" | `insight` ✅  |

### 6. Vision 분석 노드 추가 (Priority 3 확장)

카톡 일정 캡처처럼 이미지만으로 의미가 전달되는 케이스를 처리하기 위해 Vision 노드 추가.

**트러블슈팅:**

- `langchain_anthropic`이 `image_url` 타입 미지원 → Anthropic 네이티브 base64 포맷으로 변경
- `url_private`이 HTML 반환 → `url_private_download` 사용으로 변경
- `files:read` 스코프 누락 → api.slack.com에서 추가 후 Reinstall
- Haiku가 JSON을 마크다운 코드블록으로 감싸서 반환 → `re.sub`으로 제거 후 파싱

## 구현 중 고민했던 것 / 막힌 것

### intent(일기, 일정, 인사이트) 분류 시 이미지 분석 여부

짬통에 이미지를 던질 때, 이미지만 던지기, 이미지와 텍스트 함께 던지기 두 가지 경우가 가능하다. 이때 이미지와 텍스트를 함께 던지는 경우 intent(일기, 일정, 인사이트)를 분류하는데 있어서 이미지 분석 여부를 결정하는 플로우를 고민했다. 처음에는 무조건 이미지 분석을 돌리려고 했는데, 그럼 토큰 사용량이 굉장히 올라갈 것 같아서 최대한 텍스트를 활용하여 분류하고, 이미지 분석이 필요한 경우 해당 분석 내용을 이후에도 재활용하도록 변경했다. e.g.)

- 이미지 + "그대로, 오늘 발견한 옛날 사진~" → 이미지 분석 없이 텍스트만으로 intent를 일기로 분류하고 일기에 그대로 이미지 삽입
- 이미지(카톡 캡쳐) + "이번주 일정 추가" → 텍스트만으로 intent를 일정으로 분류하고 일정 관리쪽에서 이미지를 분석
- 이미지(카톡 캡쳐)만 → 이미지를 분석하여 intent를 일정으로 분류하고 이미지를 두번 분석하지 않도록 해당 정보를 함께 넘겨 재활용하도록 한다.

### AI 앞에 무력감

지금까지 클로드 딸깍 딸깍으로 작업했다. 나는 안드로이드 및 모바일 멀티플랫폼 위주의 개발을 해왔기에 파이썬 등을 제대로 다루는게 처음이다. 그래서 지금까지는 1번만 딸깍 누르며 진행해왔는데, 너무 무력감을 느껴서 다음주부터는 구현된 파일을 보고 내가 코드를 전부 이해하고 넘어가보려고한다.

## 다음주 계획

- 일정 관리 부분 구현하기
- 일기 작성 부분 구현하기

## p.s.

석범이의 습관 발표에 감명받았다. 이번주에 너무 작업을 안한 것 같다. 나도 퇴근 후 작업하기를 습관화해봐야겠다.