## 2.1 네 개의 영역
- 아키텍처 설계에서는 전형적으로 등장하는 네가지 영역이 존재

<img width="690" height="254" alt="스크린샷 2025-07-02 오후 7 19 44" src="https://github.com/user-attachments/assets/7e9aa94b-5dfa-4547-b9e7-830abbffd990" />

표현 영역 
- http 요청을 응용 영역이 필요로 하는 형식으로 변환, 응용 영역에 전달
- 응용 영역의 응답을 http 응답으로 변환하여 전송
 
응용 영역
- 사용자에게 제공해야 할 기능을 구현. '주문 등록/취소', '상품 상세 조회'와 같은 기능을 예로 들 수 있다.
- 기능을 구현하기 위해 도메인 영역의 도메인 모델을 사용
- 응용 서비스는 로직을 직접 수행하기보다 도메인 모델에 로직 수행을 위임
```kotlin
@Service  
class CancelOrderService {  
  
    @Transactional  
    fun cancelOrder(orderId: String) {  
        val order: Order = findOrderById(orderId) ?: throw OrderNotFoundException()  
        order.cancel()  
    }  
  
    private fun findOrderById(orderId: String): Order? {  
        return null  
    }  
}
```

<img width="682" height="247" alt="스크린샷 2025-07-02 오후 7 31 08" src="https://github.com/user-attachments/assets/0879a3c5-21ff-4a84-b8b2-2928201b3131" />

도메인 영역
- 도메인 모델을 구현
- 도메인의 핵심 로직을 구현. '배송지 변경', '결제 완료'와 같은 핵심 로직을 예로 들 수 있다.

인프라스트럭처 영역
- 구현 기술에 대한 것을 다룬다.
- rdbms 연동을 처리, 큐에 메시지를 전송하거나 수신하는 기능, smtp를 이용한 메일 발송 기능, http 클라이언트를 이용한 rest api 호출 등을 처리
- 논리적인 개념보다 실제 구현을 다루는 영역
- 도메인, 응용, 표현 영역은 구현 기술을 사용한 코드를 직접 만들지 않는다. 대신 인프라스트럭처 영역에서 제공하는 기능을 사용해 필요한 기능을 개발한다.

  <img width="361" height="237" alt="스크린샷 2025-07-02 오후 7 39 14" src="https://github.com/user-attachments/assets/6ba6be89-31de-4b6b-abc6-d634a7fefebc" />


## 2.2 계층 구조 아키텍처
-전체적인 아키텍처는 다음의 계층 구조를 따름

<img width="278" height="373" alt="스크린샷 2025-07-02 오후 7 41 25" src="https://github.com/user-attachments/assets/757315d1-9d87-44b9-961f-5316ac4251c3" />

- 계층 구조는 특성상 상위 -> 하위로 의존. 그러나 반대로는 의존하지 않음
- 표현, 응용, 도메인 계층이 상세한 구현 기술을 다루는 인프라 계층에 종속됨

<img width="552" height="351" alt="스크린샷 2025-07-02 오후 7 45 25" src="https://github.com/user-attachments/assets/a58b2042-db31-4968-80c8-13a3599fec0d" />

- 이로 인해 발생할 수 있는 문제 두가지
  - 첫째, 테스트하기 어렵다.응용 계층의 service가 의존하고 있는 인프라가 완벽하게 동작해야 한다.
  - 둘째, 구현 방식을 변경하기 어렵다. 구현 기술에 의존적인 코드가 추가되고, 기술을 변경하기 위해서는 코드를 변경해야 한다.

<img width="640" height="372" alt="스크린샷 2025-07-02 오후 7 47 53" src="https://github.com/user-attachments/assets/40b0e47e-96c1-413e-884e-be411cf080ec" />

## 2.3 DIP

- 고수준 모듈
  - 의미 있는 단일 기능을 제공하는 모듈
  - '가격 할인 계산'이라는 기능을 구현하기 위해 고객 정보를 구하거나, 룰을 실행하는 하위 기능이 필요
- 저수준 모듈
  - 하위 기능을 실제로 구현한 것
  - jpa를 이용해 고객 정보를 구하고, 룰을 실행하는 모듈  

<img width="538" height="228" alt="스크린샷 2025-07-02 오후 7 55 01" src="https://github.com/user-attachments/assets/4bdc5187-778a-4108-a6fc-d61e465fd5f2" />

- 고수준 모듈 -> 저수준 모듈의 의존은 테스트와 구현 변경을 어렵게 한다. 이는 DIP로 해결할 수 있다.
- 고수준 모듈에게 중요한 것은 **고객 정보를 구한다는 추상적인 사실**. service 입장에서 고객 정보를 구하는 방식, 기술을 중요하지 않음.

<img width="691" height="356" alt="스크린샷 2025-07-02 오후 8 02 04" src="https://github.com/user-attachments/assets/5e2b4873-a5d9-44e9-b563-932d0b4883cf" />

- service는 추상화된 인터페이스를 의존, 더 이상 구체적인 기술에 의존하지 않는다.

<img width="476" height="219" alt="스크린샷 2025-07-02 오후 8 07 29" src="https://github.com/user-attachments/assets/8507a03f-b843-4f94-8470-787d0b45915f" />

- DIP를 적용하여 고수준 모듈 <- 저수준 모듈로 의존성이 역전. 테스트, 기술 변경의 어려움을 해결할 수 있다.
- 의존성 주입으로 기술을 변경하고, 테스트 대역 프레임워크를 이용하거나 직접 테스트 대역을 구현하여 원활한 테스트를 수행할 수 있다.

<img width="691" height="424" alt="스크린샷 2025-07-02 오후 8 11 02" src="https://github.com/user-attachments/assets/7209d806-07d2-4cec-a89e-9abb59cf7d5b" />

### 2.3.1 DIP 주의 사항

- DIP는 단순히 클래스에서 인터페이스를 추출하는 것이 아님
- 중요한 것은 **고수준 모듈 관점에서** 하위 기능을 추상화한 인터페이스를 도출하는 것

<img width="497" height="276" alt="스크린샷 2025-07-02 오후 8 12 25" src="https://github.com/user-attachments/assets/e9084b14-509f-4869-9826-cc3a412e581d" />

<img width="496" height="353" alt="스크린샷 2025-07-02 오후 8 14 33" src="https://github.com/user-attachments/assets/6111fc10-7066-4664-8b9a-774ad109a4f2" />

### 2.3.2 DIP와 아키텍처

- DIP를 적용하면 인프라 계층이 응용, 도메인 영역에 의존하게 됨
- 인프라 영역의 변경이 응용, 도메인 영역에 대한 영향을 주지 않거나 최소화할 수 있음

<img width="616" height="416" alt="스크린샷 2025-07-02 오후 8 16 00" src="https://github.com/user-attachments/assets/42416510-cdb8-4e94-a41f-10b86e553f03" />

> DIP를 항상 적용할 필요는 없다. 구현 기술에 의존하는 편이 효과적일 때도 있고, 추상화 대상이 잘 떠오르지 않을 때도 있다. 그럴 때는 무조건 DIP를 적용하지 말고, DIP의 이점을 얻는 수준에서 적용 범위를 검토해보자.

## 2.4 도메인 영역의 주요 구성요소

- 도메인 영역의 모델은 도메인의 주요 개념을 표현, 핵심 로직을 구성

#### 엔티티(entity)
- 고유의 식별자를 갖는 객체
- 자신의 라이프 사이클을 갖는다.
- 도메인의 고유한 개념을 표현
- 도메인 모델의 데이터를 포함하며 해당 데이터와 관련된 기능을 함께 제공

#### 밸류(value)
- 고유한 식별자를 갖지 않는 객체
- 주로 개념적으로 하나인 값을 표현할 때 사용
- 엔티티나 다른 밸류 타입의 속성으로 사용할 수 있다.

#### 애그리거트(aggreate)
- 연관된 엔티티와 밸류 객체를 개념적으로 하나로 묶은 것
- Order 엔티티, OrderLine 밸류, Orderer 밸류 객체를 '주문' 애그리거트로 묶을 수 있을 것이다.

#### 리포지터리(repository)
- 도메인 모델의 영속성을 처리

#### 도메인 서비스(domain service)
- 특정 엔티티에 속하지 않은 도메인 로직을 제공
- 도메인 로직이 여러 엔티티와 밸류를 필요로 하면 도메인 서비스에서 로직을 구현
- '할인 금액 계산'은 상품, 쿠폰, 회원 등급 등 다양한 조건을 이용해 구현해야 하는 로직이 있을 것이다.

### 2.4.1 엔티티와 밸류
#### 엔티티
- 도메인 모델의 엔티티는 데이터와 함께 도메인 기능을 함께 제공
- DB 모델의 엔티티와는 다르다.

```kotlin
class Order {
    // 주문 도메인 모델의 데이터  
    var number: OrderNumber
      private set
    var orderer: Orderer
      private set
    var shippingInfo: ShippingInfo
      private set

    // 도메인 모델 엔티티는 도메인 기능도 함께 제공  
    fun changeShippingInfo(shippingInfo: ShippingInfo) {  
    }  
}

- 단순한 데이터 구조체가 아닌 데이터와 기능을 함께 제공하는 객체
- 도메인 관점에서 기능을 구현하고 기능 구현을 캡슐화하여 임의로 데이터가 변경되는 것을 막음

#### 밸류
- 두 개 이상의 데이터가 개념적으로 하나인 경우 엔티티는 밸류 타입을 이용해 표현할 수 있음
```kotlin
/**  
 * 주문자  
 */  
data class Orderer(  
    /**  
     * 이름  
     */  
    val name: String,  
    /**  
     * 이메일  
     */  
    val email: String,  
)
```

- DB 모델의 엔티티는 밸류 타입을 제대로 표현하기 어려움
- prefix를 이름으로 붙이거나, 별도 테이블로 관리해야 함
  
<img width="498" height="317" alt="스크린샷 2025-07-03 오후 8 03 26" src="https://github.com/user-attachments/assets/d0824f19-3cbc-43e4-ac4f-9c15bf816f34" />

- 왼쪽은 주문자라는 개념이 드러나지 않고 개별 테이터만 드러남. 오른쪽은 테이블의 엔티티에 가까워 밸류 타입의 의미가 드러나지 않음
- 밸류는 불변으로 구현할 것을 권장. 즉, 엔티티의 밸류 타입 데이터 변경은 객체 자체를 완전히 교체한다는 것을 의미함

```kotlin
class Order {
    // 주문 도메인 모델의 데이터  
    var number: OrderNumber
      private set
    var orderer: Orderer
      private set
    var shippingInfo: ShippingInfo
      private set

    // 도메인 모델 엔티티는 도메인 기능도 함께 제공  
    fun changeShippingInfo(shippingInfo: ShippingInfo) {  
        this.checkShippingInfoChangeable()  
        this.shippingInfo = shippingInfo  
    }  
}
```

### 2.4.2 애그리거트

- 도메인이 커지면 도메인 모델도 커지면서 많은 엔티티와 밸류가 출현

<img width="593" height="306" alt="스크린샷 2025-07-03 오후 8 06 43" src="https://github.com/user-attachments/assets/a2ab94ad-88a5-40c3-bb65-b4f15965f7e4" />

- 도메인 모델이 복잡해지면 전체 구조가 아닌 한 개의 엔티티와 밸류에만 집중하는 상황이 발생
- 도메인 모델은 상위 수준에서 모델을 볼 수 있어야 전체 모델의 관계와 개별 모델을 이해하는 데 도움이 됨. 이를 이해하는 데 도움이 되는 것이 애그리거트
- 애그리거트는 관련 객체를 하나로 묶은 군집
  - 주문 (상위 개념의 모델)
    - 주문 (하위 개념의 모델)
  	- 배송지 정보
  	- 주문자
  	- 주문 목록
  	- 총 결제 금액

<img width="544" height="309" alt="스크린샷 2025-07-03 오후 8 11 03" src="https://github.com/user-attachments/assets/3d2cb811-314d-4b2e-9f2d-ab595764f073" />

- 애그리거트 간의 관계로 도메인 모델을 이해하고, 구현하게 되어 큰 틀에서 도메인 모델을 관리할 수 있게 됨

#### 루트 엔티티
- 애그리거트는 군집에 속한 객체를 관리하는 루트 엔티티를 가짐
- 애그리거트에 속해 있는 엔티티와 밸류 객체를 이용해 애그리거트가 구현해야할 기능을 제공
- 클라이언트 코드는 루트 엔티티를 통해 간접적으로 애그리거트 내의 다른 엔티티나 밸류 객체에 접근. 이는 내부 구현을 숨겨 애그리거트 단위로 구현을 캡슐화할 수 있게 도움.

<img width="523" height="412" alt="스크린샷 2025-07-03 오후 8 14 12" src="https://github.com/user-attachments/assets/c74ce001-bf41-466e-b3f4-f6a3ff695568" />

```kotlin
class Order {  

    fun changeShippingInfo(shippingInfo: ShippingInfo) {  
        checkShippingInfoChangeable()  // 배송지 변경 가능 여부 확인  
        this.shippingInfo = shippingInfo  
    }  
      
    private fun checkShippingInfoChangeable() {  
        // 배송지 정보를 변경할 수 있는지 여부를 확인하는 도메인 규칙 구현  
    }  
}
```

- 배송지를 변경하기 위해서는 항상 루트 엔티티를 통해야하므로 Order가 구현한 도메인 로직을 따름

### 2.4.3 리포지터리
- 도메인 객체를 물리적인 저장소에 보관하기 위한 도메인 모델
- 요구사항에서 도출되는 엔티티, 밸류와 달리 구현을 위한 도메인 모델
- 리포지터리는 애그리거트 단위로 도메인 객체를 저장하고 조회하는 기능을 정의

```kotlin
interface OrderRepository {  
    fun findByNumber(number: OrderNumber): Order  
    fun save(order: Order)  
    fun delete(order: Order)  
}

- 루트 엔티티는 애그리거트에 속한 모든 객체를 포함하므로 애그리거트 단위로 저장, 조회함
- 도메인 모델을 사용하는 코드는 리포지터리를 통해 도메인 객체를 구한 뒤 기능을 실행함
```kotlin
class CancelOrderService(  
    private val orderRepository: OrderRepository,  
) {  
  
    fun cancel(number: OrderNumber) {  
        val order = this.orderRepository.findByNumber(number)  
            ?: throw NoOrderException(number)  
        order.cancel()  
    }  
}
```

- 도메인 모델 관점에서 Repository는 객체를 영속화하는 데 필요한 기능을 추상화한 고수준 모듈에 속함
- 응용 서비스는 의존성 주입을 통해 실제 구현체에 접근

<img width="692" height="451" alt="스크린샷 2025-07-03 오후 8 23 45" src="https://github.com/user-attachments/assets/fa69ef72-92f7-4ea4-ad43-038c39c98821" />

## 2.5 요청 처리 흐름
- 표현 영역은 사용자가 전송한 데이터 형식이 올바른지 검사
- 문제가 없다면 데이터를 이용해서 응용 서비스에 기능 실행을 위임

<img width="668" height="381" alt="스크린샷 2025-07-03 오후 8 26 20" src="https://github.com/user-attachments/assets/5bda7609-5305-4114-a3e0-1dad52e1c43c" />

- 응용 서비스는 도메인 모델을 이용해 기능을 구현
- 도메인의 상태 변경이 저장소에 올바르게 반영될 수 있도록 트랜잭션을 관리해야 함
```kotlin
class CancelOrderService(  
    private val orderRepository: OrderRepository,  
) {  
  
    @Transactional  
    fun cancel(number: OrderNumber) {  
        val order = this.orderRepository.findByNumber(number)  
            ?: throw NoOrderException(number)  
        order.cancel()  
    }  
}

## 2.6 인프라스트럭처 개요
- 인프라 영역은 표현, 응용, 도메인 영역을 지원. 영속성 처리, 트랜잭션, rest 클라이언트 등 다른 영역에서 필요로 하는 기술을 지원.
- DIP를 적용하면 유연한 기술 변경, 테스트가 가능
- 상황에 따라서는 `@Transactional` 같은 애노테이션을 이용하는게 더 편리할 수 있음

## 2.7 모듈 구성
- 아키텍처의 각 영역은 별도 패키지에 위치

<img width="493" height="392" alt="스크린샷 2025-07-03 오후 8 36 50" src="https://github.com/user-attachments/assets/898d9b53-07bf-448a-be8a-bc6d41bfaa77" />

- 도메인이 크면 하위 도메인별로 별도 패키지를 구성

<img width="566" height="438" alt="스크린샷 2025-07-03 오후 8 37 14" src="https://github.com/user-attachments/assets/e84f8581-24ee-4f28-bf06-554230371561" />

- 도메인 모듈은 도메인에 속한 애그리거트를 기준으로 다시 패키지를 구성

<img width="562" height="426" alt="스크린샷 2025-07-03 오후 8 38 28" src="https://github.com/user-attachments/assets/c1e2c180-4226-4173-acb3-675e83d60d91" />

- 모듈 구조에 규칙은 없음
- 한 패키지에 타입이 몰려서 코드를 찾을 때 불편한 정도만 아니면 됨
