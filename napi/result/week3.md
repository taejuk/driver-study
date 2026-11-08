하드웨어가 해당 descriptor의 DMA 전송을 끝냈는지 알 수 있는 flag : MT_DMA_CTL_DMA_DONE (ctrl에 저장됨)
flush하면 강제로 종료시키는 듯
해당 descriptor가 가져오는 데이터 길이랑 뒤에 더 있는지도 ctrl을 통해서 가져올 수 있다.

dma_sync_single_for_cpu 함수를 호출하는 것을 보니 mt76에서는 성능을 위해서 streaming을 사용하고
데이터를 가져오기 위해서 device와 cpu가 같은 최신값을 가지도록 dma_sync_single_for_cpu를 사용한다

rx_head가 의미하는 것은 무엇일까?
sk_buff는 하나의 데이터를 의미하는데, 패킷에 대한 메타데이터만 포함하니깐
dma를 할 때 패킷이 하나의 큰 데이터이면 앞의 부분은 rx_head에 담고 나머지 뒤 부분은 따로 담는 것인가?

다 사용하고 나면 page를 pool에다가 넣는다.

근데 지금은 page에 대해서 신경 안써도 될듯