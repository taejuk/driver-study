# 2026년 11월 NAPI 기반 Linux 디바이스 드라이버 설계 학습 계획

> 기간: 2026-11-01 ~ 2026-11-30  
> 최우선 진로 목표: **Linux Device Driver Engineer (PCIe / DMA / Interrupt 중심)**  
> 11월의 최종 결과물: **NAPI 코드 분석 리포트 1개 + 코드 흐름도 + 기존 실험 결과 해석 + 12월 XDMA 분석 질문 목록**

## 0. 계획의 범위와 원칙

- **11월:** NAPI를 통해 IRQ, Deferred Work, DMA Ring, Completion, 동시성, 성능의 *드라이버 설계 원칙*을 공부한다.
- **12월:** 이미 공부한 Linux DMAEngine 기반 XDMA와 AMD/Xilinx `libxdma.c`를 구분하여 XDMA 분석 보고서를 완성한다.
- 11월에는 네트워크 스택 전체를 완독하지 않는다. `mt7921e`/`mt76`의 **RX 경로**를 먼저 완성한다.
- 코드를 읽을 때 항상 **정상 흐름 → 경계 조건 → 동시성/오류 → 검증** 순서로 질문한다.
- 리포트에서 **확인된 코드 사실 / 실험 관찰 / 아직 검증되지 않은 가설**을 구분한다.
- 회사의 XDMA 제출 기한이 앞당겨지면 이 계획보다 회사 업무를 우선한다.

### 분석 환경(기존 관찰 기준)

| 항목 | 값 |
|---|---|
| Linux kernel | Ubuntu `7.0.0-38-generic` (이전에 확인한 버전) |
| NIC | MediaTek MT7922 (`14c3:0616`) |
| PCIe Wi-Fi 드라이버 | `mt7921e`, 공통 구현 `mt76` |
| 인터페이스 | `wlp13s0` |
| 이미 사용한 도구 | `trace-cmd`, ftrace, `iperf3`, `ethtool`, `/proc/net/snmp` |

> **소스 버전 유의:** 아래 링크는 메인라인 Linux 코드다. 실제 보고서에서 함수 조건과 레지스터 의미를 확정할 때는 **현재 설치된 Ubuntu 커널에 해당하는 소스**를 우선한다. 버전에 따라 구현이 다를 수 있다.

---

## 1주 차 — 11/01~11/07: Interrupt → NAPI Scheduling

**목표:** 하드웨어 이벤트가 어떻게 NAPI 실행으로 이어지고, 왜 IRQ handler에서 패킷을 모두 처리하지 않는지 설명한다.

### 읽을 코드

- [`drivers/net/wireless/mediatek/mt76/mt7921/pci.c`](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/mediatek/mt76/mt7921/pci.c)
  - `devm_request_irq()`를 통해 등록되는 ISR
  - NAPI callback 등록과 큐 초기화가 어디서 이루어지는지 추적
- [`drivers/net/wireless/mediatek/mt76/mt792x_dma.c`](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/mediatek/mt76/mt792x_dma.c)
  - `mt792x_irq_handler()`
  - `mt792x_irq_tasklet()`
  - `mt792x_rx_poll_complete()`
- [`net/core/dev.c`](https://github.com/torvalds/linux/blob/master/net/core/dev.c)
  - `napi_schedule()`에서 실제 실행으로 이어지는 경로
  - `net_rx_action()`과 `napi_threaded_poll()`의 차이

### 해야 할 일

- [ ] 논문 *Eliminating Receive Livelock in an Interrupt-Driven Kernel*의 문제 정의와 해결책을 1쪽 이내로 요약한다.
- [ ] 인터럽트 상태 확인, ACK, masking의 코드 위치를 표시한다.
- [ ] `tasklet_schedule()`와 `napi_schedule()`의 실행 주체/문맥을 구분한다.
- [ ] `napi_struct`의 등록·활성화·스케줄링 경로를 찾는다.
- [ ] 일반 `NET_RX_SOFTIRQ` 경로와 Threaded NAPI 경로를 구분한다.
- [ ] `mt76`에서 RX MAIN 큐와 MCU 큐가 각각 어떤 NAPI에 연결되는지 기록한다.

### 답할 질문

1. IRQ handler가 RX IRQ를 막는 이유는 무엇인가?
2. Interrupt Status를 지우는 것과 Interrupt Enable을 해제하는 것은 어떻게 다른가?
3. `napi_schedule()` 호출 즉시 드라이버 `poll()`이 실행되는가?
4. NAPI 스레드가 활성화된 경우 일반 softirq 경로와 무엇이 다른가?

**주간 결과물:** `IRQ → tasklet → NAPI schedule → poll` 호출 흐름도 1개와 각 함수의 역할 표.

---

## 2주 차 — 11/08~11/14: Budget, Poll, Repoll

**목표:** NAPI가 CPU를 독점하지 않으면서 처리량을 유지하도록 설계된 원리를 이해한다.

### 읽을 코드

- `net/core/dev.c`
  - `__napi_poll()`
  - `net_rx_action()`
  - `napi_threaded_poll()`
- `include/linux/netdevice.h`
  - `struct napi_struct`
  - `napi_complete_done()`의 계약
- `mt792x_dma.c`
  - `mt792x_poll_rx()`
- `mt76/dma.c`
  - `mt76_dma_rx_poll()`

### 해야 할 일

- [ ] `napi->weight`, `netdev_budget`, `netdev_budget_usecs`의 차이를 정리한다.
- [ ] `work < budget`와 `work == budget` 분기를 코드로 설명한다.
- [ ] Budget을 모두 썼을 때 다시 poll할 수 있도록 하는 메커니즘을 추적한다.
- [ ] Poll이 완료되었다는 의미와 DMA 전송 자체가 완료되었다는 의미를 구분한다.
- [ ] `napi_complete()`/`napi_complete_done()` 후 IRQ를 다시 켜는 경로를 연결한다.
- [ ] 논문의 **packet-count quota**와 NAPI의 **budget**을 공통점/차이점으로 비교한다.

### 답할 질문

1. Budget은 정확히 누구의 작업량을 제한하는가?
2. 왜 `work == budget`이면 보통 NAPI 완료 처리를 하지 않는가?
3. Poll 하나가 빨리 끝나도 전체 softirq 처리 시간이 길어질 수 있는가?
4. Budget을 무작정 크게 하면 어떤 장점과 단점이 있는가?

**주간 결과물:** Budget·Repoll·Completion 상태 전이표 1개와 코드 분기별 설명.

---

## 3주 차 — 11/15~11/21: DMA Ring, Buffer Ownership, Race Condition

**목표:** 하드웨어와 CPU가 RX descriptor 및 데이터 버퍼를 언제 소유하고 재사용하는지 설명한다.

### 읽을 코드

- [`drivers/net/wireless/mediatek/mt76/dma.c`](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/mediatek/mt76/dma.c)
  - `mt76_dma_rx_poll()`
  - `mt76_dma_rx_process()`
  - `mt76_dma_dequeue()`
  - `mt76_dma_rx_fill()` 및 RX buffer 관련 보조 함수
- `mt792x_dma.c`
  - `mt792x_rx_poll_complete()`
  - RX 인터럽트 재활성화 경로
- [Linux DMA API](https://docs.kernel.org/core-api/dma-api.html)
  - streaming/coherent DMA, CPU/Device 동기화의 의미

### 해야 할 일

- [ ] Descriptor 할당/초기화 → 하드웨어 제공 → DMA 완료 → CPU 소비 → buffer refill 생명주기를 그린다.
- [ ] `MT_DMA_CTL_DMA_DONE`을 확인하는 위치와 조건을 찾는다.
- [ ] DMA 데이터와 descriptor 접근에 필요한 메모리 순서·동기화 수단을 구분한다.
- [ ] `DMA_DONE=0`을 발견한 뒤 Poll이 종료되는 코드 경로를 추적한다.
- [ ] **시나리오 A:** Poll 종료 직전 새로운 descriptor가 완료될 때의 이벤트 순서를 작성한다.
- [ ] **시나리오 B:** IRQ가 masked인 동안 새로운 DMA 완료가 발생할 때의 이벤트 순서를 작성한다.
- [ ] 실제 하드웨어의 Status/ACK/Mask 동작은 코드·공식 하드웨어 문서로 확인된 범위까지만 단정한다.
- [ ] 버퍼 할당 실패나 descriptor 부족 시의 분기 및 후속 영향을 찾는다.

### 답할 질문

1. `DMA_DONE=1`인 descriptor를 CPU가 읽어도 안전하다는 근거는 무엇인가?
2. DMA buffer는 정확히 언제 다시 Device에 제공되는가?
3. IRQ를 재활성화하기 직전 새 이벤트가 발생하면 유실될 가능성이 있는가?
4. 현재 코드만으로 판단할 수 없는 하드웨어 동작은 무엇인가?

**주간 결과물:** RX descriptor/buffer 생명주기 그림 1개, 위험한 이벤트 순서 2개, 검증 근거 및 미확인 항목 목록.

---

## 4주 차 — 11/22~11/30: 성능 해석·리포트 완성·XDMA 연결

**목표:** 소스 분석과 기존 실험을 하나의 보고서로 통합하고 12월 XDMA 분석으로 넘긴다.

### 기존 실험 다시 해석하기

- [ ] `napi_poll`의 `budget`과 `work` 분포를 표로 정리한다.
- [ ] 과거에 측정한 `work=0`의 가능한 코드 경로를 분류한다.
- [ ] 높은 UDP 부하에서 패킷 크기별 PPS, 처리량, CPU 사용률 차이를 정리한다.
- [ ] `ethtool -S`, `/proc/net/snmp`의 `RcvbufErrors`, `rx_dropped`를 구분한다.
- [ ] **NAPI 원인으로 확정되지 않은 패킷 손실**은 가설로만 기록한다.
- [ ] 필요하다면 동일한 측정 조건에서만 추가 실험한다. 기존 결과만으로 확인 가능한 부분은 다시 측정할 필요가 없다.

### NAPI 최종 리포트 목차

1. 목적과 문제 정의: Receive Livelock, 인터럽트 과부하
2. NAPI 아키텍처: IRQ, softirq/threaded NAPI, scheduling
3. 실제 코드 분석: MT7922/mt76의 IRQ → NAPI Poll 경로
4. DMA Ring 및 버퍼 생명주기
5. Budget, Completion, IRQ 재활성화, 경쟁 조건
6. 실험 관찰과 성능 분석: 사실·가설 구분
7. NAPI로부터 얻은 드라이버 설계 원칙 및 XDMA 비교 과제

### 최종 점검

- [ ] 핵심 함수마다 **호출 주체 / 실행 문맥 / 입력 / 상태 변경 / 종료 조건**을 작성했다.
- [ ] 모든 핵심 흐름에 실제 소스 위치 또는 근거를 연결했다.
- [ ] 정상 경로뿐 아니라 Budget 소진과 버퍼 부족 등 경계 조건을 설명했다.
- [ ] 인터럽트 유실 방지 메커니즘을 확인된 사실과 추정으로 구분했다.
- [ ] 실험 조건·측정값·해석을 분리했다.
- [ ] 11월 30일까지 1차 완성본을 고정하고, 이후에는 새 NAPI 주제를 무리하게 확장하지 않는다.

**주간 결과물:** NAPI 분석 리포트 1차 완성본 및 XDMA 인계 체크리스트.

---

## 12월 XDMA 분석을 위한 인계 체크리스트 (11월 말에만 작성)

11월에는 XDMA 보고서를 본격적으로 쓰지 않더라도, 다음 질문을 정리해두면 12월의 초기 분석 시간을 줄일 수 있다.

- [ ] **드라이버 종류 구분:** 메인라인 `drivers/dma/xilinx/xdma.c`(DMAEngine)와 AMD/Xilinx `libxdma.c`(별도 구현)
- [ ] **전송 경로:** Descriptor 준비 → submit → `issue_pending()` → `xdma_xfer_start()`
- [ ] **완료 경로:** 채널 ISR → 완료 descriptor 수 확인 → `vchan_cookie_complete()`
- [ ] **공통점:** DMA mapping, descriptor, IRQ, completion, lock, memory ordering
- [ ] **차이점:** NAPI의 Poll Budget/공정성 vs XDMA의 전송 요청 및 완료 관리
- [ ] **설계 질문:** 작은 DMA 전송이 매우 자주 완료될 때 ISR 비용과 완료 batching은 어떻게 관리할 것인가?
- [ ] **업무 연결:** HDLC/UART 상위 API가 요청·완료·타임아웃·오류를 어떻게 받아야 하는가?

---

## 매번 코드 분석할 때 사용할 기록 템플릿

```md
### 분석 대상: 함수명 / 커널 버전 / 파일 위치

- 역할:
- 호출 주체와 실행 문맥(Hard IRQ / Softirq / kthread / process):
- 사용하는 상태/레지스터/Lock:
- DMA buffer/descriptor의 현재 소유자:
- 정상 동작 순서:
- 종료 조건과 반환값:
- 경계 조건/실패 경로:
- 가능한 경쟁 조건:
- 해당 가설을 확인할 코드·Trace·하드웨어 문서:
- XDMA와 연결되는 설계 원칙:
```

## 11월 최종 완료 기준

다음 다섯 가지를 **실제 코드와 근거를 들어** 설명할 수 있으면 11월 목표를 달성한 것으로 본다.

1. 왜 IRQ handler와 실제 패킷 처리를 분리하는가?
2. 왜 NAPI에 Budget과 Repoll 메커니즘이 필요한가?
3. NIC의 DMA descriptor와 수신 버퍼는 언제 CPU/Device가 소유하는가?
4. Poll 종료 및 IRQ 재활성화 과정의 경쟁 조건을 어떻게 검증하는가?
5. 실측한 패킷 손실·CPU 사용률·Poll Work 사이에서 확인된 사실과 아직 확인하지 못한 원인을 구분할 수 있는가?

## 주요 참고 자료

- [Linux Kernel — NAPI](https://docs.kernel.org/networking/napi.html)
- [Linux Kernel — DMA API](https://docs.kernel.org/core-api/dma-api.html)
- [Linux Kernel — DMAEngine Provider](https://docs.kernel.org/driver-api/dmaengine/provider.html)
- [Linux Kernel — DMAEngine Client](https://docs.kernel.org/driver-api/dmaengine/client.html)
- [Mogul & Ramakrishnan — Eliminating Receive Livelock in an Interrupt-Driven Kernel (PDF)](https://web.stanford.edu/class/archive/cs/cs240/cs240.1196/readings/p217-mogul.pdf)
- [Linux mainline mt76/mt792x_dma.c](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/mediatek/mt76/mt792x_dma.c)
- [Linux mainline mt76/dma.c](https://github.com/torvalds/linux/blob/master/drivers/net/wireless/mediatek/mt76/dma.c)
- [Linux mainline XDMA DMAEngine driver](https://github.com/torvalds/linux/blob/master/drivers/dma/xilinx/xdma.c)

---

**핵심 원칙:** 11월의 성공 기준은 읽은 코드의 양이 아니라, **드라이버 설계 결정을 설명하고 코드와 실험으로 검증할 수 있는 능력**이다.
