## 6.1 표현 영역과 응용 영역

도메인이 제 기능을 하려면 사용자와 도메인을 연결해 주는 매개체가 필요하다. 응용, 표현 영역이 이 역할을 해준다.

<img width="590" height="173" alt="스크린샷 2025-07-22 오후 8 38 06" src="https://github.com/user-attachments/assets/7925e578-a396-4d21-adb1-2df0313fdbd6" />

#### 표현 영역
- 사용자의 요청을 해석한다.
- url, 요청 파라미터, 쿠키 등을 이용해서 사용자가 실행하고 싶은 기능을 판별하고 그 기능을 제공하는 응용 서비스를 실행한다.
#### 응용 영역
- 사용자가 원하는 기능을 제공하는 주체는 응용 서비스에 위치한다.
- 응용 서비스는 기능 실행에 필요한 입력 값을 메서드 인자로 받고 실행 결과를 리턴한다.
<br/>

- 응용 서비스가 요구하는 파라미터와 사용자로부터 건네받은 데이터는 형식이 일치하지 않음
- 따라서 표현 영역은 응용 서비스에서 필요한 형식으로 사용자 요청을 변환
```kotlin
@PostMapping("/members/join")
fun join(request: HttpServletRequest): ModelAndView {
	val email = request.getParameter("email")
	val password = request.getParameter("password")
	// 사용자 요청을 응용 서비스에 맞게 변환
	val joinRequest = JoinRequest(email, password)
	// 변환한 객체(데이터)를 이용해서 응용 서비스 실행
	joinService.join(joinRequest)
	...
}
```

- 표현 영역은 응용 서비스의 실행 결과를 사용자에게 알맞은 형식으로 응답
- 사용자와의 상호작용은 표현 영역에서 처리하기 때문에 응용 서비스는 표현 영역에 의존하지 않음
- 응용 서비스는 단지 기능 실행에 필요한 값을 입력 받고 결과를 리턴

## 6.2 응용 서비스의 역할
- 응용 서비스는 클라이언트가 요청한 기능을 실행
- 사용자의 요청을 처리하기 위해 리포지터리에서 도메인 객체를 가져와 사용
- 클라이언트의 입장에서 응용 서비스는 도메인 영역과 표현 영역을 연결해주는 창구 역할
- 응용 서비스는 주로 도메인 객체 간의 흐름을 제어하기 때문에 다음과 같이 단순한 형태를 가짐
```kotlin
fun doSomeFunc(request: SomeRequest): Result {
	// 1. 리포지터리에서 애그리거트를 구한다.
	val aggregate = someRepository.findById(request.id) ?: throw NotFoundException()
	
	// 2. 애그리거트의 도메인 기능을 실행한다.
	aggregate.doFunc(request.value)
	
	//3. 결과를 리턴한다.
	return createSuccessRequest(aggregate)
}
```

- 새로운 애그리거트를 생성하는 응용 서비스도 간단함
```kotlin
fun doSomCreation(request: CreateRequest): Result {
	// 1. 데이터 중복 등 데이터가 유효한지 검사한다.
	valudate(request)
	
	// 2. 애그리거트를 생성한다.
	val newAggregate = createSome(request)
	
	// 3. 리포지터리에 애그리거트를 저장한다.
	someRepository.save(newAggregate)

	// 4. 결과를 리턴한다.
	return createSuccessResult(newAggregate)
}
```

- 응용 서비스가 복잡하다면 도메인 로직의 일부를 응용 서비스에 구현하고 있을 가능성이 높음
- 이런 경우 코드 중복, 로직 분산 등 코드 품질에 안 좋은 영향을 줄 수 있음

### 6.2.1 도메인 로직 넣지 않기
- 암호 변경을 예로 들어보자.
```kotlin
class ChangePasswordService {

	fun changePassword(memberId: String, oldPassword: String, newPassword: String) {
		val member = memberRepository.findById(memberId) ?: throw NotFoundException()
		member.changePassword(oldPassword, newPassword)
	}
}

class Member {
	fun changePassword(oldPassword: String, newPassword: String) {
		if (!matchPassword(oldPassword)) {
			throw BadPasswordException()
		}
		this.password = newPassword
	}

	fun matchPassword(password: String): Boolean {
		return passwordEncoder.matches(password)
	}
}
```

- 암호를 올바르게 입력했는지 확인하는 것은 도메인의 핵심 로직
- 이를 다음처럼 응용 서비스에서 구현해서는 안된다.
```kotlin
class ChangePasswordService {

	fun changePassword(memberId: String, oldPassword: String, newPassword: String) {
		val member = memberRepository.findById(memberId) ?: throw NotFoundException()
		if (!passwordEncoder.matches(oldPassword, newPassword)) {
			throw BadPasswordException()
		}
		member.password = newPassowrd
	}
}
```

#### 문제1. 코드의 응집성이 떨어진다.
- 도메인 데이터와 이를 조작하는 로직이 도메인, 응용 서로 다른 영역에 위치하면 도메인 로직을 파악하기 위해 여러 영역을 분석해야함을 의미

#### 문제2. 코드 중복이 발생할 수 있다.
```kotlin
class DeactivationService {

	fun deactivate(memberId: String, password: String) {
		val member = this.memberRepository.findByIdOrNull(memberId) ?: throw NotFoundException()
		if (!passwordEncoder.matches(oldPassword, newPassword)) {
			throw BadPasswordException()
		}
		member.deactivate()
	}
}
```

- 응용 서비스에 별도 서비스를 구현할 수도 있지만, 애초에 도메인 영역에 암호 확인 기능을 구현했으면 중복을 발생하지 않음
```kotlin
class DeactivationService {

	fun deactivate(memberId: String, password: String) {
		val member = this.memberRepository.findByIdOrNull(memberId) ?: throw NotFoundException()
		if (!member.matchPassword(password)) {
			throw BadPasswordException()
		}
		member.deactivate()
	}
}
```

- 이러한 두가지 문제는 코드 변경을 어렵게 만듬
- 소프트웨어의 중요한 경쟁 요소 중 하나는 변경 용이성. 이를 유념하자.

## 6.3 응용 서비스의 구현
- 응용 서비스는 표현 영역과 도메인 영역을 연결하는 매개체 역할을 한다.
- 응용 서비스 자체는 복잡한 로직을 수행하지 않기에 구현이 어렵지 않다.

### 6.3.1 응용 서비스의 크기
응용 서비스는 보통 다음 두 가지 방법으로 구현
- 한 응용 서비스 클래스에 회원 도메인의 모든 기능 구현하기
- 구분되는 기능별로 응용 서비스 클래스를 따로 구현하기
```kotlin
class MemberService {
	fun join(request: JoinRequest) {}
	fun chnagePassword(memberId: String, currentPassowrd: String, newPassword: String) {}
	fun initializePassword(memberId: String) {}
	fun leave(memberId: String, currentPassword: String) {}
}
```

- 첫번째 방법은 하나의 도메인과 관련된 기능을 구현한 코드가 한 클래스에 위치하므로 각 기능에서 동일 로직에 대한 코드 중복을 제거할 수 있음
```kotlin
class MemberService { 
	fun changePassword() {
		val member = findMember(memberId)
	}

	// 각 기능의 동일 로젝에 대한 구현 코드 중복 제거
	private fun findMember(memberId:String) {
		return memberRepository.findByIdOrNull(memberId) ?: throw NotFoundException()
	}
}
```

- 서비스 클래스의 크기가 커진다는 것은 단점
- 코드 크기가 커지면 연관성이 적은 코드가 한 클래스에 함께할 가능성이 높아짐
- 이는 코드를 이해하는데 방해
<br/>
- 두번째 방법은 구분되는 기능별로 하나 혹은 2~3개의 기능을 하나의 응용 서비스 클래스에 구현
```kotlin
class ChangePasswordService {
	fun changePassword() {}
}
```

- 클래스 개수는 많아지지만 코드 품질을 일정하게 유지할 수 있음
- 필요한 의존 관계만 포함하므로 다른 기능을 구현한 코드에 영향을 받지 않음
- 하지만 여러 응용 서비스들간에 코드 중복이 발생할 수 있음. 이 경우에는 별도 클래스에 로직을 구현해서 코드 중복을 제거할 수 있음
```kotlin
class MemberServiceHelper {
	companion object {
		fun findMember(repository: MemberRepository, memberId: String) {
			return repository.findByIdOrNull(memberId) ?: throw NotFoundException()
		}
	}
}
```

### 6.3.2 응용 서비스의 인터페이스와 클래스
- 응용 서비스 구현시 인터페이스의 필요성은 논쟁거리
- 구현 클래스가 여러개인 경우에는 인터페이스가 필요. 하지만 응용 서비스의 구현 클래스가 여러개인 경우는 드뭄
- 인터페이스가 명확하게 필요해지기 전에 미리 작성할 필요까지는 없다.
- 만일 tdd를 한다면 표현 영역 테스트를 위해 응용 서비스 인터페이스를 사용할 수도 있음. 하지만 이도 mockito 같은 도구를 이용하면 충분히 대체 가능
- 응용 서비스의 필요성은 사람마다 생각이 다른 부분이니 적당히 고민해보자.

### 6.3.3 메서드 파라미터와 값 리턴
- 응용 서비스는 도메인을 이용해 사용자가 요구한 기능을 실항하는 데 필요한 값을 파라미터로 전달받아야 함
- 개별 파라미터를 전달받을 수도 있고 별도의 데이터 클래스를 만들어 잔달받을 수도 있음
- 요구사항에 따라서는 사용자가 요청한 작업을 처리하고 결과를 곧바로 보여줘야 할 수도 있음
  - 이때는 애그리거트 루트 엔티티의 식별자를 리턴하거나, 애그리거트 그 자체를 리턴할 수 있음
  - 표현 영역에서는 애그리거트 객체에서 식별자를 구해 사용자에게 보여줄 응답화면을 생성하면 됨
- 애그리거트 자체를 리턴하는 것은 도메인 로직 실행이 표현, 응용 영역에 분산되어 코드 응집도를 낮추는 원인이 됨
- 표현 영역에서 필요한 데이터만 리턴하는 것이 응집도를 높일 수 있는 확실한 방법

### 6.3.4 표현 영역에 의존하지 않기
- 응용 서비스에는 표현 영역에서 해당하는 HttpServleteRequest 같은 것을 전달해서는 안 됨
- 이는 응용 서비스 단독 테스트를 어렵게 만듬
- 가장 심각한 것은 표현 영역의 상태를 추적하기 어려워진다는 점. 이는 코드 유지 보수 비용을 증가시키는 원인
```kotlin
class AuthenticationService {
	fun authenticate(request: HttpServletRequest) {
		if(checkIdPasswordMatching(id, password)) {
			val session = request.getSession()
			session.setAttribute("auth", Authentication(id))
		}
	}
}
```

### 6.3.5 트랜잭션 처리
- 트랜잭션을 관리하는 것은 응용 서비스의 중요한 역할
- 변경사항이 제대로 db에 반영되지 않으면 고객은 로그인하지 못하거나, 물건 배송을 받지 못할 것
- `@Transactional`과 같은 프레임워크의 기능을 적극 활용하면 트랜잭션 처리 코드를 간결하게 유지할 수 있음

## 6.4 표현 영역
- 표현 영역의 책임은 다음과 같다.
  - 사용자가 시스템을 사용할 수 있는 흐름(화면)을 제공하고 제어한다.
  - 사용자의 요청을 알맞은 응용 서비스에 전달하고 결과를 사용자에게 제공한다.
  - 사용자의 세션을 관리한다.

#### 책임1. 사용자가 시스템을 사용할 수 있도록 알맞은 흐름을 제공한다.
<img width="602" height="388" alt="스크린샷 2025-07-29 오후 9 02 37" src="https://github.com/user-attachments/assets/d7ee50d5-2466-4397-811a-10ccd5fbf80a" />

- 표현 영역은 사용자에게 적절한 폼을 제공
- 표현 영역은 응용 서비스를 이용해 사용자의 요청을 처리하고 결과를 응답으로 전송

#### 책임2. 사용자의 요청에 맞게 응용 서비스에 기능 실행을 요청한다.
- 화면을 보여주는데 필요한 데이터를 읽거나 도메인 상태를 변경해야 할 때 응용 서비스를 사용
- 표현 영역은 사용자의 요청 데이터를 응용 서비스가 요구하는 형식으로 변환하고 응용 서비스의 결과를 사용자에게 응답할 수 있는 형식으로 변환
```kotlin
@PostMapping
fun changePassword(request: ChangePasswordRequest): String {
	changePasswordService.changePassword(
		request.toCommand(
			memberId = SecurityContext.authentication.id
		)
	)
}
```

- 응용 서비스 결과를 사용자에게 알맞은 형식으로 제공하는 것 또한 표현 영역의 책임

#### 책임3. 사용자의 연결 상태인 세션을 관리한다.

## 6.5 값 검증
- 값 검증은 표현, 응용 서비스 두 곳에서 모두 수행할 수 있음
- 원칙적으로 모든 값에 대한 검증은 응용 서비스에서 처리
```kotlin
class JoinService {
	@Transactional
	fun join(request: JoinRequest) {
		checkEmpty(request.id, "id")
		checkEmpty(request.name, "id")
		checkEmpty(request.password, "id")
		if(request.password == request.confirmPassword) {
			throw InvalidPropertyException("confirmPassword")
		}
		checkDuplicateId(request.id)
	}
}
```

- 표현 영역은 잘못된 값이 존재하면 사용자에게 알려주고 값을 다시 입력받아야 함
  - BindingResult를 사용할 수 있는데, 좀 번잡함
- 응용 서비스에서 검증하고 예외를 던지는 것은 사용자에게 좋지 못한 경험을 제공할 수 있음
  - 첫 예외가 발생하면 나머지 항목에 대해 검사하지 않게 되어 사용자는 자신이 입력한 값이 얼마나 잘못되었는지 알 수 없음
- 이를 해결하려면 응용 서비스에서 에러 코드를 모아 하나의 익셉션을 발생시키는 방법이 있음
```kotlin
@Transactional
fun placeOrder(request: OrderRequest?): OrderNo {
	val errors: MutableList<ValidationError> = Mutablelistof()
	reuqest ?: errors.add(ValidationError.of("empty"))

	reuqest.ordererMemberId ?: errors.add(ValidationError.of("ordereMemberId", "empty"))
	reuqest.orderProducts ?: errors.add(ValidationError.of("orderProducts", "empty"))

	if (!errors.isEmpty()) throw ValidationErrorException(errors)
}
```

- 표현 영역은 다음과 같이 사용자에게 반환할 형태로 변환
```kotlin
@PostMapping
fun order(): String {
	try {
		...
	} catch (e: ValidationErrorException) {
		e.errors.forEach {
			if (it.hasName()) {
				bindingResult.rejectValid(it.name, it.code)
			} else {
				bindingResult.reject(it.code)
			}
		}
		return "order/confirm"
	}
}
```

- 표현 영역에서 필수값을 검증할 수도 있고, 스프링에서 제공하는 Validator 인터페이스, bean validation을 이용할 수도 있음
- 응용 서비스를 사용하는 표현 영역 코드가 한 곳이면 다음과 같이 역할을 나누어 검증할 수도 있다.
  - 표현 영역: 필수 값, 값의 형식, 범위 등을 검증한다.
  - 응용 서비스: 데이터의 존재 유무와 같은 논리적 오류를 검증한다.
- 응용 서비스에서 모든 값을 검증한면 프레임워크가 제공하는 편리함을 사용할 수는 없지만 완성도가 높아지는 이점이 있음
- 이것은 팀과의 논의를 통해 결정해야 할 것

## 6.6 권한 검사
- 권한 검사 자체는 복잡한 개념이 아님
- 그러나 시스템에 따라 단순한 인증 여부만 검사하거나, 관리자에 따라 사용할 수 있는 기능이 달라지는 경우도 있음
- 스프링 시큐리티는 유연한 기능을 제공하지만 그만큼 복잡. 프레임워크에 대한 이해가 부족하다면 개발할 시스템에 맞는 권한 검사 기능을 구현하는 것이 유지 보수에 유리할 수 있음
- 프레임워크와 별개도 보통 다음 세곳에서 권한 검사를 수행할 수 있다.
  - 표현 영역
  - 응용 서비스
  - 도메인

#### 표현 영역
- 기본적으로 인증된 사용자인지 검사. 특정 url은 인증된 사용자만 접근할 수 있어야함.
  - url을 처리하는 컨트롤러에 웹 요청을 전달하기 전에 인증 여부를 검사해서 인증된 사용자의 웹 요청만 컨트롤러에 전달
  - 인증된 사용자가 아닐 경우 로그인 화면으로 리다이렉트
- 이러한 접근 제어를 하기 좋은 위치가 서블릿 필터
<img width="607" height="539" alt="스크린샷 2025-07-29 오후 9 34 31" src="https://github.com/user-attachments/assets/038d3578-85d5-4d60-a235-f38f4c07501c" />

- 인증 여부와 더불어 url 별 권한 검사도 할 수 있음
- 스프링 시큐리티는 필터를 이용해 인증 정보를 생성하고 웹 접근을 제어

#### 응용 서비스
- url 만으로 접근 제어를 할 수 없는 경우 응용 서비스의 메서드 단위로 권한 검사를 수행해야 함
```kotlin
class BlockMemberService {

	@PreAuthorize("hasRole('ADMIN')")
	fun block(memberId: String) {
		...
	}
}
```

- 개별 도메인 단위로 권한 검사를 해야하는 경우는 구현이 복잡해짐
- 게시글 삭제는 본인 또는 관리자만 할 수 있다면 우선 게시글 애그리거트를 로딩해야 함. 즉, 메서드 수준에서는 권한 검사를 할 수 없기 때문에 직접 권한 검사 로직을 구현해야 함
```kotlin
class DeleteArticleService {

	fun delete(userId: String, articleId: Long) {
		val article = articleRepository.findById(articleId)
		checkArticleExistence(article)
		permissionService.checkDeletePermission(userId, article)
		article.markDeleted()
	}
}
```

- 스프링 시큐리티를 확장해서 개별 도메인 객체 수준의 권한 검사 기능을 프레임워크에 통합할 수도 있으나 높은 이해가 필요함
- 원하는 수준의 이해가 없다면 직접 권한 검사 기능을 구현하는 것이 코드 유지 보수에 유리

## 6.7 조회 전용 기능과 응용 서비스
- 조회 전용 모델을 만든다면 응용 서비스에서는 단순히 조회 전용 기능을 호출하는 형태로 끝날 수 있음
```kotlin
class OrderListService {
	fun getOrderList(ordererId: String): List<OrderView> {
		return orderViewDao.selectByOrderer(ordererId)
	}
}
```

- 이 경우 표현 영역에서 바로 조회 전용 기능을 사용해도 문제 없음
```kotlin
class OrderController {
	@GetMapping
	fun list() {
		val orders = orderViewDao.selectByOrderer(ordererId)
	}
}
```

- 이상하게 느껴질 수 있으나 사용자 요청 기능을 실행하는 데 별다른 기여를 하지 못한다면 굳이 서비스를 만들 이유도 없음
<img width="667" height="275" alt="스크린샷 2025-07-29 오후 9 45 33" src="https://github.com/user-attachments/assets/b54930b8-347a-479a-a91e-005048d4b85e" />
