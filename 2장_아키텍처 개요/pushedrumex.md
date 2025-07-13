# 2. 아키텍처 개요
## 2.1 네 개의 영역
- 표현 영역 (UI)
  - 사용자의 요청을 받아 응용 서비스에 전달
  - 응용 영역의 처리 결과를 사용자가 이해할 수 있는 형식으로 응답
  - 표현 영역의 사용자 : 웹 브라우저를 사용하는 사람 or REST API 를 호출하는 외부 시스템
- 응용 영역 (서비스)
  - 도메인 영역의 도메인 모델을 사용하여 시스템이 사용자에게 제공해야 할 기능을 구현
```java
public class CancelOrderService {
  @Transactional
  public void cancelOrder(String orderld) {
      Order order = findOrderByld(orderld);
      if (order == null) throw new OrderNotFoundException(orderld);
      order.cancel();
  }
}
```
- 도메인 영역
  - 도메인 모델을 구현 ex) Order, OrderLine
  - 도메인의 핵심 로직을 구현
- 인프라스트럭처 영역
  - 구현 기술을 다루는 영역
  - 데이터 연동 처리 ex) RDBMS 연동, 메시징 큐에 메시지 전달
  - 외부 서비스 연동 ex) SMTP 메일 발송, REST API 호출
  - 표현, 응용, 도메인 영역은 인프라스트럭처 영역의 기능을 사용하여 필요한 기능을 개발

## 2.2 계층 구조 아키텍처
- 기본 아키텍처 : (상위) 표현 -> 응용 -> 도메인 -> 인프라스트럭처 (하위)
  - 상위 계층에서 하위 계층으로 의존
  - 표현, 응용, 도메인 계층이 상세 구현 기술을 다루는 인프라스트럭처 계층에 종속
```java
public class CalculateDiscountService {
  private DroolsRuleEngine ruleEngine;
  public CalculateDiscountServiceO {
    ruleEngine = new DroolsRuleEngine();
  }
  public Money calculateDiscount(l_ist<Orderl_ine> orderLines, String c니stomerld) {
      Customer customer = findCustomer(ciistomerld);
      MutableMoney money = new MutableMoney(0);
      List<?> facts = Arrays.asList(customer, money);
      facts.addAll(orderLines);
      ruleEngine.evalute("discountCalculation", facts);
  }
}
```
  - 문제점 1) RuleEngine에 종속하고 있는 CaculateDiscountService만 테스트하기 어려움
  - 문제점 2) 구현 방식을 변경하기 어렵다. 즉, RuleEngine의 구현 방식이 변경되면 Service 코드도 변경해야 함

## 2.3 DIP
- 용어
  - 고수준 모듈 : 의미 있는 단일 기능을 제공하는 모듈 ex) '가격 할인 계산'이라는 기능을 구현하는 CaculateDiscountService
  - 저수준 모듈 : 하위 기능을 구현한 모듈 ex) '가격 할인 룰'을 실행하는 RuleEnigine
  - 고수준 모듈은 저수준 모듈을 사용

-> 고수준 모듈이 저수준 모듈을 사용하면 구현 변경과 테스트의 여렵다는 문제가 발생

- 해결 방법 : DIP(Dependency Inversion Principle)
  - 추상화한 인터페이스를 통해 저수준 모듈이 고수준 모듈에 의존하도록 의존성 역전
```java
public interface RuleDiscounter {
  Money applyRules(C니stomer customer, List<OrderLine> orderLines);
}
```
```java
public class CalculateDiscountService {
  private RuleDiscounter ruleDiscounter;
  public CalculateDisco니ntService(RuleDiscounter ruleDiscounter) {
    this.ruleDiscounter = ruleDiscounter;
  }
  
  public Money calculateDiscount(List<OrderLine> orderLines, String customerld) {
    C니stomer customer = findCustomer(customerld);
    return ruleDiscounter.applyRules(customer, orderLines);
  }
}
```
  - 룰 적용을 구현한 클래스는 RuleDiscounter 를 상속받아 구현
  - 기존 구조) CalculateDiscountService -> DroolsRuleDiscounter
  - DIP 구조) CalculateDiscountService -> RuleDiscounter <- DroolsRuleDiscounter
    - 이 때, '룰을 이용한 할인 금액 계산'은 고수준 모듈의 개념이므로 RuleDiscounter 인터페이스는 고수준 모듈에 속함
    - 저수준 모듈인 DroolsRuleDiscounter 이 고수준 모듈인 RuleDiscounter 에 의존하므로 의존성이 역전됨
  - 이제 CalulateDiscountService 를 수정하지 않고 룰 구현 객체를 변경할 수 있음
  - stubRule 같은 대역 객체를 생성하여 주입하면 CalculateDiscountService 만 테스트 가능
  - DIP의 핵심 : 고수준 모듈이 저수준 모듈에 의존하지 않도록 하기 위함
  - 하위 기능을 추상화한 인터페이스는 고수준 모듈 관점에서 도출
  - DIP를 적용하면 응용영역과 도메인 영역에 영향을 최소화하여 구현체를 변경하거나 추가 할 수 있게 됨

## 2.4 도메인 영역의 주요 구성요소
- 엔티티 ex) Order, Member
  - 고유 식별자를 갖는 객체
  - 자신의 라이프 사이클을 가짐
  - 도메인의 고유한 개념을 표현
  - 도메인 모델의 데이터와 기능 가짐
- 밸류 ex) Address, Money
  - 식별자를 가지지 않음
  - 개념적으로 하나의 값을 표현할 때 사용
- 애그리거트 ex) 주문 애그리거트 : Order, OrderLine 의 묶음
  - 연관된 엔티티와 밸류 객체를 하나로 묶은 것
- 리포지터리
  - 도메인 모델의 영속성을 처리
- 도메인 서비스
  - 특정 엔티티에 속하지 않은 도메인 로직을 제공
- DB의 엔티티 vs 도메인 모델의 엔티티
  - 도메인 모델의 엔티티는 데이터와 함께 도메인 기능을 함께 제공
```java
public class Order {
    // 주문 도메인 모델의 데이터
    private OrderNo number;
    private Orderer orderer;
    private Shippinginfo shippinginfo;

    // 도메인 모델 엔티티는 도메인 기능도 함께 제공
    public void changeShippingInfo(ShippingInfo newShippinglnfo) {
    }
}
```
  - 도메인 모델의 엔티티는 두개 이상의 데이터를 밸류 타입으로 표현할 수 있음
```java
public class Orderer {
  private String name;
  private String email;
}
```
- 애그리거트
  - 관련 객체를 하나로 묶은 군집
  - 애그리거트 간의 관계로 도메인 모델을 이해하고 구현하여 큰 틀에서 도메인 모델을 관리
  - 루트 엔티티 : 애그리거트에 속한 객체를 관리하는 엔티티 ex) Order
    - 애그리거트에 속한 엔티티와 밸류를 이용하여 애그리거트가 구현해야할 기능을 제공
    - 애그리거트 루트를 통해 간접적으로 다른 엔티티나 밸류 객체에 접근 -> 애그리거트 단위로 구현을 캡슐화
```java
public class Order {
    public void changeShippingInfo(ShippingInfo newlnfo) {
        checkShippinglnfoChangeableO; // 배송지 변경 가능 여부 확인
        this.shippinglnfo = newlnfo;
    }
}
```
    - 배송지를 변경하려면 루트 엔티티인 Order 를 통해 변경 -> 캡슐화
- 리포지터리
  - 애그리거트 단위로 도메인 객체를 저장하고 조회하는 기능을 정의
  - Repository 인터페이스 : 고수준 모듈, 도메인 영역
  - Repository 의 구현체 : 저수준 모듈, 인프라스트럭처 영역
  - 리포지터리는 응용 서비스가 필요로하는 메서드를 제공

## 2.5 요청 처리 흐름
1. 표현 영역응 HTTP 요청 파라미터를 응용 서비스가 필요로하는 데이터로 변환해서 응용 서비스를 실행할 때 인자로 전달
2. 응용 서비스는 기능 구현에 필요한 도메인 객체를 리포지토리에서 가져와서 실행하거나 새로운 도메인 객체를 생성하여 저장
3. 응용 서비스는 도메인 로직 실행 결과를 표현 영역에 반환하고 표현 영역은 HTTP로 응답

## 2.6 인프라스트럭처 개요
- 표현, 응용, 도메인 영역을 지원
- 도메인 영역의 영속 처리, 트랜잭션, SMTP, 구현 기술, 보조 기능 등을 지원
- 항상 인프라스트럭처에 대한 의존을 없앨 필요는 없음. ex) @Transactional (스프링에 대한 의존)

## 2.7 모듈 구성
- 영역별로 모듈이 위치할 패키지로 구성
- 도메인이 크면 하위 도메인으로 나누고 각 하위 도메인 마다 별도 패키지를 구성
- 도메인 모듈은 도메인에 속한 애그리거트를 기준으로 다시 패키지를 구성

---

# 의문

## OrderService는 응용 영역이고 CaculateDiscountService는 도메인 서비스인 이유?

- 응용 서비스는 도메인 객체들을 이용해서 흐름만 조정, 비니지스 로직이 필요하면 이걸 도메인 서비스로 분리해서 응용서비스에서 도메인 서비스를 호출
- ex) 게시글 작성할때 특정 단어 들어가면 안된다는 비지니스 로직이 있으면 이거를 검사하는 서비스(도메인 서비스)로 분리해서 사용
```java
// PostService = 응용 영역 서비스
public class PostService {

    private final ContentFilterService contentFilterService; // 도메인 영역 서비스
    private final PostRepository postRepository;

    public PostService(ContentFilterService contentFilterService,
                       PostRepository postRepository) {
        this.contentFilterService = contentFilterService;
        this.postRepository = postRepository;
    }

    public void writePost(String authorId, String title, String content) {
        // 도메인 서비스에 위임
        contentFilterService.validate(content);

        Post post = new Post(authorId, title, content);
        postRepository.save(post);
    }
}

```