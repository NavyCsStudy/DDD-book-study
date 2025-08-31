# 10. 이벤트

## 10.1 시스템 간 강결합 문제
### 주문 취소 기능에서 외부 결제 시스템을 사용할 경우 발생할 수 있는 문제
```java
public class CancelOrderService {
    private Refundservice refundservice;

    @Transactional
    public void cancel(OrderNo orderNo) {
        Order order = findOrder(orderNo);
        order.cancel();

        order.refundStarted();

        try {
            refundservice.refund(order.getPaymentId());
            order.refundCompleted();
        } catch (Exception ex) {
        }
    }
}
```
1. 외부 시스템에서 에러가 발생할 경우, 트랜잭션의 롤백 여부 문제
- 반드시 롤백해야하는 것은 아님, 주문 상태를 취소로 변경하고 환불은 추후 시도
2. 성능
- 외부 시스템 응답이 느려지면 주문 취소 요청의 응답도 길어짐
3. 주문 취소를 도메인 객체에서 처리할 경우, 서로 다른 도메인 로직이 섞이게 됨
```java
public class Order {
    // 기능을 추가할 때마다 파라미터가 함께 추가되면
    // 다른 로직이 더 많이 섞이고, 트랜잭션 처리가 더 복잡해진다.
    public void cancel(RefundService refundservice, NotiService notiSvc) {
        verifyNotYetShipped();
        this.state = OrderState.CANCELED;
        // 주문+결제+통지 로직이 섞임
        // refundservice는 성공하고, notiSvc는 실패하면?
        // refundservice와 notiSvc 중 무엇을 먼저 처리하나?
    }
}
```
- 이러한 바운디드 컨텍스트 간의 강결합을 없앨 수 있는 방법 -> 인벤트

### 10.2 이벤트 개요
- 이벤트 : 과거에 벌어진 어떤 것
- 이벤트가 발생하면 그 이벤트에 반응하여 원하는 동작을 수행하는 기능을 구현해야 함

### 이벤트 관련 구성 요소
- 이벤트, 이벤트 생성 주체, 이벤트 디스패처(퍼블리셔), 이벤트 핸들러(구독자
- 이벤트 생성 주체는 엔티티, 벨류, 도메인 서비스 같은 도메인 객체
- 도메인 로직을 실행해서 상태가 바뀌면 관련된 이벤트를 발생
- 이벤트 핸들러는 이벤트를 전달받아 이벤트에 담긴 데이터를 이용해서 원하는 기능을 실행
- 디스패처 : 이벤트 생성 주체와 핸들러를 연결해 주는 것, 이벤트를 전달 받아 핸들러에게 전파
- 디스패처의 구현 방식에 따라 동기/비동기로 이벤트를 처리

### 이벤트의 구성
- 이벤트의 종류 : 클래스 이름으로 구분
- 이벤트 발생 시각
- 추가 데이터 : 이벤트와 관련된 정보

### 이벤트 용도

1. 트리거, 도메인 상태가 바뀔 때 필요한 후처리 ex) 주문 취소하면 환불을 처리
2. 다른 시스템 간의 데이터 동기화 ex) 배송지를 변경하면 외부 배송 서비스에 바뀐 배송지 정보를 전송

### 이벤트의 장점
- 서로 다른 도메인 로직이 섞이는 것을 방지
```java
public class Order {
    public void cancel() {
        verifyNotYetShipped();
        this.state = Orderstate.CANCELED;
        Events.raise(new OrderCanceledEvent(number.getNumber()));
    }
}
```
- 기능 확장에 용이 ex) 후처리가 추가될 경우, 해당 이벤트 핸들러를 구현하면 됨

## 10.3 이벤트, 핸들러, 디스패처 구현

### 이벤트 클래스
- 원하는 클래스를 이벤트로 사용하면 됨
- 이벤트는 과거에 벌어진 상태 변화나 사건을 의미하므로 클래스명에 과거 시제 사용
- 접미사로 Event 를 사용하여 이벤트로 사용하는 클래스라는 것을 명시적으로 표현
- 이벤트 클래스는 이벤트를 처리하는 데 필요한 최소환의 데이터를 포함
```java
public class OrderCanceledEvent {
    // 이벤트는 핸들러에서 이벤트를 처리하는 데 필요한 데이터를 포함한다.
    private String orderNumber;

    public OrderCanceledEvent(String number) {
        this.orderNumber = number;
    }

    public String getOrderNumber() {
        ret니rn orderNumber;
    }
}

```

### Events 클래스와 ApplicationEventPublisher
- Events 클래스 : ApplicationEventPublisher 를 사용해서 이벤트를 발생
- Events는 스프링 설정 클래스(@Configuration)를 통해 Bean으로 등록
```java
public class Events {
    private static ApplicationEventPublisher publisher;

    static void setPublisher(ApplicationEventPublisher publisher) {
        Events.publisher = publisher;
    }

    public static void raise(Object event) {
        if (publisher != null) {
            publisher.publishEvent(event);
        }
    }
}
```

### 이벤트 발생과 이벤트 핸들러
- Events.raise() 메서드를 통해 이벤트 발생
```java
public class Order {
    public void cancel() {
        verifyNotYetShipped();
        this.state = OrderState.CANCELED;
        Events.raise(new OrderCanceledEvent(number.getNumber()));
    }
}
```
- 이벤트 핸들러는 @EventListener 를 사용하여 구현
- ApplicationEventPublisher.publishEvent() 를 실행하면 @EventListener 가 붙은 메서드를 찾아 실행
```java
@EventListener(OrderCanceledEvent.class)
public void handle(OrderCanceledEvent event) {
    refundservice, refund(event.getOrderNumber());
}
```

## 10.4 동기 이벤트 처리 문제
```java

//1. 응용 서비스 코드
@Transactional // 외부 연동 과정에서 익셉션이 발생하면 트랜잭션 처리는?
public void cancel(OrderNo orderNo) {
    Order order = findOrder(orderNo);
    order.cancel(); // order.cancel()에서 OrderCanceledEvent 발생
}

// 2. 이벤트를 처리하는 코드
@Service
public class OrderCanceledEventHandler {
    @EventListener(OrderCanceledEvent.class)
    public void handle(OrderCanceledEvent event) {
        // refundservice.refund()7|- 느려지거나 익셉션이 발생하면?
        refundservice.refund(event.getOrderNumberO);
    }
}
```
- 외부 환불 기능이 느려지면 cancel() 메서드도 함께 느려짐
- 외부 서비스의 성능 저하가 내 시스템의 성능 저하로 연결됨
- 외부 시스템에서 에러가 발생할 경우 롤백 여부에 대한 고민

## 10.5 비동기 이벤트 처리
- 후처리가 즉시 발생되어야할 필요가 없다면 이벤트를 비동기로 처리 가능

### 비동기 이벤 처리를 구현하는 방법

1. 로컬 핸들러 비동기 실행
- @Async 를 사용하여 비동기로 이벤트 핸들러를 실행
- 스프링 설정 클래스에 @EnableAsync 를 사용하여 비동기 실행 기능을 활성화
- 이벤트 핸들러 메서드에 @Async 만 붙이면 됨
```java
@Async
@EventListener(OrderCanceledEvent.class)
public void handle(OrderCanceledEvent event) {
    refundService.refund(event.getOrderNumber());
}
```

2. 메시징 시스템을 이용한 비동기 구현
카프카나 래빗MQ를 사용하여 메시징 시스템을 사용하는 방법
- 이벤트가 발생하면 디스패처는 이번트를 메시지 큐에 전송
- 메시지 큐는 이벤트를 메시지 리스너에 전달
- 메시지 리스너는 알맞은 핸들러를 이용하여 이벤트를 처리
- 메시지 큐에 이벤트를 추가하는 과정과 메시지큐에서 에븐트를 읽어와 처리하는 과정이 별도의 스레드나 프로세스로 처리

3. 이벤트 저장소를 이용한 비동기 처리
이벤트를 일단 DB에 저장한 뒤 별도의 프로그램을 이용해서 이벤트 핸들러에 전달하는 방법
- 이벤트가 발생하면 핸들러는 스토리지에 이벤트를 저장
- 포워더는 주기적으로 저장소에서 이벤트를 가져와 핸들러를 실행
- 포워더는 별도의 스레드를 이용 -> 비동기로 처리
- API 를 이용해서 이벤트를 외부에 제공하는 방식도 존재
  - 외부 핸들러가 API 호출을 하여 이벤트 목록을 가져가는 방법
- 이벤트 저장소
```java
public class EventEntry {
    private Long id; // 이벤트 식별자
    private String type; // 이벤트 타입
    private String contentType; // 직렬화한 데이터 형식
    private String payload; // 이벤트 데이터 자체 페이로드
    private long timestamp; // 이벤트 시간
}

```

#### REST API 구현
```java
@RestController
public class EventApi {
    private Eventstore eventstore;
    
    public EventApi(Eventstore eventstore) {
        this.eventstore = eventstore;
    }
    
    @GetMapping("api/events")
    public List<EventEntry> list(
    @RequestParam("offset") Long offset,
    @RequestParam("limit") Long limit) {
        return eventstore.get(offset, limit);
    }
}
```
- 외부 핸들러는 자신이 처리할 이벤트를 API 를 통해 조회
- 핸들러는 언제든지 원하는 이벤트를 가져올 수 있음
- 이벤트 처리에 실패하면 다시 실패한 이벤트부터 읽오와 이벤트를 재처리

#### 포워더 구현
```java
@Scheduled(initialDelay = 1000L, fixedDelay = 1000L)
public void getAndSend() {
    long nextOffset = getNextOffset();
    List<EventEntry> events = eventstore.get(nextOffset, limitsize);
    if (!events.isEmpty()) {
        int processedCount = sendEvent(events);
        if (processedCount > 0) {

            saveNextOffset(nextOffset + processedCount);
        }
    }
}
```
- 포워더는 일정 주기로 이벤트 저장소에서 이벤트를 읽어와 이벤트 핸들러에 전달
- 이벤트 조회에 필요한 offset 값은 물리적인 저장소에 보관하여 사용

## 10.6 이벤트 적용 시 추가 고려 사항
1. 이벤트 발생 주체를 EventEntry 에 추가 여부 - 특정 주체가 발생시킨 이벤트만 조회하는 기능을 구현하려면 이벤트 발생 주체 정보를 추가해야함
2. 포워더 전송 실패를 얼마나 허용할건지 - 이벤트 재전송 횟수 제한 필요
3. 이벤트 손실 문제 - 이벤트를 비동기로 처리할 경우 이벤트 처리에 실패하면 이벤트가 유실됨
4. 이벤트 순서 문제 - 발생 순서대로 외부 시스템에 제공해야할 경우 이벤트 저장소를 사용하여 순서를 저장하는 것을 고려
5. 이벤트 재처리 - 동일한 이벤트를 처리해야할 경우, 이미 처리한 순번을 기억해두고 무시하는 방법 고려

### 이벤트 처리와 DB 트랜잭션 고려
- 이벤트 처리 실패와 트랜잭션 처리 실패를 고려해야함
- @TransactionalEventListener 를 사용하여 트랜잭션 상태에 따라 이벤트 핸들러를 실행하는 방법
```java
@TransactionalEventListener(
    classes = OrderCanceledEvent.class,
    phase = TransactionPhase .AFTER_COMMIT
)
public void handle(OrderCanceledEvent event) {
    refundservice.refund(event.getOrderNumber());
}
```