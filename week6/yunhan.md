# 자동 SNS 사진작가 파이프라인

카메라로 사진만 찍으면 Instagram 피드까지 자동으로 올라가는 파이프라인입니다.

## 전체 흐름도

```mermaid
flowchart TD
    subgraph capture["📸 촬영 & 동기화"]
        A["카메라 촬영<br/>(Canon / Ricoh)"] -->|WiFi 연동| B["📱 iPhone"]
        B -->|iCloud 동기화| C["💻 Mac iCloud"]
    end

    subgraph filter["🔁 Claude Loop · Daily"]
        C --> D{"EXIF Metadata<br/>읽기"}
        D -->|"Canon / Ricoh"| E["💾 Local Desktop<br/>다운로드"]
        D -->|"그 외 카메라"| X["❌ 제외"]
    end

    subgraph analyze["🤖 분석 & 피드 생성"]
        E --> F["📅 날짜별 분류"]
        F --> G["Code Agent<br/>사진 읽기"]
        G --> H["상황 · 분위기 정리"]
        H --> I["🎵 Instagram 피드<br/>음악 추천"]
        I --> J["📝 피드 생성"]
    end

    subgraph publish["📲 자동 업로드"]
        J -->|"OpenClaw<br/>이틀에 한 번"| K["Instagram 게시"]
        K --> L["✨ 자동 SNS<br/>사진작가 완성!"]
    end

    style A fill:#fef3c7,stroke:#f59e0b,color:#000
    style C fill:#dbeafe,stroke:#3b82f6,color:#000
    style E fill:#dcfce7,stroke:#22c55e,color:#000
    style G fill:#fce7f3,stroke:#ec4899,color:#000
    style K fill:#e0e7ff,stroke:#6366f1,color:#000
    style L fill:#fef08a,stroke:#eab308,color:#000
    style X fill:#fee2e2,stroke:#ef4444,color:#000
```

## 단계별 상세

| 단계 | 도구 | 역할 |
|------|------|------|
| 1️⃣ 촬영 | Canon / Ricoh 카메라 | 사진 촬영 후 WiFi로 iPhone에 자동 전송 |
| 2️⃣ 동기화 | iCloud | iPhone → Mac으로 사진 자동 동기화 |
| 3️⃣ 필터링 | **Claude `/loop`** (Daily) | EXIF Metadata 검사 → Canon/Ricoh 사진만 로컬로 이동 |
| 4️⃣ 분류 | 파일 시스템 | 촬영 날짜 기준으로 폴더 분류 |
| 5️⃣ 분석 | Code Agent | 사진의 상황·분위기 해석 |
| 6️⃣ 큐레이션 | Code Agent | 분위기에 맞는 음악 추천 + 피드 문구 생성 |
| 7️⃣ 게시 | **OpenClaw** (2일 주기) | Instagram에 피드 자동 업로드 |

## 핵심 자동화 포인트

- **EXIF 필터**: `Make` 태그가 `Canon` 또는 `RICOH`인 사진만 선별 → 휴대폰 사진은 자동 제외
- **Claude Loop**: 매일 1회 실행되는 백그라운드 루틴
- **OpenClaw 스케줄러**: 격일 게시로 피드 밀도 조절
