# Plan: 다온(Daon) 플랫폼 SW 아키텍처 설계

> **Feature:** daon-sw-architecture
> **작성일:** 2026-05-22
> **Phase:** Plan ✅
> **Status:** In Progress → Design 단계 착수 대기
> **참조 PRD:** `docs/00-pm/daon.prd.md` v1.3
> **참조 워크플로우:** `docs/00-pm/daon-workflows.md` v1.0

---

## Executive Summary

| 항목 | 내용 |
|---|---|
| **Feature** | 다온 플랫폼 SW 아키텍처 — L2 Dialog Core · L3 Memory Layer · L4 Operate · L5 Trust 전체 설계 |
| **전략** | 오픈소스 최대화 (Whisper · EXAONE-Open · pgvector · Flutter · FastAPI) |
| **MVP 목표** | Phase 1 (4개월): 자서전 엔진 + 음성 대화 + 보호자 앱 + 파일럿 10~20가구 |
| **성공 기준** | 응답 지연 ≤1.5초 / 일 30회+ 상호작용 / 자서전 6개월 핵심 시점 80% 채움 |

### Value Delivered (4-Perspective)

| 관점 | 내용 |
|---|---|
| **Problem** | PRD·워크플로우가 완성되었으나 SW 구현 기준 아키텍처·API·데이터 모델이 미정 — 개발 착수 불가 상태 |
| **Solution** | L2~L5 전체 레이어를 오픈소스 기반으로 설계하고 ADR·API 인터페이스·DB 스키마 개요를 확정 |
| **Function & UX Effect** | 개발팀이 모듈별 독립 구현 가능한 명확한 인터페이스 정의 → 병렬 개발 가속 |
| **Core Value** | 데이터 주권 보장(온프레미스 옵션)·비용 최적화(오픈소스)·국내 LLM 생태계 기여 |

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

## 1. 아키텍처 결정 기록 (ADR) — 오픈소스 최대화 확정

> v1.2 PRD §9.4 ADR 초안을 **오픈소스 최대화 전략**으로 최종 결정.

| ADR-ID | 결정 사항 | **확정 선택** | 근거 |
|---|---|---|---|
| **ADR-01** | 엣지 ASR | **Whisper distil-large-v3 한국어 fine-tune** | Apache 2.0 오픈소스, RTF ≤0.1 (Jetson Orin), AI Hub 한국어 노인 음성 데이터 fine-tune (표준어+방언 4종: 경상·전라·충청·제주) |
| **ADR-02** | 클라우드 LLM | **EXAONE 3.5 (LG AI Research, Apache 2.0)** | 한국어 최고 오픈소스 LLM, 상업 사용 가능, 자체 서버 배포로 데이터 주권 확보. 폴백: HyperCLOVA X API |
| **ADR-03** | 벡터스토어 | **pgvector (Phase 1) → Milvus (Phase 2, 1K+ 사용자)** | PostgreSQL 통합 단순성 → 스케일 시 Milvus 전환 |
| **ADR-04** | 오미↔클라우드 동기화 | **이벤트 기반 (대화 종료 시 업로드, MQTT/WebSocket)** | 배터리·LTE 비용 최적화, 오프라인 내성 |
| **ADR-05** | LLM 컨텍스트 캐싱 | **시스템 프롬프트 + 자서전 요약 캐시 (Redis TTL 1hr)** | 토큰 비용 60~70% 절감 추정 |
| **ADR-06** | 보호자 앱 | **Flutter 3.x (Dart)** | iOS+Android 단일 코드베이스, MIT, 국내 개발자 풀 |
| **ADR-07** | 백엔드 프레임워크 | **FastAPI (Python 3.12)** | async 네이티브, LLM·ML 생태계 통합, 자동 OpenAPI 스펙 생성 |
| **ADR-08** | 메인 DB | **PostgreSQL 16 + pgvector** | ACID·JSONB·벡터 하나로, 오픈소스 |
| **ADR-09** | 그래프 DB (메모리그래프) | **Apache AGE (PostgreSQL extension)** | PostgreSQL 위에서 Cypher 쿼리, 별도 DB 없이 통합 |
| **ADR-10** | 한국어 임베딩 | **KoSimCSE-roberta (한국어 STS SOTA, Apache 2.0)** | 768d, 자서전 RAG 검색 최적 |
| **ADR-11** | 보호자 알림 체계 | **3-Tier 알림 분류 (긴급Push / 인앱 / 주간리포트)** | 리빙랩 F-04 — 알림 피로도가 앱 이탈 원인. 단일 Push 방식은 보호자 이탈 가속. Tier 분리로 중요 알림 주목도 유지 |

---

## 2. 레이어별 SW 요구사항

### 2.1 L2 — Daon Dialog Core (대화 엔진)

**역할:** 어르신 음성 → 응답 TTS 전 과정 오케스트레이션

#### 2.1.1 하위 모듈

| 모듈 | 책임 | 핵심 기술 |
|---|---|---|
| **대화 오케스트레이터** | 세션 상태 관리, 모듈 호출 순서 제어 | FastAPI + asyncio 상태머신 |
| **페르소나 엔진** | 호칭·말투·공유 추억 개인화 | EXAONE 시스템 프롬프트 + 자서전 RAG |
| **안전 필터 (가드레일)** | 의료 단정·인간 사칭·환각·위기 발화·**감정 부조화** 차단 | 규칙 기반 + LLM 분류기 |
| **방언 정규화기** | 경상·전라·충청·제주 방언 → 표준어 전처리 | Whisper fine-tune + 방언 정규화 사전 |
| **자서전 회상 루프** | 빈칸 우선 질문 선택, 응답 처리 | 질문 우선순위 스코어링 엔진 |
| **질문 생성기** | 시점×주제 매트릭스 기반 질문 생성 | EXAONE few-shot + 템플릿 |
| **감정 인식 모듈** | 발화에서 감정 신호 추출, Level 판별 | 감정 분류 모델 (KoBERT 계열) |
| **위기 발화 감지** | 로컬(엣지) + 클라우드 2단계 감지 | 키워드 규칙(엣지) + LLM 분류(클라우드) |

#### 2.1.2 대화 오케스트레이터 상태머신

```
States:
  IDLE → LISTENING → PROCESSING → RESPONDING → IDLE

  IDLE:       Wake Word 대기
  LISTENING:  어르신 발화 수집 (최대 30초, 5초 침묵으로 종료)
  PROCESSING: ASR → 안전필터 → RAG → LLM → 감정인식 → 자서전저장
  RESPONDING: TTS 스트리밍 재생

Transitions:
  IDLE → LISTENING:       Wake Word 감지 또는 오미 선제 시작 타이머
  LISTENING → PROCESSING: 침묵 5초 or 최대 30초
  PROCESSING → RESPONDING: LLM 첫 토큰 수신 (스트리밍 시작)
  RESPONDING → IDLE:      TTS 재생 완료
  ANY → IDLE:             오류, 네트워크 실패 (엣지 폴백 모드)
```

#### 2.1.3 안전 필터 규칙 (필수 구현)

```
금지 패턴 (regex 기반 즉각 차단):
  - 의료 단정: "치매 진단", "치료됩니다", "약을 끊어도"
  - 인간 사칭: "저는 사람이에요", "실제 [이름]이에요"
  - 외부 정보 허위: 현재 날짜/사실 단정 (불확실 시 폴백)

감정 부조화 방지 (리빙랩 F-07):
  슬픔/슬픔 신호 감지 시 → 축하·밝은 감탄 응답 차단
  EMPATHY_RESPONSES 풀로 교체:
    - "많이 힘드셨겠다, 그 마음이 느껴져요."
    - "그때 참 속상하셨겠어요. 더 이야기해 주실 수 있어요?"
    - "어르신, 제가 여기 있을게요. 천천히 말씀해 주세요."
  감지 기준: EmotionClassifier 'sadness' ≥ 0.6 또는 위기 키워드 포함 발화

폴백 응답:
  환각 감지: "잘 모르겠어요. 가족분께 여쭤보시겠어요?"
  위기 L3:   "어르신, 많이 힘드시구나. 지금 바로 도움을 드릴게요."
```

#### 2.1.4 LLM 프롬프트 구조

```
[System Prompt — Redis 캐시, TTL 1hr]
  역할: 페르소나 정의 + 행동 원칙 + 안전 규칙
  내용: 호칭, 말투 스타일, 금지 표현, 위기 대응 지침

[자서전 컨텍스트 — RAG 인출, Redis 캐시]
  - 어르신 핵심 생애사 청크 top-5 (코사인 유사도)
  - 최근 3세션 대화 요약 (슬라이딩 윈도우)
  - 현재 질문 대상 시점×주제 정보

[현재 발화 — 비캐시]
  - 어르신 입력 텍스트 (STT 결과)
  - 감정 신호 메타데이터
  - 대화 턴 번호
```

---

### 2.2 L3 — Daon Memory Layer (기억·지식 레이어)

**역할:** 자서전 누적·구조화·검색

#### 2.2.1 자서전 타임라인 DB 스키마 (개요)

```sql
-- 어르신 프로필
CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL,
  birth_year  INT,
  dementia_stage TEXT CHECK (stage IN ('MCI','early','moderate','severe')),
  persona     TEXT,           -- 선택 페르소나 (친구/자매/형제)
  dialect     TEXT DEFAULT 'standard',
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 자서전 조각 (서사 단위)
CREATE TABLE autobiography_chunks (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID REFERENCES users(id) ON DELETE CASCADE,
  period       TEXT NOT NULL,  -- 유년기/학창/청년/중년/현재
  topic        TEXT NOT NULL,  -- 인물/장소/사건/감정
  content      TEXT NOT NULL,  -- 구술 서사 텍스트
  confidence   FLOAT DEFAULT 0.8,
  emotion_score FLOAT DEFAULT 0.0,
  is_private   BOOLEAN DEFAULT FALSE,  -- 어르신 비공개 설정
  source_session_id UUID,
  embedding    vector(768),    -- KoSimCSE 임베딩
  created_at   TIMESTAMPTZ DEFAULT NOW(),
  updated_at   TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX ON autobiography_chunks USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
CREATE INDEX ON autobiography_chunks (user_id, period, topic);

-- 자서전 매트릭스 커버리지 뷰
CREATE VIEW autobiography_coverage AS
SELECT
  user_id,
  period,
  topic,
  COUNT(*) AS chunk_count,
  AVG(confidence) AS avg_confidence,
  MAX(updated_at) AS last_updated
FROM autobiography_chunks
GROUP BY user_id, period, topic;

-- 메모리그래프 노드 (Apache AGE 연동)
CREATE TABLE memory_entities (
  id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id  UUID REFERENCES users(id),
  type     TEXT,   -- Person/Place/Event/Emotion
  name     TEXT,
  metadata JSONB,
  weight   FLOAT DEFAULT 1.0  -- 감정 강도 × 언급 빈도
);

-- 세션 로그
CREATE TABLE conversation_sessions (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID REFERENCES users(id),
  started_at  TIMESTAMPTZ,
  ended_at    TIMESTAMPTZ,
  turn_count  INT DEFAULT 0,
  emotion_summary JSONB,  -- {joy: 0.3, sadness: 0.1, ...}
  anomaly_level INT DEFAULT 0  -- 0=정상, 1~3=이상징후 레벨
);

-- 동의 Lineage
CREATE TABLE consent_records (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID REFERENCES users(id),
  data_category   TEXT,  -- autobiography/voice/health_signal
  purpose         TEXT,  -- family_sharing/research/service
  consented_by    TEXT,  -- self/guardian/facility
  consented_at    TIMESTAMPTZ,
  revoked_at      TIMESTAMPTZ
);

-- 복약 기록
CREATE TABLE medication_logs (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID REFERENCES users(id),
  scheduled_at TIMESTAMPTZ,
  confirmed_at TIMESTAMPTZ,
  status      TEXT  -- confirmed/missed/refused
);
```

#### 2.2.2 빈칸 탐지 & 질문 우선순위 엔진

```python
# 질문 우선순위 계산 (lib/memory/question_priority.py)

PERIODS = ["유년기", "학창시절", "청년기", "중년기", "현재"]
TOPICS  = ["인물", "장소", "사건", "감정"]

def compute_priority(user_id: str, db: Session) -> list[QuestionCandidate]:
    """
    각 (period, topic) 셀에 대해 우선순위 점수 계산
    Priority = 0.5*(1-Coverage) + 0.3*Recency + 0.2*EmotionScore
    """
    coverage_map = get_coverage_map(user_id, db)
    candidates = []
    for period in PERIODS:
        for topic in TOPICS:
            cell = coverage_map.get((period, topic), CellStats())
            coverage  = min(cell.chunk_count / 5, 1.0)  # 5개 청크 = 충분
            recency   = days_since(cell.last_updated) / 30  # 30일 정규화
            emotion   = cell.avg_emotion_score
            priority  = 0.5*(1-coverage) + 0.3*recency + 0.2*emotion
            candidates.append(QuestionCandidate(period, topic, priority))
    return sorted(candidates, key=lambda x: -x.priority)
```

#### 2.2.3 RAG 검색 파이프라인

```
[쿼리: 현재 대화 맥락 텍스트]
      │
      ▼
[KoSimCSE 임베딩 (768d)]
      │
      ▼
[pgvector 코사인 유사도 검색]
  SELECT content, 1-(embedding <=> $query_vec) AS score
  FROM autobiography_chunks
  WHERE user_id = $uid AND is_private = FALSE
  ORDER BY score DESC LIMIT 5
      │
      ▼
[메모리그래프 연관 엔티티 확장]
  Cypher(AGE): MATCH (e:Entity)-[r]-(related)
               WHERE e.name IN $extracted_entities
               RETURN related ORDER BY r.weight DESC LIMIT 10
      │
      ▼
[컨텍스트 조립 → LLM 프롬프트 삽입]
```

---

### 2.3 L4 — Daon Operate (운영·연계 레이어)

**역할:** 외부 접점 전체 — TTS·보호자 앱·대시보드·알림·자서전 생성기

#### 2.3.1 API 설계 (REST + WebSocket)

```
Base URL: https://api.daon.ai/v1

[인증]
  POST   /auth/login          # 보호자/관리자 로그인 (JWT)
  POST   /auth/refresh        # 토큰 갱신
  POST   /auth/device         # 오미 디바이스 토큰 발급

[어르신 관리]
  POST   /users               # 어르신 등록 (온보딩)
  GET    /users/{id}          # 프로필 조회
  PATCH  /users/{id}          # 설정 변경 (페르소나·방언·치매단계)
  DELETE /users/{id}          # 탈퇴 + 데이터 삭제

[자서전]
  GET    /users/{id}/autobiography          # 전체 자서전 (타임라인)
  GET    /users/{id}/autobiography/coverage # 시점×주제 커버리지 매트릭스
  POST   /users/{id}/autobiography/chunks  # 청크 수동 추가 (가족 보완)
  PATCH  /users/{id}/autobiography/chunks/{chunk_id}  # 수정
  DELETE /users/{id}/autobiography/chunks/{chunk_id}  # 삭제

[대화 세션]
  WS     /ws/session/{user_id}  # 실시간 대화 (오미 디바이스 연결)
  GET    /users/{id}/sessions   # 세션 이력
  GET    /users/{id}/sessions/{session_id}  # 세션 상세

[알림·이상징후]
  GET    /users/{id}/anomalies  # 이상징후 이력
  POST   /users/{id}/anomalies/{id}/resolve  # 대응 완료 처리
  GET    /users/{id}/medication-logs  # 복약 기록

[가족 관리]
  POST   /users/{id}/family     # 가족 초대
  GET    /users/{id}/family     # 가족 목록
  DELETE /users/{id}/family/{family_id}  # 제거
  POST   /users/{id}/voice-messages      # 음성메시지 업로드
  GET    /users/{id}/voice-messages      # 목록

[자서전 산출물]
  POST   /users/{id}/autobiography/export  # 책자/오디오북 생성 요청
  GET    /users/{id}/autobiography/export/{job_id}  # 생성 상태
  GET    /users/{id}/autobiography/export/{job_id}/download  # 다운로드

[시설/관리자]
  GET    /facilities/{fid}/users  # 시설 어르신 목록
  GET    /facilities/{fid}/dashboard  # 대시보드 집계
  GET    /facilities/{fid}/reports    # 분기 보고서

[동의 관리]
  GET    /users/{id}/consents   # 동의 목록
  POST   /users/{id}/consents   # 동의 추가
  DELETE /users/{id}/consents/{consent_id}  # 동의 철회

[알림 Tier 관리 — 리빙랩 ADR-11]
  GET    /users/{id}/notifications           # Tier 1·2 알림 이력
  GET    /users/{id}/weekly-report           # Tier 3 주간 요약 리포트
  PATCH  /users/{id}/notification-settings  # Tier별 수신 설정

[세션 주제 요약 — 리빙랩 F-01]
  GET    /users/{id}/sessions/{session_id}/topic-summary  # 회상 주제 자동 요약

[유가족 유산 패키지 — 리빙랩 F-08, PRD §16]
  POST   /users/{id}/legacy-package         # 유산 패키지 생성 요청 (탈퇴·사망 트리거)
  GET    /users/{id}/legacy-package/{job_id}  # 생성 상태
  GET    /users/{id}/legacy-package/{job_id}/download  # 서명된 URL (TTL 90일)
```

#### 2.3.2 WebSocket 대화 프로토콜 (오미↔클라우드)

```json
// 오미 → 서버 (발화 완료)
{
  "type": "utterance",
  "session_id": "sess_xxx",
  "user_id": "user_xxx",
  "text": "경상도 밀양에 살았어요",
  "audio_duration_ms": 2300,
  "edge_emotion_hint": "neutral",
  "timestamp": "2026-05-22T09:00:00Z"
}

// 서버 → 오미 (응답 스트리밍)
{
  "type": "response_chunk",
  "text": "와, 낙동강 옆이군요!",
  "is_final": false,
  "led_code": "orange_blink"
}
{
  "type": "response_chunk",
  "text": " 강에서 자주 놀았겠네요.",
  "is_final": true,
  "led_code": "green"
}

// 서버 → 오미 (이상징후 감지)
{
  "type": "anomaly_alert",
  "level": 2,
  "action": "notify_guardian",
  "guardian_ids": ["guardian_xxx"]
}

// 오미 → 서버 (세션 종료, 이벤트 기반 동기화)
{
  "type": "session_end",
  "session_id": "sess_xxx",
  "turn_count": 23,
  "duration_sec": 840,
  "local_chunks": [...]  // 로컬 캐시 청크 업로드
}
```

#### 2.3.3 보호자 앱 주요 화면 (Flutter)

| 화면 | 라우트 | 핵심 기능 |
|---|---|---|
| 홈·대시보드 | `/` | 오늘 요약, 이상징후, 최근 대화 |
| 자서전 타임라인 | `/autobiography` | 시기별 스크롤, 사진 갤러리 |
| 자서전 상세 | `/autobiography/{period}` | 청크 목록, 추가·수정 |
| 음성메시지 | `/voice` | 녹음·전송·수신 확인 |
| 알림 이력 | `/alerts` | Level별 이력, 대응 완료 처리 |
| 설정 | `/settings` | 페르소나·복약·가족 관리 |
| 자서전 주문 | `/export` | 책자·오디오북 주문 |

#### 2.3.4 자서전 생성기 파이프라인

```
[트리거: 주문 요청]
      │
      ▼
[자서전 청크 수집]
  - user_id 기준 전체 청크 (is_private=FALSE)
  - 시점별 정렬, 감정 강도 상위 청크 우선
      │
      ▼
[LLM 서사 변환]
  - 청크 → 1인칭 구술 문단 (EXAONE)
  - "내가 열 살 때 낙동강 옆 마을에서..."
  - 문단당 200~400자
      │
      ▼
[PDF 레이아웃 생성]
  - WeasyPrint / ReportLab
  - A5 기준, 사진 자동 배치, 표지 생성
      │
      ▼
[오디오북 생성]
  - TTS (오미 목소리) 문단별 MP3 생성
  - 어르신 실제 음성 클립 삽입 (선택)
  - 챕터별 mp3 → m4b(오디오북) 패키징
      │
      ▼
[S3 업로드 → 다운로드 링크 발급]
  - 서명된 URL (TTL 7일)
  - 보호자 앱 알림 발송
```

---

### 2.4 L5 — Daon Trust (거버넌스·보안)

#### 2.4.1 암호화 전략

| 대상 | 방식 | 비고 |
|---|---|---|
| 전송 암호화 | TLS 1.3 필수 | 오미↔클라우드, 앱↔API 전체 |
| 저장 암호화 | PostgreSQL TDE + AES-256 | 자서전·음성 파일·개인정보 |
| 음성 파일 | S3 SSE-S3 or KMS | at-rest 암호화 |
| 엣지↔클라우드 | mTLS (디바이스 인증서) | 오미 디바이스 인증 |

#### 2.4.2 RBAC 권한 모델

```
Role 계층:
  SYSTEM_ADMIN     > FACILITY_ADMIN > GUARDIAN_PRIMARY > GUARDIAN_FAMILY > DEVICE

  SYSTEM_ADMIN:      전체 관리, 모델 배포, 보안 감사
  FACILITY_ADMIN:    시설 내 어르신 전체, 보고서, 이상징후
  GUARDIAN_PRIMARY:  특정 어르신 전체 + 설정 변경 + 동의 관리
  GUARDIAN_FAMILY:   자서전 열람(비공개 제외) + 음성메시지
  DEVICE(Omi):       대화 세션 + 자서전 읽기/쓰기 (자기 user_id만)

권한 행렬:
  autobiography.read:    SYSTEM_ADMIN, FACILITY_ADMIN*, GUARDIAN_PRIMARY, GUARDIAN_FAMILY
  autobiography.write:   DEVICE, GUARDIAN_PRIMARY
  autobiography.private: DEVICE, GUARDIAN_PRIMARY (본인만)
  anomaly.alert:         DEVICE (생성) / GUARDIAN_PRIMARY, FACILITY_ADMIN (수신)
  consent.manage:        GUARDIAN_PRIMARY, SYSTEM_ADMIN
  system.model_deploy:   SYSTEM_ADMIN
  
  * FACILITY_ADMIN은 시설 소속 어르신 한정
```

#### 2.4.3 동의 Lineage 구현

```python
# 모든 자서전 청크에 consent_id 연결
# 동의 철회 시 cascade 처리

class ConsentService:
    def revoke_consent(self, user_id: str, consent_id: str):
        consent = self.db.get(ConsentRecord, consent_id)
        consent.revoked_at = datetime.utcnow()
        
        # 해당 동의 기반 데이터 처리
        if consent.data_category == "autobiography":
            self.db.query(AutobiographyChunk)\
                .filter_by(user_id=user_id)\
                .filter(AutobiographyChunk.consent_id == consent_id)\
                .update({"is_deleted": True})
        
        self.db.commit()
        # 감사 로그 기록
        self.audit_log.record(action="consent_revoked", ...)
```

#### 2.4.4 MLOps 파이프라인

```
[LLM 품질 모니터링]
  - 일 샘플링: 무작위 100건 응답 품질 검수
  - 안전 필터 우회 시도 감지 (자동 플래그)
  - 환각 지표: "잘 모르겠어요" 폴백 비율 추적

[STT 성능 모니터링]
  - 인식률 (WER) 주간 집계
  - 노인 발화 특화 오류 패턴 분류
  - fine-tuning 데이터 자동 수집 파이프라인

[모델 배포 절차]
  1. 스테이징 환경 배포
  2. 안전 회귀 테스트 (위기 발화 탐지 ≥90%, 금지표현 차단 100%)
  3. A/B 테스트 (응답 품질 사람 평가 5점 척도)
  4. 카나리 배포 (5% → 20% → 100%)
  5. 롤백 기준: 응답 P95 >2초 or 안전 실패 임계 초과
```

---

## 3. 서비스 컴포넌트 구성

### 3.1 마이크로서비스 구조

```
daon-platform/
├── services/
│   ├── dialog-core/          # L2: FastAPI, 대화 오케스트레이터
│   │   ├── orchestrator.py
│   │   ├── persona_engine.py
│   │   ├── safety_filter.py
│   │   ├── question_generator.py
│   │   └── emotion_detector.py
│   │
│   ├── memory-layer/         # L3: FastAPI, 자서전 엔진
│   │   ├── autobiography_service.py
│   │   ├── question_priority.py
│   │   ├── rag_retriever.py
│   │   └── memory_graph.py
│   │
│   ├── operate/              # L4: FastAPI, 외부 API
│   │   ├── api/
│   │   │   ├── users.py
│   │   │   ├── autobiography.py
│   │   │   ├── sessions.py
│   │   │   ├── alerts.py
│   │   │   └── family.py
│   │   ├── websocket/
│   │   │   └── session_handler.py
│   │   ├── notifications/
│   │   │   └── push_service.py
│   │   └── export/
│   │       └── autobiography_generator.py
│   │
│   ├── trust/                # L5: 인증·보안·MLOps
│   │   ├── auth/
│   │   │   ├── jwt_service.py
│   │   │   └── rbac.py
│   │   ├── consent/
│   │   │   └── consent_service.py
│   │   └── mlops/
│   │       └── model_monitor.py
│   │
│   └── omi-bridge/           # 오미 디바이스 연동 게이트웨이
│       ├── device_auth.py
│       └── sync_handler.py
│
├── shared/                   # 공통 라이브러리
│   ├── db/                   # PostgreSQL 연결·마이그레이션
│   ├── models/               # SQLAlchemy ORM 모델
│   ├── schemas/              # Pydantic v2 스키마
│   └── utils/
│
├── workers/                  # 백그라운드 작업 (Celery)
│   ├── autobiography_processor.py  # 자서전 청크 처리
│   ├── export_worker.py            # 책자·오디오북 생성
│   └── alert_worker.py             # 이상징후 알림 발송
│
└── mobile/                   # Flutter 보호자 앱
    └── lib/
        ├── screens/
        ├── widgets/
        └── services/
```

### 3.2 인프라 구성 (Phase 1 MVP)

```
인터넷
  │
  ├─ [CloudFlare CDN] ─ 정적 자산, 앱 배포
  │
  ▼
[Nginx Reverse Proxy]
  ├─ api.daon.ai → FastAPI 클러스터
  ├─ ws.daon.ai  → WebSocket 서버 (오미 연결)
  └─ cdn.daon.ai → S3 (음성파일·자서전 PDF)
  │
  ├─ [FastAPI 서비스 클러스터] (Docker Compose or Kubernetes)
  │   ├─ dialog-core ×2
  │   ├─ memory-layer ×2
  │   ├─ operate ×2
  │   └─ trust ×1
  │
  ├─ [PostgreSQL 16 + pgvector] (Primary + Replica)
  ├─ [Redis 7] (세션·캐시·Celery 브로커)
  ├─ [Celery Workers] ×2 (export·alert)
  └─ [EXAONE 3.5 추론 서버] (GPU 서버, Phase 1: 임대)
```

---

## 4. 성공 기준 (Success Criteria)

| ID | 기준 | 측정 방법 | 목표값 |
|---|---|---|---|
| SC-01 | 음성 응답 지연 | P95 latency (Wake Word → 첫 TTS 음절) | ≤ 1,500ms |
| SC-02 | 자서전 수집 정확도 | 시점·주제 분류 정확도 (사람 평가 100건) | ≥ 85% |
| SC-03 | 위기 발화 감지 | Level 2·3 발화 감지 정밀도 | ≥ 90% |
| SC-04 | 안전 필터 | 금지 표현 차단율 | 100% |
| SC-05 | LLM 환각 | "잘 모르겠어요" 적절 폴백 비율 (사람 평가) | ≥ 95% |
| SC-06 | 가용성 | 서비스 업타임 | ≥ 99.5% |
| SC-07 | 데이터 정합성 | 동의 철회 시 데이터 처리 완료 | ≤ 24시간 |
| SC-08 | 자서전 Coverage | 6개월 파일럿 후 핵심 시점 채움률 | ≥ 80% |
| SC-09 | 방언 인식률 | 경상·전라·충청·제주 방언 발화 WER (리빙랩 F-05) | ≤ 15% (≥ 85% 인식) |
| SC-10 | 감정 부조화 차단 | 슬픔 신호 발화 후 감탄 응답 발생 건수 | 0건 (100% 차단) |
| SC-11 | 보호자 알림 만족도 | Tier 1 Push 적절성 평가 (1~5점, 파일럿 보호자) | ≥ 4.0점 |
| SC-12 | 유산 패키지 신뢰성 | 생성 성공률 + 90일 다운로드 링크 유효성 | ≥ 99.9% |

---

## 5. 리스크 & 대응

| 리스크 | 영향 | 가능성 | 대응 |
|---|---|---|---|
| EXAONE 오픈소스 한국어 품질 미달 | 대화 품질 저하 | 중 | HyperCLOVA X API 즉시 폴백 가능하도록 추상화 레이어 유지 |
| 응답 지연 1.5초 초과 | 어르신 이탈 | 중 | 엣지 ASR 최적화 + LLM 스트리밍 + TTS 프리버퍼링 병행 |
| pgvector 스케일 한계 | 검색 품질 저하 | 낮음 (Phase 1) | 1K 사용자 도달 시 Milvus 전환 (ADR-03) |
| 오미 디바이스 연결 불안정 | UX 저하 | 중 | 엣지 폴백 모드 (로컬 복약·간단 대화) 필수 구현 |
| LLM 환각으로 위험 정보 | 신뢰·안전 손상 | 낮음 | 안전 필터 100% 차단 + 사람 검수 루프 |
| 생애사 데이터 유출 | 법적·평판 치명 | 낮음 | TLS 1.3 + TDE + mTLS + RBAC + ISMS-P 준비 |

---

## 6. 구현 로드맵

### Phase 0 — 기반 구축 (2026.06, ~1개월)
- [ ] 개발 환경 세팅 (Docker Compose, PostgreSQL+pgvector, Redis)
- [ ] 공통 라이브러리 (DB 연결, ORM 모델, Pydantic 스키마, JWT)
- [ ] Whisper distil 한국어 fine-tune 실험 (AI Hub 노인 음성 데이터)
- [ ] EXAONE 3.5 로컬 배포 & 기본 대화 품질 평가

### Phase 1 — MVP 핵심 (2026.07~08, ~2개월)
- [ ] L3 Memory Layer: 자서전 DB + 빈칸 탐지 엔진 + RAG 파이프라인
- [ ] L2 Dialog Core: 오케스트레이터 + 페르소나 엔진 + 안전 필터
- [ ] L4 Operate: WebSocket 대화 서버 + 기본 REST API
- [ ] L5 Trust: JWT + RBAC + 동의 Lineage
- [ ] 오미 디바이스 연동 (엣지 폴백 포함)

### Phase 2 — 앱 & 운영 (2026.09~10, ~2개월)
- [ ] Flutter 보호자 앱 (자서전 열람·음성메시지·알림)
- [ ] 이상징후 3단계 알림 시스템
- [ ] 자서전 책자·오디오북 생성기
- [ ] 시설 관리자 대시보드
- [ ] MLOps 모니터링 파이프라인

### Phase 3 — 공공 실증 (2026.11~, 파일럿)
- [ ] 파일럿 10~20가구 배포
- [ ] KPI 측정 (SC-01~SC-08)
- [ ] 보건소 실증 준비 (ISMS-P 1단계)

---

## 7. 다음 단계

- **즉시:** `/pdca design daon` — 이 Plan 기반 Design 문서 작성 (컴포넌트 상세·시퀀스 다이어그램)
- **병행:** `/pdca plan daon-hw` — 오미 HW 아키텍처서 착수
- **병행:** `/pdca plan daon-ux` — UX 가이드라인 상세화
