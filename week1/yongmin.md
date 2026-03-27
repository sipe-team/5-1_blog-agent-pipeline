# Week 1 - yongmin

## 아웃풋 목표

> 이번 파이프라인의 최종 결과물

- 사용자가 평소에 텍스트 및 이미지를 던지면 능동적으로 "일기발행/일정관리/인사이트정리"를 해주는 시스템 (인생 짬통 만들기)
- 최종 발행처: 노션

## 파이프라인 설계

> 전체 흐름

```mermaid
graph TD
    %% 1. Ingestion Layer
    subgraph Ingestion ["1. Ingestion Layer (수집)"]
        A1[Android: Share Intent] -->|HTTP POST| B[FastAPI Center Hub]
        A2[Slack: Message to Self] -->|Socket Mode| B
    end

    %% 2. Intelligence Layer
    subgraph Intelligence ["2. Intelligence Layer (판단/라우팅)"]
        B --> C{LLM Intent Router}
        C -->|Category: Diary| D1[Daily Agent]
        C -->|Category: Schedule| D2[Schedule Agent]
        C -->|Category: Insight| D3[Insight Agent]
    end

    %% 3. Specialized Processing (Sequential Logic)
    subgraph Processing ["3. Specialized Processing (전문 처리)"]
        D1 --> E1[Step 1: Vision/Text 분석]
        E1 --> E2[Step 2: 정형화된 일시적 메모 생성]
        
        D2 --> E3[Step 1: 구글 캘린더 즉시 등록]
        E3 --> E4{Step 2: 충돌 검사}
        
        D3 --> E5[Step 1: 본문 스크래핑 및 추출]
        E5 --> E6[Step 2: 키워드 및 해시태그 매핑]
    end

    %% 4. Action & Storage
    subgraph Action ["4. Action Layer (발행/저장)"]
        E2 --> G1[(Notion: Raw Daily Logs)]
        E4 -->|Conflict Detected| F1[Slack: 충돌 알림 및 취소 버튼]
        E6 --> G3[(Notion: Insight Library - 본문+태그)]
        
        %% Batch Job for Final Diary
        G1 -.->|Nightly Batch| J[AI: 전체 로그 취합 및 완성된 일기 발행]
        J --> G4[(Notion: Final Diary Page)]
    end

    %% Styling
    style B fill:#f9f,stroke:#333,stroke-width:2px
    style C fill:#bbf,stroke:#333,stroke-width:2px
    style F1 fill:#fbb,stroke:#333,stroke-width:2px
    style J fill:#fff,stroke:#333,stroke-dasharray: 5 5
```

- **수집 소스 (우선순위 순)**:
  - 사용자가 직접 데이터 입력. 일상 속에서 남기고 싶은 내용을 텍스트/이미지 형태로 던짐. 모바일의 경우 공유 기능으로, PC의 경우 슬랙에 던지기
- **사용 툴/프레임워크**: Python + FastAPI + LangGraph + Pydantic V2 + Slack SDK + Notion SDK + Android + AI(모델 안 정함)
- **발행 채널**: 노션 (수동/자동 고민 중)

## 이번 주 진행 내용

- 파이프라인 전체 흐름 설계
- Slack 연결 및 분류 모델에게 데이터 전달

## 구현 중 막힌 것 / 해결한 것

| 문제                                                         | 해결 여부 | 메모                                                         |
| ------------------------------------------------------------ | --------- | ------------------------------------------------------------ |
| 다 처음하는 것들이라 굉장히 AI 의존적임                      | 해결중    | 차차 구현해나가면서 이해도를 넓혀나가야함                    |
| 기술 스택을 AI가 작성해줬는데 정말 필요한 건지 아직 잘 모름. | 해결 중   | 각 단계를 수행하며 고민 후 결정 예정                         |
| 갤럭시 S26의 나우넛지 같은 기능을 기대했지만, 불가능해서 데이터를 사용자가 직접 넘겨야함. | 해결불가  | 지금까지의 생각으로는 Framework단을 건드리지 않고서는 해결 불가능 |

## 인사이트 / 배운 것

- 현재는 노션에 저장할 예정이지만, 추후 나의 일기나 인사이트를 검색 용이하게 디벨롭하려면 자체 저장소를 사용해야할 수도 있겠다?
- 데이터를 사용자의 노력 없이 수집해오는 것은 굉장히 어렵다...

## 다음 주 계획 및 고민되는 것들

### 1. 데이터 분류하기

사용자가 넘긴 데이터가 일기/일정/인사이트 중 무엇인지 판단하기

### 2. 일기 Agent 건드리기

현재 굉장히 추상적으로 적어놨기 때문에 일기를 작성할 AI agent를 어떻게 만들어야할지 알아보기.