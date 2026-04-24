# Week 5 - hongbi

## 아웃풋

> 기술 대화 중심이던 파이프라인에 **경험 후기 트랙** 추가 — 캘린더 일정 종료부터 블로그 초안까지 완전 자동화

- Google Calendar 일정 종료 시 Discord 스레드 자동 생성 (Apps Script + GitHub Actions)
- Discord 후기 메시지 + 이미지 수집 → Supabase Storage 영구 보관
- 경험 후기 전용 파이프라인 분리 (가치 판별·클러스터링 없이 바로 글 생성)
- GitHub Actions 3개 Job 구조로 개편 + Job 실행 순서 보장

---

## 배경 — 왜 경험 후기 트랙이 필요했나

기존 파이프라인 소스:

- Claude Code 대화 로그 → 기술 지식 위주
- Notion 글 → 기술 정리 위주

직접 가본 레퍼런스 방문, 전시, 스터디 후기 같은 **개인 경험 글**은 소스가 없었음.
경험은 기술 대화처럼 로그로 남지 않으니 새로운 입력 경로가 필요했음.

**Discord 스레드를 경험 후기 입력창으로 선택한 이유:**

- 이미 사용 중인 플랫폼 → 별도 앱 필요 없음
- 이미지 첨부가 자연스러움
- 스레드 단위로 일정별 분리가 깔끔함

---

## 구현 1 — 캘린더 일정 종료 → Discord 스레드 자동 생성

### 전체 흐름

```
캘린더 일정 등록
  → Apps Script: onCalendarEventCreated 감지
  → 일정 종료 시각에 time-based trigger 예약
  → 일정 종료 시: GitHub Actions repository_dispatch
  → discord-notify job: Discord 스레드 생성 + experience_threads 저장
```

### 왜 Apps Script → GitHub Actions를 거쳤나

Apps Script에서 Discord API를 직접 호출하면 `40333 internal network error` 발생.
Google Apps Script 서버 IP 대역이 Discord API에서 차단되어 있음.

**우회 방법:** Apps Script → GitHub API(`repository_dispatch`) → GitHub Actions → Discord API

```javascript
// apps-script/calendar-discord.gs
function triggerGithubActions(title) {
  UrlFetchApp.fetch(`https://api.github.com/repos/${GITHUB_REPO}/dispatches`, {
    method: "post",
    headers: { Authorization: `token ${GITHUB_TOKEN}`, ... },
    payload: JSON.stringify({
      event_type: "calendar-event-ended",
      client_payload: { title },
    }),
  });
}

function onCalendarEventCreated(e) {
  const endTime = new Date(e.calendarEventUpdated.endTime);
  ScriptApp.newTrigger("sendDiscordThread")
    .timeBased()
    .at(endTime)
    .create();
}
```

### discord-notify.ts

1. Discord 채널에 안내 메시지 전송
2. 해당 메시지에 스레드 생성
3. `experience_threads` 테이블에 저장 (수집 단계에서 참조용)

```typescript
// src/discord-notify.ts
// 1. 메시지 전송
const msg = await fetch(`${DISCORD_API}/channels/${channelId}/messages`, {
  body: JSON.stringify({
    content: `🗓️ **"${title}"** 일정이 끝났어~!\n후기를 이 스레드에 남겨줘!`,
  }),
});

// 2. 스레드 생성
const thread = await fetch(`.../${msg.id}/threads`, {
  body: JSON.stringify({ name: `${title} 후기`, auto_archive_duration: 10080 }),
});

// 3. Supabase에 저장
await supabase.from("experience_threads").upsert({
  event_title: title,
  discord_thread_id: thread.id,
  discord_channel_id: channelId,
});
```

---

## 구현 2 — Discord 메시지 수집 + 이미지 Supabase Storage 업로드

### collect-discord.ts

`experience_threads` 테이블의 스레드 목록을 순회하며 새 메시지를 수집.
Discord CDN 이미지 URL은 만료되므로 **Supabase Storage에 직접 업로드**해 영구 보관.

```typescript
// src/collect-discord.ts
async function uploadImageToSupabase(
  imageUrl: string,
  messageId: string,
): Promise<string | null> {
  const res = await fetch(imageUrl);
  const buffer = await res.arrayBuffer();
  const { error } = await supabase.storage
    .from("experience-images")
    .upload(filename, buffer, { contentType, upsert: true });

  const {
    data: { publicUrl },
  } = supabase.storage.from("experience-images").getPublicUrl(filename);
  return publicUrl;
}
```

수집된 메시지는 `experiences` 테이블에 저장:

```typescript
await supabase.from("experiences").upsert(rows, {
  onConflict: "discord_message_id",
  ignoreDuplicates: true,
});
```

---

## 구현 3 — 경험 후기 전용 블로그 생성 파이프라인

### 기술 대화 vs 경험 후기 파이프라인 비교

| 항목              | 기술 대화 (Claude 로그 + Notion) | 경험 후기 (Discord)         |
| ----------------- | -------------------------------- | --------------------------- |
| 가치 판별 (Haiku) | ✅ yes/no 필터링                 | ❌ 없음                     |
| 인사이트 추출     | ✅ Haiku 병렬                    | ❌ 없음                     |
| 클러스터링        | ✅ Sonnet 1회                    | ❌ 없음                     |
| 품질 점수 필터링  | ✅ 3점 미만 스킵                 | ❌ 없음                     |
| 글 생성           | Sonnet (클러스터당)              | Sonnet (경험 메시지 묶음당) |

**경험 후기에 필터링을 두지 않은 이유:**
기술 대화 파이프라인의 `extractInsights`는 10턴 이상, 문제 해결 과정 등 기준을 봄.
짧은 후기 텍스트는 항상 필터링에서 탈락함 → 별도 경로 필요.

### summarize.ts 분기 처리

```typescript
// 경험 후기 vs 기술 대화 분리
const experienceConvs = conversations.filter((c) =>
  c.projectName.startsWith("discord-experience-"),
);
const techConvs = conversations.filter(
  (c) => !c.projectName.startsWith("discord-experience-"),
);

// 경험 후기: 필터링 없이 바로 글 생성
const experienceDrafts = await Promise.all(
  experienceConvs.map((c) => generateFromExperience(c)),
);

// 기술 대화: 기존 3단계 파이프라인
const techInsights = await extractInsights(techConvs);
const clusters = await clusterInsights(techInsights, existingTitles);
const techDrafts = await Promise.all(
  clusters.filter((c) => c.qualityScore >= 3).map(generateFromCluster),
);
```

---

## 구현 4 — GitHub Actions 3개 Job 구조

### 기존 → 개편

|             | 기존             | 개편                              |
| ----------- | ---------------- | --------------------------------- |
| Job 수      | 1개 (generate만) | 3개                               |
| 경험 수집   | 없음             | collect job (매시간)              |
| 스레드 생성 | 없음             | discord-notify job (event-driven) |
| Job 순서    | 독립 실행        | collect → generate (needs)        |

### Job 구조

```yaml
jobs:
  # 캘린더 일정 종료 시 (event-driven)
  discord-notify:
    if: github.event_name == 'repository_dispatch'

  # 매시간 or 수동 (job=collect|all)
  collect:
    if: schedule || workflow_dispatch

  # 매주 월요일 or 수동 (job=generate|all)
  generate:
    needs: [collect]
    if: |
      always() && (schedule || workflow_dispatch) &&
      (needs.collect.result == 'success' || needs.collect.result == 'skipped')
```

`needs: [collect]`만 추가하면 `collect`가 skip된 경우(월요일 cron, job=generate)에 generate가 같이 skip되는 문제가 있음.
`always()` + `needs.collect.result == 'skipped'` 조건으로 해결.

---

## 새 DB 테이블

```sql
-- 후기 메시지 저장
CREATE TABLE IF NOT EXISTS experiences (
  id BIGSERIAL PRIMARY KEY,
  discord_message_id TEXT UNIQUE NOT NULL,
  channel_id TEXT NOT NULL,
  content TEXT NOT NULL,
  image_urls TEXT[] DEFAULT '{}',
  calendar_event_title TEXT,
  processed BOOLEAN NOT NULL DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 봇이 생성한 스레드 추적 (수집 대상 목록)
CREATE TABLE IF NOT EXISTS experience_threads (
  id BIGSERIAL PRIMARY KEY,
  event_title TEXT NOT NULL,
  calendar_event_id TEXT UNIQUE NOT NULL,
  discord_thread_id TEXT UNIQUE NOT NULL,
  discord_channel_id TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 트러블슈팅

**Discord API 40333 — internal network error**

- Apps Script에서 Discord 직접 호출 시 발생
- Google Apps Script 서버 IP가 Discord에서 차단됨
- 해결: Apps Script → GitHub API(repository_dispatch) → GitHub Actions → Discord

---

## 개선 전후 비교

| 항목               | 4주차                | 5주차                        |
| ------------------ | -------------------- | ---------------------------- |
| 입력 소스          | Claude 로그 + Notion | + Discord 경험 후기          |
| 경험 글 생성       | 불가                 | 캘린더 일정 종료 → 자동      |
| 이미지 처리        | 없음                 | Supabase Storage 영구 보관   |
| GitHub Actions Job | 1개                  | 3개 (역할 분리)              |
| Job 실행 순서      | 독립 실행            | collect → generate 순서 보장 |
| 파이프라인 분기    | 단일 파이프라인      | 기술 대화 / 경험 후기 분리   |

---

## 다음 주 계획

- 실제 일정 후기 작성 후 end-to-end 테스트
- 토큰 사용량 모니터링 후 불필요한 Sonnet 호출 줄이기
