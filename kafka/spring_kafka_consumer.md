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
