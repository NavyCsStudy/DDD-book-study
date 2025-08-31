## 10.1 시스템 간 강결합 문제

- 주문 도메인에서 환불 기능을 구현하는 경우, 환불 기능을 제공하는 도메인 서비스를 파라미터로 전달받거나 CancelOrderService 라는 응용 서비스에서 기능을 실행할 수 있다.

```java
public class Order {
		...
		public void cancel(RefundService refunService) {
				
		}		
}

public class CancelOrderService {
		private RefundService refundService;
		
		@Transactional
		public void cancel(OrderNo orderNo) {
		
		}
}		
```

- RefundService는 외부 결제 시스템이 제공하는 환불 서비스 호출
- 이때 다음과 같은 문제가 발생할 수 있다.
    1. 외부 서비스 실패 시 트랜잭션을 어떻게 처리해야 할지
        - 주문 취소 롤백
        - 주문은 취소 상태로 변경하고 환불만 나중에 재시도
    2. 성능 문제 → 외부 서비스 성능에 직접적인 영향을 받게 됨
    3. 주문 로직과 결제 로직이 섞이는 문제
- 바운디드 컨텍스트 간의 강결합이 문제의 원인
- 비동기 이벤트를 통해 두 시스템 간의 결합을 크게 낮출 수 있다.

## 10.2 이벤트 개요

- 이벤트: ‘과거에 벌어진 어떤 것’을 의미
    - ex) ‘암호를 변경했음’, ‘주문을 취소했음’
- 이벤트가 발생하면 그 이벤트에 반응하여 원하는 동작을 수행하는 기능을 구현한다.

### 10.2.1 이벤트 관련 구성요소

👉 이벤트, 이벤트 생성 주체, 이벤트 디스패처(퍼블리셔), 이벤트 핸들러(구독자)

- 도메인 모델에서 이벤트 생성 주체는 엔티티, 밸류, 도메인 서비스와 같은 ‘도메인 객체’
    - 도메인 객체는 도메인 로직을 실행해서 상태가 바뀌면 관련 이벤트를 발생시킨다.
- 이벤트 핸들러는 이벤트 생성 주체가 발생한 이벤트를 전달받아 이벤트에 담긴 데이터를 이용해서 원하는 기능을 실행한다.
- 이벤트 디스패처는 이벤트 생성 주체가 발행한 이벤트를 핸들러에 전파한다. → 동기/비동기 방식

### 102.2 이벤트의 구성

이벤트는 발생한 이벤트에 대한 정보를 담는다.

- 이벤트 종류: 클래스 이름으로 이벤트 종류를 표현
    - 이때 이벤트 이름은 과거형으로 표현한다.
- 이벤트 발생 시간
- 추가 데이터: 주문번호, 신규 배송지 정보 등 이벤트와 관련되 정보
    - 이벤트 핸들러가 작업을 수행하는 데 필요한 데이터를 담아야 한다.

### 10.2.3 이벤트 용도

- 트리거
    - 도메인의 상태가 바뀔 때 다른 후처리가 필요하면 후처리를 실행하기 위한 트리거로 이벤트 사용
- 서로 다른 시스템 간의 데이터 동기화

### 10.2.4 이벤트 장점

- 서로 다른 도메인 로직이 섞이는 것을 방지할 수 있다.
- 기능 확장도 용이
    - 핸들러 구현하면 됨
    - ex) 구매 취소 시 환불과 함께 이메일 발송을 하고 싶다면 이벤트 발송 처리 핸들러를 구현하면 된다.

## 10.3 이벤트, 핸들러, 디스패처 구현

### 10.3.1 이벤트 클래스

- 원하는 클래스를 이벤트로 사용하되, 클래스명은 과거 시제를 사용해야 한다.
    - ex) OrderCanceledEvent, OrderCanceled
- 이벤트 클래스는 이벤트를 처리하는 데 필요한 최소한의 데이터를 포함해야 한다.
- 모든 이벤트가 공통으로 갖는 프로퍼티가 존재한다면 관련 상위 클래스를 만들 수도 있다.

### 10.3.2 Events 클래스와 ApplicationEventPublisher

- 이벤트 발생과 출판을 위한 스프링이 제공하는 ApplicationEventPublisher를 사용한다.

```java
public class Events {
		private static ApplicationEventPublisher publisher; // 스프링 제공
		
		static void setPublisher(ApplicationEventPublisher publisher) {
				Events.publisher = publisher;
		}		
		
		public static void raise(Object object) {
				if (publisher != null) {
						publisher.publishEvent(event);
				}
		}				
}		
```

```java
@Configuration
public class EventsConfiguration {
		@Autowired
		private ApplicationContext applicationContext;
		
		@Bean
		public InitializeBean eventsInitializer() {
				return () -> Events.setPublisher(applicationContext);
		}
}				
```

```java
public class Order {
		public void cancel() {
				...
				// 도메인 로직 수행 후 이벤트 발행
				Events.raise(new OrderCancelEvent(number.getNumber()));
		}
}				
```

```java
@EventListener(OrderCancelEvent.class)
public void handle(OrderCancelEvent event) {
		refundService.refund(event.getOrderNumber());
}		
```

<img width="826" height="589" alt="Image" src="https://github.com/user-attachments/assets/cf9bb86d-2250-4d57-bc75-1ac440dbaf14" />

## 10.4 동기 이벤트 처리 문제

이벤트 강결합 문제는 해소했지만 여전히 외부 서비스에 영향을 받는 문제가 있다.

```java
@EventListener(OrderCancelEvent.class)
public void handle(OrderCancelEvent event) {
		refundService.refund(event.getOrderNumber());
}		
```

refundService.refund()가 외부 환불 서비스와 연동한다고 가정했을 때,

성능 저하 문제와 트랜잭션 문제가 여전히 남아 있다.

## 10.5 비동기 이벤트 처리

- 우리가 처리해야 할 이벤트 중 ‘일정 시간 안에만 후속 조치를 처리하면 되는 경우’가 있다.
- 또한 실패하더라도 일정 간격으로 재시도를 하거나 수동 처리를 해도 상관 없는 경우가 있다.
    - ex) 회원 가입 신청 시점에 이메일 발송이 실패하더라도 사용자는 이메일 재전송 요청을 이용하여 수동으로 인증 이메일을 받을 수 있다.
- ‘A 하면 B 하라’는 요구사항 중에서 ‘A 하면 최대 언제까지 B하라’로 바꿀 수 있는 것은 이벤트를 비동기로 처리하는 방식으로 구현할 수 있다.
    - B 처리는 A와 다른 스레드에서 수행할 수 있게 됨

### 10.5.1 로컬 핸들러 비동기 실행

- 이벤트 핸들러를 별도 스레드로 실행하는 방법
- `@Async` 애너테이션 활용

### 10.5.2 메시징 시스템을 이용한 비동기 구현

- Kafka나 RabbitMQ와 같은 메시징 시스템 사용
- 이벤트를 메시지 큐에 저장하는 과정과 메시지 큐에서 이벤트를 읽어와 처리하는 과정은 별도 스레드나 프로세스로 처리된다.
    - 이벤트 발생 JVM ≠ 이벤트 처리 JVM

<img width="1283" height="877" alt="Image" src="https://github.com/user-attachments/assets/b264c980-d3ec-4cab-8831-60c3e09f821a" />

- 필요하다면 이벤트 발행부와 수신부를 글로벌 트랜잭션으로 묶을 수도 있다.
    - 전체 성능 저하 문제
- 메시징 시스템은 글로벌 트랜잭션 지원과 함께 클러스터와 고가용성을 지원하기 때문에 안정적으로 메시지를 전달할 수 있다.

### 10.5.3 이벤트 저장소를 이용한 비동기 처리

- 이벤트를 DB에 저장한 뒤 별도 프로그램을 이용해서 이벤트 핸들러에 전달
- 포워더를 이용한 비동기 처리
    
    <img width="1011" height="604" alt="Image" src="https://github.com/user-attachments/assets/37bd58ac-3696-4db0-a867-db887cf63ac5" />
    
    - 이벤트가 발생하면 핸들러는 스토리지에 이벤트를 저장하고, 포워더는 주기적으로 이벹트 저장소에서 이벤트를 가져와 이벤트 핸들러를 실행한다.
    - 포워더는 별도 스레드를 이용하기 때문에 이벤트 발행과 처리가 비동기로 처리된다.
    - 도메인의 상태와 이벤트 저장소로 동일한 DB를 사용한다. → 도메인의 상태 변화와 이벤트 저장이 로컬 트랜잭션으로 처리된다.
    - 이벤트 처리가 실패할 경우 포워더는 다시 이벤트 저장소에서 이벤트를 읽어와 핸들러를 실행하면 된다.
- API를 이용해서 이벤트를 외부에 제공하는 방식
    
    <img width="1297" height="847" alt="Image" src="https://github.com/user-attachments/assets/7f7dba74-4a5d-48a9-b98d-0a19691e5bb6" />
    
    - 외부 핸들러가 API 서버를 통해 이벤트 목록을 가져간다.
    - 포워더 방식은 이벤트를 어디까지 처리했는지 추적하는 역할이 포워더에 있다면, API 방식에서는 이벤트 목록을 요구하는 외부 핸들러가 자신이 어디까지 이벤트를 처리했는지 기억해야 한다.
    - API 호출시 lastOffset 포함하여 호출

## 10.6 이벤트 적용 시 추가 고려 사항

- 첫째, 이벤트 소스를 EventEntry에 추가할지 여부
    - 특정 주체가 발생시킨 이벤트만 조회하는 기능을 구현하기 위해서는 이벤트에 발행 주체 정보를 추가해야 한다.
- 둘째, 포워더에서 전송 실패를 얼마나 허용할 것인지
    - 실패한 이벤트의 재전송 횟수 제한
- 셋째, 이벤트 손실
    - 이벤트 발생과 이벤트 저장을 한 트랜잭션으로 처리하기 때문에 트랜잭션에 성공하면 이벤트가 저장소에 보관된다는 것을 보장할 수 있는 반면, 로컬 핸들러를 이용해서 이벤트를 비동기로 처리할 경우 이벤트 처리 실패 시 이벤트를 유실하게 된다.
- 넷째, 이벤트 순서
    - 메시징 시스템 사용 기술에 따라 이벤트 발생 순서와 메시지 전달 순서가 다를 수도 있다.
- 다섯 번째, 이벤트 재처리

### 10.6.1 이벤트 처리와 DB 트랜잭션 고려

- 동기 처리
    
    <img width="1087" height="538" alt="Image" src="https://github.com/user-attachments/assets/c66fe508-1ab9-4dcf-b83c-e40292817421" />
    
    - 12번까지 성공한 후 13번에서 실패한다면, 결제는 취소됐는데 DB에는 주문이 취소되지 않은 상태로 남게 된다.
- 비동기 처리
    
    <img width="1094" height="511" alt="Image" src="https://github.com/user-attachments/assets/635aa343-383b-44c0-b622-db66020db032" />
    
    - 12번 과정에 실패하면, DB에는 주문이 취소된 상태로 남지만 결제는 취소되지 않는다.
- 이벤트 처리를 동기로 하든 비동기로 하든 이벤트 처리 실패와 트랜잭션 실패를 함께 고려해야 한다.
    - 트랜잭션이 성공할 때만 이벤트 핸들러를 실행하는 것이 좋다.
    - `@TransactionalEventListener` , `TransactionPhase.AFTER_COMMIT`
    - 이 기능을 활용하면 트랜잭션은 실패했는데 이벤트 핸들러가 실행되는 상황은 발생하지 않는다.