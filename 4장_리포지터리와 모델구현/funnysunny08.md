## 4.1 JPA를 이용한 리포지터리 구현

- 데이터 보관소로 RDBMS를 사용할 때, 객체 기반의 도메인 모델과 관계형 데이터 모델 간의 매핑을 처리하는 기술로 ORM 만한 것이 없다.
- 자바의 ORM 표준인 JPA를 이용해서 리포지터리와 애그리거트를 구현하는 방법에 대해 살펴본다.

### 4.1.1 모듈 위치

<img width="751" height="261" alt="Image" src="https://github.com/user-attachments/assets/67e92f50-fa31-4f92-9228-075e4a8c9e46" />

- 리포지터리 인터페이스 → 도메인 영역
- 리포지터리 구현체 → 인프라스트럭처 영역

### 4.1.2 리포지터리 기본 기능 구현

- 리포지터리가 제공하는 기본 기능
    - ID로 애그리거트 조회
    - 애그리거트 저장
- 인터페이스는 애그리거트 루트를 기준으로 작성한다.
    - Ex) Order, OrderLine, Orderer, ShippingInfo → 루트 엔티티인 Order를 기준으로 리포지터리 인터페이스 작성
- JPA를 사용하면 트랜잭션 범위에서 변경한 데이터는 자동으로 DB에 반영한다.
- ID 외에 다른 조건으로 애그리거트를 조회할 때는 JPA의 Criteria나 JPQL을 사용할 수 있다.

## 4.2 스프링 데이터 JPA를 이용한 리포지터리 구현

- 스프링과 JPA를 함께 적용할 때는 스프링 데이터 JPA를 사용한다.
- 스프링 데이터 JPA는 지정한 규칙에 맞게 리포지터리 인터페이스를 정의하면 리포지터리를 구현한 객체를 알아서 만들어 스프링 빈으로 등록해준다.
    - org.springframework.data.repository.Repository<T, ID> 인터페이스 상속
    - T는 엔티티 타입을 지정하고 ID는 식별자 타입을 지정

## 4.3 매핑 구현

### 4.3.1 엔티티와 밸류 기본 매핑 구현

- 기본 규칙
    - 애그리거트 루트는 엔티티이므로 @Entity로 매핑 설정한다.
    - 한 테이블에 엔티티와 밸류 데이터가 같이 있다면
        - 밸류는 @Embeddable로 매핑 설정한다.
        - 밸류 타입 프로퍼티는 @Embedded로 매핑 설정한다.

<img width="764" height="423" alt="Image" src="https://github.com/user-attachments/assets/3682866d-8053-4f7b-8e7d-ffbd3c220f07" />

루트 엔티티는 JPA의 @Entity로 매핑한다.

```java
@Entity
@Table(name = "purchase_order")
public class Order {
   ...
   @Embedded
   private Orderer orderer;
   
   @Embedded
   private ShippingInfo shippingInfo;
   ...
}
```

Orderer는 Order에 속하는 밸류이므로 @Embeddable로 매핑한다.

```java
@Embeddable
public class Orderer {
    
    // Member에 정의된 칼럼 이름을 변경하게 위해 @AttrbuteOverride 사용
    @Embedded
    @AttributeOverrides(
            @AttributeOverride(name = "id", column = @Column(name = "orderer_id"))
    )
    private MemberId memberId; 

    @Column(name = "orderer_name")
    private String name;
}

// Member 애그리거트를 ID로 참조한다.
@Embeddable
public class MemberId implements Serializable {
    @Column(name = "member_id")
    private String id;
}    
```

### 4.3.2 기본 생성자

- JPA에서 @Entity와 @Embeddable로 클래스를 매핑하려면 기본 생성자를 제공해야 한다
    - DB에서 데이터를 읽어와 매핑된 객체를 생성할 때 기본 생성자를 사용해서 객체를 생성하기 때문
- 이런 기술적인 제약으로 Receiver와 같은 불변 타입은 기본 생성자가 필요 없음에도 불구하고 위와 같이 기본 생성자를 추가해야 한다

```java
@Embeddable
public class Receiver {
		@Column(name = "receiver_name")
		private String name;
		
		...
		
		protected Receiver() {} // 다른 코드에서 기본 생성자를 호출하지 못하도록 protected로 선언
}		
```

### 4.3.3 필드 접근 방식 사용

- JPA는 필드와 메서드의 두 가지 방식으로 매핑을 처리할 수 있다.
- 메서드 방식을 사용하려면 프로퍼티를 위한 get/set 메서드를 구현해야 한다.
    - 엔티티에 프로퍼티를 위한 공개 get/set 메서드를 추가하면 도메인의 의도가 사라지고 객체가 아닌 데이터를 기반으로 엔티티를 구현할 가능성이 높아진다.
    - 특히 set 메서드는 내부 데이터를 외부에서 변경할 수 있는 수단이 되기 때문에 캡슐화를 깨는 원인이 될 수 있다.
    - set 메서드 대신 의미를 잘 드러내는 메서드를 제공하자 → cancel(), changeShippingInfo()
- 필드 방식을 선택해 불필요한 get/set 메서드를 구현하지 않는 것도 방법
    
    ```java
    @Entity
    @Access(AccessType.FIELD)
    public class Order {
    		
    		@Embedded
    		private OrderNo number;
    		
    		@Column(name = "state)
    		@Enumerated(EnumType.STRING)
    		private OrderState state;
    		
    		... // cancel(), changeShippingInfo() 등 도메인 기능 구현
    		... // 필요한 get 메서드 제공
    }		
    ```
    

### 4.3.4 AttributeConverter를 이용한 밸류 매핑 처리

<img width="860" height="280" alt="Image" src="https://github.com/user-attachments/assets/e67ee836-b990-4c45-bd75-dfe97e3b64a6" />

- 두 개 이상의 프로퍼티를 가진 밸류 타입을 한 개 컬럼에 매핑하려면 @Embeddable 애너테이션으로는 처리할 수가 없다.
    - 이때 AttributeConverter를 사용할 수 있다.
    - AttributeConverter는 밸류 타입과 컬럼 데이터 간의 변환을 처리하기 위한 기능을 정의하고 있다.

```java
@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Money, Integer> {
// AttributeConverter<밸류 타입, DB 타입>

    @Override
    public Integer convertToDatabaseColumn(Money money) {
        return money == null ? null : money.getValue();
    }

    @Override
    public Money convertToEntityAttribute(Integer value) {
        return value == null ? null : new Money(value);
    }
}    
```

- autoApply 속성을 true로 지정하면 모델에 출현하는 모든 Money 타입의 프로퍼티에 대해 MoneyConverter를 자동으로 적용한다.
- autoApply 속성을 false로 지정하면 프로퍼티 값을 변환할 때 사용할 컨버터를 직접 지정해야 한다.
    
    ```java
    public class Order {
    
        @Convert(converter = MoneyConverter.class)
        @Column(name = "total_amounts")
        private Money totalAmounts;
    }    
    ```
    

### 4.3.5 밸류 컬렉션: 별도 테이블 매핑

<img width="830" height="558" alt="Image" src="https://github.com/user-attachments/assets/ece8ab9d-d1a6-482f-aba7-bb2d05b764ef" />

- 밸류 컬렉션을 별도 테이블로 매핑할 때는 @ElementCollection과 @CollectionTable을 함께 사용한다.
    - @CollectionTable은 밸류를 저장한 테이블을 지정한다.
    - name 속성은 테이블 이름을 지정하고 joinColumns 속성은 외래키로 사용할 컬럼을 지정한다.

```java
@Entity
@Table(name = "purchase_order")
@Access(AccessType.FIELD)
public class Order {
    @EmbeddedId
    private OrderNo number;

    ...

    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(name = "order_line", joinColumns = @JoinColumn(name = "order_number"))
    @OrderColumn(name = "line_idx")
    private List<OrderLine> orderLines;
    ...
}    
```

### 4.3.6 밸류 컬렉션: 한 개 칼럼 매핑

- 밸류 컬렉션을 별도 테이블이 아닌 한 개 컬럼에 저장해야 하는 경우
    - Ex) 이메일 주소 목록을 콤마로 구분하여 하나의 문자열로 저장 → “a@test.com, b@test.com”
- AttributeConverter를 사용하여 해결

```java
public class EmailSet {
    private Set<Email> emails = new HashSet<>();

    public EmailSet(Set<Email> emails) {
        this.emails.addAll(emails);
    }

    public Set<Email> getEmails() {
        return Collections.unmodifiableSet(emails);
    }

}

public class EmailSetConverter implements AttributeConverter<EmailSet, String> {
    @Override
    public String convertToDatabaseColumn(EmailSet attribute) {
        if (attribute == null) return null;
        return attribute.getEmails().stream()
                .map(email -> email.getAddress())
                .collect(Collectors.joining(","));
    }

    @Override
    public EmailSet convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        String[] emails = dbData.split(",");
        Set<Email> emailSet = Arrays.stream(emails)
                .map(value -> new Email(value))
                .collect(toSet());
        return new EmailSet(emailSet);
    }
}

@Column(name = "emails")
@Convert(converter = EmailSetConveter.class)
private EmailSet emailSet;
```

### 4.3.7 밸류를 이용한 ID 매핑

- 식별자라는 의미를 부각시키기 위해 식별자 자체를 밸류 타입으로 만들 수 있다. → MemberId, OrderNo
- 밸류 타입을 식별자로 매핑하면 @Id 대신 `@EmbeddedId` 애너테이션을 사용한다.
- JPA에서 식별자 타입은 Serializable 타입이어야 하므로 식별자로 사용할 밸류 타입은 Serializable 인터페이스를 상속받아야 한다
- 밸류 타입으로 식별자를 구현하면 식별자에 기능을 추가할 수 있게 된다.

```java
@Entity
@Table(name = "purchase_order")
@Access(AccessType.FIELD)
public class Order {
    @EmbeddedId
    private OrderNo number;
    ...
}

@Embeddable
public class OrderNo implements Serializable {
    @Column(name = "order_number")
    private String number;
    ...
}
```

### 4.3.8 별도 테이블에 저장하는 밸류 매핑

- 별도 테이블에 저장된다고 해서 모두 엔티티인 것은 아니다.
- 밸류가 아니라 엔티티가 확실하다면 해당 엔티티가 다른 애그리거트는 아닌지 확인해야 한다.
    - 특히 자신만의 독자적인 라이프 사이클을 갖는다면 구분되는 애그리거트일 가능성이 높다.
- 애그리거트에 속한 객체가 밸류인지 엔티티인지 구분하는 방법은 고유 식별자를 갖는지를 확인하는 것이다.
    - 하지만 식별자를 찾을 때 매핑되는 테이블의 식별자를 애그리거트 구성요소의 식별자와 동일한 것으로 착각하면 안 된다.
    - 별도 테이블로 저장하고 테이블에 PK가 있다고 해서 테이블과 매핑되는 애그리거트 구성요소가 항상 고유 식별자를 갖는 것은 아니기 때문이다.
    - Ex) Order의 밸류인 OrderLine의 line_idx는 식별자이긴 하지만 Order와 연결하기 위함
- 이때 밸류를 매핑한 테이블을 지정하기 위해 @SecondaryTable과 @AttributeOverride을 사용한다.

```java
@Entity
@Table(name = "article")
@SecondaryTable(
        name = "article_content",
        pkJoinColumns = @PrimaryKeyJoinColumn(name = "id")
)
public class Article {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @AttributeOverrides({
            @AttributeOverride(
                    name = "content",
                    column = @Column(table = "article_content", name = "content")),
            @AttributeOverride(
                    name = "contentType",
                    column = @Column(table = "article_content", name = "content_type"))
    })
    @Embedded
    private ArticleContent content;
    
    ...
}

// @SecondaryTble로 매핑된 article_content 테이블을 조인
Article article = entityManager.find(Article.class, 1L);
```

### 4.3.9 밸류 컬렉션을 @Entity로 매핑하기

- 개념적으로 밸류인데 구현 기술의 한계나 팀 표준 때문에 @Entity를 사용해야 할 때도 있다.
- JPA 는 @Embeddable 타입의 클래스 상속 매핑을 지원하지 않기 때문에 @Embeddable 대신 @Entity를 이용한 상속 매핑으로 처리해야 한다.
- 밸류 타입을 @Entity로 매핑하므로 식별자 매핑을 위한 필드도 추가해야 하며 구현 클래스를 구분하기 위한 타입 식별(discriminator) 칼럼을 추가해야 한다.

<img width="696" height="376" alt="Image" src="https://github.com/user-attachments/assets/60c5e987-6759-4b2b-8530-ace7f9df53ed" />

<img width="781" height="284" alt="Image" src="https://github.com/user-attachments/assets/0e70435d-59d4-44bc-b06d-e59574584375" />

- 한 테이블에 Image와 그 하위 클래스를 매핑하므로 Image 클래스에 다음 설정을 사용한다.
    - @Inheritance 애너테이션 적용
        - strategy 값으로 SINGLE_TABLE 사용
    - @DiscriminatorColumn 애너테이션을 이용하여 타입 구분용으로 사용할 컬럼 지정

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "image_type")
@Table(name = "image")
public abstract class Image {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "image_id")
    private Long id;

    @Column(name = "image_path")
    private String path;

    @Column(name = "upload_time")
    private LocalDateTime uploadTime;

    protected Image() {
    }

    public Image(String path) {
        this.path = path;
        this.uploadTime = LocalDateTime.now();
    }

    protected String getPath() {
        return path;
    }

    public LocalDateTime getUploadTime() {
        return uploadTime;
    }

    public abstract String getUrl();

    public abstract boolean hasThumbnail();

    public abstract String getThumbnailUrl();

}

@Entity
@DiscriminatorValue("II")
public class InternalImage extends Image {
   ...
}
@Entity
@DiscriminatorValue("EI")
public class ExternalImage extends Image {
   ...
}
```

- Image가 @Entity이므로 목록을 담고 있는 Product는 @OneToMany를 이용해서 매핑을 처리한다.
- Image는 밸류이므로 독자적인 라이프 사이클을 가지는 대신 Product에 완전히 의존한다.
    - 따라서 Product를 저장할 때 함께 저장되고 Product를 삭제할 때 함께 삭제되도록 cascade 속성을 지정한다.
    - 리스트에서 Image 객체를 제거하면 DB에서 함께 삭제되도록 orphanRemoval도 true로 설정한다.

```java
@Entity
@Table(name = "product")
public class Product {
    @EmbeddedId
    private ProductId id;
    private String name;

    @Convert(converter = MoneyConverter.class)
    private Money price;
    private String detail;

    @OneToMany(cascade = {CascadeType.PERSIST, CascadeType.REMOVE},
            orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    @OrderColumn(name = "list_idx")
    private List<Image> images = new ArrayList<>();
    
    ...

    public void changeImages(List<Image> newImages) {
        images.clear();
        images.addAll(newImages);
    }
}
```

- @Entity에 대한 @OneToMany 매핑에서 컬렉션의 clear() 메서드를 호출하면 삭제 과정이 다소 비효율적이다.
    - 하이버네이트의 경우 @Entity를 위한 컬렉션 객체의 clear() 메서드를 호출하면 select 쿼리로 대상 엔티티를 로딩하고 각 개별 엔티티에 대해 delete 쿼리를 실행한다. → 성능 저하 가능성
- 하이버네이트는 @Embeddable 타입에 대한 컬렉션의 clear() 메서드를 호출하면 컬렉션에 속한 객체를 로딩하지 않고 한 번의 delete 쿼리로 삭제 처리를 수행한다.
    - 따라서 애그리거트의 특성을 유지하면서 이 문제를 해소하려면 결국 상속을 포기하고 @Embeddable로 매핑된 단일 클래스로 구현해야 한다.

### 4.3.10 ID 참조와 조인 테이블을 이용한 단방향 M-N 매핑

- 애그리거트 간 집한 연관은 성능상 이유로 피해야 하지만, 그럼에도 불구하고 사용해야 한다면 ID 참조를 이용한 단방향 집한 연관을 적용해 볼 수 있다.

```java
@Entity
@Table(name = "product")
public class Product {
    @EmbeddedId
    private ProductId id;

    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(name = "product_category",
            joinColumns = @JoinColumn(name = "product_id"))
    private Set<CategoryId> categoryIds;
    ...
```

- ID 참조를 이용한 애그리거트 간 단방향 M-N 연관은 밸류 컬렉션 매핑과 동일한 방식으로 설정한 것을 알 수 있다
    - 집합의 값에 밸류 대신 연관을 맺는 식별자가 온다
- @ElementCollection을 이용하므로 Product를 삭제할 때 매핑에 사용한 조인 테이블의 데이터도 함께 삭제된다.
    - 애그리거트를 직접 참조하는 방식을 사용하면 영속성 전파나 로딩 전략을 고민해야 하는데, ID 참조 방식을 사용함으로써 이런 고민을 없앨 수 있다.

## 4.4 애그리거트 로딩 전략

- 조회 시점에서 애그리거트를 완전한 상태가 되도록 하려면 애그리거트 루트에서 연관 매핑의 조회 방식을 즉시 로딩 `FetchType.EAGER` 으로 설정하면 된다. → 연관된 구성요소를 DB에서 함께 읽어옴
    - 하지만 컬렉션에 대해 즉시 로딩 방식을 적용할 때 N+1 문제가 발생할 수 있다.
    - 조회되는 데이터 개수가 많아지면 즉시 로딩 방식을 사용할 때 성능(실행 빈도, 트래픽, 지연 로딩 시 실행 속도 등)을 검토해 봐야 한다.
- 애그리거트가 완전해야 하는 이유
    1. 상태를 변경하는 기능을 실행할 때 애그리거트 상태가 완전해야 한다.
    2. 표현 영역에서 애그리거트의 상태 정보를 보여줄 때 필요하다.
        
        ⇒ 별도의 조회 전용 기능과 모델을 구현하는 방식 유리
        
- 하지만 JPA는 트랜잭션 범위 내에서 지연 로딩을 허용하기 때문에 실제로 변경하는 시점에 필요한 구성요소만 로딩해도 문제가 되지 않는다.
    - 일반적인 애플리케이션은 상태 변경 기능을 실행하는 빈도보다 조회 기능을 실행하는 빈도가 훨씬 높기 때문에 상태 변경을 위한 추가 조회 쿼리로 인한 실행 속도 저하는 보통 문제가 되지 않는다.
- 이런 이유로 애그리거트 내의 모든 연관을 즉시 로딩으로 설정할 필요는 없다.
- 지연 로딩은 동작 방식이 항상 동일하기 때문에 즉시 로딩처럼 경우의 수를 따질 필요가 없다는 장점이 있다.
- 애그리거트에 맞게 즉시 로딩과 지연 로딩을 선택하자.

## 4.5 애그리거트의 영속성 전파

- 애그리거트가 완전한 상태여야 한다는 것은 저장하고 삭제할 때도 하나로 처리애햐 함을 의미한다.
    - 애그리거트 루트와 연관된 모든 객체를 함께 저장/삭제
- @Embeddable 매핑 타입은 함께 저장되고 삭제되므로 cascade 속성을 추가로 설정하지 않아도 된다.
- 반면 애그리거트에 속한 @Entity 타입에 대한 매핑은 cascade 속성을 사용해서 저장과 삭제 시에 함께 처리되도록 설정해야 한다.
- @OneToOne, @OneToMany는 cascade 속성의 기본값이 없으므로 설정해야 한다.

## 4.6 식별자 생성 기능

- 식별자를 생성하는 세 가지 방식
    - 사용자가 직접 생성
    - 도메인 로직으로 생성
    - DB를 이용한 일련번호 사용
- 식별자 생성 규칙이 있다면 엔티티가 별도 서비스로 식별자 생성 기능을 분리해야 하며, 도메인 규칙이기 때문에 도메인 영역에 위치시켜야 한다.
    
    ```java
    public class ProductIdService {
    		public ProductId nextId() {
    				... 
    		}
    }		
    ```
    
    - 응용 서비스는 이 도메인 서비스를 이용해서 식별자를 구하고 엔티티를 생성한다.
- 리포지터리에서도 식별자 생성 규칙을 구현하기에 적합하다.
    
    ```java
    public interface ProductRepository {
    		...
    		ProductId nextId();
    }		
    ```
    
- DB 자동 증가 칼럼을 식별자로 사용하면 식별자 매핑에서 @GeneratedValue를 사용한다.
    - DB insert 쿼리를 실행해야 식별자가 생성되므로 도메인 객체를 리포지터리에 저장할 때 식별자가 생성된다.

## 4.7 도메인 구현과 DIP

- 현재 이 책에서는 DIP 원칙을 어기고 있다.
    - 엔티티에 구현 기술인 JPA에 특화된 @Entity, @Table, @Id, @Column 등의 애너테이션을 사용하고, 리포지터리 인터페이스는 JPA의 Repository 인터페이스를 상속하고 있다.
    
    ⇒ 도메인이 인프라에 의존하는 상황
    
- 이러한 상황을 없애기 위해서는 엔티티에서는 애너테이션을 모두 지우고 JPA를 연동하기 위한 클래스를 추가해야 하고, 리포지터리는 JPA의 Repository 상속을 제거하고 리포지터리를 구현한 클래스를 인프라에 위치시켜야 한다.
- DIP를 적용하는 주된 이유는 저수준 구현이 변경되더라도 고수준이 영향을 받지 않도록 하기 위함인데, 리포지터리와 도메인 모델의 구현 기술ㄹ은 거의 바뀌지 않는다.
- 이렇게 변경이 거의 없는 상황에서 변경을 미리 대비하는 것은 과하다.
- DIP를 완벽하게 지키면 좋겠지만 개발 편의성과 실용성을 가져가면서 구조적인 유연함은 어느정도 유지하자