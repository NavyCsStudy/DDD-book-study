## 3.1 애그리거트
- **상위 수준 개념**을 이용해서 **전체 모델을 정리**하는 것은 전반적인 관계를 이해하는 데 도움이 됨

<img width="560" height="269" alt="스크린샷 2025-07-05 오전 11 02 27" src="https://github.com/user-attachments/assets/b0ad44c6-2c77-4ee7-bdcf-fe62f296da91" />


- 다음은 상위 수준의 개념을 개별 객체로 풀어놓은 것. 상위 수준의 개념 이해가 없으면 파악하기 어려움.

  <img width="563" height="349" alt="스크린샷 2025-07-05 오전 11 03 42" src="https://github.com/user-attachments/assets/423b91e7-7d84-4e80-9357-23b3fb8daefd" />

- 도메인 객체 모델이 복잡해지면 개별 구성요소 위주로 모델을 이해하고, 전반적인 구조나 큰 수준에서 도메인 간 관계를 파악하기 어려워짐.
- 즉, 코드 변경과 확장이 어려워짐. 상위 수준에서 모델이 어떻게 엮여있는지 알아야 전체 모델을 망가뜨리지 않으면서 추가 요구사항을 모델에 반영할 수 있음.
- 세부적인 모델만 이해한 상태로는 코드 변경을 최대한 회피하는 쪽으로 요구사항을 협의하게 됨
- 애그리거트는 관련된 객체를 하나의 군으로 묶어줌. 이를 통해 복잡한 도메인을 상위 수준의 모델에서 조망하여 이해하기 쉽게 관리할 수 있음.

<img width="619" height="401" alt="스크린샷 2025-07-05 오전 11 16 49" src="https://github.com/user-attachments/assets/f81024a7-2a05-48a8-b17e-cdb4bbf3efc4" />

- 애그리거트는 일관성을 관리하는 기준.
  - 애그리거트 단위로 **일관성을 관리**하기 때문에 복잡한 도메인을 단순한 구조로 만들어줌.
- 하나의 애그리거트에 속한 객체는 유사하거나, **동일한 라이프 사이클**을 가짐.
  - 도메인 규칙에 따라 최초 생성 시점에 일부 객체를 만들 필요가 없는 경우도 있지만 애그리거트에 속한 구성요소는 대부분 함께 생성하고 제거.
- 애그리거트는 경계를 가짐.
  - 애그리거트는 독립된 객체 군이며 자기 자신을 관리할 뿐 다른 애그리거트를 관리하지 않음. 경계를 설정할 때 기본이 되는 것은 도메인 규칙과 요구사항. 도메인 규칙에 따라 함께 생성되는 구성요소는 한 애그리거트에 속할 가능성이 높음.

- 예를 들어, 상품이 리뷰를 가진다고 하여 하나의 애그리거트에 속하지는 않음

<img width="684" height="313" alt="스크린샷 2025-07-05 오전 11 25 55" src="https://github.com/user-attachments/assets/2ee7affe-b98b-40a4-b757-3d5b1a4e1fd7" />

- 도메인에 대한 경험이 생기고 도메인 규칙을 제대로 이해할수록 애그리거트의 실제 크기는 줄어듬

## 3.2 애그리거트 루트

- 애그리거트는 여러 객체로 구성됨. 때문에 한 객체만 상태가 정상이면 안 됨. 도메인 규칙을 지키려면 애그리거트에 속한 모든 객체가 정상 상태를 가져야 함.
- 애그리거트에 속한 모든 객체가 일관된 상태를 가지려면 전체를 관리할 주체가 필요. 이 책임을 지는 것이 애그리거트의 루트 엔티티.

<img width="370" height="343" alt="스크린샷 2025-07-05 오후 4 37 47" src="https://github.com/user-attachments/assets/6597e4e0-fc91-4c71-8b3c-ea784d5be056" />

### 3.2.1 도메인 규칙과 일관성

- 애그리거트 루트의 핵심 역할은 애그리거트의 일관성이 깨지지 않도록 하는 것
- 이를 위해 애그리거트 루트는 애그리거트가 제공해야 할 도메인 기능을 제공

```kotlin
class Order(  
    var shippingInfo: ShippingInfo,  
    var state: OrderState,  
) {  
    // 애그리거트 루트는 도메인 규칙을 구현한 기능을 제공한다.  
    fun changeShippingInfo(shippingInfo: ShippingInfo) {  
        verifyNotYetShipped()  
        this.shippingInfo = shippingInfo  
    }  
  
    private fun verifyNotYetShipped() {  
        if(setof(OrderState.PAYMENT_WAITING && OrderSate.PREPARING).contains(this.state)) {  
            throw IllegalStateException("already shipped")  
        }  
    }  
}
```

- 애그리거트 외부에서 애그리거트에 속한 객체를 직접 변경하면 안 됨. 이는 애그리거트가 강제하는 규칙을 적용하지 못해 모델의 일관성을 깨는 원인이 됨.

```kotlin
val si = order.getShippingInfo()
si.address = newAddress  // 결제 대기, 준비중일때는 배송지 정보를 변경할 수 없다는 핵심 규칙을 우회한다.
```

- 일관성을 지키키 위한 상태 로직을 응용 서비스에 구현할 수도 있으나, 동일한 검사 로직이 여러 응용 서비스에 중복으로 구현된다면 유지 보수성이 떨어짐

- 불필요한 중복을 피하고, 애거리거트 루트를 통해서만 도메인 로직을 구현하게 만들려면 다음 규칙을 지켜야 함.
  - 필드만 변경할 뿐인 setter를 public 으로 만들지 않는다.
  - 밸류 타입은 불변으로 한다.

- public setter는 도메인의 의도를 표현하지 못하고 도메인 로직을 응용, 표현 계층으로 분산시킴
- 밸류 객체의 값을 변경할 수 없으면 애그리거트 루트에서 밸류 객체를 구해도 외부에서 상태를 변경할 수 없음. 이는 애그리거트의 일관성을 지켜줌.

```kotlin
class Order(shippingInfo: ShippingInfo, orderState: OrderState) {  
    var shippingInfo: ShippingInfo = shippingInfo  
        private set  
    var orderState: OrderState = orderState  
        private set  
  
    // set 메서드의 접근 허용 범위는 private 이다.
    fun changeShippingInfo(newShippingInfo: ShippingInfo) {  
	    // 밸류가 불변이면 새로운 객체를 할당해서 값을 변경해야 한다.
	    // 불변이므로 this.shippingInfo.address = newShippingInfo.address와 같은
	    // 코드를 사용할 수 없다.
        verifyNotYetShipped()  
        this.shippingInfo = newShippingInfo  
    }  
  
    private fun verifyNotYetShipped() {  
        if (setof(OrderState.PAYMENT_WAITING && OrderSate.PREPARING).contains(this.state)) {  
            throw IllegalStateException("already shipped")  
        }  
    }  
}
```

### 3.2.2 애그리거트 루트의 기능 구현

- 애그리거트 루트는 내부의 다른 객체를 조합해 기능을 완성. 단순히 구성요소의 상태만 참조하지 않고, 기능 실행을 위임하기도 함.

```kotlin
class Order(orderLines: OrderLines) {  
    var orderLines: OrderLines = orderLines  
        private set  
        
    fun changeOrderLines(newLines: List<OrderLine>) {  
        orderLines.changeOrderLines(newLines)  
        this.totalAmounts = orderLines.getTotalAmounts()  
    }  
}

class OrderLines(lines: List<OrderLine>) {  
    var lines: List<OrderLine> = lines  
        private set  
        
    fun getTotalAmounts(): Money {
    }  
      
    fun changeOrderLines(newLines: List<OrderLine>) {  
        this.lines = newLines  
    }  
}
```

- 위 코드에서 OrderLines는 changeOrderLines()메서드를 제공. 이 경우 루트 외부에서 해당 기능을 실행할 수 있음.

```kotlin
val lines = order.orderLines  
lines.changeOrderLines(newOrderLines)  // order의 totalAmounts 값이 orderLines와 일치하지 않게 된다.
```

- 이런 버그가 생기지 않도록 하려면 애그리거트 외부에서 OrderLine 목록을 변경할 수 없도록 OrderLines를 불변으로 구현하면 됨. 불변으로 구현할 수 없다면 변경 범위를 protected로 제한하여 외부에서 기능을 실행할 수 없도록 만들 수도 있음.

```kotlin
class Order(orderLines: OrderLines) {  
    var orderLines: OrderLines = orderLines  
        private set  
  
    fun changeOrderLines(newLines: OrderLines) {  
        this.orderLines = newLines  // 기존에 기능의 실행을 위임하던 것에서 객체 자체를 교체하는 것으로 변경
        this.totalAmounts = orderLines.getTotalAmounts()  
    }  
}
```

### 3.2.3 트랜잭션 범위

- 트랜잭션 범위는 작을수록 좋음.
  - 한 개의 테이블을 수정하면 충돌을 막기 위해 잠그는 대상이 한 개 테이블의 한 행으로 한정되지만, 세 개의 테이블을 수정하면 잠금 대상이 더 많아짐
  - 동일하게 한 트랜잭션에서 두 개 이상의 애그리거트를 수정하면 충돌이 발생할 가능성은 더 높아짐

- 애그리거트는 서로 독립적이어야 함.
  - 한 애그리거트가 다른 애그리거트의 기능에 의존하기 시작하면 애그리거트 간 결합도가 높아짐.
  - 결합도가 높아질수록 향후 수정 비용이 증가함.
  - 따라서 다른 애그리거트의 상태를 변경하지 말아야 함.
 
- 부득이하게 두 개 이상의 애그리거트를 수정해야 한다면 직접 다른 애그리거트를 수정하기보다 응용 서비스에서 두 애그리거트를 수정하도록 구현

```kotlin
class ChangeOrderService(  
    private val orderRepository: OrderRepository,  
) {  
	// 두 개 이상의 애그리거트를 변경해야 하면,
	// 응용 서비스에서 각 애그리거트의 상태를 변경한다.
    @Transactional  
    fun changeShippingInfo(  
        id: OrderId,  
        newShippingInfo: ShippingInfo,  
        useNewShippingAddressAsMemberAddress: Boolean,  
    ) {  
        val order: Order = this.orderRepository.findById(id) ?: throw OrderNotFoundException(id)  
        order.shipTo(newShippingInfo)  
        if (useNewShippingAddressAsMemberAddress) {  
            val member: Member = findMember(order.orderer)  
            member.changeAddress(newShippingInfo.address)  
        }  
    }  
}
```

- 도메인 이벤트를 사용하면 한 트랜잭션 내에서 하나의 애그리거트만 수정하면서 다른 애그리거트를 수정하는 코드를 작성할 수 있음
- 단, 다음의 경우에는 한 트랜잭션에서 두 개 이상의 애그리거트를 변경하는 것을 고려
  - 팀 표준
	  - 팀 표준에 따라 사용자 유스케이스와 관련된 응용 서비스의 기능을 한 트랜잭션으로 실행해야 하는 경우가 있다.
  - 기술 제약
	  - 이벤트 방식 도입이 어려운 경우 한 트랜잭션에서 다수의 애그리거트를 수정해서 일관성을 처리해야 한다.
  - UI 구현의 편리
	  - 운영자의 편리함을 추구해야 하는 경우 한 트랜잭션에서 여러 주문 애그리거트의 상태를 변경해야 한다.

## 3.3 리포지터리와 애그리거트

- 애그리거트는 개념상 완전한 한 개의 도메인 모델을 표현. 따라서 객체의 영속성을 처리하는 리포지터리는 애그리거트 단위로 존재.
- 하나의 애그리거트와 관련된 테이블이 세 개라면 애그리거트 루트와 매핑되는 테이블 뿐만 아니라 모든 구성요소에 매핑된 테이블에 데이터를 저장

```kotlin
// 루트 엔티티 Order 뿐만 아니라, OrderLine 이나 다른 구성 요소도 함께 영속화되어야 한다.
this.orderRepository.save(order)
```

- 동일하게 애그리거트를 구하는 리포지터리 메서드는 완전한 애그리거트를 제공해야 함

```kotlin
val order: Order = this.orderRepository.findById(orderId);
// order가 완전한 애그리거트가 아니면
// 기능 실행 도중 npe가 발생할 수 있다.
order.cancel()
```

- 애그리거트를 영속화하는 기술, 저장소가 무엇이든 상태가 변경되면 모든 변경을 원자적으로 저장소에 반영해야 한다. 
  - jpa라면 관계형 모델에 도메인 모델을 맞춰야 할 것이다.
  - rdbms라면 트랜잭션을 이용할 수 있고, 몽고db라면 한개 애그리거트를 하나의 문서에 저장할 수 있을 것이다.
 
## 3.4 id를 이용한 애그리거트 참조

- 하나의 객체가 다른 객체를 참조하는 것처럼, 애그리거트도 다른 애그리거트를 참조
- 애그리거트 관리 주체는 루트이므로 이는 하나의 루트가 다른 루트를 참조한다는 것과 같음

<img width="588" height="375" alt="스크린샷 2025-07-07 오후 8 18 59" src="https://github.com/user-attachments/assets/43583657-d219-43c2-af30-91229f7fbdbb" />

- 필드를 이용해서 다른 애그리거트를 직접 참조하는 것은 개발에게 구현의 편리함을 제공. 하지만 필드를 이용한 애그리거트 참조는 다음 문제를 야기할 수 있음
  - 편한 탐색 오용
  - 성능에 대한 고민
  - 확장 어려움

#### 편한 탐색 오용
- 한 애그리거트 내부에서 다른 애그리거트 객체에 접근할 수 있으면 다른 애그리거트의 상태를 쉽게 변경 가능
```kotlin
class Order(  
    val orderer: Orderer,  
) {  
    fun changeShippingInfo(  
        newShippingInfo: ShippingInfo,  
        useNewShippingAddressAsMemberAddress: Boolean,  
    ) {  
        if (useNewShippingAddressAsMemberAddress) {  
            this.orderer.member.changeAddress(newShippingInfo)  
        }  
    }  
}
```

- 이는 애그리거트 간의 의존 결합도를 높여서 결과적으로 애그리거트의 변경을 어렵게 만듬

#### 성능에 대한 고민
- jpa를 사용한다면 지연 로딩과 즉시 로딩 중에 선택해야 함
- 이는 애그리거트의 어떤 기능을 사용하느냐, 다양한 경우의 수를 고려해 로딩 전략을 결정해야 함

#### 확장 어려움
- 초기에는 단일 서버, 단일 dbms로 서비스스 제공하는 것이 가능
- 사용자가 몰리면서 하위 도메인별로 시스템을 분리하고, 서로 다른 기술을 사용하기 시작. 더 이상 다른 애그리거트 루트를 참조하기 위해 단일 기술을 사용할 수 없어짐


- 이런 문제를 완화하기 위해 사용할 수 있는 것이 id를 이용해 다른 애그리거트를 참조하는 것

<img width="618" height="401" alt="스크린샷 2025-07-07 오후 8 33 35" src="https://github.com/user-attachments/assets/44905d97-3444-40b1-b016-fd2434763c26" />

- id 참조를 사용하면 한 애그리거트에 속한 객체들만 참조로 연결
  - 이는 애그리거트의 경계를 명확히 하고 물리적인 연결을 제거하여 모델의 복잡도를 낮춰줌
  - 애그리거트 간 의존을 제거하므로 응집도를 높여주는 효과도 있음
  
- 더 이상 로딩 전략에 대해서도 고민할 필요도 없음. 참조하는 애그리거트가 필요하면 응용 서비스에서 id를 이용해서 로딩하면 됨.
- 애그리거트에서 다른 애그리거트의 상태 변경과 서로 다른 구현 기술에 대한 고민도 필요 없음

```kotlin
class ChangeOrderService(  
    private val orderRepository: OrderRepository,  
    private val memberRepository: MemberRepository,  
) {  
    @Transactional  
    fun changeShippingInfo(  
        id: OrderId,  
        newShippingInfo: ShippingInfo,  
        useNewShippingAddressAsMemberAddress: Boolean,  
    ) {  
        val order: Order = this.orderRepository.findById(id) ?: throw OrderNotFoundException(id)  
        order.shipTo(newShippingInfo)  
        if (useNewShippingAddressAsMemberAddress) {  
	        // id를 이용해 참조하는 애그리거트를 구한다.
            val member: Member = this.memberRepository.findById(order.orderer.memberId)  
            member.changeAddress(newShippingInfo.address)  
        }  
    }  
}
```

### 3.4.1 id를 이용한 참조와 조회 성능

- 하나의 dbms에 데이터가 있다면 조인을 이용해 한 번에 모든 데이터를 가져올 수 있음에도 매번 다른 애그리거트를 조회하는 쿼리를 실행해야할 수 있음

```kotlin
val member: Member = this.memberRepository.findById(orderId)  
val orders: List<Order> = this.orderRepository.findByOrderer(orderId)  
val dtos: List<OrderView> = orders.map {  
    val productId: ProductId = it.orderLines.get(0).productId  
    val product: Product = this.productRepository.findById(productId)  
    OrderView(it, member, product)  
}
```

- id를 이용한 애그리거트 참조는 지연 로딩처럼 n + 1 문제가 발생
- 이런 문제를 방지하기 위해서는 데이터 조회 전용 쿼리를 사용
- 데이터 조회를 위한 별도 dao를 만들고 dao의 조회 메서드에서 조인을 이용해 한 번의 쿼리로 필요한 데이터를 로딩

```kotlin
@Repository  
class JpaOrderViewDao(  
    private val em: EntityManager,  
): OrderViewDao {  
    fun selectByOrderer(ordererId: String): List<OrderView> {  
        return em.createQuery(  
            """  
            select new org.seong.dddstart.app.OrderView(o, m, p)  
            from Order o  
            join o.orderLines ol, Member m, Product p  
            where o.orderer.memberId.id = :ordererId  
            and index(ol) = 0  
            and ol.productId = p.id  
            order by o.numbe.number desc  
        """.trimIndent(),  
            OrderView::class.java  
        ).setParameter("ordererId", ordererId)  
            .resultList  
    }  
}
```

- 애그리거트마다 서로 다른 저장소를 사용하면 한 번의 쿼리로 관련 애그리거트를 조회할 수 없음
- 이때는 캐시를 적용하거나 조회 전용 저장소를 따로 구성
- 코드가 복잡해지지만 시스템 처리량을 높일 수 있다는 장점이 존재

## 3.5 애그리거트 간 집합 연관

- 애그리거트 간 1:n 관계는 컬렉션을 이용해 표현

```kotlin
class Category(
	private val products: Set<Product>,
) {
}
```

- 그런데 개념적으로 존재하는 1:n 연관을 실제 구현에 반영하는 것이 요구사항 충족과는 상관없을 때가 있음

```kotlin
class Category {
	fun getProducts(page: Int, size: Int) {
		val sortedProducts: List<Product> = sortById(products)
		return sortedProducts.subList((page - 1) * size, page * size)
	}
}
```

- 위의 코드는 category에 속한 모든 prodcut를 조회하게 되면서 성능상 문제가 발생
- 이러한 문제 때문에 개념적인 연관을 실제 구현에 반영하지는 않음

- 카테고리에 속한 상품을 구할 필요가 있다면 상품 입장에서 n:1로 연관 지어 구하면 됨

```kotlin
class Product(
	private val categoryId: CategoryId,
) {
}
```

- 카테고리에 속하는 상품 목록을 제공하는 응용 서비스는 다음과 같이 Product의 categoryId 식별자를 이용해 구할 수 있음

```kotlin
class ProductListService {

	fun getProductOfCategory(categoryId: Long, page: Int, size: Int): Page<Product> {
		...
		val products: List<Product> = productRepository.findByCategoryId(category.id, page, size);
		...
	}
}
```

- m:n 연관은 개념적으로 양쪽 애그리거트에 연관을 만듬
- 1:n 연관과 같이 실제 요구사항을 고려하여 m:n 연관을 구현에 포함시킬지 결정해야 함

- 보통 카테고리에 속한 상품 목록을 띄워줄 때상품이 속한 모든 카테고리를 띄워주지 않음. 이는 상품상세 화면에서 필요.
- 이를 고려하면 카테고리 -> 상품으로의 집합 연관은 필요 없음

```kotlin
class Product(
	private val categoryIds: Set<CategoryId>,
) {
}
```

<img width="503" height="151" alt="스크린샷 2025-07-08 오후 8 03 04" src="https://github.com/user-attachments/assets/b7e0a3e4-3a75-4db9-9448-a6bdfa092957" />

- jpa에서는 다음과 같이 categoryId 컬렉션에 지정한 값이 존재하는지 확인하여 원하는 product 목록을 구할 수 있음
```kotlin
fun findByCategoryId(catId: CategoryId, page: Int, size: Int): List<Product> {
	return em.createQuery(  
    """  
        select p from product p
        where :catId member of p.categoryIds order by p.id.id desc  
    """.trimMargin(),  
    Product::class.java  
	).setParameter("catId", catId)
	.setFirstResult((page - 1) * size)
	.setMaxResult(size)
	.getResultList()
}
```

## 3.6 애그리거트를 팩토리로 사용하기

- 다음의 코드는 상품이 생성 가능한지 판단하는 코드와 생성하는 코드가 분리되어 있음
- 중요한 도메인 로직 처리가 응용 서비스에 노출되어 있음

```kotlin
class RegisterProductService {  
  
    fun registerNewProduct(request: NewProductRequest): ProductId {  
        val store: Store = storeRepository.findById(request.storeId)  
        if (store.isBlocked()) {  
            throw StoreBlockedException()  
        }  
        val id: ProductId = productRepository.nextId()  
        val product: Product = Product(id, store.id, ...)  
        productRepository.save(product)  
        return id  
    }  
}
```

- 이는 논리적으로 하나의 도메인 기능
- 이 기능을 위해 별도의 도메인 서비스, 팩토리 클래스를 만들 수도 있지만 Store 애그리거트에 구현할 수도 있음
```kotlin
class Store {  
    fun createProduct(newProductId: ProductId): Product {  
        if(this.isBlocked()) throw StoreBlockedException()  
        return Product(newProductId, this.id, ...)  
    }  
}
```

- Store#createProduct는 팩토리 역할을 하면서도 중요한 도메인 로직을 구현
- 응용 서비스에서는 해당 팩토리를 이용해 더 이상 Store의 상태를 확인하지 않음.
  - 이제 Product 생성 가능 여부를 확인하는 도메인 로직을 변경해도 Store만 변경하면 되고 응용 서비스는 영향을 받지 않음
  - 도메인의 응집도도 높아짐

- 하나의 애그리거트가 가진 데이터를 이용해 다른 애그리거트를 생성해야 한다면 애그리거트에 팩토리 메서드를 구현하는 것을 고려해볼 것
- 다른 애그리거트를 생성할 때 많은 정보를 알아야 한다면 팩토리에 위임하는 방법도 있음
```kotlin
class Store {
	fun createProduct(newProductId: ProductId, info: ProductInfo): Product {
		if(this.isBlocked()) throw StoreBlockedException()
		return ProductFactory.create(newProductId, this.id, info)
	}
}
```

- 여전히 차단 상태에서 상품은 만들 수 없다는 도메인 로직은 한곳에 계속 위치함
