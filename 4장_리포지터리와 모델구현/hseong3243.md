## 4.1 jpa를 이용한 리포지터리 구현

- 데이터 보관소로 rdbms를 사용할 때, 객체 기반의 도메인 모델 - 관계형 테이터 모델 간의 매핑을 처리하는 기술을 orm 만한 것이 없음

### 4.1.1 모듈 위치

- 리포지터리 인터페이스는 애그리거트와 같이 도메인 영역에 속함
- 이를를 구현한 클래스는 인프라 영역에 속함

<img width="632" height="210" alt="스크린샷 2025-07-08 오후 8 26 45" src="https://github.com/user-attachments/assets/e6e84f3e-fe40-43fc-92ac-b72214633c6b" />


### 4.1.2 리포지터리 기본 기능 구현

- 기본 기능은 다음 두 가지
  - id로 애그리거트 조회하기
  - 애그리거트 저장하기

- 리포지터리 인터페이스는 루트 엔티티를 기준으로 작성
```kotlin
interface OrderRepository {
	fun findByid(number: OrderNumber): Order?
	fun save(order: Order)
}
```

- id 외에 다른 조건으로 조회할 때는 findBy 뒤에 조건 대상이 되는 프로퍼티 이름을 붙임
```kotlin
interface OrderRepository {
	fun findByordererId(ordererId: String, startRow: Int, size: Int): List<Order>
}
```

## 4.2 스프링 데이터 jpa를 이용한 리포지터리 구현

- 스프링 데이터 jpa는 규칙에 따라 작성한 인터페이스를 찾아 그를 구현한 스프링 빈 객체를 자동으로 등록
  - org.springframework.data.repository.Repository<T, ID> 인터페이스 상속한 객체
 
## 4.3 매핑 구현
### 4.3.1 엔티티와 밸류 기본 매핑 구현

- 애그리거트와 jpa 매핑을 위한 기본 규칙은 다음과 같음
  - 애그리거트 루트는 엔티티이므로 @Entity 로 매핑 설정

- 한 테이블에 엔티티와 밸류 데이터가 같이 있다면
  - 밸류는 @Embeddable 로 매핑 설정
  - 밸류 타입 프로퍼티는 @Embedded 로 매핑 설정
  
<img width="624" height="338" alt="스크린샷 2025-07-12 오전 10 27 01" src="https://github.com/user-attachments/assets/e79f4040-c1fc-4d3b-9bce-41fee80a98ea" />

```kotlin
@Entity  
@Table(name = "orders")  
class Order(  
    orderer: Orderer,  
) {  
    @Id  
    var id: UUID = UUID.randomUUID()  
        private set  
  
    var orderer: Orderer = orderer  
        private set  
}

@Embeddable  
data class Orderer(  
    @Embedded  
    @AttributeOverrides(  
        value = [  
            AttributeOverride(name = "id", column = Column(name = "orderer_id"))  
        ]  
    )  
    val memberId: MemberId,  
  
    @Column(name = "orderer_name")  
    val name: String,  
)

@Embeddable  
data class MemberId(  
    @Column(name = "member_id")  
    val id: String,  
)
```

### 4.3.2 기본 생성자

- 엔티티와 밸류의 생성자는 객체 생성시 필요한 것을 전달 받음
- 밸류가 불변이라면 기본 생성자는 필요하지 않으나, jpa는 @Entity와 @Embeddable로 클래스를 매핑하려면 기본 생성자를 제공해야 함
- 코틀린을 사용한다면 다음의 플러그인을 통해 기본 생성자를 자동으로 추가할 수 있음
```groovy
plugins {
	id "org.jetbrains.kotlin.plugin.jpa" version "2.2.0"
}
```

- 자바를 사용한다면 다른 코드에서 생성하지 못하도록 기본 생성자에 protected를 선언해야 함


### 4.3.3 필드 접근 방식 사용

- 엔티티에 프로퍼티를 위한 공개 get/set 메서드를 추가하면 도메인의 의도가 사라지고 캡슐화가 깨질 수 있음
  - 객체로서 제 역할을 다하게 하려면 setter 대신 의도가 잘 드러나는 기능을 제공해야 함
  - 제공할 기능 중심으로 엔티티를 구현하게끔 유도하려면 jpa 매핑 처리 방식을 필드 방식으로 선택하여 불필요한 get/set 구현을 하지 말아야 함
 
- 하이버네이트에 의해 암시적으로 동작할 수 있지만 `@Access` 어노테이션으로 명시적으로 제어하거나, `@Id` 어노테이션이 매핑된 위치에 따라 결정하도록 만들 수 있음
```kotlin
@Entity  
@Table(name = "orders")  
@Access(AccessType.FIELD)  
class Order(  
    orderer: Orderer,  
) {  
    @Id  // 또는 @field:Id
    var id: UUID = UUID.randomUUID()  
        private set  
  
    var orderer: Orderer = orderer  
        private set  
}
```

### 4.3.4 AttributeConverter를 이용한 밸류 매핑 처리

- 밸류 타입의 프로퍼티를 한 개 컬럼에 매핑해야 할 때도 있음
<img width="545" height="142" alt="스크린샷 2025-07-12 오전 11 30 13" src="https://github.com/user-attachments/assets/b594d1e1-48eb-450b-b120-7483a5ebb76a" />

- AttributeConverter 는 밸류 타입과 컬럼 데이터 간의 변환을 처리하기 위한 기능을 정의

```java
public interface AttributeConverter<X,Y> {  
    public Y convertToDatabaseColumn (X attribute);  
    public X convertToEntityAttribute (Y dbData);  
}
```
```kotlin
@Converter(autoApply = true)  
class MoneyConverter: AttributeConverter<Money, Int> {  
  
    override fun convertToDatabaseColumn(money: Money?): Int? {  
        return money?.let { money.value }  
    }  
  
    override fun convertToEntityAttribute(value: Int?): Money? {  
        return value?.let { Money(it) }  
    }  
}
```

- AttributeConverter를 구현한 클래스는 @Converter 애노테이션을 사용. autoApply=true로 설정한 경우 모델에 출현하는 모든 Money 타입의 프로퍼티에 대해 컨버터를 자동으로 적용
- false 라면 컨버터를 직접 지정

```kotlin
class Order {
	@Column(name = "total_amounts")
	@Convert(converter = MoneyConverter::class.java)
	var total_amounts: Money
}
```

### 4.3.5 밸류 컬렉션: 별도 테이블 매핑

- Order 엔티티는 한 개 이상의 OrderLine을 가질 수 있음
<img width="568" height="337" alt="스크린샷 2025-07-12 오후 2 00 02" src="https://github.com/user-attachments/assets/c960ea53-2a0a-45c9-a296-a8c38be3ed3e" />

- List 타입의 컬렉션은 인덱스 값이 필요하므로 order_line 테이블에는 인덱스 값을 저장하기 위한 컬럼(line_idx)도 존재
- 밸류 컬렉션을 별도 테이블로 매핑할 때는 @ElementCollection과 @CollectionTable을 함께 사용
```kotlin
@Entity  
@Table(name = "orders")  
@Access(AccessType.FIELD)  
class Order(  
    orderLines: List<OrderLine>,  
) {  
    @EmbeddedId  
	val number: OrderNumber,
  
    @ElementCollection(fetch = FetchType.EAGER)  
    @CollectionTable(  
        name = "order_line",  
        joinColumns = [JoinColumn(name = "order_number")]  
    )  
    @OrderColumn(name = "line_idx")  
    var orderLines: List<OrderLine> = orderLines  
        private set  
}

@Embeddable  
data class OrderLine(  
    @Column(name = "price")  
    val pricer: Money,  
    @Column(name = "quantity")  
    val quantity: Int,  
    @Column(name = "amounts")  
    val amounts: Money,  
)
```

- jpa는 @OrderColumn 애노테이션을 이용해서 지정한 컬럼에 리스트의 인덱스 값을 저장
- @CollectionTable은 밸류를 저장한 테이블을 지정. joinColumns 속성은 외부키로 사용할 컬럼을 지정. 두 개 이상인 경우 @JoinColumn 배열을 이용해서 외부키 목록을 지정

### 4.3.6 밸류 컬렉션:한 개 컬럼 매핑

- 밸류 컬렉션을 별도 테이블이 아니라 한 개 컬럼에 저장해야 할 때가 있음
- AttributeConverter를 사용하면 밸류 컬렉션을 한 개 컬럼에 쉽게 매핑 가능
```kotlin
data class EmailSet(  
    val emails: Set<Email>,  
)  
  
data class Email(  
    val value: String,  
)

class EmailSetConverter : AttributeConverter<EmailSet, String> {  
    override fun convertToDatabaseColumn(emailSet: EmailSet?): String? {  
        return (emailSet ?: return null)  
            .emails  
            .joinToString(",") { it.value }  
    }  
  
    override fun convertToEntityAttribute(dbData: String?): EmailSet? {  
        return (dbData ?: return null)  
            .split(",")  
            .map { Email(it) }  
            .toSet()  
            .let { EmailSet(it) }  
    }  
}
```

### 4.3.7 밸류를 이용한 id 매핑

- 식별자의 의미를 부각시키기 위해 식별자 자체를 밸류 타입으로 만들 수도 있음
- 밸류 타입을 식별자로 매핑하면 @Id 대신 @EmbeddedId 애노테이션을 사용
```kotlin
@Entity  
@Table(name = "orders")  
class Order(  
) {  
    @EmbeddedId  
    var number: OrderNumber = OrderNumber(UUID.randomUUID().toString())  
        private set  
}

@Embeddable  
data class OrderNumber(  
    @Column(name = "order_number")  
    val number: String,  
): Serializable
```

- jpa에서 식별자 타입은 Serializable 타입이어야 함. 따라서 밸류 타입은 반드시 Serializable 인터페이스를 구현
- jpa는 엔티티 비교를 위해 equals(), hashcode()를 사용하므로 밸류 타입은 이를 적절하게 구현해야 함

### 4.3.8 별도 테이블에 저장하는 밸류 매핑

- 애그리거트에서 루트 엔티티를 뺀 나머지는 대부분 밸류
  - 루트 이외에 엔티티가 있다면 진짜 엔티티인지 의심이 필요
  - 별도 테이블에 저장한다고 엔티티가 되지는 않음
  - 엔티티가 확실하다면 다른 애그리거트는 아닌지 확인. 자신만의 라이프사이클을 갖는다면 구분되는 애그리거트일 가능성이 높음
  - Product와 Review는 함께 보여주겠지만 생성 시점도 다르고, 함께 변경되지도 않음. 변경 주체도 다름. 때문에 둘은 서로 다른 애그리거트에 속함

- 밸류를 구분하는 방법은 고유 식별자를 갖는지 여부
  - 그러나 테이블의 식별자는 애그리거트 구성요소의 식별자와 동일한 것으로 생각해서는 안됨
<img width="536" height="279" alt="스크린샷 2025-07-15 오후 9 16 21" src="https://github.com/user-attachments/assets/201e7bf8-912c-4e6f-8b47-39129a304927" />

- Article_Content는 Article의 내용을 담고 있는 밸류
- 테이블의 id는 Article 테이블과 연결하기 위함이지 식별자가 필요하기 때문이 아님. 따라서 올바른 매핑은 다음과 같음
<img width="438" height="149" alt="스크린샷 2025-07-15 오후 9 17 40" src="https://github.com/user-attachments/assets/eab0b6f6-1a01-4909-bba3-8e926ac22392" />

- jpa로 이를 구현하는 과정에서 `@SecondaryTable`, `@AttributeOverride`와 같은 애노테이션이 필요
- 테이블 조인 과정에서 즉시 로딩으로 인한 불필요한 성능 저하가 발생할 수도 있음. 대신 조회 전용 기능을 구현하는 방법을 사용한다면 해결 가능

### 4.3.9 밸류 컬렉션을 @Entity로 매핑하기
- 개념적으로 밸류이나 기술적 한계, 팀 표준때문에 `@Entity`를 사용해야 할 수도 있음
<img width="538" height="257" alt="스크린샷 2025-07-15 오후 9 23 03" src="https://github.com/user-attachments/assets/4c2eece9-63a0-4f16-868e-ceab58ad3203" />

- `@Embeddable`은 상속 구조를 지원하지 않기 때문에 `@Entity`를 사용해야하는 상황
- 이때는 `@Inheritance`, `@DiscriminatorColumn` 애노테이션을 이용해 해결 가능
<img width="587" height="210" alt="스크린샷 2025-07-15 오후 9 23 47" src="https://github.com/user-attachments/assets/0ed6ddb3-1965-477d-98f4-d1ec05f83a8c" />

- 하이버네이트는 `@Entity` 컬렉션에 대한 `clear()`를 호출하면 컬렉션을 가져오기 위한 한 번의 쿼리와 네번의 delete 쿼리를 실행
- `@Embeddable`의 경우 `clear()` 호출 시 객체를 로딩하지 않고 한 번의 delete 쿼리를 실행. 이 부분을 잘 고려하여 결정해야 함

### 4.3.10 id 참조와 조인 테이블을 이용한 단방향 m-n 매핑

- 요구사항을 구현하는 데 집합 연관을 사용하는 것이 유리하다면 id 참조를 이용한 단방향 집합 연관을 적용해 볼 수 있음
<img width="697" height="308" alt="스크린샷 2025-07-15 오후 9 34 33" src="https://github.com/user-attachments/assets/568d8fc5-1e66-4ef5-8273-6b012cc6a2fe" />

### 4.4 애그리거트 로딩 전략

- 항상 염두에 둬야 하는 점은 애그리거트 루트 로딩시 루트에 속한 모든 객체가 완전한 상태여야 한다는 것
- jpa는 즉시 로딩을 통해 연관된 컬렉션, `@Entity`를 db에서 함께 읽어올 수 있음
  - 그러나 컬렉션 매핑이 두개이상일 때, 카테시안 곱 문제가 발생
```sql
select 
	...
from
	product p
	left join image i on p.product_id = i.product_id
	left join product_option po on p.product_id = po.product_id
where p.product_id = ?
```

- image가 2개, product_option이 2개이면 4개의 행을 구해옴
  - product는 4번 중복, image, product_option은 각각 2번 중복

- 애그리거트는 개념적으로 하나여야 함
- 하지만 루트 엔티티 로딩 시점에 애그리거트에 속한 모든 객체를 로딩해야하는 것은 아님
  - 애그리거트가 완전해야 하는 이유는 두가지로 생각해볼 수 있음
    - 상태 변경 기능 실행 시 애그리거트 상태가 완전해야 함
    - 표현 영역에서 애그리거트 상태 정보를 보여줄 때 필요

- 애그리거트의 완전한 로딩은 첫 번째 문제와 더 연관이 있음
  - 두 번째의 경우 조회 전용 기능과 모델을 만드는 편이 유리함
- 하지만 jpa는 동일한 트랜잭션 안에서 지연 로딩을 허용하기 때문에 실제 상태를 변경하는 시점에 필요한 구성요소만 로딩해도 문제되지 않음

<img width="667" height="590" alt="스크린샷 2025-07-15 오후 9 42 01" src="https://github.com/user-attachments/assets/cf542c95-c526-4edb-a34d-6d19af789589" />

- 일반적인 애플리케이션은 조회 기능 실행이 더 빈번
- 상태 변경을 위한 추가 쿼리로 인한 실행 속도 저하는 보통 문제가 되지 않음
- 무조건 즉시 로딩보다는 애그리거트에 맞게 즉시 로딩과 지연 로딩을 선택해야 함

## 4.5 애그리거트의 영속성 전파
- 애그리거트가 완전해야한다는 의미는 조회뿐만 아니라 저장, 삭제할 때도 하나로 처리해야 함을 뜻함
- `@Embeddable`은 함께 저장되고 삭제된다. 하지만 `@Entity`는 cascade 속성을 적절하게 사용해 함께 처리되도록 만들어야 함

## 4.6 식별자 생성 기능
- 식별자는 세 가지 방식 중 하나로 생성해야 함
  - 사용자가 직접 생성
  - 도메인 로직으로 생성
  - db를 이용한 일련번호 사용

- 식별자 생성 규칙이 있다면 별도 서비스로 식별자 생성 기능을 분리해야 함
- 이는 도메인 규칙이므로 도메인 영역에 생성 기능을 위치시켜야 함
```kotlin
class ProductIdService {
	fun nextId(): ProductId { 
		...
	}
}
```

- 응용 서비스는 도메인 서비스를 이용해 식별자를 구하고 엔티티를 생성
```kotlin
class CreateProductService {
	@Transactional
	fun createProduct(command: ProductCreationCommand): ProductId {
		val id = this.productIdService.nextId()
		val product = Product(id, command.detail, command.price, ...)
		this.productRepository.save(product)
		return id
	}
}
```

- 특정 값의 조합으로 식별자가 생성되는 것 역시 규칙이므로 도메인 서비스를 이용해서 식별자를 생성할 수 있음
```kotlin
class OrderIdService {
	fun createId(userId: UserId): OrderId {
		userId ?: throw IllegalArgumentException("invalid userId: $userId")
		return new OrderId("${userId.toString()}-${timestamp()}")
	}
	fun timestamp(): String {
		return Long.toString(System.currentTimeMillis())
	}
}
```

- Repository도 식별자 생성 규칙을 구현하기에 적합
- 메서드를 추가하고 구현 클래스에서 알맞게 구현
```kotlin
interface ProductRepository {
	fun nextId(): ProductId
}
```

- db 자동 증가 컬럼을 이용하면 `@GenerateValue`를 이용
- 다만 insert 후에야 식별자가 생성되므로 도메인 객체 생성시점에는 식별자를 구할 수 없음
- 이외에 jpa 기능들 역시 insert 후에야 알 수 있다는 점은 동일

## 4.7 도메인 구현과 DIP
- 지금까지 나온 예제는 DIP 원칙을 어기고 있음
- jpa는 구현 기술에 속하므로 도메인 모델, 리포지터리 인터페이스 둘 다 마찬가지
- 도메인을 순수하게 유지하기 위해서는 jpa에 의존적인 클래스를 인프라 영역에 위치시켜야 함
<img width="564" height="259" alt="스크린샷 2025-07-15 오후 9 55 31" src="https://github.com/user-attachments/assets/9e8e2cfb-d2f6-4298-aa10-9ac35b682edc" />


- 하지만 리포지터리나 도메인 모델 구현 기술은 거의 바뀌지 않음. 이렇게 변경 없는 상황에서도 순수하게만 만들 필요는 없음
- 애그리거트, 리포지터리 등 도메인 모델을 구현할 때는 적절히 타협할 수도
- jpa 전용 애노테이션을 사용하더라도 도메인 모델 단위 테스트는 문제 없음. repository 역시 마찬가지
- 직접 대역 객체를 구현한다면 구현해야할 메서드가 좀 있긴 하지만 테스트하는데 큰 어려움은 없음
