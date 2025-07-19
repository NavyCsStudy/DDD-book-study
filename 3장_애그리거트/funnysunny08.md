## 3.1 애그리거트

<img width="772" height="528" alt="Image" src="https://github.com/user-attachments/assets/f57c2968-e3a6-4015-8683-a04812aea837" />

- 도메인 객체 모델이 복잡해지면 개별 구성요소 위주로 모델을 이해하게 되고 전반적인 구조나 큰 수준에서 도메인 간의 관계를 파악하기 어려워진다.
    - 코드를 변경하고 확장하는 것이 어려워진다.
- 복잡한 도메인을 이해하고 관리하기 쉬운 단위로 만들려면 상위 수준에서 모델을 조망할 수 있는 방법이 필요하다.
- 애그리거트는 관련된 객체를 하나의 군으로 묶어준다.
- 장점
    - 모델을 이해하는 데 도움을 준다
    - 일관성을 관리하는 기준
    - 복잡한 구조를 단순한 구조로 만들어줌
    - 복잡도가 낮아지는 만큼 기능을 확장하고 변경하는데 필요한 노력도 줄어든다.
- 애그리거트는 경계를 가지며, 한 애그리거트에 속한 객체는 다른 애그리거트에 속하지 않는다.
    - 경계를 설정하는 기준은 도메인 규칙과 요구사항이다.
    - 도메인 규칙에 따라 함께 생성, 변경되는 구성요소는 한 애그리거트에 속할 가능성이 높다. (생명주기)
- A가 B를 갖는다 → 같은 애그리거트일까?
    - 항상 그런 것은 아니다. 상품과 리뷰를 예로 들면, 상품이 생성될 때 리뷰가 생성되나? 변경이 함께 이루어져야 하나?
- 처음 도메인 모델을 만들기 시작하면 큰 애그리거트로 보이는 것들이 많지만, 도메인에 대한 경험이 생기고 도메인 규칙을 제대로 이해할수록 애그리거트의 실제 크기는 줄어든다.

## 3.2 애그리거트 루트

<img width="448" height="473" alt="Image" src="https://github.com/user-attachments/assets/996f4a4f-1ffe-4c94-90f6-b14223584140" />

- 애그리거트에 속한 모든 객체가 일관된 상태를 유지하려면 애그리거트 전체를 관리할 주체가 필요하다 → **애그리거트의 루트 엔티티**
- 애그리거트에 속한 객체는 애그리거트 루트 엔티티에 직간접적으로 속하게 된다.

### 3.2.1 도메인 규칙과 일관성

- 애그리거트 루트의 핵심 역할은 애그리거트의 일관성이 깨지지 않도록 하는 것으로 도메인 규칙에 따라 **애그리거트가 제공해야 할 도메인 기능**을 구현한다.
    - Ex) 주문 애그리거트 - 배송지 변경, 상품 변경
- 애그리거트 외부에서 애그리거트에 속한 객체를 직접 변경하면 안 된다.
    - 애그리거트 루트가 강제하는 규칙을 적용할 수 없어 모델의 일관성이 깨짐
- 그렇다면 도메인 규칙을 위한 상태 검증 같은 것들응 응용 서비스에서 구현하면?
    - 중복 구현할 가능성이 높아져 유지보수가 어려워진다.
- 불필요한 중복을 피하고 애그리거트 루트를 통해서만 도메인 로직을 구현하게 만들려면 도메인 모델에 대해 다음의 두 가지를 습관적으로 적용해야 한다.
    1. set 메서드를 public으로 만들지 않는다.
        - 도메인의 의미나 의도를 표현하지 못한다.
        - 도메인 로직을 객체가 아닌 응용 영역이나 표현 영역으로 분산시킨다.
            - 도메인 로직의 응집도가 떨어져 유지보수가 어려워진다.
    2. 밸류 타입은 불변으로 구현한다.
        - 애그리거트 외부에서 밸류 객체의 상태를 변경할 수 없다.
- 애그리거트 루트가 도메인 규칙을 올바르게만 구현하면 애그리거트 전체의 일관성을 올바르게 유지할 수 있다.

### 3.2.2 애그리거트 루트의 기능 구현

- 애그리거트 루트는 애그리거트 내부의 다른 객체를 조합해서 기능을 완성한다.
- 구성요소의 상태만 참조하는 것이 아니라 기능 실행을 위임하기도 한다.

```java
public class Order {
		private OrderLines orderLines; // 애그리거트 내 다른 객체 참조
		
		public void changeOrderLines(List<OrderLine> newLines) {
				orderLines.changeOrderLines(newLines);
				// totalAmount 계산하는 기능 구현을 OrderLines에 위임
				this.totalAmounts = orderLines.getTotalAmount(); 
		}
}				
```

### 3.2.3 트랜잭션 범위

- 트랜잭션 범위는 작을 수록 좋다.
    - 잠금 대상이 많아지면 동시에 처리할 수 있는 트랜잭션 개수 또한 줄어들기 때문 (처리량)
- 한 트랜잭션에서는 한 개의 애그리거트만 수정해야 한다.
    - 이 말은 한 애그리거트에서 다른 애그리거트를 변경하지 않는다는 것을 의미한다.
- 만약 부득이하게 한 트랜잭션에서 두 개 이상의 애그리거트를 수정해야 한다면 애그리거트에서 다른 애그리거트를 직접 수정하지 말고 **응용 서비스에서 두 애그리거트를 수정하도록 구현**한다.

```java
public class ChangeOrderService {
		@Transactional
		public void changeShippingInfo(OrderId id, ShippingInfo newShippingInfo) {
				Order order = orderRepository.findById(id);
				order.shipTo(newShippingInfo);
				Member member = findMember(order.getOrderer());
				member.changeAddress(newShippingInfo.getAddress());
		}
}				
```

## 3.3 리포지터리와 애그리거트

- 애그리거트는 개념상 완전한 한 개의 도메인 모델을 표현하므로 객체의 영속성을 처리하는 리포지터리는 애그리거트 단위로 존재한다.
    - Order와 OrderLine이 있을 때, 리포지터리는 OrderRepository만 생성한다.
- 애그리거트는 개념적으로 하나이므로 리포지터리는 애그리거트 전체를 저장소에 영속화해야 한다.
    - Order를 저장할 때 관련 객체 모두 함께 저장해야 한다. (Order, OrderLine, Orderer)
- 리포지터리가 완전한 애그리거트를 제공하지 않으면 필드나 값이 올바르지 않아 기능 수행 중 `NullPointerException`이 발생할 수 있다.
- 트랜잭션을 이용해서 애그리거트의 변경이 원자적으로 저장소에 반영되는 것을 보장할 수 있다.

## 3.4 ID를 이용한 애그리거트 참조

- 애그리거트 간의 참조를 필드를 통해 구현하게 된다면?
    
    <img width="843" height="537" alt="Image" src="https://github.com/user-attachments/assets/a4c6568e-3678-4589-897a-bf5b2b4cb354" />
    
    - JPA에서는 `@ManyToOne`, `@OneToMany` 애너테이션을 이용해서 연관된 객체를 로딩하는 기능을 제공한다.
    - 구현의 편리함은 있지만 다음과 같은 문제가 있다.
        - **편한 탐색 오용:** 다른 애그리거트의 상태를 쉽게 변경할 수 있게 된다.
            - 한 애그리거트에서 다른 애그리거트의 상태를 변경하는 것은 애그리거트 간의 의존 결합도를 높여서 결과적으로 애그리거트의 변경을 어렵게 만든다.
        - **성능에 대한 고민:** 지연 로딩 vs. 즉시 로딩
        - **확장 어려움:** 필드로 참조하게 되면 그 애그리거트들은 동일한 DB 종류를 사용해야 한다.
- ID 참조를 사용하면 모든 객체가 참조로 연결되지 않고 한 애그리거트에 속한 객체들만 참조로 연결된다.
    - 👍 애그리거트의 경계를 명확히 하고 애그리거트 간 물리적인 연결을 제거하기 때문에 모델의 복잡도를 낮춰준다.
    - 👍 응집도를 높여준다.
    - 👍 구현 복잡도가 낮아진다.
    - 👍 애그리거트별로 다른 구현 기술을 사용하는 것도 가능해진다.

### 3.4.1 ID를 이용한 참조와 조회 성능

- 다른 애그리거트를 ID로 참조하면 추가 조회 쿼리가 발생하기 때문에 조회 속도가 문제될 수 있다.

```java
Member member = memberRepository.findById(orderId);
List<Order> orders = orderRepository.findByOrderer(orderId);
List<OrderView> dtos = orders.stream()
				.map(order -> {
						ProductId prodId = order.getOrderLines().get(0).getProductId();
						// 각 주문마다 첫번째 주문 상품 정보 로딩을 위한 추가 n개의 쿼리 발생
						Product product = productRepository.findById(orderId);
						return new OrderView(order, member, product);
				}).collect(toList());
```

👉 N + 1 문제 발생!

- 조회 전용 쿼리를 사용하여 해결할 수 있다.

```java
@Repository
public class JpaOrderViewDao implements OrderViewDao {
		@PersistenceContext
		private EntityManager em;
		
		@Override
		public List<OrderView> selectByOrderer(String orderId) {
				String selectQuery = 
						"select new ~";
						
						...
		}
}		
```

- 애그리거트마다 서로 다른 저장소를 사용하면 한 번의 쿼리로 관련 애그리거트를 조회할 수 없다.
    - 이때는 조회 성능을 높이기 위해 캐시를 적용하거나 조회 적용 저장소를 따로 구성한다.
    - 코드가 복잡해지는 단점이 있지만 시스템의 처리량을 높일 수 있다.

## 3.5 애그리거트 간 집한 연관

- 카테고리와 상품 간의 연관을 생각해 보면,
    - 카테고리 입장에서 한 카테고리에 한 개 이상의 상품이 속할 수 있으므로 카테고리와 상품은 1-N 관계이다.
    - 한 상품이 한 카테고리에만 속할 수 있다면 상품과 카테고리는 N-1 관계이다.
- 이러한 요구사항을 모두 반영하게 된다면?
    - 카테고리에 속한 상품은 매우 많기 때문에 보통 조회를 한다 하더라도 페이징을 통해 나눠서 가져온다.
    - **이러한 성능 문제 때문에 애그리거트 간의 1-N 연관을 실제 구현에 반영하지 않는다.**
    - 카테고리에 속한 상품을 구할 필요가 있다면, 상품 입장에서 자신이 속한 카테고리를 N-1로 연관 지어 구하면 된다.
    
    ```java
    public class Product {
    		...
    		private CategoryId categoryId;
    		...
    }		
    ```
    
    ```java
    public class ProductListService {
    		public Page<Product> getProductOfCategory(Long categoryId, int page, int size) {
    				Category category = categoryRepository.findById(categoryId);
    				List<Product> products = productRepository.findByCategoryId(category.getId(), page, size);
    				int totalCount = productRepository.countsByCategoryId(category.getId());
    				return new Page(page, size, totalCount, products);
    		}
    		...				
    ```
    
- 카테고리의 양방향 M-N 연관이 존재하지만 실제 구현에서는 상품에서 카테고리로의 단방향 M-N 연관만 적용하면 된다.

## 3.6 애그리거트를 팩토리로 사용하기

- 신고를 통해 차단된 상점은 물건을 등록하지 못하는 도메인 규칙이 있다.

```java
public class RegisterProductService {
		public ProductId registerNewProduct(NewProductRequest req) {
				Store store = storeRepository.findById(req.getStoreId());
				checkNull(store);
				if (store.isBlock()) {
						throw new StoreBlockedException();
				}
				... 상품 등록 
		}
}		
```

- 위의 코드는 Product를 생성 가능한지 판단하는 코드와 Product를 생성하는 코드가 분리되어 있어, 중요한 도메인 로직 처리가 응용 서비스에 노출되었다.
    - 이 도메인 기능을 넣기 위한 별도의 도메인 서비스나 팩토리 클래스를 만들 수도 있지만 이 기능을 Store 애그리거트에 구현할 수 있다.
    
    ```java
    public class Store {
    		public Product createProduct(ProductId newProductId, ...) {
    				if (isBlocked()) throw new StoreBlockedException();
    				return new Product(newProductId, getId(), ...);
    		}
    }			
    
    public class RegisterProductService {
    		public ProductId registerNewProduct(NewProductRequest req) {
    				Store store = storeRepository.findById(req.getStoreId());
    				checkNull(store);
    				ProductId id = productRepository.nextId();
    				Product product = store.createProduct(id, ...);
    				productRepository.save(product);
    				return id;
    		}		
    }			 
    ```
    
    - Store 애그리거트의 `createProduct())`는 Product 애그리거트를 생성하는 팩토리 역할을 한다.
    - 장점
        - Store가 Product를 생성할 수 있는지를 확인하는 도메인 로직은 Store에서 구현하므로, Product 생성 가능 여부를 확이하는 도메인 로직을 변경해도 도메인 영역의 Store만 변경하면 되고 응용 서비스는 영향을 받지 않는다.
        - ⇒ 도메인의 응집도도 높아졌다.
    - 💡 애그리거트가 갖고 있는 데이터를 이용해서 다른 애그리거트를 생성해야 한다면 애그리거트에 팩토리 메서드를 구현하는 것을 고려해보자!
    - Store 애그리거트가 Product 애그리거트를 생성할 때 많은 정보를 알아야 한다면, Store 애그리거트에서 Product 애그리거트를 직접 생성하지 않고 다른 팩토리에 위임하는 방법도 있다.