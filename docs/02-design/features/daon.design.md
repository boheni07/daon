# Design: 다온(Daon) 플랫폼 SW 아키텍처

> **Feature:** daon-sw-architecture
> **작성일:** 2026-05-22
> **Phase:** Design ✅
> **선택 설계안:** Option C — Modular Monolith → Microservices
> **참조 Plan:** `docs/01-plan/features/daon.plan.md`
> **참조 PRD:** `docs/00-pm/daon.prd.md` v1.3

---

## Context Anchor

| Anchor | 값 |
|---|---|
| **WHY** | 외로움이 치매 위험 50%↑를 유발하는 가운데, 오미가 어르신 생애사를 누적 수집하며 정서적 동반자가 되는 서비스를 만들기 위한 SW 기반 확립 |
| **WHO** | (1차) 경도인지장애·초로기 치매·고위험군 어르신 / (2차) 가족 보호자·치매안심센터·요양시설 |
| **RISK** | LLM 환각·오픈소스 한국어 품질·응답 지연 1.5초 달성·데이터 주권·스케일 시 벡터스토어 전환 비용 |
| **SUCCESS** | 응답 P95 ≤1.5초 / 자서전 수집 정확도 ≥85% / 자서전 Coverage 6개월 80% / 위기 발화 감지 정밀도 ≥90% |
| **SCOPE** | (IN) L2~L5 SW 전체, 오미↔클라우드 프로토콜 / (OUT) L1 HW 설계, 모바일 앱 UI 픽셀 상세, 인프라 프로비저닝 |

---

## 1. 개요

### 1.1 설계 철학

**Modular Monolith → Microservices** 전략을 채택한다.

- **Phase 0~1 (MVP)**: 단일 FastAPI 애플리케이션. 내부는 `dialog_core / memory_layer / operate / trust` 패키지로 엄격하게 분리한다. 패키지 간 통신은 명시적 인터페이스(Python 추상 클래스)를 통해서만 이루어진다.
- **Phase 2+**: 트래픽·팀 규모에 따라 패키지를 독립 컨테이너로 추출한다. 인터페이스 계약이 보존되므로 API 변경 없이 추출 가능하다.

### 1.2 선택 근거 (Option C)

| 기준 | 근거 |
|---|---|
| MVP 속도 | 개발 4개월(Phase 1) 내에 파일럿 준비 필요 |
| 인터페이스 명확성 | 패키지별 ABC(추상 기반 클래스) 정의로 미래 추출 경로 확보 |
| 운영 단순성 | Docker Compose 3개(app / worker / llm-server)로 Phase 1 충분 |
| ADR 정합성 | 10개 ADR의 기술 스택을 모두 수용, 레이어 경계를 코드 구조로 표현 |

---

## 2. 시스템 아키텍처

### 2.1 전체 구성도

```
┌─────────────────────────────────────────────────────────────┐
│                      오미 디바이스 (L1)                       │
│  Jetson Orin Nano │ Whisper distil-large-v3 │ Wake Word      │
│  로컬 TTS │ Edge 폴백 모드 │ mTLS 인증서                      │
└──────────────────────────┬──────────────────────────────────┘
                           │ WebSocket / MQTT (mTLS, TLS 1.3)
┌──────────────────────────▼──────────────────────────────────┐
│                  Daon Cloud Platform                         │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  daon-app (FastAPI, Python 3.12)                    │    │
│  │                                                     │    │
│  │  ┌──────────────┐  ┌──────────────┐                │    │
│  │  │ dialog_core  │  │ memory_layer │                │    │
│  │  │  L2          │◄─►  L3          │                │    │
│  │  │ 대화 오케스트레이터│  │ 자서전 엔진    │                │    │
│  │  │ 페르소나 엔진   │  │ RAG 검색      │                │    │
│  │  │ 안전 필터     │  │ 메모리 그래프  │                │    │
│  │  │ 감정 인식     │  │ 질문 우선순위  │                │    │
│  │  └──────────────┘  └──────────────┘                │    │
│  │                                                     │    │
│  │  ┌──────────────┐  ┌──────────────┐                │    │
│  │  │   operate    │  │    trust     │                │    │
│  │  │  L4          │  │  L5          │                │    │
│  │  │ REST API      │  │ JWT / RBAC   │                │    │
│  │  │ WebSocket GW  │  │ 동의 Lineage  │                │    │
│  │  │ 보호자 앱 API  │  │ MLOps 모니터  │                │    │
│  │  │ 자서전 생성기  │  │ 감사 로그     │                │    │
│  │  └──────────────┘  └──────────────┘                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌───────────┐  ┌──────────┐  ┌──────────────────────┐     │
│  │PostgreSQL │  │ Redis 7  │  │ EXAONE 3.5 추론 서버  │     │
│  │16+pgvector│  │ 캐시/브로커 │  │ (GPU 서버, HTTP API)  │     │
│  │+Apache AGE│  └──────────┘  └──────────────────────┘     │
│  └───────────┘                                              │
│                                                              │
│  ┌────────────────────────────────────────┐                 │
│  │ Celery Workers                         │                 │
│  │  autobiography_processor │ export_worker │ alert_worker  │
│  └────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
                           │
              ┌────────────┘
              │
┌─────────────▼──────────────┐    ┌─────────────────────────┐
│  Flutter 보호자 앱           │    │  시설 관리자 대시보드      │
│  iOS / Android              │    │  (Web, React)            │
└────────────────────────────┘    └─────────────────────────┘
```

### 2.2 패키지 디렉터리 구조

```
daon-platform/
├── app/
│   ├── main.py                      # FastAPI 앱 엔트리포인트
│   ├── core/
│   │   ├── config.py                # 환경변수·설정 (pydantic-settings)
│   │   ├── database.py              # SQLAlchemy async 엔진
│   │   ├── redis_client.py          # Redis 클라이언트
│   │   └── dependencies.py          # FastAPI 의존성 (DB세션, 인증)
│   │
│   ├── dialog_core/                 # L2 — 대화 엔진 패키지
│   │   ├── __init__.py
│   │   ├── interfaces.py            # ABC: IDialogCore
│   │   ├── orchestrator.py          # 상태머신 오케스트레이터
│   │   ├── persona_engine.py        # 페르소나 개인화
│   │   ├── safety_filter.py         # 안전 필터 + 가드레일 + 감정 부조화 방지
│   │   ├── dialect_normalizer.py    # 방언 전처리 (경상·전라·충청·제주 → 표준어)
│   │   ├── question_generator.py   # 시점×주제 질문 생성
│   │   ├── emotion_detector.py      # 감정 분류
│   │   └── crisis_detector.py       # 위기 발화 감지 (L1~L3)
│   │
│   ├── memory_layer/                # L3 — 기억·지식 레이어 패키지
│   │   ├── __init__.py
│   │   ├── interfaces.py            # ABC: IMemoryLayer
│   │   ├── autobiography_service.py # 청크 CRUD + 저장
│   │   ├── question_priority.py     # 빈칸 탐지 + 우선순위 스코어
│   │   ├── rag_retriever.py         # KoSimCSE + pgvector 검색
│   │   ├── memory_graph.py          # Apache AGE Cypher 쿼리
│   │   └── embedding_service.py     # KoSimCSE 모델 래퍼
│   │
│   ├── operate/                     # L4 — 운영·연계 레이어 패키지
│   │   ├── __init__.py
│   │   ├── api/
│   │   │   ├── auth.py              # POST /auth/*
│   │   │   ├── users.py             # CRUD /users/*
│   │   │   ├── autobiography.py     # /users/{id}/autobiography/*
│   │   │   ├── sessions.py          # /users/{id}/sessions/* + topic-summary
│   │   │   ├── alerts.py            # /users/{id}/anomalies/*
│   │   │   ├── family.py            # /users/{id}/family/*
│   │   │   ├── export.py            # /users/{id}/autobiography/export/*
│   │   │   ├── notifications.py     # /users/{id}/notifications + weekly-report
│   │   │   ├── legacy.py            # /users/{id}/legacy-package (유산 패키지)
│   │   │   └── facilities.py        # /facilities/*
│   │   ├── websocket/
│   │   │   ├── session_handler.py   # WS /ws/session/{user_id}
│   │   │   └── protocol.py          # WebSocket 메시지 타입 정의
│   │   ├── notifications/
│   │   │   ├── push_service.py      # FCM/APNs 푸시 알림 (Tier 1 긴급)
│   │   │   ├── notification_tier.py # Tier 1/2/3 분류 라우터
│   │   │   └── weekly_report_builder.py  # Tier 3 주간 요약 생성
│   │   └── export/
│   │       ├── autobiography_generator.py  # PDF+오디오북 생성
│   │       ├── pdf_renderer.py             # WeasyPrint 래퍼
│   │       ├── audiobook_builder.py        # TTS MP3 → m4b
│   │       └── legacy_package_builder.py   # 유산 패키지 (자서전PDF+음성클립)
│   │
│   ├── trust/                       # L5 — 거버넌스·보안 패키지
│   │   ├── __init__.py
│   │   ├── auth/
│   │   │   ├── jwt_service.py       # JWT 발급·검증
│   │   │   ├── rbac.py              # RBAC 5레벨 권한 모델
│   │   │   └── device_auth.py       # mTLS 디바이스 인증
│   │   ├── consent/
│   │   │   └── consent_service.py   # 동의 Lineage + 철회 cascade
│   │   ├── audit/
│   │   │   └── audit_logger.py      # 감사 로그 (append-only)
│   │   └── mlops/
│   │       └── model_monitor.py     # LLM 품질·STT WER 모니터링
│   │
│   ├── models/                      # SQLAlchemy ORM 모델
│   │   ├── user.py
│   │   ├── autobiography.py
│   │   ├── session.py
│   │   ├── consent.py
│   │   └── medication.py
│   │
│   └── schemas/                     # Pydantic v2 요청/응답 스키마
│       ├── user.py
│       ├── autobiography.py
│       ├── session.py
│       └── alert.py
│
├── workers/                         # Celery 비동기 작업
│   ├── celery_app.py
│   ├── autobiography_processor.py   # 청크 임베딩·저장
│   ├── export_worker.py             # 책자·오디오북 생성
│   └── alert_worker.py              # 이상징후 알림 발송
│
├── migrations/                      # Alembic DB 마이그레이션
│   ├── env.py
│   └── versions/
│
├── mobile/                          # Flutter 보호자 앱
│   └── lib/
│       ├── main.dart
│       ├── screens/
│       │   ├── home_screen.dart
│       │   ├── autobiography_screen.dart
│       │   ├── voice_screen.dart
│       │   ├── alerts_screen.dart
│       │   ├── settings_screen.dart
│       │   └── export_screen.dart
│       ├── widgets/
│       └── services/
│           ├── api_service.dart
│           └── auth_service.dart
│
├── tests/
│   ├── unit/
│   │   ├── dialog_core/
│   │   ├── memory_layer/
│   │   └── trust/
│   ├── integration/
│   │   ├── test_conversation_flow.py
│   │   ├── test_rag_pipeline.py
│   │   └── test_safety_filter.py
│   └── e2e/
│       └── daon.spec.ts             # Playwright E2E
│
├── docker-compose.yml               # Phase 1 MVP 배포
├── docker-compose.llm.yml           # EXAONE 추론 서버
├── Dockerfile
├── pyproject.toml                   # uv / ruff / pytest 설정
└── alembic.ini
```

---

## 3. 패키지 인터페이스 설계 (ABC)

패키지 간 통신은 반드시 인터페이스를 통한다. 미래 마이크로서비스 추출 시 구현체만 교체한다.

### 3.1 IDialogCore (L2 인터페이스)

```python
# app/dialog_core/interfaces.py
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class DialogRequest:
    user_id: str
    session_id: str
    utterance_text: str
    edge_emotion_hint: str = "neutral"
    audio_duration_ms: int = 0

@dataclass
class DialogResponse:
    text: str
    is_final: bool
    led_code: str
    emotion_detected: str
    autobiography_chunk_saved: bool
    crisis_level: int  # 0=정상, 1~3=위기

class IDialogCore(ABC):
    @abstractmethod
    async def process_utterance(self, req: DialogRequest) -> AsyncIterator[DialogResponse]:
        """음성 발화를 처리하고 응답을 스트리밍한다."""
        ...

    @abstractmethod
    async def get_proactive_topic(self, user_id: str) -> str | None:
        """오미 선제 발화 주제를 반환한다 (없으면 None)."""
        ...
```

### 3.2 IMemoryLayer (L3 인터페이스)

```python
# app/memory_layer/interfaces.py
from abc import ABC, abstractmethod

@dataclass
class AutobiographyChunk:
    period: str
    topic: str
    content: str
    confidence: float
    emotion_score: float
    embedding: list[float] | None = None

@dataclass
class RAGContext:
    chunks: list[AutobiographyChunk]
    related_entities: list[dict]

class IMemoryLayer(ABC):
    @abstractmethod
    async def save_chunk(self, user_id: str, chunk: AutobiographyChunk) -> str:
        """청크를 저장하고 chunk_id를 반환한다."""
        ...

    @abstractmethod
    async def retrieve_context(self, user_id: str, query: str, top_k: int = 5) -> RAGContext:
        """RAG 검색으로 관련 자서전 컨텍스트를 반환한다."""
        ...

    @abstractmethod
    async def get_next_question(self, user_id: str) -> tuple[str, str]:
        """우선순위 스코어 기반으로 (period, topic) 다음 질문 대상을 반환한다."""
        ...

    @abstractmethod
    async def get_coverage_matrix(self, user_id: str) -> dict[tuple[str, str], float]:
        """5×4 coverage 매트릭스를 반환한다."""
        ...
```

---

## 4. API 계약 명세

### 4.1 공통 응답 형식

```python
# app/schemas/common.py
from pydantic import BaseModel
from typing import Generic, TypeVar

T = TypeVar("T")

class ApiResponse(BaseModel, Generic[T]):
    data: T | None = None
    error: str | None = None
    pagination: dict | None = None

# 성공: {"data": {...}, "error": null}
# 실패: {"data": null, "error": "에러 메시지"}
# 목록: {"data": [...], "error": null, "pagination": {"total": 100, "page": 1}}
```

### 4.2 핵심 엔드포인트 계약

```
Base URL: https://api.daon.ai/v1

[인증]
POST   /auth/login
  Request:  {"email": str, "password": str}
  Response: {"data": {"access_token": str, "refresh_token": str, "role": str}}
  Auth:     없음

POST   /auth/device
  Request:  {"device_id": str, "cert_fingerprint": str}
  Response: {"data": {"device_token": str, "expires_in": 86400}}
  Auth:     mTLS 인증서 필수

[어르신 관리]
POST   /users
  Request:  UserCreate {name, birth_year, dementia_stage, persona, dialect}
  Response: {"data": UserProfile}
  Auth:     GUARDIAN_PRIMARY | FACILITY_ADMIN

GET    /users/{user_id}
  Response: {"data": UserProfile}
  Auth:     GUARDIAN_FAMILY+

PATCH  /users/{user_id}
  Request:  UserUpdate {persona?, dialect?, dementia_stage?}
  Response: {"data": UserProfile}
  Auth:     GUARDIAN_PRIMARY

DELETE /users/{user_id}
  Response: {"data": {"deleted": true, "scheduled_at": datetime}}
  Auth:     GUARDIAN_PRIMARY (비동기 삭제, ≤24hr)

[자서전]
GET    /users/{user_id}/autobiography
  Query:    ?period=유년기&topic=인물&limit=20&cursor=xxx
  Response: {"data": [AutobiographyChunk], "pagination": {...}}
  Auth:     GUARDIAN_FAMILY+

GET    /users/{user_id}/autobiography/coverage
  Response: {"data": {"matrix": {period: {topic: {count, confidence, last_updated}}}}}
  Auth:     GUARDIAN_FAMILY+

POST   /users/{user_id}/autobiography/chunks
  Request:  ChunkCreate {period, topic, content, is_private?}
  Response: {"data": AutobiographyChunk}
  Auth:     GUARDIAN_PRIMARY | DEVICE

[대화 세션]
WS     /ws/session/{user_id}
  Auth:     DEVICE (device_token)
  Protocol: 별도 §5 WebSocket 프로토콜 명세

GET    /users/{user_id}/sessions
  Query:    ?from=date&to=date&limit=30
  Response: {"data": [SessionSummary]}
  Auth:     GUARDIAN_FAMILY+

[알림·이상징후]
GET    /users/{user_id}/anomalies
  Query:    ?level=2&resolved=false
  Response: {"data": [AnomalyEvent]}
  Auth:     GUARDIAN_PRIMARY | FACILITY_ADMIN

POST   /users/{user_id}/anomalies/{anomaly_id}/resolve
  Request:  {"note": str}
  Response: {"data": {"resolved": true}}
  Auth:     GUARDIAN_PRIMARY | FACILITY_ADMIN

[자서전 산출물]
POST   /users/{user_id}/autobiography/export
  Request:  {"format": "pdf" | "audiobook" | "both", "title"?: str}
  Response: {"data": {"job_id": str, "status": "queued"}}
  Auth:     GUARDIAN_PRIMARY

GET    /users/{user_id}/autobiography/export/{job_id}
  Response: {"data": {"status": "processing|completed|failed", "progress": 0.7}}
  Auth:     GUARDIAN_FAMILY+

GET    /users/{user_id}/autobiography/export/{job_id}/download
  Response: Redirect → Signed S3 URL (TTL 7일)
  Auth:     GUARDIAN_FAMILY+

[알림 Tier 관리 — ADR-11]
GET    /users/{user_id}/notifications
  Query:    ?tier=1&unread=true&limit=50
  Response: {"data": [NotificationItem {id, tier, title, body, created_at, read}]}
  Auth:     GUARDIAN_FAMILY+

GET    /users/{user_id}/weekly-report
  Query:    ?week=2026-W21
  Response: {"data": WeeklyReport {period, interaction_count, emotion_summary, autobiography_coverage_delta, highlights}}
  Auth:     GUARDIAN_FAMILY+

PATCH  /users/{user_id}/notification-settings
  Request:  {"tier1_push": bool, "tier2_inapp": bool, "tier3_weekly": bool}
  Response: {"data": NotificationSettings}
  Auth:     GUARDIAN_PRIMARY

[세션 주제 요약]
GET    /users/{user_id}/sessions/{session_id}/topic-summary
  Response: {"data": {"topics": [str], "emotion_arc": str, "autobiography_periods_covered": [str]}}
  Auth:     GUARDIAN_FAMILY+

[유가족 유산 패키지 — PRD §16]
POST   /users/{user_id}/legacy-package
  Request:  {"trigger": "deceased" | "withdrawal", "recipient_email"?: str}
  Response: {"data": {"job_id": str, "status": "queued"}}
  Auth:     GUARDIAN_PRIMARY | SYSTEM_ADMIN

GET    /users/{user_id}/legacy-package/{job_id}
  Response: {"data": {"status": "processing|completed|failed", "progress": float}}
  Auth:     GUARDIAN_PRIMARY

GET    /users/{user_id}/legacy-package/{job_id}/download
  Response: Redirect → Signed S3 URL (TTL 90일, SC-12)
  Auth:     GUARDIAN_PRIMARY
```

### 4.3 오류 코드

| HTTP Status | error_code | 설명 |
|---|---|---|
| 400 | VALIDATION_ERROR | 요청 파라미터 오류 (fieldErrors 포함) |
| 401 | UNAUTHORIZED | 인증 토큰 없음·만료 |
| 403 | FORBIDDEN | RBAC 권한 부족 |
| 404 | NOT_FOUND | 리소스 없음 |
| 409 | CONFLICT | 중복 생성 |
| 422 | UNPROCESSABLE | 비즈니스 규칙 위반 |
| 429 | RATE_LIMIT | 요청 한도 초과 |
| 500 | INTERNAL_ERROR | 서버 내부 오류 |

---

## 5. WebSocket 대화 프로토콜 상세

### 5.1 연결 수립

```
클라이언트: GET wss://ws.daon.ai/v1/ws/session/{user_id}
            Authorization: Bearer {device_token}
            
서버 응답: HTTP 101 Switching Protocols
           연결 확인: {"type": "connected", "session_id": "sess_xxx"}
```

### 5.2 메시지 시퀀스 (정상 대화 흐름)

```
오미                          서버
 │                              │
 │── utterance ──────────────► │  발화 수신
 │   {text, audio_duration_ms, │
 │    edge_emotion_hint}        │
 │                              │── [안전 필터 체크]
 │                              │── [RAG 검색]
 │                              │── [EXAONE 스트리밍 요청]
 │◄── response_chunk ────────── │  첫 토큰 (is_final=false)
 │   {text, is_final:false,     │
 │    led_code:"orange_blink"}  │── [자서전 청크 저장 (비동기)]
 │◄── response_chunk ────────── │  마지막 토큰 (is_final=true)
 │   {text, is_final:true,      │
 │    led_code:"green"}         │
 │                              │
 │── session_end ─────────────► │  세션 종료 + 로컬 청크 업로드
 │   {turn_count, duration_sec, │
 │    local_chunks:[...]}       │── [DB 동기화 + 감정 요약 저장]
 │                              │── [이상징후 레벨 판정]
```

### 5.3 메시지 타입 전체 목록

| type | 방향 | 설명 |
|---|---|---|
| `connected` | 서버→오미 | 연결 확인, session_id 발급 |
| `utterance` | 오미→서버 | 발화 텍스트 전송 |
| `response_chunk` | 서버→오미 | LLM 응답 스트리밍 청크 |
| `proactive_start` | 서버→오미 | 서버 선제 발화 시작 지시 |
| `anomaly_alert` | 서버→오미 | 이상징후 감지 알림 |
| `medication_reminder` | 서버→오미 | 복약 알림 트리거 |
| `session_end` | 오미→서버 | 세션 종료 + 로컬 데이터 동기화 |
| `heartbeat` | 양방향 | 연결 유지 (30초 주기) |
| `error` | 서버→오미 | 오류 알림 (엣지 폴백 지시 포함) |

---

## 6. 데이터 모델 상세

### 6.1 ORM 모델 (SQLAlchemy 2.x, async)

```python
# app/models/user.py
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import String, Integer, Text, Boolean, ForeignKey
from sqlalchemy.dialects.postgresql import UUID, JSONB, TIMESTAMPTZ
import uuid

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    birth_year: Mapped[int | None] = mapped_column(Integer)
    dementia_stage: Mapped[str | None] = mapped_column(
        String(20),
        # CHECK (dementia_stage IN ('MCI','early','moderate','severe'))
    )
    persona: Mapped[str | None] = mapped_column(String(20))  # friend/sister/brother
    dialect: Mapped[str] = mapped_column(String(20), default="standard")
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=func.now())
    
    autobiography_chunks: Mapped[list["AutobiographyChunk"]] = relationship(back_populates="user")
    sessions: Mapped[list["ConversationSession"]] = relationship(back_populates="user")

# app/models/autobiography.py
from pgvector.sqlalchemy import Vector

class AutobiographyChunk(Base):
    __tablename__ = "autobiography_chunks"
    
    id: Mapped[uuid.UUID] = mapped_column(UUID, primary_key=True, default=uuid.uuid4)
    user_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("users.id", ondelete="CASCADE"))
    period: Mapped[str] = mapped_column(String(20), nullable=False)
    topic: Mapped[str] = mapped_column(String(20), nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    confidence: Mapped[float] = mapped_column(default=0.8)
    emotion_score: Mapped[float] = mapped_column(default=0.0)
    is_private: Mapped[bool] = mapped_column(Boolean, default=False)
    is_deleted: Mapped[bool] = mapped_column(Boolean, default=False)
    consent_id: Mapped[uuid.UUID | None] = mapped_column(ForeignKey("consent_records.id"))
    source_session_id: Mapped[uuid.UUID | None]
    embedding: Mapped[list[float] | None] = mapped_column(Vector(768))
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=func.now())
    updated_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, onupdate=func.now())
    
    user: Mapped["User"] = relationship(back_populates="autobiography_chunks")

    __table_args__ = (
        # ivfflat 인덱스는 migration에서 별도 생성
        {"schema": None}
    )
```

### 6.2 Alembic 마이그레이션 순서

| 순번 | 마이그레이션 | 내용 |
|---|---|---|
| 001 | `create_extensions` | pgvector, uuid-ossp, Apache AGE extension 활성화 |
| 002 | `create_users` | users 테이블 |
| 003 | `create_autobiography` | autobiography_chunks 테이블 + ivfflat 인덱스 |
| 004 | `create_sessions` | conversation_sessions 테이블 |
| 005 | `create_consent` | consent_records 테이블 |
| 006 | `create_medication` | medication_logs 테이블 |
| 007 | `create_memory_entities` | memory_entities 테이블 (AGE 연동) |
| 008 | `create_coverage_view` | autobiography_coverage 뷰 |
| 009 | `create_rbac_tables` | user_roles, family_members 테이블 |
| 010 | `create_audit_log` | audit_logs 테이블 (append-only) |

---

## 7. 핵심 시퀀스 다이어그램

### 7.1 WF-02 일상 대화 (정상 흐름, SC-01 ≤1.5초)

```
어르신  오미(엣지)    dialog_core   memory_layer   EXAONE서버    오미TTS
  │       │              │              │               │           │
  │ 발화   │              │              │               │           │
  ├──────►│ (t=0ms)      │              │               │           │
  │  Wake Word 감지       │              │               │           │
  │       │ Whisper ASR   │              │               │           │
  │       │ (t=100~400ms) │              │               │           │
  │       │──utterance──►│              │               │           │
  │       │              │ 안전필터체크  │               │           │
  │       │              │ (t<10ms)      │              │           │
  │       │              │──retrieve────►│              │           │
  │       │              │  context      │              │           │
  │       │              │ (t=50ms, pgvector)           │           │
  │       │              │◄──RAGContext──│              │           │
  │       │              │              │               │           │
  │       │              │──LLM stream request──────────►│          │
  │       │              │              │     (t=200ms TTFT)       │
  │       │◄─response_chunk─────────────────────────────│          │
  │       │(t≈800ms)     │              │               │           │
  │       │──────────────────────────────────────────── TTS 프리버퍼│
  │       │              │              │               │           │
  │◄──────│ 첫 음절 재생 (t≈900~1000ms)                             │
  │       │              │              │               │           │
  │       │◄─response_chunk(is_final=true)───────────── │          │
  │       │              │──save_chunk─►│ (비동기)       │           │
  │       │              │              │ Celery Worker │           │
  │       │              │              │ 임베딩+저장    │           │
```

**지연 분해 (P95 목표 ≤1,500ms):**

| 단계 | 평균 | P95 | 최적화 |
|---|---|---|---|
| Wake Word 감지 | <50ms | <100ms | 엣지 로컬 |
| Whisper ASR | 200ms | 400ms | distil-large-v3, RTF≤0.1 |
| 네트워크(엣지→클라우드) | 50ms | 150ms | 웹소켓 지속연결 |
| 안전필터 | <5ms | <10ms | 규칙기반 우선 |
| RAG 검색(pgvector) | 20ms | 50ms | ivfflat 인덱스 |
| LLM TTFT(EXAONE) | 150ms | 200ms | KV 캐시 + 스트리밍 |
| TTS 프리버퍼(첫 문장) | 80ms | 150ms | 스트리밍 병렬 합성 |
| **합계** | **~555ms** | **~1,060ms** | **≤1,500ms 달성** |

### 7.2 WF-07 위기 발화 감지 시퀀스

```
어르신  오미(엣지)   crisis_detector   alert_worker   보호자앱   1393
  │       │               │                 │            │        │
  │ "죽고 싶어"│            │                 │            │        │
  ├──────►│               │                 │            │        │
  │       │─L1 키워드──►│(엣지, <5ms)       │            │        │
  │       │  즉각응답     │ level=2 판정      │            │        │
  │       │"많이 힘드시구나"│                 │            │        │
  │◄──────│               │                 │            │        │
  │       │──utterance──►cloud              │            │        │
  │       │               │ LLM 분류기       │            │        │
  │       │               │ level=2 확정     │            │        │
  │       │               │────────────────►│(Celery)    │        │
  │       │               │  anomaly_alert  │──FCM/APNs─►│        │
  │       │               │  {level:2,      │            │ 알림 수신│
  │       │               │   guardian_ids} │            │        │
  │       │               │                 │            │        │
  │ [level=3 시]          │                 │            │        │
  │       │               │────────────────►│            │        │
  │       │               │  1393 자동연결  │──────────────────────►│
  │       │               │  지시           │            │        │ 연결
```

**위기 레벨 처리 규칙:**

| Level | 트리거 조건 | 오미 행동 | 알림 대상 |
|---|---|---|---|
| L1 정보 | 우울 감정 지속 3세션 | 공감 대화 강화, 가족 메시지 유도 | 앱 알림 (일반) |
| L2 주의 | 자해/죽음 언급 간접 | "많이 힘드시구나" 공감 + 가족연락 권유 | 보호자 즉시 Push |
| L3 위기 | 자해 의도 직접·명확 | 1393 연결 + 보호자 전화 자동 발신 | 전체 보호자 + 시설 + 119 |

### 7.3 WF-01 온보딩 시퀀스

```
보호자앱  operate/api   trust/auth   memory_layer    오미디바이스
  │          │              │              │               │
  │ 어르신등록 │              │              │               │
  ├─POST /users─►│           │              │               │
  │          │─JWT검증──►│   │              │               │
  │          │◄─OK──────│   │              │               │
  │          │────────────────────────────►│               │
  │          │  초기 사용자 DB 생성          │               │
  │          │  6×4 빈 Coverage 초기화      │               │
  │          │                             │               │
  │          │─POST /auth/device──────────────────────────►│
  │          │              │              │  mTLS 인증서  │
  │          │◄────────────────────────── device_token ───│
  │          │              │              │               │
  │ 동의 설정  │              │              │               │
  ├─POST /users/{id}/consents►│            │               │
  │          │─consent_id 생성─────────────►│              │
  │          │                             │               │
  │ 온보딩 완료│              │              │               │
  │◄──────────│              │              │               │
  │          │              │              │               │
  │          │  첫 대화 시작 → 오미가 인사 + 첫 질문 생성   │
  │          │              │              │───get_next────►│
  │          │              │              │  question     │
```

### 7.4 WF-06 자서전 생성 시퀀스

```
보호자앱  operate/export  Celery Worker   EXAONE    S3       보호자앱
  │           │                │              │       │         │
  │ 주문요청   │                │              │       │         │
  ├─POST export►│              │              │       │         │
  │           │─Celery enqueue─►│             │       │         │
  │           │  job_id 반환   │              │       │         │
  │◄─{job_id}─│                │              │       │         │
  │           │                │ 청크 수집    │       │         │
  │           │                │──→DB 조회    │       │         │
  │           │                │              │       │         │
  │           │                │ LLM 서사 변환│       │         │
  │           │                │─────────────►│       │         │
  │           │                │ 1인칭 문단×N  │       │         │
  │           │                │◄─────────────│       │         │
  │           │                │              │       │         │
  │           │                │ PDF 렌더링   │       │         │
  │           │                │ WeasyPrint   │       │         │
  │           │                │              │       │         │
  │           │                │ TTS 오디오북  │       │         │
  │           │                │──────────────────────►│        │
  │           │                │  S3 업로드   │       │         │
  │           │                │──────────────────────►│        │
  │           │                │ job status=completed  │         │
  │           │                │ FCM Push──────────────────────►│
  │           │                │              │       │ 다운로드링크│
```

### 7.5 WF-08 감정 부조화 방지 시퀀스 (SC-10)

```
오미(Edge)  dialog_core  EmotionDetector  SafetyFilter  EXAONE   오미(Edge)
  │              │               │               │          │        │
  │ utterance    │               │               │          │        │
  ├──────────────►│              │               │          │        │
  │              │ 감정 분류 요청 │               │          │        │
  │              ├───────────────►│              │          │        │
  │              │               │sadness=0.73   │          │        │
  │              │◄──────────────│               │          │        │
  │              │               │               │          │        │
  │              │ RAG + LLM 응답 생성            │          │        │
  │              │──────────────────────────────────────────►│        │
  │              │ "와! 정말 잘 됐네요!" (감탄 응답)           │        │
  │              │◄──────────────────────────────────────────│        │
  │              │               │               │          │        │
  │              │ apply_emotion_mismatch_guard(sadness=0.73)│        │
  │              ├───────────────────────────────►│         │        │
  │              │ sadness ≥ 0.6 → 감탄 패턴 감지  │         │        │
  │              │ EMPATHY_RESPONSES 랜덤 선택     │         │        │
  │              │ "그때 참 속상하셨겠어요..."      │         │        │
  │              │◄──────────────────────────────│          │        │
  │              │               │               │          │        │
  │ 교체된 공감 응답 TTS                           │          │        │
  │◄─────────────│               │               │          │        │
  │              │               │               │          │        │
  [SC-10 검증: 슬픔 신호 시 감탄 응답 0건 달성]
```

---

## 8. 보안 설계

### 8.1 인증 흐름

```
[보호자 앱]
  POST /auth/login → JWT(access 1hr + refresh 30d)
  모든 API: Authorization: Bearer {access_token}
  토큰 만료: POST /auth/refresh → 새 access_token

[오미 디바이스]
  공장 출고 시 디바이스 인증서(X.509) 설치
  WebSocket 연결: mTLS 핸드셰이크 → device_token(24hr)
  device_token으로 WS 연결 유지
```

### 8.2 RBAC 미들웨어 구현

```python
# app/trust/auth/rbac.py
from enum import IntEnum
from functools import wraps
from fastapi import Depends, HTTPException

class Role(IntEnum):
    DEVICE = 1
    GUARDIAN_FAMILY = 2
    GUARDIAN_PRIMARY = 3
    FACILITY_ADMIN = 4
    SYSTEM_ADMIN = 5

PERMISSION_MAP = {
    "autobiography.read":    [Role.GUARDIAN_FAMILY, Role.GUARDIAN_PRIMARY,
                               Role.FACILITY_ADMIN, Role.SYSTEM_ADMIN],
    "autobiography.write":   [Role.DEVICE, Role.GUARDIAN_PRIMARY],
    "consent.manage":        [Role.GUARDIAN_PRIMARY, Role.SYSTEM_ADMIN],
    "anomaly.alert.receive": [Role.GUARDIAN_PRIMARY, Role.FACILITY_ADMIN],
    "system.model_deploy":   [Role.SYSTEM_ADMIN],
}

def require_permission(permission: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, current_user=Depends(get_current_user), **kwargs):
            allowed_roles = PERMISSION_MAP.get(permission, [])
            if current_user.role not in [r.value for r in allowed_roles]:
                raise HTTPException(403, detail="FORBIDDEN")
            return await func(*args, current_user=current_user, **kwargs)
        return wrapper
    return decorator
```

### 8.3 동의 Lineage 처리 흐름

```python
# app/trust/consent/consent_service.py
# 동의 철회 시 ≤24hr 내 cascade 처리 (SC-07)

async def revoke_consent(user_id: str, consent_id: str, db: AsyncSession):
    consent = await db.get(ConsentRecord, consent_id)
    consent.revoked_at = datetime.utcnow()
    
    if consent.data_category == "autobiography":
        await db.execute(
            update(AutobiographyChunk)
            .where(AutobiographyChunk.user_id == user_id)
            .where(AutobiographyChunk.consent_id == consent_id)
            .values(is_deleted=True)
        )
    
    # 감사 로그 (append-only)
    await audit_logger.record(
        action="consent_revoked",
        user_id=user_id,
        consent_id=str(consent_id),
        category=consent.data_category,
    )
    
    await db.commit()
    
    # Celery: 물리 삭제 및 S3 파일 삭제 (24hr 큐)
    purge_user_data.apply_async(
        args=[str(user_id), str(consent_id)],
        countdown=3600  # 1hr 후 실행 (검토 버퍼)
    )
```

---

## 9. 대화 엔진 상세 설계

### 9.1 상태머신 구현

```python
# app/dialog_core/orchestrator.py
from enum import Enum
import asyncio

class DialogState(Enum):
    IDLE = "idle"
    LISTENING = "listening"
    PROCESSING = "processing"
    RESPONDING = "responding"

class DialogOrchestrator:
    def __init__(self, memory: IMemoryLayer, safety: SafetyFilter,
                 llm: LLMClient, tts: TTSClient):
        self.memory = memory
        self.safety = safety
        self.llm = llm
        self.tts = tts
        self.state = DialogState.IDLE

    async def process_utterance(self, req: DialogRequest) -> AsyncIterator[DialogResponse]:
        self.state = DialogState.PROCESSING
        
        # 1. 안전 필터 (즉각 차단)
        safety_result = await self.safety.check(req.utterance_text)
        if safety_result.blocked:
            yield DialogResponse(
                text=safety_result.fallback_text,
                is_final=True,
                led_code="blue",
                emotion_detected="neutral",
                autobiography_chunk_saved=False,
                crisis_level=safety_result.crisis_level,
            )
            return

        # 2. RAG 검색 (병렬)
        rag_context_task = asyncio.create_task(
            self.memory.retrieve_context(req.user_id, req.utterance_text)
        )
        next_question_task = asyncio.create_task(
            self.memory.get_next_question(req.user_id)
        )
        
        rag_context = await rag_context_task
        next_period, next_topic = await next_question_task

        # 3. LLM 스트리밍
        self.state = DialogState.RESPONDING
        prompt = self._build_prompt(req, rag_context, next_period, next_topic)
        
        full_response = ""
        async for chunk in self.llm.stream_complete(prompt):
            full_response += chunk.text
            yield DialogResponse(
                text=chunk.text,
                is_final=chunk.is_final,
                led_code="green" if chunk.is_final else "orange_blink",
                emotion_detected=req.edge_emotion_hint,
                autobiography_chunk_saved=False,
                crisis_level=0,
            )
        
        # 4. 자서전 청크 추출·저장 (비동기, Celery)
        extract_and_save_chunks.delay(
            user_id=req.user_id,
            session_id=req.session_id,
            utterance=req.utterance_text,
            response=full_response,
            period=next_period,
            topic=next_topic,
        )
        
        self.state = DialogState.IDLE
```

### 9.2 안전 필터 상세

```python
# app/dialog_core/safety_filter.py
import re
from dataclasses import dataclass

@dataclass
class SafetyResult:
    blocked: bool
    fallback_text: str = ""
    crisis_level: int = 0  # 0=정상, 1~3=위기

BLOCK_PATTERNS = {
    "medical_assertion": re.compile(
        r"(치매\s*진단|치료됩니다|약을\s*끊어도|완치|병이\s*나았)", re.IGNORECASE
    ),
    "human_impersonation": re.compile(
        r"(저는\s*사람이에요|실제\s*사람|AI가\s*아니에요|진짜\s*사람)", re.IGNORECASE
    ),
    "crisis_direct": re.compile(
        r"(죽고\s*싶|자살|자해하|세상을\s*떠나고\s*싶|살기\s*싫)", re.IGNORECASE
    ),
    "crisis_indirect": re.compile(
        r"(힘들어\s*죽겠|너무\s*힘들어|아무도\s*없어|외로워\s*죽겠)", re.IGNORECASE
    ),
}

# 감정 부조화 방지 — 리빙랩 F-07 (SC-10)
# EmotionDetector가 sadness ≥ 0.6 판정 시, LLM 응답에 포함된
# 감탄·축하 표현을 EMPATHY_RESPONSES 중 랜덤으로 교체한다.
EMOTION_MISMATCH_GUARD = re.compile(
    r"(와!|좋겠네요!|정말\s*잘\s*됐네요|축하|기뻐요|신나네요)", re.IGNORECASE
)
EMPATHY_RESPONSES = [
    "많이 힘드셨겠다, 그 마음이 느껴져요.",
    "그때 참 속상하셨겠어요. 더 이야기해 주실 수 있어요?",
    "어르신, 제가 여기 있을게요. 천천히 말씀해 주세요.",
    "그런 일이 있으셨군요. 많이 힘드셨죠.",
]

CRISIS_FALLBACK = {
    1: "어르신, 많이 힘드시겠어요. 저도 항상 곁에 있을게요.",
    2: "어르신, 많이 힘드시구나. 가족분께 연락드릴게요.",
    3: "어르신, 많이 힘드시구나. 지금 바로 도움을 드릴게요.",
}

class SafetyFilter:
    async def check(self, text: str, emotion_score: float = 0.0) -> SafetyResult:
        # 위기 직접 발화 (L3)
        if BLOCK_PATTERNS["crisis_direct"].search(text):
            return SafetyResult(blocked=True, fallback_text=CRISIS_FALLBACK[3], crisis_level=3)
        
        # 위기 간접 발화 (L2)
        if BLOCK_PATTERNS["crisis_indirect"].search(text):
            return SafetyResult(blocked=False, fallback_text=CRISIS_FALLBACK[2], crisis_level=2)
        
        # 의료 단정 차단
        if BLOCK_PATTERNS["medical_assertion"].search(text):
            return SafetyResult(
                blocked=True,
                fallback_text="잘 모르겠어요. 가족분께 여쭤보시겠어요?",
                crisis_level=0,
            )
        
        # 인간 사칭 차단
        if BLOCK_PATTERNS["human_impersonation"].search(text):
            return SafetyResult(
                blocked=True,
                fallback_text="저는 어르신의 말동무 오미예요. 어르신 이야기를 듣고 싶어요.",
                crisis_level=0,
            )
        
        return SafetyResult(blocked=False, crisis_level=0)

    def apply_emotion_mismatch_guard(self, llm_response: str, sadness_score: float) -> str:
        """sadness ≥ 0.6 시 감탄·축하 표현을 공감 표현으로 교체 (SC-10)."""
        if sadness_score < 0.6:
            return llm_response
        import random
        if EMOTION_MISMATCH_GUARD.search(llm_response):
            return random.choice(EMPATHY_RESPONSES)
        return llm_response
```

---

## 10. Docker Compose 배포 설계 (Phase 1)

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build: .
    image: daon-app:latest
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql+asyncpg://daon:${PG_PASSWORD}@postgres:5432/daon
      - REDIS_URL=redis://redis:6379/0
      - EXAONE_API_URL=http://llm-server:8080/v1
      - JWT_SECRET=${JWT_SECRET}
      - S3_BUCKET=${S3_BUCKET}
    depends_on:
      postgres: {condition: service_healthy}
      redis: {condition: service_healthy}
    deploy:
      replicas: 2

  worker:
    build: .
    image: daon-app:latest
    command: celery -A workers.celery_app worker --loglevel=info -Q default,export,alert
    environment:
      - DATABASE_URL=postgresql+asyncpg://daon:${PG_PASSWORD}@postgres:5432/daon
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - app
    deploy:
      replicas: 2

  postgres:
    image: pgvector/pgvector:pg16
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations/init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      - POSTGRES_USER=daon
      - POSTGRES_PASSWORD=${PG_PASSWORD}
      - POSTGRES_DB=daon
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U daon"]
      interval: 10s
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/ssl/daon
    depends_on:
      - app

volumes:
  postgres_data:
  redis_data:

# docker-compose.llm.yml (별도, GPU 서버)
# EXAONE 3.5 추론 서버 (vLLM 또는 Ollama)
```

---

## 11. 구현 가이드

### 11.1 기술 스택 & 의존성

```toml
# pyproject.toml
[project]
name = "daon-platform"
requires-python = ">=3.12"
dependencies = [
    # Web Framework
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "python-multipart>=0.0.9",
    
    # Database
    "sqlalchemy[asyncio]>=2.0.30",
    "asyncpg>=0.29.0",
    "alembic>=1.13.0",
    "pgvector>=0.3.0",
    
    # Cache & Queue
    "redis[hiredis]>=5.0.0",
    "celery[redis]>=5.4.0",
    
    # AI/ML
    "sentence-transformers>=3.0.0",  # KoSimCSE
    "openai>=1.30.0",  # EXAONE API (OpenAI-compatible)
    "httpx>=0.27.0",
    
    # Auth & Security
    "python-jose[cryptography]>=3.3.0",
    "passlib[bcrypt]>=1.7.4",
    "pydantic-settings>=2.3.0",
    
    # Export
    "weasyprint>=62.0",
    "boto3>=1.34.0",  # S3
    
    # Monitoring
    "prometheus-fastapi-instrumentator>=7.0.0",
]

[tool.uv]
dev-dependencies = [
    "pytest>=8.2.0",
    "pytest-asyncio>=0.23.0",
    "httpx>=0.27.0",
    "ruff>=0.4.0",
    "mypy>=1.10.0",
]
```

### 11.2 구현 순서 (Phase 0→1)

#### Phase 0 — 기반 (2026.06, ~4주)

```
Week 1: 프로젝트 셋업
  □ Docker Compose 환경 구성 (PostgreSQL+pgvector, Redis)
  □ FastAPI 앱 기본 구조 (app/core/: config, database, dependencies)
  □ Alembic 마이그레이션 001~010 실행
  □ 공통 ORM 모델 (models/) + Pydantic 스키마 (schemas/)

Week 2: Trust (L5) — 인증 기반
  □ JWT 발급·검증 (jwt_service.py)
  □ RBAC 미들웨어 (rbac.py)
  □ mTLS 디바이스 인증 (device_auth.py)
  □ 동의 Lineage (consent_service.py)
  □ 감사 로그 (audit_logger.py)

Week 3: Memory Layer (L3) — 자서전 기반
  □ autobiography_service.py (청크 CRUD)
  □ embedding_service.py (KoSimCSE 로컬 로드)
  □ rag_retriever.py (pgvector 코사인 검색)
  □ question_priority.py (빈칸 탐지 + 스코어링)
  □ memory_graph.py (Apache AGE Cypher 쿼리)

Week 4: AI 모델 평가
  □ Whisper distil-large-v3 한국어 fine-tune 실험
  □ EXAONE 3.5 로컬 배포 (vLLM)
  □ KoSimCSE 한국어 임베딩 품질 검증
  □ 기본 대화 파이프라인 단독 테스트
```

#### Phase 1 — MVP 핵심 (2026.07~08, ~8주)

```
Month 1 (7월):
  □ Dialog Core (L2): orchestrator + persona_engine + safety_filter
  □ WebSocket 대화 서버 (operate/websocket/)
  □ 기본 REST API (auth, users, autobiography, sessions)
  □ Celery Workers (autobiography_processor, alert_worker)
  □ 오미 디바이스 연동 + 엣지 폴백 모드 테스트

Month 2 (8월):
  □ 위기 발화 감지 (crisis_detector)
  □ 감정 인식 모듈 (emotion_detector)
  □ 이상징후 알림 3단계 (anomaly → FCM push)
  □ 복약 알림 (medication_reminder)
  □ Flutter 보호자 앱 기본 화면 (홈·자서전·알림·설정)
  □ 통합 테스트 (SC-01~08 기준)
```

### 11.3 Session Guide (모듈별 구현 세션)

| 세션 | 모듈 | 주요 파일 | 예상 소요 | SC 연관 |
|---|---|---|---|---|
| S-01 | 기반 인프라 | core/, models/, migrations/ | 3일 | 전체 기반 |
| S-02 | Trust/Auth | trust/auth/, trust/consent/ | 2일 | SC-07 |
| S-03 | Memory Layer | memory_layer/ (CRUD+RAG) | 4일 | SC-02, SC-08 |
| S-04 | Dialog Core | dialog_core/ (오케스트레이터) | 4일 | SC-01, SC-04, SC-05 |
| S-05 | WebSocket 서버 | operate/websocket/ | 2일 | SC-01 |
| S-06 | REST API | operate/api/ | 3일 | SC-06 |
| S-07 | 위기 감지 + 감정 부조화 방지 | crisis_detector + safety_filter (EMOTION_MISMATCH_GUARD) | 2일 | SC-03, SC-10 |
| S-08 | Celery Workers | workers/ | 2일 | SC-07, SC-08 |
| S-09 | Flutter 앱 | mobile/ (기본 화면) | 5일 | - |
| S-10 | 자서전 생성기 | operate/export/ | 3일 | SC-08 |
| S-11 | MLOps 모니터 | trust/mlops/ | 2일 | SC-02, SC-03 |
| S-12 | 통합·E2E 테스트 | tests/ | 3일 | SC-01~08 전체 |
| S-13 | 방언 정규화기 | dialog_core/dialect_normalizer.py + fine-tune 스크립트 | 2일 | SC-09 |
| S-14 | 유산 패키지 + 알림 Tier | operate/export/legacy_package_builder.py + notifications/notification_tier.py | 2일 | SC-11, SC-12 |

**권장 세션 분할:**
```
/pdca do daon --scope S-01,S-02    # Week 1~2: 기반+인증
/pdca do daon --scope S-03,S-04    # Week 3~4: 메모리+대화
/pdca do daon --scope S-05,S-06    # Month 2 Week 1: API
/pdca do daon --scope S-07,S-08    # Month 2 Week 2: 위기+워커
/pdca do daon --scope S-09,S-10    # Month 3: 앱+생성기
/pdca do daon --scope S-11,S-12    # Month 3 후반: 모니터+테스트
/pdca do daon --scope S-13,S-14    # Month 3 말: 방언+유산패키지 (리빙랩 반영)
```

---

## 12. 테스트 계획

### 12.1 단위 테스트 (pytest)

| 대상 | 파일 | 핵심 테스트 케이스 |
|---|---|---|
| safety_filter | tests/unit/dialog_core/test_safety.py | 위기 L1~L3 패턴 감지, 의료 단정 차단, 정상 발화 통과 |
| question_priority | tests/unit/memory_layer/test_priority.py | 빈 coverage에서 우선순위 계산, 균형 분포 |
| rag_retriever | tests/unit/memory_layer/test_rag.py | pgvector 코사인 검색 결과 순서, is_private 필터 |
| rbac | tests/unit/trust/test_rbac.py | 각 Role별 permission 허용·거부 |
| consent_service | tests/unit/trust/test_consent.py | 동의 철회 시 is_deleted cascade |

### 12.2 통합 테스트

| 테스트 | 파일 | 검증 항목 |
|---|---|---|
| 대화 흐름 | test_conversation_flow.py | utterance→response 전체 파이프라인, P95 ≤1.5초 |
| RAG 파이프라인 | test_rag_pipeline.py | 임베딩 생성→저장→검색 end-to-end |
| 안전 필터 | test_safety_filter.py | 100개 위기 발화 테스트셋 감지율 ≥90% + 감정 부조화 차단 100% (SC-10) |
| 동의 삭제 | test_consent_cascade.py | 철회 후 24hr 내 완료 |
| 방언 정규화 | test_dialect_normalizer.py | 경상·전라·충청·제주 방언 100문장 WER ≤15% (SC-09) |
| 유산 패키지 | test_legacy_package.py | 생성 성공률 99.9%, 서명 URL TTL 90일 검증 (SC-12) |

### 12.3 성능 테스트 (SC-01 검증)

```python
# tests/integration/test_latency.py
import pytest
import asyncio
import time

@pytest.mark.asyncio
async def test_response_latency_p95():
    """SC-01: P95 응답 지연 ≤1,500ms"""
    latencies = []
    for _ in range(100):
        start = time.perf_counter()
        async for chunk in dialog_core.process_utterance(mock_request):
            if chunk.is_final:
                break
        first_token_latency = (time.perf_counter() - start) * 1000
        latencies.append(first_token_latency)
    
    latencies.sort()
    p95 = latencies[94]
    assert p95 <= 1500, f"P95 latency {p95:.0f}ms exceeds 1500ms"
```

---

## 13. 모니터링 & 운영

### 13.1 핵심 메트릭 (Prometheus)

| 메트릭 | 타입 | 설명 | SC 연관 |
|---|---|---|---|
| `dialog_response_latency_ms` | Histogram | Wake Word → 첫 TTS 음절 지연 | SC-01 |
| `autobiography_chunk_accuracy` | Gauge | 청크 분류 정확도 (사람 평가) | SC-02 |
| `crisis_detection_precision` | Gauge | 위기 발화 감지 정밀도 | SC-03 |
| `safety_filter_block_rate` | Counter | 안전 필터 차단 횟수 | SC-04 |
| `llm_hallucination_fallback_rate` | Gauge | 환각 폴백 비율 | SC-05 |
| `service_uptime` | Gauge | 서비스 가용성 | SC-06 |
| `autobiography_coverage_avg` | Gauge | 평균 Coverage (6×4 매트릭스) | SC-08 |
| `dialect_recognition_wer` | Gauge | 방언 발화 WER (낮을수록 좋음) | SC-09 |
| `emotion_mismatch_block_count` | Counter | 감정 부조화 차단 건수 (0이어야 함) | SC-10 |
| `notification_tier1_satisfaction` | Gauge | Tier 1 알림 만족도 점수 | SC-11 |
| `legacy_package_success_rate` | Gauge | 유산 패키지 생성 성공률 | SC-12 |

### 13.2 알림 기준

```yaml
# alerting_rules.yml
- alert: HighResponseLatency
  expr: histogram_quantile(0.95, dialog_response_latency_ms) > 1500
  for: 5m
  severity: critical

- alert: SafetyFilterFailure
  expr: safety_filter_bypass_count > 0
  for: 1m
  severity: critical  # SC-04: 즉각 대응

- alert: CrisisDetectionDrop
  expr: crisis_detection_precision < 0.90
  for: 15m
  severity: warning
```

---

## 14. 다음 단계

- **즉시:** `/pdca do daon --scope S-01,S-02` — 기반 인프라 + Trust 구현 시작
- **병행:** `/pdca plan daon-hw` — 오미 HW 아키텍처서
- **병행:** `/pdca plan daon-ux` — UX 가이드라인 상세화
