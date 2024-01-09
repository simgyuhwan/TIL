# 기본 데이터형

## 1. null

```sql
{"x" : null}
```

## 2. 불리언

```sql
{"x" : true}
```

## 3. 숫자

셸은 64비트 부동소수점 수를 기본으로 사용한다.

```sql
{"x" : 3.14}
{"x" : 3} 
```

4바이트 혹은 8바이트 부호 정수는 각각 **NumberInt, NumberLong** 클래스를 사용한다.

```sql
{"x" : NumberInt("3")}
{"x" : NumberLong("3")}
```

## 4. 문자열

어떤 UTF-8 문자열이든 문자열형으로 표현할 수 있다.

```sql
{"x" : "foobar"}
```

## 5. 날짜

```sql
{"x" : new Date()}
```

몽고 DB에서 날짜를 표현하려면 **꼭 생성자를 사용하자.** 만약 그냥 Date() 를 사용하면 문자열로 반환이 되어 날짜 타입이 되지 않는다. 아래의 예시를 보자. 

```sql
video> Date()
Fri Sep 29 2023 00:48:29 GMT+0900 (대한민국 표준시)

video> new Date()
ISODate("2023-09-28T15:48:34.606Z")
```

## 6. 정규 표현식

쿼리는 자바스크립트의 정규 표현식 문법을 사용할 수 있다.

```sql
{"x" : /foobar/i}
```

## 7. 배열

값의 set, list를 배열로 표현할 수 있다.

```sql
{"x" : ["a", "b", "c"]}
```

배열은 정렬 연산(리스트, 스택, 큐)과 비정렬 연산(셋)에 호환성 있게 사용 가능한 값이다.

```sql
{"things" : ["pie", 3.14]}
```

위의 예처럼 **배열의 값은 데이터형이 자유롭다**.

또 몽고DB는 배열을 단순한 값의 집합이 아니라 **필드처럼** 인식한다. 이게 무슨 말인가 하면 예를 들어, `db.list.find({hobbies: “reading”})` 은 hobbies 필드에 reading이 포함된 모든 문서를 찾아준다.

이런 식으로 배열을 쿼리한다거나 자주 쓰는 요소라면 **인덱스를 생성**할 수도 있다.

## 8. 내장 도큐먼트

```sql
{"x" : {"foo" : "bar"}}
```

몽고DB에서는 도큐먼트도 키의 값이 될 수 있다. 이를 **내장 도큐먼트**라고 하는데. 아래의 예시를 살펴보자.

```json
{"name" : "John Doe",
	"address" : {
		"street" : "123 Park Street",
		 "city" : "Anytown",
		 "state" : "NY"
	}
}
```

배열과 마찬가지로 몽고DB는 내장 도큐먼트의 구조를 ‘이해’하고, 인덱스를 구성하여 쿼리하며 갱신하기 위해 내장 도큐먼트에 접근할 수 있다.

RDBS에서는 이러한 구조를 두 개의 테이블로 저장하고 변경이 있으면 조인을 통해 한 번에 변경할 수 있다. 하지만 **몽고DB에서는 각 도큐먼트에서 오타를 수정해야 한다**.

## 9. 객체 ID

객체 ID는 도큐먼트용 12바이트 ID다.

```sql
{"x" : ObjectID()}
```

## 10. 코드

쿼리와 도큐먼트는 임의의 자바스크립트 코드를 포함할 수 있다. 

```sql
{"x" : function() { /* ,,, */}}
```

# _id와 ObjectId

RDBS의 Primary 키처럼 몽고DB도 고유한 Id를 가지는데. **‘_id’** 가 바로 고유한 ID이다. 같은 컬렉션 내에서는 ‘_id’는 중복될 수 없다. 하지만 다른 컬렉션에는 중복될 수 있다. 데이터형은 기본적으로 **ObjectId**이다.

## ObjectId

몽고 DB에서 `_id`의 생성은  서버가 아닌 클라이언트에서 진행된다. 샤딩된 각 구역에서 `_id`를 생성하기 때문에 중복이 있으면 안된다. `_id`의 기본 크기는 12바이트의 이진 데이터이다. 이 12바이트를 4개로 분류할 수 있는데. 

- **4 byte** : 처음 4 byte는 1970년 1월 1일 0시 0분 0초부터 현재까지의 **초**를 표현한다.
- **3 byte** : Mac + Ip address 혹은 hostname을 md5 hash 한 값에서 처음 3 byte를 가져와서 표현한다.
- **2 byte** : Process ID 정보를 담는다. 만약 64 bit 머신이라면 이 3 byte를 넘을 수 있는데. 하위 2 byte만 담는다.
- **3 byte** : 마지막 3 byte는 임의의 값에서 시작하여 자동으로 증가하는 카운터 값을 나타낸다. 동시에 여러 개가 생길 수 있기 때문에 겹치지 않게 +1 씩 증가시켜준다.

중간의 3 + 2 byte는 같은 위치에서 생성됨을 보장할 수 있다. 이런 식으로 생성됐기 때문에 분산된 환경에서도 unique 함을 유지할 수 있게 된다.