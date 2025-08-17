# 8장 애그리거트 트랜잭션 관리

## 8.1 애그리거트와 트랜잭션
- 두 개 이상의 요청에서 동일한 애그리거트 데이터를 변경한다면 일관성이 깨질 위험성이 높음
- 애그리거트에 대해서 사용할 수 있는 트랜잭션 처리 방식
  1. 선점 잠금(Pessimistic, 비관적 잠금)
  2. 비선점 잠금(Optimistic, 낙관적 잠금)
  
## 8.2 선점 잠금
- 애그리거트를 구현한 스레드가 애그리거트 사용이 끝날 때까지 다른 스레드가 해당 애그리거트를 수정하지 못하게 막는 방식
    - 한 애그리거트에 대해 요청 순서가 스레드1 -> 스레드2일 경우, 스레드2는 애그리거트에 대한 잠금이 해제될 때까지 블로킹
    - 스레드1이 애그리거트를 수정하고 트랜잭션을 커밋하면 잠금을 해제하고 스레드2가 애그리거트를 구하게 됨
    - 결국 스레드2는 스레드1이 수정한 애그리거트 내용을 조회
- JPA에서는 @Lock 애너테이션을 통해 잠금 모드를 지정
```java
public interface MemberRepository extends Repository<Member, Memberld> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdForUpdate(@Param("id") Memberld memberld);
}
```

### 선점 잠금과 교착상태
``
1. 스레드1: A 애그리거트에 대한 선점 잠금 구함
2. 스레드2： B 애그리거트에 대한 선점 잠금 구함
3. 스레드1: B 애그리거트에 대한 선점 잠금 시도
4. 스레드2： A 애그리거트에 대한 선점 잠금 시도
``
- 교착 상태 발생
  - 이 상황에서 스레드1은 영원히 B 애그리거트에 대한 선점 잠금을 구할 수 없음
  - 스레드2도 A 애그래거트에 대한 잠금을 구할 수 없음
- 해결 방법 : 최대 대기시간 지정
```java
public interface MemberRepository extends Repository<Member, Memberld> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({
        @QueryHint(name = "javax.persistence.lock.timeout", value = "2000")
    })
    @Query("select m from Member m where m.id = :id")
    Optional<Member> findByIdForUpdate(@Param("id") Memberld memberld);
}
```

## 8.3 비선점 잠금
- 동시에 접근하는 요청을 막는 것 대신 변경한 데이터를 실제 DBMS에 반영하는 시점에 변경 가능 여부를 확인하는 방식
- 애그리거트를 수정할 때마다 버전을 증가
```
UPDATE aggtable SET version = version + 1, colx = ?, coly = ?
WHERE aggid = ? and version = 현재버전
```
- 수정할 애그리거트의 버전 값이 현재 애그리거트의 버전과 동일한 경우에만 데이터를 수정
  - 수정에 성공하면 버전을 1 증가
  - 버전값이 바뀌었다면 데이터 수정에 실패
- JPA는 @Version 애너테이션을 통해 비선점 잠금을 지원
```java
@Entity
@Table(name = "purchase—Order")
@Access(AccessType.FIELD)
public class Order {
    @EmbeddedId
    private OrderNo number;
    @Version
    private long version;
}
```
- 비선점 잠금 사용 시, 트랜잭션 충돌이 발생하면 OptimisticLockingFailureException 발생
- 표현 영역에서는 Exception 발생 여부에 따라 트랜잭션 충돌이 발생했는 지 확인 가능
```java
@PostMapping("/changeShipping")
public String changeshipping(ChangeShippingRequest changeReq) {
    try {
        changeShippingService.changeshipping(changeReq);
        return "changeShippingSuccess";
    } catch(OptimisticLockingFailureException ex) {
        // 누군가 먼저 같은 주문 애그리거트를 수정했으므로
        // 트랜잭션이 충돌했다는 메시지를 보여준다.
        return "changeShippingTxConflict";
    }
}

```
- 비선점 잠금 방식을 여러 트랜잭션으로 확장하려면 버전 정보도 함께 사용자에게 전달
- 전달 받은 버전을 상태 변경 요청 dto에 포함
```java
public class StartShippingRequest {
    private String orderNumber;
    private long version;
}
```
- 응용 서비스는 전달 받은 버전값을 이용하여 현재 애그리거트 버전과 일치하는 지 확인
```java
@Transactional
public void startshipping(StartShippingRequest req) {
    Order order = orderRepository.findById(new OrderNo(req.getOrderN니mber()));
    checkOrder(order);
    if (!order.matchVersion(req.getVersion())) {
        throw new VersionConflictException();
    }
    order.startShipping();
}
```

### 강제 버전 증가
- 애그리거트 루트 엔티티가 아닌 다른 엔티티나 밸류가 변경되더라도 루트 엔티티의 버전 값을 증가시켜야하는 경우
- JPA에서는 EntityManager#find() 메서드를 통해 엔티티를 조회할 때 강제로 버전을 증가하는 잠금 모드를 지원
```java
@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;
    @Override
    public Order findByIdOptimisticLockMode(OrderNo id) {
        return entityManager.find(
            Order.class, id, LockModeType.OPTIMISTIC_FORCE_INCREMENT);
    }
}
```
## 8.4 오프라인 선점 잠금
- 여러 트랜잭션에 걸쳐 동시 변경을 막는 잠금 방식
- 오프라인 선점 잠금은 크게 잠근 선점 시도, 잠금 확인, 잠금 해제, 잠금 유효시간 연장의 네 가지 기능이 필요
```java
public interface LockManager {
    // 잠금 대상 타입과 식별자를 전달 -> 잠금의 식별자인 LockId를 반환
    Lockld tryLock(String type, String id) throws LockException;

    void checkLock(LockId lockld) throws LockException;

    void releaseLock(LockId lockld) throws LockException;

    void extendLockExpiration(LockId lockld, long inc) throws LockException;
}
```
- 오프라인 선점 잠금 사용 과정
1. 수정 폼에 데이터를 전송할 때 오프라인 잠금을 획득하고 LockId를 전송
```java
public DataAndLockld getDataWithLock(Long id) {
    // 1. 오프라인 선점 잠금 시도
    Lockld lockld = lockManager.tryLock("data", id);
    // 2. 기능 실행
    Data data = someDao.select(id);
    return new DataAndLockld(data, lockld);
}

@RequestMapping("/some/edit/{id}")
public String editForm(@PathVariable("id") Long id, ModelMap model) {
    DataAndLockld dl = dataService.getDataWithLock(id);
    model.addAttribute("data", dl.getData());
    model.addAttrib니te("lockld", dl.getl_ockld());
    return "editForm";
}
```
2. 전달받은 LockId를 통해 잠금이 유효한지 확인 -> 기능 실행 -> 잠금 해제
```java
public void edit(EditRequest editReq, Lockld lockld) {
    // 1. 잠금 선점 확인
    lockManager.checkLock(lockld);
    // 2. 기능 실행
    // 3. 잠금 해제
    lockManager.releaseLock(lockld);
}

@RequestMapping(value = "some/edit/{id}", method = RequestMethod.POST)
public String edit(@PathVariable("id") Long id,
@ModelAttribute("editReq") EditRequest editReq,
@RequestParamC("id") String lockldValue) {
    editReq.setld(id);
    someEditService.edit(editReq, new Lockld(lockldValue));
}
```

- 오프라인 잠금이 필요한 경우 잠금을 저장하는 DDL을 정의하고 LockManager 구현체를 구현하여 사용
- DDL 예시
```
create table locks (
  type varchar(255),
  id varchar(255),
  lockid varchar(255),
  expiration_time datetime,
  primary key ('type', id)
  )
  create unique index locks idx ON locks (lockid);
```
