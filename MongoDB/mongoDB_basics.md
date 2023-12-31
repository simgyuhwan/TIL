몽고DB의 기본 특징은 아래와 같다. 

- 몽고DB의 기본 단위는 **도큐먼트**(Document)이다.
- **컬렉션(Collection)**은 동적 스키마가 있는 테이블과 같다. : 스키마가 없고 형식이 자유롭다.
- 몽고DB의 단일 인스턴스는 여러 개의 독립적인 데이터베이스를 가질 수 있다.
- 모든 도큐먼트는 컬랙션 내에서 특수키인 **“_id”**를 가진다.
- 몽고 쉘(The mongo Shell)

# 1. 도큐먼트

도큐먼트 표현 방식은 프로그래밍 언어마다 다르겠지만 일반적으로 맵, 해시, 딕셔너리로 사용된다. 기본 특징으로는 아래와 같다. 

- **Key는 `\0(null 문자)` 를 포함하면 안된다. \0은 키의 끝을 의미한다.**
- **. 과 $는 예약어이니 사용을 자제하자.**
- **대소문자를 구분한다.**
- **타입을 구분한다.**
- **키는 중복될 수 없다.**

# 2. 컬렉션

**컬렉션**은 도큐먼트의 모음이다. 

## 2.1 동적 스키마

컬렉션 내부의 스키마는 자유롭다. 어떠한 형식으로 구성되어도 상관이 없으며 일치하지 않아도 된다. 그러면 컬렉션 하나에 다 넣으면 될거 같은데. 왜 하나 이상의 컬렉션이 필요할까?

- 컬렉션에 다양한 종류가 들어있다고 해보자. 여기서 내가 원하는 종류를 제거하려면 상당히 힘들어진다.
- 컬렉션을 분류하면서 **지역성(locality)**에도 좋다.
- **인덱스는 컬렉션별로 정의**하는데. 같은 유형의 도큐먼트를 하나의 컬렉션에 넣음으로써 컬렉션을 **효율적으로 인덱싱**할 수 있다.

결론적으로 같은 유형별로 컬렉션에 모아두고 인덱싱을 거는게 좋다.

## 2.2 네이밍

컬렉션은 이름으로 식별되는데. 다음과 같은 특징을 가진다.

- 빈 문자열(`””`)은 유효한 컬렉션이 아니다.
- `\0(null 문자)`은 컬렉션명의 끝을 나타내는 문자이므로 컬렉션명으로 사용할 수 없다.
- `system.` 으로 시작하는 컬렉션명은 시스템 컬렉션에서 사용하는 예약어이므로 사용할 수 없다.
- 예약어 `$`는 사용할 수 없다.

## 2.3 서브컬렉션

서브컬렉션은 이름에 ‘**.**’ 표시하면서 체계화한다. 예를 들어, `blog`, `blog.posts`, `blog.authors` 가 있는데. 그렇다고 다른 컬렉션과 다르지않다. 그저 하나의 컬렉션일 뿐이다. 다만 좀 더 체계화할 수 있으며 서브컬렉션별로 관리할 수 있는 기능을 제공해준다.

## 2.4 데이터베이스

컬렉션을 그룹화한 것이 데이터베이스이다. 몽고DB의 단일 인스턴스는 여러 데이터베이스를 호스팅할 수 있다. 데이터베이스는 컬렉션과 이름 짓는게 비슷한데. 다음과 같다.

- **빈 문자열(””)은 사용할 수 없다.**
- **\0 , / ,\, . 등과 같은 특수문자는 사용할 수없다.**
- **대소문자를 구분한다.**
- **데이터베이스의 이름은 최대 64바이트이다.**

다음은 이미 예약된 데이터베이스의 이름이다.

- admin : 인증과 권한 부여 역할을 한다.
- local : 단일 서버에 대한 데이터를 저장한다.
- config : 샤딩된 몽고DB 클러스터는 config 데이터베이스를 사용해 각 샤드의 정보를 저장한다.