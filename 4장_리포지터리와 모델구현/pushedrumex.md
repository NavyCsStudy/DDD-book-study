# 리포지터리와 모델 구현

## 4.1 JPA를 이용한 리포지터리 구현
- 리포지터리 인터페이스는 도메인 영역에 속하고 리포지터리를 구현한 클래스는 인프라에 속함
- 리포지터리의 기본 기능
  - ID로 애그리거트 조회
  - 애그리거트 저장
- 인터페이스는 애그릐거트 루트를 기준으로 작성
```java
public interface OrderRepository {
    Order findById(OrderNo no);
    void save(Order order);
}
```
- 조회 메서드명 규칙 : findBy + 프로퍼티이름
- 스프링 데이터 JPA 는 인터페이스만 정의하면 구현 객체는 알아서 생성해 줌
- JPA는 트랜잭션 내에서 변경한 데이터를 자동으로 DB에 반영
```java
public class ChangeOrderService {
    @Transactional
    public void changeShippingInfo(OrderNo no, Shippinginfo newShippinglnfo) {
        Optional<Order> orderOpt = orderRepository.findByld(no);
        Order order = orderOpt.orElseThrow(() -> new OrderNotFoundException());
        order.changeShippinglnfo(newShippinglnfo);
    }
}
```
- ID 외의 다른 조건으로 조회할 경우 Criteria나 JPQL을 사용

## 4.2 스프링 데이터 JPA를 이용한 리포지터리 구현
- 스프링 데이터 JPA는 리포지터리 인터페이스를 정의하면 구현 객체를 알아서 만들어 스프링 빈으로 등록
- 리포지터리 인터페이스 규칙
  - org.springframework.data.repository.RepositoryCL ID> 인터페이스 상속
  - 丁는 엔티티 타입을 지정하고 ID는 식별자 타입을 지정
```java
public interface OrderRepository extends Repository<Order, OrderNo> {
    Optional<Order> findByld(OrderNo id);
    void save (Order order);
}
```
- 메서드 작성 규칙
```java
Order save(Order entity);
void save(Order entity);

Order findById(OrderNo id);
Optional<Order> findById(OrderNo id);

List<Order> findByOrderer(Orderer orderer);

void delete(Order order);
void deleteById(OrderNo id);
```

## 4.3 매핑 구현
### 엔티티, 밸류 기본 매핑 구현
- 애그리거트 류트는 엔테테이므로 @Entity로 매핑 설정
```java
@Entity
@Table(name = "purchase_order")
public class Order {
```
- 밸류는 @Embeddable로 매핑 설정
```java
@Embeddable
public class Orderer {
```
- 밸류 타입 프로퍼티는 @Embedded로 매핑 설정
- @AttributeOverrides 애노테이션을 이용하여 프로퍼티와 매핑할 컬럼 이름 변경 가능
```java
@Embeddable
public class Orderer {
    // Memberld에 정의된 칼럼 이름을 변경하기 위해
    // @AttributeOverride 애너테이션 사용
    @Embedded
    @Attrib니teOverrides(
        @AttributeOverride(name = "id", column = @Column(name = "orderer_id"))
    )
    private Memberld memberld;
```
### 기본 생성자
- JPA에서는 데이터를 읽어와 매핑된 객체를 생성할 때 기본 생성자를 이용하기 때문에 기본 생성자가 필수임
- 다른 코드에서 기본 생성자를 사용하지 못하도록 protected로 선언하는 것을 추천
### 필드 접근 방식 사용
- 메서드 방식으로 매핑을 처리할 수 있음
- @Access 애노테이션을 통해 접근 방식 지정 가능
```java
@Entity
@Access(AccessType.PROPERTY)
public class Order {
    @Column(name = "state")
    @Enumerated(EnumType.STRING)
    public OrderState getState() {
        return state;
    }

    public void setState(OrderState state) {
        this.state = state;
    }
}
```
- 메서드 방식을 사용할 경우 도메인 의도가 사라지고 데이터 기반으로 엔티티를 구현할 가능성이 높아짐
- set 메서드 대신 의도가 드러나는 이름을 사용하는 것을 추천 ex) cancel()
### AttributeConverter
- 두 개 이상의 프로퍼티를 가진 밸류 타입을 한 개의 컬럼에 매핑할 수 있는 방법
- AttributeConverter는 밸류 타입과 칼럼 데이터 간의 변환을 처리하기 위한 기능을 제공
```java
public interface AttributeConverter<X, Y> {
    public Y convertToDatabaseColumn(X attribute);
    public X convertToEntityAttribute(Y dbData);
}
```
- X : 밸류 타입 / Y : DB 타입
- 사용 예시 : Money
```java
@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, Integeo {
@Override
public Integer convertToDatabaseColumn(Money money) {
    return money == null ? null : money.getValue();
}
@Override
public Money convertToEntityAttribute(Integer value){
        return value == null ? null : new Money(value);
    }
}
```
```java
@Entity
@Table(name = "purchase_order")
public class Order {
    @Colmn(name = "total_amount")
    private Money totalAmounts; // MoneyConverter를 적용해서 값 변환
}
```
### 밸류 컬렉션 : 별도 테이블 매핑
- 밸류 컬렉션을 별도 테이블로 매필할 경우 @ElementCollection과 @CollectionTable을 함
  께 사용
```java
@Entity
@Table(name = "purchase_order")
public class Order {
    @EmbeddedId
    private OrderNo number;
    
    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "order_line",
        joinColumns = @JoinColumn(name = "order_number"))
    @OrderColumn(name = "lineidx")
    private List<DrderLine> orderLines;
}
```
- @CollectionTable은 밸류를 저장할 테이블을 지정
  - name 속성은 테이블 이름을 지정하고 joinColumns 속성은 외부키로 사용할 칼럼을 지정
- JPA는 @OrderC이니mn 애너테이션을 이용해서 지정한 칼럼에 리스트의 인덱스 값을 저장

### 밸류 컬렉션 : 한 개 칼럼 매핑
- 밸류 컬렉션을 한 개 컬럼에 저장할 경우
- AttributeConverter를 사용하여 밸류 컬렉션을 한 개 컬럼에 매핑 가능
```java
public class EmailSet {
    private Set<Email> emails = new HashSetoO;

    public EmailSet(Set<Email> emails) {
        this.emails.addAU(emails);
    }

    public Set<Email> getEmails() {
        return Collections.unmodifiableSet(emails);
    }
}
```
```java
public class EmailSetConverter implements Attrib니teConverter<EmailSet, String> {
    @Override
    public String convertToDatabaseColumn(EmailSet attrib니te) {
        if (attribute == nuU) return null;
        return attribute.getEmails().stream()
            .map(email -> email.getAddressO)
            .collect (Collectors. joining ("/'))；
    }
    @Override
    public EmailSet convertToEntityAttribute(String dbData) {
    if (dbData == nuU) return null;
    String[] emails = dbData.split("/')；
    Set<Email> emailSet = Arrays .stream(email.s)
    .map(value -> new Email(value))
    .collect(toSet()));
    return new EmailSet(emailSet);
  }
```
```java
@Column(name = "emails")
@Convert(converter = EmailSetConverter.class)
private EmailSet emailSet;
```
### 밸류를 이용한 ID 매핑
```java
@Entity
@Table(name = "purchase_order")
public class Order {
    @EmbeddedId
    private OrderNo number;
}

@Embeddable
public class OrderNo implements Serializable {
    @Column(name ="order_number")
    private String number;
}
```
- JPA에서 식별자 타입은 Serializable 타입이어야 하므로 식별자로 사용할 밸류 타입은 Serializable 인터페이스를 상속
### 별도 테이블에 저장하는 밸류 매핑
- @SecondaryTable 를 사용하여 밸류를 매핑한 테이블을 지정 할 수 있음
  - name : 밸류를 지정할 테이블
  - pkJoinColumns : 밸류 테이블에서 엔티티 테이블로 조인할 때 사용할 컬럼
```java
@Entity
@Table(name = "article")
@SecondaryTable(
    name = "article_content",
    pkJoinColumns = @PrimaryKeyJoinColumn(name = "id")
)
public class Article {}
```
### 밸류 컬렉션을 @Entity로 매핑
- JPA 에서는 상속 구조를 갖는 밸류 타입을 사용하려면 @Entity를 이용하여 상속 매핑으로 처리해야함
- Inheritance 애너테이션 적용
  - strategy 값으로 SINGLE_TABLE 사용
- @DiscriminatorColumn 애너테이션을 이용하여 타입 구분용으로 사용할 칼럼 지정
```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "image_type")
@Table(name = "image")
public abstract class Image {
}
```
```java
@Entity
@DiscriminatorValue("II")
public class Internallmage extends Image {
}
@Entity
@DiscriminatorValue("EI")
public class Externalimage extends Image {
}
```
### ID 참조와 조인테이블을 이용한 단방향 M-N 매핑
- Product에서 Category로의 단방향 M-N 연관을 ID 참조 방식으로 구현
- 집합의 값에 밸류 대신 연관을 맺는 식별자
````java
@Entity
@Table(name = "product")
public class Product {
    @EmbeddedId
    private Productld id;
    
    @ElementCollection
    @CollectionTable(name = "product_category",
        joinColumns = @JoinColumn(name = "product_id"))
    private Set<CategoryId> categorylds;
}
````
## 4.4 애그리거트 로딩 전략
- 애그리거트 루트를 로딩하면 루트에 속한 모든 객체가 완전한 상태여야함
  - 첫 번째 이유: 상태를 변경하는 기능을 실행할 때 애그리거트 상태가 완전해야 하기 때문
  - 두 번째 이유: 표현 영역에서 애그리거트의 상태 정보를 보여줄 때 필요하기 때문
- 조회 방식을 즉시 로딩으로 설정하면 연관된 구성 요소를 DB에서 함께 읽어옴
- 하지만 즉시 로딩일 경우 조회되는 데이터 개수가 많아질 수 잇으므로 성능을 검토해야함
- JPA는 트랜잭션 번위 내에서 지연 로딩을 허용하기 때문에 실제로 상태를 변경하는 시점에 필요한 구성요소만 로딩해도 문제가 되지 않음

## 4.5 애그리거트의 영속성 전파
- @Embeddable 매핑 타입은 함께 저장되고 삭제되므로 cascade 속성을 추가로 설정하지 않아도 됨
- 애그리거트에 속한 ©Entity 타입에 대한 매핑은 cascade 속성을 사용해서 저장과 식제 시에 함께 처리되도록 설정

## 4.6 식별자 생성 기능
- 사용자가 직접 생성
- 도메인 로직으로 생성
- DB 일련번호

## 4.7 도메인 구현과 DIP
- 지금까지 구현한 리포지터리는 DIP 원칙을 어기고 있음. JPA 에 특화된 구현 기술들을 사용하고 있음
- DIP를 적용하는 주된 이유는 저수준 구현이 변경되더라도 고수준이 영향을 받지 않도록 하기위함
- 하지만 리포지터리와 도메인 모델의 구현 기술은 거의 바뀌지 않으므로 JPA에 의존하는 것은 개발 편의성과 실용성을 위한 합리적인 선택임