# Plan: 오미(Omi) HW 아키텍처서 작성

> **Feature:** daon-hw
> **작성일:** 2026-05-24
> **Phase:** Plan
> **Status:** In Progress
> **참조 PRD:** `docs/00-pm/daon.prd.md` v1.3 §8·§9.1~9.2
> **참조 플랫폼 Plan:** `docs/01-plan/features/daon.plan.md`
> **참조 SW 설계:** `docs/02-design/features/daon-sw.design.md`

---

## Executive Summary

| 항목 | 내용 |
|---|---|
| **Feature** | 오미(Omi) 반려로봇 HW 아키텍처서 — 폼팩터·음성 회로·SBC·임베디드 OS·펌웨어·BOM 전체 명세 |
| **입력** | PRD §9.1 HW 스펙 요구사항 + §9.2 응답 지연 1.5초 파이프라인 + ADR-04 이벤트 기반 동기화 |
| **출력** | `daon-hw.design.md` — HW 설계자·임베디드 엔지니어가 즉시 착수 가능한 HW 아키텍처 문서 |
| **MVP 목표** | Phase 1 프로토타입: BOM ≤ $150, 로컬 개발 환경 셋업 ≤ 1일, 파일럿 10~20가구 |

### Value Delivered (4-Perspective)

| 관점 | 내용 |
|---|---|
| **Problem** | PRD §9.1이 요구사항을 정의했지만, SBC 선정·회로 설계·NervesOS 펌웨어 구조·BOM 상세가 미정 — HW 설계자가 독립 착수 불가 |
| **Solution** | 폼팩터→컴퓨팅→음성→전원→통신→임베디드 OS→펌웨어→Wake Word→엣지 ASR→클라우드 통신까지 레이어별 상세 명세 + BOM 원가 분석 |
| **Function & UX Effect** | HW 설계자·임베디드 엔지니어·제조 파트너가 단일 문서로 프로토타입 착수 / 응답 지연 1.5초 달성 경로 확인 |
| **Core Value** | "하드웨어에서 소프트웨어까지 한 문서" — L1 Omi 디바이스가 L2~L5 다온 플랫폼과 어떻게 연결되는지를 계층별로 추적 가능 |

---

## Context Anchor

| Anchor | 값 |
|---|---|
| **WHY** | PRD §9.1 스펙 요구사항이 확정된 지금, HW 설계자가 독립적으로 프로토타입 착수하려면 SBC 선정·회로·BOM·펌웨어 수준의 명세가 필요 |
| **WHO** | NUBiz AX HW 설계자 · 임베디드 엔지니어 (Elixir/NervesOS) · ODM/제조 파트너 |
| **RISK** | Jetson Orin Nano 수급·단가 변동 / 방언 ASR fine-tune 데이터셋 라이선스 / KC 인증 리드타임 (6~12주) / NervesOS Jetson 보드 패키지 성숙도 |
| **SUCCESS** | daon-hw.design.md 완성 후 HW 설계자·임베디드 엔지니어가 각자 독립 착수 가능 / 로컬 NervesOS 개발 환경 ≤ 1일 셋업 |
| **SCOPE** | (IN) L1 Omi 디바이스 HW + 임베디드 OS/펌웨어, BOM, 인증 요건 / (OUT) L2~L5 클라우드 SW (daon-sw 담당), 양산 금형·제조 공정, 클라우드 인프라 |

---

## 1. 설계 문서 구성 계획 (daon-hw.design.md 목차)

### 1.1 전체 목차

| 섹션 | 내용 | 대응 PRD |
|---|---|---|
| §1 시스템 전체 구조 | L1 디바이스 블록 다이어그램 + L2 클라우드 인터페이스 | §9 |
| §2 폼팩터 & 기구 설계 | 외관 치수·봉제 커버·LED 위치·버튼 배치·IP54 | §9.1 폼팩터 |
| §3 SBC 플랫폼 선정 | Jetson Orin Nano vs RK3588 비교 + 최종 결정 ADR | §9.1 컴퓨팅 |
| §4 음성 입출력 회로 | 4-mic MEMS 어레이 + 빔포밍 DSP + 앰프 + 스피커 | §9.1 음성 |
| §5 전원 & 연결성 | 어댑터 + 배터리 (2hr 백업) + WiFi 6 + BT 5.0 + LTE | §9.1 전원 |
| §6 센서 인터페이스 | 터치 + 조도 + 온도 + 가속도 | §9.1 센서 |
| §7 임베디드 OS — NervesOS | Buildroot 기반 Elixir/NervesOS 부트 구조 + OTA | ADR-04 |
| §8 펌웨어 아키텍처 | OTP 모듈 구성 (Supervisor 트리) + GenServer 목록 | — |
| §9 Wake Word 엔진 | openWakeWord / Picovoice Porcupine 비교 + 통합 | §9.1 |
| §10 엣지 ASR 파이프라인 | Whisper distil KO fine-tune + RTF ≤ 0.1 달성 | §9.2 |
| §11 오미↔클라우드 통신 | WebSocket 프로토콜 + 재연결·오프라인 큐 | ADR-04 |
| §12 LED 감정 피드백 제어 | 5색 LED 패턴 테이블 + PWM 제어 | §9.1 시각 피드백 |
| §13 로컬 개발 환경 | Nerves firmware 빌드 + burn + livebook 디버깅 | — |
| §14 BOM & 원가 분석 | 부품별 단가 + BOM 합계 목표 ≤ $150 | §9.1 원가 |
| §15 인증 요건 | KC 전자파 + 노인복지용구 + ISMS-P 로드맵 | §9.1 인증 |
| §16 테스트 & 검증 | 음성 SNR 측정 · 지연 측정 · 낙하 테스트 · OTA 검증 | §9.2 |

---

## 2. HW 요구사항 정의 (PRD §9.1 기반)

### 2.1 폼팩터

| 항목 | 요구사항 | 설계 비고 |
|---|---|---|
| 형태 | 탁상형 + 봉제 외피 분리 구조 | 내부 ABS 골격 + 탈착형 봉제 커버 |
| 크기·무게 | H 25~30cm, 무게 ≤ 800g | 어르신 혼자 이동 가능한 한 손 파지 |
| 낙하 내구성 | 70cm 낙하 후 정상 동작 | 탁상 기준, MIL-STD-810G 일부 준용 |
| 생활방수 | IP54 이상 | 음료 엎지름 방지 (Ingress Protection) |
| 시각 피드백 | LED 귀·눈 5색 | 청각 장애 어르신 보조, §12에서 상세 |
| 물리 버튼 | 볼륨 +/-, 전원, 비상호출 | 3개 이상, 촉각 식별 가능 크기 |

### 2.2 음성 입출력

| 항목 | 요구사항 | 구현 방향 |
|---|---|---|
| 마이크 | 4-mic MEMS 원형 어레이 + 빔포밍 | xCORE 또는 XVSM-2000 급 DSP |
| Wake Word 지연 | ≤ 500ms (로컬 처리) | 전용 Wake Word 칩 또는 NPU 오프로드 |
| TV 소음 제거 | SNR ≥ 20dB (TV 동시 재생 환경) | 빔포밍 + 에코 캔슬레이션 |
| 스피커 출력 | 1~3W, 최대 90dB@1m | 청각 저하 어르신 대응 |
| 볼륨 조절 | 물리 +/- 버튼 + 음성 명령 양방향 | |

### 2.3 컴퓨팅 플랫폼

| 항목 | 요구사항 | 후보 |
|---|---|---|
| 메인 SBC | NPU 지원, RAM ≥ 4GB | Jetson Orin Nano (8GB) / RK3588 기반 Orange Pi 5 |
| 저장 | ≥ 32GB (로컬 ASR 모델 + 자서전 캐시) | eMMC 또는 NVMe SSD |
| 엣지 ASR | Whisper distil KO fine-tune, RAM ≤ 2GB | ONNX Runtime + CUDA/NPU |
| RTF 목표 | ≤ 0.1 (10초 발화 → ≤ 1초 처리) | §9.2 지연 분석 달성 조건 |

#### SBC 후보 비교 (ADR-HW-01)

| 항목 | Jetson Orin Nano 8GB | RK3588 (Orange Pi 5 Plus) |
|---|---|---|
| CPU | 6-core ARM Cortex-A78AE | 4×A76 + 4×A55 |
| GPU/NPU | 1024 CUDA cores + 1.0 TOPS NPU | 6 TOPS NPU (Rockchip NPU) |
| RAM | 8GB LPDDR5 | 16GB LPDDR4X |
| 저장 | NVMe + UHS-1 microSD | NVMe + UHS-1 microSD |
| WiFi | 별도 M.2 필요 | WiFi 6 내장 옵션 |
| 단가 | ~$149 (2026 기준) | ~$89 |
| NervesOS 지원 | 비공식 (패키지 직접 작성) | 비공식 (비슷한 수준) |
| CUDA 생태계 | 강함 (PyTorch/ONNX 직접) | ONNX NPU 필요 |
| **권고** | **1차 프로토타입 권장** | 양산 단가 절감 시 전환 검토 |

### 2.4 전원 & 연결성

| 항목 | 요구사항 | 비고 |
|---|---|---|
| 기본 전원 | AC 어댑터 (5V/5A 또는 12V/3A) | 상시 전원 기본 |
| 배터리 백업 | Li-Po 5,000~10,000mAh, 2hr 동작 | 정전 대비 |
| 통신 (필수) | WiFi 6 (802.11ax) + BT 5.0 | 공공시설 AP 2.4/5GHz 모두 대응 |
| 통신 (옵션) | LTE Cat-M1 (eSIM/나노심) | WiFi 불안정 환경 (농촌 시설 등) |
| USB | USB-A × 1 이상 | 사진 업로드 + USB 카메라 연결 |
| OTA 업데이트 | WiFi/LTE 통해 NervesOS 이미지 푸시 | Nerves OTA Hub 또는 자체 서버 |

### 2.5 센서

| 센서 | 용도 | 인터페이스 |
|---|---|---|
| 정전식 터치 | 쓰다듬기 인식 → 정서 인터랙션 | I2C (FT6236 또는 유사) |
| 조도 | 야간 모드 자동 전환 | I2C (BH1750) |
| 온도/습도 | 환경 이상 감지 (방치 알림) | I2C (SHT31) |
| 6축 IMU | 낙하·이동 감지 | I2C/SPI (ICM-42688) |

---

## 3. 임베디드 OS & 펌웨어 계획

### 3.1 NervesOS 선택 근거

| 항목 | 내용 |
|---|---|
| **OS** | Nerves (Elixir + Buildroot) |
| **이유** | ① Elixir OTP 내결함성 — 개별 GenServer 크래시가 전체 재부팅 없이 재시작 ② BEAM VM의 경량 실시간 처리 ③ 동일 언어(Elixir)로 엣지·클라우드 공유 가능 |
| **대안** | Android Things (지원 종료), Yocto (학습 비용 높음), Raspberry Pi OS (과잉) |

### 3.2 펌웨어 OTP Supervisor 트리

```
Application (daon_omi)
├── AudioSupervisor
│   ├── WakeWordDetector        # GenServer — Wake Word 탐지 루프
│   ├── ASRPipeline             # GenServer — Whisper distil 추론
│   ├── TTSPlayer               # GenServer — TTS 오디오 스트리밍 재생
│   └── MicArrayCapture         # GenServer — ALSA 마이크 캡처
│
├── NetworkSupervisor
│   ├── WebSocketClient         # GenServer — 다온 클라우드 WebSocket
│   ├── OfflineQueue            # GenServer — WiFi 단절 시 발화 버퍼링
│   └── OTAClient               # GenServer — NervesOTA 업데이트 수신
│
├── SensorSupervisor
│   ├── TouchSensor             # GenServer — I2C 터치 이벤트
│   ├── IMUSensor               # GenServer — 낙하·이동 감지
│   └── EnvironmentSensor       # GenServer — 온도·조도
│
└── UISupervisor
    ├── LEDController           # GenServer — PWM LED 패턴 제어
    └── ButtonHandler           # GenServer — 물리 버튼 인터럽트
```

### 3.3 핵심 GenServer 명세

| 모듈 | 상태 | 핵심 메시지 |
|---|---|---|
| `WakeWordDetector` | `{model_pid, threshold}` | `{:audio_chunk, binary}` → `{:wake_word_detected}` |
| `ASRPipeline` | `{model_path, session}` | `{:transcribe, audio}` → `{:text_result, String.t()}` |
| `WebSocketClient` | `{conn, queue, retry_count}` | `{:send_utterance, text}` / `{:receive_response, json}` |
| `OfflineQueue` | `[{timestamp, utterance}]` | 재연결 시 자동 플러시 |
| `LEDController` | `{current_pattern, intensity}` | `{:set_pattern, atom}` |

---

## 4. 음성 파이프라인 & 응답 지연 계획

### 4.1 응답 지연 예산 (PRD §9.2 기반)

```
[어르신 발화 종료]
  Wake Word 로컬 탐지:              < 100ms   WakeWordDetector (NPU 오프로드)
  엣지 ASR (Whisper distil KO):    200~400ms  ASRPipeline (Jetson CUDA, RTF ≤ 0.1)
  네트워크 왕복 (WiFi → 클라우드):  50~150ms   WebSocketClient
  LLM 첫 토큰 출력 (TTFT):         ≤ 200ms   EXAONE 스트리밍
  TTS 합성 (첫 청크 재생):          100~200ms  TTSPlayer 스트리밍
  스피커 출력 버퍼:                  < 50ms
  ─────────────────────────────────────────────
  합계 (평균):                      ~700~1,100ms  ✅ 목표 달성
  합계 (비관):                      ~1,400~1,800ms ⚠️ 한계 (SC-01)

달성 조건:
  1. Whisper distil RTF ≤ 0.1 (Jetson Orin Nano Cuda 기준)
  2. WebSocket 스트리밍 응답 (첫 토큰 즉시 TTS 시작)
  3. TTS 스트리밍 합성 (문장 완성 전 앞부분 재생)
```

### 4.2 Wake Word 엔진 비교 (ADR-HW-02)

| 항목 | openWakeWord | Picovoice Porcupine |
|---|---|---|
| 라이선스 | Apache 2.0 | 상업 (무료 티어 제한) |
| 한국어 지원 | 커스텀 학습 필요 | 커스텀 학습 지원 |
| 정확도 (FPR) | ~ 0.5/hr | ~ 0.5/hr |
| 온디바이스 크기 | ~10MB | ~2MB |
| 지연 | < 100ms | < 100ms |
| **권고** | **openWakeWord** (오픈소스·비용 우선) | 상업 정확도 필요 시 전환 |

### 4.3 엣지 ASR 파이프라인

```
ALSA Raw Audio (PCM 16kHz, 16bit, mono)
  │
  ▼ MicArrayCapture (MEMS 4ch → 빔포밍 → 1ch)
  │
  ▼ WakeWordDetector (openWakeWord, NPU)
  │ {wake_word_detected}
  ▼ ASRPipeline
  │  ├─ VAD (음성 구간 감지, Silero VAD ONNX)
  │  ├─ Whisper distil-large-v3-ko (ONNX, CUDA)
  │  └─ 출력: {text: "...", language: "ko", duration_ms: NNN}
  │
  ▼ WebSocketClient.send_utterance(text)
```

---

## 5. 오미↔클라우드 통신 설계 (ADR-04 구현)

### 5.1 연결 상태 머신

```
[초기화] → [WiFi 연결 대기] → [WebSocket 연결]
                                      │
                            ┌─────────▼──────────┐
                            │  CONNECTED          │
                            │  정상 대화 처리      │
                            └─────────┬──────────┘
                                      │ 연결 끊김
                            ┌─────────▼──────────┐
                            │  OFFLINE_BUFFERING  │
                            │  발화 → OfflineQueue │
                            └─────────┬──────────┘
                                      │ 재연결 성공
                            ┌─────────▼──────────┐
                            │  RECONNECTING       │
                            │  큐 플러시 + 재개    │
                            └─────────────────────┘
```

### 5.2 WebSocket 메시지 타입 (daon-sw.design.md §8.9와 동기화)

| 방향 | 타입 | 내용 |
|---|---|---|
| 오미→클라우드 | `utterance` | `{text, session_id, timestamp, emotion_pre_score}` |
| 클라우드→오미 | `response_chunk` | `{text_chunk, audio_chunk_b64, is_final}` |
| 클라우드→오미 | `session_event` | `{event: crisis_detected, level, payload}` |
| 오미→클라우드 | `sensor_event` | `{type: touch/fall/idle, timestamp}` |

### 5.3 오프라인 큐 정책

- **큐 용량:** 최대 50개 발화 (로컬 ETS 또는 DETS 파일)
- **재연결 후 처리:** FIFO, 타임스탬프 포함 전송
- **TTL:** 24시간 초과 발화는 폐기 (로컬 저장 → 클라우드 유실 방지 우선)

---

## 6. LED 감정 피드백 제어 계획

### 6.1 5색 LED 패턴 테이블 (PRD §9.1 시각 피드백)

| 상태 | 색 | 패턴 | 의미 |
|---|---|---|---|
| 대기 (Idle) | 흰색 | 느린 호흡 (2초 주기) | 대기 중 |
| Wake Word 감지 | 하늘색 | 빠른 점멸 (0.2초) | 듣고 있어요 |
| 발화 인식 중 | 노란색 | 회전 (스피너) | 처리 중 |
| 응답 재생 | 초록색 | 밝기 맥박 (TTS 동기) | 말하고 있어요 |
| 슬픔 감지 (sadness ≥ 0.6) | 보라색 | 느린 페이드인 | 공감 모드 |
| 위기 감지 (Level 2+) | 빨간색 | 급속 점멸 | 긴급 상태 |
| 충전 중 | 주황색 | 서서히 채워짐 | 충전 진행 |
| 오류·오프라인 | 빨간색 | 3회 점멸 후 꺼짐 | 연결 오류 |

### 6.2 PWM 제어 인터페이스

```elixir
# lib/daon_omi/ui/led_controller.ex

defmodule DaonOmi.UI.LEDController do
  use GenServer

  @patterns %{
    idle:        {0xFFFFFF, :breathe, 2000},
    listening:   {0x87CEEB, :blink_fast, 200},
    processing:  {0xFFFF00, :spinner, 500},
    speaking:    {0x00FF00, :pulse, :tts_sync},
    empathy:     {0x800080, :fade_in_slow, 3000},
    crisis:      {0xFF0000, :rapid_blink, 100},
    charging:    {0xFF8C00, :fill, :charge_level},
    error:       {0xFF0000, :blink_3x, 500},
  }

  def set_pattern(pattern) when is_atom(pattern) do
    GenServer.cast(__MODULE__, {:set_pattern, pattern})
  end
end
```

---

## 7. BOM 원가 계획

### 7.1 부품별 단가 (Phase 1 프로토타입 기준)

| 분류 | 부품 | 단가 (USD) | 수량 | 소계 |
|---|---|---|---|---|
| **컴퓨팅** | Jetson Orin Nano 8GB (개발 키트) | $149 | 1 | $149 |
| | *(양산 시 RK3588 전환 검토)* | *$89* | | |
| **마이크** | MEMS 4ch 어레이 모듈 + 빔포밍 DSP | $20 | 1 | $20 |
| **스피커** | 3W 스피커 + 클래스 D 앰프 모듈 | $12 | 1 | $12 |
| **통신** | WiFi 6 + BT 5.0 M.2 카드 | $15 | 1 | $15 |
| | LTE Cat-M1 모듈 (옵션) | $20 | 1 | ($20) |
| **전원** | Li-Po 7,500mAh 팩 + BMS | $18 | 1 | $18 |
| | 12V/3A 어댑터 | $8 | 1 | $8 |
| **센서** | 터치+조도+온도+IMU 패키지 | $10 | 1 | $10 |
| **LED** | RGB LED 어레이 + 드라이버 | $8 | 1 | $8 |
| **케이스** | ABS 골격 3D 프린팅 (프로토타입) | $15 | 1 | $15 |
| **봉제 외피** | 커스텀 봉제 (시제품) | $30 | 1 | $30 |
| **기타** | PCB 기판·커넥터·케이블 | $15 | 1 | $15 |
| **합계 (Jetson)** | | | | **~$300** (프로토타입) |
| **합계 (RK3588)** | | | | **~$160** (양산 목표 ≤$150) |

> **참고:** PRD §9.1 BOM 목표 $150은 양산 기준. Phase 1 프로토타입은 Jetson 개발킷 사용으로 초과.
> 양산 시 RK3588 SoM + 커스텀 PCB로 전환 → $150 달성 가능.

### 7.2 완제품 가격 목표

| 채널 | 납품가 목표 | 비고 |
|---|---|---|
| B2G (보건소·지자체) | ≤ 200만원 | PRD 기준 |
| B2C (가족 직접) | ≤ 300만원 | 프리미엄 포지셔닝 |
| B2B (요양시설) | ≤ 180만원/대 (10대+ 구매 시) | 볼륨 할인 |

---

## 8. 인증 요건 로드맵

| 인증 | 필수 여부 | 예상 기간 | 비고 |
|---|---|---|---|
| **KC 전자파** | 필수 (법적 요건) | 6~8주 | 국내 판매 법적 요건, 시험기관: KTC/KR |
| **KC 전기안전** | 필수 | 6~8주 (KC EMC와 병행) | 어댑터·배터리 포함 |
| **노인복지용구 인증** | B2G 진입 시 필수 | 12~24주 | 보건복지부 지정, 공공 조달 요건 |
| **ISMS-P** | 단계적 (Phase 2+) | 6~12개월 | B2G 신뢰도 요건, 의료 인접 서비스 |
| **IP54 시험** | 권장 | 2~4주 | IEC 60529 기준, 마케팅 소구점 |

---

## 9. 성공 기준 (daon-hw.design.md 기준)

| ID | 기준 | 측정 방법 |
|---|---|---|
| HW-01 | Wake Word 지연 ≤ 500ms | 오디오 입력~감지 이벤트 타임스탬프 차이 |
| HW-02 | 엣지 ASR RTF ≤ 0.1 (Jetson CUDA) | 10초 발화 → 처리 ≤ 1초 |
| HW-03 | 마이크 SNR ≥ 20dB (TV 동시 재생) | 표준 소음 환경 측정 |
| HW-04 | 배터리 백업 ≥ 2시간 (무충전 동작) | 충전 없이 대화 루프 실행 |
| HW-05 | 70cm 낙하 후 정상 동작 | 3회 반복 낙하 테스트 |
| HW-06 | NervesOS 빌드·플래시 ≤ 1시간 (초기) | mix firmware && mix upload 성공 |
| HW-07 | OTA 업데이트 성공률 ≥ 99% | 10회 반복 OTA 테스트 |
| HW-08 | LED 패턴 8종 + 감정 연동 정상 동작 | 각 상태 수동 트리거 검증 |

---

## 10. 리스크

| 리스크 | 영향 | 대응 |
|---|---|---|
| Jetson Orin Nano 수급·단가 | 프로토타입 일정 지연 | RK3588 Orange Pi 5를 2순위로 병행 발주 |
| NervesOS Jetson 패키지 미성숙 | 포팅 시간 초과 | Raspberry Pi 5로 1차 펌웨어 개발 후 Jetson 이식 |
| KC 인증 리드타임 6~8주 | 파일럿 일정 지연 | Phase 1 프로토타입 인증 면제 구간 활용 → 별도 협의 |
| 방언 ASR fine-tune 데이터셋 라이선스 | 법적 리스크 | AI Hub 방언 데이터셋 (공공) 우선 사용 + 라이선스 검토 |
| 봉제 외피 ODM 납기 | 외관 지연 | 3D 프린팅 케이스로 파일럿 시작 |

---

## 11. 구현 세션 계획 (daon-hw.design.md 작성 순서)

| 세션 | 범위 | 예상 소요 |
|---|---|---|
| S-01 | §1~2 시스템 구조 + 폼팩터 기구 설계 | 1일 |
| S-02 | §3 SBC 선정 ADR + §4 음성 입출력 회로 | 1일 |
| S-03 | §5 전원·연결성 + §6 센서 인터페이스 | 1일 |
| S-04 | §7 NervesOS 부트 구조 + §8 펌웨어 OTP 트리 | 2일 |
| S-05 | §9 Wake Word + §10 엣지 ASR 파이프라인 | 2일 |
| S-06 | §11 클라우드 통신 + §12 LED 제어 | 1일 |
| S-07 | §13 로컬 개발 환경 + §14 BOM 상세 | 1일 |
| S-08 | §15 인증 + §16 테스트·검증 계획 | 1일 |
| **합계** | **전체 HW 아키텍처 문서** | **~10일** |

---

## 12. 다음 단계

- `/pdca design daon-hw` — 위 계획 기반으로 `daon-hw.design.md` 작성 시작
- `/pdca design daon-hw --scope S-01,S-02` — 첫 세션 (시스템 구조 + 음성 회로)
