# 7장. 도메인 서비스

## **여러 애그리거트가 필요한 기능**

한 애그리거트에 넣기 애매한 도메인 기능을 특정 애그리거트에서 억지로 구현하면 안된다. 
자신의 책임 범위를 넘어서는 기능을 구현하게되면 코드가 길어지고 외부 의존이 높아진다. 
때문에 코드가 복잡해지고 유지보수하기 어려워진다. 
게다가 애그리거트의 범위를 넘어서는 도메인 개념이 애그리거트에 숨어들어서 명시적으로 드러나지 않게 된다.

이런 문제를 해소하는 가장 쉬운 방법이 바로 도메인 서비스를 별도로 구현하는 것이다.

## **도메인 서비스**

한 애그리거트에 넣기 애매한 도메인 개념을 구현하기위해 도메인 서비스를 이용해서 도메인 개념을 명시적으로 드러낼 수 있다. 
응용 영역의 서비스가 응용 로직을 다루듯, 도메인 서비스는 도메인 로직을 다룬다.

도메인 영역의 애그리거트나 밸류와 다른 점은 상태 없이 로직만 구현한다는 점이다. 
도메인 서비스를 구현하는데 필요한 상태는 애그리거트나 다른 방법으로 전달받는다.

---

```java
// 할인 금액 계산 로직을 위한 도메인 서비스
public class DiscountCalculationService {
	public Money calculateDiscountAmounts(
			List<OrderLIne> orderLines,
			List<Coupon> coupons,
			MemberGrade grade) {
		Money couponDiscount = coupons.stream()
                                      .map(coupon -> calculateDiscount(coupon))
                                      .reduce(Money(0), (v1, v2) -> v1.add(v2));

		Money membershipDiscount = calculateDiscount(orderer.getMember().getGrade());

		return couponDiscount.add(membershipDiscount);
	}

	...
}
```

이러한 도메인 서비스를 사용하는 주체는 애그리거트가 될 수도 있고 응용 서비스가 될 수도 있다.

위 도메인 서비스(DiscountCalculationService)를 애그리거트의 결제 금액 계산 기능에 전달하면 사용 주체는 애그리거트가 된다.

```java
// 주문 애그리거트
public class Order {
	public void calculateAmounts(DiscountCalculationService disCalSvc, MemberGrade grade) {
		Money totalAmounts = getTotalAmounts();
		Money discountAmounts = disCalSvc.calculateDiscountAmounts(this.orderLInes, this.coupons, greade);
		this.paymentAmounts = totalAmounts.minus(discountAmounts);
	}
	...
```

애그리거트 객체에 도메인 서비스를 전달하는 것은 응용 서비스 책임이다.

```java
// 주문 응용 서비스
public class OrderService {
	private DiscountCalculationService discountCalculationService;

	@Transactional
	public OrderNo placeOrder(OrderRequest orderRequest) {
		OrderNo orderno = orderRepository.nextId();
		Order order = createOrder(orderNo, orderRequest);
		orderRepository.save(order);
		// 응용 서비스 실행 후 표현 영역에서 필요한 값 리턴

		return orderNo;
	}

	private Order createOrder(OrderNo orderNo, OrderRequest orderReq) {
		Member member =findMember(orderReq.getOrdererId());
		Order order = new Order(orderNo, orderReq.gerOrderLines(),
							orderReq.getCoupons(), createOrderer(member),
							orderReq.getShippingInfo());
		order.calculateAmounts(this.discountCalculationService, member.getGrade());
		return order;
	}
	...
}
```

---

> **도메인 서비스 객체를 애그리거트에 주입하지 않기**

위 예시처럼 애그리거트의 메서드를 실행할 때 도메인 서비스 객체를 파라미터로 전달한다는 것은 애그리거트가 도메인 서비스에 의존한다는 것을 뜻한다. 이를 의존 주입으로 처리하고 싶어질 수 있다.

하지만, 필자는 이는 좋은 방법이 아니라고 생각한다. 

도메인 객체는 필드(프로퍼티)로 구성된 데이터와 메서드를 이용한 기능을 이용해서 개념적으로 하나인 모델을 표현한다. 모델의 데이터를 담는 필드는 모델에서 중요한 구성요소이다. 그런데, 도메인 서비스 필드는 데이터 자체와는 관련이 없다. 해당 도메인 객체를 DB에 저장할 때 다른 필드와는 달리 저장 대상도 아니다. 또, 주입한 도메인 서비스를 해당 도메인에서 제공하는 모든 기능에서 필요로 하는 것도 아니다. 

그러므로 굳이 도메인 서비스 객체를 애그리거트에 의존 주입할 이유는 없다.

---

애그리거트 메서드를 실행할 때 도메인 서비스를 인자로 전달하지 않고 반대로 도메인 서비스의 기능을 실행할 때 애그리거트를 전달하기도 한다.

```java
// 계좌 이체 도메인 서비스
public class TransferService {
	public void transfer(Account fromAcc, Account toAcc, Money amounts) {
		fromAcc.withdraw(amounts);
		toAcc.credit(amounts);
	}
}
```

응용 서비스는 두 Account 애그리거트를 구한 뒤에 해당 도메인 영역의 TransferService 를 이용해서 계좌 이체 도메인 기능을 실행할 것이다.

도메인 서비스는 도메인 로직을 수행하지 응용 로직을 수행하지는 않는다.

트랜잭션 처리와 같은 로직은 응용 로직이므로 도메인 서비스가 아닌 응용 서비스에서 처리해야 한다.

> 특정 기능이 응용 서비스인지 도메인 서비스인지 감을 잡기 어려울 때는 해당 로직이 애그리거트의 상태를 변경하거나 애그리거트의 상태 값을 계산하는지 검사해 보면 된다. 예를 들어, 계좌 이체 로직은 계좌 애그리거트의 상태를 변경한다. 결제 금액 로직은 주문 애그리거트의 주문 금액을 계산한다. 이 두 로직은 각각 애그리거트를 변경하고 애그리거트의 값을 계산하는 도메인 로직이다. 도메인 로직이면서 한 애그리거트에 넣기 적합하지 않으므로 이 두 로직은 도메인 서비스로 구현하게 된다.

## 도메인 서비스의 패키지 위치

도메인 서비스는 도메인 로직을 실행하므로 다른 도메인 구성 요소와 동일한 패키지(domain)에 위치한다.

도메인 서비스의 개수가 많거나 엔티티나 밸류와 같은 다른 구성요소와 명시적으로 구분하고 싶다면 domain 패키지 밑에 domain.model, domain.service, domain.repository와 같이 하위 패키지를 구분해서 위치시켜도 된다.

## 도메인 서비스의 인터페이스와 클래스

도메인 서비스의 구현이 특정 기술에 종속되면 인터페이스와 구현 클래스로 분리한다.

도메인 서비스의 구현이 특정 구현 기술에 의존적이거나 외부 시스템의 API를 실행한다면 도메인 영역의 도메인 서비스는 인터페이스로 추상화해야 한다. 이를 통해 도메인 영역이 특정 구현에 종속되는 것을 방지할 수 있고 도메인 영역에 대한 테스트가 수월해진다.
