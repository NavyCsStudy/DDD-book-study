## 1.1 도메인이란?

- 소프트웨어로 해결하고자 하는 문제 영역
- 한 도메인은 다시 하위 도메인으로 나눌 수 있으며, 한 하위 도메인은 다른 하위 도메인과 연동하여 완전한 기능을 제공한다.
    - ex) 온라인 서점 도메인
    
    ![image.png](/1%EC%9E%A5_%EB%8F%84%EB%A9%94%EC%9D%B8%20%EB%AA%A8%EB%8D%B8%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/image/fs_image.png)
    

## 1.2 도메인 전문가와 개발자 간 지식 공유

- 제품 개발과 관련된 도메인 전문가, 관계자, 개발자가 같은 지식을 공유하고 직접 소통할 수록 도메인 전문가가 원하는 제품을 만들 가능성이 높아진다.

## 1.3 도메인 모델

- 도메인 모델을 사용하면 여러 관계자들이 동일한 모습으로 도메인을 이해하고 도메인 지식을 공유하는 데 도움이 된다.
- 클래스 다이어그램, 상태 다이어그램, 수학 공식 등 다양한 방법을 활용하여 표현할 수 있다.

![image.png](/1%EC%9E%A5_%EB%8F%84%EB%A9%94%EC%9D%B8%20%EB%AA%A8%EB%8D%B8%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/image/fs_image2.png)

![image.png](/1%EC%9E%A5_%EB%8F%84%EB%A9%94%EC%9D%B8%20%EB%AA%A8%EB%8D%B8%20%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0/image/fs_image3.png)

## 1.4 도메인 모델 패턴

- 일반적인 애플리케이션 아키텍처 구성 : 표현 - 응용 - 도메인 - 인프라스트럭처
- 도메인 모델은 아키텍처 상의 **도메인 계층을 객체 지향 기법으로 구현하는 패턴**을 말한다.
- 도메인 계층은 도메인의 핵심 규칙을 구현한다.
    - 주문 도메인의 경우 ‘출고 전에 배송지를 변경할 수 있다’, ‘주문 취소는 배송 전에만 할 수 있다’ 라는 규칙을 구현한 코드는 도메인 계층에 위치하게 된다.
    - 주문과 관련된 중요 업무 규칙은 주문 도메인 모델인 `Order`나 `OrderState`에서 구현해야 한다.
- **핵심 규칙을 구현한 코드는 도메인 모델에만 위치**하기 때문에 **규칙이 바뀌거나 규칙을 확장해야 할 때 다른 코드에 영향을 덜 주고 변경 내역을 모델에 반영할 수 있게 된다.**

```java
public class Order {
		private OrderState state;
		private ShippingInfo shippingInfo;
		
		public void changeShippingInfo(ShippingInfo shippingInfo) {
				if (!isShippingChangeable()) {
						throw new IllegalStateException("can't change shipping in " + state);
				}
				this.shippingInfo = shippingInfo;
		}
		
		private boolean isShippingChangeable() {
				return state == OrderState.PREPARING;
		}
}

혹은

state.isShippingChangeable()

public enum OrderState {
		PREPARING {
				public boolean isShippingChangeable() {return true;}
		},
		SHIPPED, DELIVERING, ..;
		
		public boolean isShippingChangeable() {return false;}
}		
```

## 1.5 도메인 모델 도출

- 도메인을 모델링할 때 기본이 되는 작업은 모델을 구성하는 핵심 구성요소, 규칙, 기능을 찾는 것이다.
- 이 과정은 요구사항에서 출발한다.
- Order와 관련된 기능을 모델링하는 과정 35p~40p 참고

## 1.6 엔티티와 밸류

- 도출한 모델은 크게 엔티티와 밸류로 구분할 수 있다.

### 1.6.1 엔티티

엔티티는 식별자를 가진다.

- Order가 엔티티가 되고 안의 상태가 바뀌더라도 식별자를 유지한다.
- 엔티티의 식별자 생성
    - 특정 규칙에 따라 생성
    - UUID나 Nano ID와 같은 고유 식별자 생성기 사용
    - 값을 직접 입력
    - 일련변호 사용 (시퀀스나 DB의 자동 증가 컬럼 사용)

### 1.6.2 밸류

밸류 타입은 개념적으로 완전한 하나를 표현할 때 사용한다.

예를 들어, 배송 정보는 받는 사람과 주소로 구성되는 것을 알 수 있다.

```java
public class ShippingInfo {
		private Receiver receiver;
		private Address address;
		...
}		
```

- 밸류 타입이 꼭 두 개 이상의 데이터를 가져야 하는 것은 아니며, 의미를 명확하게 하기 위해 밸류 타입을 사용하는 경우도 있다.
    - Ex
    
    ```java
    public class Money {
    		private int value;
    		
    		public Money(int value) {
    				this.value = value;
    		}
    }
    	
    private Money price;
    private Money amount;
    ```
    
- 밸류 타입을 위한 기능을 추가할 수 있다.
    
    ```java
    public class Money {
    		private int value;
    		
    		... 생성자, getter
    		
    		// '정수 타입 연산'이 아니라 '돈 계산'이라는 의미
    		public Money add(Money money) {
    				return new Money(this.value + money.value);
    		}
    }
    ```
    
- 밸류 객체의 데이터를 변경할 때는 기존 데이터를 변경하기보다 변경한 데이터를 갖는 새로운 밸류 객체를 생성하는 방식을 선호한다.
    - 불변성으로 안전한 코드를 작성할 수 있게 된다. (+ 참조 투명성과 스레드에 안전한 특징)
        - 참조 투명성: 모든 프로그램 p에 대해 표현식 e의 모든 출현을 e의 평가로 치환해도 p의 의미에 아무 영향이 미치지 않는다면, 그 표현식 e는 참조에 투명하다.
- 두 밸류 객체를 비교할 때는 모든 속성이 같은 비교한다.

### 1.6.4 엔티티 식별자와 밸류 타입

- 식별자를 위한 밸류 타입을 사용해서 의미가 잘 드러나도록 한다.
    - Order의 식별자를 String이 아닌 **OrderNo 밸류 타입**을 통해 해당 필드가 주문번호라는 것을 알 수 있다.

### 1.6.5 도메인 모델에 set 메서드 넣지 않기

- set 메서드는 도메인의 핵심 개념이나 의도를 코드에서 사라지게 한다.
    - set 메서드는 필드값만 변경하고 끝나기 때문에 상태 변경과 관련된 도메인 지식이 코드에서 사라진다.
- 도메인 객체가 불완전한 상태로 사용되는 것을 막으려면 생성자를 통해 필요한 데이터를 모두 받아야 하며, 생성 시점에 데이터가 올바른지 검사한다.

## 1.7 도메인 용어와 유비쿼터스 언어

- 도메인에서 사용하는 용어를 코드에 최대한 반영하여 코드를 해석하는 과정을 줄인다.

```java
public enum OrderState {
		PAYMENT_WAITING, PREPARING, SHIPPED, DELIVERING, DELIVERY_COMPLETED;
}
```