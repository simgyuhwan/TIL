**페치(fetch) 조인**은 JPQL의 성능 최적화를 위해 제공하는 기능이다. 일반적으로 **N+1** 문제를 해결하기 위해 사용되는데.  N+1이란, 일대다 관계에서 다쪽의 데이터를 가져오기 위해 쿼리를 1개씩 사용하는 것을 의미한다. 만약 List로 객체가 1000개가 있다면 1000개의 쿼리가 진행된다. 

**기본 문법**

`[ LEFT [OUTER] | INNER ] JOIN FETCH 조인경로`

# JPQL

페치 조인을 사용해서 회원 엔티티를 조회하면서 연관된 팀 엔티티를 조회하는 JPQL을 살펴보자. 

`select m from Member m join fetch m.team`

join 인 다음에 fetch를 입력했고 JPQL에서는 m.team 뒤에는 별칭을 사용할 수 없다. 

위의 쿼리는 실제 다음과 같이 쿼리가 진행된다. 

`SELECT  M.*, T.* FROM MEMBER M JOIN TEAM T ON M.TEAM_ID=T.ID;`

## 컬렉션 페치 조인

팀 엔티티에는 일대다로 많은 맴버를 가지고 있다고 하자. 특정 팀의 `List<Member> members` 를 모두 조회하기 위해 **fetch join** 을 사용해보자.

```sql
// 컬렉션 페치 조인 SQL
SELECT T FROM TEAM T JOIN FETCH T.members WHERE t.name = 'Korea'

// 실행된 SQL
SELECT M.*, T.* FROM TEAM T INNER JOIN MEMBER M ON T.ID=M.TEAM_ID WHERE T.name='Korea';
```

위와 같은 쿼리를 실행하면 다음과 같이 조회가 될 것이다. 

| ID(PK) | NAME | ID(PK) | TEAM_ID | NAME |
| --- | --- | --- | --- | --- |
| 1 | Korea | 1 | 1 | 안정환 |
| 1 | Korea | 2 | 1 | 메시 |

**동일한 TEAM ID가 중복되어 조회**되는 것을 확인할 수 있는데.

실제로 쿼리 실행 후, 반환된 Team을 보면 List<Team> 형태로 두 개를 가져온 것을 확인할 수 있다. 

하지만 두 **Team 내용물은 동일하다**.

이럴땐 레코드 자체의 중복을 제거하기 위해 **DISTINCT** 를 사용하자

## 페치 조인과 DISTINCT

SQL의 DISTINCT는 중복을 제거하는 명령어이다. JPQL에서는 SELECT 쿼리에 **DISTINCT를 추가**하고 **애플리케이션에서 한 번 더 중복을 제거**한다.

```sql
SELECT DISTINCT T FROM TEAM T JOIN FETCH T.members WHERE T.name='Korea'
```

## 페치 조인과 일반 조인의 차이

페치 조인을 사용하지 않고 조인만 사용하면 어떻게 될까?

```sql
// 내부 조인 JPQL
SELECT T FROM TEAM T JOIN T.members m WHERE T.NAME='Korea'

// 실행된 SQL
SELECT T.* FROM TEAM T INNER JOIN MEMBER M ON T.ID=M.TEAM_ID WHERE T.NAME='Korea';
```

예상과는 달리 일반 조인은 MEMBER를 가져오지 않는다. 오직 TEAM만 가져오고 JPQL은 연관관계를 고려하지 않는다. 단지 SELECT 절에 지정한 엔티티만 조회할 뿐이다.

페치 조인은 연관된 엔티티까지 SELECT 절에 추가해서 조회해준다. 

## 페치 조인의 특징과 한계

페치 조인을 사용하면 연관된 엔티티를 함께 조회할 수 있어서 효과적이다. 그렇지만 주의해야 할 것들이 있다. 

### 1) 페치 조인 대상에는 별칭을 줄 수 없다.

- JPA에서는 페치 조인에 대한 별칭 기능이 없다. 그렇기 때문에 억지로 넣으면 예상과는 다른 결과가 나올 수 있기 때문에 사용하지 말자.

### 2) 둘 이상의 컬렉션을 페치할 수 없다.

- 둘 이상의 컬렉션을 페치 조인하면 카테시안 곱이 만들어지므로 사용하지 말자.

### 3) 컬렉션을 페치 조인하면 페이징 API(setFirstResult, setMaxResults)를 사용할 수 없다.

- 컬렉션이 아닌 단일 값 연관 필드는 페치 조인을 해도 페이징 API 사용이 가능하다.
- 하이버네이트에서 컬렉션을 페치 조인하고 페이징 API를 사용하면 경고 로그를 남기면서 메모리에서 페이징 처리된다. 데이터가 많으면 성능 이슈가 있기 때문에 주의하자.