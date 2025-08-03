## 5.1 시작에 앞서

- CQRS: 명령 모델과 조회 모델을 분리하는 패턴
    - 명령 모델: 상태를 변경하는 기능
    - 조회 모델: 데이터를 조회하는 기능

## 5.2 검색을 위한 스펙

- 스펙: 애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스
    - 검색 조건을 다양하게 조합해야 할 때 사용
    
    ```java
    public interface Specification<T> {
    		public boolean isSatisfiedBy(T agg); // agg는 검사 대상이 되는 객체
    		// 조건 충족하면 true 반환
    }		
    ```
    
    - 리포지터리나 DAO는 검색 대상을 걸러내는 용도로 스펙 사용
    - 특정 조건을 충족하는 애그리거트를 찾고 싶으면 원하는 스펙을 생성해서 리포지터리에 전달
    
    ```java
    Specification<Order> orderSpec = new OrderSpec("madvirus");
    List<Order> orders = orderRepository.findAll(orderSpec);
    ```
    

## 5.3 스프링 데이터 JPA를 이용한 스펙 구현

- 스프링 데이터 JPA는 검색 조건을 표현하기 위한 인테페이스인 Specification을 제공한다.

```java
public interface Specification<T> extends Serializable {
	@Nullable
	Predicate toPredicate(Root<T> root, 
    				CriteriaQuery<?> query, 
    				CriteriaBuilder cb);
}
```

- 스펙 인터페이스에서 지네릭 타입 파라미터 `T` 는 JPA의 엔티티타입을 의미
- JPA 크리테리아 API에서 조건을 표현할 때 Predicate 생성

```java
public class OrdererIdSpec implements Specification<OrderSummary> {
    private String ordererId;

    public OrdererIdSpec(String ordererId) {
        this.ordererId = ordererId;
    }

    @Override
    public Predicate toPredicate(Root<OrderSummary> root,
					    CriteriaQuery<?> query,
    					CriteriaBuilder cb) {
        return cb.equal(root.get(OrderSummary_.ordererId), ordererId);
        // 생성자로 전달받은 orderId와 동일한지 비교하는 Predicate 생성
    }
}
```

## 5.4 리포지터리/DAO에서 스펙 사용하기

- 스펙을 충족하는 엔티티를 검색하고 싶다면 findAll() 메서드 사용
- 이때 파라미터로 스펙 인터페이스를 갖는다.

```java
public interface OrderSummaryDao extends Repository<OrderSummary, String> {
    List<OrderSummary> findAll(Specification<OrderSummary> spec);
}

// 스펙 객체 생성하고
Specification<OrderSummary> spec = new OrdererIdSpec("user1");
// findAll() 메서드를 이용해서 검색
List<OrderSummary> results = orderSummaryDao.findAll(spec);
```

## 5.5 스펙 조합

- 스펙 인터헤이스는 스펙을 조합할 수 있는 두 메서드를 제공한다. → `and`, `or`
    - and : 두 스펙을 모두 충족하는 조건
    - or : 두 스펙 중 하나 이상 충족하는 조건

```java
public interface Specification<T> extends Serializable {

	default Specification<T> and(@Nullable Specification<T> other) {...}
	default Specification<T> or(@Nullable Specification<T> other) {...}

	@Nullable
	Predicate toPredicate(Root<T> root, 
    				CriteriaQuery<?> query, 
    				CriteriaBuilder cb);
	}
}
```

```java
// 조합 예시
default Specification<T> and(@Nullable Specification<T> other) { ... }
default Specification<T> or(@Nullable Specification<T> other) { ... }
static <T> Specification<T> not(@Nullable Specification<T> spec) { ... }
static <T> Specification<T> where(@Nullable Specification<T> spec) { ... }
```

## 5.6 정렬 지정하기

- 스프링 데이터 JPA의 정렬 지정하는 방법
    - 메서드 이름에 OrderBy를 사용해서 정렬 기준 지정
    - Sort를 인자로 전달
- OrderBy
    - 메서드 이름이 길어지고, 메서드 이름을 변경해야 만 정렬 순서를 변경할 수 있다.
    
    ```java
    public interface OrderSummaryDao extends Repository<OrderSummary, String> {
        List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererId);
    }    
    ```
    
- Sort 타입
    
    ```java
    public interface OrderSummaryDao extends Repository<OrderSummary, String> {
        List<OrderSummary> findByOrdererId(String ordererId, Sort sort);
    }
    
    Sort sort1 = Sort.by("number").ascending();
    Sort sort2 = Sort.by("orderDate").ascending();
    Sort sort = Sort1.and(sort2);
    List<OrderSummary> results = orderSummaryDao.findByOrdererId("user1",sort);
    ```
    

## 5.7 페이징 처리하기

- JPA는 페이징 처리를 위해 Pageable 타입을 이용한다.
    - find 메서드에 Pageable 타입 파라미터를 사용하면 페이징을 자동으로 처리한다.

```java
public interface MemberDataDao extends Repository<MemberData, String> {
    
  List<MemberData> findByNameLike(String name, Pageable pageable);

}
```

- Pageable 타입 객체는 PageRequest 클래스를 이용해서 생성한다.

```java
// 페이지 1번, 총 10개 데이터, sort로 정렬 요청
PageRequest pageReq = PageRequest.of(1, 10, sort);
```

- Page 타입을 사용하면 데이터 목록과 전체 개수도 구할 수 있다.
    - 자동으로 count 쿼리를 날리기 때문에 필요하지 않다면 List 타입을 선택하자

## 5.8 스펙 조합을 위한 스펙 빌더 클래스

- 스펙을 생성하다보면 스펙을 조합하는 경우가 생기는데 if와 스펙을 조합하는 코드가 같이 있다면 복잡성이 증가할 수 있음
- 스펙 빌더를 사용한다면 코드 가독성과 구조를 단순화 시킬 수 있을

```java
Specification<MemberData> spec = SpecBuilder.builder(MemberData.class)
		.ifTrue(searchRequest.isOnlyNotBlocked(),
				() -> MemberDataSpecs.nonBolcked())
		.ifHasText(searchRequest.getName(),
				() -> MemberDataSpecs.nameLike(searchRequest.getName()))
		.toSpec();
List<MemberData> result = memberDataDao.findAll(spec, PageRequest.of(0, 5));		
```

```java
public class SpecBuilder {
    public static <T> Builder<T> builder(Class<T> type) {
        return new Builder<T>();
    }

    public static class Builder<T> {
        private List<Specification<T>> specs = new ArrayList<>();

        public Builder<T> and(Specification<T> spec) {
            specs.add(spec);
            return this;
        }

        public Builder<T> ifHasText(String str,
                                    Function<String, Specification<T>> specSupplier) {
            if (StringUtils.hasText(str)) {
                specs.add(specSupplier.apply(str));
            }
            return this;
        }

        public Builder<T> ifTure(Boolean cond,
                                 Supplier<Specification<T>> specSupplier) {
            if (cond != null && cond.booleanValue()) {
                specs.add(specSupplier.get());
            }
            return this;
        }

        public Specification<T> toSpec() {
            Specification<T> spec = Specification.where(null);
            for (Specification<T> s : specs) {
                spec = spec.and(s);
            }
            return spec;
        }
    }
}
```

## 5.9 동적 인스턴스 생성

- JPA 는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공한다.

```java
@Query("""
		select new com.myshop.order.query.dto.OrderView(
				o.number, o.state, m.name, m.id, p.name
		)
		from Order o join o.orderLines o1 ...
)
List<OrderView> findOrderView(String orderId);		
```

- JPQL을 그대로 사용하기에 지연/즉시 로딩과 같은 고민이 필요없이 데이터를 조회할 수 있다.

## 5.10 하이버네이트 @Subselect 사용

- 하이버네이트는 JPA 확장 기능으로 @Subselect 기능을 제공한다.
    - 쿼리 결과를 @Entity로 매핑할 수 있는 기능
- @Immutable, @Subselect, @Synchronize는 하이버네이트 전용 애너테이션으로 테이블이 아닌 쿼리 결과를 @Entity로 매핑할 수 있다.
- @Subselect로 조회한 @Entity는 수정할 수 없다.
- @Immutable을 사용하면 하이버네이트는 해당 엔티티의 매핑 필드/프로퍼티가 변경되도 DB에 반영하지 않고 무시한다.
- @Synchronize는 해당 엔티티와 관련된 테이블 목록을 명시하며, 엔티티를 로딩하기 전에 지정한 테이블과 관련된 변경이 발생하면 플로시를 먼저 한다.