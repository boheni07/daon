# Plan: 다온(Daon) SW 아키텍처서 작성

> **Feature:** daon-sw
> **작성일:** 2026-05-24
> **Phase:** Plan
> **Status:** In Progress
> **참조 PRD:** `docs/00-pm/daon.prd.md` v1.3
> **참조 플랫폼 Plan:** `docs/01-plan/features/daon.plan.md`
> **참조 설계:** `docs/02-design/features/daon.design.md`

---

## Executive Summary

| 항목 | 내용 |
|---|---|
| **Feature** | 다온 플랫폼 SW 아키텍처서 — FastAPI·PostgreSQL·pgvector·EXAONE·Flutter 기반 전체 SW 구현 명세 |
| **입력** | daon.plan.md (ADR 1~11 확정) + daon.design.md (레이어 설계) + daon-workflows.md v1.1 |
| **출력** | `daon-sw.design.md` — 개발팀이 즉시 착수 가능한 완전한 SW 아키텍처 문서 |
| **MVP 목표** | Phase 1 (4개월): 자서전 엔진 + 음성 대화 + 보호자 앱 + 파일럿 10~20가구 |

### Value Delivered (4-Perspective)

| 관점 | 내용 |
|---|---|
| **Problem** | daon.design.md가 레이어·API·워크플로우 수준을 다루지만, 서비스 내부 구조(서비스 클래스·의존성 주입·ORM 매핑·Celery 태스크 그래프)는 미정 — 개발자가 독립 착수 불가 |
| **Solution** | L2 Dialog Core → L3 Memory → L4 Operate → L5 Trust 4개 레이어를 모듈·클래스·함수 수준으로 명세하고, DB 마이그레이션·CI 파이프라인·로컬 개발 환경까지 포함 |
| **Function & UX Effect** | 개발자 온보딩 시간 ½ 단축 / 모듈 병렬 개발 가능 / 테스트 경계 명확화로 품질 향상 |
| **Core Value** | "설계에서 코드까지 직선" — 아키텍처 결정이 클래스·파일 수준으로 추적 가능한 단일 진실 소스 |

---

## Context Anchor

| Anchor | 값 |
|---|---|
| **WHY** | ADR 1~11이 확정된 지금, 개발팀이 독립적으로 각 모듈을 착수하려면 클래스·인터페이스·DB 마이그레이션 수준의 명세가 필요 |
| **WHO** | NUBiz AX 백엔드 개발자 (Python/FastAPI) · 앱 개발자 (Flutter) · MLOps 엔지니어 |
| **RISK** | EXAONE 로컬 추론 GPU 요건 미확인 / pgvector IVFFlat 인덱스 튜닝 / 방언 fine-tune 데이터셋 라이선스 |
| **SUCCESS** | daon-sw.design.md 완성 후 개발자 3명이 각 레이어 독립 착수 가능 / 로컬 dev 환경 30분 내 셋업 |
| **SCOPE** | (IN) L2~L5 SW 전체, Docker 개발환경, DB 마이그레이션, CI 파이프라인 개요 / (OUT) L1 HW, 인프라 프로비저닝(AWS/K8s), 픽셀 레벨 UI |

---

## 1. 설계 문서 구성 계획 (daon-sw.design.md 목차)

### 1.1 전체 목차

| 섹션 | 내용 | 대응 ADR |
|---|---|---|
| §1 시스템 전체 구조 | 5-Layer 다이어그램 + 서비스 경계 | — |
| §2 프로젝트 디렉터리 구조 | `src/` 전체 트리 + 모듈 책임 | ADR-07 |
| §3 L2 Dialog Core 상세 설계 | 오케스트레이터 · 안전필터 · 방언정규화 클래스 명세 | ADR-01~02 |
| §4 L3 Memory Layer 상세 설계 | 자서전 엔진 · RAG · 메모리그래프 서비스 클래스 | ADR-03·09·10 |
| §5 L4 Operate 상세 설계 | REST API 라우터 · 알림 Tier · 유산 패키지 | ADR-06·11 |
| §6 L5 Trust 상세 설계 | RBAC · 암호화 · 감사로그 | — |
| §7 데이터베이스 설계 | 완전한 ERD · 마이그레이션 스크립트 (Alembic) | ADR-03·08·09 |
| §8 API 명세 | OpenAPI 스펙 (핵심 엔드포인트 30개 이상) | ADR-07 |
| §9 비동기 태스크 설계 | Celery 태스크 그래프 · 큐 구성 | — |
| §10 Flutter 앱 구조 | 화면 구조 · Provider 상태 관리 · API 클라이언트 | ADR-06 |
| §11 로컬 개발 환경 | docker-compose.yml · .env 구성 · 시드 데이터 | — |
| §12 테스트 전략 | Unit · Integration · E2E 경계 + 커버리지 목표 | — |
| §13 CI/CD 파이프라인 개요 | GitHub Actions 워크플로우 | — |
| §14 성능 최적화 | Redis 캐싱 전략 · pgvector 튜닝 · LLM 스트리밍 | ADR-05 |
| §15 보안 아키텍처 | JWT · AES-256 · 감사 로그 설계 | — |
| §16 모니터링 | Prometheus 메트릭 정의 (M1~M10) | — |

### 1.2 레이어별 핵심 설계 요소

#### L2 Dialog Core (다이얼로그 엔진)

| 모듈 | 핵심 클래스 | 주요 메서드 | 의존성 |
|---|---|---|---|
| 대화 오케스트레이터 | `DialogOrchestrator` | `process_utterance()`, `run_state_machine()` | SafetyFilter, MemoryService, LLMClient |
| 안전 필터 | `SafetyFilter` | `check()`, `apply_emotion_mismatch_guard()` | EmotionDetector |
| 방언 정규화기 | `DialectNormalizer` | `normalize()`, `detect_dialect()` | Whisper fine-tune 모델 |
| 감정 인식 | `EmotionDetector` | `classify()` | KoBERT 계열 분류 모델 |
| 위기 감지 | `CrisisDetector` | `detect_level()` | 키워드 규칙 + LLMClient |
| 페르소나 엔진 | `PersonaEngine` | `build_system_prompt()` | MemoryService, UserRepository |
| TTS 클라이언트 | `TTSClient` | `synthesize_stream()` | 외부 TTS API |
| LLM 클라이언트 | `LLMClient` | `generate_stream()` | EXAONE 로컬 / HyperCLOVA X 폴백 |

#### L3 Memory Layer

| 모듈 | 핵심 클래스 | 주요 메서드 | 의존성 |
|---|---|---|---|
| 자서전 서비스 | `AutobiographyService` | `save_chunk()`, `get_coverage()`, `get_next_question_target()` | ChunkRepository, EmbeddingService |
| RAG 서비스 | `RAGService` | `retrieve()`, `build_context()` | pgvector, RedisCache |
| 임베딩 서비스 | `EmbeddingService` | `embed()` | KoSimCSE-roberta |
| 메모리 그래프 | `MemoryGraphService` | `upsert_entity()`, `query_related()` | Apache AGE (Cypher) |
| 세션 서비스 | `SessionService` | `start()`, `end()`, `summarize_topics()` | SessionRepository |

#### L4 Operate

| 모듈 | 핵심 클래스 | 주요 메서드 |
|---|---|---|
| 알림 서비스 | `NotificationService` | `send_tier1()`, `send_tier2()`, `schedule_tier3_report()` |
| 알림 Tier 매니저 | `NotificationTierManager` | `classify_event()`, `route_to_tier()` |
| 유산 패키지 빌더 | `LegacyPackageBuilder` | `build()`, `upload_to_s3()`, `generate_signed_url()` |
| 주간 리포트 빌더 | `WeeklyReportBuilder` | `build()`, `send()` |
| FCM 클라이언트 | `FCMClient` | `send_push()` |

#### L5 Trust

| 모듈 | 책임 |
|---|---|
| RBAC 미들웨어 | JWT 검증 + 5-Level 권한 체크 (SYSTEM_ADMIN → DEVICE) |
| 암호화 서비스 | AES-256-GCM (생애사 텍스트), bcrypt (비밀번호) |
| 감사 로그 | 모든 데이터 접근·변경 이벤트 기록 (감사 추적) |
| 동의 관리 | GDPR·개인정보보호법 동의 버전 관리 |

---

## 2. 기술 스택 확정 (ADR 1~11 기반)

| 레이어 | 기술 | 버전 | 라이선스 |
|---|---|---|---|
| **백엔드 프레임워크** | FastAPI | 0.115+ | MIT |
| **언어** | Python | 3.12 | PSF |
| **ORM** | SQLAlchemy | 2.x (async) | MIT |
| **스키마 검증** | Pydantic | v2 | MIT |
| **메인 DB** | PostgreSQL | 16 | PostgreSQL |
| **벡터 확장** | pgvector | 0.7+ | MIT |
| **그래프 확장** | Apache AGE | 1.5+ | Apache 2.0 |
| **캐시** | Redis | 7 | BSD |
| **비동기 태스크** | Celery | 5.x | BSD |
| **메시지 브로커** | Redis (Celery broker) | 7 | BSD |
| **엣지 ASR** | Whisper distil-large-v3 | — | Apache 2.0 |
| **LLM (primary)** | EXAONE 3.5 | — | Apache 2.0 |
| **LLM (fallback)** | HyperCLOVA X | API | 상업 |
| **임베딩** | KoSimCSE-roberta | — | Apache 2.0 |
| **모바일 앱** | Flutter | 3.x | BSD |
| **컨테이너** | Docker + Docker Compose | — | Apache 2.0 |
| **CI** | GitHub Actions | — | — |
| **모니터링** | Prometheus + Grafana | — | Apache 2.0 |

---

## 3. DB 설계 범위

### 3.1 테이블 목록 (전체)

| 테이블 | 역할 | 핵심 컬럼 |
|---|---|---|
| `users` | 어르신 프로필 | id, name, birth_year, dementia_stage, persona, dialect |
| `autobiography_chunks` | 서사 조각 (벡터 포함) | user_id, period, topic, content, embedding(768), confidence |
| `sessions` | 대화 세션 | user_id, started_at, ended_at, turn_count, emotion_summary |
| `session_turns` | 발화·응답 쌍 | session_id, speaker, text, emotion_score, created_at |
| `notifications` | 알림 이력 | user_id, tier, title, body, read_at, created_at |
| `notification_settings` | 사용자 알림 설정 | user_id, tier1_push, tier2_inapp, tier3_weekly |
| `legacy_jobs` | 유산 패키지 생성 작업 | user_id, status, s3_key, signed_url, signed_url_expires_at |
| `consents` | 동의 이력 | user_id, consent_type, version, agreed_at |
| `audit_logs` | 감사 로그 | actor_id, action, resource_type, resource_id, ip, created_at |
| `guardians` | 보호자 관계 | user_id, guardian_id, role (PRIMARY/FAMILY), fcm_token |

### 3.2 마이그레이션 전략

- **도구:** Alembic (SQLAlchemy 연동)
- **네이밍:** `YYYY_MM_DD_HHMM_<description>.py`
- **초기화:** `alembic upgrade head` 한 번으로 전체 스키마 생성
- **시드:** `scripts/seed_dev.py` — 파일럿 테스트용 더미 어르신 3명 + 자서전 청크 50개

---

## 4. API 설계 범위

### 4.1 REST API 엔드포인트 목록 (30개)

| 그룹 | 엔드포인트 수 | 주요 |
|---|---|---|
| 인증 | 3 | POST /auth/token, POST /auth/refresh, POST /auth/logout |
| 사용자 | 5 | CRUD + 프로필 업데이트 |
| 자서전 | 6 | 청크 CRUD + Coverage 조회 + 타임라인 |
| 세션 | 4 | 시작·종료·목록·주제 요약 |
| 알림 | 4 | 이력 조회 + 설정 변경 + 주간리포트 |
| 유산 패키지 | 3 | 생성 요청·상태 조회·다운로드 URL |
| 시설 대시보드 | 3 | 어르신 목록·이상징후 현황·KPI |
| 관리자 | 2 | 사용자 관리·감사 로그 |

### 4.2 WebSocket 프로토콜 (오미↔클라우드)

- **경로:** `wss://{domain}/ws/dialog/{session_id}`
- **인증:** JWT 쿼리 파라미터
- **메시지 타입:** `utterance` (오미→서버) / `response_chunk` (서버→오미, 스트리밍) / `session_event` (both)

---

## 5. Celery 태스크 설계

| 태스크 | 큐 | 트리거 | 소요 예상 |
|---|---|---|---|
| `process_autobiography_chunk` | `dialog` | 대화 턴 종료 시 | < 2초 |
| `update_memory_graph` | `memory` | 자서전 청크 저장 후 | < 5초 |
| `generate_topic_summary` | `memory` | 세션 종료 시 | < 10초 |
| `send_tier1_notification` | `notifications` | 이상징후 감지 즉시 | < 1초 |
| `build_weekly_report` | `reports` | 매주 월요일 08:00 (Celery Beat) | < 30초 |
| `build_legacy_package` | `export` | 탈퇴·사망 이벤트 | < 5분 |
| `cleanup_expired_signed_urls` | `maintenance` | 매일 02:00 | < 1분 |

---

## 6. 로컬 개발 환경 계획

### 6.1 docker-compose.yml 서비스

```
services:
  db:       PostgreSQL 16 + pgvector + Apache AGE
  redis:    Redis 7 (캐시 + Celery 브로커)
  api:      FastAPI (uvicorn, hot-reload)
  worker:   Celery worker (dialog, memory, notifications, export)
  beat:     Celery Beat (스케줄러)
  flower:   Celery Flower (태스크 모니터링 UI)
  app:      Flutter Web (개발용)
```

### 6.2 셋업 절차 목표

```bash
git clone https://github.com/boheni07/daon.git
cp .env.example .env          # EXAONE 모델 경로, AWS 키 등
docker compose up -d
alembic upgrade head
python scripts/seed_dev.py
# → http://localhost:8000/docs (Swagger) 30분 내 접근 가능
```

---

## 7. 성공 기준 (daon-sw.design.md 기준)

| ID | 기준 | 측정 방법 |
|---|---|---|
| DS-01 | 개발자 3명이 각 레이어 독립 착수 가능 | 온보딩 리뷰 (30분 이내 로컬 실행) |
| DS-02 | 전체 30개 API 엔드포인트 명세 완료 | OpenAPI 스펙 자동 생성 확인 |
| DS-03 | 10개 테이블 ERD + Alembic 마이그레이션 완성 | `alembic upgrade head` 성공 |
| DS-04 | Celery 태스크 7개 명세 완료 | 태스크 클래스 + 큐 설정 확인 |
| DS-05 | docker-compose 단일 명령 환경 구축 | `docker compose up -d` + Swagger 접근 |
| DS-06 | daon.plan.md SC-01~12 전 항목이 설계에 매핑 | 설계문서 §16 SC 커버리지 표 |

---

## 8. 리스크

| 리스크 | 영향 | 대응 |
|---|---|---|
| EXAONE 3.5 로컬 추론 GPU 요건 (VRAM ≥24GB) | 개발 서버 선택 제약 | 개발 환경은 API 모드(HyperCLOVA X) 폴백으로 시작 |
| Apache AGE 안정성 (PostgreSQL 확장) | 그래프 쿼리 오류 | Phase 1에서 AGE 없이 JSONB 관계 표현 → Phase 2 전환 옵션 |
| pgvector IVFFlat 성능 (1K 청크+ 시) | 검색 지연 | lists=100 기본, 실측 후 HNSW 전환 검토 |
| Celery 워커 장애 시 자서전 유실 | 데이터 손실 | 발화 텍스트 먼저 DB 저장(동기) → 임베딩은 비동기 재시도(acks_late=True) |

---

## 9. 구현 세션 계획 (daon-sw.design.md 작성 순서)

| 세션 | 범위 | 예상 소요 |
|---|---|---|
| S-01 | §1~2 전체 구조 + 디렉터리 트리 | 1일 |
| S-02 | §3 L2 Dialog Core 상세 (오케스트레이터·안전필터·방언정규화) | 2일 |
| S-03 | §4 L3 Memory Layer (자서전 엔진·RAG·메모리그래프) | 2일 |
| S-04 | §5~6 L4 Operate + L5 Trust (알림 Tier·유산 패키지·RBAC) | 2일 |
| S-05 | §7 DB 완전 ERD + Alembic 마이그레이션 | 1일 |
| S-06 | §8 API 명세 30개 (OpenAPI YAML) | 1일 |
| S-07 | §9~10 Celery 태스크 + Flutter 앱 구조 | 1일 |
| S-08 | §11~16 개발환경·테스트·CI·모니터링·보안 | 1일 |
| **합계** | **전체 설계 문서** | **~11일** |

---

## 10. 다음 단계

- `/pdca design daon-sw` — 위 계획 기반으로 `daon-sw.design.md` 작성 시작
- `/pdca design daon-sw --scope S-01,S-02` — 첫 세션 (전체 구조 + Dialog Core)
