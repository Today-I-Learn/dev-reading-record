# 5장 리포지터리의 조회 기능 (JPA 중심)

## 검색을 위한 스펙

리포지터리는 애그리거트의 저장소이다. 애그리거트를 저장하고 찾고 삭제하는 것이 기본 기능이다.

애그리거트를 찾을 때 식별자를 이용하는 것이 기본이지만 식별자 외에 여러 다양한 조건으로 애그리거트를 찾아야 할 때가 있다. 검색 조건의 조합이 다양할 경우 **스펙**을 이용해야 한다.

### 스펙

스펙 (Specification)은 애그리거트가 특정 조건을 충족하는지 검사한다.

```java
public interface Specification<T> {
  public boolean isSatisfiedBy(T agg);
}
```

`isSatisfiedBy()` 메서드는 검사 대상 객체가 조건을 충족하면 true를 리턴하고, 그렇지 않으면 false를 리턴한다.

### 스펙 조합

두 스펙을 AND 연산자나 OR 연산자로 조합해서 새로운 스펙을 만들 수 있고, 조합한 스펙을 다시 조합해서 더 복잡한 스펙을 만들 수 있다.

```java
public class AndSpec<T> implements Specification<T> {
  private List<Specification<T>> specs;

  public AndSpecification(Specification<T> ... specs) {
    this.spec = Arrays.asList(specs);
  }

  public boolean isSatisfiedBy(T agg) {
    for (Specification<T> spec : specs) {
      if (!spec.isSatisfied(agg)) {
        return false;
      }
    }
    return true;
  }
}
```

specs라는 List를 만들어 `isSatisfiedBy()` 메서드를 통해 해당 스펙들을 검사하여 AND로 조합하는 스펙 AndSpec을 구현할 수 있다. 마찬가지로 OrSpec도 비슷하게 구현할 수 있다.

## JPA를 위한 스펙 구현

위의 예로 보여준 방식은 모든 애그리거트를 조회한 다음에 스펙을 이용해서 걸러내는 방식을 사용하는데 이 방식을 이용하면 실행 속도 문제가 발생한다.

위 문제를 해결하려면 쿼리의 where 절에 조건을 붙여서 필요한 데이터를 걸러야 한다.

이는 스펙 구현도 메모리에서 걸러내는 방식에서 쿼리의 where를 사용하는 방식으로 바꿔야 한다는 뜻이다.

### JPA 스펙 구현

JPA는 다양한 검색 조건을 조합하기 위해 CriteriaBuilder와 Predicate를 사용한다.

```java
public interface Specification<T> {
  Predicate toPredicate(Root<T> root, CriteriaBuilder cb);
}
```

```java
public class OrdererSpec implements Specification<Order> {
  private String OrdererId;

  ...

  @Override
  public Predicate toPredicate(Root<Order> root, CriteriaBuilder cb) {
    return cb.equal(root.get(Order_.orderer)
                      .get(Orderer_.memberId).get(MemberId_.id), ordererId);
    }
}
```

OrdererSpec의 `toPredicate()` 메서드는 Order의 `orderer.memberId.id` 프로퍼티가 생성자로 전달받은 ordererId와 같은지 비교하는 Predicate를 생성해서 리턴한다.

응용 서비스는 원하는 스펙을 생성하고 리포지터리에 전달해서 필요한 애그리거트를 검색하면 된다.

Specification 구현 클래스를 개별적으로 만들지 않고 별도 클래스에 스펙 생성 기능을 모아도 된다.

### @StaticMetamodel

위 코드에 Order_ 클래스와 Orderer_ 클래스가 있는데 이 것은 JPA의 정적 메타 모델을 정의한 코드이다.

@StaticMetamodel 애노테이션을 이용해서 관련 모델을 지정한다. 메타 모델 클래스는 모델 클래스의 이름 뒤에 '_'을 붙인 이름을 갖는다.

정적 모델 클래스는 대상 모델의 각 프로퍼티와 동일한 이름을 갖는 정적 필드를 정의한다. 이 정적 필드를 프로퍼티에 대한 메타 모델로서 프로퍼티 타입에 따라 SingualAttribute, ListAttribute 등의 타입을 사용해서 메타 모델을 정의한다.

정적 메타 모델을 사용하는 대신 문자열로 프로퍼티를 지정할 수도 있지만 코드 안정성이나 생산성 등의 이유로 정적 메타 모델 클래스를 사용한다.

[참고 자료](https://docs.jboss.org/hibernate/orm/5.4/topical/html_single/metamodelgen/MetamodelGenerator.html)

### AND/OR 스펙 조합을 위한 구현

생성자로 전달받은 Specification 리스트를 Predicate 리스트로 바꾸고 CriteriaBuilder의 and() 와 or()를 사용해서 새로운 Predicate를 생성한다.

### 스펙을 사용하는 JPA 리포지터리 구현

1. 스펙의 루트를 생성
2. 파라미터로 전달받은 스펙을 이용해서 Predicate를 생성
3. Predicate 전달

## 정렬 구현

정렬 순서가 고정된 경우에는 CreteriaQuery#orderBy()나 JPQL의 order by 절을 이용하면 되지만, 정렬 순서를 응용 서비스에서 결정하는 경우에는 정렬 순서를 리포지터리에 전달할 수 있어야 한다.

정렬 순서를 지정하는 코드는 리포지터리를 사용하는 응용 서비스에 위치하게 되는데 응용 서비스는 CriteriaBuilder에 접근할 수 없다. 따라서 응용 서비스는 JPA Order가 아닌 다른 타입을 이용해서 리포지터리에 정렬 순서를 전달하고 JPA 리포지터리는 다시 JPA Order로 변환하는 작업을 해야 한다.

## 페이징과 개수 구하기 구현

JPA 쿼리는 `setFirstResult()`와 `setMaxResults()` 메서드를 제공하고 있는데 이 두 메서드를 이용해서 페이징을 구현할 수 있다.

- `setFirstResult()` 메서드는 읽어올 척 번째 행 번호를 지정한다.
- `setMaxResults()` 메서드는 읽어올 행 개수를 지정한다.

## 조회 전용 기능 구현

리포지터리는 애그리거트의 저장소를 표현하는 것으로서 다음 용도로 리포지터리를 사용하는 것은 적합하지 않다.

- 여러 애그리거트를 조합해서 한 화면에 보여주는 데이터 제공
    - JPA의 지연 로딩과 즉시 로딩 설정, 연관 매핑의 어려움
    - 애그리거트 간에 직접 연관을 맺으면 ID로 참조할 때의 장점을 활용할 수 없다.
- 각종 통계 데이터 제공
    - 다양한 테이블을 조인하거나 DBMS 전용 기능을 사용해야 하는데 JPQL이나 Criteria로 처리하기 힘들다.

이런 기능은 조회 전용 쿼리로 처리해야 한다.

### 동적 인스턴스 생성

JPQL에서 new 키워드 뒤에 생성할 인스턴스의 완전한 클래스 이름을 지정하고 괄호 안에 생성자에 인자로 전달할 값을 지정한다.

조회 전용 모델을 만드는 이유는 표현 영역을 통해 사용자에게 데이터를 보여주기 위함이다.

동적 인스턴스의 장점은 JPQL을 그대로 사용하므로 객체 기준으로 쿼리를 작성하면서도 동시에 지연/즉시 로딩과 같은 고민 없이 원하는 모습으로 데이터를 조회할 수 있다는 점이다.

### 하이버네이트 @Subselect

- @Subselect
    - 쿼리 결과를 @Entity로 매핑할 수 있는 기능이다.
    - 조회 쿼리를 값으로 갖는다. 하이버네이트는 select 쿼리의 결과를 매핑할 테이블처럼 사용한다. DBMS에서 여러 테이블을 조인해서 조회한 결과를 한 테이블처럼 보여주기 위한 용도로 뷰를 사용하는 것처럼 쿼리 실행 결과를 매핑할 테이블처럼 사용한다.
    - 뷰를 수정할 수 없듯이 @Subselect로 조회한 @Entity 역시 수정할 수 없다. 실수로 수정하면 매핑한 테이블이 없으므로 에러가 발생한다.
    - 일반 @Entity와 같기 때문에 EntityManger#find(), JPQL, Criteria를 사용해서 조회할 수 있다는 장점이 있다.
    - @Subselect의 값으로 지정한 쿼리를 from 절의 서브쿼리로 사용한다.
    - 서브쿼리를 사용하고 싶지 않다면 네이티브 SQL 쿼리를 사용하거나 MyBatis와 같은 별도 매퍼를 사용해서 조회 기능을 구현해야 한다.
- @Immutable
    - 위와 같은 문제를 방지하기 위해 사용된다.
    - 해당 엔티티의 매핑 필드/프로퍼티가 변경되어도 DB에 반영하지 않고 무시한다.
- @Synchronize
    - 하이버네이트는 변경사항을 트랜잭션을 커밋하는 시점에 DB에 반영하므로 반영하지 않은 상태에서 조회하면 이전의 값이 발생하는데 이 문제를 해결하기 위한 용도로 사용한다.
    - 지정한 테이블과 관련된 변경이 발생하면 플러시를 먼저 실행한다.
