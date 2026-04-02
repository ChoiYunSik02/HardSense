# ⬡ HardSense — AI 하드웨어 모니터링 비서

> **캡스톤 디자인 프로젝트** | 2101091 최윤식
> 최초 작성: 2026년 03월 24일 | 최종 수정: 2026년 04월 02일

---

## 1. 프로젝트 개요

### 1.1 한 줄 정의

> 실시간 하드웨어 데이터를 게임 화면 위에 커스텀 OSD로 표시하고,
> Claude AI가 하드웨어 상태를 **이해·분석·최적화**까지 도와주는 지능형 PC 하드웨어 모니터링 비서

### 1.2 개발 환경 하드웨어

| 구분 | 사양 |
|------|------|
| CPU | AMD Ryzen 9 7945HX |
| GPU | NVIDIA RTX 4070 Laptop |
| OS | Windows 11 (노트북) |
| 서버 | Raspberry Pi 4 |

### 1.3 기존 소프트웨어와의 차이점

| 구분 | RivaTuner / HWiNFO | **HardSense** |
|------|-------------------|---------------|
| OSD 표시 | ✅ 고정 레이아웃 | ✅ 완전 커스텀 (순서·폰트·색·그래프) |
| AI 분석 | ❌ | ✅ Claude API 심층 분석 |
| 병목 표시 | ❌ | ✅ CPU/GPU 병목 자동 판단 |
| 쓰로틀링 감지 | ❌ | ✅ 실시간 감지 + 경고 |
| 온도 색상 경고 | ❌ | ✅ 수치별 색상 자동 변환 |
| 배터리 표시 | ❌ | ✅ 잔량·충전상태·남은시간 |
| 모바일 대시보드 | ❌ | ✅ PWA 지원 |
| 분산 서버 구조 | ❌ | ✅ 라즈베리파이 중앙 서버 |

### 1.4 하드웨어 구성

```
[Windows 노트북]  ──────────────►  [Raspberry Pi 4]
 데이터 수집 + OSD 렌더링              중앙 서버 + 웹 대시보드
 (psutil / pynvml / WMI / LHM)        (FastAPI + SQLite + React)
 (PyQt6 OSD / PresentMon)                     │
                               ┌──────────────┼──────────────┐
                               ▼              ▼               ▼
                        데스크탑 브라우저  모바일 PWA    라파 전용 모니터
```

---

## 2. 데이터 수집 방식 확정

### 2.1 항목별 수집 방법

| 항목 | 수집 방법 | 비고 |
|------|-----------|------|
| CPU 온도 | LHM + WMI | `CPU Package` 센서 |
| CPU 전력 (W) | LHM + WMI | AMD 전력 센서 안정적 |
| CPU 클럭 (MHz) | psutil | `cpu_freq().current` |
| CPU 사용량 (%) | psutil | 전체 + 코어별 |
| GPU 온도 | pynvml | RTX 4070 직접 지원 |
| GPU 클럭 (MHz) | pynvml | `NVML_CLOCK_GRAPHICS` |
| GPU 사용량 (%) | pynvml | `getUtilizationRates` |
| GPU VRAM (GB) | pynvml | `getMemoryInfo` |
| GPU 전력 (W) | pynvml | `getPowerUsage` |
| GPU TDP 한계 (W) | pynvml | `getEnforcedPowerLimit` |
| RAM 사용량 (GB) | psutil | `virtual_memory` |
| RAM 클럭 (MHz) | WMI | `ConfiguredClockSpeed` |
| FPS | PresentMon | ETW 기반 실측값 |
| 1% Low FPS | PresentMon | 하위 1% 프레임 |
| 프레임타임 (ms) | PresentMon | 평균 프레임 간격 |
| 배터리 (%) | psutil | `sensors_battery` |
| 배터리 상태 | psutil | 충전 중 / 방전 중 |
| 배터리 남은시간 | psutil | `secsleft` |
| 전원 모드 | WMI | 고성능 / 균형 / 절전 |
| 네트워크 핑 (ms) | icmplib | 8.8.8.8 ICMP |
| 네트워크 업 (Mbps) | psutil | `net_io_counters` |
| 네트워크 다운 (Mbps) | psutil | `net_io_counters` |

### 2.2 수집 주체별 정리

```mermaid
graph TB
    subgraph LHM["LibreHardwareMonitor + WMI"]
        L1["CPU 온도 (Package)"]
        L2["CPU 전력 (W)"]
        L3["RAM 클럭 (MHz)"]
        L4["전원 모드"]
    end

    subgraph PYNVML["pynvml (NVIDIA 공식)"]
        N1["GPU 온도"]
        N2["GPU 클럭 (MHz)"]
        N3["GPU 사용량 (%)"]
        N4["GPU VRAM (GB)"]
        N5["GPU 전력 (W)"]
        N6["GPU TDP 한계 (W)"]
    end

    subgraph PSUTIL["psutil"]
        P1["CPU 클럭 (MHz)"]
        P2["CPU 사용량 (%)"]
        P3["RAM 사용량 (GB)"]
        P4["배터리 상태"]
        P5["네트워크 업·다운"]
    end

    subgraph PM["PresentMon.exe"]
        M1["FPS"]
        M2["1% Low FPS"]
        M3["프레임타임 (ms)"]
    end

    LHM & PYNVML & PSUTIL & PM --> C["hardware_collector.py\n통합 수집"]
    C --> OSD["PyQt6 OSD"]
    C --> SERVER["라즈베리파이 서버"]
```

---

## 3. 시스템 아키텍처

### 3.1 전체 블록도

```mermaid
graph TB
    subgraph PC["🖥️ Windows 노트북 — 데이터 수집 + OSD"]
        LHM["LibreHardwareMonitor.exe\n관리자 권한 백그라운드"]
        PM["PresentMon.exe\nFPS · 1% Low · 프레임타임"]
        HC["hardware_collector.py\n모든 센서 통합"]
        CLIENT["pc_client.py\n수집 + 전송 + OSD 구동"]
        OSD["PyQt6 OSD\n투명 오버레이"]
        SETTINGS["설정 패널\nosd_config.json"]

        LHM -->|"WMI 등록"| HC
        PM -->|"FPS 데이터"| HC
        HC --> CLIENT
        CLIENT --> OSD
        SETTINGS --> OSD
    end

    subgraph RPI["🍓 Raspberry Pi 4 — 중앙 서버"]
        FASTAPI["FastAPI\n:8000"]
        DB["SQLite\n이력 저장"]
        REACT["React PWA"]
        FASTAPI --> DB
        FASTAPI --> REACT
    end

    subgraph VIEWS["📱 클라이언트"]
        D["데스크탑 브라우저"]
        M["모바일 PWA"]
        R["라파 전용 모니터"]
    end

    subgraph AI["☁️ Claude API"]
        CL["claude-sonnet-4-5"]
    end

    CLIENT -->|"HTTP POST 1초"| FASTAPI
    FASTAPI -->|"WebSocket"| D & M & R
    FASTAPI <-->|"AI 분석"| CL
```

### 3.2 데이터 흐름

```mermaid
sequenceDiagram
    participant HC  as hardware_collector
    participant PM  as PresentMon
    participant CLI as pc_client.py
    participant OSD as PyQt6 OSD
    participant RPI as Raspberry Pi
    participant DB  as SQLite
    participant WS  as WebSocket 구독자
    participant AI  as Claude API

    loop 매 1초
        HC  ->> HC  : LHM WMI: CPU 온도·전력
        HC  ->> HC  : pynvml: GPU 전체
        HC  ->> HC  : psutil: CPU·RAM·배터리·네트워크
        PM -->> HC  : FPS·1%Low·프레임타임
        HC  ->> CLI : 통합 데이터 전달
        CLI ->> OSD : OSD 업데이트
        OSD ->> OSD : 게임 화면 위 렌더링
        CLI ->> RPI : POST /api/hardware
        RPI ->> DB  : 이력 저장
        RPI ->> WS  : WebSocket 브로드캐스트
    end

    Note over CLI,AI: AI 분석 버튼 클릭 시
    CLI ->> RPI : POST /api/analyze
    RPI ->> AI  : 스펙 + 현재 데이터
    AI -->> RPI : 분석 결과 Markdown
    RPI ->> WS  : 결과 브로드캐스트
```

---

## 4. OSD 기능 상세

### 4.1 표시 항목 전체 목록

| 항목 | 표시 포맷 | 기본값 | 그래프 |
|------|-----------|--------|--------|
| FPS | `144 fps` | ON | 라인 |
| 1% Low | `1%Low: 98 fps` | ON | 라인 |
| 프레임타임 | `6.9ms` | ON | 라인 |
| CPU 온도 | `72°C` | ON | 라인 |
| CPU 사용량 | `45%` | ON | 라인 |
| CPU 클럭 | `4800MHz` | ON | 라인 |
| CPU 전력 | `95W / 65W TDP` | ON | 라인 |
| GPU 온도 | `68°C` | ON | 라인 |
| GPU 사용량 | `89%` | ON | 라인 |
| GPU 클럭 | `2100MHz` | ON | 라인 |
| GPU 전력 | `85W / 115W TDP` | ON | 라인 |
| VRAM | `8.2 / 8.0GB` | ON | 바 |
| RAM | `14.2 / 32.0GB` | ON | 바 |
| 배터리 | `78% ⚡충전중` | ON | — |
| 배터리 남은시간 | `약 2:14 남음` | ON | — |
| 네트워크 핑 | `12.3ms` | ON | 라인 |
| 네트워크 업 | `↑8 Mbps` | OFF | — |
| 네트워크 다운 | `↓120 Mbps` | OFF | — |
| 현재 시간 | `23:47` | ON | — |
| 세션 타이머 | `1:23:45` | OFF | — |
| 병목 표시 | `병목: GPU` | ON | — |
| 쓰로틀링 감지 | `⚠ GPU 쓰로틀링` | ON | — |

### 4.2 온도·사용량 색상 경고

```mermaid
graph LR
    T1["~ 70°C / ~ 70%"] -->|"#00FF00 초록"| SAFE["정상"]
    T2["70 ~ 85°C / 70 ~ 90%"] -->|"#FFAA00 노랑"| WARN["주의"]
    T3["85°C+ / 90%+"] -->|"#FF0000 빨강 깜빡임"| DANGER["위험"]
```

CPU·GPU 온도, CPU·GPU 사용량, GPU 전력, 배터리 잔량 전부 동일 적용

### 4.3 병목 자동 판단

```
CPU 사용량 >= 90%  AND  GPU 사용량 < 80%  →  병목: CPU
GPU 사용량 >= 90%  AND  CPU 사용량 < 80%  →  병목: GPU
둘 다 >= 80%                               →  병목: CPU+GPU
둘 다 < 80%                                →  표시 없음
```

### 4.4 쓰로틀링 감지

```
GPU 클럭이 최대 대비 20% 이상 급락  →  ⚠ GPU 쓰로틀링
CPU 클럭이 기본 클럭 이하 지속      →  ⚠ CPU 쓰로틀링
GPU 전력이 TDP 한계 도달            →  ⚠ 전력 제한
```

### 4.5 OSD 커스텀 설정

```mermaid
mindmap
  root((OSD\n설정 패널))
    레이아웃
      항목 순서 드래그 변경
      항목별 ON/OFF 토글
      OSD 위치 9분할 프리셋
      좌표 직접 입력
      전체 너비 조정
    스타일
      폰트 선택
      폰트 크기·굵기
      항목별 색상 개별 지정
      배경 투명도·색상
    그래프
      라인 / 바 / 없음 선택
      히스토리 길이 조정
    기타
      OSD 토글 단축키
      게임 감지 자동 표시
      온도 경고 임계값 설정
```

### 4.6 설정 파일 구조 (osd_config.json)

```json
{
  "position": { "x": 10, "y": 10 },
  "width": 220,
  "font": "Consolas",
  "font_size": 14,
  "font_bold": false,
  "bg_color": "#000000",
  "bg_opacity": 0.6,
  "toggle_hotkey": "F12",
  "temp_warn": 70,
  "temp_danger": 85,
  "items": [
    { "id": "fps",        "order": 0,  "show": true,  "graph": "line", "color": "#00FF00" },
    { "id": "fps_low",    "order": 1,  "show": true,  "graph": "line", "color": "#00FF00" },
    { "id": "frametime",  "order": 2,  "show": true,  "graph": "line", "color": "#AAFFAA" },
    { "id": "gpu_temp",   "order": 3,  "show": true,  "graph": "line", "color": "#FF6600" },
    { "id": "gpu_usage",  "order": 4,  "show": true,  "graph": "line", "color": "#FF6600" },
    { "id": "gpu_clock",  "order": 5,  "show": true,  "graph": "line", "color": "#FF9900" },
    { "id": "gpu_power",  "order": 6,  "show": true,  "graph": "line", "color": "#FFCC00" },
    { "id": "vram",       "order": 7,  "show": true,  "graph": "bar",  "color": "#FF6600" },
    { "id": "cpu_temp",   "order": 8,  "show": true,  "graph": "line", "color": "#00AAFF" },
    { "id": "cpu_usage",  "order": 9,  "show": true,  "graph": "line", "color": "#00AAFF" },
    { "id": "cpu_clock",  "order": 10, "show": true,  "graph": "line", "color": "#66CCFF" },
    { "id": "cpu_power",  "order": 11, "show": true,  "graph": "line", "color": "#AADDFF" },
    { "id": "ram",        "order": 12, "show": true,  "graph": "bar",  "color": "#AA88FF" },
    { "id": "battery",    "order": 13, "show": true,  "graph": "none", "color": "#88FF88" },
    { "id": "bat_time",   "order": 14, "show": true,  "graph": "none", "color": "#88FF88" },
    { "id": "net_ping",   "order": 15, "show": true,  "graph": "line", "color": "#FFFFFF" },
    { "id": "net_up",     "order": 16, "show": false, "graph": "none", "color": "#AAAAAA" },
    { "id": "net_down",   "order": 17, "show": false, "graph": "none", "color": "#AAAAAA" },
    { "id": "time",       "order": 18, "show": true,  "graph": "none", "color": "#CCCCCC" },
    { "id": "session",    "order": 19, "show": false, "graph": "none", "color": "#CCCCCC" },
    { "id": "bottleneck", "order": 20, "show": true,  "graph": "none", "color": "#FFFF00" },
    { "id": "throttle",   "order": 21, "show": true,  "graph": "none", "color": "#FF0000" }
  ]
}
```

---

## 5. AI 분석 기능

### 5.1 분석 항목

```mermaid
graph LR
    A["AI 분석 버튼"] --> B["스펙 수집\nRyzen 9 7945HX\nRTX 4070 Laptop"]
    B --> C["최근 5분 평균 데이터"]
    C --> D["Claude API"]

    D --> E1["벤치마크 비교\n동급 모델 성능 위치"]
    D --> E2["게임 성능 예측\n주요 타이틀 FPS"]
    D --> E3["오버클럭·XMP 추천"]
    D --> E4["언더볼팅 추천\n발열·배터리 최적화"]
    D --> E5["병목 분석"]
    D --> E6["이상 원인 분석"]

    E1 & E2 & E3 & E4 & E5 & E6 --> F["AI 분석 패널\nMarkdown 렌더링"]
```

### 5.2 이상 감지 자동 분석

```mermaid
graph TD
    M["모니터링 데이터"] --> CHECK{임계값 초과?}
    CHECK -->|"✅ 정상"| DISP["정상 표시"]
    CHECK -->|"⚠️ 이상"| TYPE{유형}
    TYPE --> T1["CPU 온도 > 85°C"]
    TYPE --> T2["GPU 온도 > 85°C"]
    TYPE --> T3["쓰로틀링 감지"]
    TYPE --> T4["RAM > 90%"]
    T1 & T2 & T3 & T4 --> ALERT["OSD 경고 + 대시보드 배너"]
    T1 & T2 & T3 & T4 --> AI["Claude API 자동 분석"]
    AI --> RESULT["원인 + 해결책 출력"]
```

---

## 6. 웹 대시보드

```mermaid
graph TB
    subgraph MAIN["메인 대시보드 — /"]
        HEADER["헤더: 연결 상태 · 시각 · AI 분석 버튼"]
        subgraph CARDS["카드 그리드"]
            FC["FPS 카드\nFPS · 1%Low · 프레임타임"]
            CC["CPU 카드\n온도·사용량·클럭·전력"]
            GC["GPU 카드\n온도·사용량·클럭·전력·VRAM"]
            RC["RAM 카드\n사용량 · XMP 클럭"]
            BC["배터리 카드\n잔량·상태·남은시간"]
            NC["네트워크 카드\n핑·업·다운"]
        end
        HISTORY["히스토리 그래프 — 5분 · 30분 · 1시간"]
        AI_PANEL["AI 분석 패널 — Markdown 렌더링"]
    end
    subgraph OBS["OBS 오버레이 — /overlay"]
        OV["투명 배경 최소 UI\nOBS 브라우저 소스 전용"]
    end
```

---

## 7. 기술 스택

```mermaid
graph TB
    subgraph PC_STACK["PC — Windows 노트북"]
        P1["Python 3.11+"]
        P2["psutil — CPU·RAM·배터리·네트워크"]
        P3["pynvml — RTX 4070 전체"]
        P4["wmi — CPU 온도·전력·RAM 클럭"]
        P5["PyQt6 — OSD + 설정 패널"]
        P6["icmplib — 네트워크 핑"]
        P7["requests — HTTP POST"]
        P8["LibreHardwareMonitor.exe"]
        P9["PresentMon.exe — FPS"]
    end

    subgraph RPI_STACK["Raspberry Pi 4 — 서버"]
        R1["Python 3.11+"]
        R2["FastAPI + WebSocket"]
        R3["SQLite"]
        R4["uvicorn"]
        R5["httpx — Claude API"]
    end

    subgraph FE_STACK["프론트엔드 — React PWA"]
        F1["React 18"]
        F2["Chart.js"]
        F3["Tailwind CSS"]
        F4["Vite + manifest.json"]
    end

    PC_STACK -->|"HTTP POST 1초"| RPI_STACK
    RPI_STACK -->|"WebSocket"| FE_STACK
    RPI_STACK <-->|"HTTPS"| CL["Claude API"]
```

---

## 8. API 엔드포인트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `POST` | `/api/hardware` | PC → 라파 데이터 수신 |
| `GET` | `/api/hardware/latest` | 최신 데이터 조회 |
| `GET` | `/api/hardware/history?minutes=30` | 이력 조회 |
| `WebSocket` | `/ws` | 실시간 브로드캐스트 |
| `POST` | `/api/analyze` | Claude AI 분석 요청 |
| `GET` | `/api/system/info` | PC 스펙 정보 |

---

## 9. 디렉토리 구조

```
HardSense/
│
├── pc_client/
│   ├── pc_client.py              # 메인 — 수집·전송·OSD 구동
│   ├── hardware_collector.py     # LHM·pynvml·psutil 통합 수집
│   ├── presentmon_reader.py      # PresentMon FPS 수집
│   ├── osd_overlay.py            # PyQt6 투명 OSD
│   ├── osd_settings.py           # 설정 패널 UI
│   ├── osd_config.json           # OSD 설정 저장
│   ├── PresentMon.exe            # FPS 수집 도구 동봉
│   └── requirements_pc.txt
│
├── server/
│   ├── main.py
│   ├── routers/
│   │   ├── hardware.py
│   │   ├── analyze.py
│   │   └── websocket.py
│   ├── database.py
│   ├── claude_client.py
│   └── requirements_server.txt
│
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── FpsCard.jsx
│   │   │   ├── CpuCard.jsx
│   │   │   ├── GpuCard.jsx
│   │   │   ├── RamCard.jsx
│   │   │   ├── BatteryCard.jsx
│   │   │   ├── NetworkCard.jsx
│   │   │   ├── HistoryChart.jsx
│   │   │   └── AiAnalysisPanel.jsx
│   │   ├── pages/
│   │   │   ├── Dashboard.jsx
│   │   │   └── Overlay.jsx
│   │   ├── hooks/
│   │   │   └── useWebSocket.js
│   │   └── App.jsx
│   ├── public/manifest.json
│   └── package.json
│
└── README.md
```

---

## 10. 개발 단계별 계획

```mermaid
gantt
    title HardSense 개발 계획
    dateFormat  YYYY-MM-DD

    section Phase 1 — 데이터 수집
    hardware_collector.py         :a1, 2026-04-03, 3d
    presentmon_reader.py          :a2, after a1, 2d

    section Phase 2 — 라파 서버
    FastAPI + WebSocket           :b1, 2026-04-03, 3d
    SQLite 이력 저장              :b2, after b1, 2d

    section Phase 3 — OSD
    PyQt6 투명 오버레이           :c1, 2026-04-10, 3d
    항목 렌더링 + 색상 경고       :c2, after c1, 2d
    병목·쓰로틀링 감지            :c3, after c2, 2d
    설정 패널 UI                  :c4, after c3, 3d

    section Phase 4 — 프론트엔드
    React 셋업 + 카드 컴포넌트    :d1, 2026-04-20, 4d
    반응형 대시보드               :d2, after d1, 3d
    OBS 오버레이 페이지           :d3, after d2, 1d

    section Phase 5 — AI 분석
    Claude API 연동               :e1, 2026-04-28, 2d
    이상 감지 자동 분석           :e2, after e1, 2d
    프롬프트 최적화               :e3, after e2, 2d

    section Phase 6 — 마무리
    PWA + 통합 테스트             :f1, 2026-05-04, 4d
    발표 자료·문서화              :f2, after f1, 5d
```

---

## 11. OSD 동작 조건

| 게임 실행 모드 | OSD 동작 | 비고 |
|---------------|---------|------|
| 보더리스 창모드 | ✅ 완벽 동작 | 권장 설정 |
| 창모드 | ✅ 완벽 동작 | |
| 풀스크린 독점 | ❌ 미지원 | 보더리스 전환 권장 |

> Windows 10/11 풀스크린 최적화로 보더리스와 성능 차이 사실상 없음

---

## 12. 기대 결과물

| 결과물 | 설명 |
|--------|------|
| 커스텀 OSD | 게임 화면 위 투명 오버레이, 완전 커스텀 |
| OSD 설정 패널 | 항목·순서·폰트·색·그래프 실시간 조정 |
| 라즈베리파이 서버 | FastAPI REST + WebSocket 중앙 서버 |
| 웹 대시보드 | 실시간 그래프 + AI 분석 패널 |
| 모바일 PWA | 스마트폰 홈 화면 설치 지원 |
| OBS 오버레이 | 브라우저 소스용 최소 UI |
| AI 분석 리포트 | Claude 기반 하드웨어 종합 분석 |

---

*이 문서는 캡스톤 디자인 프로젝트 설계 문서입니다.*
