# 스프링 데이터 JPA를 이용한 조회 기능

## 5.1 시작에 앞서
- CQRS 패턴 : 명령 모델과 조회 모델을 분리하는 패턴
  - 명령 모델 : 상태를 변경하는 기능, 도메인 모델에 주로 사용
  - 조회 모델 : 데이터를 조회, 조회 기능에 사용
  
## 5.2 검색을 위한 스펙
- 스펙(specification) : 애그리거트가 특정 조건을 충족하는지 검사할때 사용하는 인터페이스
```java
public interface Speficiation<T> {
    public boolean isSatisfiedBy(T agg);
}
```
  - agg 파라미터 : 검색 대상이 되는 객체
  - isSatifiedBy() 는 검사 대상 객체가 조건을 충족하면 true, 그렇지 않으면 false
  - Order 애그리거트 객체가 특정 고객의 주문인지 확인하는 스펙
```java
public class OrdererSpec implements Specification<Order> {
    private String ordererld;

    public OrdererSpec(String ordererld) {
        this.ordererld = ordererld;
    }

    public boolean isSatisfiedBy(Order agg) {
        return agg.getOrdererId().getMemberId().getId().equals(ordererId);
    }
}
```
  - 검색 대상을 거러내는 용도로 스펙을 사용
```java
public class MemoryOrderRepository implements OrderRepository {
    public List<Order> findAll(Specification<Order> spec) {
        List<Drder> aUOrders = findAll();
        return aUOrders.stream()
            .filter(order -> spec.isSatisfiedBy(order))
            .toList();
    }
}

// 검색 조건을 표현하는 스펙을 생성해서
Specification<Order> ordererSpec = new OrdererSpec("madvirus");

// 리포지터리에 전달
List<Order> orders = orderRepository.findAll(ordererSpec);
```
- 이 방법은 저장되어있는 모든 애그리거트 객체를 조회해오고 필터링하는 방식인데, 모든 데이터를 메모리에 보관하는 것은 어렵고 조회 성능 저하로 이어질 수 있음

## 5.3 스프링 데이터 JPA를 이용한 스펙 구현
- 스프링 데이터 JPA는 검색 조건을 표현하기 위해 Specification 인터페이스를 제공
```java
public interface Specification<T> extends Serializable {
    @Nullable 
    Predicate toPredicate(Root<T> root, 
                          CriteriaQ니ery<?> query, 
                          CriteriaBuilder cb);
}
```
- 제네릭 파라미터 T : JPA 엔티티 타입
- toPredicat() 는 JPA 크리테리아 API 에서 조건을 표현하는 Predicate 를 생성
- 스펙 생성 예시
```java
public class OrderSummarySpecs {
    public static Specification<OrderSummary> ordererId(String ordererld) {
        return (Root<OrderSummary> root, CriteriaQuery<?> query,
                CriteriaBuilder cb) ->
            cb.equal(root.<String>get("ordererld"), ordererld);
    }

    public static Specification<OrderSummary> orderDateBetween(
        LocaWateTime from, LocalDateTime to) {
        return (Root<OrderSummary> root, CriteriaQuery<?> query,
                CriteriaBuilder cb) ->
            cb.between(root.get(OrderSummary_.orderDate), from, to);
    }
}
```
  - 람다식을 이용해서 스펙을 생성

## 5.4 리포지터리/DAO에서 스펙 사용하기
```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> { 
    List<OrderSummary> findAll(Specification<OrderSummary> spec);
}
```
```java
Specification<OrderSummary> betweenSpec = OrderSummarySpecs.orderDateBetween(from, to);
List<OrderSummary> results = orderSummaryDao.findAll(betweenSpec);
```

## 5.5 스펙 조합
- JPA는 스펙을 조합할 수 있는 두 메서드를 제공
  - and : 두 스펙을 모두 충족하는 조건을 표현하는 스펙을 생성
  - or : 두 스펙 중 하나 이상을 충족하는 조건을 표현하는 스펙을 생성
```java
Specification<OrderSummary> spec = OrderSummarySpecs .ordererld("userl")
.and(OrderSummarySpecs.orderDateBetween(froni, to));
```
- where() : 스펙 인터체이스의 정적 메서드로 null을 전달하면 아무 조건도 생성하지 않는 스펙 객체를 리턴, null이면 인자로 받은 스펙 객체를 리턴
```java
Specification<OrderSummary> spec = Specification.where(createNullableSpec()).and (createOtherSpec());
```

## 5.6 정렬 지정하기
- JPA에서 정렬을 지정하는 방법
  - 메서드 이름에 OrderBy 를 사용하여 정렬 기준 지정
```java
List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererld);
```
  - Sort 를 인자로 전달
```java
List<OrderSummary> findByOrdererId(String ordererld, Sort sort);
List<OrderSummary> findAU(Specification<OrderSummary> spec, Sort sort);
```
    - 스프링 데이터 JPA는 파라미터로 전달 받은 Sort를 사용해서 알맞게 정렬 쿼리를 생성

## 5.7 페이징 처리하기
- 스프링 데이터 JPA는 페이징 처리를 위해 Pageable 타입을 이용
```java
List<MemberData> findByNameLike(String name, Pageable pageable);
```
- Pageable 객체는 PageRequest 클래스를 이용해서 생성
```java
PageRequest pageReq = PageRequest.of(1, 10); // 페이지 번호, 페이지 크기
List<MemberData> user = memberDataDao.findByNameLike("사용시%", pageReq);
```
- sort와 같이 생성도 가능
```java
Sort sort = Sort.by("name").descending();
PageRequest pageReq = PageRequest.of(1, 2, sort);
```
- Pageable을 사용하는 메서드의 리턴타입이 Page일 경우 목록조회 쿼리와 함께 COUNT 쿼리도 실행하여 조건에 해당하는 데이터 개수도 구함
```java
Page<MemberData> findByBlocked(boolean blocked, Pageable pageable);
```
```java
Pageable pageReq = PageRequest.of(2, 3);
Page<MemberData> page = memberDataDao.findByBlockec(false, pageReq);
List<MemberData> content = page.getContent(); // 조회 결과 목록
long totalElements = page.getTotalElements(); // 조건에 해당하는 전체 개수
int totalPages = page.getTotalPages(); // 전체 페이지 번호
int number = page.getNumber(); // 현재 페이지 번호
int numberOfElements = page .getNumberOfElements(); // 조회 결과 개수
int size = page.getSize(); // 페이지 크기
```

## 5.8 스펙 조합을 위한 스펙 빌더 클래스
- 스펙 빌더를 사용하여 스펙을 생성
```java
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
    .ifTrueSearchRequest.isOnlyNotBlocked(), () -> MemberDataSpecs.nonBlocked())
    .ifHasText(searchRequest.getName(), name -> MemberDataSpecs.nameLike(searchRequest.getName()))
    .toSpec();
List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0, 5));
```
- 메서드 호출 체인으로 연속된 변수 할당을 줄여 코드의 가독성 향상

## 5.9 동적 인스턴스 생성
- JPA는 쿼리 결과에 임의의 객체를 동적으로 생성할 수 있는 기능을 제공
```java
public interface OrderSummaryDao
extends Repository<OrderSummary, String> {
    @Query("""
    select new com.myshop.order.query.dto.OrderView(
        o.number, o.state, m.name, m.id, p.name
    )
    from Order o join o.orderLines ol. Member m. Product p
    where o.orderer.memberld .id = :ordererld
    and o.orderer.memberld.id = m.id
    and index(ol) = 0
    and ol.prod니ctld.id = p.id
    order by o.number.number desc
    """)
    List<OrderView> findOrderView(String ordererld);
}
```
- new 키워드를 사용하요 생성자에 인자로 전달할 값을 지정
- 지연/즉시 로딩 고민 없이 원하는 데이터를 뽑아 한번에 조회할 수 있음

## 5.10 하이버네이트 @Subselect 사용
- 하이버네이트는 JPA 확장 기능으로 @Subselet 를 제공
- @Subselect : 쿼리 결과를 @Entity로 매핑할 수 있는 유용한 기능
- @Subselect의 값으로 지정한 쿼리를 from 절의 서브 쿼리로 사용
```java
@Entity
@Immutable
@Subselect(
    """
    select o.order_number as number,
    o.versiono.orderer_id, o.orderer_name,
    o.total_amounts, o.receiver_name, o.state, o.order_date,
    p.product_id, p.name as prod니ct_name
    from purchase_order o inner join order_line ol
    on o.order_number = ol.order_number
    cross join product p
    where ol.line_idx = 0
    and ol.product_id = p.product_id"""
)
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {...}
```
- @Immutable, @Subselect, @Synchronize는 하이버네이트 전용 애너테이션인, 테이블이 아닌 쿼리 결과를 @Entity로 매핑
- 하이버네이트는 select 쿼리의 결과를 매핑할 테이블처럼 사용
- @Entity의 매핑 필드를 수정하면 하이버네이트는 변경 내역을 반영하는 update 쿼리를 실행 -> 매핑한 테이블이 없기에 에러 발생
- @Immutable을 사용하면 하이버네이트는 해당 엔티티의 매핑 필드/프로퍼티가 변경되도 DB에 반영하지 않고 무시
- @Synchronize는 해당 엔티티와 관련된 테이블 목록을 명시 -> 엔티티를 로딩하기 전에 지정한 테이블과 관련된 변경이 발생하면 플러시 실행