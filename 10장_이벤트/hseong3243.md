## 10.1 시스템 간 강결합 문제
환불 기능이 있고 이를 실행하는 주체가 주문 도메인 엔티티가 될 수 있다고 해보자. 다음처럼 도메인 서비스를 이용해 기능을 실행할 수 있다.
```kotlin
class Order {

	// 외부 서비스를 실행하기 위해 도메인 서비스를 파라미터로 전달받는다.
	fun cancel(service: RefundService) {
		verifyNotYetShipped()
		this.state = OrderState.CANCELED

		this.refundStatus = State.REFUND_STRATED
		try {
			service.refund(getPaymentId())
			this.refundStatus = State.REFUND_COMPLETED
		} catch(e: Exception) {
			???
		}
	}
}
```

아니면 응용 서비스를 통해 실행할 수도 있다.
```kotlin
class CancelOrderService {

	@Transactional
	fun cancel(orderNo: OrderNo) {
		val order = findOrder(orderNo)
		order.cancel()

		order.refundStarted()
		try {
			refundService.refund(order.paymentId)
			order.refundCompleted()
		} catch(e: Exception) {
			???
		}
	}
}
```

보통 결제 시스템은 외부에 존재하므로 RefundService는 외부에 있는 결제 시스템이 제공하는 환불 서비스를 호출한다. 이때, 발생할 수 있는 문제는 두가지다.
1. 외부 서비스가 정상이 아닐 경우 어떻게 트랜잭션을 처리하는가?
	- 환불 기능 실행 중 예외가 터지면 롤백해야하나? 아니면 커밋해야하나?
	- 롤백도 맞지만 반드시 그럴 필요는 없다.
	- 주문은 취소 상태로 변경하고 환불만 나중에 다시 시도하는 방식으로 처리할 수도 있다.
2. 성능은 어떤가?
	- 환불 처리 시스템의 응답 시간이 길어지면 대기 시간도 길어진다.
	- 외부 서비스 성능에 직접적인 영향을 받는다.

추가로 도메인 객체에 서비스를 전달하면 설계상 문제도 나타날 수 있다. 
```kotlin
class Order {
	fun cancel(service: RefundService) {
		// 주문 로직
		this.state = OrderState.CANCELD

		// 결제 로직
		this.refundStatus = State.REFUND_STARTED
		try {
			service.refund(getPaymentId())
			this.refundStatus = State.REFUND_COMPLETED
		} catch(e: Exception) {
			...
		}
	}
}
```

Order는 주문을 표현하는 도메인 객체이나 결제 도메인의 환불 관련 로직이 뒤섞였다. 환불 기능이 변경되면 Order도 영향을 받게 된다.

만약 새로운 기능이 추가된다면? 그러면 결제 로직에 더불어 통지 서비스를 위한 로직도 섞이면서 트랜잭션 처리도 복잡해진다.
```kotlin
class Order {
	fun cancel(refundService: RefundService, notiService, NotiService) {
		verifyNotYetShipped()
		this.state = OrderState.CANCELD

		// 주문 + 결제 + 통지 로직이 섞인다.
		// refundService는 성공하고, notService는 실패하면?
		// 그리고 둘 중 무엇을 먼저 처리하지?
	}
}
```

문제의 원인은 **컨텍스트간의 강결합(high coupling)** 때문이다. 주문과 결제가 강하게 결합되어 있어 주문 컨텍스트가 결제 컨텍스트에 영향을 받는 것이다.

이런 강결합을 없앨 수 있는 방법이 이벤트이다.

## 10.2 이벤트 개요
이벤트(event)라는 용어는 '과거에 벌어진 어떤 것'을 의미한다. '암호 변경 이벤트', '주문 취소 이벤트'가 그 예이다.

이벤트가 발생한다는 것은 상태가 변경됐다는 것을 의미한다. 사용자는 암호를 변경했고, 주문을 취소한 것이다.

이벤트는 발생하는 것에서 끝나지 않는다. 이벤트가 발생하면 그에 반응하여 원하는 동작을 수행하는 기능을 구현한다.
```javascript
$("#myBtn").click( function(evt) {
	alert("경고");
});
```

도메인 모델에서도 ui 컴포넌트와 같이 도메인의 상태 변경을 이벤트로 표현할 수 있다.

### 10.2.1 이벤트 관련 구성요소
도메인 모델에 이벤트를 도입하려면 네 개의 구성요소를 구현해야 한다.
- 이벤트
- 이벤트 생성 주체
- 이벤트 디스패처(퍼블리셔)
- 이벤트 핸들러(구독자)
<img width="587" height="121" alt="스크린샷 2025-08-09 오후 2 38 35" src="https://github.com/user-attachments/assets/f557e0a1-19bd-40f9-97a4-7bd2fe8586d3" />


#### 이벤트 생성 주체
- 도메인 모델에서는 엔티티, 밸류, 도메인 서비스와 같은 도메인 객체
#### 이벤트 핸들러
- 이벤트 생성 주체가 발행한 이벤트에 반응
- 이벤트에 담긴 데이터를 이용해서 원하는 기능을 실행
#### 이벤트 디스패처
- 이벤트 생성 주체와 이벤트 핸들러를 연결
- 이벤트를 전달받은 디스패처는 해당 이벤트를 핸들어게 전파

### 10.2.2 이벤트의 구성
- 이벤트 종류
  - 클래스 이름으로 이벤트 종류를 표현

- 이벤트 발생 시간
- 추가 데이터
  - 주분번호, 신규 배송지 정보 등 이벤트와 관련된 정보

배송지 변경 이벤트를 생각해보자.
```kotlin
data class ShippingInfoChangedEvent(
	val orderNumber: String,
	val timestamp: Long,
	val newShippingInfo: ShippingInfo,
)
```

클래스 이름에 'Changed'라는 과거 시제를 사용했다. 이벤트는 현재 기준으로 과거(바로 직전일지라도)에 벌어진 것을 표현하기 때문에 과거 시제를 사용한다.

이벤트 발행 주체는 Order 애그리거트이다. 배송지 정보를 변경한 뒤 이 이벤트를 발행할 것이다.
```kotlin
class Order {
	fun changeShippingInfo(newShippingInfo: ShippingInfo) {
		verifyNotYetShipped()
		setShippingInfo(newShippingInfo)
		Events.raise(ShippingInfoChangedEvent(number, newShippingInfo))
	}
}
```

ShippingInfoChnagedEvent 핸들러는 디스패처로부터 이벤트를 전달받아 필요한 작업을 수행한다.
```kotlin
class ShippingInfoChangedHandler {
	@EventListener(ShippingInfoChangedEvent::class)
	fun handle(event: ShippingInfoChangedEvent) {
		shippingInfoSynchronizter.sync(
			event.orderNumber,
			event.newShippingInfo,
		)
	}
}
```

이벤트는 핸들러가 작업을 수행하는 데 필요한 데이터를 담아야 한다. 만일 작업을 수행하기에 데이터가 부족하다면 필요한 데이터를 읽기 위해 api를 호출하거나 db에서 직접 데이터를 읽어와야 한다.
```kotlin
class ShippingInfoChangedHandler {
	@EventListener(ShippingInfoChangedEvent.class)
	fun handle(event: ShippingInfoChangedEvent) {
		val order = orderRepository.findById(event.orderNo)
		shippingInfoSynchronizer.sync(
			order.number.value,
			order.shippingInfo,
		)
	}
}
```

이벤트는 데이터를 담아야하지만 이벤트 자체와 관련 없는 데이터를 포함할 필요는 없다. 
- 배송지 변경 이벤트가 배송지 정보를 포함하는 것은 옳다.
- 그러나 배송지 변경과 관련없는 주문 상품번호와 개수를 담을 필요는 없다.

### 10.2.3 이벤트 용도
이벤트는 보통 두 가지 용도로 쓰인다.
#### 트리거(trigger)
도메인 상태가 바뀔 때 다른 후처리가 필요하면 후처리를 실행하기 위한 트리거로 사용할 수 있다.
<img width="594" height="214" alt="스크린샷 2025-08-09 오후 2 54 25" src="https://github.com/user-attachments/assets/33756573-cb4f-4fd0-ab8c-93f577bd1e34" />


#### 서로 다른 시스템 간의 데이터 동기화
배송지를 변경하면 외부 배송 서비스에 바뀐 배송지 정보를 전송해야 한다. 주문 도메인은 배송지 변경 이벤트를 발생시키고 이벤트 핸들러는 외부 배송 서비스와 배송지 정보를 동기화할 수 있다.

### 10.2.4 이벤트 장점
이벤트를 사용하면 서로 다른 도메인 로직이 섞이는 것을 방지할 수 있다.
```kotlin
class Order {
	fun cancel(refundSerivce: RefundService) {
		verifyNotYetShipped()
		this.state = OrderState.CANCELD
		Events.raise(OrderCancelEvent(this.number.value))
		}
	}
}
```

더 이상 `cancel()` 메서드에는 환불 로직이나, 환불 서비스를 사용하기 위한 파라미터도 없다. 환불 실행 로직은 주문 취소 이벤트 핸들러로 이동하였다. 주문 도메인 -> 결제 도메인 의존이 제거되었다.

이벤트 핸들러를 사용하면 기능 확장도 용이하다. 주문 취소에 대한 이베일 발송이 필요하다면 해당 작업을 처리하는 핸들러를 만들면 된다.
<img width="615" height="231" alt="스크린샷 2025-08-09 오후 2 59 40" src="https://github.com/user-attachments/assets/d4b2c09b-20f5-4c06-a3e3-14b3073c253f" />


## 10.3 이벤트, 핸들러, 디스패처 구현
이벤트와 관련된 코드는 다음과 같다.
- 이벤트 클래스: 이벤트를 표현한다.
- 디스패처: 스프링이 제공하는 `ApplicationEventPublisher`를 이용한다.
- Events: 이벤트를 발행한다. 이벤트 발행을 위해 ApplicationEventPublisher를 사용한다.
- 이벤트 핸들러: 이벤트를 수신해서 처리한다. 스프링이 제공하는 기능을 사용한다.

### 10.3.1 이벤트 클래스
이벤트는 과거의 상태 변화, 사건을 의미한다. 따라서 이름 결정 시 과거 시제를 사용해야 한다는 점을 유의하자. 
- OrderCancledEvent 처럼 명시적으로 이벤트 클래스라는 것을 표현할 수도 있다.
- OrderCanceled처럼 간결하게 과거 시제만 사용할 수도 있다.

이벤트 클래스는 이벤트를 처리하는 데 필요한 최소한의 데이터를 포함해야 한다. 주문 취소 이벤트라면 적어도 주문번호는 포함해야 관련 핸들러에 후속 처리를 할 수 있다.
```kotlin
data class OrderCanceledEvent(
	val orderNumber: String,
)
```

모든 이벤트가 공통으로 가지는 프로퍼티가 있다면 관련 상위 클래스를 만들 수도 있다.
```kotlin
abstract class Event(
	val timestamp: Long = System.currentTimeMillis(),
)
```

발생 시간이 필요한 이벤트 클래스를 이를 상속받아 구현하면 된다.
```kotlin
data class OrderCanceledEvent(
	val orderNumber: String,
): Event()
```

### 10.3.2 Events 클래스와 ApplicationEventPublisher
Events 클래스는 스프링의 ApplicationEventPublisher를 사용해본다.
```kotlin
class Events {
	companion object {
		var publisher: ApplicationEventPublisher? = null  
  
		fun raise(event: Any) {  
		    publisher?.publishEvent(event)  
		}
	}
}
```

Events 클래스에 이벤트 퍼블리셔를 전달하기 위해 스프링 설정 클래스를 다음과 같이 작성한다.
```kotlin
@Configuration  
class EventsConfiguration(  
    private val applicationContext: ApplicationContext,  
) {  
    @Bean  
    fun eventsInitializer(): InitializingBean {  
        return InitializingBean {  
            Events.publisher = applicationContext  
        }  
    }  
}
```

InitializingBean은 빈 객체를 초기화할 때 사용하는 인터페이스이다. ApplicationContext는 ApplicationEventPublisher를 상속하므로 Events 클래스 초기화에 사용할 수 있다.

### 10.3.3 이벤트 발생과 이벤트 핸들러
이벤트 발행에는 `Events.raise()` 메서드를 사용한다.

이벤트 처리에 사용할 핸들러는 스프링의 `@EventListener`를 사용한다.
```kotlin
@Service
class OrderCancelEventHandler(
	private val refundService: RefundService,
) {
	@EventListener(OrderCanceledEvent::class)
	fun handle(event: OrderCanceledEvent) {
		refundService.refund(event.orderNumber)
	}
}
```

`ApplicationEventPublisher#publishEvent()` 메서드를 실행하면 OrderCanceledEvent.class 값을 갖는 `@EventListener` 메서드를 찾아 실행한다.

### 10.3.4 흐름 정리
<img width="646" height="327" alt="스크린샷 2025-08-09 오후 4 09 54" src="https://github.com/user-attachments/assets/ba0775fb-3cce-4742-ab14-76f9c6c4f69c" />

1. 도메인 기능을 실행한다.
2. 도메인 기능은 `Events.raise()`를 이용해서 이벤트를 발생시킨다.
3. `Events.raise()`는 스프링이 제공하는 ApplicationEventPublisher를 이용해서 이벤트를 발행한다.
4. ApplicationEventPublisher는 `@EventListener(이벤트타입::class)` 애노테이션이 붙은 메서드를 찾아 실행한다.

이벤트 핸들러는 응용 서비스와 동일한 트랜잭션 범위에서 실행하고 있다. 즉, 도메인 상태 변경과 이벤트 핸들러는 같은 트랜잭션 범위에서 실행된다.

## 10.4 동기 이벤트 처리 문제
이벤트를 사용하여 강결합 문제는 해소되었다. 그러나 여전히 외부 서비스에 영향을 받고 있다.
```kotlin
@Transactional
fun cancel(orderNo: OrderNo) {
	val order = findOrder(orderNo)
	order.cancel()
}

@EventListener(OrderCanceledEvent::class)
fun handle(event: OrderCanceledEvent) {
	refundService.refund(event.orderNumber)
}
```

만약 외부 환불 기능이 갑자기 느려지면 이벤트를 발행한 `cancel()` 메서드도 함께 느려진다.

만약 핸들러 쪽에서 예외가 발생하면 전체 트랜잭션이 롤백될 것이다. 하지만 외부 환불 서비스 실행에 실패했다고 전체 트랜잭션이 롤백할 필요가 있을까? 구매 취소만 처리하고 실패한 환불은 재처리하거나 수동으로 처리할 수도 있다.

## 10.5 비동기 이벤트 처리
우리가 구현해야 할 것 중에서 'A 하면 이어서 B 하라'는 내용을 담고 있는 요구사항은 실제로 'A 하면 최대 언제까지 B 하라'인 경우가 많다. 즉, **일정 시간 안에만 후속 조치를 처리하면 되는 경우가 적지 않다.** B를 하는데 실패하더라도 일정 간격으로 재시도하거나 수동 처리를 해도 상관없는 경우가 있다. 회원 가입 시 인증 메일 발송이 실패해도 사용자는 재전송을 요청하여 수동으로 인증 메일을 다시 받아볼 수 있다.

'A 하면 B 하라'는 요구사항에서 **'A 하면'은 이벤트로 볼 수도 있다.** 따라서 이벤트 핸들러에서 'B'를 수행할 수 있다.

'A 하면 이어서 B 하라'는 요구사항 중에서 'a 하면 최대 언제까지 b 하라'로 바꿀 수 있는 요구사항은 이벤트를 비동기로 처리하는 방식으로 구현할 수 있다.
### 10.5.1 로컬 핸들러 비동기 실행
이벤트 핸들러를 별도 스레드로 실행하여 비동기 처리할 수 있다. 스프링 `@Async`를 사용하면 손쉽게 가능하다.
- `@EnableAsync`를 사용해서 비동기 기능을 활성화한다.
- 이벤트 핸들러 메서드에 `@Async`를 붙인다.
### 10.5.2 메시징 시스템을 이용한 비동기 구현
카프카나 rabbitMQ와 같은 메시징 시스템을 사용할 수도 있다. 메시지 큐는 이벤트를 메세지 리스너에게 전달하고, 리스너는 알맞은 이벤트 핸들러를 이용해서 이벤트를 처리한다. 이때 이벤트를 메시지 큐에 저장하는 과정과 읽어와 처리하는 과정은 별도 스레드나 프로세스로 처리된다.
<img width="592" height="371" alt="스크린샷 2025-08-09 오후 4 59 58" src="https://github.com/user-attachments/assets/b1f5b0c7-810d-47fd-bfd0-721e34d57bf5" />

필요하다면 이벤트를 발생시키는 도메인 기능과 메시지 큐에 이벤트를 저장하는 절차를 **하나의 트랜잭션으로 묶어야 한다.** 도메인 기능 실행 결과를 db에 반영하고 발생한 이벤트를 메시지 큐에 저장하는 것을 **같은 트랜잭션 범위에서 실행하려면 글로벌 트랜잭션이 필요**하다.

메세지 큐를 사용하면 보통 이벤트 발생 주체와 이벤트 핸들러가 별도 프로세스에서 동작한다.

rabbitmq처럼 많이 쓰이는 메시징 시스템은 글로벌 트랜잭션 지원과 함께 클러스터와 고가용성을 지원하기 때문에 안정적으로 메세지를 전달할 수 있는 장점이 있다.
반면, 카프카는 글로벌 트랜잭션을 지원하진 않지만 다른 메시징 시스템에 비해 높은 성능을 보여준다.

### 10.5.3 이벤트 저장소를 이용한 비동기 처리
이벤트를 비동기로 처리하는 다른 방법은 이벤트를 db에 저장한 뒤 별도 프로그램을 이용해서 이벤트 핸들러에 전달하는 것이다. 이는 트랜잭션 아웃박스라는 패턴으로 불린다.
<img width="604" height="350" alt="스크린샷 2025-08-13 오후 8 20 46" src="https://github.com/user-attachments/assets/cd643396-fdca-470d-b887-4338fb902f6e" />

#### 이벤트 핸들러
- 스토리지에 이벤트를 저장한다.
#### 포워더
- 주기적으로 이벤트 저장소에서 이벤트를 가져와 이벤트 핸들러를 실행한다.
- 별도 스레드를 이용하기 때문에 이벤트 발행과 처리가 비동기로 처리된다.

이 방식은 도메인의 상태와 이벤트 저장소로 동일한 db를 사용한다. 즉, **도메인의 상태 변화와 이벤트 저장이 로컬 트랜잭션으로 처리**된다. 이벤트를 물리적 저장소에 보관하기 떄문에 핸들러가 이벤트 처리에 실패할 경우 포워더는 다시 이벤트 저장소에서 이벤트를 읽어와 핸들러를 실행하면 된다.

이벤트 저장소를 이용한 두 번째 방법은 이벤트를 외부에 제공하는 api를 사용하는 것이다.
<img width="615" height="391" alt="스크린샷 2025-08-13 오후 8 24 40" src="https://github.com/user-attachments/assets/d86bb33b-7b05-4919-a90a-0b744cde57d8" />


#### api
- 포워더 방식과 달리 외부 핸들러가 api 서버를 통해 이벤트 목록을 가져간다.
- 포워더 방식이 이벤트를 어디까지 처리했는지 추적하는 역할이 포워더에 있다면 api 방식에서는 이벤트 목록을 요구하는 외부 핸들러가 자신이 어디까지 이벤트를 처리했는지 기억해야 한다.

#### 이벤트 저장소 구현
<img width="644" height="421" alt="스크린샷 2025-08-13 오후 8 26 28" src="https://github.com/user-attachments/assets/2d05d1bc-de75-4d1c-b4b8-8b5d8d112d47" />

- EventEntity
	- 이벤트 저장소에 보관할 데이터
	- id(이벤트 식별자), type(이벤트 타입), contentType(직렬화한 데이터 형식), payload(이벤트 데이터 자체), timestamp(이벤트 시간)
- EventStore
	- 이벤트를 저장하고 조회하는 인터페이스를 제공한다.
- JdbcEventStore
	- jdbc를 이용한 EventStore 구현 클래스
- EventApi
	- rest api를 이용해서 이벤트 목록을 제공하는 컨트롤러

이벤트 엔티티 클래스는 다음과 같다. 이벤트 데이터를 정의한다.
```kotlin
data class EventEntity(
	val id: Long,
	val type: String,
	val contentType: String,
	val payload: String,
	val timestamp: Long = System.currentTimeMillis(),
)
```

EventStore는 이벤트 객체를 직렬화해서 payload에 저장한다. json으로 직렬화한다면 contentType 값으로 'application/json'을 갖는다.

이벤트 스토어 인터페이스는 다음과 같다.
```kotlin
interface EventStore {
	fun save(event: Object)
	fun get(offset: Long, limit: Long): List<EventEntity>
}
```

이벤트는 과거에 벌어진 사거이므로 데이터가 변경되지 않는다. 때문에 이벤트 추가, 조회 기능만 제공한다.

구현체는 다음과 같다.
<img width="678" height="493" alt="스크린샷 2025-08-13 오후 8 32 39" src="https://github.com/user-attachments/assets/cdb7921e-b962-48a0-8f77-86aa1fff9a22" />
<img width="675" height="957" alt="스크린샷 2025-08-13 오후 8 33 11" src="https://github.com/user-attachments/assets/3d2a05aa-8f24-444c-9ae2-ea3f5ad57a0a" />
<img width="675" height="226" alt="스크린샷 2025-08-13 오후 8 33 20" src="https://github.com/user-attachments/assets/1c79576c-ffac-4da1-9a06-3c9852808a9c" />

테이블의 ddl은 다음과 같다.
<img width="695" height="223" alt="스크린샷 2025-08-13 오후 8 34 04" src="https://github.com/user-attachments/assets/38b1f0d0-5469-493b-a486-c914a0bb1155" />

#### 이벤트 저장을 위한 이벤트 핸들러 구현
```kotlin
class EventStoreHandler(
	private val eventStore: EventStore,
) {
	@EventListener(Event.class)
	fun handle(event: Event) {
		this.eventStore.save(event)
	}
}
```

#### rest api 구현
이벤트를 수정하는 기능이 없으므로 rest api도 단순 조회 기능만 존재한다.
```kotlin
class EventController(
	private val eventStore: EventStore,
) {
	@GetMapping("/api/events")
	fun list(
		@RequestParam("offset") offset: Long,
		@RequestParam("limit") limit: Long,
	): List<EventEntity> {
		return this.eventStore.get(offset, limit)
	}
}
```

api를 사용하는 클라이언트는 일정 간격으로 다음 과정을 실행한다.
1. 가장 마지막에 처리한 데이터의 offset인 lastOffset을 구한다. 저장한 lastOffset이 없으면 0을 사용한다.
2. 마지막에 처리한 lastOffset을 offset으로 사용해서 api를 실행한다.
3. api 결과로 받은 데이터를 처리한다.
4. offset + 데이터 개수를 lastOffset으로 저장한다.

마지막에 처리한 lastOffset을 저장하는 이유는 같은 이벤트를 중복해서 처리하지 않기 위함이다.
<img width="550" height="280" alt="스크린샷 2025-08-13 오후 8 48 25" src="https://github.com/user-attachments/assets/274823f8-952b-4c5f-ba38-2c925e958e32" />

클라이언트 api를 이용해서 언제든지 원하는 이벤트를 가져올 수 있기 때문에 이벤트 처리에 실패하면 다시 실패한 이벤트부터 읽어와 이벤트를 재처리할 수 있다. api 서버에 장애가 발생한 경우에도 주기적으로 재시도를 해서 api 서버가 살아나면 이벤트를 처리할 수 있다.

#### 포워더 구현
포워더는 일정 주기로 이벤트 스토어에서 이벤트를 읽어와 이벤트 핸들러에 전달하면 된다. api 방식과 마찬가지로 마지막에 전달한 이벤트의 offset을 기억해 두었다가 다음 조회 시점에 마지막으로 처리한 offset부터 이벤트를 가져오면 된다.
<img width="691" height="768" alt="스크린샷 2025-08-13 오후 9 06 14" src="https://github.com/user-attachments/assets/ba091ab7-10ad-4d92-9c02-f06a0f35e5f9" />
<img width="664" height="533" alt="스크린샷 2025-08-13 오후 9 06 26" src="https://github.com/user-attachments/assets/e811bd4c-c6cb-49c6-bd2d-470681eb65fc" />

이벤트를 주기적으로 읽어오고 전달하기 위해 스프링 스케줄러를 사용하였다. offset은 OffsetStore 인터페이스를 이용해 얻어온다.
```kotlin
interface OffsetStore {
	fun get(): Long
	fun update(nextOffset: Long)
}
```

이벤트 발행을 책임지는 EventSender 인터페이스는 다음과 같이 단순하다.
```kotlin
interface EventSender {
	fun send(event: EventEntry)
}
```

구현체에서는 외부 메시징 시스템에 이벤트를 전송하거나 원하는 핸들러에 이벤트를 전달하면 된다. 이벤트 처리 중에 익셉션이 발생하면 그대로 전파해서 다음 주기에 재처리할 수 있도록 한다.

> 자동 증가 컬럼 주의 사항
> insert 쿼리를 실행해서 자동 증가 컬럼이 증가해도 트랜잭션 커밋 전에는 증가한 값을 가진 레코드가 조회되지 않는다. 또한 커밋 시점에 따라 id 12가 먼저 반영되고 11이 나중에 반영될 수 있다. 

## 10.6 이벤트 적용 시 추가 고려 사항
#### 이벤트 소스를 EventEntry에 추가할 것인가?
- EventEntry는 이벤트 발생 주체에 대한 정보를 갖지 않는다.
- Order가 발생시킨 이벤트만 조회하기 처럼 특정 주체가 발생시킨 이벤트만 조회하는 기능을 구현할 수 없다.
#### 포워더에서 전송 실패를 얼마나 허용할 것인가?
- 포워더는 이벤트 전송에 실패하면 실패한 이벤트부터 다시 읽어와 전송을 시도한다.
- 특정 이벤트에서 전송 실패가 계속 발생하면 그 이벤트 때문에 나머지 이벤트를 전송할 수 없게 된다.
- 따라서 포워더 구현 시에는 이벤트 재전송 횟수 제한을 두어야 한다.
> 처리에 실패한 이벤트를 별도 db나 메세지 큐에 저장하기도 한다. 이는 추후 실패 이유 분석이나 후처리에 도움이 된다.
#### 이벤트 손실
- 이벤트 저장소 방식은 이벤트 발생과 저장을 하나의 트랜잭션으로 처리하기 때문에 트랜잭션이 성공하면 이벤트가 저장소에 보관된다는 것을 보장한다.
- 반면 로컬 핸들러로 이벤트를 비동기로 처리하면 이벤트 처리에 실패했을 때 이벤트는 유실된다.
#### 이벤트 손실
- 이벤트 발생 순서대로 외부 시스템에 전달해야 한다면 이벤트 저장소가 좋다.
- 이벤트 저장소는 발생 순서대로 이벤트를 제공하고 순서대로 이벤트 목록을 제공한다.
- 반면 메시징 시스템은 사용 기술에 따라 발생 순서와 메세지 전달 순서가 다를 수 있다.
#### 이벤트 재처리
- 동일 이벤트를 재처리 할 때 어떻게 할 것인가?
- 가장 쉬운 방법은 마지막 이벤트의 순번을 기억해두었다가 이미 처리한 순번이 들어오면 처리하지 않고 무시하는 것이다.
- 또는 이벤트를 멱등성 있게 처리하는 방법도 있다.
> 멱등성
> 연산을 여러 번 적용해도 결과가 달라지지 않는 성질을 말한다.
> 이벤트 핸들러가 멱등성을 가지면 이벤트 중복이 발생해도 결과적으로 동일 상태가 된다.

### 10.6.1 이벤트 처리와 db 트랜잭션 고려
이벤트를 처리할 때는 db 트랜잭션을 함께 고려해야 한다. 주문 취소, 환불 기능을 예로 들어보자.
- 주문 취소 기능은 주문 취소 이벤트를 발생시킨다.
- 주문 취소 이벤트 핸들러는 환불 서비스에 환불 처리를 요청한다.
- 환불 서비스는 외부 api를 호출해서 결제를 취소한다.
<img width="677" height="336" alt="스크린샷 2025-08-17 오후 2 27 34" src="https://github.com/user-attachments/assets/c27aab3f-5b9d-4198-aea0-fb13393bd43c" />

12번까지 성공했는데 13번에서 실패한다면 db에는 주문이 취소되지 않은 상태로 남게 된다.

이벤트를 비동기로 처리할 때도 db 트랜잭션을 고려해야 한다.
<img width="671" height="321" alt="스크린샷 2025-08-17 오후 2 28 27" src="https://github.com/user-attachments/assets/dc7fa997-c548-401b-bade-bc6a36614602" />

db 트랜잭션을 다 커밋한 뒤에 11~13번 과정이 실했되었다고 해보자. db에는 주문이 취소된 상태로 바뀌었지만 결제는 취소되지 않은 상태로 남게 된다.

이벤트 처리를 동기나 비동기 어떻게 하든 이벤트 처리 실패와 트랜잭션 실패를 함께 고려해야 한다. 둘 모두를 고려하는  것은 복잡하므로 경우의 수를 줄이는 것이 도움이 된다. 경우의 수를 줄이는 방법은 트랜잭션이 성공할 때만 이벤트 핸들러를 실행하는 것이다.

스프링은 `@TransactionalEventListener`로 트랜잭션 상태에 따라 이벤트 핸들러를 실행할 수 있게 한다.
<img width="671" height="210" alt="스크린샷 2025-08-17 오후 2 31 31" src="https://github.com/user-attachments/assets/e503eb1c-86de-4252-9a90-034830b8e591" />


`TransactionPhase.AFTER_COMMIT`을 사용하면 트랜잭션이 커밋에 성공한 뒤에 핸들러 메서드를 실행한다. 이를 사용하면 핸들러를 실행했는데 트랜잭션이 롤백되는 상황은 발생하지 않는다.

이벤트 저장소로 db를 사용해도 동일한 효과를 볼 수 있다. 이벤트 발생 코드와 이벤트 저장 처리를 한 트랜잭션에서 처리하기 때문이다.

이제 트랜잭션 실패에 대한 경우의 수는 줄었고, 이벤트 처리 실패만 고민하면 된다. 이벤트 처리 실패는 이벤트 특성에 따라 재처리 방식을 결정하면 된다.
