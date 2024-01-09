# JPA와 컬렉션

하이버네이트는 컬렉션을 영속성 컨텍스트에 넣으면 이 컬렉션을 **하이버네이터가 제공하는 내장 컬렉션**으로 감싸고 관리한다. 이렇게 컬렉션을 감싸는 이유는 다음과 같다. 

- 내장 컬렉션은 지연 로딩을 지원한다.
- 내장 컬렉션은 변경 감지 기능을 제공한다.

이런 기능을 사용하기 위해 영속성 컨텍스트에 넣는 순간 내장 컬렉션 감싸게 되는데. Collection의 종류마다 조금의 차이가 있다. 

## 1. Collection, List

`Collection, List` 는 **중복을 허용**하고 **순서에 상관없이** 저장된다. add() 로 데이터를 넣으면 무조건 true이다.

특징으로는 엔티티를 추가할 때, 중복 검사를 하지 않기 때문에 **지연 로딩된 컬렉션을 초기화하지 않는다.**

## 2. Set

`Set`은 **PersistentSet** 을 컬렉션 레퍼로 사용하는데. 데이터를 넣으면 해시 값을 사용해서 equals를 확인하기 때문에 이 과정에서 **지연 로딩된 컬렉션을 초기화**한다. 또 중복을 허용하지 않는다.

## 3. List + @OrderColumn

```java
@OneToMany(mappedBy = "board")
@OrderColumn(name = "POSITION")
private List<Comment> comments = new ArrayList<Comment>();
```

OrderColumn 애너테이션을 사용하면 일대다에서 **다**쪽에 순서를 저장하는 칼럼이 생성된다. 이 칼럼을 사용해서 컬렉션의 순서를 보장한다. 하지만 **실무에서는 사용을 지양**하는 것이 좋은데. 그 이유는 다음과 같다.

- **다쪽에 만들어지기 때문에 관리가 어렵다.**
- **중간 순번의 엔티티를 지우게 되면 자리를 다시 정렬하기 위해 UPDATE가 여러번 발생한다.**
- **데이터베이스에서 강제로 중간 순번의 엔티티를 삭제할 경우, NullPointException이 발생한다.**

## 4. @OrderBy

OrderBy는 컬렉션을 조회할 때, 자동으로 쿼리에 **OrderBy**를 붙여서 조회해준다. 그러면 아래와 같이 쿼리가 진행된다.

```java
@OneToMany(mappedBy = "team")
@OrderBy("username desc, id asc")
private Set<Member> members = new HashSet<Member>();

// 쿼리
select M.* from member m where m.team_id=? order by m.member_name desc, m.id asc;
```

# Converter

컨버터를 사용하면 엔티티의 데이터를 변환해서 데이터베이스에 저장할 수 있다.

예를 들어, 회원의 VIP를 기존의 자바는 boolean으로 저장하는데. 데이터베이스에는 1, 0 으로 저장하고 싶다고 해보자. 

```java
// Entity 클래스
class Member {
	...
	@Convert(converter=BooleanToTnConverter.class)
	private boolean vip;
}

// Converter 구현체
@Converter
public class BooleanToYNConverter implements AttributeConverter<Boolean, String> {

	// field -> database
	@Override
	public String convertToDatabaseColumn (Boolean attribute) {
		return (attribute != null && attribute) ? "Y" : "N";
	}

	// database -> field
	@Override
	public Boolean convertToEntityAttribute(String dbData) {
		return "Y".equals(dbData);
	}
}
```

컨버터를 사용하려면 **@Converter** 애너테이션을 사용하고 **AttributeConverter** 인터페이스를 구현해야 한다.

### AttributeConverter Interface

```java
public interface AttributeConverter<X,Y> {
	public Y convertToDatabaseColumn (X attribute);
	public X convertToEntityAttribute(Y dbData)
}
```

같은 효과로 **Convert 속성을 클래스**에 줄 수도 있다.

```java
@Entity
@Convert(converter=BooleanToYNConverter.class, attributeName = "vip")
public class Member {
	...
}
```

### 글로벌 설정

모든 Boolean 타입에 공통적으로 컨버터를 적용하고 싶다면 구현체 위에 아래와 같이 적용하면 된다. 

```java
@Converter(autoApply=true)
public class BooleanToYNConverter implements AttributeConverter<Boolean, String> {
	...
}
```

# 리스너

모든 엔티티를 대상으로 언제 어떤 사용자가 삭제를 요청했는지 모두 로그를 남겨야 하는 요구사항이 있다면 모든 삭제 로직에 로그를 추가하는 로직을 넣어줘야 한다. JPA 리스너 기능을 사용하면 간단하게 처리할 수 있다.

## 1. 이벤트의 종류

- **PostLoad** : 엔티티가 영속성 컨텍스트에 조회된 직후 또는 refresh를 호출한 후
- **PrePersist** : persist() 메소드를 호출해서 엔티티를 영속성 컨텍스트에 관리하기 직전에 호출된다. 식별자 생성 전략을 사용한 경우 엔티티에 식별자는 아직 존재하지 않는다. 새로운 인스턴스를 merge할 때도 수행된다.
- **PreUpdate** : flush, commit 을 호출해서 엔티티를 데이터베이스에 수정하기 직전에 호출된다.
- **PreRemove** : remove() 메소드를 호출해서 엔티티를 영속성 컨텍스트에서 삭제하기 직전에 호출된다.
- **PostPersist** : flush나 commit을 호출해서 엔티티를 데이터베이스에 저장한 직후에 호출된다. 식별자는 항상 존재한다.
- **PostUpdate** : flush나 commit을 호출해서 엔티티를 데이터베이스에 수정한 직후에 호출된다.
- **PostRemove** : flush나 commit을 호출해서 엔티티를 데이터베이스에 삭제한 직후 호출된다.

## 2. 이벤트 적용 위치

### 1) 엔티티에 직접 적용

```java
@Entity
public class Duck {
	@Id @GeneratedValue
	public Long id;

	private String name;

	@PrePersist
	public void prePersist() {
		...
	}

	@PostPersist
	public void postPersist() {
		...	
	}

	@PostLoad
	public void postLoad() {
		...
	}
}
```

### 2) 별도의 리스너 등록

```java
@Entity
@EntityListeners(DuckListener.class)
public class Duck {
	...
}

public class DuckListener {

	@PrePersist
	private void prePersist(Object obj) {
		...
	}

	@PostPersist
	private void postPersist(Obejct obj) {

	}
}
```

반환 타입은 **void**로 설정해야 한다.

### 3) 기본 리스너

XML을 사용하면 되는데. 사용안할 것 같으니 제외한다.

이벤트 호출 순서는 아래의 순서로 진행된다.

- **기본 리스너**
- **부모 클래스 리스너**
- **리스너**
- **엔티티**