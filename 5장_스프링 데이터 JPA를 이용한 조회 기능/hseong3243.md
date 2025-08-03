## 5.1 시작에 앞서
#### cqrs
- 명령(command) 모델과 조회(query) 모델을 분리하는 패턴
- 명령 모델
	- 상태를 변경하는 기능 구현 시 사용
- 조회 모델
	- 데이터를 조회하는 기능 구현 시 사용

- 엔티티, 애그리거트, 리포지터리 등은 상태를 변경할 때 주로 사용
- 즉, 도메인 모델은 명령 모델로 주로 사용
- 반면, 정렬, 페이징, 검색 조건 지정 같은 조회 기능은 즉, 조회 모델로 주로 사용

저자는 이 장에서 리포지터리 (도메인 모델에 속함), dao(데이터 접근을 의미)라는 이름을 혼용합니다.

## 5.2 검색을 위한 스펙
- 조건이 단순하다면 문제 X
- 보통 다양한 검색 조건을 조합해야 하는데, 필요한 조합마다 find를 구현하는 것은 좋은 방법이 아님
- 검색 조건을 다양하게 조합할 때는 스펙(Specification)을 사용. 이는 애그리거트가 특정 조건을 충족하는지를 검사할 때 사용하는 인터페이스
```kotlin
interface Specification<T> {  
    fun isSatisfiedBy(aggregate: T): Boolean  
}
```

- 스펙을 리포지터리에 사용하면 루트 엔티티
- dao에 사용하면 검색 결과로 리턴할 데이터 객체

- Order가 특정 고객의 주문인지 확인하는 스펙은 다음과 같습니다.
```kotlin
class OrderSpecification(  
    private val orderId: String,  
): Specification<Order> {  
    override fun isSatisfiedBy(aggregate: Order): Boolean {  
        return aggregate.orderId.memberId.id == this.orderId  
    }  
}
```

- 이제 리포지터리나 dao는 검색 대상을 걸러내는 용도로 스펙을 사용
- 필터링은 리포지터리가 해주므로 조건을 충족하는 애그리거트를 찾고 싶다면 원하는 스펙을 생성해서 전달해주기만 하면 됨
```kotlin
class MemoryOrderRepository: OrderRepository {  
    fun findAll(spec: Specification<Order>): List<Order> {  
        val allOrders = this.findAll()  
        return allOrders.filter { spec.isSatisfiedBy(it) }  
    }  
}
```

- 하지만 실제 스펙은 이렇게 구현하지 않음
- 이는 어디까지나 모든 데이터를 메모리에 보관할때나 사용 가능. 실제 스펙은 사용하는 기술에 맞춰 구현함

## 5.3 스프링 데이터 jpa를 이용한 스펙 구현
- 스프링 데이터 jpa는 검색 조건을 표현하는 Specification 인터페이스를 제공
<img width="673" height="441" alt="스크린샷 2025-07-21 오후 8 46 00" src="https://github.com/user-attachments/assets/66077b35-5f40-45d3-9afb-be708f37b6ce" />

- 타입 파라미터 T는 jpa 엔티티 타입
- toPredicate()는 jpa 크리테리아 api에서 조건을 표현하는 Predicate를 생성
<img width="669" height="658" alt="스크린샷 2025-07-21 오후 8 47 14" src="https://github.com/user-attachments/assets/07db0de1-1c3e-4239-8fbc-78d68f7fd5e5" />

- 스펙 구현 클래스를 개별로 만들지 않고 별도 클래스에 스펙 생성 기능을 모아도 됨
- 스펙 인터페이스는 함수형 인터페이스이므로 람다로 좀 더 간결하게 생성 가능
<img width="670" height="655" alt="스크린샷 2025-07-21 오후 8 51 33" src="https://github.com/user-attachments/assets/66eed621-817e-441e-9dae-1c6044811e25" />

## 5.4 리포지터리/dao에서 스펙 사용하기
- findAll() 메서드는 스펙 인터페이스를 파라미터로 가짐
- 이를 이용하여 스펙을 충족하는 엔티티를 검색
<img width="673" height="134" alt="스크린샷 2025-07-21 오후 8 53 44" src="https://github.com/user-attachments/assets/6d4940ec-2a34-4276-8205-f57777914db5" />

## 5.5 스펙 조합
- 스프링 데이터 jpa는 스펙 인터페이스끼리 조합할 수 있는 and와 or 메서드를 제공
<img width="671" height="304" alt="스크린샷 2025-07-21 오후 8 54 17" src="https://github.com/user-attachments/assets/26de1862-78e5-473e-a2d6-a013d4f64799" />

- 예를 들어 spec1.and(spec2)는 spec1과 spec2를 충족하는 조건 spec3을 생성
- 반대로 적용하는 not() 조건, nullable한 spec을 받는 where() 조건도 있음

## 5.6 정렬 지정하기
- 스프링 데이터 jpa는 두 가지 방법으로 정렬을 지정할 수 있음
  - 메서드 이름에 OrderBy 사용
  - Sort를 인자로 전달

<img width="676" height="122" alt="스크린샷 2025-07-21 오후 8 56 44" src="https://github.com/user-attachments/assets/b8a77cc2-6229-45d6-9502-78b5efc9d5f2" />

- 메서드 이름에 OrderBy를 사용하는 방법은 간단하지만 메서드 이름이 길어지는 단점이 있음
- 스프링 데이터 jpa는 정렬 순서를 지정할 때 사용할 수 있는 Sort 타입을 제공
- 인자로 Sort 객체를 전달하면 jpa는 알맞는 정렬 쿼리를 생성
<img width="673" height="121" alt="스크린샷 2025-07-21 오후 8 58 32" src="https://github.com/user-attachments/assets/72c07fe7-b44d-4e05-b8e4-53fe42b937a4" />

## 5.7 페이징 처리하기
- Sort와 마찬가지로 Pageable 은 페이징을 자동으로 처리
- PageReuqest와 Sort를 사용하면 정렬과 페이징을 조합할 수 있음
```kotlin
val pageable = PageRequest.of(1, 10, sort)
```

- Page 타입을 사용하면 목록 뿐만 아니라 전체 개수도 구할 수 있음
```kotlin
fun findAll(pageable: Pageable): Page<MemberData> 
```

- Pageable을 사용하는 메서드 리턴 타입이 Page이면 스프링 데이터 jpa는 count 쿼리도 함께 실행

## 5.8 스펙 조합을 위한 스펙 빌더 클래스
- jpa에서 제공하는 별도의 스펙 빌더 클래스는 없음
- 저자는 빌더를 만들어서 사용한다고 합니다.

## 5.9 동적 인스턴스 생성
- jpa는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공
- 원하는 컬럼을 선택적으로 가져올 수 있는 projection을 이용할 수 있음
<img width="681" height="428" alt="스크린샷 2025-07-22 오후 8 23 24" src="https://github.com/user-attachments/assets/80c79c9a-3655-4e81-b749-68d34591e90c" />

- 동적 인스턴스의 장점은 jpql을 그대로 사용하므로 지연/즉시 로딩에 대해 고민 없이 원하는 모습으로 데이터를 조회할 수 있다는 점

## 5.10 하이버네이트 @Subselect 사용
- @Subselect는 하이버네이트 확장 기능으로 쿼리 결과를 @Entity로 매핑할 수 있는 기능
<img width="767" height="662" alt="스크린샷 2025-07-22 오후 8 27 15" src="https://github.com/user-attachments/assets/facc8ab1-feeb-4dae-b9d7-049d46c601b7" />


- @Immutable, @Subselect, @Synchronize는 하이버네이트 전용 애노테이션으로 테이블이 아닌 쿼리 결과를 @Entity로 매핑할 수 있음

#### @Subselect
- 조회 쿼리를 값으로 갖는다.
- 조회 쿼리의 결과를 매핑할 테이블처럼 사용한다.
- 이를 통해 얻은 @Entity의 필드를 수정하는 경우 하이버네이트는 변경 내역을 반영하는 쿼리를 실행한다. 그러나 매핑 한 테이블이 없으므로 에러가 발생한다.
#### @Immutable
- 하이버네이트는 이 애노테이션을 사용한 엔티티의 필드가 변경되어도 db에 반영하지 않고 무시한다.
#### @Synchronize
- 해당 엔티티와 테이블 목록을 명시한다.
- 엔티티를 로딩하기 전에 명시한 테이블과 관련된 변경이 발생하면 플러시를 먼저 한다.
- 이를 통해 @Subselect로 가져온 엔티티를 로딩하는 시점에 변경 내역도 함께 가져올 수 있다.
<br/>

- @Subselect를 사용해도 @Entity와 같기 때문에 Criteria를 그대로 사용할 수 있다는 장점이 있음
- @Subselect의 값으로 지정한 쿼리를 from 절의 서브 쿼리로 사용
- 서브쿼리를 사용하고 싶지 않다면 다른 방법을 사용해서 조회 기능을 구현해야 함

