## Introduction
![](https://velog.velcdn.com/images/daehoon12/post/4aa3ffef-298a-46b6-a20a-d56e73182edc/image.png)
- 서비스 최적화 
- 카프카 설정
- 벤치마크, 모니터링, 튜닝

## Deciding Which Service Goals to Optimize
![](https://velog.velcdn.com/images/daehoon12/post/f2de7422-381e-42d0-92b2-4e850c241be7/image.png)
- 위의 그림을 이루고자하는 서비스의 목표가 있으며, 이 4가지는 서로 **Trade-Off** 관계다.
- 위의 4가지를 모두 달성하는 방법은 없으므로 만들려는 서비스의 목적에 맞게 설정해야 한다.

![](https://velog.velcdn.com/images/daehoon12/post/a7bed94c-ac0b-4e7a-82c9-ef9a80326ffd/image.png)


### 1. High Throughput
- 데이터가 Producer-Broker-Consumer로 이동하는 속도. 즉 **데이터가 이동하는 비율**

### 2. Low Latency
- 메시지가 **End to End로 이동할 때 경과하는 시간**을 Latency라고 정의한다.
- 채팅 어플리케이션, SNS 등 해당 서비스에서 중요하다.

### 3. High Durability
- ** 한 번 커밋된 데이터는 유실되지 않아야 한다.**
- 프로듀서에 의해 카프카로 전송되는 모든 메세지는 **안전한 저장소인 카프카 로컬 디스크에 저장**된다.
- 또한 컨슈머가 메세지를 가져가더라도 **메세지는 삭제되지 않고 지정한 설정 시간 또는 로그의 크기 만큼 로컬 디스크에 보관**되므로 코드의 버그나 장애가 발생하더라도 **과거의 메세지들을 불러와 재처리 할 수 있다.**

### 4. High Availability
- kafka는 분산시스템으로 **fault-tolerance가 구현**되어 있다.
- High Availability를 요구하는 사례에서 **카프카를 최대한 빨리 장애에서 복구**할 수 있도록 구성하는 것이 중요하다.


## Optimizing for Throughput
- Throughput를 최적하하려면 Producer, Broker, Consumer가 시간 내에 많은 데이터를 이동해야 한다. 즉, **Throughput을 높이기 위해 데이터가 이동하는 속도를 최대화해야 한다.**

### Producer 설정
- Partition 수를 증가
    - 토픽의 **파티션은 카프카에서 병렬로 처리되는 단위**다. 파티션을 사용하면** 처리율이 높아지며 **처리율을 극대화하려면 클러스터의 모든 브로커를 활용할 수 있는 충분한 파티션이 필요하다.
    - 다만 파티션 수를 늘리는 것은 Trade Off 관계다. ([파티션 수를 결정하는 전략](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/))
        - 리더 파티션이 장애가 발생했을 경우 다음 리더를 선출해야 하는데, 이 때 **파티션이 너무 많으면 리더 선출에 많은 시간이 소요** 된다. (Unavailability)
        - 카프카는 메시지가 커밋된 후, 즉 **메시지가 in-sync replicas에 복제가 완료된 이후 Consumer에게 메시지를 노출**시킨다. 즉 메시지를 커밋하는 시간은 종단 간의 지연 시간이 길어질 수 있다. (High Latency)
- Batch 전송 전략을 최적화
    - producer는 **같은 파티션에 대한 데이터를 batch로 처리** 한다. 즉, **여러 message를 하나의 request로 묶어서 전송**.
    - 한 번에 많은 양을 보내면 **브로커에게 요청하는 전송횟수가 줄어 I/O 작업이 감소**한다.
    - 배치 크기가 클 수록, Producer가 전송 전에 기다리는 시간이 클 수록 브로커에게 요청하는 횟수가 줄어들어 브로커와 컨슈머의 부하가 감소한다.
        
**Parameter 설정**
- `batch.size = 100,000~200,000 (default : 16384)`
    - batch request의 데이터 사이즈
- `linger.ms = 10~100 (default : 0)`
    - 배치사이즈가 다 채워질 때까지 기다리는 시간
- `compression.type = lz4 (default : none)`
    - batch 데이터를 압축할 알고리즘
    - Confluent 공식 문서에서는 lz4를 추천하고 gzip으로 압축하는 것은 권장하지 않는다. 이는 gzip이 압축 비율이 높아서 압축 속도나, CPU 사용량, 디스크 저장 속도, 네트워크 대역폭 사용량이 줄어들기 때문이다.
- `ack = 1 (dafault 1)`
    - Producer가 데이터 전송 후 commit 결과를 받는 방식.
    - 1로 설정 시, Leader Broker로부터 메시지를 잘 받았는지 ack를 기다린다.
- `retries = 0 (default 0)`
    - producer가 전송 실패시 재시도 횟수. 
    - 0으로 설정하면 처리량은 증가하나 메시지 유실 가능성이 존재한다.
- `buffer.memory (default 33,554,432)`
    - Producer가 보내지 못한 메시지를 보관할 메모리의 크기. (producer의 accumulator)
    - 보내지 않은 메세지를 담은 buffer가 가득차면, max.block.ms가 지나거나, 메모리가 해제되기 전까지 전송을 하지 않는다.
    - partition이 많지 않다면 이 설정값을 수정할 필요는 없으나 많다면 메모리를 늘려 Blocking 없이 더 많은 데이터가 전송되도록 설정해야 한다.
    
### Consumer 설정
- 메시지 수신량을 최대화
    - fetch.min.bytes 값 증가 시키기
    	- 이 값보다 적은 데이터가 있으면, Broker에서 메시지를 가져오지 않는다. 즉 **값을 올린다면, fetch하는 요청 횟수가 줄어드므로, Broker의 CPU OverHead (I/O)가 감소한다.**
    - fetch.max.wait.ms: Consumer에서 데이터를 가져오는 최소 시간
        -  fetch.max.wait.ms이 만료 될 때까지 broker는 consumer에게 message를 전송하지 않는다.
     - Consumer Group 활용
        - consumer 수가 많으면 parallelism 증가, load 분산, 동시에 여러개 partition에 연결하여 Througput을 증가시킨다.
     - GC 튜닝 ([카프카 JVM 튜닝](https://docs.confluent.io/current/kafka/deployment.html#jvm))
 
 
#### Parameter 설정
- fetch.min.bytes = ~ 100,000 (default 1)
    - Consumer에서 가져오는 최소 데이터 사이즈

### 설정 요약
![](https://velog.velcdn.com/images/daehoon12/post/d49715bb-1935-4820-9943-970c24c9e912/image.png)

## Optimizing for Latency
- kafka에서 병렬성 (parallelation)이 증가하면 처리량이 증가하나, **지연 시간도 같이 증가한다.** 기본적으로 브로커는 싱글 스레드로 다른 브로커의 데이터를 복제하므로, **파티션의 수가 많을 수록 복제 시간이 더 오래걸리고** 복제가 모두 완료되어야 Consumer가 메시지에 접근할 수 있기 때문이다. 

### Broker 설정
- Partition 개수 제한
    - ACK가 all일 겨우 모든 복제본의 ack를 받아야 메시지를 전송하기 때문에 **신뢰성 있게 데이터를 전송할 경우에는 파티션의 개수를 줄이는 것도 방법이다.**
- Broker의 수를 늘리고 Partition의 수를 줄이기.
    - **Broker 서버를 증설해서 broker 서버당 Consumer에 할당되는 partition 개수를 줄이기**
- num.replica.fetchers
    - Broker의 옵션으로 소스 브로커의 메시지를 복제하는 데 사용되는 스레드 수를 지정하는 파라미터
    - 이 값을 늘리면 브로커의 I/O 작업 시 병렬 처리가 늘어날 수 있다. 즉, **병렬처리가 늘어나 I/O 작업을 처리하는데 Latency를 줄일 수 있다.**

### Producer 설정
- `linger.ms = 0`
    - 데이터를 수집하는 순간 브로커로 전송시킨다.
- `compression = none`
    - 압축 알고리즘 미적용
- `ack = 1`
    - 데이터 복제 없이 리더 파티션의 ack만 받으면 브로커에게 메시지 전송

### Consumer 설정
- `fetch.min.bytes = 1`
    - 한 번에 가져올 수 있는 최소 데이터 사이즈. 해당 옵션의 파라미터를 1로 설정하면 요청시 바로 전송 할 수 있어 Latency가 없다.

### Streams 설정 (Kafka Streams 사용 시)
- `StreamsConfig.TOPOLOGY_OPTIMIZATION: StreamsConfig.OPTIMIZE (default StreamsConfig.NO_OPTIMIZATION)`
    - 토폴로지 최적화를 활성화하면 재분할된 토픽을 통해 저장되고 파이프링되는 스트림 (reshuffled Streams) 양이 줄어들 수 있다. (Enabling topology optimizations may reduce the amount of reshuffled streams that are stored and piped via repartition topics.)
   

### 설정 요약
![](https://velog.velcdn.com/images/daehoon12/post/145c97f8-3244-458e-8ab6-bb7f2b866681/image.png)

## Optimizing for Durability
- Durability은 메시지의 유실을 최소화 하는 지표다. Durability 가능하게 하는 가장 중요한 기능은 replication다. 메시지를 여러 브로커에 복제를 하면 장애 발생 시 다른 브로커로부터 데이터를 얻을 수 있다.

### Broker 설정
- `replication.factor = 3` (default 1) or `auto.create.topics.enable=false` (default true)
    - replication.factor: 토픽의 파티션의 복제본을 몇 개를 생성할 지에 대한 설정
        -  producer가 존재하지 않은 topic에 producing되는 경우 `auto.create.topic.enable = true` 일 경우 토픽이 자동으로 생성된다.
    - `auto.create.topics.enable`: Broker 설정 중에 자동으로 Topic을 생성해 주는 것에 대한 설정.
    - `auto.create.topics.enable = false`로 하던지,`default.replication.factor = 3`으로 설정할 것
- `min.insync.replicas = 2`
    - 최소 리플리케이션 팩터를 지정하는 옵션
    - 해당 값을 `replication.factor`과 동일하게 지정 시, Broker가 한대만 장애가 나도 **Replication Factor가 부족하다는 내용의 메세지가 나타나면서 카프카 메시지를 보낼 수 없는 상황이 발생**한다. ([acks=all 일 때 min.insync.replicas=2 설정을 권장하는 이유](https://devlog-wjdrbs96.tistory.com/436))
- `unclean.leader.election.enable = false`
    - broker 서버가 내려가면 해당 broker의 문제를 파악하고 Broker가 가진 Leader Partition을 다른 partition으로 변경한다. 또한, 새로운 Leader를 선정하는 방법은 **해당 Partition의 ISR 리스트 중 하나를 Leader로 선정하고 나머지 Follower들은 새로운 Leader Partition 정보를 복제**한다.
    - `unclean.leader.election.enable`은 ISR가 아닌 Broker 서버(Out of Sync Replication)를 leader로 선정할 것 인가에 대한 설정 값이며 해당 값을 false로 선언 시 **싱크가 되지 않은 브로커를 Leader로 선정하지 않아 Durablity는 증가**시키지만, **replica가 sync될 때 까지 기다리는 시간이 생겨 availablity 측면에서 손해를 본다.**
    - **in-sync replicas의 서버가 전부 죽었을 경우에만 에러**가 나면서 partition을 쓸 수 없는 상태가 된다. 즉, data에 대한 durability가 증가한다.
- `broker.rack`
    - 하나의 Rack에 Broker가 동작하지 않도록 설정한다. 해당 값은 Cloud 기반 서비스에서 Broker가 각각 다른 Zone에 구동되도록 한다.
    - Durability은 높아지지만 데이터 복제 시 Network I/O가 증가한다.
- `log.flush.interval.messages, log.flush.interval.ms`
    - 메시지를 Memory에서 Disk로 저장하는 수준
        - 해당 값이 클수록 Disk I/O는 적게 발생하나 메모리 데이터 유실 가능성이 존재한다.
        - 반대로 값이 작을수록 Disk I/O는 많이 발생하나 메모리 데이터 유실이 거의 존재하지 않는다.
        - 예를 들어, 모든 메시지 처리 후 디스크에 저장하고 싶으면`log.flush.interval.messages=1`로 설정하면 된다.

### Producer 설정
- `replication.factor = 3 ( Topic Level )`
- `enable.idempotence=true (default false)`
    - Producer 가 Record 쓰기 작업을 단 한번만 허용할 것인지 멱등성을 보장
- `acks = all`
    - 모든 Replica의 복제 완료 ack를 수신해야 메시지 송신
- `retries > 0` 
    - Message를 send하기 위해 재시도 되는 횟수
- `max.in.flight.requests.per.connection <= 5`
    - Batch 처리할 때 한번에 보내는 레코드의 양을 정의.
    - 만약 Batch 5개의 메시지를 보내는 도중에 Batch 2가 발송에 실패 했다면 나머지 Batch가 보내지는 중인 상태에서 Batch 2만 Retry되고 이미 보내진 Batch는 전송이 완료 되므로 순서를 보장할 수 없게 된다.
    - `enable.itemoptence`를 true로 설정하면 Batch 2 전송 실패 시 나머지 Batch들을 보내지 않고 에러 처리가 되며 Batch 1~5를 다시 보낸다.
    
### Consumer 설정
- 읽어온 메시지의 위치를 정확하게 기록하기
    - 만약 Consumer가 메시지를 읽어오고 Broker가 Offset을 자동으로 Commit한 이 후 Consumer에서 커밋을 못하고 에러 발생 시, 마지막 커밋 데이터부터 데이터를 처리하기 때문에 이 중복이 발생할 가능성이 있다.
    - `auto.commit.enable = false`로 자동 커밋을 비활성화 시킨 다음 commitAsync() , commitSync, 또는 consumer의 RebalanceLinstener를 상속을 받아 수동 커밋을 구현해야 한다.
    - 또한 여러 카프카 토픽으로 구성된 이벤트 로직에 대한 신뢰성을 보장하기 위해 카프카 트랜잭션을 사용할 수 있다. A, B 토픽이 있다고 가정하고 A 토픽에서 Consume한 메시지를 B 토픽에 send하는 로직이 있다고 가정하자. 이 때, 트랜잭션이 완료가 되지 않는다면 컨슈머는 해당 Record를 가져가지 않아야 한다. 이를 구현하기 위해 `isolation.level=read_committed`로 설정함으로써 `Isolation Level`을 `read_committed`로 설정한다. 
    
### 설정 요약
![](https://velog.velcdn.com/images/daehoon12/post/58e7a1f6-40b8-4d73-9912-568cddf889f4/image.png)


## Optimizing for Avaliablity
- 장애로부터 가능한 빠르게 복구하는데 중점을 둔 지표
    - 파티션이 많을수록 병렬성이 높아지나, 브로커에 문제 발생 시 복구 시간이 더 오래걸린다. 이는 **파티션이 많을수록 리더 선출에 오래 걸리기 때문**이다. 
    - 브로커 안에 파티션 리더가 선정되기 전까지 producer/consumer는 모두 멈춘다.
        
### Broker 설정
- `min.insync.replicas = 1`
    - Producer가 응답을 받기 위한 최소 복제의 수
    - 값이 크면 복제 실패 시 producer의 장애 유발 -> High Durability
    - 값이 1이면 원본만 저장 시 producer 정상 동작 -> Low Durability
- `unclean.leader.election.enable = true`
    - Broker가 죽었을 때, Out Of Sync Replica도 Leader로 선출 가능
    - 리더 선출이 더욱 빠르게 진행되어 전반적인 가용성이 향상
- `num.recovery.threads.per.data.dir > 1 (default 1)`
    - 브로커 가동 시 브로커는 다른 브로커와 동기화 하기 위해 다른로그 데이터 파일을 검색한다. 이를 **log recovery**라고 한다.
    - 브로커 시작 시 log recovery, 브로커 종료 시 로그 플러싱에 사용될 Data Directory 당 스레드는 `num.recovery.threads.per.data.dir`로 정의된다.
    - 브로커에 많은 수의 인덱스 파일이 있어 로드가 느릴 경우에 해당 값을 늘려 동시에 log recovery를 수행하게 설정한다.
    
### Consumer 설정
- Consumer 장애를 감지하여 남은 Consumer로 Partition을 재배치
  - `session.timeout.ms`
      - consumer가 비정상적으로 종료될 경우, Broker가 장애로 인지하는 최소 시간
      - Consumer Failure 유형
          - Hard Failure: SIGKILL (강제 종료)
          - Soft Failure: session time out (대부분의 경우)
              - `poll()` 메서드 호출 후, consumer에서 처리시간이 오래 걸리는 경우
              - JVM GC로 인한 멈춤 현상
      - session.timeout.ms 값이 적으면 컨슈머의 장애를 더 빠르게 감지한다. 가능한 낮은 값을 설정하여, Hard Failure를 빠르게 감지하도록 설정하자.
      - 주의할 점은 너무 낮게 설정하면 Soft Failure까지 감지하여 너무 잦은 에러 처리가 발생한다. Soft Failure를 감지하지 않을 정도로만 설정해야 한다.
      - `poll()` 시 Soft Failure가 자주 발생하면 `max.poll.interval.ms` 값을 늘리거나, `max.poll`로 반환되는 배치의 최대 크기를 줄이면 된다.

### 설정 요약
![](https://velog.velcdn.com/images/daehoon12/post/5e9e4c1c-8f48-4f81-9cca-18d7f14ec15f/image.png)

