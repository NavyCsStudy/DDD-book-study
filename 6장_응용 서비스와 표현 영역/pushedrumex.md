# 6. 응용 서비스와 표현 영역

## 6.1 표현 영역과 응용 영역
- 표현 영역과 응용 영역 : 사용자와 도메인을 연결해주는 매개체
- 표현 영역 : 사용자의 요청을 해석
  - 사용자의 요청을 URL, 파라미터, 쿠키, 헤더 등을 이용해서 요청을 판별하여 해당 기능을 하는 응용 서비스에게 전달
- 응용 영역 : 사용자가 원하는 기능을 제공
  - 기능을 실행하는 데 필요한 입력 값을 메서드 인자로 받고 실행 결과를 리턴
- 표현 영역은 응용 서비스가 요구하는 형식으로 사용자 요청을 변환
  - ex) 파라미터 값 -> dto 객체
- 응용 서비스를 실행한 뒤 표현 영역은 실행 결과를 사용자에게 알맞은 형식으로 응답

## 6.2 응용 서비스의 역할
- 응용 서비스는 사용자가 요청한 기능을 실행
- 응용 서비스는 도메인 영역과 표현 영역을 연결
- 사용자의 요청을 처리하기 위해 리포지터리에서 도메인 객체를 가져와서 사용
- 응용 서비스가 복잡하다면 응용 서비스에서 도메인 로직을 구현하고 있을 가능성이 있음
  - 응용 서비스에서 도메인 로직을 구현하면 코드 품질에 안좋은 영향을 줄 수 있음
- 응용 서비스는 트랜잭션도 담당

### 6.2.1 도메인 로직 넣지 않기
- ex) 회원의 비밀 번호 변경 -> `기존의 암호로 변경하는 것이 불가능하다`는 도메인의 핵심 로직 존재
  - 응용 서비스에 해당 도메인 로직을 구현한다면?
```java
public class ChangePasswordService {
    public void changePassword(String memberld, String oldPw, String newPw) {
        Member member = memberRepository.findByld(memberld);
        checkMemberExists(member);
        if (IpasswordEncoder.matches(oldPW, member.getPassword())) {
            throw new BadPasswordException();
        }
        member.setPassword(newPw);
    }
}
```
  - 문제1 : 코드의 응집성 저하
    - 도메인 데이터와 그 데이터를 조작하는 로직이 서로 다른 영역에 위치
  - 문제2 : 여러 응용 서비스에서 동일한 도메인 로직을 구현할 가능성이 높아짐
    - 도메인 영역에 구현했다면 여러 응용 서비스에서 동일한 로직을 구현할 필요가 없음

## 6.3 응용 서비스의 구현

### 6.3.1 응용 서비스의 크기
- 방법 1. 한 응용 서비스 클래스에 회원 도메인의 모든 기능 구현
  - 동일 로직에 대한 코드 중복을 제거할 수 있음
  - 한 서비스 클래스의 크기가 커짐 -> 관련 없는 코드가 뒤섞여 코드를 이해하는 데 방해가 됨 ex) 의존 객체
  - 분리하는게 좋은 것을 알면서도 기존에 존재하는 클래스에 억지로 끼워넣게 됨
- 방법 2. 구분되는 기능별로 응용 서비스 클래스를 따로 구현
  - 한 응용 서비스 클래스에서 1~3개의 기능을 구현
  - 클래스 개수를 많아지지만 코드 품질이 유지됨
  - 여러 클래스에 중복해서 동일한 코드를 구현할 가능성이 있음 -> 별도의 클래스를 구현하여 코드 중복을 방지하는 방법

### 6.3.2 응용 서비스의 인터페이스와 클래스
- 인터페이스가 과연 필요할까?
  - 응용 서비스는 런타임에 교체하는 경우가 거의 없고 한 응용 서비스의 구현 클래스가 두 개인 경우가 드뭄
  - 인터페이스와 클래스를 따로 구현하면 소스 파일만 많아지고 구현 클래스에 대한 간접참조만 늘어나 전체 구조가 복잡해짐

### 6.3.3 메서드 파라미터와 값 리턴
- 응용 서비스 메서드는 기능을 실행하는 데 필요한 값을 파라미터로 전달받아야함
  - 요청 파라미터가 두 개 이상 존재하면 데이터 전달을 위핸 별도의 클래스를 사용하는 것이 편함
- 응용 서비스는 메서드의 결과로 표현 영역에서 필요한 데이터를 리턴
  - 애그리거트 자체를 리턴하면 도메인 로직을 표현 영역에서도 사용할 수 있게 돼서 응집도를 낮춤 -> 필요한 데이터만 리턴하는 것이 응집도를 높이는 확실한 방법

### 6.3.4 표현 영역에 의존하지 않기
- 응용 서비스에서 표현 영역과 관련된 타입을 사용하면 안됨 ex) HttpSession
  - 표현 영역에 대한 의존이 높아지면 응용 서비스만 단독으로 테스트하기가 어려워짐
  - 표현 영역의 구현이 변경되면 응용 서비스의 구현도 함께 변경해야 함
  - 응용 서비스에서 표현 영역의 역할까지 대신하는 문제도 발생

### 6.3.5 트랜잭션 처리
- 프레임 워크가 제공하는 트랜잭션 기능을 적극 사용하는 것이 좋음 ex) @Transactional

### 6.4 표현 영역
- 표현 영역의 책임 
1. 사용자에게 시스템을 사용할 수 있는 흐름(화면) 제공/제어
2. 알맞은 응용 서비스에 요청을 전달하고 사용자에게 제공 
3. 사용자의 세션을 관리

### 6.5 값 검증
- 값 검증은 표현 영역과 응용 서비스 두 곳에서 모두 할 수 있음
- 원칙적으로 모든 값에 대한 검증은 응용 서비스에서 처리
  - 컨트롤러는 폼 에러 메시지를 보여주기 위해 예외 클래스 별로 catch 해서 처리해야하므로 매우 번잡함
```java
public String join(JoinRequest joinRequest, Errors errors){
    try {
        joinService.join(joinRequest);
        return successview;
    }catch(EmptyPropertyException ex) {
        // 표현 영역은 잘못 입력한 값이 존재하면 이를 사용자에게 알려주고
        // 폼을 다시 입력할 수 있도록 하기 위해 관련 기능 사용
        errors.rejectValue(ex.getPropertyName(), "empty");
        return formView;
    } catch(InvalidPropertyException ex) {
        errors.rejectVaUie(ex.getPropertyName(), "invalid");
        return formView;
    } catch(DuplicateIdException ex) {
        errors.rejectValue(ex.getPropertyName(), "duplicate");
        return formView;
    }
}
```
  - 폼의 여러 항목에서 검증 예외가 발생하는 경우, 에러 코드를 모아 하나의 예외를 발생시키는 방법이 존재
```java
@Transactional
public OrderNo placeOrder(OrderRequest orderRequest) {
    List<ValidationError> errors = new ArrayList<>();
    if (orderRequest == null) {
        errors.add (ValidationError.of ("empty"));
    } else {
        if (orderRequest.getOrdererMemberId() == null)
            errors.add(ValidationError.of("ordererMemberId", "empty"));
        if (orderRequest.getOrderProducts() == null)
            errors.add(ValidationError.of("orderProducts", "empty"));
        if (orderRequest.getOrderProducts().isEmpty())
            errors.add(ValidationError.of("orderProducts", "empty"));
    }
    // 응용 서비스가 입력 오류를 하나의 익셉션으로 모아서 발생
    if (errors.isEmpty()) throw new ValidationErrorException(errors);
}
```
  - 표현 영역은 익셉션에서 예외 목록을 가져와 표현 영역에서 사용할 형태로 변환 처리
- 표현 영역에서 필수 값을 검증하는 방법도 존재
- 스프링에서 제공하는 Validator 을 사용하여 검증
  - 표현 영역에서는 필수 값, 값 형식, 범위 등을 검증
  - 응용 서비스에서는 데이터의 존재 유무 같은 논리적인 오류를 검증
- 요즘은 가능하면 응용 서비스에서 필수 값 검증과 논리적인 검증을 모두 하는 편
  - 작성할 코드가 늘어나지만 응용 서비스의 완성도가 높아짐

### 6.6 권한 검사
- 무조건 보안 프레임워크를 사용하는 것보다 권한 검사 기능을 구현하는것이 유지보수에 유리할 수도 있음
- 표현영역, 응용 서비스, 도메인 영억에서 권한 검사를 수행할 수 있음
- 표현 영역 -> 서블릿 필터를 통해 URL 별로 인증/권한 검사
  - URL만으로 접근 제어를 할 수 없는 경우 응용 서비스의 매서드 단위로 검사 수행
  - 스프링 시큐리티는 AOP 를 통애 애너테이셔으로 메서드별 권한 검사를 하는 기능을 제공
  - 개별 도메인 객체 단위로 검사를 해야할 경우 직접 권한 검사 로직을 구현
- 보안 프레임워크를 확장해서 개별 도메인 객체 수준의 권한 검사 기능을 통합할 수도 있음
  - 하지만, 프레임워크에 대한 이해도가 높지 않다면, 도메인에 맞는 권한 검사를 직접 구현하는것이 코드 유지보수에 유리

### 6.7 조회 전용 기능과 응용 서비스
- 조회 전용 기능을 사용하면 서비스 코드가 단순히 조회 전용 기능을 호출하는 형태
- 트랜잭션도 필요로 하지 않음
- 굳이 서비스를 만들 필요 없기 표현 영역에서 바로 조회 전용 기능을 사용하는 방법도 존재
```java
@RequestMapping("/myorders")
public String list(ModelMap model) {
    String ordererld = Securitycontext.getAuthentication().getId();
    List<OrderView> orders = orderViewDao.selectByOrderer(ordererld);
    model.addAttribute("orders", orders);
    ...
}
```