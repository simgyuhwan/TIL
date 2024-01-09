# 예외 처리

JPA 표준 예외들은 `javax.persistence.PersistenceException` 의 자식 클래스다. 그리고 이 예외 클래스는 `RuntimeException`의 자식이다. 따라서 JPA 예외는 모두 언체크 예외다.

JPA 표준 예외는 크게 두 가지로 나눌 수 있다. 

- **트랜잭션 롤백을 표시하는 예외**
- **트랜잭션 롤백을 표시하지 않는 예외**

### 트랜잭션을 표시하는 예외

| 예외 | 설명 |
| --- | --- |
| javax.persistence.EntityExistsException | EntityManager.persist(...) 호출 시 이미 같은 엔티티가 있으면 발생 |
| javax.persistence.EntityNotFoundException | EntityManager.getReference(...)를 호출했는데 실제 사용 시 엔티티가 존재하지 않으면 발생. refresh(), lock() 에서도 발생 |
| javax.persistence.OptimisticLockException | 낙관적 락 충돌 시 발생 |
| javax.persistence.PessimisticLockException | 비관적 락 충돌 시 발생 |
| javax.persistence.RollbackException | EntityTransaction.commit() 실패 시 발생. 롤백이 표시되어 있는 트랜젝션 커밋 시에도 발생 |
| javax.persistence.TransactionRequiredException | 트랜잭션이 필요할 때 트랜젝션이 없으면 발생. 트랜잭션 없이 엔티티를 변경할 때 주로 발생 |

### 트랜잭션을 표시하지 않는 예외

| javax.persistence.NoResultException | Query.getSingleResult() 호출 했는데 결과가 없을 때 반환 |
| --- | --- |
| javax.persistence.NonUniqueResultException | Query.getSingleResult() 호출 했는데 결과가 2 이상일 때 반환 |
| javax.persistence.LockTimeoutException | 비관적 락에서 시간 초과 시 발생 |
| javax.persistence.QueryTimeoutException | 쿼리 실행 시간 초과 시 발생 |

## 스프링 프레임워크의 JPA 예외 변환

일반 JPA를 사용하고 예외가 발생하면 서비스 계층에도 예외가 전달된다. 이는 데이터 접근 계층의 구현 기술이 서비스 계층까지 영향을 주는 것이므로 좋은 설계라고 볼 수 없다.

**스프링 프레임워크**에서는 이를 해결하기 위해 JPA 예외를 추상화해서 제공해준다. J**PA 예외 변환기**를 사용하려면 `PersistenceExceptionTranslationPostProcessor` 를 스프링 빈으로 등록하면 된다. 빈으로 등록하면 `@Repository`로 사용된 곳을 AOP를 적용하여 추상화한 예외로 변환해준다. 

```java
@Bean
public PersistenceExceptionTranslationPostProcessor exceptionTranslation{
	return new PersistenceExceptionTranslationPostProcessor();
}
```

# 엔티티 비교

영속성 컨텍스트 내부에는 엔티티 인스턴스를 비교하기 위해 1차 캐시가 존재한다. 만약 영속성 컨텍스트가 동일하다면 모두 이 1차 캐시에서 가져오기 때문에 엔티티 간의 **동일성을 보장**한다.

- **동일성** : == 비교
- **동등성** : equals() 비교
- **데이터베이스 동등성** : @Id인 데이터베이스 식별자 비교

하지만 다른 영속성 컨텍스트라면 동일성을 보장하지 않는다. 내부적인 필드의 값이 동일하더라도 서로 다른 인스턴스로 다른 주소를 가지고 있기 때문이다.

이때는 **동등성**을 비교해야 한다. 동등성을 비교하기 위해 **변경되지 않고 유일성을 가진 값**을 사용하는 것이 좋다.

# N+1의 해결

## 1. 페치 조인 사용

N+1 문제를 해결하는 가장 일반적인 방법은 페치 조인을 사용하는 것이다. 페치 조인은 SQL 조인을 사용해서 연관된 엔티티를 함께 조회하므로 N+1 문제가 발생하지 않는다.

```java
select m from Member m join fetch m.orders
```

## 2. 하이버네이트 @BatchSize

하이버네이트가 제공하는**BatchSize** 애너테이션을 사용하면 연관된 엔티티를 조회할 때, 지정한 size만큼 SQL의 **IN**절을 사용해서 조회한다. 만약 10건을 조회한다면 5개씩 두 번 조회한다.

```java
@org.hibernate.annotations.BatchSize(size = 5)
@OneToMany(mappedBy = "member", fetch = FetchType.EAGER)
private List<Order> orders = new ArrayList<>();
```

<aside>
💡 **hibernate.default_batch_fetch_size를 사용하면 애플리케이션 전체에 기본으로 @BatchSize를 적용한다.**

</aside>

## 3. 하이버네이트 @Fetch(FetchMode.SUBSELECT)

**FetchMode**를 **SUBSELECT** 로 사용하면 **서브쿼리**를 이용하여 N+1를 해결한다.

```java
@org.hibernate.annotations.Fetch(FetchMode.SUBSELECT)
@OneToMany(mappedBy = "member", fetch=FetchType.EAGER)
private List<Order> orders = new ArrayList<>();
```

다음과 같은 쿼리가

```java
select m from Member m where m.id > 10
```

이렇게 서브쿼리로 변환된다.

```java
SELECT O FROM ORDERS O WHERE O.MEMBER_ID IN(SELECT M.ID FROM MEMBER M WHERE M.ID > 10)
```

## 쓰기 지연

쓰기 지연이란 insert를 대량으로 할 때, 하나씩 데이터베이스에 전달하면 지연이 발생할 수 있는데. 이를 하이버네이트에서 제공하는 SQL 배치 기능을 사용하면 한 번에 여러 건을 모아서 처리할 수 있게 된다. 

```java
hibernate.jdbc.batch_size=50 // 한 번에 50개 씩 모아서 SQL 배치 사용
```