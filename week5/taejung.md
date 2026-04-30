# Week 5 - taejung

## 이번 주 주제

3주차에 만든 블로그 자동화 파이프라인을 실제로 쓸 수 있는 수준으로 완성했다.

1. **AI 메모용 디지털 가든 공간 추가** — Quartz v4로 `Aqudi/notes` 셋업
2. **주제 선정 → 발행 전체 파이프라인 완성** — Discord 알림 + launchd 로컬 크론 + Dry-run 검증

---

## 1. AI 메모용 공간: Quartz v4

기존 블로그(`blog.aqudi.me`)는 그대로 두고, **AI가 생성하는 글과 내 메모를 쌓을 별도 공간**이 필요했다. `[[wikilink]]`로 개념들이 자동으로 연결되고, 나중에 노트가 쌓이면 그래프 뷰로 한눈에 볼 수 있는 디지털 가든 형태가 딱 좋았다. (옵시디언 러버)

**Quartz v4 특징**

| 항목           | 특징                                 |
| -------------- | ------------------------------------ |
| `[[wikilink]]` | 네이티브 지원                        |
| 백링크         | AI가 삽입한 링크들이 자동으로 연결됨 |
| 그래프 뷰      | 노트 사이의 관계를 시각화            |
| 한국어         | `locale: "ko-KR"` 설정으로 깔끔하게  |

### AI 글쓰기 프롬프트에 wikilink 추가

`writer.py`의 `write_draft` 프롬프트에 가이드라인을 추가했다.

```python
디지털 가든 연결 가이드라인:
- 관련 개념/기술이 등장하면 [[개념명]] 형식의 wikilink로 표기
- 예: "[[LLM]]", "[[Rust]]", "[[AST]]"
- 문단당 1~2개, 과도하지 않게
- 글 마지막에 "## 관련 노트" 섹션 추가, [[링크]] 2~3개 나열
```

결과물에서 `[[머신러닝]]`, `[[AST]]`, `[[크로스 컴파일]]` 같은 링크가 자연스럽게 생성됐다.
지금 당장은 연결될 노트가 없어서 dead link지만, 나중에 노트가 쌓이면 자동으로 백링크가 연결될 것으로 기대된다.

---

## 2. 주제 선정 → 발행 전체 파이프라인

### 스케줄러: GitHub Actions → launchd

처음엔 GitHub Actions `schedule` 트리거로 매일 자동 실행하려 했다. 그런데 생각해보니 집에 항상 켜져 있는 Mac Mini가 있고, GitHub Actions도 무료 플랜이면 실행 시간 제한이 있어서 그냥 로컬 컴퓨터를 쓰는 게 더 마음 편할 것 같았다.

마침 요즘 Claude Code 스킬 사용 통계 데이터를 모으는 데도 **launchd**를 쓰고 있어서 같은 방식으로 적용해봤다.

```xml
<!-- ~/Library/LaunchAgents/com.aqudi.blog-automation.plist -->
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key><integer>9</integer>
    <key>Minute</key><integer>0</integer>
</dict>
```

> 삽질: `StartCalendarInterval`과 `StartInterval`을 동시에 쓰면 두 번 실행된다. 하나만 써야 한다.

### Discord + GitHub Issues 연동 (예정)

- Discord: 알림 채널로 웹훅 메시지로 이벤트 알림 전송
- Github Issues: 인간 피드백 남기는 공간으로 활용

```mermaid
flowchart LR
    A[매일 09:00 launchd] --> B[RSS 수집 + AI 분석]
    B --> C[주제 후보 5개\nGitHub Issue 생성]
    C --> D[Discord 알림\n"이슈 가서 선택하세요"]
    D --> E{체크박스 선택}
    E --> F[issues.edited 트리거]
    F --> G[블로그 글 작성 + push]
```

아직 실제로 연동은 못 해봤다. 근데 요즘 [Happy Code](https://github.com/slopus/happy)라는 서비스를 쓰고 있는데, 여기서 Claude Code 세션을 직접 관리할 수 있어서 GitHub Issues 없이 그쪽으로 리뷰 플로우를 연결하는 게 더 나을 수도 있겠다 싶다. Github Issues를 다시 AI에게 피드백 먹이는 것보다 Happy Code로 로컬에서 피드백까지 주는 게 더 낫지 않을까 싶다.

### Dry-run 검증

전체 파이프라인을 처음으로 end-to-end로 돌려봤다.

```bash
uv run blog-automation fetch      # RSS 50개 수집
uv run blog-automation analyze    # 주제 후보 4개 추출
uv run blog-automation draft "Zig로 C 컴파일러 작성하기: 시스템 프로그래밍 언어의 새로운 가능성"
uv run blog-automation revise zig로-c-컴파일러-작성하기.md \
  --feedback "태그 추가, comptime 설명 구체화, CTA 추가"
gh api repos/Aqudi/notes/contents/content/posts/파일명 --method PUT ...
# → 커밋 -> 푸시 -> GitHub Pages 자동 배포 트리거
```

---

## 만들어진 블로그 글

[**"Zig로 C 컴파일러 작성하기: 시스템 프로그래밍 언어의 새로운 가능성"**](https://aqudi.github.io/notes/posts/zig%EB%A1%9C-c-%EC%BB%B4%ED%8C%8C%EC%9D%BC%EB%9F%AC-%EC%9E%91%EC%84%B1%ED%95%98%EA%B8%B0-%EC%8B%9C%EC%8A%A4%ED%85%9C-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EC%96%B8%EC%96%B4%EC%9D%98-%EC%83%88%EB%A1%9C%EC%9A%B4-%EA%B0%80%EB%8A%A5%EC%84%B1)

- Hacker News에 올라온 "Writing a C Compiler, in Zig" 기사에서 주제 착안
- `[[Zig]]`, `[[Rust]]`, `[[AST]]`, `[[크로스 컴파일]]`, `[[메모리 안전성]]` 등 wikilink 자동 삽입
- `ArenaAllocator`, `comptime` 등 Zig 핵심 개념 코드 예시 포함
- 태그: `[tech, zig, compiler, systems-programming]`

---

## 인사이트 / 배운 것

- Z.AI의 GLM5.1이 생각보다 글을 잘 쓴다.
- Max Token을 짧게 잡으니까 글을 쓰다가 만다.

## 다음 주 계획

- Discord 웹훅 URL 설정 + 실제 알림 테스트
- GitHub Issues 연동 vs Happy Coder 활용 방향 결정 (아마 Happy Coder로 가지 않을까)
- 웹 대시보드 만들기 & 글 특정 부분 하이라이트해서 수정 요청하기 기능 만들기
- Threads 용으로 봇을 만들어서 자동 게시하는 계정 만들어보기
