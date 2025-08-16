## 7.1 여러 애그리거트가 필요한 기능
도메인 영역의 코드를 작성하다 보면 한 애그리거트로 기능을 구현할 수 없을 때가 없다. 결제 금액 계산을 예로 들어보자.
- 상품 애그리거트: 구매하는 상품 가격이 필요. 상품에 따라 배송비가 추가
- 주문 애그리거트: 상품별로 구매 개수가 필요
- 할인 쿠폰 애그리거트: 쿠폰별로 지정한 할인 금액이나 비율에 따라 주문 총 금액을 할인
- 회원 애그리거트: 회원 등급에 따라 추가 할인

실제 결제 금액을 계산하는 주체는 어떤 애그리거트인가? 주문 애그리거트가 필요한 데이터를 모두 가지고 할인 금액 계산 책임을 주문 애그리거트에 할당해볼 수 있다.
```kotlin
class Order(
	private var orderer: Orderer,
	private var orderLines: List<OrderLine>,
	private val usedCoupons: List<Coupon>,
) {
	private fun calculatePayAmounts(): Money {
		val totalAmounts = calculdateTotalAmounts()
		// 쿠폰별로 할인 금액을 구한다.
		val discount = coupons.map { calculateDiscount(it) }
						.reduce(Money(0), (v1, v2) -> v1.add(v2))
		// 회원에 따른 추가 할인을 구한다.
		val membershipDsicount = calculateDiscount(ordere.member.grade)
		// 실제 결제 금액 계산
		return totalAmounts.minus(discount).minus(membershipDiscount)
	}

	private fun calculateDiscount(coupon: Coupon): Money {
		// orderLines의 각 상품에 대해 쿠폰을 적용해서 할인 금액 계산하는 로직
		// 쿠폰의 적용 조건 등을 확인하는 코드
		// 정책에 따라 복잡한 if-else와 계산 코드
	}

	private fun calculdateDiscount(grade: MemberGrade): Money {
		// 등급에 따라 할인 금액 계산
	}
}
```

결제 금액 계산 로직이 주문 애그리거트의 책임이 맞을까? 
- 할인 정책은 주문 애그리거트의 구성 요소와는 상관이 없다.
- 그러나 정책이 변하면 주문 애그리거트를 수정해야 한다.

이처럼 하나의 애그리거트에 넣기 애매한 도메인 기능을 억지로 구현하면 안 된다. 
- 책임 범위를 넘어서는 기능 구현은 코드를 복잡하고 수정하기 어렵게 만든다.
- 도메인 개념이 애그리거트에 숨어들어 명시적으로 드러나지 않는다.

가장 손쉬운 방법은 도메인 기능을 별도 서비스로 구현하는 것이다.


## 7.2 도메인 서비스
도메인 서비스는 도메인 영역에 위치한 도메인 로직을 표현할 때 사용한다.
- 계산 로직: 여러 애그리거트가 필요한 계산 로직이나, 한 애그리거트에 넣기에는 복잡한 로직
- 외부 시스템 연동이 필요한 도메인 로직: 구현하기 위해 타 시스템을 사용해야 하는 도메인 로직

### 7.2.1 계산 로직과 도메인 서비스
- 하나의 애그리거트에 넣기 애매한 도메인 개념은 도메인 서비스를 이용해서 명시적으로 드러내면 된다.
- 도메인 영역의 애그리거트, 밸류와의 차이점은 상태 없이 로직만 구현한다는 점이다.
</br>
할인 금액 계산 로직은 다음과 같이 도메인 의미가 드러나는 용어를 타입과 메서드 이름으로 갖는다.

```kotlin
class DiscountCalculationService {
	fun calculateDiscountAmounts(
		orderLines: List<OrderLine>,
		coupons: List<Coupon>,
		grade: MemberGrade,
	): Money {
		val couponDiscount = coupons.map { calculateDiscount(it) }
						.reduce(Money(0), (v1, v2) -> v1.add(v2))
		val membershipDsicount = calculateDiscount(ordere.member.grade)
		return couponDiscount.add(membershipDiscount)
	}
	
	private fun calculateDiscount(coupon: Coupon): Money {
	}

	private fun calculdateDiscount(grade: MemberGrade): Money {
	}
}
```

할인 계산 서비스를 사용하는 주체는 애그리거트가 될 수도, 응용 서비스가 될 수도 있다. 

#### 사용 주체가 애그리거트인 경우
다음과 같이 도메인 서비스를 애그리거트에 전달한다.
```kotlin
class Order{
	fun calculateAmounts(
		disCalService: DiscountCalculdateService,
		grade: MemberGrade,
	) {
		val totalAmounts = getTotalAmounts()
		val discountAmounts = disCalService.calculateDiscountAmounts(this.orderLines, this.coupons, grade)
		this.paymentAmounts = totalAmounts.minus(discountAmounts)
	}
}
```

애그리거트 객체에 도메인 서비스를 전달하는 것은 응용 서비스의 책임이다.
```kotlin
class OrderService(
	private discountCalculateService: DiscountCalculateService,
) {
	@Transactional
	fun placeOrder(request: OrderRequest): OrderNo {
		val orderNo = orderRepository.nextId()
		val order = createOrder(orderNo, request)
		orderRepository.save(order)
		// 응용 서비스 실행 후 표현 영역에서 필요한 값 리턴
		return orderNo
	}

	fun createOrder(orderNo: OrderNo, request: OrderRequest): Order {
		val member = findMember(request.ordererId)
		val order = Order(
			orderNo = orderNo,
			orderLines = request.orderLines,
			shippingInfo = request.shippingInfo,
		)
		order.calculdateAmounts(this.discountCalculateService, member.grade)
		return order
	}
}
```

> 애그리거트가 도메인 서비스에 의존한다고 의존성 주입으로 처리하는 것은 적절하지 않다.
> 도메인 모델은 데이터와 메서드로 개념적으로 하나인 모델을 표현한다. 도메인 서비스는 데이터와는 관련이 없다. 또한, 애그리거트의 모든 기능을 제공하기 위해 도메인 서비스를 필요로 하지도 않는다.

#### 사용 주체가 응용 서비스인 경우
다음과 같이 도메인 서비스 기능을 실행할 때 애그리거트를 전달한다.
```kotlin
class TransferService {
	fun transfer(fromAcc: Account, toAcc: Account, amounts: Money) {
		fromAcc.withdraw(amounts)
		toAcc.credit(amounts)
	}
}
```

응용 서비스는 두 애그리거트를 구하고 TransferService를 이용해 계좌 이체 도메인 기능을 실행한다. 참고로 트랜잭션 처리는 응용 로직이므로 응용 서비스에서 처리할 것이다.

> 응용 서비스와 도메인 서비스의 기준을 알기 어려울 수 있다. 그때는 해당 로직이 애그리거트의 상태를 변경하거나, 상태 값을 계산하는지 검사하면 된다. 이러한 로직은 도메인 로직이므로 도메인 서비스에 들어갈 수 있다.


### 7.2.2 외부 시스템 연동과 도메인 서비스
외부 시스템이나 타 도메인과의 연동 기능도 도메인 서비스가 될 수 있다.

설문 조사 시스템과 사용자 역할 관리 시스템이 분리되어 있다고 하자. 
- 시스템간 연동은 http api 호출로 이루어질 수 있다.
- 설문 조사 도메인 입장에서 **사용자가 설문 조사 권한을 가졌는지** 확인하는 것은 도메인 로직으로 생각할 수 있다.
- 역할 관리 시스템과 **연동한다는 관점이 아니다!**
<br/>

이는 도메인 로직의 관점에서 다음과 같이 인터페이스로 표현할 수 있다. 

```kotlin
interface SurveyPermissionChecker {
	fun hasUseCreationPermission(userId: String): Boolean
}
```

응용 서비스는 이 도메인 서비스를 이용해 생성 권한을 검사한다.
```kotlin
class CreateSurbeyService(
	private val permissionChecker: SurveyPermissionChecker,
) {
	fun createSurvey(request: CreateSurveyRequest): Long {
		validate(request)
		// 도메인 서비스를 이용해서 외부 시스템 연동을 표현
		if (!permissionChecker.hasUserCreationPermission(reqquest.requestorId)) {
			throw NoPermissionException()
		}
	}
}
```

인터페이스의 구현체는 인프라 영역에 위치해 연동을 포함한 권한 검사 기능을 구현한다.

### 7.2.3 도메인 서비스의 패키지 위치
도메인 서비스는 도메인 로직을 표현한다. 따라서 애그리거트와 같은 패키지에 위치한다.
<img width="570" height="393" alt="스크린샷 2025-07-31 오후 10 24 29" src="https://github.com/user-attachments/assets/f83038d0-77b5-49a0-9804-3fc67a44f2c4" />

엔티티, 밸류와 명시적으로 구분하고 싶다면 domain.model, domain.service, domain.repository와 같이 하위 패키지를 나눌수도 있다.

### 7.2.4 도메인 서비스의 인터페이스와 클래스
도메인 서비스 로직이 고정되어 있지 않은 경우 인터페이스로 둘 수 있다. 다음처럼 실제 구현은 인프라 영역에 위치할 것이다.
<img width="590" height="253" alt="스크린샷 2025-07-31 오후 10 30 00" src="https://github.com/user-attachments/assets/b9537108-2ddc-4c9e-890f-331ce487c788" />

도메인 영역이 특정 구현에 종속되는 것을 방지하고 도메인 영역에 대한 테스트가 쉬워진다.
