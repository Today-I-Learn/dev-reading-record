# 2장. 아키텍처 개요

- 아키텍처
- DIP
- 도메인 영역의 주요 구성요소
- 인프라스터럭처
- 모듈

---

## 네 개의 영역

아키텍처를 설계할 때 전형적으로 표현, 응용, 도메인, 인프라스트럭처 네 개의 영역으로 나누게 된다.

### 표현 영역

사용자의 요청을 해석해서 응용 서비스에 전달하고 응용 서비스의 실행 결과를 사용자가 이해할 수 있는 형식으로 변환해서 응답한다.

### 응용 영역

도메인 모델을 이용해서 시스템이 사용자에게 제공해야 할 기능을 구현한다.

실제 로직 구현은 도메인 모델에 위임한다.

### 도메인 영역

도메인 모델을 구현하고, 도메인의 핵심 로직을 구현한다.

### 인프라스트럭처 영역

RDBMS 연동, 메세징 전송 & 수신, HTTP 클라이언트를 이용한 REST API 호출 등 논리적인 개념을 표현하기보다는 실제 구현을 다룬다.

표현, 응용, 도메인 영역은 구현 기술을 사용한 코드를 직접 만들지 않고, 인프라스트럭처 영역에서 제공하는 기능을 사용해서 필요한 기능을 개발한다.

---

## 계층 구조 아키텍처

기본적으로 계층 구조는 표현 → 응용 → 도메인 → 인프라스트럭처 로 구성되고, 특성상 상위 계층에서 하위 계층으로의 의존만 존재하고 하위 계층은 상위 계층에 의존하지 않는다.

계층 구조를 엄격하게 적용하면 상위 계층은 바로 아래의 계층에만 의존을 가져야 하지만 보통 유현하게 적용한다.

예를 들어 응용 계층은 외부 시스템과의 연동을 위해 두단계 아래 계층인 인프라스트럭처 계층에 의존하기도 한다.

계층 구조는 직관적으로 이해하기는 쉽지만 표현, 응용, 도메인 계층이 상세한 구현 기술을 다루는 인프라스트럭처 계층에 종속된다는 문제가 있다.

첫번째 문제로는 테스트 하기 어려워진다는 문제점이 있고, 두번째 문제는 구현 방식을 변경하기 어렵다는 점이다.

결론적으로 이처럼 인프라스트럭처에 의존하게 되면 '테스트 어려움'과 '기능 확장의 어려움' 이라는 두가지 문제가 발생한다.

이를 해결하기 위해 DIP를 적용할 수 있다.

---

## DIP

고수준 모듈 - 의미 있는 단일 기능을 제공하는 모듈

저수준 모듈 - 고수준 모듈의 기능을 구현하기 위한 하위 기능을 실제로 구현한 모듈

고수준 모듈이 제대로 동작하려면 저수준 모듈을 사용해야하는데 이렇게 되면 계층 구조 아키텍처에서 언급했던 두 가지 문제('테스트 어려움'과 '기능 확장의 어려움')가 발행한다.

DIP는 이 문제를 해결하기 위해 추상화한 인터페이스를 통해 저수준 모듈이 고수준 모듈에 의존하도록 바꾼다.

고수준 모듈의 개념을 추상화한 인터페이스는 고수준 모듈에 속하게 되고 실제 구현(저수준 모듈)은 이를 의존(상속은 의존의 다른 형태)한다.

이렇게 반대로 저수준 모듈이 고수준 모듈에 의존한다고 해서 이를 DIP(Dependency Inversion Principle, 의존 역전 원칙)라고 부른다.

### DIP 주의사항

DIP 를 적용할 때 단순하게 인터페이스와 구현 클래스 분리가 아니라 하위 기능을 추상화한 인터페이스는 고수준 모듈 관점에서 도출해야한다. 즉, 저수준 모듈에서 인터페이스를 추출하면 안된다.

### DIP와 아키텍처

아키텍처 수준에서 DIP 를 적용하면 인프라스트럭처 영역이 응용 영역과 도메인 영역에 의존하는 구조가 된다.

인프라스트럭처에 위치한 클래스가 도메인이나 응용 영역에 정의한 인터페이스를 상속받아 구현하는 구조가 되면 도메인과 응용 영역에 영향을 최소화하면서 구현 기술을 변경하는 것이 가능하다.

## 도메인 영역의 주요 구성요소

도메인 영역의 모델은 도메인의 주요 개념을 표현하며 핵심이 되는 로직을 구현한다.

|요소|설명|
|:---|:---|
|엔티티 (ENTITY)|고유의 식별자를 갖는 객체로 자신의 라이프 사이클을 가짐 <br> 도메인의 고유한 개념을 표현하고 도메인 모델의 데이터를 포함하며 해당 데이터와 관련된 기능을 함께 제공한다. <br> ex) 주문(Order), 회원(Member), 상품(Product)|
|밸류 (VALUE)|개념적으로 하나인 도메인 객체의 속성을 표현할 때 사용된다. <br> 엔티티의 속성으로 사용될 뿐만 아니라 다른 밸류 타입의 속성으로도 사용될 수 있다. <br> ex) 주소(Address), 금액(Money)|
|애그리거트 (AGGREGATE)|관련된 엔티티와 밸류 객체를 개념적으로 하나로 묶은 것 <br> ex) 주문과 관련된 Order 엔티티, OrderLine 밸류, Orderer 밸류 객체를 '주문' 애그리거트로 묶을 수 있다.|
|리포지터리 (REPOSITORY)|도메인 모델의 영속성을 처리한다. <br> ex) DBMS 테이블에서 엔티티 객체를 로딩 하거나 저장하는 기능을 제공|
|도메인 서비스 (DOMAIN SERVICE)|특정 엔티티에 속하지 않은 도메인 로직을 제공 <br> ex) '할인 금액 계산'은 상품, 쿠폰, 회원 등급, 구매 금액 등 다양한 조건을 이용해서 구현 - 도메인 로직이 여러 엔티티와 밸류를 필요로 할 경우 도메인 서비스에서 로직을 구현한다.|

---

### 엔티티와 밸류

도메인 모델의 엔티티와 DB 관계형 모델의 엔티티는 같은 것이 아니다!

가장 큰 차이점은 도메인 모델의 엔티티는 데이터와 함께 도메인 기능을 함께 제공한다는 점이다. 예를 들어, 주문을 표현하는 엔티티는 주문과 관련된 데이터뿐만 아니라 배송지 주소 변경을 위한 기능을 함께 제공한다.

또 다른 차이점은 도메인 모델의 엔티티는 두 개 이상의 데이터가 개념적으로 하나인 경우 밸류 타입을 이용해서 표현할 수 있다는 것이다. RDBMS와 같은 관계형 데이터베이스는 밸류 타입을 제대로 표현하기 힘들다. 덕분에 도메인 모델의 엔티티를 통해 도메인을 보다 잘 이해할 수 있게 된다.

밸류는 불변으로 구현하는 것을 권장하는데, 이는 엔티티의 밸류 타입 데이터를 변경할 때 객체 자체를 완전히 교체한다는 것을 의미한다.

---

### 애그리거트

도메인 모델의 구성요소는 규모가 커질수록 복잡해진다.

도메인 모델을 볼 때 개별 객체뿐만 아니라 상위 수준에서 모델을 볼 수 있어야 전체 모델의 관계와 개별 모델을 이해하는데 도움이 된다. 도메인 모델에서 전체 구조를 이해하는데 도움이 되는 것이 바로 애그리거트(AGGREGATE)이다.

한마디로, 애그리거트는 관련 객체를 하나로 묶은 군집이다. 애그리거트를 통해 큰 틀에서 도메인 모델을 관리할 수 있게 된다.

애그리거트는 군집에 속한 객체들을 관리하는 루트 엔티티를 갖고, 이는 애그리거트에 속해 있는 엔티티와 밸류 객체를 이용해서 애그리거트가 구현해야 할 기능을 제공한다.

애그리거트를 사용하는 코드는 애그리거트 루트를 통해 애그리거트 내의 다른 엔티티나 밸류 객체에 접근하게 되고 이는 내부 구현을 숨겨서 애그리거트 단위로 구현을 캡슐화 하기 위함이다.

애그리거트를 어떻게 구성했느냐에 따라 구현이 복잡해지기도 하고 트랜잭션 범위가 달라지기도 한다.

---

### 리포지터리

도메인 객체를 지속적으로 사용하려면 물리적인 저장소에 도메인 객체를 보관해야 하는데, 이를 위한 도메인 모델이 리포지터리(repository)이다.

리포지터리는 애그리거트 단위로 도메인 객체를 저장하고 조회하는 기능을 정의한다. 애그리거트 루트는 애그리거트에 속한 모든 객체를 포함하고 있으므로 결과적으로 애그리거트 단위로 저장하고 조회한다.

도메인 모델 관점에서 리포지터리 인터페이스는 도메인 객체를 영속화하는데 필요한 기능을 추상화한 것으로 고수준 모듈에 속한다. 기반 기술을 이용해서 해당 인터페이스를 구현한 리포지터리 구현체는 저수준 모듈로 인프라스트럭처 영역에 속한다.

---

## 요청 처리 흐름

사용자가 애플리케이션에 기능 실행을 요청하면 처음에 표현 영역에서 요청을 받게된다. 스프링 MVC 에서는 컨트롤러가 그 역할을 하게 된다.

표현 영역은 사용자가 전송한 데이터 형식이 올바른지 검사하고 문제가 없다면 데이터를 이용해서 응용 서비스에 기능 실행을 위임한다.

응용 서비스는 도메인 모델을 이용해서 기능을 구현한다. 기능 구현에 필요한 도메인 객체를 리포지터리에서 가져와 실행하거나 신규 도메인 객체를 생성해서 리포지터리에 저장한다.

---

## 인프라스트럭처 개요

인프라스트럭처(Infrastructure)는 표현 영역, 응용 영역, 도메인 영역을 지원한다. 이때 도메인 영역과 응용 영역에서 인프라스트럭처의 기능을 직접 사용하는 것보다 이 두 영역에 정의한 인터페이스를 인프라스트럭처 영역에서 구현하는것이 시스템을 더 유연하고 테스트하기 쉽게 만들어준다.

구현의 편리함은 DIP가 주는 장점(변경의 유연함, 테스트가 쉬움)만큼 중요하기 때문에 DIP의 장점을 해치지 않는 범위에서 응용 영역과 도메인 영역에서 구현 기술에 대한 의존을 가져가는 것이 현명하다. 응용 영역과 도메인 영역이 인프라스트럭처에 대한 의존을 완전히 갖지 않도록 시도하게 되면 자칫 구현을 더 복잡하고 어렵게 만들 수 있기 때문이다.

---

## 모듈 구성

아키텍처의 각 영역은 별도 패키지에 위치한다.

모듈 구조를 얼마나 세분화해야 하는지에 대한 정해진 규칙은 없다. 단지, 한 패키지에 너무 많은 타입이 몰려서 코드를 찾을 때 불편한 정도만 아니면 된다. 패키지가 너무 복잡해지는 것 같다면 모듈을 분리하는 시도를 하면 될 것이다.
