# 다온(Daon) SW 아키텍처 설계서

> **Feature:** daon-sw
> **작성일:** 2026-05-24
> **Phase:** Design
> **Status:** In Progress
> **Option:** C — 실용적 명세 (클래스 시그니처 + 핵심 알고리즘 + 완전한 DB/API 계약)
> **참조 Plan:** `docs/01-plan/features/daon-sw.plan.md`
> **참조 설계:** `docs/02-design/features/daon.design.md`

---

## Context Anchor

| Anchor | 값 |
|---|---|
| **WHY** | ADR 1~11이 확정된 지금, 개발팀이 독립적으로 각 모듈을 착수하려면 클래스·인터페이스·DB 마이그레이션 수준의 명세가 필요 |
| **WHO** | NUBiz AX 백엔드 개발자 (Python/FastAPI) · 앱 개발자 (Flutter) · MLOps 엔지니어 |
| **RISK** | EXAONE 3.5 GPU 요건 (VRAM ≥ 24GB) / pgvector IVFFlat 튜닝 / Celery 워커 장애 시 자서전 유실 |
| **SUCCESS** | 개발자 3명이 각 레이어 독립 착수 가능 / 로컬 dev 환경 30분 내 셋업 (DS-01) |
| **SCOPE** | (IN) L2~L5 SW 전체, Docker 개발환경, DB 마이그레이션, CI 파이프라인 개요 / (OUT) L1 HW, AWS/K8s 인프라, 픽셀 UI |

---

## §1. 시스템 전체 구조

### 1.1 5-Layer 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│  L1  Omi 디바이스 (Elixir + NervesOS)                           │
│       엣지 ASR (Whisper distil-large-v3) · 감정 프리-스코어링   │
│       WebSocket Client ──────────────────────────────────────┐  │
└────────────────────────────────────────────────────┬─────────┘  │
                                                     │ wss://      │
┌────────────────────────────────────────────────────▼─────────┐  │
│  L2  Dialog Core (FastAPI + Python 3.12)                      │  │
│       DialogOrchestrator → SafetyFilter → PersonaEngine       │  │
│       DialectNormalizer · EmotionDetector · CrisisDetector    │  │
│       LLMClient (EXAONE 3.5 / HyperCLOVA X fallback)        │  │
│       TTSClient                                               │  │
└───────────────────────┬───────────────────────────────────────┘  │
                        │ 내부 서비스 호출                           │
┌───────────────────────▼───────────────────────────────────────┐  │
│  L3  Memory Layer (PostgreSQL 16 + pgvector + Apache AGE)     │  │
│       AutobiographyService · RAGService · EmbeddingService    │  │
│       MemoryGraphService · SessionService                     │  │
│       Redis 7 캐시 레이어                                      │  │
└───────────────────────┬───────────────────────────────────────┘  │
                        │                                           │
┌───────────────────────▼───────────────────────────────────────┐  │
│  L4  Operate (REST API + Celery 5.x)                          │  │
│       FastAPI 라우터 (30 endpoints) · Celery 워커 (7 태스크)  │  │
│       NotificationService · LegacyPackageBuilder              │  │
│       Flutter 보호자 앱 ◄──── REST/JSON ────────────────────  │  │
└───────────────────────┬───────────────────────────────────────┘  │
                        │                                           │
┌───────────────────────▼───────────────────────────────────────┐  │
│  L5  Trust (횡단 관심사)                                        │  │
│       RBAC 미들웨어 · JWT · AES-256-GCM · 감사 로그 · 동의    │  │
└───────────────────────────────────────────────────────────────┘  │
```

### 1.2 서비스 경계 요약

| 경계 | 프로토콜 | 인증 |
|---|---|---|
| L1 Omi ↔ L2 Dialog | WebSocket (binary + JSON) | JWT query param |
| L4 Operate ↔ Flutter 앱 | HTTPS REST | Bearer JWT |
| L2 ↔ L3 내부 | 동일 프로세스 함수 호출 (Phase 1 모놀리스) | — |
| L4 ↔ Celery | Redis 메시지 큐 | — |
| LLMClient ↔ EXAONE | HTTP (로컬 vLLM 서버) | API Key |

---

## §2. 프로젝트 디렉터리 구조

```
daon/
├── src/
│   ├── api/                        # L4 FastAPI 라우터
│   │   ├── deps.py                 # 공통 의존성 (get_db, get_current_user)
│   │   ├── router.py               # API 라우터 등록
│   │   └── v1/
│   │       ├── auth.py             # POST /auth/token, /refresh, /logout
│   │       ├── users.py            # CRUD + 프로필
│   │       ├── autobiography.py    # 자서전 청크 + 타임라인
│   │       ├── sessions.py         # 대화 세션 + 주제 요약
│   │       ├── notifications.py    # 알림 이력 + 설정
│   │       ├── legacy.py           # 유산 패키지
│   │       ├── dashboard.py        # 시설 대시보드
│   │       └── admin.py            # 관리자
│   │
│   ├── core/                       # 공통 인프라
│   │   ├── config.py               # Pydantic Settings (환경변수)
│   │   ├── database.py             # SQLAlchemy async engine + session factory
│   │   ├── redis.py                # aioredis 클라이언트 팩토리
│   │   ├── security.py             # JWT 생성/검증, AES-256-GCM
│   │   └── logging.py              # structlog 설정
│   │
│   ├── dialog/                     # L2 Dialog Core
│   │   ├── orchestrator.py         # DialogOrchestrator
│   │   ├── safety.py               # SafetyFilter + EmotionMismatchGuard
│   │   ├── dialect.py              # DialectNormalizer
│   │   ├── emotion.py              # EmotionDetector
│   │   ├── crisis.py               # CrisisDetector
│   │   ├── persona.py              # PersonaEngine
│   │   ├── llm_client.py           # LLMClient (EXAONE + fallback)
│   │   └── tts_client.py           # TTSClient
│   │
│   ├── memory/                     # L3 Memory Layer
│   │   ├── autobiography.py        # AutobiographyService
│   │   ├── rag.py                  # RAGService
│   │   ├── embedding.py            # EmbeddingService
│   │   ├── graph.py                # MemoryGraphService (Apache AGE)
│   │   └── session_svc.py          # SessionService
│   │
│   ├── operate/                    # L4 Operate
│   │   ├── notification.py         # NotificationService + TierManager
│   │   ├── legacy_package.py       # LegacyPackageBuilder
│   │   ├── weekly_report.py        # WeeklyReportBuilder
│   │   └── fcm_client.py           # FCMClient
│   │
│   ├── trust/                      # L5 Trust
│   │   ├── rbac.py                 # RBAC 미들웨어 + Permission enum
│   │   ├── encryption.py           # AES-256-GCM 래퍼
│   │   ├── audit.py                # 감사 로그 서비스
│   │   └── consent.py              # 동의 관리
│   │
│   ├── models/                     # SQLAlchemy ORM 모델
│   │   ├── user.py
│   │   ├── autobiography.py
│   │   ├── session.py
│   │   ├── notification.py
│   │   ├── legacy.py
│   │   ├── audit.py
│   │   └── guardian.py
│   │
│   ├── schemas/                    # Pydantic 요청/응답 스키마
│   │   ├── auth.py
│   │   ├── user.py
│   │   ├── autobiography.py
│   │   ├── session.py
│   │   ├── notification.py
│   │   ├── legacy.py
│   │   └── common.py               # Pagination, ErrorResponse
│   │
│   ├── repositories/               # DB 접근 추상화
│   │   ├── base.py                 # BaseRepository[T] — CRUD 패턴
│   │   ├── user_repo.py
│   │   ├── autobiography_repo.py
│   │   └── session_repo.py
│   │
│   ├── tasks/                      # Celery 태스크
│   │   ├── celery_app.py           # Celery 앱 설정
│   │   ├── dialog_tasks.py         # process_autobiography_chunk
│   │   ├── memory_tasks.py         # update_memory_graph, generate_topic_summary
│   │   ├── notification_tasks.py   # send_tier1_notification
│   │   ├── report_tasks.py         # build_weekly_report
│   │   ├── export_tasks.py         # build_legacy_package
│   │   └── maintenance_tasks.py    # cleanup_expired_signed_urls
│   │
│   ├── websocket/
│   │   └── dialog_ws.py            # WebSocket 핸들러 (/ws/dialog/{session_id})
│   │
│   └── main.py                     # FastAPI 앱 엔트리포인트
│
├── alembic/
│   ├── env.py
│   └── versions/                   # 마이그레이션 파일들
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── scripts/
│   ├── seed_dev.py                 # 개발 시드 데이터
│   └── check_doc_consistency.ps1
│
├── app/                            # Flutter 앱
│   └── lib/
│       ├── screens/
│       ├── providers/
│       ├── services/
│       └── models/
│
├── docker-compose.yml
├── docker-compose.prod.yml
├── Dockerfile
├── pyproject.toml                  # Poetry 의존성
├── alembic.ini
└── .env.example
```

---

## §3. L2 Dialog Core 상세 설계

### 3.1 DialogOrchestrator

```python
# src/dialog/orchestrator.py

class DialogOrchestrator:
    """대화 세션의 전체 흐름을 제어하는 메인 오케스트레이터."""

    def __init__(
        self,
        safety_filter: SafetyFilter,
        dialect_normalizer: DialectNormalizer,
        emotion_detector: EmotionDetector,
        crisis_detector: CrisisDetector,
        persona_engine: PersonaEngine,
        rag_service: RAGService,
        autobiography_service: AutobiographyService,
        llm_client: LLMClient,
        tts_client: TTSClient,
        session_service: SessionService,
    ) -> None: ...

    async def process_utterance(
        self,
        session_id: UUID,
        audio_bytes: bytes,
        user: User,
    ) -> AsyncIterator[ResponseChunk]:
        """
        오미 디바이스에서 수신한 음성을 처리하고 응답을 스트리밍 반환.

        흐름:
        1. ASR: audio_bytes → text (Whisper, 엣지에서 수행 — 이미 텍스트로 수신)
        2. 방언 정규화: DialectNormalizer.normalize(text, user.dialect)
        3. 감정 분류: EmotionDetector.classify(text)
        4. 위기 감지: CrisisDetector.detect_level(text, emotion_score)
           → Level 3: 즉시 응급 알림 후 세션 중단
        5. 안전 필터: SafetyFilter.check(emotion_score, planned_response_tone)
        6. 컨텍스트 빌드: RAGService.retrieve(text, user_id)
        7. 시스템 프롬프트: PersonaEngine.build_system_prompt(user, rag_context)
        8. LLM 스트리밍: LLMClient.generate_stream(messages)
        9. TTS 스트리밍: TTSClient.synthesize_stream(text_chunk)
        10. 세션 저장: SessionService.save_turn(session_id, utterance, response)
        11. 비동기 태스크: process_autobiography_chunk.delay(turn_id)
        """

    async def _run_safety_check(
        self,
        emotion_score: EmotionScore,
        response_draft: str,
    ) -> SafetyCheckResult:
        """EmotionMismatchGuard: sadness ≥ 0.6 시 축하 응답 차단."""

    def _build_state_machine(self, user: User) -> DialogStateMachine:
        """치매 단계별 대화 상태 머신 초기화 (MCI/초로기/중등도/중증)."""
```

### 3.2 SafetyFilter + EmotionMismatchGuard

```python
# src/dialog/safety.py

EMPATHY_RESPONSES = [
    "많이 힘드셨겠어요. 제가 곁에 있을게요.",
    "그런 마음이 드셨군요. 천천히 이야기해 주세요.",
    "지금 많이 어려우신 것 같아요. 함께 있을게요.",
]

@dataclass
class EmotionScore:
    sadness: float      # 0.0 ~ 1.0
    joy: float
    anger: float
    neutral: float

@dataclass
class SafetyCheckResult:
    allowed: bool
    override_response: str | None   # None이면 원래 응답 사용

class SafetyFilter:
    """감정 부조화 방지 필터 — SC-10 감정 부조화 발생률 0% 보장."""

    SADNESS_THRESHOLD = 0.6         # V-01 기준값

    def check(
        self,
        emotion_score: EmotionScore,
        planned_response: str,
    ) -> SafetyCheckResult:
        """
        알고리즘:
        1. emotion_score.sadness >= SADNESS_THRESHOLD 확인
        2. planned_response가 축하/긍정 어조인지 분류
           (키워드: ["축하", "좋아요", "잘됐네요", "신나는", "기뻐요"])
        3. 조건 둘 다 True → allowed=False, override=EMPATHY_RESPONSES 랜덤 선택
        4. 아니면 allowed=True
        """

    def _is_celebratory_tone(self, text: str) -> bool:
        CELEBRATORY_KEYWORDS = ["축하", "좋아요", "잘됐네요", "신나는", "기뻐요", "대단해요"]
        return any(kw in text for kw in CELEBRATORY_KEYWORDS)
```

### 3.3 DialectNormalizer

```python
# src/dialog/dialect.py

class DialectType(str, Enum):
    STANDARD = "standard"
    GYEONGSANG = "gyeongsang"   # 경상
    JEOLLA = "jeolla"           # 전라
    CHUNGCHEONG = "chungcheong" # 충청
    JEJU = "jeju"               # 제주

class DialectNormalizer:
    """방언 인식 및 표준어 정규화 — SC-09 방언 인식률 ≥ 85% 보장."""

    def __init__(self, model_path: str) -> None:
        # Whisper fine-tune 모델 (방언별 학습)
        self._model = WhisperFineTune.load(model_path)

    def normalize(self, text: str, dialect: DialectType) -> str:
        """방언 텍스트 → 표준어 변환."""

    def detect_dialect(self, audio_bytes: bytes) -> tuple[str, DialectType]:
        """음성 → (표준어 텍스트, 감지된 방언 유형)."""
```

### 3.4 EmotionDetector

```python
# src/dialog/emotion.py

class EmotionDetector:
    """KoBERT 계열 감정 분류 모델 래퍼."""

    def __init__(self, model_name: str = "snunlp/KR-FinBert-SC") -> None:
        self._pipeline = pipeline("text-classification", model=model_name)

    def classify(self, text: str) -> EmotionScore:
        """텍스트 → EmotionScore (sadness/joy/anger/neutral 합산 1.0)."""
        results = self._pipeline(text, top_k=4)
        return EmotionScore(**{r["label"].lower(): r["score"] for r in results})
```

### 3.5 CrisisDetector

```python
# src/dialog/crisis.py

class CrisisLevel(int, Enum):
    NONE = 0
    LEVEL_1 = 1     # 경미한 불안/슬픔 → Tier 2 인앱 알림
    LEVEL_2 = 2     # 심각한 우울/자해 언급 → Tier 1 Push + 보호자 즉시 알림
    LEVEL_3 = 3     # 즉각 위험 → 긴급 전화 연결 + 세션 중단

CRISIS_KEYWORDS = {
    CrisisLevel.LEVEL_3: ["죽고 싶", "죽을 것 같", "살기 싫", "없어지고 싶"],
    CrisisLevel.LEVEL_2: ["힘들어 죽겠", "못 살겠", "괴로워 죽겠"],
    CrisisLevel.LEVEL_1: ["너무 슬프", "외로워", "아무도 없"],
}

class CrisisDetector:
    def detect_level(self, text: str, emotion: EmotionScore) -> CrisisLevel:
        """
        알고리즘:
        1. CRISIS_KEYWORDS 순차 매칭 (Level 3 → 1 우선순위)
        2. emotion.sadness >= 0.8 AND 키워드 없을 때 → Level 1 상향 가능
        3. LLM 2차 검증 (Level 2+ 의심 시, 낮은 temperature)
        """
```

### 3.6 PersonaEngine

```python
# src/dialog/persona.py

DEMENTIA_STAGE_PROMPTS = {
    "MCI": "천천히 명확하게 말씀해 주시고, 반복을 자연스럽게 받아들이세요.",
    "early_onset": "짧은 문장을 사용하고, 친숙한 주제로 자주 돌아오세요.",
    "moderate": "단순한 예/아니오 질문을 활용하고, 즉각 긍정 피드백을 주세요.",
    "severe": "감각적 자극과 감정 반응에 집중하고 인지적 요구를 최소화하세요.",
}

class PersonaEngine:
    def build_system_prompt(
        self,
        user: User,
        rag_context: list[RAGChunk],
        session_history: list[Turn],
    ) -> str:
        """
        시스템 프롬프트 구성:
        1. 기본 페르소나: 어르신 이름, 출생연도, 방언 지역
        2. 치매 단계별 가이드라인: DEMENTIA_STAGE_PROMPTS[user.dementia_stage]
        3. RAG 컨텍스트: 최근 관련 자서전 청크 3~5개
        4. 대화 기록: 최근 5턴
        5. 오늘 자서전 목표 시기: autobiography_coverage에서 미작성 period
        """
```

### 3.7 LLMClient

```python
# src/dialog/llm_client.py

class LLMClient:
    """EXAONE 3.5 (로컬 vLLM) + HyperCLOVA X (API 폴백)."""

    def __init__(self, config: LLMConfig) -> None:
        self._primary = vLLMClient(base_url=config.exaone_url)
        self._fallback = HyperCLOVAClient(api_key=config.hyperclova_key)

    async def generate_stream(
        self,
        messages: list[Message],
        temperature: float = 0.7,
        max_tokens: int = 512,
    ) -> AsyncIterator[str]:
        """
        1. primary 시도 (timeout=3s)
        2. ConnectionError 또는 timeout → fallback으로 전환
        3. 각 청크를 yield (SSE 스트리밍)
        """
```

---

## §4. L3 Memory Layer 상세 설계

### 4.1 AutobiographyService

```python
# src/memory/autobiography.py

PERIODS = ["유년기", "학창시절", "청년기", "중년기", "현재"]

@dataclass
class AutobiographyCoverage:
    period: str
    topic_count: int
    last_updated: datetime
    confidence_avg: float

class AutobiographyService:
    def __init__(
        self,
        repo: AutobiographyRepository,
        embedding_svc: EmbeddingService,
    ) -> None: ...

    async def save_chunk(
        self,
        user_id: UUID,
        period: str,                # PERIODS 중 하나
        topic: str,
        content: str,               # 평문 (AES-256-GCM 암호화 후 저장)
        confidence: float,
    ) -> AutobiographyChunk:
        """
        1. content 암호화: EncryptionService.encrypt(content)
        2. embedding 생성: EmbeddingService.embed(content)
        3. DB 저장: repo.upsert(user_id, period, topic, encrypted_content, embedding)
        4. 비동기 태스크: update_memory_graph.delay(chunk_id)
        """

    async def get_coverage(self, user_id: UUID) -> list[AutobiographyCoverage]:
        """사용자별 5개 시기(PERIODS)의 자서전 작성 현황 반환."""

    async def get_next_question_target(self, user_id: UUID) -> tuple[str, str]:
        """
        가장 작성이 적은 period와 topic을 반환 (LLM 질문 생성용).
        알고리즘: min(topic_count) 시기 → 해당 시기 내 미작성 토픽 목록 중 랜덤
        """
```

### 4.2 RAGService

```python
# src/memory/rag.py

@dataclass
class RAGChunk:
    chunk_id: UUID
    period: str
    topic: str
    content: str        # 복호화된 평문
    similarity: float

class RAGService:
    RETRIEVE_TOP_K = 5
    CACHE_TTL = 300     # 5분

    def __init__(
        self,
        db: AsyncSession,
        cache: Redis,
        embedding_svc: EmbeddingService,
    ) -> None: ...

    async def retrieve(
        self,
        query: str,
        user_id: UUID,
        top_k: int = RETRIEVE_TOP_K,
    ) -> list[RAGChunk]:
        """
        알고리즘:
        1. 캐시 확인: cache.get(f"rag:{user_id}:{hash(query)}")
        2. 캐시 미스 → 쿼리 임베딩: EmbeddingService.embed(query)
        3. pgvector 코사인 유사도 검색:
           SELECT ... FROM autobiography_chunks
           WHERE user_id = :uid
           ORDER BY embedding <=> :query_vec
           LIMIT :top_k
        4. AES 복호화: EncryptionService.decrypt(chunk.content)
        5. 캐시 저장 (TTL=300s)
        6. RAGChunk 리스트 반환
        """

    async def build_context(self, chunks: list[RAGChunk]) -> str:
        """RAGChunk 리스트 → LLM 시스템 프롬프트용 컨텍스트 문자열."""
```

### 4.3 EmbeddingService

```python
# src/memory/embedding.py

class EmbeddingService:
    """KoSimCSE-roberta 임베딩 (768차원)."""

    MODEL_NAME = "BM-K/KoSimCSE-roberta"
    DIMENSION = 768

    def __init__(self) -> None:
        self._model = SentenceTransformer(self.MODEL_NAME)

    def embed(self, text: str) -> list[float]:
        """텍스트 → 768차원 정규화 벡터."""
        return self._model.encode(text, normalize_embeddings=True).tolist()

    def embed_batch(self, texts: list[str]) -> list[list[float]]:
        """배치 임베딩 (대량 청크 처리용)."""
```

### 4.4 MemoryGraphService

```python
# src/memory/graph.py

class MemoryGraphService:
    """Apache AGE (Cypher) 기반 인물·사건·장소 관계 그래프."""

    async def upsert_entity(
        self,
        user_id: UUID,
        entity_type: str,   # "Person" | "Event" | "Place"
        name: str,
        properties: dict,
    ) -> None:
        """
        Cypher:
        MERGE (e:{entity_type} {name: $name, user_id: $user_id})
        ON CREATE SET e += $properties
        ON MATCH SET e += $properties
        """

    async def upsert_relation(
        self,
        from_entity: str,
        relation_type: str, # "KNOWS" | "ATTENDED" | "LIVED_IN"
        to_entity: str,
        properties: dict = {},
    ) -> None:
        """엔티티 간 관계 생성/갱신."""

    async def query_related(
        self,
        user_id: UUID,
        entity_name: str,
        depth: int = 2,
    ) -> list[dict]:
        """
        Cypher:
        MATCH (start {name: $name, user_id: $user_id})-[*1..{depth}]-(related)
        RETURN related, labels(related), relationships
        """

    # Phase 1 대안: Apache AGE 미설치 환경에서는 JSONB로 fallback
    async def upsert_entity_jsonb_fallback(
        self, user_id: UUID, entity_type: str, name: str, properties: dict
    ) -> None:
        """autobiography_chunks.metadata JSONB에 엔티티 저장 (AGE 없을 때)."""
```

---

## §5. L4 Operate 상세 설계

### 5.1 NotificationService + TierManager

```python
# src/operate/notification.py

class NotificationTier(str, Enum):
    TIER_1 = "tier1"    # 긴급 Push (FCM)
    TIER_2 = "tier2"    # 인앱 알림
    TIER_3 = "tier3"    # 주간 리포트

class TierClassificationRule:
    """ADR-11 알림 Tier 분류 규칙."""
    TIER_1_TRIGGERS = [CrisisLevel.LEVEL_2, CrisisLevel.LEVEL_3]
    TIER_2_TRIGGERS = [CrisisLevel.LEVEL_1, "session_anomaly", "medication_reminder"]
    TIER_3_SCHEDULE = "0 8 * * MON"  # 매주 월요일 08:00

class NotificationTierManager:
    def classify_event(
        self, event_type: str, context: dict
    ) -> NotificationTier:
        """
        분류 흐름:
        1. event_type이 TIER_1_TRIGGERS → TIER_1
        2. event_type이 TIER_2_TRIGGERS → TIER_2
        3. 그 외 → TIER_3 (주간 리포트 버퍼)
        """

    async def route_to_tier(
        self, tier: NotificationTier, payload: NotificationPayload
    ) -> None:
        """Tier → 해당 전송 채널로 라우팅."""

class NotificationService:
    def __init__(
        self,
        tier_manager: NotificationTierManager,
        fcm_client: FCMClient,
        repo: NotificationRepository,
    ) -> None: ...

    async def send_tier1(
        self,
        guardian_id: UUID,
        title: str,
        body: str,
        data: dict = {},
    ) -> None:
        """FCM Push + DB 이력 저장."""

    async def send_tier2(
        self,
        user_id: UUID,
        title: str,
        body: str,
    ) -> None:
        """인앱 알림 DB 저장 (FCM 없음)."""

    async def schedule_tier3_report(self, user_id: UUID) -> None:
        """build_weekly_report Celery 태스크 예약."""
```

### 5.2 LegacyPackageBuilder

```python
# src/operate/legacy_package.py

SIGNED_URL_TTL_DAYS = 90    # V-03

class LegacyPackageBuilder:
    """유산 패키지 생성 — SC-12 생성 성공률 ≥ 99.9%."""

    def __init__(
        self,
        autobiography_svc: AutobiographyService,
        s3_client: S3Client,
        repo: LegacyJobRepository,
    ) -> None: ...

    async def build(self, job_id: UUID, user_id: UUID) -> str:
        """
        1. 자서전 청크 전체 조회 (복호화)
        2. PDF/EPUB 생성 (reportlab / ebooklib)
        3. S3 업로드: s3://daon-legacy/{user_id}/{job_id}.pdf
        4. Signed URL 생성 (TTL=90일)
        5. legacy_jobs 업데이트: status=DONE, signed_url, signed_url_expires_at
        6. Tier 1 알림: 보호자에게 다운로드 링크 Push
        """

    async def upload_to_s3(self, local_path: str, s3_key: str) -> str:
        """S3 업로드 후 object key 반환."""

    async def generate_signed_url(self, s3_key: str) -> tuple[str, datetime]:
        """90일 서명 URL + 만료일시 반환."""
```

---

## §6. L5 Trust 상세 설계

### 6.1 RBAC

```python
# src/trust/rbac.py

class UserRole(str, Enum):
    SYSTEM_ADMIN = "system_admin"
    FACILITY_ADMIN = "facility_admin"
    GUARDIAN_PRIMARY = "guardian_primary"
    GUARDIAN_FAMILY = "guardian_family"
    DEVICE = "device"

ROLE_PERMISSIONS: dict[UserRole, set[str]] = {
    UserRole.SYSTEM_ADMIN: {"*"},
    UserRole.FACILITY_ADMIN: {
        "users:read", "users:write", "dashboard:read", "audit:read"
    },
    UserRole.GUARDIAN_PRIMARY: {
        "users:read", "autobiography:read", "sessions:read",
        "notifications:read", "notifications:write", "legacy:read", "legacy:create"
    },
    UserRole.GUARDIAN_FAMILY: {
        "users:read", "sessions:read", "notifications:read"
    },
    UserRole.DEVICE: {"dialog:write", "sessions:write"},
}

def require_permission(permission: str):
    """FastAPI 의존성 — 권한 부족 시 HTTP 403."""
    async def checker(current_user: User = Depends(get_current_user)) -> User:
        user_perms = ROLE_PERMISSIONS.get(current_user.role, set())
        if "*" not in user_perms and permission not in user_perms:
            raise HTTPException(status_code=403, detail="권한이 없습니다.")
        return current_user
    return checker
```

### 6.2 JWT + Encryption

```python
# src/core/security.py

class SecurityService:
    """JWT 생성/검증 + AES-256-GCM 암호화."""

    # JWT
    def create_access_token(self, user_id: UUID, role: UserRole) -> str:
        """exp=30분, 알고리즘 HS256."""

    def create_refresh_token(self, user_id: UUID) -> str:
        """exp=30일, Redis에 저장 (jti → user_id 매핑)."""

    def verify_token(self, token: str) -> TokenPayload:
        """만료/변조 시 HTTP 401."""

    # AES-256-GCM
    def encrypt(self, plaintext: str) -> str:
        """random nonce(12B) + ciphertext → base64 인코딩."""

    def decrypt(self, ciphertext_b64: str) -> str:
        """base64 디코딩 → nonce 분리 → GCM 복호화."""
```

### 6.3 감사 로그

```python
# src/trust/audit.py

class AuditService:
    async def log(
        self,
        actor_id: UUID,
        action: str,            # "READ" | "WRITE" | "DELETE" | "EXPORT"
        resource_type: str,     # "autobiography_chunk" | "session" | "legacy_job"
        resource_id: UUID,
        ip: str,
        metadata: dict = {},
    ) -> None:
        """audit_logs 테이블에 비동기 삽입."""
```

---

## §7. 데이터베이스 설계

### 7.1 완전한 ERD (테이블 정의)

```sql
-- users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    birth_year      SMALLINT,
    dementia_stage  TEXT CHECK (dementia_stage IN ('MCI','early_onset','moderate','severe')),
    persona         JSONB,              -- {nickname, tone, interests[]}
    dialect         TEXT DEFAULT 'standard'
                    CHECK (dialect IN ('standard','gyeongsang','jeolla','chungcheong','jeju')),
    facility_id     UUID REFERENCES facilities(id),
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- autobiography_chunks
CREATE TABLE autobiography_chunks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    period          TEXT NOT NULL
                    CHECK (period IN ('유년기','학창시절','청년기','중년기','현재')),
    topic           TEXT NOT NULL,
    content         TEXT NOT NULL,      -- AES-256-GCM 암호화
    embedding       vector(768),        -- pgvector
    confidence      FLOAT CHECK (confidence BETWEEN 0 AND 1),
    session_id      UUID REFERENCES sessions(id),
    created_at      TIMESTAMPTZ DEFAULT now(),
    updated_at      TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON autobiography_chunks USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX ON autobiography_chunks (user_id, period);

-- sessions
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    started_at      TIMESTAMPTZ DEFAULT now(),
    ended_at        TIMESTAMPTZ,
    turn_count      INT DEFAULT 0,
    emotion_summary JSONB,              -- {sadness_avg, joy_avg, anomaly_detected}
    topic_summary   TEXT                -- generate_topic_summary Celery 결과
);

-- session_turns
CREATE TABLE session_turns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    speaker         TEXT CHECK (speaker IN ('user','assistant')),
    text            TEXT NOT NULL,      -- 평문 (분석용, 장기 보관 필요 시 암호화)
    emotion_score   JSONB,              -- EmotionScore {sadness, joy, anger, neutral}
    crisis_level    SMALLINT DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- notifications
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    tier            TEXT CHECK (tier IN ('tier1','tier2','tier3')),
    title           TEXT NOT NULL,
    body            TEXT NOT NULL,
    data            JSONB DEFAULT '{}',
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- notification_settings
CREATE TABLE notification_settings (
    user_id         UUID PRIMARY KEY REFERENCES users(id),
    tier1_push      BOOLEAN DEFAULT TRUE,
    tier2_inapp     BOOLEAN DEFAULT TRUE,
    tier3_weekly    BOOLEAN DEFAULT TRUE,
    quiet_start     TIME DEFAULT '22:00',
    quiet_end       TIME DEFAULT '08:00',
    updated_at      TIMESTAMPTZ DEFAULT now()
);

-- legacy_jobs
CREATE TABLE legacy_jobs (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id                 UUID NOT NULL REFERENCES users(id),
    status                  TEXT DEFAULT 'pending'
                            CHECK (status IN ('pending','running','done','failed')),
    s3_key                  TEXT,
    signed_url              TEXT,
    signed_url_expires_at   TIMESTAMPTZ,
    error_message           TEXT,
    created_at              TIMESTAMPTZ DEFAULT now(),
    updated_at              TIMESTAMPTZ DEFAULT now()
);

-- consents
CREATE TABLE consents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    consent_type    TEXT NOT NULL,      -- "privacy_policy" | "voice_recording" | "ai_analysis"
    version         TEXT NOT NULL,      -- "1.0", "1.1" ...
    agreed_at       TIMESTAMPTZ DEFAULT now(),
    revoked_at      TIMESTAMPTZ
);

-- audit_logs
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id        UUID NOT NULL,      -- users.id 또는 system
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    ip              INET,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX ON audit_logs (actor_id, created_at DESC);

-- guardians
CREATE TABLE guardians (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    guardian_id     UUID NOT NULL REFERENCES users(id),
    role            TEXT CHECK (role IN ('PRIMARY','FAMILY')),
    fcm_token       TEXT,
    created_at      TIMESTAMPTZ DEFAULT now(),
    UNIQUE (user_id, guardian_id)
);
```

### 7.2 Alembic 마이그레이션 템플릿

```python
# alembic/versions/2026_05_24_0001_initial_schema.py
"""Initial schema — all tables."""

from alembic import op
import sqlalchemy as sa
from pgvector.sqlalchemy import Vector

def upgrade() -> None:
    op.execute("CREATE EXTENSION IF NOT EXISTS vector;")
    op.execute("CREATE EXTENSION IF NOT EXISTS age;")
    # ... CREATE TABLE 문 순서대로 실행
    # 실행: alembic upgrade head

def downgrade() -> None:
    # DROP TABLE 역순
    pass
```

---

## §8. API 명세 (핵심 30개)

### 8.1 공통 Pydantic 스키마

```python
# src/schemas/common.py

class PaginationParams(BaseModel):
    page: int = 1
    size: int = 20

class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    size: int

class ErrorResponse(BaseModel):
    code: str
    message: str
    detail: dict = {}
```

### 8.2 인증 (3개)

| Method | Path | Request | Response |
|---|---|---|---|
| POST | `/auth/token` | `{username, password}` | `{access_token, refresh_token, token_type}` |
| POST | `/auth/refresh` | `{refresh_token}` | `{access_token}` |
| POST | `/auth/logout` | Bearer header | `204 No Content` |

```python
# src/schemas/auth.py
class TokenRequest(BaseModel):
    username: str
    password: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
```

### 8.3 사용자 (5개)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| GET | `/users/{id}` | GUARDIAN_PRIMARY+ | 사용자 프로필 조회 |
| PATCH | `/users/{id}` | FACILITY_ADMIN+ | 프로필 업데이트 |
| GET | `/users/{id}/guardians` | GUARDIAN_PRIMARY+ | 보호자 목록 |
| POST | `/users/{id}/guardians` | FACILITY_ADMIN+ | 보호자 추가 |
| DELETE | `/users/{id}/guardians/{gid}` | FACILITY_ADMIN+ | 보호자 제거 |

```python
# src/schemas/user.py
class UserProfile(BaseModel):
    id: UUID
    name: str
    birth_year: int | None
    dementia_stage: str
    dialect: str
    persona: dict

class UserUpdateRequest(BaseModel):
    dementia_stage: str | None = None
    dialect: str | None = None
    persona: dict | None = None
```

### 8.4 자서전 (6개)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| GET | `/users/{id}/autobiography` | GUARDIAN_PRIMARY+ | 전체 타임라인 |
| GET | `/users/{id}/autobiography/coverage` | GUARDIAN_PRIMARY+ | 시기별 작성 현황 |
| GET | `/users/{id}/autobiography/{period}` | GUARDIAN_PRIMARY+ | 특정 시기 청크 목록 |
| POST | `/users/{id}/autobiography` | DEVICE | 청크 저장 (내부용) |
| PATCH | `/users/{id}/autobiography/{chunk_id}` | FACILITY_ADMIN+ | 청크 수정 |
| DELETE | `/users/{id}/autobiography/{chunk_id}` | SYSTEM_ADMIN | 청크 삭제 |

```python
# src/schemas/autobiography.py
class AutobiographyChunkResponse(BaseModel):
    id: UUID
    period: str
    topic: str
    content: str    # 복호화된 평문
    confidence: float
    created_at: datetime

class CoverageResponse(BaseModel):
    period: str
    topic_count: int
    last_updated: datetime | None
    confidence_avg: float
```

### 8.5 세션 (4개)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| POST | `/users/{id}/sessions` | DEVICE | 세션 시작 |
| PATCH | `/users/{id}/sessions/{sid}` | DEVICE | 세션 종료 |
| GET | `/users/{id}/sessions` | GUARDIAN_PRIMARY+ | 세션 목록 |
| GET | `/users/{id}/sessions/{sid}/topic-summary` | GUARDIAN_PRIMARY+ | 주제 요약 (A-02) |

### 8.6 알림 (4개)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| GET | `/users/{id}/notifications` | GUARDIAN_PRIMARY+ | 알림 이력 (A-01) |
| PATCH | `/users/{id}/notification-settings` | GUARDIAN_PRIMARY+ | 알림 설정 변경 (A-01b) |
| POST | `/users/{id}/notifications/{nid}/read` | GUARDIAN_PRIMARY+ | 읽음 처리 |
| GET | `/users/{id}/notifications/weekly-report` | GUARDIAN_PRIMARY+ | 주간 리포트 |

```python
# src/schemas/notification.py
class NotificationResponse(BaseModel):
    id: UUID
    tier: str
    title: str
    body: str
    read_at: datetime | None
    created_at: datetime

class NotificationSettingsRequest(BaseModel):
    tier1_push: bool | None = None
    tier2_inapp: bool | None = None
    tier3_weekly: bool | None = None
    quiet_start: str | None = None  # "HH:MM"
    quiet_end: str | None = None
```

### 8.7 유산 패키지 (3개)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| POST | `/users/{id}/legacy-package` | GUARDIAN_PRIMARY+ | 생성 요청 (A-03) |
| GET | `/users/{id}/legacy-package/{job_id}` | GUARDIAN_PRIMARY+ | 상태 조회 |
| GET | `/users/{id}/legacy-package/{job_id}/download` | GUARDIAN_PRIMARY+ | 다운로드 URL (A-03b) |

```python
# src/schemas/legacy.py
class LegacyJobResponse(BaseModel):
    id: UUID
    status: str             # pending | running | done | failed
    signed_url: str | None
    signed_url_expires_at: datetime | None
    created_at: datetime

class LegacyDownloadResponse(BaseModel):
    download_url: str
    expires_at: datetime
    ttl_days: int = 90
```

### 8.8 시설 대시보드 (3개) + 관리자 (2개)

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| GET | `/dashboard/users` | FACILITY_ADMIN+ | 어르신 목록 (이상징후 포함) |
| GET | `/dashboard/anomalies` | FACILITY_ADMIN+ | 이상징후 현황 |
| GET | `/dashboard/kpi` | FACILITY_ADMIN+ | KPI 지표 |
| GET | `/admin/users` | SYSTEM_ADMIN | 전체 사용자 관리 |
| GET | `/admin/audit-logs` | SYSTEM_ADMIN | 감사 로그 |

### 8.9 WebSocket 프로토콜

```
엔드포인트: wss://{domain}/ws/dialog/{session_id}?token={JWT}

클라이언트 → 서버 (utterance):
{
  "type": "utterance",
  "audio_base64": "...",       // base64 인코딩 PCM (16kHz, 16bit mono)
  "timestamp": "2026-05-24T10:00:00Z"
}

서버 → 클라이언트 (response_chunk):
{
  "type": "response_chunk",
  "text_chunk": "안녕하세요,",  // 스트리밍 텍스트
  "audio_chunk": "...",         // base64 TTS 오디오
  "is_final": false
}

서버 → 클라이언트 (session_event):
{
  "type": "session_event",
  "event": "crisis_detected",  // "session_started" | "session_ended" | "crisis_detected"
  "crisis_level": 2,
  "payload": {}
}
```

---

## §9. 비동기 태스크 설계 (Celery)

### 9.1 Celery 앱 설정

```python
# src/tasks/celery_app.py

celery_app = Celery(
    "daon",
    broker="redis://redis:6379/0",
    backend="redis://redis:6379/1",
)

celery_app.conf.update(
    task_serializer="json",
    result_serializer="json",
    task_acks_late=True,        # 워커 장애 시 재시도 보장
    task_reject_on_worker_lost=True,
    worker_prefetch_multiplier=1,
    task_routes={
        "tasks.dialog.*": {"queue": "dialog"},
        "tasks.memory.*": {"queue": "memory"},
        "tasks.notification.*": {"queue": "notifications"},
        "tasks.report.*": {"queue": "reports"},
        "tasks.export.*": {"queue": "export"},
        "tasks.maintenance.*": {"queue": "maintenance"},
    },
)

# Celery Beat 스케줄
celery_app.conf.beat_schedule = {
    "weekly-report": {
        "task": "tasks.report.build_weekly_report",
        "schedule": crontab(hour=8, minute=0, day_of_week="monday"),
    },
    "cleanup-signed-urls": {
        "task": "tasks.maintenance.cleanup_expired_signed_urls",
        "schedule": crontab(hour=2, minute=0),
    },
}
```

### 9.2 태스크 그래프

```
대화 턴 완료
    │
    ├─► process_autobiography_chunk (dialog 큐, < 2초)
    │       │
    │       └─► update_memory_graph (memory 큐, < 5초)
    │
세션 종료
    │
    └─► generate_topic_summary (memory 큐, < 10초)

이상징후 감지
    │
    └─► send_tier1_notification (notifications 큐, < 1초)

[Celery Beat: 월요일 08:00]
    └─► build_weekly_report (reports 큐, < 30초)

[Celery Beat: 매일 02:00]
    └─► cleanup_expired_signed_urls (maintenance 큐, < 1분)

유산 패키지 요청
    └─► build_legacy_package (export 큐, < 5분)
            ack_late=True, max_retries=3
```

### 9.3 핵심 태스크 구현

```python
# src/tasks/dialog_tasks.py

@celery_app.task(
    name="tasks.dialog.process_autobiography_chunk",
    bind=True,
    max_retries=3,
    default_retry_delay=30,
    acks_late=True,
)
def process_autobiography_chunk(self, turn_id: str) -> dict:
    """
    1. session_turns에서 발화 텍스트 로드
    2. LLM으로 자서전 정보 추출 (period, topic, content, confidence)
    3. AutobiographyService.save_chunk() 호출
    4. update_memory_graph.delay(chunk_id) 체이닝
    """
```

---

## §10. Flutter 앱 구조

### 10.1 화면 구조

```
lib/
├── screens/
│   ├── auth/
│   │   └── login_screen.dart           # 보호자 로그인
│   ├── home/
│   │   └── home_screen.dart            # 어르신 목록 + 이상징후 뱃지
│   ├── user/
│   │   ├── user_detail_screen.dart     # 어르신 상세 (자서전 + 세션 이력)
│   │   └── autobiography_screen.dart   # 자서전 타임라인 뷰
│   ├── notifications/
│   │   └── notification_screen.dart    # 알림 이력 + 설정
│   └── legacy/
│       └── legacy_screen.dart          # 유산 패키지 요청/다운로드
│
├── providers/                          # Riverpod (또는 Provider)
│   ├── auth_provider.dart
│   ├── users_provider.dart
│   ├── notification_provider.dart
│   └── legacy_provider.dart
│
├── services/                           # API 클라이언트
│   ├── api_client.dart                 # Dio + JWT 인터셉터
│   ├── auth_service.dart
│   ├── user_service.dart
│   ├── notification_service.dart
│   └── legacy_service.dart
│
└── models/                             # Freezed 데이터 모델
    ├── user.dart
    ├── notification.dart
    └── legacy_job.dart
```

### 10.2 API 클라이언트

```dart
// lib/services/api_client.dart

class ApiClient {
  final Dio _dio;

  ApiClient(String baseUrl) : _dio = Dio(BaseOptions(baseUrl: baseUrl)) {
    _dio.interceptors.add(JwtInterceptor());
    _dio.interceptors.add(RetryInterceptor(retries: 3));
  }

  // JWT 자동 갱신 인터셉터
  // 401 응답 → refresh token으로 재발급 → 원 요청 재시도
}
```

---

## §11. 로컬 개발 환경

### 11.1 docker-compose.yml

```yaml
version: "3.9"
services:
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: daon_dev
      POSTGRES_USER: daon
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init-age.sql:/docker-entrypoint-initdb.d/01-age.sql

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  api:
    build: .
    command: uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
    ports: ["8000:8000"]
    volumes: [".:/app"]
    env_file: .env
    depends_on: [db, redis]

  worker:
    build: .
    command: celery -A src.tasks.celery_app worker -Q dialog,memory,notifications,export -c 4 --loglevel=info
    env_file: .env
    depends_on: [db, redis]

  beat:
    build: .
    command: celery -A src.tasks.celery_app beat --loglevel=info
    env_file: .env
    depends_on: [redis]

  flower:
    image: mher/flower:2.0
    ports: ["5555:5555"]
    environment:
      CELERY_BROKER_URL: redis://redis:6379/0
    depends_on: [redis]

  app:
    image: cirrusci/flutter:stable
    command: flutter run -d web-server --web-port 3000 --web-hostname 0.0.0.0
    ports: ["3000:3000"]
    volumes: ["./app:/app"]
    working_dir: /app

volumes:
  pgdata:
```

### 11.2 .env.example

```bash
# DB
DB_PASSWORD=devpassword
DATABASE_URL=postgresql+asyncpg://daon:devpassword@db:5432/daon_dev

# Redis
REDIS_URL=redis://redis:6379

# JWT
JWT_SECRET_KEY=change-me-in-production
JWT_ALGORITHM=HS256

# EXAONE (로컬 vLLM)
EXAONE_BASE_URL=http://host.docker.internal:8001/v1
EXAONE_API_KEY=local-key

# HyperCLOVA X (폴백)
HYPERCLOVA_API_KEY=

# AWS S3 (유산 패키지)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=daon-legacy-dev
AWS_REGION=ap-northeast-2

# FCM (Firebase Push)
FCM_SERVER_KEY=

# 암호화
AES_KEY=32-bytes-hex-key-change-in-prod

# 환경
ENVIRONMENT=development
LOG_LEVEL=DEBUG
```

### 11.3 셋업 절차

```bash
git clone https://github.com/boheni07/daon.git
cd daon
cp .env.example .env            # API 키 입력
docker compose up -d
docker compose exec api alembic upgrade head
docker compose exec api python scripts/seed_dev.py
# → http://localhost:8000/docs  (Swagger UI, ~30분 이내)
# → http://localhost:5555       (Flower 태스크 모니터)
```

---

## §12. 테스트 전략

### 12.1 테스트 경계

| 레이어 | 테스트 유형 | 도구 | 커버리지 목표 |
|---|---|---|---|
| 비즈니스 로직 (SafetyFilter, CrisisDetector, TierManager) | Unit | pytest + pytest-asyncio | ≥ 80% |
| 서비스 + DB | Integration | pytest + TestContainers (PostgreSQL) | ≥ 70% |
| API 엔드포인트 | Integration | pytest + httpx (AsyncClient) | 핵심 30개 전부 |
| 대화 E2E | E2E | pytest + 시뮬레이터 (Omi mock) | 주요 시나리오 5개 |

### 12.2 핵심 Unit 테스트 케이스

```python
# tests/unit/test_safety_filter.py

def test_emotion_mismatch_guard_blocks_celebratory():
    """sadness ≥ 0.6 + 축하 응답 → 공감 응답으로 교체."""
    sf = SafetyFilter()
    emotion = EmotionScore(sadness=0.7, joy=0.1, anger=0.1, neutral=0.1)
    result = sf.check(emotion, "정말 축하드려요!")
    assert result.allowed is False
    assert result.override_response in EMPATHY_RESPONSES

def test_emotion_mismatch_guard_allows_empathetic():
    """sadness ≥ 0.6 + 공감 응답 → 허용."""
    sf = SafetyFilter()
    emotion = EmotionScore(sadness=0.7, joy=0.1, anger=0.1, neutral=0.1)
    result = sf.check(emotion, "많이 힘드셨겠어요.")
    assert result.allowed is True

def test_normal_emotion_allows_any_response():
    """sadness < 0.6 → 모든 응답 허용."""
    sf = SafetyFilter()
    emotion = EmotionScore(sadness=0.3, joy=0.5, anger=0.1, neutral=0.1)
    result = sf.check(emotion, "정말 축하드려요!")
    assert result.allowed is True
```

---

## §13. CI/CD 파이프라인 개요

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env: {POSTGRES_PASSWORD: test}
      redis:
        image: redis:7-alpine

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: "3.12"}
      - run: pip install poetry && poetry install
      - run: poetry run alembic upgrade head
      - run: poetry run pytest tests/ -v --cov=src --cov-report=xml
      - uses: codecov/codecov-action@v4

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ruff mypy
      - run: ruff check src/
      - run: mypy src/ --ignore-missing-imports
```

---

## §14. 성능 최적화

### 14.1 Redis 캐싱 전략

| 캐시 키 패턴 | TTL | 용도 |
|---|---|---|
| `rag:{user_id}:{query_hash}` | 300s | RAG 검색 결과 |
| `user:{user_id}:profile` | 600s | 사용자 프로필 |
| `coverage:{user_id}` | 60s | 자서전 커버리지 |
| `session:{session_id}:turns` | 세션 종료 시 삭제 | 최근 5턴 |

### 14.2 pgvector IVFFlat 튜닝

```sql
-- 초기 설정 (1K 청크 기준)
CREATE INDEX ON autobiography_chunks
USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- 검색 시 probe 수 설정
SET ivfflat.probes = 10;

-- 10K 청크 이상 → HNSW 전환 검토
CREATE INDEX ON autobiography_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

### 14.3 LLM 스트리밍

- **스트리밍 방식:** SSE (Server-Sent Events) — WebSocket response_chunk 타입
- **첫 토큰 지연 목표:** ≤ 1.5초 (SC-01, V-04)
- **EXAONE vLLM 설정:** `--max-model-len 4096 --tensor-parallel-size 2`

---

## §15. 보안 아키텍처

### 15.1 데이터 분류 및 암호화

| 데이터 | 분류 | 처리 |
|---|---|---|
| 생애사 텍스트 (`autobiography_chunks.content`) | 민감 (개인정보) | AES-256-GCM 암호화 저장 |
| 음성 원본 | 민감 | 엣지 처리 후 즉시 삭제 (L1에서 미전송) |
| 세션 발화 텍스트 (`session_turns.text`) | 일반 | 평문 (분석 목적, 90일 후 자동 삭제 검토) |
| FCM 토큰 | 민감 | 전송 암호화 (TLS) |
| JWT | 인증 | HS256, access=30분, refresh=30일 |

### 15.2 API 보안

- **Rate Limiting:** `slowapi` — `/auth/token` 10 req/min, 일반 API 100 req/min
- **CORS:** 허용 오리진 명시 (`ALLOWED_ORIGINS` 환경변수)
- **입력 검증:** Pydantic v2 (전 엔드포인트 자동 적용)
- **SQL Injection:** SQLAlchemy ORM + 파라미터 바인딩 (raw SQL 금지)
- **XSS:** API 전용 (HTML 렌더링 없음) — Flutter 앱에서 HTML 이스케이프

---

## §16. 모니터링

### 16.1 Prometheus 메트릭 (M1~M10)

| ID | 메트릭 이름 | 타입 | 설명 |
|---|---|---|---|
| M1 | `daon_dialog_latency_seconds` | Histogram | 대화 응답 지연 (목표: p95 ≤ 1.5초) |
| M2 | `daon_asr_accuracy` | Gauge | 방언 인식 정확도 (목표: ≥ 85%) |
| M3 | `daon_emotion_mismatch_total` | Counter | 감정 부조화 발생 횟수 (목표: 0) |
| M4 | `daon_crisis_level_total` | Counter | 위기 레벨별 감지 횟수 |
| M5 | `daon_notification_sent_total` | Counter | Tier별 알림 발송 수 |
| M6 | `daon_legacy_build_success_rate` | Gauge | 유산 패키지 생성 성공률 (목표: ≥ 99.9%) |
| M7 | `daon_autobiography_coverage` | Gauge | 사용자별 자서전 시기 커버리지 |
| M8 | `daon_session_duration_seconds` | Histogram | 평균 대화 세션 길이 |
| M9 | `daon_rag_cache_hit_rate` | Gauge | RAG 캐시 히트율 |
| M10 | `daon_celery_task_duration_seconds` | Histogram | Celery 태스크 처리 시간 |

### 16.2 알림 규칙 (Alertmanager)

```yaml
groups:
  - name: daon_critical
    rules:
      - alert: DialogLatencyHigh
        expr: histogram_quantile(0.95, daon_dialog_latency_seconds) > 1.5
        for: 5m
        labels: {severity: warning}

      - alert: EmotionMismatchDetected
        expr: increase(daon_emotion_mismatch_total[1h]) > 0
        for: 0m
        labels: {severity: critical}   # SC-10: 발생 즉시 알림

      - alert: LegacyBuildFailureRate
        expr: daon_legacy_build_success_rate < 0.999
        for: 10m
        labels: {severity: critical}   # SC-12
```

---

## §17. 성공 기준 SC 커버리지 매핑

| SC | Plan 기준 | 설계 섹션 | 달성 방법 |
|---|---|---|---|
| SC-01 응답 지연 ≤ 1.5초 | V-04 | §14.3 + §16.1 M1 | LLM 스트리밍 + p95 모니터링 |
| SC-09 방언 인식률 ≥ 85% | V-02 | §3.3 + §16.1 M2 | DialectNormalizer + Gauge 추적 |
| SC-10 감정 부조화 0% | V-07 | §3.2 + §16.1 M3 | SafetyFilter + Counter=0 알림 |
| SC-11 알림 만족도 ≥ 4.0/5점 | V-05 | §5.1 + §8.6 | 3-Tier 구조 + 설정 UI |
| SC-12 유산 패키지 ≥ 99.9% | V-06 | §5.2 + §16.1 M6 | acks_late=True + Gauge 알림 |

---

## §18. Implementation Guide (Session Guide)

### 18.1 Module Map

| 모듈 | 섹션 | 파일 수 | 예상 소요 |
|---|---|---|---|
| M-01 | §1~2 전체 구조 + 디렉터리 | 2 (구조 파일) | 1일 |
| M-02 | §3 L2 Dialog Core | 8 py 파일 | 2일 |
| M-03 | §4 L3 Memory Layer | 5 py 파일 | 2일 |
| M-04 | §5~6 L4 Operate + L5 Trust | 6 py 파일 | 2일 |
| M-05 | §7 DB ERD + Alembic | 10 SQL + 1 py | 1일 |
| M-06 | §8 API 명세 30개 | 8 py 파일 | 1일 |
| M-07 | §9~10 Celery + Flutter | 7 py + 구조 | 1일 |
| M-08 | §11~16 환경·테스트·CI·모니터링 | 설정 파일들 | 1일 |

### 18.2 구현 순서

```
Phase 1 (Week 1):
  M-05 DB 스키마 → alembic upgrade head 확인
  M-06 API 스키마 (Pydantic) → 의존성 없는 순수 모델

Phase 2 (Week 2):
  M-02 Dialog Core → unit test 병행
  M-03 Memory Layer → integration test (TestContainers)

Phase 3 (Week 3):
  M-04 Operate + Trust → API 라우터 연결
  M-07 Celery 태스크 → Flower로 모니터링

Phase 4 (Week 4):
  M-08 docker-compose + CI → E2E 검증
  전체 통합 테스트 → /pdca analyze daon-sw
```

---

*문서 버전: v1.0 | 작성일: 2026-05-24 | 다음 단계: `/pdca do daon-sw --scope M-05,M-06` (DB + API 스키마 구현 착수)*
