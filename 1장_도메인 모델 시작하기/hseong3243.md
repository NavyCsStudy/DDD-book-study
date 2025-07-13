## 1.1 도메인이란
- **소프트웨어로 해결하고자 하는 문제 영역**을 말한다.
- 하나의 도메인은 다시 **하위 도메인으로 나눌 수 있다.** 서점 도메인에서 주문 하위 도메인은 고객의 주문을 처리한다. 혜택 하위 도메인은 쿠폰 같은 서비스를 제공한다. 

![스크린샷 2025-06-30 오후 9 03 31](https://github.com/user-attachments/assets/c4d67645-1bc8-4fe7-92c9-f6f76e15590f)

- 하나의 하위 도메인은 **다른 하위 도메인과 연동하여 완전한 기능을 제공**한다. 고객이 물건을 구매하면 주문, 결제, 배송, 혜택 하위 도메인의 기능이 엮이게 된다.

![스크린샷 2025-06-30 오후 9 05 35](https://github.com/user-attachments/assets/e964346e-010a-4992-b07e-c1962a1f8e52)

- 도메인마다 **고정된 하위 도메인이 존재하지는 않는다.**
- 하위 도메인을 어떻게 구성할지는 상황에 따라 달라진다. 기업이 대상이라면 온라인 카탈로그를 제공하고, 주문서를 받는 정도만 필요하다. 일반 고객이 대상이라면 리뷰, 배송, 회원 기능이 필요할 것이다.

## 1.2 도메인 전문가와 개발자 간 지식 공유
- 마케팅, 정산 등 **각 영역에는 전문가가 있다.**
- 개발자는 **도메인 전문가의 요구사항을 분석**하여 애플리케이션을 개발한다. 여기서 요구사항은 첫단추와 같다. 요구사항을 잘못 이해하면 제품을 만드는데 실패할 수도 있다.
- 요구사항을 올바르게 이해하기 위해서는 **도메인 전문가와 대화하는 것이 중요**하다. 또한, 개발자는 도메인 지식을 갖추고 직접 **소통할수록 전문가가 원하는 제품을 만들 가능성이 높아진다.**
- 도메인 전문가가 항상 올바른 요구사항을 주지는 않는다. 개발자는 **왜 이런 요구를 하는지 이해하고 파악**하여 진짜로 원하는 것을 찾아야 한다.

## 1.3 도메인 모델
- 도메인 모델은 **특정 도메인을 개념적으로 표현한 것**이다. 다음의 도메인 모델을 보면 주문이 어떻게 이루어지는지 파악할 수 있다.
![스크린샷 2025-06-30 오후 9 11 03](https://github.com/user-attachments/assets/20cfea91-7138-420d-94c7-7a1dfff1b7b2)

- 도메인 모델을 사용하면 **이해관계자들이 도메인을 이해하고 지식을 공유하는 데 도움이 된다.**
- 도메인 모델은 객체뿐만 아니라 상태 다이어그램으로도 모델링 할 수 있다. 하지만 중요한 것은 어떤 모델을 사용하느냐가 아니다. 이해하는데 도움이된다면 어떤 표현 방식이든 활용 가능하다.
- 도메인 모델은 도메인 자체를 이해하기 위한 개념 모델이다. 구현 모델이 개념 모델을 그대로 표현할 수는 없다. 그러나 객체 지향 언어 등을 이용해 도메인 모델과 유사하게 만들 수는 있다.

#### 하위 도메인 모델
> 같은 용어라도 **하위 도메인마다 의미가 달라질 수 있다.** **도메인에 따라 용어 의미가 결정**되므로 여러 하위 도메인을 하나의 다이어그램으로 모델링해서는 안 된다. 모델의 각 구성요소는 **특정 도메인으로 한정할 때 의미가 완전해짐**을 유의하라.

## 1.4 도메인 모델 패턴
- 일반적인 애플리케이션 아키텍처는 네 개의 영역으로 구성된다.
![스크린샷 2025-06-30 오후 9 18 35](https://github.com/user-attachments/assets/3612265a-bef2-4d18-af99-13a84e93b1a6)

- 표현 계층
	- 사용자 **요청을 처리**하고 **정보를 보여준다.**
	- 사용자는 소프트웨어 사용자이거나 외부 시스템일 수 있다.
- 응용 계층
	- 사용자가 **요청한 기능을 실행**한다.
	- 업무 로직을 직접 구현하지 않으며 도메인 계층을 조합해서 기능을 실행한다.
- 도메인 계층
	- 시스템이 제공할 **도메인 규칙을 구현**한다.
- 인프라스트럭처 계층
	- DB나 메시진 시스템과 같은 **외부 시스템과의 연동**을 처리한다.


- 도메인 계층은 도메인의 핵심 규칙을 구현한다. 주문의 경우 '출고 전에 배송지를 변경할 수 있다.', '주문 취소는 배송 전에만 할 수 있다.'라는 규칙을 구현한 코드가 위치한다.
```kotlin
class Order(
	val state: OrderState,
	val shippingInfo: ShippingInfo,
){
	fun changeShippingInfo(newShippingInfo: ShippingInfo) {
		// 출고 전에 배송지를 변경할 수 있다.
		if (!state.isShippingChangeable()) {
			throw IllegalStateException("cant change shipping in $state")
		}
		this.shippingInfo = newShippingInfo
	}
}

enum class OrderState {  
    PAYMENT_WAITING {  
        override fun isShippingChangeable() = true  
    },  
    PREPARING {  
        override fun isShippingChangeable() = false  
    },  
    SHIPPED, DELIVERING, DELIVERY_COMPLETED  
    ;  
      
    open fun isShippingChangeable() = false  
}
```

- 큰 틀에서 보면 OrderState는 Order에 속한 데이터이므로 배송지 변경 여부를 판단하는 코드를 Order로 이동할 수도 있다.

```kotlin
class Order(  
    val state: OrderState,  
    val shippingInfo: ShippingInfo,  
){  
    fun changeShippingInfo(newShippingInfo: ShippingInfo) {  
        // 출고 전에 배송지를 변경할 수 있다.  
        if (isShippingChangeable()) {  
            throw IllegalStateException("cant change shipping in $state")  
        }  
        this.shippingInfo = newShippingInfo  
    }  
  
    private fun isShippingChangeable(): Boolean =  
        setOf(OrderState.PAYMENT_WAITING, OrderState.PREPARING).contains(state)  
}  
  
enum class OrderState {  
    PAYMENT_WAITING, PREPARING, SHIPPED, DELIVERING, DELIVERY_COMPLETED  
    ;  
}
```

- 배송지 변경 가능 여부를 판단하는데 다른 정보를 함께 사용한다면 OrderState 만으로는 가능 여부를 판단할 수 없으므로 Order에서 로직을 구현해야 한다.
- 중요한 점은 **핵심 업무 규칙**을 주문 도메인 모델인 Order, OrderState에서 구현한다는 점이다. **핵심 규칙을 구현한 코드는 도메인 모델에만 위치**하여 규칙이 변경, 확장해야할 때 **다른 코드에 영향을 덜 주고 변경 내역을 모델에 반영할 수 있게 된다.**

#### 개념 모델과 구현 모델
> 개념 모델을 완벽하게 만들 수는 없다. 이해관계자들은 **개발을 하는 동안 해당 도메인을 더 잘 이해**하게 된다. **시간이 흐름에 따라 도메인 모델을 보완하거나 변경하는 일이 발생**할 수 있다. 따라서 완벽한 모델보다는 **전반적인 개요를 알 수 있는 수준으로 개념 모델을 작성**해야 한다. 전체 윤곽을 이해하는 데 집중하고, 구현하는 과정에서 개념 모델에서 **구현 모델로 점진적으로 발전**시켜 나가야 한다.


## 1.5 도메인 모델 도출
- 도메인 모델링의 기본 작업은 다음을 찾는 것이다.
  - 모델을 구성하는 핵심 구성요소
  - 규칙
  - 기능

- 주문을 예로 들어보자.
  - 한 상품을 한 개 이상 주문할 수 있다.
  - 각 상품의 구매 가격 합은 상품 가격에 구매 개수를 곱한 값이다.
```kotlin
class OrderLine(  
    val product: Product,  
    val price: Int,  
    val quantity: Int,  
) {  
    val amounts: Int = calculateAmounts()  
  
    private fun calculateAmounts(): Int = price * quantity  
}
```

- OrderLine은 한 상품(product)을 얼마에(price) 몇 개 살지 (quantity)를 담고 있고, `calculateAmounts()` 메서드로 구매 가격을 구하는 로직을 구현한고 있다.
- 한 종류 이상의 상품을 주문할 수 있으므로 Order는 한 개 이상의 OrderLine을 포함해야 한다. 또한, 총 주문 금액을 OrderLine에서 구할 수 있다.

```kotlin
class Order(orderLines: List<OrderLine>) {  
    private val orderLines: List<OrderLine>  
    private val totalAmounts: Money  
  
    init {  
        verifyAtLeastOneOrMoreOrderLines(orderLines)  
        this.orderLines = orderLines  
        this.totalAmounts = calculateTotalAmounts(orderLines)  
    }  
  
    private fun verifyAtLeastOneOrMoreOrderLines(orderLines: List<OrderLine>?) {  
        if (orderLines.isNullOrEmpty()) {  
            throw IllegalArgumentException("no OrderLine")  
        }  
    }  
  
    private fun calculateTotalAmounts(orderLines: List<OrderLine>) =  
        Money(orderLines.sumOf { it.amounts })  
}
```

- 요구사항 중에는 '주문할 때 배송지 정보를 반드시 지정해야 한다.'는 내용도 있다. 이는 Order 생성시 OrderLine 목록뿐 아니라 ShippingInfo도 함께 전달해야 함을 의미한다.

```kotlin
class Order(orderLines: List<OrderLine>, shippingInfo: ShippingInfo) {  
    private val orderLines: List<OrderLine>  
    private val totalAmounts: Money  
    private val shippingInfo: ShippingInfo  
  
    init {  
        verifyAtLeastOneOrMoreOrderLines(orderLines)  
        this.orderLines = orderLines  
        this.totalAmounts = calculateTotalAmounts(orderLines)  
        this.shippingInfo = shippingInfo  
    }  
  
    private fun verifyAtLeastOneOrMoreOrderLines(orderLines: List<OrderLine>?) {  
        if (orderLines.isNullOrEmpty()) {  
            throw IllegalArgumentException("no OrderLine")  
        }  
    }  
  
    private fun calculateTotalAmounts(orderLines: List<OrderLine>) =  
        Money(orderLines.sumOf { it.amounts })  
}
```

- 생성자에서 shippingInfo를 null 이 아닌 타입만 받을 수 있도록 규정하여 '배송지 정보 필수'라는 도메인 규칙을 구현한다.
- 이렇게 **점진적으로 만들어나간 모델은 이해관계자와 논의하는 과정에서 공유하기도 한다.**

#### 문서화
> **코드 자체도 문서화의 대상**이된다. 도메인 지식이 묻어나지 않는다면 동작은 해석할 수 있어도, 도메인 관점에서 왜 코드를 그렇게 작성했는지 이해하는데 도움이 되지 않는다. **도메인 관점에서 코드가 도메인을 잘 표현해서 가독성이 높아지고 문서로서 의미를 가진다.**

## 1.6 엔티티와 밸류
- 도출한 모델은 엔티티(Entity)와 밸류(value)로 구분할 수 있다.

### 1.6.1 엔티티
- 엔티티의 가장 큰 특징은 **식별자를 가진다**는 점이다. 식별자는 엔티티 객체마다 고유하여 서로 다른 식별자를 가진다.
![스크린샷 2025-07-01 오후 7 59 27](https://github.com/user-attachments/assets/f395fb6b-78cc-4ef0-81c2-46f37367a091)

- 주소나 배송 상태가 바뀌더라도 주문번호가 바뀌지 않는 것처럼 엔티티의 식별자는 바뀌지 않고 고유하다. 그러므로 엔티티 객체는 식별자가 같으면 같은 엔티티로 판단할 수 있다.

### 1.6.2 엔티티의 식별자 생성
- 식별자는 다음 중 한 가지 방식으로 생성한다.
  - 특정 규칙에 따라 생성
  - UUID와 같은 고유 식별자 생성기
  - 값을 직접 입력
  - 일련번호(DB의 auto increment)


- DB의 auto increment를 제외한 방법은 식별자를 먼저 만들고 엔티티 객체를 생성할 때 식별자를 전달한다. 반면, DB를 이용하게 되면 테이블에 데이터를 추가하기 전에는 식별자를 알 수 없다.
- 값을 직접 입력하는 경우에는 식별자를 중복해서 입력하지 않도록 사전에 방지하는 것이 중요하다.

### 1.6.3 밸류 타입
- ShippingInfo 클래스는 받는 사람과 주소에 대한 데이터를 갖고 있다.
![스크린샷 2025-07-01 오후 8 05 32](https://github.com/user-attachments/assets/fc036ca5-a7d5-4684-9221-1195269c876a)

- ShippingInfo는 여러 필드를 가지고 있지만 개념적으로는 받는 사람, 주소 두개의 개념을 표현한다.
- 밸류 타입은 **개념적으로 완전한 하나**를 표현할 때 사용한다. 예를 들면 받는 사람을 위한 밸류 타입은 다음과 같이 작성할 수 있다.
```kotlin
data class Receiver(  
    val name: String,  
    val phoneNumber: String,  
)
```

- Receiver는 '받는 사람'이라는 도메인 개념을 표현한다. 앞서 ShippingInfo의 필드명을 통해 받는 사람과 관련된 데이터라는 것을 유추하는 것과는 다르다. 밸류 타입을 사용하여 개념적으로 완전한 하나를 표현할 수 있다.
- 밸류 타입이 반드시 둘 이상의 데이터를 가져야하는 것은 아니다. 돈을 표현하는 Money가 그렇다.
```kotlin
data class Money(  
    val value: Int,  
) {  
    fun add(money: Money): Money = Money(this.value + money.value)      
    fun multiply(money: Money): Money = Money(this.value * money.value)  
}
```

- 또한, 돈 계산과 같이 밸류 타입을 위한 기능 역시 추가할 수 있다. 이러한 기능은 코드를 더 잘 이해할 수 있게 만들어준다.
![스크린샷 2025-07-01 오후 8 15 57](https://github.com/user-attachments/assets/db263762-923b-4139-b9b3-d3c58f6e0eed)

- 밸류 타입의 객체를 변경할 때는 변경된 데이터를 갖는 새로운 객체를 생성하는 방식을 선호한다. 이러한 객체를 불변 객체라 한다. 이러한 불변성은 객체가 여러 곳에서 참조되더라도 안전하게 사용할 수 있도록 만들어준다.

### 1.6.4 엔티티 식별자와 밸류 타입
- 식별자는 도메인에서 특별한 의미를 지니는 경우가 많다. 이 때문에 특별한 밸류 타입을 사용해서 의미가 잘 드러나도록 할 수 있다. 예를 들면 OrderNumber 밸류 타입으로 주문 번호를 표현할 수 있다. 
```kotlin
class Order(  
    val id: OrderNumber,  
)
```

- 필드 이름은 id이지만 실제 의미를 찾는 것은 어렵지 않을 것이다.

### 1.6.5 도메인 모델에 set 메서드 넣지 않기
- set 이라는 키워드는 단순히 어떠한 값을 설정한다는 의미를 가진다. 반면 change, complete 같은 키워드는 상태를 변경하고, 무언가를 완료한다는 것을 의미한다. set 이라는 키워드로는 도메인 지식을 표현할 수 없다.
```kotlin
class Order {
	fun changeShippingInfo()
	fun setShippingInfo()
}
```

### 1.7 도메인 용어와 유비쿼터스 언어
- 도메인에서 사용하는 용어를 코드에 반영하지 않으면 개발자에게 코드의 의미를 해석해야 하는 부담을 준다.
```kotlin
enum class OrderState {
	STEP1, STEP2, STEP3, ...
}

enum class OrderState {
	PAYMENT_WATING, PREPARING. SHIPPED, ...
}
```


- 위의 배송 상태와 달리 아래의 배송 상태는 코드를 도메인 용어로 해석하거나 도메인 용어를 코드로 해석하는 과정이 줄어든다. 도메인 용어를 사용해 도메인 규칙을 코드로 작성하므로 의미 변환 과정에서 발생하는 버그도 줄어든다.
- 이러한 것을 가리켜 에릭 에반스는 **유비쿼터스 언어**라고 표현했다. 이해관계자들간의 **도메인 공통의 언어**를 만들어 개발, 문서 등 모든 곳에서 같은 용어를 사용하는 것이다. 이를 통해 **소통에서 발생하는 용어의 모호함을 줄이고 개발자는 코드와 도메인 간의 불필요한 해석 과정을 줄일 수 있다.**
- 도메인에 어울리는 용어를 찾는 것은 쉽지 않은 일이다. 그러나 그러한 노력을 기울이지 않는다면 코드와 도메인은 점점 멀어진다. 그러니 도메인 용어를 찾는 시간을 아까워하지 말자.
