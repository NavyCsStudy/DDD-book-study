## 8.1 애그리거트와 트랜잭션
운영자의 배송 상태 변경과 사용자의 배송지 주소 변경이 동시에 이루어진다면 어떻게 될 것인가?
<img width="586" height="333" alt="스크린샷 2025-08-03 오전 10 11 16" src="https://github.com/user-attachments/assets/0b3ab0e0-83f7-4ebb-a534-d89423bbc8f9" />

- 운영자 스레드, 사용자 스레드는 각자의 트랜잭션에서 주문 애그리거트를 나타내는 다른 객체를 구한다.
  - 개념적으로는 동일
  - 물리적으로는 다름
  - 각 트랜잭션은 서로 영향을 주지 않음
  - 사용자 입장에서는 아직 배송 상태가 변경되지 않았기에 배송지 정보 변경이 가능
 
- 배송 상태, 배송지 정보가 변경이 db에 반영되면서 애그리거트의 일관성이 깨진다. 이러한 문제를 방지하려면 두 가지 방법 중 하나를 선택해야 한다.
  - 운영자가 배송지 정보를 조회하고 상태를 변경하는 동안, 고객이 애그리거트를 수정하지 못하게 막음
  - 운영자가 배송지 정보를 조회한 이후에 고객이 정보를 변경하면, 운영자가 애그리거트를 다시 조회한 뒤 수정하도록 함

- 이를 위해 애그리거트가 사용할 수 있는 대표적인 트랜잭션 처리 방식에는 비관락과 낙관락 두가지 방식이 있다.

## 8.2 선점 잠금(비관락)
비관락은 애그리거트를 구한 스레드가 애그리거트 사용이 끝날 때까지 다른 스레드가 해당 애그리거트를 수정하지 못하게 막는 방식이다.

<img width="411" height="369" alt="스크린샷 2025-08-03 오전 10 16 08" src="https://github.com/user-attachments/assets/4dff50d7-e4cf-444b-aabc-57c74c27f75d" />

1. 스레드1이 애그리거트를 구하면 스레드2는 스레드1이 잠금을 해제할 때까지 블로킹(blocking)
2. 스레드1이 트랜잭션을 커밋
3. 대기하고 있던 스레드2가 애그리거트에 접근
4. 스레드2는 스레드1이 수정한 내용을 보게 된다.

한 스레드가 애그리거트를 구하고 데이터를 수정하는 동안 다른 스레드가 수정할 수 없으므로 동시에 애그리거트를 수정할 때 발생하는 데이터 충돌 문제를 해소할 수 있다.
<img width="489" height="374" alt="스크린샷 2025-08-03 오전 10 18 18" src="https://github.com/user-attachments/assets/e3a6463f-1200-4d26-8c5f-538ed7ac4a53" />

- 처음과 달리 고객은 배송 상태의 주문 애그리거트를 변경하려고 시도
- 고객은 '이미 배송시 시작되어 배송지를 변경할 수 없습니다'와 같은 안내 문구를 본다.
<br/>

비관락은 보통 dbms가 제공하는 행단위 잠금을 사용해 구현. 대다수의 dbms는 for update와 같은 쿼리를 사용해 특정 레코드에 한 커넥션만 접근할 수 있는 잠금을 제공한다.

스프링 데이터 jpa는 `@Lock` 애노테이션을 사용해서 잠금 모드를 지정할 수 있다.

### 8.2.1 선점 잠금과 교착 상태
비관락 사용시에는 데드락(교착 상태)이 발생하지 않도록 주의해야 한다. 다음과 같을 때, 스레드1과 스레드2는 영원히 비관락을 구할 수 없다.
1. 스레드1: a 애그리거트에 대한 비관락을 구함
2. 스레드2: b 애그리거트에 대한 비관락을 구함
3. 스레드1: b 애그리거트에 대한 비관락을 구하려고 시도
4. 스레드2: a 애그리거트에 대한 비관락을 구하려고 시도

- 데드락은 상대적으로 사용자 수가 많을 때 발생할 가능성이 높다. 이 때문에 잠금을 구할 때 최대 대기 시간을 지정해야 한다.
<br/>

jpa는 다음과 같은 힌트를 통해 최대 대기 시간을 조절할 수 있다.

```kotlin
hints.put("javax.persistence.lock.timeout", 2000)
val order = em.find(Order.class, orderNo, LockModeType.PERSSIMISTIC_WRITE, hints)
```

- 지정한 시간 이내에 잠금을 획득하지 못하면 익셉션을 발생시킨다. 주의할 점은 dbms에 따라 힌트가 적용되지 않을 수 있다.
<br/>

스프링 데이터 jpa는 `@QueryHints` 애노테이션을 사용해서 쿼리 힌트를 지정할 수 있다.

<img width="696" height="286" alt="스크린샷 2025-08-03 오후 12 40 04" src="https://github.com/user-attachments/assets/7f07d2d4-b432-490b-979b-60d2c256fd0e" />

> dbms에 따라 데드락에 빠진 커넥션을 처리하는 방식이 다르다. 쿼리별 대기 시간을 지정하거나, 커넥션 단위로만 대기 시간을 지정할 수 있는 dbms도 있다.

## 8.3 비선점 잠금(낙관락)
비관락으로 모든 트랜잭션 충돌 문제가 해결되지는 않는다.
<img width="489" height="417" alt="스크린샷 2025-08-03 오후 12 41 13" src="https://github.com/user-attachments/assets/41e0c9d6-381b-41a1-ab29-e55cf105ee1e" />

1. 운영자는 배송을 위해 주문 정보를 조회. 시스템은 정보를 제공
2. 고객이 배송지 변경을 위해 변경 폼을 요청. 시스템은 변경 폼을 제공
3. 고객이 새로운 배송지를 입력하고 폼을 전송하여 배송지를 변경
4. 운영자가 1번에서 조회한 주문 정보를 기준으로 배송지를 정하고 배송 상태 변경을 요청

여기서 배송 상태 변경 전에 배송지를 한 번 더 확인하지 않으면 운영자는 다른 배송지로 물건을 발송하게 된다. 고객은 배송지를 변경했음에도 엉뚱한 곳으로 주문한 물건을 받는 상황이 발생한다.

이 문제는 비관락으로 해결할 수 없다. 운영자의 주문 정보 조회 -> 배송 상태 변경 요청이 하나의 트랜잭션에서 이루어지는 작업이 아니기 때문이다.

- 낙관락은 비관락처럼 동시 접근을 막지 않음
- 대신 변경한 데이터를 실제 dbms에 반영하는 시점에 변경 가능 여부를 확인하는 방식
<br/>
이를 구현하려면 버전으로 사용할 숫자 타입 프로퍼티를 추가하고 다음과 같은 쿼리를 사용한다.

```sql
update agg set version = version + 1, col_a = ?, col_b = ?
where id = ? and version = 현재버전
```

이 쿼리는 현재 버전과 동일한 경우에만 데이터를 수정한다.
<img width="618" height="382" alt="스크린샷 2025-08-03 오후 12 46 36" src="https://github.com/user-attachments/assets/8e06d2cb-1986-4ed3-87e2-efd818c91b6a" />

- 스레드1, 2는 같은 버전의 애그리거트를 읽어와 수정
- 스레드1이 먼저 커밋했으므로 버전이 변경되고 스레드2의 수정은 실패
  - jpa는 `@Version` 애노테이션을 통해 이 기능을 지원한다.
 
- 쿼리의 결과로 수정된 행의 개수가 0이면 이미 데이터가 수정된 것. 이 시점에 스프링 데이터는 예외를 발생시킴

이제 낙관락을 문제되는 상황으로 확장해서 적용해 볼 수 있다. 
- 시스템은 사용자에게 수정 폼을 제공할 때 애그리거트 버전을 함께 제공
- 폼을 서버에 전송할 때 버전을 함께 전송
<br/>

이로써 트랜잭션 충돌 문제를 해소할 수 있다.

<img width="589" height="617" alt="스크린샷 2025-08-03 오후 1 06 52" src="https://github.com/user-attachments/assets/cc689221-bdf3-4c5d-92f2-64445fc61678" />

<img width="676" height="213" alt="스크린샷 2025-08-03 오후 1 07 26" src="https://github.com/user-attachments/assets/ae39844b-70cd-409d-8a1b-1ef2a921c13f" />

응용 서비스에 전달할 요청 데이터도 사용자가 전송한 버전 값을 포함해야 한다.
```kotlin
data class StartShippingRequest(
	val orderNumber: String,
	val version: Long,
)
```

응용 서비스는 버전 값을 이용 애그리거트의 버전과 일치하는지 확인하고, 기능을 수행한다.
```kotlin
class StartShippingService {
	@Transactional
	fun startShipping(request: StartShippingRequest) {
		val order = orderRepository.findById(OrderNo(request.orderNumber))
		checkOrder(order)
		if (!order.matchVersion(request.version)) {
			throw VersionConfliceException()
		}
		order.startShipping()
	}
}
```

표현 계층은 버전 충돌 예외가 발생하면 사용자가 알맞은 후속 처리를 할 수 있도록 알린다.
```kotlin
class OrderAdminController {
	@PostMapping
	fun startShipping(request: StartShippingRequest): String {
		try {
			startShippingService.startShipping(request)
			return "shippingStarted"
		} catch(e: Exception) {
			when(e) {
			    is OptimisticLockingFailureException, is VersionConflictException -> 
				    return "startShippingTxConflict"  
			    else -> throw e  
			}
		}
	}
}
```

이 코드에서 두 개의 예외를 처리하고 있다. 이 둘은 트래낵션 충돌이 발생한 시점을 명확히 구분해준다.
- 스프링의 `OptimisticLockingFailureException`
	- 동시에 애그리거트 수정을 시도했음을 의미

- 커스텀하게 정의한 `VersionConfliceException`
	- 이미 누군가가 애그리거트를 수정했음을 의미
<br/>

만약 버전 충돌을 명시적으로 구분할 필요가 없다면 응용 서비스에서 스프링 예외를 발생시키는 것도 고려할 수 있다.

```kotlin
class StartShippingService {
	@Transactional
	fun startShipping(request: StartShippingRequest) {
		val order = orderRepository.findById(OrderNo(request.orderNumber))
		checkOrder(order)
		if (!order.matchVersion(request.version)) {
			throw OptimisticLockingFailureException("version conflict")
		}
		order.startShipping()
	}
}
```

### 8.3.1 강제 버전 증가
루트 엔티티가 아닌 다른 엔티티의 값만 변경되는 경우 jpa는 버전 값을 증가시키지 않는다. 애그리거트의 상태가 변경되었으나 그것이 루트가 아니기 때문에 버전이 변경되지 않는다는 것은 **논리적으로 올바르지 않다.**

jpa는 이를 해결할 수 있도록 트랜잭션 종료 시점에 강제로 버전을 증가시키는 잠금 모드도 제공한다.
<img width="678" height="301" alt="스크린샷 2025-08-03 오후 1 21 08" src="https://github.com/user-attachments/assets/bd8bed23-c0b5-47ee-a534-5e86162599bc" />

## 8.4 오프라인 선점 잠금
엄격하게 데이터 충돌을 막고 싶다면 누군가 **수정 화면을 보고 있을 때 수정 화면 자체를 실행하지 못하게** 해야 한다. 이를 위한 것이 오프라인 선점 잠금(Offline Pessimistic Lock)이다.

오프라인 선점 잠금은 여러 트랜잭션에 걸쳐 동시 변경을 막는다. 첫번째 트랜잭션에서 잠금을 선점하고, 마지막 트랜잭션에서 잠금을 해제한다.
<img width="495" height="525" alt="스크린샷 2025-08-04 오후 8 26 31" src="https://github.com/user-attachments/assets/37949a50-59f4-477a-adc6-372e8d9ce41e" />

만일 위 그림에서 사용자A가 도중에 프로그램을 종료한다면? 
- 이런 상태를 방지하려면 잠금 유효 시간을 가져야 한다. 그럼 일정 시간 후에 사용자B는 잠금을 구할 수 있을 것이다.
<br/>

반대로 사용자A가 잠금 유효 시간을 넘도록 수정 페이지에 머문다면? 
- 수정 요청이 실패하게 될 것이다. 때문에 일정 주기로 유효 시간을 증가시키는 방식도 필요하다.

### 8.4.1 오프라인 선점 잠금을 위한 LockManager 인터페이스와 클래스
오프라인 선점 잠금은 다음과 같이 네가지 기능이 필요하다.
```kotlin
interface LockManager {
	// 선점 시도
	@Throws(LockException::class)
	fun tryLock(type: String, id: String): LockId
	// 잠금 확인
	fun checkLock(lockId: LockId)
	// 잠금 해제
	fun releaseLock(lockId: LockId)
	// 잠금 유효시간 연장
	fun extendLockExpiration(lockId: LockId, inc: Long)
}

// 저자는 각 잠금마다 고유한 식별자를 갖도록 구현했다.
data class LockId(
	val value: String,
)
```

`tryLock()`은 잠금에 성공하면 LockId를 리턴한다. 컨트롤러는 수정 폼에 데이터를 전송할 때 LockId를 전송하여 추후 잠금을 해제하거나 유효시간을 연장하는데 사용한다.

```kotlin
@PostMapping
fun editForm(id: Long, model: ModelMap): String {
	val result = dataService.getDataWithLock(id)
	model.addAttribute("lockId", result.lockId)
	return "editForm"
}
```

잠금 해제 코드는 다음과 같을 것이다.
```kotlin
fun edit(request: EditRequest, lockId: LockId) {
	lockManager.checkLock(lockId)
	...
	lockManager.releaseLock(lockId)
}
```

잠금 유효 시간이 지났다면 다른 사용자가 잠금을 선점했을 것이고, 잠금을 선점하지 않은 사용자가 기능을 실행했다면 기능을 막는 등의 행위를 `checkLock()`으로 적절히 구현해야 한다.

### 8.4.2 db를 이용한 LockManager 구현
db를 이용한다면 다음과 같이 테이블이 필요할 것이다.
```sql
create table locks (
	`type` varchar(255),
	id varchar(255),
	lock_id varchar(255),
	expiration_tiem datetime,
	primary key (`type`, id)
	unique(lock_id)
);
```

JdbcTemplate을 이용해 구현한다면 다음과 같다.
<img width="677" height="684" alt="스크린샷 2025-08-04 오후 8 50 37" src="https://github.com/user-attachments/assets/6b2b2e1b-e6ba-4fa7-ba12-754c54fba542" />
<img width="674" height="645" alt="스크린샷 2025-08-04 오후 8 50 48" src="https://github.com/user-attachments/assets/4361a427-15fe-4aea-8db4-d8b26205fc13" />
<img width="691" height="627" alt="스크린샷 2025-08-04 오후 8 51 45" src="https://github.com/user-attachments/assets/b848a2a8-5706-49f4-b1a5-db2370795f26" />
<img width="675" height="355" alt="스크린샷 2025-08-04 오후 8 51 56" src="https://github.com/user-attachments/assets/67ca279c-8bb3-4a35-932d-1ff3f64c93c5" />

