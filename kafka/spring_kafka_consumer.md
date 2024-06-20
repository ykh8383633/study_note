## Spring Kafka Consumer 

### ConcurrentMessageListenerContainer
 - dostart()에서 설정한 concurrency 만큼 KafkaMessageListnerContainer를 생성해 해당 instance를의 doStart를 호출
 <p/>

ConcurrentMessageListenerContainer
 ````java

    protected void doStart(){
        // ...
        if (topicPartitions != null && this.concurrency > topicPartitions.length) {
				// ...
                // 토픽의 파티션 이상으로 concurrency가 설정되어 있다면 partition갯수로 낮춤
				this.concurrency = topicPartitions.length;
			}
			setRunning(true);

        for (int i = 0; i < this.concurrency; i++) {
            KafkaMessageListenerContainer<K, V> container =
                    constructContainer(containerProperties, topicPartitions, i);
            configureChildContainer(i, container);
            if (isPaused()) {
                container.pause();
            }
            container.start();
            this.containers.add(container);
		}
    }
````
KafkaMessageListnerContainer
````java


    protected void doStart() {
        // ...

		this.listenerConsumer = new ListenerConsumer(listener, listenerType, observationRegistry); // 컨슈머 생성
		setRunning(true);
		this.startLatch = new CountDownLatch(1);
        // 컨슈머 실행
		this.listenerConsumerFuture = consumerExecutor.submitCompletable(this.listenerConsumer);

        // ...
    }

````

ListenerConsumer(KafkaMessageListnerContainer 내부 private class)
````java
    public void run() {
        // ...

        while (isRunning()) {
				try {
					pollAndInvoke(); // message를 poll하고 listner를 실행 시킴
					if (failedAuthRetry) {
						publishRetryAuthSuccessfulEvent();
						failedAuthRetry = false;
					}
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

    // invokeIfHaveRecords를 타고 가면 나옴 (메세지 처리 부, batch 아님)
    // polling한 메세지를 순회하며 listner 실행
    private void doInvokeWithRecords(final ConsumerRecords<K, V> records) {
			Iterator<ConsumerRecord<K, V>> iterator = records.iterator();
			while (iterator.hasNext()) {
				if (this.stopImmediate && !isRunning()) {
					break;
				}
				final ConsumerRecord<K, V> cRecord = checkEarlyIntercept(iterator.next());
				if (cRecord == null) {
					continue;
				}
				this.logger.trace(() -> "Processing " + KafkaUtils.format(cRecord));
				doInvokeRecordListener(cRecord, iterator); // listner 실행
				if (this.commonRecordInterceptor !=  null) {
					this.commonRecordInterceptor.afterRecord(cRecord, this.consumer);
				}
				if (this.nackSleepDurationMillis >= 0) {
					handleNack(records, cRecord);
					break;
				}
				if (checkImmediatePause(iterator)) {
					break;
				}
			}
		}
 ````

 ### 언제 consumer를 멈출까?
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
 - 어떤 경우 offsetsInThisBatch가 null이 아닐까? <P/>

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
### ackMode 와 asyncAcks

- kafka ack?
  - 브로커에 잘 수신 했다는 응답. 
  - ack을 수행하면 offset을 이동해 다음 메세지 부터 받을 수 있음

- `ackMode = MANUAL`이면?
  - ackMode는 기본적으로 BATCH
    - 일정 주기마다 자동으로 ack을 보낸다.
    - 메세지 처리 여부와 상관없이 offset이 올라가 메세지 중복, 유실이 발생할 수 있음
  - ackMode.MANUAL
    - ack 호출을 개발자가 코드로 호출
    - 메세지를 잘 처리하고 호출해 메시지를 보다 안전하게 처리
    - ack을 호출하기 위해서는 AcknowledgingMessageListener, ConsumerAwareMessageListener, AcknowledgingConsumerAwareMessageListener 를 messageListner로 구현해야함. 
      - onMessage 파라미터로 ack을 호출 할 수 있는 Acknowledgment or Consumer 객체를 넘겨줌
    
- asyncAcks?
  - kafka consumer는 기본적으로 message의 offset 순서대로 Acknowledgment.acknowledge를 호출해야함.
  - asyncAcks는 비동기로 message를 처리하는 경우, 순서를 지키지 않고 Acknowledgment.acknowledge를 호출할 수 있도록 해줌 <p/>
    > 참고: <p/>https://docs.spring.io/spring-kafka/reference/kafka/container-props.html <p/> https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/ooo-commits.html <p/>

<p/>

- 왜 `ackMode = MANUAL` 이면서 `asyncAcks = true` 이면 consuemr를 pause 할까?
  - `ackMode = MANUAL` 이라면 messageListner에서 임의 시점에 `Acknowledgment.acknowledge`가 호출된다. container는 언제 호출 될지 모른다.
  - async, non-blocking 으로 messageListner를 실행하는 상황이라면 메세지 처리 동안 쓰레드가 blocking 되지 않는다.
  - blocking 되지 않은 쓰레드는 `ListenerConsumer.run` 함수 내의 `while`문에서 다시 `pollAndInvoke`를 호출한다.
  - messageListner가 아직 message를 처리 중이라 `acknowledge`를 호출하지 않았다면 consumer는 이전 poll에서 가져온 message를 중복해서 가져오게 된다.
  - 따라서 `ackMode = MANUAL`이고 비동기로 message를 처리한다면 `asyncAcks = true`로 설정해 메세지 처리 완료 전 까지 poll을 멈추도록 해야한다.




