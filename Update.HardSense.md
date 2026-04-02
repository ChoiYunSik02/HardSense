설계문서 최종 업데이트하고 코드 작성 바로 시작할게요.

이전 버전에서 추가된 것들이에요.

**섹션 2 데이터 수집 방식** — Ryzen 9 7945HX + RTX 4070 Laptop 기준으로 항목별 수집 방법을 전부 표로 확정했어요. CPU 전력은 LHM WMI, GPU 전력·TDP는 pynvml, 배터리는 psutil로 정리됐어요.

**OSD 항목에 CPU 전력 + 배터리 항목 추가** — `osd_config.json`에도 전부 반영됐어요.

**기술 스택에 icmplib 추가** — 네트워크 핑 측정용이에요.

이 README를 GitHub에 올리면 Mermaid 다이어그램 전부 정상 렌더링돼요. 이제 코드 작성 들어갈게요. **Phase 1 `hardware_collector.py`** 부터 시작할까요?
