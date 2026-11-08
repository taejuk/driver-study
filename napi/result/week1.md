왜 mt76_connac_irq_enable()은 인터럽트를 즉시 하드웨어에 활성화하지 않고, 먼저 소프트웨어 마스크를 갱신한 다음 tasklet을 다시 예약할까?

sw : dev->mmio.irq_enable , hw : register interrupt enable

tasklet이 하는 작업이 하드웨어 인터럽트 state를 보고 pending중인 interrupt가 있으면 해당 작업을 처리하는 NAPI를 scheduling하고, 만약에 없으면 IRQ enable를 다시 활성화한다.
만약 하드웨어 마스크를 바로 업데이트하면 이미 들어온 hw interrupt 요청을 누락시킬 수도 있다. (물론 다른 방식으로 처리 가능)

hardware irq > softirq > kernel context > user context
irq에서는 sleep금지. 그리고 lock을 잡을 때는 irqsave를 통해서 flag 저장하고 interrupt 수신하면 안된다. 이유는 lock 잡은 상태에서 interrupt 수신 -> interrupt 실행 근데 그 interrupt가 같은 lock 잡으려고 시도. -> deadlock 발생

NAPI는 특정 CPU에서만 실행되도록 해서 cache affinity를 극대화시킨다.
local_irq : 현재 CPU에 대한 interrupt. interrupt는 하드웨어가 CPU에게 보내는 것이다.

net_rx_action
budget은 hotdata에서 읽는다. 아마 hotdata랑 colddata 구분하는 이유는 hotdata에 대한 접근을 빠르게 하기 위함이다.

local_irq_disable();
list_splice_init(&sd->poll_list, &list);
local_irq_enable();
여기서 local_irq막는 이유는 중간에 interrupt 들어와서 옮기는 도중에 cpu 빼앗기면 다른 cpu가 실행 -> 값이 깨질 수 있기 때문이다.
in_net_rx_action을 쓰고 barrier하는 이유는 이게 false이고 함수가 끝나면 다른 cpu가 net_rx_action 호출 가능하게 하기 위함. 즉 중복으로 실행되는 것을 막기 위함이다.

list_first_entry : napi_struct->poll_list의 첫번째 값을 가져온다.
net_rx_action에서는 napi_poll 함수를 호출한다.

napi_threaded_poll
일단 TASK_INTERRUPTIBLE이거를 통해서 sleep -> 시그널을 기다린다.
napi는 하나의 thread만 실행 가능하다. 즉, 현재 kthread가 가지고 있어야지 실행 가능하다.

kthread_should_stop : 해당 kernel thread가 멈춰야 하는지 확인.
schedule : 현재 CPU를 양보하고 다른 작업을 하도록 수행.
napi thread인 경우에는 실행할 수 있는 thread가 하나로 고정되어 있다. 그래서



1. list를 어디서 가져오는가?