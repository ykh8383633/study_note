# Multi-Thread 비동기 방식으로 Consumer 성능 향상 시킬 시 문제점

### multi-thread 비동기 방식으로 메세지 처리 시 발생할 수 있는 문제

- `ListenerConsumer`코드를 보면 기본적으로 while문을 돌며 `pollAndInvoke` 함수로 브로커에서 message를 polling 하고 가져온 메세지 처리를 기다린 후 다음 while문을 진행한다.
- `pollAndInvoke` 함수를 실행한 thread가 아닌 다른 thread에서 비동기로 메세지 처리 시, 메세지 처리를 기다리지 않고 다음 while문을 진행하기 때문에 모든 메세지의 처리가 완료되지 않은 상태에서 다음 메세지를 polling 하는 상황이 발생할 수 있다.
- 문제 1: kafka는 commit된 offset 이후 메세지를 가져오기 때문에 이전에 가져왔지만 아직 처리가 되지 않아 ack을 호출 하지 않은 메세지를 중복해서 가져오는 상황이 발생할 수 있다.
- 문제 2: ack은 offset 순서대로 호출해야 하는데, multi-thread로 메세지를 처리한다면 어떤 메세지가 먼저 완료될지 모른다는 것이다.

### 메세지 처리 완료 전 다시 polling(메세지 중복 처리) 하는 문제

ListenerConsumer(KafkaMessageListnerContainer 내부 private class)
````java

public void run() {
    // ...
    // while문을 돌며 pollAndInvoke 호출
    while (isRunning()) {
        try {
            pollAndInvoke(); // message를 poll하고 listner를 실행 시킴
        }
    
    // ...
}

protected void pollAndInvoke() {
    // ...
    pauseConsumerIfNecessary(); // 특정 상황에 consumer 를 멈춤 (polling 중단)
    pausePartitionsIfNecessary();
    // ...
    ConsumerRecords<K, V> records = doPoll(); // 메세지 poll
    // ...
    invokeIfHaveRecords(records); // messageListner 실행
    if (this.remainingRecords == null) {
        resumeConsumerIfNeccessary(); // consumer를 재기동 해야한다면 재기동(polling을 다시 시작)
        if (!this.consumerPaused) {
            resumePartitionsIfNecessary();
        }
    }

}
````
- pollAndInvoke 함수 내부를 보면 함수명에서도 알 수 있듯 `특정 상황`에 consumer를 정지시키는 `pauseConsumerIfNecessary` 함수를 호출 하는 것을 알 수 있다.
- 여기서 말하는 `특정 상황` 이란 무엇일까?

ListenerConsumer
````java
private void doPauseConsumerIfNecessary() {
    // ...
    // offsetsInThisBatch가 null 혹은 비어 있지 않을 경우 this.pausedForAsyncAcks = true
    if (this.offsetsInThisBatch != null && !this.offsetsInThisBatch.isEmpty() && !this.pausedForAsyncAcks) {
        this.pausedForAsyncAcks = true; 
        this.logger.debug(() -> "Pausing for incomplete async acks: " + this.offsetsInThisBatch);
    }
    if (!this.consumerPaused && (isPaused() || this.pausedForAsyncAcks)
            || this.pauseForPending) {

        Collection<TopicPartition> assigned = getAssignedPartitions();
        if (!CollectionUtils.isEmpty(assigned)) {
            this.consumer.pause(assigned); // 컨슈머 멈춤
            this.consumerPaused = true;
            // ...
        }
    }
}
````
- 여기서 말한 `특정 상황`이란 `offsetsInThisBatch != null && offsetsInThisBatch.isNotEmpty`인 상황이다.
- 언제 위와 같은 상황이 발생할까?

ListenerConsumer 생성자
````java
ListenerConsumer(GenericMessageListener<?> listener, ListenerType listenerType, ObservationRegistry observationRegistry){
    AckMode ackMode = determineAckMode();
    this.isManualAck = ackMode.equals(AckMode.MANUAL);
    // ...
    this.isManualImmediateAck = ackMode.equals(AckMode.MANUAL_IMMEDIATE);
    this.isAnyManualAck = this.isManualAck || this.isManualImmediateAck;
    // ...
    /*
    * ackMode가 MANUAL or MANUAL_IMMEDIATE
    * this.containerProperties.isAsyncAcks = true인 경우
    * 둘 다 messageListnerContainer configuration 에서 설정
    */ 
    this.offsetsInThisBatch = 
            this.isAnyManualAck && this.containerProperties.isAsyncAcks()
                    ? new HashMap<>()
                    : null;
    this.deferredOffsets =
            this.isAnyManualAck && this.containerProperties.isAsyncAcks()
                    ? new HashMap<>()
                    : null;
    // ...
}
````
- `ackMode`가 `MANUAL` or `MANUAL_IMMEDIATE`이고, `this.containerProperties.isAsyncAcks = true` 인 경우 위에서 말한 `특정 상황`을 만족한다.
- 왜 두 조건을 만족하면 컨슈머를 멈출까?

### ackMode 와 asyncAcks

- kafka acks?
  - 브로커에 잘 수신 했다는 응답.
  - ack을 수행하면 offset을 이동해 다음 메세지 부터 받을 수 있음
- `ackMode == MANUAL || akcMode == MANUAL_IMMEDIATE` 이면?
  - ackMode는 기본적으로 `BATCH`
    - 일정 주기마다 자동으로 ack을 보낸다.
    - 메세지 처리 여부와 상관없이 offset이 올라가 메세지 중복, 유실이 발생할 수 있다.
  - `ackMode.MANUAL` or `ackMode.MANUAL_IMMEDIATE`
    - ack 호출을 개발자가 코드로 호출한다. 
    - 메세지를 잘 처리한 이후에 ack을 호출해 메시지를 보다 안전하게 처리할 수 있다.
- syncAcks?
  - consumer는 기본적으로 message의 offset 순서대로 `Acknowledgment.acknowledge`를 호출해야한다.
  - asyncAcks는 비동기로 message를 처리하는 경우, 순서를 지키지 않고 `Acknowledgment.acknowledge`를 호출할 수 있도록 해준다.
    - 어떻게 순서를 지키지 않고 호출할 수 있는지는 뒤에서 설명
  > 참고:<p/>
  https://docs.spring.io/spring-kafka/reference/kafka/container-props.html <p/> 
  https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/ooo-commits.html

- `ackMode == MANUAL || akcMode == MANUAL_IMMEDIATE`이면서 `asyncAcks = true`이면 왜 consumer를 멈출까?
  - `ackMode = MANUAL` 이라면 messageListner에서 임의 시점에 `Acknowledgment.acknowledge`가 호출된다.(`ListenerConsumer` acks가 언제 호출 될지 모름)
  - 비동기로 `messageListner`를 실행하는 상황이라면 `pollAndInvoke`를 실행하는 thread가 blocking 되지 않아 다음 while문을 진행하고 메세지 중복이 발생할 수 있다.
  - 따라서 `ackMode == MANUAL || akcMode == MANUAL_IMMEDIATE`이고 비동기로 message를 처리한다면 `asyncAcks = true`로 설정해 메세지 처리 완료 전 까지 polling을 멈추도록 해야한다.

### multi-thread로 메세지 처리 시 offset 순서와 상관없이 ack을 호출하는 문제

ConsumerAcknowledgment
````java
@Override
public void acknowledge() {
    if (!this.acked) {
        doAck(this.cRecord);
        this.acked = true;
    }
}

private void doAck(ConsumerRecord<K, V> cRecord) {
    traceAck(cRecord);
    if (this.offsetsInThisBatch != null) { // NOSONAR (sync)
        ackInOrder(cRecord);
    }
    else {
        processAck(cRecord);
    }
}

private synchronized void ackInOrder(ConsumerRecord<K, V> cRecord) {
    TopicPartition part = new TopicPartition(cRecord.topic(), cRecord.partition());
    List<Long> offs = this.offsetsInThisBatch.get(part);
    if (!ObjectUtils.isEmpty(offs)) {
        List<ConsumerRecord<K, V>> deferred = this.deferredOffsets.get(part);
        if (offs.get(0) == cRecord.offset()) {
            offs.remove(0);
            ConsumerRecord<K, V> recordToAck = cRecord;
            if (!deferred.isEmpty()) {
                Collections.sort(deferred, (a, b) -> Long.compare(a.offset(), b.offset()));
                while (!ObjectUtils.isEmpty(deferred) && deferred.get(0).offset() == recordToAck.offset() + 1) {
                    recordToAck = deferred.remove(0);
                    offs.remove(0);
                }
            }
            processAck(recordToAck);
            if (offs.isEmpty()) {
                this.deferredOffsets.remove(part);
                this.offsetsInThisBatch.remove(part);
            }
        }
        else if (cRecord.offset() < offs.get(0)) {
            // ...
        }
        else {
            deferred.add(cRecord);
        }
    }
    // ...
}

private void processAck(ConsumerRecord<K, V> cRecord) {
    if (!Thread.currentThread().equals(this.consumerThread)) { // 비동기 호출이 아닌 경우
        // ...
    }
    else { // 비동기 호출인 경우
        if (this.isManualImmediateAck) { 
            try {
                ackImmediate(cRecord);
            }
            // ...
        }
        else {
            addOffset(cRecord);
        }
    }
}
````
- 메세지 처리 후 ack 호출을 담당하는 `ConsumerAcknowledgment` class 코드이다.
- `acknowledge`함수 호출 시 호출 되는 `ackInOrder` 함수를 보면
  - 처리가 완료된 record는 `deferred`에 담고 처리가 되지 않은 record 들은 `offs`에 유지되고 있다.
  - 처리 중 `offs`에 가장 첫번째 offset에 해당하는 record가 `ackInOrder` 함수로 들어오면 그 offset 부터 연속적으로 순서가 맞는 offset 까지 `processAck`을 호출하고 그 레코드 까지 `offs`와 `deferred`에서 제거한다.
- 순서에 상관없이 `acknowledge`를 호출해도 순서가 맞을 때 까지 기다렸다가 ack을 호출한다는 말이다.
- `ackMode = MANUAL` 과 `ackMode = MAUAL_IMMEDIATE` 의 차이
  - `processAck` 함수를 보면 `ackMode = MAUAL_IMMEDIATE` 이면 브로커에 순서가 맞은 offset 까지 즉시 ack을 전달하고  `ackMode = MAUAL`이면 polling으로 가져온 모든 메세지가 처리된 이후에 ack을 호출한다.

### 비동기 메세지 처리의 문제점
- polling한 메세지들을 처리하던 중 서버가 다운된다면 이미 처리 되었지만 offset 순서가 아직 맞지 않아 ack을 호출하지 않은 메세지가 생긴다.
- 위와 같은 상황에서 리벨런싱 후 다시 consume이 시작된다면 메세지 중복이 일어날 수 있다.
- 이 문제를 해결하려면? `ackMode = MAUAL_IMMEDIATE`로 설정해 중복을 최소화 하고 DB에 처리 여부를 잘 남겨두어야 할 것 같다. 