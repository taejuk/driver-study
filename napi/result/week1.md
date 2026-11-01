
왜 mt76_connac_irq_enable()은 인터럽트를 즉시 하드웨어에 활성화하지 않고, 먼저 소프트웨어 마스크를 갱신한 다음 tasklet을 다시 예약할까?

sw : dev->mmio.irq_enable , hw : register interrupt enable

tasklet이 하는 작업이 하드웨어 인터럽트 state를 보고 pending중인 interrupt가 있으면 해당 작업을 처리하는 NAPI를 scheduling하고, 만약에 없으면 IRQ enable를 다시 활성화한다.
만약 하드웨어 마스크를 바로 업데이트하면 이미 들어온 hw interrupt 요청을 누락시킬 수도 있다. (물론 다른 방식으로 처리 가능)

hardware irq > softirq > kernel context > user context
irq에서는 sleep금지. 그리고 lock을 잡을 때는 irqsave를 통해서 flag 저장하고 interrupt 수신하면 안된다. 이유는 lock 잡은 상태에서 interrupt 수신 -> interrupt 실행 근데 그 interrupt가 같은 lock 잡으려고 시도. -> deadlock 발생