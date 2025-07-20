# 애그리거트

## 3.1 애그리거트
- 애그리거트 : 관련된 객체를 하나의 군으로 묶은 것
- 복잡한 도메인을 애그리거트로 묶어 구성하면 상위 수준에서 도메인 모델 간의 관계를 파악할 수 있음
- 복잡도가 낮아져 효율적으로 도메인 기능을 확장/변경
- 애그리거트에 속한 객체는 유사하거나 동일한 라이프 사이클을 갖음
  - ex) Order 애그리거트 - Orderer, ShippingInfo
- 한 애그리거트에 속한 객체를 다른 애그리거트에 속하지 않음
- 다른 애그리거트와는 독립적
  - ex) 주문 애그리거트가 회원의 비밀번호를 변경하지 않음
- 애그리거트의 경계 설정 기준 : 도메인 규칙과 요구사항
  - ex) 주문할 상품 개수, 배송지 정보, 주문자 정보는 하나의 애그리거트를 구성
- 'A가 B는 갖는다' 라고 해석되는 요구사항은 반드시 A와 B가 한 애그리거트에 속한다는 것을 의미하지 않음
  - ex) 상품 정보에 리뷰 정보를 보여줘야한다 -> 상품과 리뷰는 함께 생성되지 않고 변경되지 않음. 각각 다른 라이프사이클을 갖으며 다른 애그리거트임

## 3.2 애그리거트 루트
- 도메인 규칙을 지키려면 애그리거트에 속한 모든 객체가 정상 상태를 가져야한다.
  - ex) 주문 애그리거트에서 OrderLine을 변경하면 Order의 totalAmounts 를 다시 계산
- 애그리거트 루트 : 애그리거트 전체를 관리하는 주체
  - 핵심 역할 : 애그리거트의 일관성이 깨지지 않도록 하는 것
    - ex) 주문 애그리거트는 배송지 변경 기능을 제공하고 애그리거트 루트인 Order 가 이를 구현한 메서드를 제공
  - 애그리거트 루트의 메서드는 애그리거트에 속한 객체들의 일관성을 유지해야함
    - ex) 배송지 정보 수정
```java
public class Order {
    // 애그리거트 루트는 도메인 규칙을 구현한 기능을 제공한다.
    public void changeShippingInfo(ShippingInfo newShippinglnfo) {
        verifyNotYetShipped();
        setShippinglnfo(newShippinglnfo);
    }
}
```
- 애그리거트 외부에서 애그리거트에 속한 객체를 직접 변경하면 일관성이 깨질 위험성이 존재
  - ex) 직접 ShippingInfo 를 변경
```java
Shippinginfo si = order.getShippingInfo();
si.setAddress(newAddress);
```

### 도메인 모델 규칙

1. 단순히 필드를 변경하는 set 메서드를 public 으로 만들지 않는다
   - 도메인 로직의 응집도가 낮아져 유지 보수할 때 분석/수정 하는데 많은 시간 소요
   - set 이름을 사용하지 보다는 cacel, changePassword 처럼 의미가 분명한 이름을 사용하자
2. 밸류 타입은 불변으로 구현한다
   - 외부에서 애그리거트의 내부 상태를 변경하지 못하므로 일관성이 깨질 가능성을 줄일 수 있음

### 애그리거트 루트의 기능 구현

- 애그리거트 루트는 내부의 다른 객체를 조합하여 기능을 완성
  - ex) Order는 OrderLine 목록을 사용하여 총 주문 금액을 계산
- 애그리거트 루트는 기능 실행을 위임
  - ex) OrderLines 에 changeOrderLines(), getTotalAmounts() 메서드 위임
```java
public class Order {
    private OrderLines orderLines;

    public void changeOrderLines(List<OrderLine> newLines) {
        orderLines.changeOrderLines(newLines);
        this.totalAmounts = orderLines.getTotalAmounts();
    }
}
```

### 트랜잭션 범위

- 트랜잭션 범위는 작을 수록 좋음. 트랜잭션의 크기가 커서 잠금 대상이 많아지면 낮은 성능을 초래
- 한 트랜잭션에서는 한 개의 애그리거트만 수정해야함
  - 한 트랜잭션에서 두 개 이상의 애그리거트를 수정하면 충돌 가능성이 높아짐
  - ex) 배송지 정보를 변경하면 동시에 회원의 주소를 변경
  - 애그리거트에서 다른 애그리거트의 기능에 의존하면 결합도가 증가하여 수정 비용 증가
  - 되도록이면 응용 서비스에서 두 애그리거트를 수정하도록 구현
  - 도메인 이벤트를 사용하여 한 트랜잭션에서 한개의 애그리거트를 수정하면서 동기나 비동기로 다른 애그리거트의 상태를 변경할 수도 있음

## 3.3 리포지터리와 애그리거트
- 객체의 영속성을 처리하는 리포지터리는 애그래거트 단위로 존재
- 리포지터리는 기본적으로 save, findById 메서드 제공
- 리포지터리는 애그리거트 전체를 저장소에 영속화
  - ex) Order 애그리거트를 저장할 때 해당 애그리거터에 속한 모든 구성 요소에 매핑된 테이블에 데이터를 저장

## 3.4 ID를 이용한 애그리거트 참조
- 애그리거트도 다른 애그리거트 루트를 통해 다른 애그리거트를 참조
  - ex) 주문 애그리거트에 속한 Orderer 는 주문한 회원을 참조하기 우해 회원 애그리거트의 루트인 Member를 참조
```java
public class Orderer {
    private Member member;
}
```

### 필드 참조의 단점

필드로 다른 애그리거트를 참조하는 것은 구현의 편리함을 제공하지만 편한 탐색 오용, 성능 고민, 확장의 여려움의 단점도 존재

1. 편한 탐색 오용
   - 다른 애그리거트의 상태를 쉽게 변경할 수 있어 애그리거트가의 결합도를 높일 수 있음
2. 성능 고민
   - JPA 사용 시, 애그리거트와 연관된 애그리거트를 조회해올 때 지연 로딩, 즉시 로딩 등을 고려하여 로딩 전략을 결정해야 함
3. 확장의 어려움
   - 부하를 분산하기 위해 하위 도메인 별로 시스템을 분리해야할 때, 도메인 마다 사용하는 DBMS 가 다르다면 JPA 와 같은 단일 기술을 사용할 수 없게 됨

#### 해결 방법 : ID를 통한 참조
```java
public class Orderer {
    private MemberId memberId;
}
```
- ID로만 참조하므로 한 애그리거트에 속한 객체들만 참조로 연결
- 애그리거트 같의 의존이 제거되므로 응집도 증가
- 참조하는 애그리거트가 필요하다면 응용 서비스에서 ID를 이용하여 로딩
```java
public class ChangeOrderService {
    @Transactional
    public void changeShippingInfo(OrderId id, Shippinginfo newShippinglnfo, boolean useNewShippingAddrAsMemberAddr) {
        Order order = orderRepository.findbyld(id);
        order.changeShippinglnfo(newShippinglnfo);
        if (useNewShippingAsMemberAddr) {
            // ID를 이용해서 참조하는 애그리거트를 구한다.
            Member member = memberRepository.findById(order.getOrderer().getMemberId());
        }
    }
}
```
- 다른 애그리거트를 수정하는 문제를 방지
- 애그리거트 별로 다른 구현 기술(DBMS) 사용 가능
- ID 참조 시, 조회 성능이 낮아질 수 있음 
    - ex) join 을 통해 한번에 모든 데이터를 가져올 수 있음에도 각각 ID로 조회하는 쿼리를 실행
    - 해결 방법 : 조회를 위한 별도의 DAO를 만들고 DAO의 조회 메서드에서 조인을 시용해 필요한 데이터 로딩
```java
@Repository
public class JpaOrderViewDao implements OrderViewDao {
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<OrderView> selectByOrderer(String ordererld) {
        String selectQuery =
            "select new com.myshop.order.application.dto.OrderViewCOj m, p) " +
                "from Order o join o.orderLines ol. Member m. Product p " +
                "where o.orderer.memberld .id = :ordererld " +
                "and o.orderer.memberld = m.id " +
                "and index(ol) = 0 " +
                "and ol.productld = p.id " +
                "order by o.number.number desc";
        TypedQuery<OrderView> query =
            em.createQuery(selectQuery, OrderView.class);
        query.setParameter("ordererld", ordererld);
        return q니ery.getResultl_ist();
    }
}
```
  - 애그리거트 마다 다른 DBMS를 사용한다면 join을 사용할 수 없으므로 캐시나 조회 전용 저장소를 고려해볼 수 있음

## 3.5 애그리거트 간 집합 연관
### 1-N 관계
- 상품이 하나의 카테고리를 가질 경우
```java
public class Category {
    private Set<Product> products; // 다른 애그리거트에 대한 1-N 연관
}
```
- 모든 product를 조회하게 되므로 N-1로 연관지어 조회
```java
public class Product {
    private Categoryld categoryld;
}

public class ProductListService {
    public Page<Product> getProductOfCategory(Long categoryld, int page, int size) {
        Category category = categoryRepository.findByld(categoryld);
        checkcategory(category);
        List<Product> products = productRepository.findByCategoryld(category.getld(), page, size);
        int totalCount = productRepository.countsByCategoryld(category.getld());
    }
}
```

### M-N 관계
- 상품이 두 개 이상의 카테고리를 가질 경우
```java
public class Product {
    private Set<CategoryId> categorylds;
}
```
- RDBMS 를 이용할 경우 조인 테이블(중간 테이블)을 사용
- JPA 를 사용할 경우 ID 참조를 이용한 M-N 단방향 연관 구현
```java
@Entity
@Table(name = "product")
public class Product {
    @EmbeddedId
    private Productld id;
    
    @ElementCollection
    @CollectionTable(name = "product_category", joinColumns = @JoinColumn(name = "product_id"))
    private Set<CategoryId> categorylds;
}

```
- `:catId member of p.categoriIds` 를 통해 catId 에 해당하는 카테고리에 속한 Product 목록 조회 가능

## 3.6 애그리거트를 팩토리로 사용하기
### 요구 사항 : 차단 상태가 아닌 상점만 상품을 등록할 수 있다
- 응용 서비스에서 구현
```java
public class RegisterProductService {
    public Productld registerNewProduct(NewProductRequest req) {
        Store store = storeRepository.findByld(req.getStoreld());
        checkNuU(store);
        if (store.isBlockedO) {
            throw new StoreBlockedException();
        }
        Productld id = productRepository.nextld();
        Product product = new Product(id, store.getld(),...생략);
        productRepository.save(product);
        return id;
    }
}
```
  - 도메인 로직이 응용 서비스에 노출

- Product 를 생성하면서 도메인 로직을 구현하는 Store 애그리거트
```java
public class Store {
    public Product createProduct(ProductId newProductld, ...생략) {
        if (isBlockedO) throw new StoreBlockedException();
        return new Product(newPcoductld, getld(), ...생략);
    }
}
```
  - 도메인 로직이 한 곳에 위치 -> 응집도 증가
  - Product 생성 규칙이 변경된다면 도메인 영역인 Store 만 변경, 응용 서비스는 영향을 받지 않게 됨