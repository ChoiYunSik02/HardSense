# HardSense
AI-Powered Hardware Monitoring Assistant

# ⬡ CoreMind — AI 하드웨어 모니터링 비서
### 프로젝트 기획서 (캡스톤 디자인)

---

## 1. 프로젝트 개요

### 1.1 프로젝트 명

**CoreMind** — AI-Powered Hardware Monitoring Assistant

### 1.2 한 줄 정의

> 수치를 보여주는 것을 넘어, AI가 하드웨어 상태를 **이해하고, 예측하고, 최적화**까지 도와주는 지능형 PC 하드웨어 모니터링 비서

### 1.3 기존 소프트웨어와의 차이점

| 구분 | RivaTuner / HWiNFO | **CoreMind** |
|------|-------------------|--------------|
| 데이터 표시 | ✅ 수치만 표시 | ✅ 수치 + 의미 해석 |
| AI 분석 | ❌ | ✅ 2단계 AI 구조 |
| 이상 원인 설명 | ❌ | ✅ 자동 원인 추론 |
| 해결책 제시 | ❌ | ✅ 단계별 안내 |
| 벤치마크 예측 | ❌ | ✅ ML 기반 점수 예측 |
| 언더볼팅 가이드 | ❌ | ✅ 실사용 데이터 기반 자동 계산 |
| 상시 백그라운드 실행 | ❌ (게임 시만) | ✅ 항상 실행 |

---

## 2. 프로젝트 목적

### 2.1 배경

현재 시중에 출시된 하드웨어 모니터링 소프트웨어(RivaTuner, HWiNFO, MSI Afterburner 등)는 단순히 수치를 시각화하는 역할에 그친다. 사용자는 CPU가 92°C를 표시해도 이것이 위험한지, 왜 높아졌는지, 어떻게 해야 하는지 스스로 판단해야 한다. AI가 일상에 깊숙이 자리 잡은 현재, 이러한 판단을 AI가 대신 수행해주는 도구가 존재하지 않는다.

### 2.2 목적

AI를 하드웨어 모니터링에 접목하여:

- 일반 사용자도 자신의 PC 상태를 **직관적으로 이해**할 수 있도록 한다.
- 이상 징후를 **자동으로 감지하고 원인을 설명**해준다.
- 며칠간의 데이터를 분석해 **언더볼팅 최적값을 제안**함으로써, 초보자도 안전하게 PC를 튜닝할 수 있게 한다.
- **벤치마크 점수를 예측**하여 실제 성능이 기대치에 못 미칠 경우 원인을 분석해준다.

---

## 3. 프로젝트 목표

### 3.1 핵심 목표 (Must Have)

- [x] CPU / GPU / RAM / 디스크 / 네트워크 실시간 모니터링 대시보드 구현
- [x] 로컬 AI (Ollama) 기반 1차 이상 감지 및 자동 경고
- [x] Claude API 기반 심층 원인 분석 및 해결책 제시
- [x] 이상 감지 시 Windows 토스트 알림 발송
- [x] SQLite 기반 시계열 데이터 로깅 (24시간)

### 3.2 확장 목표 (Should Have)

- [ ] ML 회귀 모델 기반 벤치마크 점수 예측 (Cinebench, 3DMark)
- [ ] 실제 벤치마크 결과와 예측값 비교 분석
- [ ] 며칠치 게임 세션 데이터 기반 언더볼팅 최적값 자동 계산
- [ ] 언더볼팅 단계별 가이드 (초보자용)

### 3.3 선택 목표 (Nice to Have)

- [ ] 음성 알림 기능
- [ ] 모바일 접속 대시보드 지원
- [ ] 다중 PC 모니터링 지원

---

## 4. 시스템 블록도

### 4.1 전체 아키텍처

```mermaid
graph TD
    subgraph HARDWARE["🖥️ 하드웨어 레이어"]
        CPU["CPU\n사용률 · 온도 · 클럭 · 전압 · 전력"]
        GPU["GPU\n사용률 · 온도 · VRAM · 팬RPM · TGP"]
        RAM["RAM\n사용량 · 클럭"]
        DISK["디스크\n읽기/쓰기 속도 · 용량"]
        NET["네트워크\n업로드/다운로드"]
    end

    subgraph COLLECT["📡 데이터 수집 레이어 (Python)"]
        PSUTIL["psutil\nCPU · RAM · 디스크 · 네트워크"]
        PYNVML["pynvml\nGPU (NVIDIA)"]
        LHM["LibreHardwareMonitor\n전압 · 전력 · 팬RPM 상세값"]
    end

    subgraph STORAGE["🗄️ 저장 레이어"]
        SQLITE["SQLite DB\n시계열 데이터 누적 저장\n24시간 자동 보관"]
    end

    subgraph AI["🧠 AI 분석 레이어"]
        OLLAMA["Ollama (로컬 AI)\nqwen3:8b\n1차 이상 감지 · 정상/비정상 분류\n간단한 Q&A · 상시 실행"]
        CLAUDE["Claude API (클라우드)\n이상 감지 시에만 호출\n심층 원인 분석\n언더볼팅 값 계산\n며칠치 데이터 종합 분석"]
        ML["ML 회귀 모델\nscikit-learn\n벤치마크 점수 예측\nCinebench · 3DMark"]
    end

    subgraph NOTIFY["🔔 알림 시스템"]
        TOAST["Windows 토스트 알림\n온도 임계값 초과\n팬 정지 감지\n성능 이상 감지"]
    end

    subgraph UI["🎨 대시보드 UI"]
        DASH["웹 대시보드\nHTML + CSS + JS\nApexCharts 실시간 그래프"]
        CHAT["AI 채팅창\n자연어 질문/응답"]
        ALERT["이상 감지 알림 패널\n원인 분석 + 해결책 표시"]
        BENCH["벤치마크 예측 패널\n예측값 vs 실제값 비교"]
        UNDER["언더볼팅 가이드 패널\n최적값 + 단계별 안내"]
    end

    CPU & GPU & RAM & DISK & NET --> PSUTIL & PYNVML & LHM
    PSUTIL & PYNVML & LHM --> SQLITE
    PSUTIL & PYNVML & LHM --> OLLAMA
    SQLITE --> CLAUDE
    SQLITE --> ML
    OLLAMA -->|"이상 감지 시"| CLAUDE
    OLLAMA --> TOAST
    CLAUDE --> ALERT
    CLAUDE --> UNDER
    ML --> BENCH
    PSUTIL & PYNVML & LHM --> DASH
    DASH --- CHAT & ALERT & BENCH & UNDER
```

---

### 4.2 AI 2단계 분석 흐름

```mermaid
flowchart TD
    SENSOR["센서 데이터 수집\n1초마다"] --> OLLAMA

    subgraph STAGE1["1단계: Ollama (로컬 · 상시 실행)"]
        OLLAMA["Ollama AI\nqwen3:8b"] --> CHECK{{"정상\n범위?<br/>"}}
        CHECK -->|"✅ 정상"| DISPLAY["대시보드 업데이트"]
        CHECK -->|"⚠️ 이상 감지"| CLASSIFY["이상 분류\n온도 · 팬 · 사용률 · 누수"]
    end

    CLASSIFY --> TOAST["🔔 Windows 알림 발송"]
    CLASSIFY --> LOG["이상 로그 저장\nSQLite"]

    subgraph STAGE2["2단계: Claude API (이상 감지 시에만)"]
        LOG --> CLAUDE["Claude API\n심층 분석 요청"]
        CLAUDE --> CAUSE["원인 추론\n먼지 · 팬 불량 · 소프트웨어 충돌 등"]
        CLAUDE --> SOLUTION["해결책 제시\n단계별 안내"]
        CLAUDE --> UNDER["언더볼팅 계산\n(며칠치 게임 데이터 기반)"]
    end

    CAUSE & SOLUTION & UNDER --> UI["대시보드 AI 패널 표시"]
```

---

### 4.3 벤치마크 예측 흐름

```mermaid
flowchart LR
    subgraph INPUT["입력"]
        HW["하드웨어 사양 자동 인식\nCPU 모델 · GPU 모델\nRAM 클럭 · 쿨링 환경"]
    end

    subgraph ML["ML 예측 모델 (scikit-learn)"]
        REG["회귀 모델\n학습 데이터:\n시중 벤치마크 DB\nCPU/GPU 조합별 실측값"]
        PRED["예측 점수 산출\nCinebench R24\n3DMark Time Spy"]
    end

    subgraph COMPARE["비교 분석"]
        REAL["사용자 실제 점수 입력"]
        DIFF{{"차이\n분석"}}
        OK["✅ 정상 범위\n하드웨어 최적 상태"]
        WARN["⚠️ 예측보다 낮음\nClaude API 원인 분석 요청"]
    end

    HW --> REG --> PRED
    PRED --> DIFF
    REAL --> DIFF
    DIFF -->|"오차 10% 이내"| OK
    DIFF -->|"오차 10% 초과"| WARN
    WARN --> CAUSE["원인: 스로틀링 · 백그라운드 충돌\n온도 제한 설정 오류 등"]
```

---

### 4.4 언더볼팅 가이드 흐름

```mermaid
flowchart TD
    GAME["게임 세션 시작 감지"] --> LOG["세션 데이터 로깅\nCPU 전력 · GPU 전력\n온도 · 클럭 · 프레임타임"]
    LOG --> DAYS["며칠간 데이터 누적\n(최소 3일 권장)"]
    DAYS --> ANALYZE["Claude API 분석\n안정 동작 전압/전력 범위 계산"]
    ANALYZE --> RESULT["최적 언더볼팅 값 제안\nCPU 코어 전압: 0.95V 권장\nGPU TGP: 85W 권장\n예상 온도 감소: ~10°C"]
    RESULT --> GUIDE["단계별 적용 가이드\n초보자용"]
    GUIDE --> B1["STEP 1\nBIOS 진입 방법 안내"]
    GUIDE --> B2["STEP 2\n전압 설정 항목 안내"]
    GUIDE --> B3["STEP 3\n안정성 테스트 방법"]
    GUIDE --> B4["STEP 4\n결과 확인 및 추가 조정"]
```

---

## 5. 개발 일정 (Phase별)

```mermaid
gantt
    title CoreMind 개발 일정
    dateFormat  YYYY-MM-DD
    section Phase 1
    기본 모니터링 대시보드     :done,    p1, 2026-03-24, 7d
    section Phase 2
    SQLite 로그 + Ollama 1차 AI :active,  p2, 2026-03-31, 7d
    section Phase 3
    Windows 알림 + 이상감지 고도화 :        p3, 2026-04-07, 7d
    section Phase 4
    벤치마크 ML 예측 모델      :        p4, 2026-04-14, 10d
    section Phase 5
    Claude API 심층 분석 연동  :        p5, 2026-04-24, 7d
    section Phase 6
    언더볼팅 가이드 기능       :        p6, 2026-05-01, 7d
    section Phase 7
    UI 완성도 + 발표 준비      :        p7, 2026-05-08, 14d
```

---

## 6. 기술 스택 요약

```mermaid
graph LR
    subgraph BE["Backend"]
        PY["Python 3.11+"]
        FA["FastAPI\n+ WebSocket"]
        PS["psutil"]
        NV["pynvml"]
        SK["scikit-learn"]
        SQ["SQLite"]
        OL["Ollama API"]
        CA["Claude API"]
    end

    subgraph FE["Frontend"]
        HTML["HTML + CSS + JS"]
        AP["ApexCharts\n실시간 그래프"]
    end

    subgraph NOTIF["Notification"]
        WIN["plyer\nWindows 토스트"]
    end

    PY --> FA --> HTML
    FA --> AP
    PS & NV --> FA
    SK & SQ --> FA
    OL & CA --> FA
    FA --> WIN
```

---

## 7. 기대 효과

- PC를 잘 모르는 일반 사용자도 **AI의 설명을 통해 자신의 PC 상태를 쉽게 파악**
- 먼지 쌓임, 팬 불량, 소프트웨어 충돌 등 **발열 원인을 조기에 감지**하여 하드웨어 수명 연장
- 언더볼팅 초보자도 **안전하게 전압/전력 최적화**를 시도할 수 있는 가이드 제공
- 벤치마크 측정 없이 **AI로 예측 점수를 확인**하고, 실제 성능 저하 원인을 파악

---

*이 문서는 캡스톤 디자인 프로젝트 기획 단계 문서입니다.*
