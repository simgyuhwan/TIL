# 1. find

몽고DB에서 **find** 함수는 쿼리에 사용한다. find는 컬렉션에서 첫 번째 매개변수와 일치하는 도큐먼트를 가져온다. 첫 번째 매개변수를 다음과 같이 `{}` 로 지정하면 컬랙션 내의 모든 도큐먼트와 일치하게 된다.

`> db.c.find({})` == `> db.c.find()`

만약 `()` 안에 아무것도 입력하지 않으면 `{}` 와 동일한 쿼리로 인식한다.

쿼리는 Key/Value 형식으로 지정할 수 있다. 

`> db.user.find({”age” : 27})`

`> db.user.find({”age”: 27, “name”: “holy”});`

만약 여러 조건을 추가하면 **AND** 조건으로 처리된다.

## 반환 받을 키 지정

도큐먼트에서 원하는 키/값만 필요할 때가 있다. 그럴때는 두 번째 매개변수를 지정하면 되는데. **1**이면 반환받는 다는 뜻이고 **0**이면 제외한다는 뜻이다.

```sql
// 저장된 데이터
test> db.users.find()
[
  {
    _id: ObjectId("6516cd29cdce029126e9781d"),
    relationships: { friends: 32, enemies: 2 },
    username: 'joe'
  },
  {
    _id: ObjectId("65183a8febac8301af2872cb"),
    username: 'hello',
    email: 'holy@naver.com'
  }
]

// username 가져오고 _id는 빼는 쿼리
test> db.users.find({}, {"username":1 , "_id" : 0})
[ { username: 'joe' }, { username: 'hello' } ]
```

## 제약 사항

쿼리에는 몇 가지 제약이 있는데. 쿼리 내에 **변수**나 **함수**를 사용할 수 없다.

예를 들어, 다음과 같은 쿼리는 실패한다.

```sql
db.users.find({age: {$gt: Math.random() * 100}})
```

이 쿼리는 age가 0~100 사이의 랜덤한 값보다 큰 도큐먼트를 찾는 쿼리인데. 몽고DB는 쿼리 도큐먼트에 함수를 허용하지 않으므로, 이 쿼리는 에러를 발생시킨다.

<aside>
💡 쿼리 도큐먼트 값이 상수여야 한다는 제약은 몽고DB의 설계 원칙과 관련있다. 몽고DB에서 인덱스는 특정 필드의 값에 따라 도큐먼트를 정렬한 자료구조인데, 이 쿼리 도큐먼트 값이 변수나 함수로 변할 수 있다면 인덱스를 사용할 수 없게 된다.

</aside>

# 2.쿼리 조건

## 쿼리 조건절

`**<, ≤, >, ≥**` 의 비교 연산자는 각각 **`“$lt”, “$lte”, “$gt”, “$gte”`** 이다. 예를 들어, 18세 이상 30세 이하의 사용자를 찾으려면 다음과 같이 입력하면 된다. 

```sql
> db.users.find({"age" : {"$gte" : 18, "$lte" : 30}})
```

만약 2021년 1월 1일 이후에 등록한 사람을 찾는다면 아래와 같이 입력하자.

```sql
> start = new Date("01/01/2021")
> db.users.find({"registered" : {"$gte" : start}})
```

### $ne

일치하지 않는 도큐먼트는 찾으려면 “not equal”을 나타내는 `“$ne”`를 사용하자. 사용자 명이 “holy”가 아닌 사용자를 찾으려면 다음과 같이 입력하자.

```sql
> db.users.find({"name" :{"$ne" : "holy"}})
```

## OR 쿼리

몽고DB에서 OR 쿼리는 두 가지 방법이 있는데. **$in**과 **$or** 이다. 먼저 **$in**은 조건으로 주어진 배열에 하나라도 일치하는 도큐먼트가 있으면 찾아준다. 배열의 데이터형은 일치하지 않아도 된다. 

다음 예제는 티켓 번호가 725, 542, 390 중 하나라도 맞는 사람을 조회한다.

```sql
> db.raffle.find({"ticket_no" : {"$in" : [725, 542, 390]})
```

만약 일치하지 않는 사람을 구하려면 **“$nin”** 을 사용하면 된다.

**$in**은 “ticket_no’ **하나의 키**를 사용해서 조건에 일치하는 것을 찾았다. 만약 여러 개의 키를 통해 조건을 조회하고 싶다면 **$or**을 사용하면 된다. 다음은 ticket_no이 725 이거나 winner가 true인 도큐먼트를 찾는 쿼리다.

```sql
> db.raffle.find("$or", [{"ticket" : 725}, {"winner" : true"}])
```

### $not

**“$not”**은 메타 조건절이며 어떤 조건에도 적용할 수 있다. 아래의 예시는 **$mod**를 사용했다. $mod의 첫 번째 인수는 값과 나눌 수이고 두 번째는 일치해야 할 나머지 값을 뜻한다. 쉽게 말해서

도큐먼트의 값이 6, 7이고, `$mod : [5, 1]` 조건이라면 6에서 5를 나눈 값이 1인 도큐먼트만 조회한다. 고로 6이 조회된다. 

```sql
> db.users.find({"id_num" : {"$mod" : [5, 1]}})

// $not 위의 조건에 일치하지 않는 값을 모두 조회한다.
> db.users.find({"id_num" : {"$not" : {"$mod" : [5, 1]}}})
```

# 3. 형 특정 쿼리

## 정규 표현식

`“$regex”`는 정규식 기능을 제공한다. 다음 예제는 username이 hel이 들어간 도큐먼트를 찾는 예제다. 

```sql
// 저장 데이터
test> db.users.find()
[
  {
    _id: ObjectId("6516cd29cdce029126e9781d"),
    relationships: { friends: 32, enemies: 2 },
    username: 'joe'
  },
  {
    _id: ObjectId("65183a8febac8301af2872cb"),
    username: 'hello',
    email: 'holy@naver.com'
  }
]

// 정규식
test> db.users.find({"username" : {"$regex" : /llo/i }})
[
  {
    _id: ObjectId("65183a8febac8301af2872cb"),
    username: 'hello',
    email: 'holy@naver.com'
  }
]
```

### 배열에 쿼리하기

다음과 같이 배열이 저장되어 있으면 find를 통해서 특정 값으로 조회할 수 있다.

```sql
// 삽입
test> db.food.insertOne({"fruit" : ["apple", "banana", "peach"]})
{
  acknowledged: true,
  insertedId: ObjectId("6518431febac8301af2872cf")
}

// 조회
test> db.food.find({"fruit" : "apple"})
[
  {
    _id: ObjectId("6518431febac8301af2872cf"),
    fruit: [ 'apple', 'banana', 'peach' ]
  }
]
```

### $all 연산자

2개 이상의 배열 요소가 일치하는 배열을 찾고 싶으면 `$all` 연산자를 사용하자. 순서는 중요하지 않고 일치하는게 있으면 조회된다. 단, 배열에 모두 있어야 조회된다.

```sql
test> db.food.insertOne({"_id" : 1, "fruit" : ["apple", "banana", "peach"]})
test> db.food.insertOne({"_id" : 2, "fruit" : ["apple", "ddong", "orange"]})
test> db.food.insertOne({"_id" : 3, "fruit" : ["banana", "cherry", "apple"]})

test> db.food.find({fruit : {$all : ["apple", "banana"]}})
[
  { _id: 1, fruit: [ **'apple'**, **'banana'**, 'peach' ] },
  { _id: 3, fruit: [ **'banana'**, 'cherry', **'apple'** ] }
]
```

### $size

`$size` 연산자는 배열의 크기가 일치하는 도큐먼트를 가져온다. 

```sql
test> db.food.find({fruit : {$size : 3}})
[
  { _id: 1, fruit: [ 'apple', 'banana', 'peach' ] },
  { _id: 2, fruit: [ 'apple', 'ddong', 'orange' ] },
  { _id: 3, fruit: [ 'banana', 'cherry', 'apple' ] },
  { _id: 4, fruit: [ 'banana', 'cherry', 'apple' ] }
]
```

### $slice

`$slice` 는 JAVA의 substring처럼 특정 범위의 값을 가져오는데 쓰인다. 

예를 들어, 블로그 게시물에서 **앞의 댓글 열 개**를 반환받는다고 가정하자

```sql
> db.blog.posts.findOne(criteria, {"comments" : {"$slice" : 10}})
```

**뒤의 댓글 열 개**를 반환하고 싶으면 -10을 하면 된다.

```sql
> db.blog.posts.findOne(criteria, {"comments" : {"$slice" : -10}})
```

만약 댓글의 **40번째부터 30개**를 가져오고 싶다면 아래와 같이 작성하자.

```sql
> db.blog.posts.findOne(criteria, {"comments" : {"$slice" : [40, 30]}})
```

### 일치하는 배열 요소 반환

`$slice`는 배열의 인덱스를 알고 있을 경우 유용하다. 하지만 모른다면 어떡해야 할까? 이럴땐 `$`를 사용하여 조건을 지정하면 된다. 

```sql
> db.blog.posts.find({"comments.name" : "bob"},{"comments.$" : 1})
```