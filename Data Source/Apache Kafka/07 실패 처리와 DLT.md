컨슈머 처리가 실패하면 **다시 시도해서 성공할 수 있는지, 뒤 이벤트를 먼저 처리해도 되는지**부터 판단한다. 무조건 재시도하거나 무조건 건너뛰면 지연·순서 오류·데이터 누락이 생긴다.

## 1. 실패를 나누기

| 실패 | ex) | 기본 대응 |
|---|---|---|
| 일시적 | DB 연결 오류, 외부 API 타임아웃·일시적 5xx | **횟수·간격(backoff)을 제한한 재시도** |
| 재시도로 해결 불가 | 잘못된 JSON, 필수 값 누락, 미지원 이벤트 버전 | 재시도 없이 **DLT(실패 레코드 격리 토픽) 등으로 격리** |
| 업무상 거절 | 이미 취소된 주문의 결제 완료 이벤트 | 업무 규칙 확인 후 결정 |

- **업무상 거절**: 정상 결과로 확정하면 **소비 완료로 보고 오프셋을 커밋**한다. 조사할 이상 이벤트면 DLT로 격리한다.
- 예외 클래스만 보고 정하지 않는다. 선행 데이터가 아직 도착하지 않아 생긴 `404`라면 일시적 실패일 수 있다.
- ==**고칠 수 없는 데이터**와 **복구 가능한 의존 시스템 장애**를 구분==한다.

## 2. 재시도 방식과 순서

| 방식 | 동작 | 적합한 경우 | 주의 |
|---|---|---|---|
| **같은 파티션에서 재시도** (blocking) | 재시도 동안 뒤 레코드 대기 | 같은 키의 처리 순서가 중요 | 컨슈머 스레드 대기로 **같은 컨슈머가 맡은 다른 파티션도 지연**, lag 증가 |
| **재시도 토픽으로 이동** (non-blocking) | 실패 레코드를 별도 토픽에 보내고 원본 소비 계속 | 뒤 이벤트 선처리가 안전, 실시간 처리량 중요 | **업무 처리 순서 변경 가능** |

ex) `OrderCreated(O1)`이 실패해 재시도 토픽으로 가면 원본의 `OrderCancelled(O1)`이 먼저 처리된다.

- ==**같은 키라도 토픽이 다르면 처리 순서는 보장되지 않는다.**==
- 순서가 필수면 원본 파티션에서 재시도한다. 장애가 끝나지 않으면 알림·운영 개입으로 대응한다.
- DLT로 보내고 다음 레코드로 넘어가는 방식도 순서를 건너뛴다.

![Uber의 재시도 토픽과 DLQ 흐름](https://tb-static.uber.com/prod/udam-assets/555b6743-6ca7-54da-83ae-c528fda52cbb.png)

이미지 출처: [Uber Engineering — Building Reliable Reprocessing and Dead Letter Queues with Apache Kafka](https://www.uber.com/us/en/blog/reliable-reprocessing/). Uber는 별도 재시도 토픽으로 실패를 격리해 원본 처리 지연을 줄였다. **순서가 바뀌어도 되는 흐름**에 맞는 구조다.

> [!note] 참고: 재시도 토픽
> Spring Kafka의 `@RetryableTopic`은 재시도 토픽·DLT·컨슈머를 구성하고 지연 후 재처리한다. 공식 문서도 이 방식은 **토픽의 순서 보장이 깨진다**고 적는다. 배치 리스너·컨테이너 트랜잭션과는 함께 쓰지 못한다. [Spring Kafka — 동작 방식](https://docs.spring.io/spring-kafka/reference/3.3/retrytopic/how-the-pattern-works.html) · [제약](https://docs.spring.io/spring-kafka/reference/3.3/retrytopic.html)

## 3. DLT(Dead Letter Topic)

- **DLT**: 자동 처리에 실패한 레코드를 ==**조사·수정·재처리하려고 격리**==하는 토픽. DLT로 보내도 처리가 끝난 것은 아니다. 실패 기록으로 남을 뿐이다.
- **발행 주체**: 브로커는 소비 실패를 감지해 DLT로 옮기지 않는다. 실패 레코드는 컨슈머/프레임워크가 발행하고, DLT 토픽 생성은 별도 설정을 따른다.
- **원본 정보 보존**: 원본 토픽·파티션·오프셋, 키, payload, 오류 원인, 시각. Spring Kafka의 `DeadLetterPublishingRecoverer`는 원본 위치·예외 정보를 헤더에 넣는다.
- **발행 확인 후 커밋**: DLT 발행이 실패했는데 원본 오프셋을 넘기면 레코드가 유실된다. DLT 발행을 확인하고 원본 위치를 커밋한다.
- **중복 발행**: 이 노트 예제처럼 트랜잭션 없이 구성하면 두 작업이 따로 일어나 장애 시 DLT에 중복 발행될 수 있다. Kafka 트랜잭션으로 두 작업을 묶는 구성도 가능하다.
- **운영**: DLT 유입량·최초 실패 시각·원본 lag에 알림을 건다. 원인을 고친 뒤 대상을 골라 재발행한다. 이때 **같은 이벤트 ID를 유지**하고 소비 측 멱등 처리([[06 전달 보장과 중복 처리]])를 적용한다.
- **순서가 필수일 때**: DLT로 보내고 넘어가지 않는다. 문제 레코드를 고치거나 처리 방침을 정할 때까지 해당 파티션을 멈추는 정책이 필요하다.

## 4. 코드 ex) Kotlin + Spring Boot

[[06 전달 보장과 중복 처리]]의 `PaymentListener`·`PaymentService`를 그대로 쓴다. 기준은 **Spring Boot 3.x + Spring Kafka 3.x**, 문자열 payload, 수동 즉시 커밋. 06의 **무한 재시도 `KafkaErrorConfig`는 아래 설정으로 교체**한다. `payments.completed-dlt`는 원본과 **같은 파티션 수 이상**으로 미리 만든다.

```yaml
spring:
  kafka:
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
```

> [!warning] 순서가 중요한 결제 흐름
> 아래 코드는 재시도를 다 쓰면 원본을 건너뛰고 다음 이벤트를 처리한다. **같은 주문의 선후 이벤트를 반드시 순서대로 처리해야 하면 그대로 쓰지 않는다.** DLT로 보내지 않고 파티션을 멈춘 뒤 복구·알림하는 정책을 택한다.

```kotlin
import com.fasterxml.jackson.core.JsonProcessingException
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.kafka.core.KafkaTemplate
import org.springframework.kafka.listener.DefaultErrorHandler
import org.springframework.kafka.listener.DeadLetterPublishingRecoverer
import org.springframework.util.backoff.FixedBackOff

@Configuration
class KafkaErrorConfig {
    @Bean
    fun errorHandler(template: KafkaTemplate<String, String>): DefaultErrorHandler {
        val recoverer = DeadLetterPublishingRecoverer(template).apply {
            setFailIfSendResultIsError(true) // DLT 저장 실패를 성공으로 처리하지 않음
        }
        return DefaultErrorHandler(recoverer, FixedBackOff(1_000L, 2L)).apply {
            addNotRetryableExceptions(
                JsonProcessingException::class.java,
                IllegalArgumentException::class.java,
            )
            setCommitRecovered(true) // manual-immediate: DLT 저장 후 원본 오프셋 커밋
        }
    }
}
```

- `FixedBackOff(1_000L, 2L)`: ==**최초 처리 + 최대 2회 재시도**==. 그래도 실패하면 DLT로 발행한다. 파싱/입력 오류는 재시도 없이 바로 DLT로 간다.
- **커밋**: 성공한 레코드는 06처럼 리스너의 `ack.acknowledge()`가, 실패 레코드는 오류 처리기가 **DLT 발행 성공 후** 커밋한다. DLT 발행이 실패하면 다시 처리한다.
- 외부 DB와 Kafka 사이에 원자적 트랜잭션은 없다. 06의 DB 멱등 처리가 계속 필요하다.

## 실무 체크

- **일시적 오류**: 재시도 횟수·간격을 정한다. 재시도가 외부 시스템 장애를 키우지 않게 한다.
- **처리 순서**: 순서를 지켜야 하면 원본 파티션 재시도, 순서가 바뀌어도 되면 재시도 토픽을 검토한다.
- **추적·복구**: DLT 알림, 보존 기간, 담당자, 선별 재처리 절차를 정한다.
- **재처리 안전성**: 같은 이벤트 ID와 멱등 처리를 확인한다. 재발행해도 중복 가능성은 남는다.

참고: [Spring Kafka — 오류 처리·DLT 발행](https://docs.spring.io/spring-kafka/reference/3.3/kafka/annotation-error-handling.html) · [Spring Kafka — 재시도 토픽](https://docs.spring.io/spring-kafka/reference/3.3/retrytopic/how-the-pattern-works.html) · [Uber Engineering — 재처리와 DLQ](https://www.uber.com/us/en/blog/reliable-reprocessing/) · [Uber Engineering — Consumer Proxy의 DLQ](https://www.uber.com/us/en/blog/kafka-async-queuing-with-consumer-proxy/)
