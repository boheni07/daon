# 오미(Omi) HW 아키텍처 설계서

> **Feature:** daon-hw
> **작성일:** 2026-05-24
> **Phase:** Design
> **Status:** In Progress
> **Option:** C — 실용적 HW+펌웨어 명세 (블록 다이어그램 + GenServer 시그니처 + 핵심 알고리즘 + BOM)
> **참조 Plan:** `docs/01-plan/features/daon-hw.plan.md`
> **참조 SW 설계:** `docs/02-design/features/daon-sw.design.md`

---

## Context Anchor

| Anchor | 값 |
|---|---|
| **WHY** | PRD §9.1 스펙 요구사항이 확정된 지금, HW 설계자·임베디드 엔지니어가 독립 착수하려면 SBC 선정·회로·NervesOS 펌웨어 수준의 명세가 필요 |
| **WHO** | NUBiz AX HW 설계자 · 임베디드 엔지니어 (Elixir/NervesOS) · ODM/제조 파트너 |
| **RISK** | Jetson Orin Nano 수급·단가 / NervesOS Jetson 패키지 미성숙 / KC 인증 리드타임 6~8주 |
| **SUCCESS** | HW 설계자·임베디드 엔지니어 각자 독립 착수 / 로컬 NervesOS 개발 환경 ≤ 1일 셋업 (HW-06) |
| **SCOPE** | (IN) L1 Omi HW + 임베디드 OS/펌웨어, BOM, 인증 / (OUT) L2~L5 클라우드 SW, 양산 금형·공정, KiCad 스키매틱 |

---

## §1. 시스템 전체 구조

### 1.1 L1 Omi 블록 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│  오미(Omi) 디바이스  ─────────────────────────────────────────────  │
│                                                                     │
│  ┌─────────────┐   I2S/USB   ┌────────────────────────────────┐   │
│  │ 4-mic MEMS  │────────────►│   SBC (Jetson Orin Nano 8GB)   │   │
│  │ 어레이 + DSP │            │                                │   │
│  └─────────────┘            │  ┌──────────┐  ┌────────────┐  │   │
│                              │  │ NervesOS │  │  CUDA NPU  │  │   │
│  ┌─────────────┐   I2S       │  │ (Elixir) │  │ Whisper    │  │   │
│  │ 스피커 3W   │◄────────────│  │ OTP 앱   │  │ distil KO  │  │   │
│  │ 클래스D 앰프 │            │  └──────────┘  └────────────┘  │   │
│  └─────────────┘            │                                │   │
│                              │  ┌──────────┐  ┌────────────┐  │   │
│  ┌─────────────┐   I2C/SPI  │  │ WiFi 6   │  │ BT 5.0     │  │   │
│  │ 센서 허브   │────────────►│  │ M.2 카드 │  │ 내장       │  │   │
│  │(터치·IMU·   │            │  └──────────┘  └────────────┘  │   │
│  │ 온도·조도)  │            │                                │   │
│  └─────────────┘            │  ┌──────────┐  ┌────────────┐  │   │
│                              │  │ eMMC 64GB│  │ LTE Cat-M1 │  │   │
│  ┌─────────────┐   GPIO      │  │ (NVMe    │  │ (옵션)     │  │   │
│  │ LED 어레이  │◄────────────│  │  옵션)   │  │            │  │   │
│  │ (눈·귀 RGB) │            │  └──────────┘  └────────────┘  │   │
│  └─────────────┘            └────────────────────────────────┘   │
│                                          │                         │
│  ┌─────────────┐   USB-C               │ USB-A (사진)             │
│  │ 물리 버튼 3 │◄──────────────────────│                          │
│  │(볼륨+/-, 전 │                       │ WebSocket / TLS          │
│  │ 원, 비상)   │            ┌──────────▼─────────────────────┐   │
│  └─────────────┘            │          인터넷 (WiFi/LTE)      │   │
│                              └────────────┬────────────────────┘   │
│  ┌─────────────┐                          │                         │
│  │ 배터리 7.5Ah│◄─ BMS ──── 12V 어댑터   │                         │
│  └─────────────┘                          ▼                         │
└──────────────────────────────   다온 클라우드 L2~L5   ─────────────┘
```

### 1.2 L1↔L2 인터페이스 요약

| 인터페이스 | 방향 | 프로토콜 | 포맷 |
|---|---|---|---|
| 대화 발화 | Omi→Cloud | WebSocket (TLS) | `{type: utterance, text, session_id, timestamp}` |
| 응답 스트림 | Cloud→Omi | WebSocket | `{type: response_chunk, text_chunk, audio_b64, is_final}` |
| 세션 이벤트 | Cloud→Omi | WebSocket | `{type: session_event, event, crisis_level}` |
| 센서 이벤트 | Omi→Cloud | WebSocket | `{type: sensor_event, event: touch/fall/idle}` |
| OTA 업데이트 | Cloud→Omi | HTTPS | NervesOS 펌웨어 이미지 |

---

## §2. 폼팩터 & 기구 설계

### 2.1 외관 치수

```
        ┌──────────────────────┐
   30cm │   ╭──────────────╮   │
        │   │  봉제 외피    │   │
        │   │  (탈착·세탁)  │   │
        │   │               │   │
        │   │  ◉ 왼쪽 눈   │   │
        │   │  ◉ 오른쪽 눈  │   ├─► LED RGB 눈 (표정)
        │   │    ─── ────   │   │
        │   │  ◉ 왼쪽 귀   │   ├─► LED RGB 귀 (상태)
        │   │  ◉ 오른쪽 귀  │   │
        │   │               │   │
        │   │  [  마이크  ] │   ├─► 4-mic 어레이 (전면 중앙)
        │   ╰──────────────╯   │
        │  ┌────────────────┐  │
        │  │ ABS 베이스 골격 │  ├─► [+] [-] 볼륨, [전원], [비상]
        │  │ H=30cm, ≤800g │  ├─► USB-A (측면)
        │  └────────────────┘  │
        └──────────────────────┘
           ◄──── 20cm ────►
```

### 2.2 기구 설계 요구사항

| 항목 | 사양 | 구현 방향 |
|---|---|---|
| 크기 | H 25~30cm, W ~20cm | ABS 사출 골격 (양산) / FDM 3D 프린팅 (프로토타입) |
| 무게 | ≤ 800g | SBC ~150g + 배터리 ~200g + 기구 ~300g + 봉제 ~100g |
| 낙하 내구성 | 70cm 3회 정상 동작 | 내부 폼 완충재 + ABS 두께 ≥ 2.5mm |
| 방수 | IP54 (생활방수) | 마이크 홀: 방수 메시 필터, 버튼: 실리콘 씰 |
| 봉제 외피 | 탈착·세탁 가능 | 자석 버튼 또는 벨크로 탈착, 면 혼방 소재 |
| 냉각 | 수동 (팬리스) | SBC 써멀패드 + ABS 골격 히트싱크 역할 |

---

## §3. SBC 플랫폼 선정 (ADR-HW-01)

### 3.1 후보 비교

| 항목 | Jetson Orin Nano 8GB | Orange Pi 5 Plus (RK3588) | Raspberry Pi 5 (대안) |
|---|---|---|---|
| CPU | 6×A78AE (2.0GHz) | 4×A76 + 4×A55 | 4×A76 (2.4GHz) |
| GPU | 1024 CUDA (11.1) | Mali-G610 MP4 | VideoCore VII |
| NPU | 1.0 TOPS | 6.0 TOPS (RKNPU) | 없음 |
| RAM | 8GB LPDDR5 | 16GB LPDDR4X | 8GB LPDDR4X |
| 저장 | NVMe PCIe + microSD | NVMe PCIe + microSD | NVMe + microSD |
| WiFi | M.2 별도 필요 | WiFi 6 내장 옵션 | WiFi 5 내장 |
| 단가 | ~$149 (개발킷) | ~$89 | ~$80 |
| CUDA/PyTorch | 완벽 지원 | RKNN Toolkit 필요 | 미지원 |
| NervesOS 지원 | 비공식 (직접 패키지) | 비공식 | 공식 (rpi5) |
| Whisper RTF ≤ 0.1 | ✅ CUDA 기준 | ⚠️ RKNPU (확인 필요) | ❌ CPU만으로 불가 |
| **결정** | **Phase 1 프로토타입** | **양산 전환 시 검토** | **펌웨어 개발 대안** |

### 3.2 결정 근거 (ADR-HW-01)

```
결정: Phase 1 → Jetson Orin Nano 8GB
      Phase 2+ → RK3588 (양산 단가 절감 시)

근거:
  1. Whisper distil KO 추론 RTF ≤ 0.1 달성 가능성
     - Jetson CUDA: ~0.07 RTF (10초 발화 → 0.7초 처리)
     - RK3588 NPU: ~0.12 RTF (검증 필요)
  2. ONNX/PyTorch 생태계 성숙도 (Jetson 압도적 유리)
  3. NervesOS 포팅: Rpi5로 1차 개발 → Jetson 이식 전략

위험 완화:
  - 단가 $149 초과 → 양산 시 RK3588 + 커스텀 PCB로 전환
  - Jetson 수급 이슈 → Orange Pi 5 병행 발주
```

---

## §4. 음성 입출력 회로

### 4.1 4-마이크 MEMS 어레이

```
오미 전면 마이크 배치 (원형):
      MIC_0 (12시)
   ╭──────────╮
MIC_3 ●       ● MIC_1    반경 ~3cm 원형
   │     ●     │          → 360° 빔포밍
MIC_2 ●       │           → SNR ≥ 20dB (TV 소음 환경)
   ╰──────────╯
      (6시)

마이크 칩: Knowles SPH0645LM4H-B (I2S, MEMS, SNR 65dB)
           또는 TDK ICS-43434 (I2S, MEMS, SNR 65dB)
           × 4개 (I2S TDM 다중화 또는 각각 I2S)

빔포밍 DSP:
  - XVSM-2000 (XMOS xCORE) : 전용 빔포밍·에코캔슬·VAD
  - 또는 SBC CPU 소프트웨어 빔포밍 (Phase 1 단순화)

에코 캔슬레이션 (AEC):
  - 스피커 출력 → 피드백 → 마이크 로컬 루프백 (full-duplex)
  - XVSM 또는 WebRTC AEC (SBC 소프트웨어)
```

### 4.2 스피커 앰프 회로

```
SBC I2S 출력
     │
     ▼
[MAX98357A 클래스D 앰프]  ← I2S 직결, 외장 부품 최소화
  - 출력: 3.2W (4Ω, 5V)
  - THD+N: < 0.013%
  - I2S 24bit / 48kHz
     │
     ▼
[스피커: 3W / 4Ω / 90dB@1m]  ← 청각 저하 어르신 대응
  예: Visaton FRS 7 또는 동급
```

### 4.3 음성 파이프라인 데이터 흐름

```
MEMS 마이크 4ch (I2S TDM)
         │
         ▼
[빔포밍 DSP (XVSM-2000 또는 SW)]
  - AEC (에코 캔슬)
  - 빔포밍 (방향 집속)
  - VAD (발화 구간 감지)
  → 출력: 16kHz / 16bit / mono PCM
         │
         ▼
[WakeWordDetector GenServer]
  openWakeWord ONNX (NPU)
  → :wake_word_detected
         │
         ▼
[ASRPipeline GenServer]
  Silero VAD → Whisper distil-large-v3-ko (CUDA/ONNX)
  → {:text_result, "발화 텍스트", duration_ms}
         │
         ▼
[WebSocketClient GenServer]
  → {:send_utterance, text, session_id}
  ← {:response_chunk, text, audio_b64}
         │
         ▼
[TTSPlayer GenServer]
  base64 decode → PCM 청크 → ALSA 출력
  → MAX98357A → 스피커
```

---

## §5. 전원 & 연결성

### 5.1 전원 아키텍처

```
12V / 3A DC 어댑터 (36W)
         │
         ▼
[Power Distribution Board]
  ├─► 12V → 5V (DC-DC 벅): SBC (Jetson 최대 15W)
  ├─► 5V → 3.3V (LDO): 마이크 DSP, 센서
  ├─► 5V → 앰프 (MAX98357A)
  └─► Li-Po 충전 회로 (BQ24195 또는 유사)
              │
              ▼
[Li-Po 배터리 7,500mAh 3.7V]
  BMS: 과충전·과방전·단락 보호
  방전 시: 부스트 변환기 → 12V 공급 라인 유지
  백업 용량: 2시간 이상 (HW-04)
              │
              ▼
[퓨즈 + TVS 다이오드] ← 서지 보호
```

### 5.2 통신 모듈

| 모듈 | 사양 | 인터페이스 |
|---|---|---|
| WiFi 6 + BT 5.0 | Intel AX210 M.2 (Jetson용) | PCIe M.2 E-key |
| LTE Cat-M1 (옵션) | SIMCOM SIM7080G | USB 또는 mPCIe |
| GPS (옵션) | u-blox M8N | UART (복지 서비스 위치 확인용) |

### 5.3 USB 포트 용도

| 포트 | 용도 |
|---|---|
| USB-A × 1 (외부 노출) | 사진 USB 드라이브 업로드, USB 카메라 연결 |
| USB-C (내부 디버그) | UART 콘솔 (개발·서비스용) |

---

## §6. 센서 인터페이스

### 6.1 센서 연결 맵

```
SBC I2C Bus 0:
  ├─ 0x38: FT6236 정전식 터치 (쓰다듬기 인식)
  ├─ 0x23: BH1750 조도 센서 (야간 모드)
  └─ 0x44: SHT31 온습도 센서 (환경 모니터링)

SBC I2C Bus 1 (또는 SPI):
  └─ ICM-42688-P 6축 IMU (가속도·자이로, 낙하·이동 감지)

GPIO:
  ├─ 물리 버튼 3개 (볼륨+, 볼륨-, 비상호출) — 풀업 + 디바운스
  └─ LED 제어 (I2C PWM 드라이버: PCA9685 16ch)
```

### 6.2 센서 이벤트 → GenServer 매핑

| 센서 이벤트 | GenServer | 클라우드 전달 |
|---|---|---|
| 터치 (쓰다듬기) | `TouchSensor` | `sensor_event: {type: touch}` |
| 낙하 감지 (acc > 2g) | `IMUSensor` | `sensor_event: {type: fall}` |
| 24시간 모션 없음 | `IMUSensor` | `sensor_event: {type: idle}` |
| 조도 < 10lux | `EnvironmentSensor` | LED 야간 모드 전환 (로컬) |
| 온도 > 40°C | `EnvironmentSensor` | `sensor_event: {type: temp_high}` |

---

## §7. 임베디드 OS — NervesOS

### 7.1 NervesOS 아키텍처

```
┌─────────────────────────────────────────────────────┐
│  NervesOS (Buildroot + Linux + BEAM VM)              │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Mix/OTP Application: daon_omi              │   │
│  │                                             │   │
│  │  AudioSupervisor │ NetworkSupervisor        │   │
│  │  SensorSupervisor │ UISupervisor            │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  BEAM VM (Erlang/OTP 26)                            │
│  Linux Kernel (커스텀, 최소 드라이버셋)               │
│  Buildroot (루트 파일시스템)                         │
│  U-Boot (부트로더)                                   │
└─────────────────────────────────────────────────────┘
```

### 7.2 Nerves 프로젝트 구조

```
daon_omi/
├── mix.exs                    # Mix 의존성 (nerves, circuits_i2c 등)
├── config/
│   ├── config.exs             # 공통 설정
│   ├── target.exs             # 타깃별 설정 (jetson / rpi5)
│   └── host.exs               # 호스트 시뮬레이션 설정
├── lib/
│   └── daon_omi/
│       ├── application.ex     # OTP 앱 엔트리포인트
│       ├── audio/
│       │   ├── audio_supervisor.ex
│       │   ├── wake_word_detector.ex
│       │   ├── asr_pipeline.ex
│       │   ├── tts_player.ex
│       │   └── mic_array_capture.ex
│       ├── network/
│       │   ├── network_supervisor.ex
│       │   ├── websocket_client.ex
│       │   ├── offline_queue.ex
│       │   └── ota_client.ex
│       ├── sensor/
│       │   ├── sensor_supervisor.ex
│       │   ├── touch_sensor.ex
│       │   ├── imu_sensor.ex
│       │   └── environment_sensor.ex
│       └── ui/
│           ├── ui_supervisor.ex
│           ├── led_controller.ex
│           └── button_handler.ex
├── rootfs_overlay/            # 타깃 파일시스템 오버레이
│   └── etc/
│       ├── nerves_network.json
│       └── omi_config.json    # 디바이스 설정 (WiFi, 클라우드 URL)
└── priv/
    └── models/
        ├── wake_word.onnx     # openWakeWord 모델
        └── whisper_distil_ko.onnx
```

### 7.3 mix.exs 핵심 의존성

```elixir
# mix.exs
defp deps do
  [
    {:nerves, "~> 1.10", runtime: false},
    {:nerves_runtime, "~> 0.13"},
    {:nerves_pack, "~> 0.7"},            # WiFi, mDNS, NTP 기본 셋업
    {:vintage_net, "~> 0.13"},           # 네트워크 관리
    {:circuits_i2c, "~> 2.0"},           # I2C 센서
    {:circuits_gpio, "~> 1.0"},          # GPIO 버튼·LED
    {:ortex, "~> 0.1"},                  # ONNX Runtime (Whisper, Wake Word)
    {:websock_adapter, "~> 0.5"},        # WebSocket 클라이언트
    {:mint_web_socket, "~> 1.0"},        # HTTP/WS 클라이언트
    {:jason, "~> 1.4"},                  # JSON
    {:ring_logger, "~> 0.10"},           # 순환 로그 (임베디드 최적화)
  ]
end

@all_targets [:jetson_orin_nano, :rpi5, :host]
```

---

## §8. 펌웨어 OTP Supervisor 트리 & GenServer 명세

### 8.1 Application 초기화

```elixir
# lib/daon_omi/application.ex

defmodule DaonOmi.Application do
  use Application

  @impl true
  def start(_type, _args) do
    children = [
      DaonOmi.Audio.AudioSupervisor,
      DaonOmi.Network.NetworkSupervisor,
      DaonOmi.Sensor.SensorSupervisor,
      DaonOmi.UI.UISupervisor,
    ]

    opts = [strategy: :one_for_one, name: DaonOmi.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

### 8.2 핵심 GenServer — WakeWordDetector

```elixir
# lib/daon_omi/audio/wake_word_detector.ex

defmodule DaonOmi.Audio.WakeWordDetector do
  use GenServer
  require Logger

  @model_path "/priv/models/wake_word.onnx"
  @threshold 0.5
  @window_ms 1280   # 1.28초 슬라이딩 윈도우

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  @impl true
  def init(_opts) do
    {:ok, model} = Ortex.load(@model_path)
    {:ok, %{model: model, buffer: <<>>, threshold: @threshold}}
  end

  @doc "마이크 청크 수신 → 슬라이딩 윈도우로 Wake Word 탐지"
  @impl true
  def handle_cast({:audio_chunk, pcm_binary}, state) do
    buffer = state.buffer <> pcm_binary
    {score, new_buffer} = run_inference(state.model, buffer)

    if score >= state.threshold do
      Logger.info("Wake word detected (score=#{score})")
      DaonOmi.Audio.ASRPipeline.start_recording()
      DaonOmi.UI.LEDController.set_pattern(:listening)
    end

    {:noreply, %{state | buffer: new_buffer}}
  end

  defp run_inference(model, buffer) when byte_size(buffer) >= @window_ms * 32 do
    # 16kHz × 16bit × 1ch × 1.28s = 40,960 bytes
    <<window::binary-size(40_960), rest::binary>> = buffer
    tensor = Ortex.run(model, [Nx.from_binary(window, :f32)])
    score = Nx.to_number(tensor)
    {score, rest}
  end
  defp run_inference(_model, buffer), do: {0.0, buffer}
end
```

### 8.3 핵심 GenServer — ASRPipeline

```elixir
# lib/daon_omi/audio/asr_pipeline.ex

defmodule DaonOmi.Audio.ASRPipeline do
  use GenServer

  @whisper_model "/priv/models/whisper_distil_ko.onnx"
  @sample_rate 16_000
  @max_record_ms 30_000   # 최대 30초 발화

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def start_recording do
    GenServer.cast(__MODULE__, :start_recording)
  end

  @impl true
  def init(_opts) do
    {:ok, model} = Ortex.load(@whisper_model)
    {:ok, %{model: model, recording: false, buffer: <<>>, timer: nil}}
  end

  @impl true
  def handle_cast(:start_recording, state) do
    timer = Process.send_after(self(), :recording_timeout, @max_record_ms)
    {:noreply, %{state | recording: true, buffer: <<>>, timer: timer}}
  end

  @impl true
  def handle_cast({:audio_chunk, pcm}, %{recording: true} = state) do
    new_buffer = state.buffer <> pcm

    # VAD (음성 활동 감지) — Silero VAD ONNX 또는 간단한 에너지 기반
    if silence_detected?(pcm) and byte_size(new_buffer) > @sample_rate * 2 * 0.5 do
      # 0.5초+ 발화 후 묵음 → 발화 종료
      transcribe_and_send(state.model, new_buffer)
      {:noreply, %{state | recording: false, buffer: <<>>, timer: nil}}
    else
      {:noreply, %{state | buffer: new_buffer}}
    end
  end

  @impl true
  def handle_info(:recording_timeout, state) do
    if byte_size(state.buffer) > 0, do: transcribe_and_send(state.model, state.buffer)
    {:noreply, %{state | recording: false, buffer: <<>>, timer: nil}}
  end

  defp transcribe_and_send(model, pcm_binary) do
    start_ts = System.monotonic_time(:millisecond)

    # Whisper distil ONNX 추론
    audio_tensor = pcm_binary |> :binary.bin_to_list() |> Nx.tensor(type: :f32)
    result = Ortex.run(model, [audio_tensor])
    text = decode_tokens(result)

    duration = System.monotonic_time(:millisecond) - start_ts
    Logger.info("ASR completed in #{duration}ms: #{text}")

    DaonOmi.Network.WebSocketClient.send_utterance(text)
    DaonOmi.UI.LEDController.set_pattern(:processing)
  end

  defp silence_detected?(pcm) do
    # RMS 에너지 < 임계값 → 묵음
    samples = for <<s::signed-16-little <- pcm>>, do: s / 32768.0
    rms = :math.sqrt(Enum.sum(Enum.map(samples, &(&1 * &1))) / length(samples))
    rms < 0.01
  end

  defp decode_tokens(result) do
    # Whisper 토큰 → 텍스트 디코딩
    result |> Nx.to_flat_list() |> Enum.map(&token_to_char/1) |> Enum.join()
  end
end
```

### 8.4 핵심 GenServer — WebSocketClient

```elixir
# lib/daon_omi/network/websocket_client.ex

defmodule DaonOmi.Network.WebSocketClient do
  use GenServer
  require Logger

  @reconnect_interval_ms 5_000
  @max_retry 10

  defmodule State do
    defstruct [:conn, :session_id, :retry_count, status: :disconnected]
  end

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def send_utterance(text) do
    GenServer.cast(__MODULE__, {:send_utterance, text})
  end

  @impl true
  def init(_opts) do
    Process.send_after(self(), :connect, 100)
    {:ok, %State{retry_count: 0}}
  end

  @impl true
  def handle_info(:connect, state) do
    url = Application.get_env(:daon_omi, :cloud_ws_url)
    session_id = generate_session_id()

    case MintWebSocket.connect(url, session_id) do
      {:ok, conn} ->
        Logger.info("WebSocket connected")
        DaonOmi.UI.LEDController.set_pattern(:idle)
        {:noreply, %State{conn: conn, session_id: session_id, status: :connected}}

      {:error, reason} ->
        Logger.warning("WebSocket connect failed: #{inspect(reason)}, retry #{state.retry_count}")
        retry_after = min(@reconnect_interval_ms * (state.retry_count + 1), 60_000)
        Process.send_after(self(), :connect, retry_after)
        {:noreply, %{state | retry_count: min(state.retry_count + 1, @max_retry)}}
    end
  end

  @impl true
  def handle_cast({:send_utterance, text}, %State{status: :connected} = state) do
    payload = Jason.encode!(%{
      type: "utterance",
      text: text,
      session_id: state.session_id,
      timestamp: DateTime.utc_now() |> DateTime.to_iso8601()
    })
    MintWebSocket.send_text(state.conn, payload)
    {:noreply, state}
  end

  @impl true
  def handle_cast({:send_utterance, text}, %State{status: :disconnected} = state) do
    # 오프라인 시 큐에 저장
    DaonOmi.Network.OfflineQueue.enqueue(text)
    {:noreply, state}
  end

  @impl true
  def handle_info({:websocket_message, json}, state) do
    case Jason.decode!(json) do
      %{"type" => "response_chunk"} = msg ->
        handle_response_chunk(msg)

      %{"type" => "session_event", "event" => "crisis_detected", "crisis_level" => level} ->
        handle_crisis(level)

      _ -> :ok
    end
    {:noreply, state}
  end

  @impl true
  def handle_info(:disconnected, state) do
    Process.send_after(self(), :connect, @reconnect_interval_ms)
    DaonOmi.Network.OfflineQueue.flush_on_reconnect()
    {:noreply, %{state | status: :disconnected, retry_count: 0}}
  end

  defp handle_response_chunk(%{"audio_chunk" => audio_b64, "is_final" => is_final}) do
    audio = Base.decode64!(audio_b64)
    DaonOmi.Audio.TTSPlayer.play_chunk(audio, is_final)
    DaonOmi.UI.LEDController.set_pattern(:speaking)
  end

  defp handle_crisis(level) when level >= 2 do
    DaonOmi.UI.LEDController.set_pattern(:crisis)
    DaonOmi.UI.ButtonHandler.enable_emergency_call()
  end
  defp handle_crisis(_level) do
    DaonOmi.UI.LEDController.set_pattern(:empathy)
  end

  defp generate_session_id, do: "omi_" <> Base.encode16(:crypto.strong_rand_bytes(8))
end
```

### 8.5 핵심 GenServer — LEDController

```elixir
# lib/daon_omi/ui/led_controller.ex

defmodule DaonOmi.UI.LEDController do
  use GenServer

  # {색 RGB, 패턴, 파라미터}
  @patterns %{
    idle:       {0xFFFFFF, :breathe, 2000},
    listening:  {0x87CEEB, :blink_fast, 200},
    processing: {0xFFFF00, :spinner, 500},
    speaking:   {0x00FF00, :pulse, :tts_sync},
    empathy:    {0x800080, :fade_in_slow, 3000},
    crisis:     {0xFF0000, :rapid_blink, 100},
    charging:   {0xFF8C00, :fill, :charge_level},
    error:      {0xFF0000, :blink_3x, 500},
  }

  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, opts, name: __MODULE__)
  end

  def set_pattern(pattern) when is_atom(pattern) do
    GenServer.cast(__MODULE__, {:set_pattern, pattern})
  end

  @impl true
  def init(_opts) do
    {:ok, i2c} = Circuits.I2C.open("i2c-1")
    # PCA9685 PWM 드라이버 초기화
    Circuits.I2C.write(i2c, 0x40, <<0x00, 0x20>>)  # MODE1: auto-increment
    {:ok, %{i2c: i2c, current: :idle}}
  end

  @impl true
  def handle_cast({:set_pattern, pattern}, state) do
    {rgb, anim, param} = Map.get(@patterns, pattern, @patterns.idle)
    apply_pattern(state.i2c, rgb, anim, param)
    {:noreply, %{state | current: pattern}}
  end

  defp apply_pattern(i2c, rgb, :breathe, period_ms) do
    # PCA9685 채널 0-2에 PWM 값 설정 (사인파 호흡 애니메이션)
    Task.start(fn ->
      for t <- 0..100 do
        intensity = :math.sin(:math.pi() * t / 100) |> abs()
        set_rgb_pwm(i2c, rgb, intensity)
        Process.sleep(div(period_ms, 100))
      end
    end)
  end

  defp apply_pattern(i2c, rgb, :blink_fast, interval_ms) do
    Task.start(fn ->
      Stream.repeatedly(fn -> nil end)
      |> Enum.each(fn _ ->
        set_rgb_pwm(i2c, rgb, 1.0)
        Process.sleep(interval_ms)
        set_rgb_pwm(i2c, rgb, 0.0)
        Process.sleep(interval_ms)
      end)
    end)
  end

  defp set_rgb_pwm(i2c, rgb, intensity) do
    r = trunc(((rgb >>> 16) &&& 0xFF) * intensity)
    g = trunc(((rgb >>> 8) &&& 0xFF) * intensity)
    b = trunc((rgb &&& 0xFF) * intensity)
    # PCA9685 채널 0=R, 1=G, 2=B
    Circuits.I2C.write(i2c, 0x40, <<0x06, 0, 0, r, 0>>)
    Circuits.I2C.write(i2c, 0x40, <<0x0A, 0, 0, g, 0>>)
    Circuits.I2C.write(i2c, 0x40, <<0x0E, 0, 0, b, 0>>)
  end
end
```

---

## §9. Wake Word 엔진 (ADR-HW-02)

### 9.1 openWakeWord 통합

```
모델: openWakeWord 커스텀 학습
  - 한국어 Wake Word: "오미야" (4음절, 자음 명확)
  - 학습 데이터: 양성 샘플 500+ (다양한 어르신 음성) + 부정 샘플 10,000+
  - ONNX 내보내기 → Jetson NPU 배포

성능 목표:
  - 지연: < 100ms (NPU 추론)
  - FAR: < 0.5회/시간 (오감지)
  - FRR: < 5% (미감지)

대안 (정확도 우선):
  - Picovoice Porcupine: 상업 라이선스, 더 높은 정확도
  - 전환 조건: FAR > 1회/시간 또는 FRR > 10%
```

### 9.2 Wake Word 학습 파이프라인

```bash
# 학습 환경 (개발 PC)
pip install openwakeword
python scripts/train_wake_word.py \
  --positive-dir data/wake_word/positive/ \
  --negative-dir data/wake_word/negative/ \
  --model-name omi_ya \
  --output priv/models/wake_word.onnx
```

---

## §10. 엣지 ASR 파이프라인

### 10.1 Whisper distil-large-v3-KO 통합

```
모델: openai/whisper-large-v3 → distil-whisper/distil-large-v3
  한국어 fine-tune: AI Hub 방언 데이터셋 + 어르신 음성 수집분

ONNX 변환:
  optimum-cli export onnx \
    --model openai/whisper-large-v3 \
    --task automatic-speech-recognition \
    --opset 17 \
    priv/models/whisper_distil_ko_onnx/

Jetson CUDA 최적화:
  - TensorRT 엔진 변환 (FP16)
  - 예상 RTF: ~0.07 (10초 발화 → 0.7초 처리)

성능 검증 (HW-02):
  # Jetson에서 실행
  python scripts/benchmark_asr.py \
    --audio tests/fixtures/10sec_dialect_gyeongsang.wav \
    --model priv/models/whisper_distil_ko_onnx/ \
    --device cuda
  # 목표: RTF ≤ 0.10
```

### 10.2 방언 fine-tune 데이터셋

| 데이터셋 | 방언 | 라이선스 | 시간 |
|---|---|---|---|
| AI Hub 한국어 방언 데이터셋 | 경상·전라·충청·제주 | CC BY | 약 3,000시간 |
| AI Hub 노인 음성 데이터 | 표준어 (고령자) | CC BY | 약 500시간 |
| 자체 수집 (리빙랩) | 4방언 + 표준 | 내부 | 프로토타입 단계 ~10시간 |

---

## §11. 오미↔클라우드 통신

### 11.1 연결 상태 머신 (OfflineQueue 포함)

```
                    ┌─────────────────────────────┐
[시스템 부팅]        │        DISCONNECTED          │
       │            │  - OfflineQueue 적재          │
       ▼            │  - 5초 간격 재연결 시도        │
[WiFi 연결 완료]    └──────────────┬──────────────┘
       │                           │ connect()
       ▼                           ▼
  [CONNECTING] ──성공──► [CONNECTED]
       │                │  - 정상 send_utterance
       │ 실패            │  - OfflineQueue 플러시
       │                │  - 하트비트 30초
       ▼                │
  [재연결 대기]  ◄─────── disconnect 감지
  (지수 백오프)
```

### 11.2 OfflineQueue 정책

```elixir
# lib/daon_omi/network/offline_queue.ex

defmodule DaonOmi.Network.OfflineQueue do
  use GenServer

  @max_size 50
  @ttl_hours 24

  # 큐 저장: ETS (메모리) + DETS (디스크 영속)
  # 재부팅 후에도 미전송 발화 복원

  def enqueue(text) do
    entry = %{text: text, timestamp: DateTime.utc_now(), id: gen_id()}
    GenServer.cast(__MODULE__, {:enqueue, entry})
  end

  def flush_on_reconnect do
    GenServer.cast(__MODULE__, :flush)
  end

  # handle_cast :flush → 큐 FIFO 순서로 WebSocketClient.send_utterance/1 호출
  # TTL 초과 항목 자동 폐기
  # 큐 MAX_SIZE 초과 시 가장 오래된 항목 폐기
end
```

---

## §12. LED 감정 피드백

### 12.1 5색 LED 패턴 테이블

| 상태 | 색 | HEX | 패턴 | 주기 | 감정 연동 |
|---|---|---|---|---|---|
| 대기 (Idle) | 흰색 | #FFFFFF | 호흡 | 2초 | — |
| Wake 감지 | 하늘색 | #87CEEB | 빠른 점멸 | 0.2초 | — |
| 처리 중 | 노란색 | #FFFF00 | 스피너 | 0.5초 | — |
| 응답 재생 | 초록색 | #00FF00 | 맥박 (TTS 동기) | — | — |
| 슬픔 감지 | 보라색 | #800080 | 느린 페이드인 | 3초 | sadness ≥ 0.6 |
| 위기 L2+ | 빨간색 | #FF0000 | 급속 점멸 | 0.1초 | crisis ≥ 2 |
| 충전 중 | 주황색 | #FF8C00 | 채워짐 (배터리%) | — | — |
| 오류·오프라인 | 빨간색 | #FF0000 | 3회 점멸 후 꺼짐 | — | disconnected |

### 12.2 하드웨어 연결

```
SBC I2C → PCA9685 (PWM 드라이버, 0x40)
  채널 0: LED_EYE_R (오른쪽 눈 R)
  채널 1: LED_EYE_G (오른쪽 눈 G)
  채널 2: LED_EYE_B (오른쪽 눈 B)
  채널 3: LED_EYE_L_R (왼쪽 눈 R)
  채널 4: LED_EYE_L_G (왼쪽 눈 G)
  채널 5: LED_EYE_L_B (왼쪽 눈 B)
  채널 6: LED_EAR_R_R (오른쪽 귀 R)
  ...
  채널 11: LED_EAR_L_B (왼쪽 귀 B)
```

---

## §13. 로컬 개발 환경

### 13.1 Nerves 개발 환경 셋업

```bash
# 1. Nerves 환경 설치 (macOS/Linux)
mix local.hex && mix local.rebar
mix archive.install hex nerves_bootstrap

# 2. 타깃 의존성 (Jetson은 커스텀 패키지 필요)
#    Phase 1 개발: Raspberry Pi 5 사용
export MIX_TARGET=rpi5

# 3. 의존성 설치 + 펌웨어 빌드
cd daon_omi && mix deps.get && mix firmware

# 4. SD 카드 굽기 (rpi5 microSD)
mix burn

# 5. 실행 확인
ssh nerves.local
iex> DaonOmi.Audio.ASRPipeline.transcribe_test()

# Jetson 타깃 (커스텀 패키지 구성 후)
export MIX_TARGET=jetson_orin_nano
mix firmware && mix upload
```

### 13.2 IEx 디버그 명령

```elixir
# 음성 파이프라인 테스트
iex> DaonOmi.Audio.WakeWordDetector.test_with_file("test/fixtures/omi_ya.wav")

# WebSocket 연결 상태
iex> :sys.get_state(DaonOmi.Network.WebSocketClient)

# LED 패턴 수동 테스트
iex> DaonOmi.UI.LEDController.set_pattern(:crisis)
iex> Process.sleep(3000)
iex> DaonOmi.UI.LEDController.set_pattern(:idle)

# 센서 읽기
iex> DaonOmi.Sensor.EnvironmentSensor.read_all()
# → %{temp: 23.5, humidity: 45.2, lux: 120}

# OfflineQueue 상태
iex> :sys.get_state(DaonOmi.Network.OfflineQueue)
```

### 13.3 OTA 업데이트 절차

```bash
# 클라우드에서 새 펌웨어 빌드 → GitHub Releases에 업로드
# 디바이스는 24시간마다 업데이트 확인 (OTAClient GenServer)

# 수동 OTA (개발 중)
mix upload nerves.local   # SSH를 통한 직접 업로드
```

---

## §14. BOM & 원가 분석

### 14.1 Phase 1 프로토타입 BOM

| 분류 | 부품명 | 부품 번호 / 모델 | 수량 | 단가(USD) | 소계 |
|---|---|---|---|---|---|
| **SBC** | Jetson Orin Nano 8GB DevKit | 945-13766-0050-000 | 1 | $149 | $149 |
| **마이크** | Knowles SPH0645LM4H-B MEMS | SPH0645LM4H-B | 4 | $3 | $12 |
| **마이크 DSP** | ReSpeaker 4-mic Array for Pi | (제조사 기성품) | 1 | $18 | $18 |
| **스피커** | Visaton FRS 7 (3W, 4Ω) | 2040 | 1 | $8 | $8 |
| **앰프** | MAX98357AEWL (I2S 클래스D) | MAX98357AEWL+T | 1 | $3 | $3 |
| **WiFi/BT** | Intel AX210 M.2 | AX210.NGWG | 1 | $20 | $20 |
| **LTE (옵션)** | SIMCOM SIM7080G EVK | — | 1 | $25 | ($25) |
| **배터리** | Li-Po 7,500mAh 3.7V | (범용) | 1 | $18 | $18 |
| **BMS** | DW01A + FS8205A | — | 1 | $3 | $3 |
| **어댑터** | 12V/3A DC 어댑터 | — | 1 | $8 | $8 |
| **센서** | BH1750 + SHT31 + ICM-42688 | (개별 모듈) | 각 1 | $4 | $12 |
| **터치** | FT6236 정전식 터치 모듈 | — | 1 | $5 | $5 |
| **LED 드라이버** | PCA9685 PWM 모듈 | — | 1 | $4 | $4 |
| **RGB LED** | WS2812B (눈·귀 각 2개) | — | 4 | $0.5 | $2 |
| **버튼** | 물리 버튼 3개 + 캡 | — | 3 | $0.5 | $1.5 |
| **PCB** | 커스텀 인터페이스 PCB (OSH Park) | — | 1 | $20 | $20 |
| **케이스** | ABS FDM 3D프린팅 (프로토타입) | — | 1 | $15 | $15 |
| **봉제 외피** | 커스텀 봉제 (시제품) | — | 1 | $35 | $35 |
| **기타** | 커넥터·케이블·소모품 | — | — | $15 | $15 |
| **합계 (LTE 제외)** | | | | | **~$348** |
| **합계 (LTE 포함)** | | | | | **~$373** |

> **참고:** Jetson DevKit $149는 개발킷 가격. 양산 시 Jetson SoM + 커스텀 캐리어 보드 또는 RK3588 SoM으로 전환하면 BOM $150 이하 달성 가능.

### 14.2 양산 전환 시 BOM 목표

| 변경 항목 | 프로토타입 | 양산 |
|---|---|---|
| SBC | Jetson DevKit $149 | RK3588 SoM + 커스텀 PCB $60 |
| 봉제 외피 | 수작업 시제품 $35 | ODM 대량 생산 $8~12 |
| 케이스 | FDM 3D 프린팅 $15 | ABS 사출 (금형비 별도) $5 |
| **BOM 합계** | **~$348** | **~$130~150** |

---

## §15. 인증 요건 로드맵

### 15.1 인증 일정

```
2026-06      2026-07      2026-08      2026-09
   │            │            │            │
   ├─ 프로토타입 완성
   ├─────────── KC EMC 시험 시작 (6~8주)
   │            ├─────────── KC 전기안전 병행
   │            │            ├─ KC 인증 취득 목표
   │            │            │
   │            │            ├─ 파일럿 시작 (인증 면제 프로토타입 협의)
   │            │            │
   └────────────┴────────────┴──────────────────► 노인복지용구 인증 준비 (12~24주)
```

### 15.2 KC 인증 준비 체크리스트

| 항목 | 내용 | 담당 |
|---|---|---|
| KC EMC | 전자파 적합성 (KN 32/35) | 시험기관: KTC |
| KC 전기안전 | 어댑터·배터리 포함 (KS C 8305) | 시험기관: KR |
| IP54 시험 | IEC 60529 방수·방진 | 내부 사전 검증 후 외부 |
| ISMS-P | 개인정보·정보보안 인증 | Phase 2 이후 별도 추진 |

---

## §16. 테스트 & 검증 계획

### 16.1 성공 기준별 테스트

| ID | 기준 | 테스트 방법 | 합격 기준 |
|---|---|---|---|
| HW-01 | Wake Word 지연 ≤ 500ms | 오디오 입력 타임스탬프 vs GenServer 이벤트 | 10회 평균 ≤ 500ms |
| HW-02 | ASR RTF ≤ 0.1 | 10초 발화 파일 → 처리 시간 측정 | RTF ≤ 0.10 (5회 평균) |
| HW-03 | SNR ≥ 20dB (TV 소음) | IEC 60268-1 준용, 65dB TV 재생 환경 | SNR ≥ 20dB |
| HW-04 | 배터리 ≥ 2시간 | 충전 없이 대화 루프 (30초 간격) 실행 | 2시간 후 정상 동작 |
| HW-05 | 낙하 내구성 70cm | 3회 낙하 후 ASR 파이프라인 정상 | 3/3 정상 동작 |
| HW-06 | NervesOS 개발 환경 ≤ 1시간 | 클린 환경 → mix firmware && mix burn | 1시간 이내 완료 |
| HW-07 | OTA 성공률 ≥ 99% | 10회 반복 OTA 업데이트 | 10/10 성공 |
| HW-08 | LED 8패턴 정상 동작 | 각 패턴 수동 트리거 + 시각 확인 | 8/8 패턴 정상 |

### 16.2 통합 테스트 시나리오

```
시나리오 1: 기본 대화 루프
  1. 전원 켜기 → LED idle 확인
  2. "오미야" 호출 → LED listening 전환 ≤ 500ms
  3. 발화 → ASR 결과 → WebSocket 전송
  4. 클라우드 응답 → TTS 재생 → LED speaking
  5. 세션 정상 완료

시나리오 2: 오프라인 복구
  1. WiFi 차단 → LED error
  2. 발화 → OfflineQueue 적재 확인
  3. WiFi 복구 → 자동 재연결 → 큐 플러시
  4. 발화 클라우드 수신 확인

시나리오 3: 위기 감지
  1. 클라우드에서 crisis_level=2 주입
  2. LED crisis (빨간 급속 점멸) 확인
  3. 비상호출 버튼 활성화 확인
```

---

## §17. Implementation Guide (Session Guide)

### 17.1 Module Map

| 모듈 | 섹션 | 파일 수 | 예상 소요 |
|---|---|---|---|
| M-01 | §1~2 시스템 구조 + 폼팩터 | 구조 문서 | 1일 |
| M-02 | §3 SBC 선정 ADR + §4 음성 회로 | 3 Elixir 파일 | 1일 |
| M-03 | §5 전원·연결성 + §6 센서 | 2 Elixir 파일 | 1일 |
| M-04 | §7 NervesOS OS 구조 + mix.exs | mix.exs + config/ | 2일 |
| M-05 | §8 펌웨어 GenServer 전체 | 8 Elixir 파일 | 2일 |
| M-06 | §9~10 Wake Word + ASR | 학습 스크립트 + 통합 | 2일 |
| M-07 | §11~12 클라우드 통신 + LED | 2 Elixir 파일 | 1일 |
| M-08 | §13~16 개발환경·BOM·인증·테스트 | 문서 + 스크립트 | 1일 |

### 17.2 구현 순서 권장

```
Phase 1 (Week 1): Rpi5 기반 NervesOS 환경 구축 (M-04)
Phase 2 (Week 2): 음성 파이프라인 (Wake Word + ASR) (M-05, M-06)
Phase 3 (Week 3): WebSocket 통신 + LED 제어 (M-07)
Phase 4 (Week 4): 센서 + 전원 + BOM 확정 + Jetson 이식 (M-02, M-03, M-08)
```

---

*문서 버전: v1.0 | 작성일: 2026-05-24 | 다음 단계: `/pdca do daon-hw --scope M-04,M-05` (NervesOS 환경 + 펌웨어 GenServer 구현)*
