# 7장 도메인 서비스

## 7.1 여러 애그리거트가 필요한 기능
- 결제 금액을 계산해야하는 경우
```java
public class Order {
    private Orderer orderer;
    private List<OrderLine> orderLines;
    private List<Coupon> usedCoupon;

    private Money calculatePayAmounts() {
        // 할인 금액 계산 책임을 모두 주문 애그리거트에 할당
    }
}

```
- 한 애그리거트에 넣기 애매한 도메인 기능을 억지로 특정 애그리거트에 구현하면 안됨
  - 애그리거트는 자신의 책인 범위를 넘어서는 기능을 구현하게 됨
  - 코드가 길어지고 외부에 대한 의존이 높아짐
  - 코드가 복잡해지고 수정이 어려워짐
- 해결 방법 : 도메인 기능을 별도 서비스로 구현

## 7.2 도메인 서비스
- 도메인 서비스를 사용하는 상황
  - 계산 로직 : 여러 애그리거트가 필요한 계산 로직이나, 한 애그리거트에 넣기에는 복잡한 계산 로직
  - 외부 시스템 연동이 필요한 도메인 로직 : 구현하기 위해 타 시스템을 사용해야하는 도메인 로직

- 한 애그리거트에 넣기 애매한 개념을 구혀하려면 도메인 서비스를 이용해서 도메인 개념을 명시적으로 드러내면 됨
  - 응용 서비스는 응용 로직을 다루고 도메인 서비스는 도메인 로직을 다룸
- 도메인 서비스는 상태 없이 로직만 구현
```java
public class DiscountCalculationService {
    public Money calculateDiscountAmounts(
        List<OrderLine> orderLines,
        List<Coupon> coupons,
        MemberGrade grade) {
        // 계산 로직
    }
}

```
- 할인 계산 서비스를 사용하는 주체는 애그리거트 또는 응용 서비스
- 사용주체가 애그리거트인 경우
```java
public class Order {
    public void calculateAmounts(DiscountCalculationService disCalSvc, MemberGrade grade) {
        Money totalAmounts = getTotalAmounts();
        Money discountAmounts =
            disCalSvc.calculateDiscountAmounts(this.orderl_ines, this.coupons, grade);
        this.paymentAmounts = totalAmounts.minus(discountAmounts);
    }
}

```
- 이 때, 애그리거트 객체에 도메인 서비스를 전달하는 것은 응용서비스 책임
```java
public class OrderService {
    private DiscountCalculationService discountCalculationService;

    @Transactional
    public OrderNo placeOrder(OrderRequest orderRequest) {
        ...
        Order order = createOrder(orderNo, orderRequest);
        ...
    }

    private Order createOrder(OrderNo orderNo, OrderRequest orderReq) {
        ...
        order.calculateAmounts(this.discountCalculationService, member.getGradeO);
        ...
    }
}
```

- 도메인 서비스의 기능을 실행할 때 애그리거트를 전달하는 경우
```java
public class TransferService {
    public void transfer(Account fromAcc, Account toAcc, Money amounts) {
        fromAcc.withdraw(amounts);
        toAcc.credit(amounts);
    }
}
```
- 응용 서비스는 Account 애그리거트를 구한 후, 도메인 서비스인 TransferService를 이용해 계좌 이체 도메인 기능을 실행
- 트랜잭션 처리는 응용 로직이므로 도메인 서비스가 아닌 응용 서비스에서 처리
- 외부 시스템이나 타 도메인과의 연동 기능도 도메인 서비스가 될 수 있음
- ex) 설문조사 생성시, 사용자 권한을 체크 (설문조사와 사용자 시스템이 별도로 존재)
```java
public class CreateSurveyService { // 응용 서비스

    private SurveyPermissionChecker permissionchecker; // 도메인 서비스

    public Long createSurvey(CreateSurveyRequest req) {
        validate(req);
        // 도메인 서비스를 이용해서 외부 시스템 연동을 표현
        if (!permissionChecker.hasUserCreationPermission(req.getRequestorIdO)) {
            throw new NoPermissionException();
        }
    }
}
```
- SurveyPermissionChecker의 구현체는 인프라스트럭처 영역. 시스템 연동을 포함한 권한 검사 기능 구현
- 도메인 서비스는 도메인 로직을 표현하므로 도메인 영역 패키지에 위치
```
- order
  - application: OrderService
  - domain : Order, OrderRepository(interface), DiscountCalculateService
```

- 도메인 서비스의 구현이 특정 구현 기술에 의존하거나 외부 시스템의 API를 실행한다면 도메인 서비스는 인터페이스로 추상화
  - 이 때, 도메인 서비스의 인터페이스는 도메인 영역에 위치, 구현체는 인프라스트럭처에 위치 -> 도메인 영역이 특정 구현에 종속되는 것을 방지