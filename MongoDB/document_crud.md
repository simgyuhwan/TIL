# 삽입

## 1. 단일 삽입

컬렉션에 도큐먼트를 삽입하려면 `insertOne` 메서드를 사용한다.

```sql
test> db.movies.insertOne({"title":"Stand by Me"})
{
  acknowledged: true,
  insertedId: ObjectId("6516c437cdce029126e97819")
}
```

## 2. 대량 삽입

여러 도큐먼트를 컬렉션에 삽입하려면 **insertMany** 메서드를 사용한다. 한 번에 넣을 수 있는 용량은 **48MB**으로 만약 이 용량을 넘어서면 48MB 크기로 여러 개로 분할하여 등록한다.

```sql
test> db.movies.insertMany([{"title" : "Ghostbuster"}, {"title" : "E.T."}, {"title" : "Blade Runner"}]);
{
  acknowledged: true,
  insertedIds: {
    '0': ObjectId("6516c4f5cdce029126e9781a"),
    '1': ObjectId("6516c4f5cdce029126e9781b"),
    '2': ObjectId("6516c4f5cdce029126e9781c")
  }
}

test> db.movies.find()
[
  { _id: ObjectId("6516c4f5cdce029126e9781a"), title: 'Ghostbuster' },
  { _id: ObjectId("6516c4f5cdce029126e9781b"), title: 'E.T.' },
  { _id: ObjectId("6516c4f5cdce029126e9781c"), title: 'Blade Runner' }
]
```

**insertMany**에서는 두 번째 매개변수로 **“ordered”** 를 줄 수 있는데.  기본 값은 **true**이다. true로 지정되면 처음 insertMany에 작성했던 배열의 순서대로 등록된다. false로 설정하면 배열의 순서와 상관없이 임의로 저장된다. 

**ordered의 값에 따라 중간에 장애가 발생했을 때의 처리 방법**이 달라진다.

**true**일 때, 중간 값이 잘못됐다면 정상적으로 넣은 것까지만 저장된다. 

```sql
test> db.movies.insertMany(
														[
																{"_id" : 0, "title" : "Top Gun"}, 
																{"_id" : 1, "title" : "Back to the Future"},
																{"_id" : 1, "title" : "Gremlins"}
														]
													)
Uncaught:
MongoBulkWriteError: E11000 duplicate key error collection: test.movies index: _id_ dup key: { _id: 1 }
Result: BulkWriteResult {
  insertedCount: 2,
  matchedCount: 0,
  modifiedCount: 0,
  deletedCount: 0,
  upsertedCount: 0,
  upsertedIds: {},
  insertedIds: { '0': 0, '1': 1, '2': 1 }
}
Write Errors: [
  WriteError {
    err: {
      index: 2,
      code: 11000,
      errmsg: 'E11000 duplicate key error collection: test.movies index: _id_ dup key: { _id: 1 }',
      errInfo: undefined,
      op: { _id: 1, title: 'Gremlins' }
    }
  }
]

// 두 개만 저장됨
test> db.movies.find()
[
  { _id: 0, title: 'Top Gun' },
  { _id: 1, title: 'Back to the Future' }
]
```

false 로 설정하면 장애와 상관없이 장애난 부분을 제외하고 저장된다. 

```sql
test> db.movies.insertMany(
														[
															{"_id" : 5, "title" : "First"},
															{"_id" : 6, "title" : "Second"},
															**{"_id" : 6, "title" : "Third"},  // 저장 안됨**
															{"_id" : 7, "title": "Fourth"}
														], {"ordered": false})

test> db.movies.find()
[
  { _id: 0, title: 'Top Gun' },
  { _id: 1, title: 'Back to the Future' },
  { _id: 3, title: 'First' },
  { _id: 4, title: 'Second' },
  { _id: 5, title: 'First' },
  { _id: 6, title: 'Second' },
  { _id: 7, title: 'Fourth' }
]
```

## 3. 삽입 유효성 검사

몽고DB는 삽입된 데이터데 최소한의 검사를 수행한다.

- **_id 필드 확인** : 만약 입력 값에 id가 존재하지 않으면 추가해준다.
- **16MB의 크기 제한** : 도큐먼트는 16MB보다 작아야 한다. 그 이유는 단일 도큐먼트가 과도한 양의 RAM, 대역폭을 사용하지 않도록 제한하여 일관된 성능 보장한다.

# 삭제

데이터베이스의 데이터를 삭제하기 위한 명령어는 두 가지이다. **deleteOne, deleteMany** 

제거할 도큐먼트의 id를 명시하면 삭제가 진행된다. 컬렉션에서 id는 고유하기 때문에 id만 사용했는데. id가 아닌 다른 값도 사용이 가능하다.

```sql
test> db.movies.find()
[
  { _id: 0, title: 'Top Gun' },
  { _id: 1, title: 'Back to the Future' },
  { _id: 3, title: 'First' },
  { _id: 4, title: 'Second' },
  { _id: 5, title: 'First' },
  { _id: 6, title: 'Second' },
  { _id: 7, title: 'Fourth' }
]

test> db.movies.deleteOne({"_id": 0})
{ acknowledged: true, deletedCount: 1 }

test> db.movies.find()
[
  { _id: 1, title: 'Back to the Future' },
  { _id: 3, title: 'First' },
  { _id: 4, title: 'Second' },
  { _id: 5, title: 'First' },
  { _id: 6, title: 'Second' },
  { _id: 7, title: 'Fourth' }
]
```

다음과 같이 id가 아니라 다른 필드값과 일치하는 도큐먼트를 삭제할 수 있다. 필터에 일치하는 가장 처음에 발견한 값을 삭제하기 때문에 잘 생각하고 써야 한다.

```sql
test> db.movies.deleteOne({"title" : "First"})
{ acknowledged: true, deletedCount: 1 }
```

만약 필터에 일치하는 모든 도큐먼트를 삭제하려면 **deleteMany**를 사용하자.

```sql
db.moives.deleteMany({"year" : 1984});
```

## 1. 전체 삭제

전체 삭제를 위한 방법은 **deleteMany**와 **drop**을 사용하면 된다. 단, 한 번 제거하면 영원히 사라지니 주의해서 사용하자. 두 방법중 하나를 택하자.

```sql
> db.movies.deleteMany({})
> db.movies.drop()
```

# 갱신

도큐먼트를 갱신하는 메서드는 **updateOne, updateMany, replaceOne** 이다. 갱신은 원자적으로 이루어지는데. 동시에 여러 건이 발생해도 제일 **마지막 갱신이 적용**된다. 

## 1. 도큐먼트 치환

예시는 users 컬렉션을 생성하고 안에 도큐먼트를 삽입했다. 

```sql
test> db.users.insertOne({"name" : "joe", "friends" : 32, "enemies" : 2})
{
  acknowledged: true,
  insertedId: ObjectId("6516cd29cdce029126e9781d")
}

test> db.users.find()
[
  {
    _id: ObjectId("6516cd29cdce029126e9781d"),
    name: 'joe',
    friends: 32,
    enemies: 2
  }
]
```

이제 도큐먼트의 스키마를 새롭게 **변경**하고 싶다고 해보자. **friends**와 **enemies**를 **relationships라는 서브도큐먼트로 옮겨보자**.

```sql
test> joe.relationships = {"friends" : joe.friends, "enemies" : joe.enemies};
{ friends: 32, enemies: 2 }
test> joe
{
  _id: ObjectId("6516cd29cdce029126e9781d"),
  name: 'joe',
  friends: 32,
  enemies: 2,
  relationships: { friends: 32, enemies: 2 }
}
test> delete joe.friends
true
test> delete joe.enemies
true
test> delete joe.name
true
test> joe.username = "joe"

// 변경됨
test> db.users.replaceOne({"name" : "joe"}, joe)
test> db.users.find()
[
  {
    _id: ObjectId("6516cd29cdce029126e9781d"),
    relationships: { friends: 32, enemies: 2 },
    username: 'joe'
  }
]
```

## 2. 갱신 연산자

일반적으로 도큐먼트의 특정 필드만 갱신하는 경우가 많다. 예를 들어, url과 조회수를 도큐먼트로 저장했을 때, url의 조회수 부분만 증가시키기 위해 **$inc** 연산자를 사용할 수 있다.

### (1) $inc

```sql
// pageviews 52개
test> db.analytics.find()
[
  {
    _id: ObjectId("6516d618cdce029126e9781f"),
    url: 'www.example.com',
    pageviews: 52
  }
]

// $inc 연산자로 특정 부분 증가
test> db.analytics.updateOne({"url" : "www.example.com"}, {"$inc" : {"pageviews" : 8}})

// 확인
test> db.analytics.find()
[
  {
    _id: ObjectId("6516d618cdce029126e9781f"),
    url: 'www.example.com',
    pageviews: 60
  }
]
```

### (2) $set

**$set** 은 필드가 존재하면 값을 변경해주고 존재하지 않으면 새 필드를 생성해준다. 여러 데이터형으로 변경할 수 있고 삭제하고 싶으면 **$unset** 을 사용하자. 만약 **$set 연산자를 사용하지 않으면 도큐먼트 자체가 바뀔 수 있으니 주의하자.**

```sql
test> db.analytics.find()
[
  {
    _id: ObjectId("6516d618cdce029126e9781f"),
    url: 'www.example.com',
    pageviews: 60
  }
]

// $set
test> db.analytics.updateOne({"_id" : ObjectId("6516d618cdce029126e9781f")}, {"$set" : {"humuns" : 50}})
test> db.analytics.find()
[
  {
    _id: ObjectId("6516d618cdce029126e9781f"),
    url: 'www.example.com',
    pageviews: 60,
    humuns: 50
  }
]

// $unset
test> db.analytics.updateOne({"_id" : ObjectId("6516d618cdce029126e9781f")}, {"$unset" : {"humuns" : 1}})
```

### (3) $inc 증가와 감소

**$inc** 연산은 **필드가 없으면 추가**를 하고 있으면 **값을 증가(감소)**시킨다. 단, **숫자와 관련된 데이터형**만 사용이 가능하다. 문자열에 사용하면 오류 메시지가 발생한다.

```sql
test> db.games.find()
[
  {
    _id: ObjectId("6516debdcdce029126e97820"),
    game: 'pinball',
    user: 'joe'
  }
]

// $inc
test> db.games.updateOne({"game" : "pinball", "user" : "joe"}, {"$inc" : {"score" : 50}})
test> db.games.findOne()
{
  _id: ObjectId("6516debdcdce029126e97820"),
  game: 'pinball',
  user: 'joe',
  score: 50   <------- 새로 추가
}

// $inc 
test> db.games.updateOne({"game" : "pinball", "user" : "joe"}, {"$inc" : {"score" : 10000}})
test> db.games.findOne()
{
  _id: ObjectId("6516debdcdce029126e97820"),
  game: 'pinball',
  user: 'joe',
  score: 10050  <------- 증가
}
```

### (4) $push 배열 연산자

**$push** 연산자는 배열이 존재하면 끝에 요소를 추가하고 없으면 생성해준다. 다음은 블로그의 댓글을 배열로 추가하는 예시이다.

```sql
test> db.blog.posts.findOne()
{
  _id: ObjectId("6516e00dcdce029126e97821"),
  title: '해리포터',
  content: '판타지'
}

// 댓글 추가
test> db.blog.posts.updateOne({"title" : "해리포터"}, {"$push" : {"contents" : {"name" : "joe", "email" : "example@example.com"}}})
test> db.blog.posts.updateOne({"title" : "해리포터"}, {"$push" : {"contents" : {"name" : "bob", "email" : "example2@example.com"}}})

// 배열 생성 및 추가
test> db.blog.posts.findOne()
{
  _id: ObjectId("6516e00dcdce029126e97821"),
  title: '해리포터',
  content: '판타지',
  contents: [
    { name: 'joe', email: 'example@example.com' },
    { name: 'bob', email: 'example2@example.com' }
  ]
}
```

$push와 함께 쓸 수 있는 연산자가 $each, $slice, $sort 이다. 

- $each : 여러 개를 한 번에 넣을 수 있다.
- $slice : 최대 개수를 지정한다. -10으로 지정하면 10개까지 유지된다.
- $sort : 정렬할 필드 값을 지정한다.

### (5) $ne (Not Equal)

**$ne** 연산자는 해당 값과 일치하지 않는 값을 찾는 연산자이다. 예를 들어, 다음과 같은 예시는 name이 “Alice” 가 아닌 도큐먼트를 찾는다. 

`db.users.find({name: {$ne: “Alice”}})`

또 아래의 예시는 name이 “misa”가 아닌 사람을 찾아서 해당 이름을 추가한다.

`db.papers.updateOne({”name” : {”$ne” : “misa”}}, {$push : {”name” : “holy”}})`

### (6) $addToSet (중복 없이 추가)

배열에 추가할 때, 중복없이 추가하고 싶을 수 있다. 그럴 땐 **$addToSet** 연산자를 사용하자.

```sql
> db.users.updateOne({"_id" : ObjectId("6516e00dcdce029126e97821")},{"$addToSet" : {"emails" : "joe@gmail.com"}})
```

만약 유니크한 이메일을 여러 개 추가하고 싶으면 **$each** 와 **$addToSet**을 결합해서 사용해보자.

```sql
> db.users.updateOne({"_id" : ObjectId("6516e00dcdce029126e97821")}, 
{"addToSet" : {"emails" : {"$each" : ["test@naver.com", "test2@naver.com", "test3@naver.com"]}}})
```

### (7) $pop, $pull 요소 제거하기

**$pop**을 사용하면 배열의 앞이나 뒤를 제거하는데. `{”$pop” : {”key”: 1}}` 을 하면 배열의 마지막 요소를 제거한다. `{”$pop” : {”key” : -1}}` 은 배열의 처음부터 요소를 제거한다.

**$pull** 연산자를 사용하면 지정된 조건에 맞는 요소를 제거하는데. **일치하는 모든 요소를 제거**하기 때문에 주의해서 사용하자.

`db.lists.updateOne({}, {”$pull” : {”todo” : “laundry”}})`

todo의 값이 laundry인 요소를 모두 제거한다.

### (8) 배열 위치 기반 변경

게시글의 댓글이 여러 개라고 가정을 해보자. 댓글은 배열로 저장되어 있고 각 위치는 인덱스로 구분된다. 만약 첫 번째 댓글(0번 인덱스)의 투표수를 증가시키려면 다음과 같이 사용한다.

```sql
> db.blog.updateOne({"post" : post_id}, {"$inc" : {"comments.0.votes" : 1}})
```

그런데 이런 특정 요소의 인덱스 위치를 찾기가 쉽지가 않다. 그래서 배열의 요소를 찾기 위한 연산자 **“$”**를 제공한다. 다음은 댓글의 작성자의 이름을 변경하는 예시이다. 

```sql
> db.blog.updateOne({"comments.author" : "John"}, {"$set" : {"comments.$.author" : "Jim"}})
```