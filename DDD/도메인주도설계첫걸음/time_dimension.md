이번 장은 도메인 모델 패턴과 유사한 **이벤트 소싱 도메인 패턴**을 살펴본다. 구성 요소는 도메인 모델 패턴과 같다. 하지만 다른 것이 있는데. 바로 **애그리게이트의 상태를 저장하는 방식**이 다르다. 이벤트 소싱 도메인 패턴은 애그리게이트의 특정한 상태를 저장한다기보단 애그리게이트의 상태의 변경 사항을 전부 기록한다. 이때 사용되는 패턴이 **이벤트 소싱 패턴**이다.

# 이벤트 소싱

**이벤트 소싱**은 오직 **Insert** 만 존재한다. 하나의 데이터의 순차적인 상태 변경에 대해서 Insert 를 하고 기록된 순서를 통해 데이터의 흐름을 파악할 수 있다.

시간의 흐름에 따라 데이터의 변화를 저장하기 때문에 장애의 시작 지점이나 원하는 곳의 위치를 손쉽게 특정할 수 있다. 그렇기 때문에 이벤트 소싱 패턴이 데이터 모델에 시간 차원을 도입한다고 볼 수 있다.

 

**도메인 이벤트**

```json
{
	"lead-id" 12,
	"event-id" : 0,
	"event-type" : "lead-initialized",
	"first-name" : "Casey",
	"last-name" : ""David,
	"phone-number" : "555-2951",
	"timestamp": "2020-05-20T09:52:55.95Z"
},
{
	"lead-id" 12,
	"event-id" : 1,
	"event-type" : "contacted",
	"first-name" : "Casey",
	"last-name" : ""David,
	"phone-number" : "555-2951",
	"timestamp": "2020-05-20T12:32:08.95Z"
},
{
	"lead-id" 12,
	"event-id" : 2,
	"event-type" : "followup-set",
	"followup-on" : "2020-05-27T12:32:08.95Z",
	"timestamp": "2020-05-20T12:32:08.95Z"
},
.....

```

위의 데이터를 살펴보자. **lead-id** 라는 식별 값을 통해서 시간의 순서대로 데이터를 보여준다. **event-type**을 통해 해당 이벤트의 상세 타입을 확인할 수 있다. 처음 데이터를 살펴보면 “lead-initialized” 리드 초기화를 위한 회원 정보가 담긴 것을 볼 수 있다. 

두 번째 데이터는 회원과 처음 접촉했다는 것을 알 수 있고 세 번째 데이터는 일주일 후 다시 연락하기로 한 것을 알 수 있다.

이런 식으로 고객의 상태는 이러한 도메인 이벤트로부터 쉽게 프로젝션할 수 있다. 

<aside>
💡 **프로젝션 : 이벤트 소싱 시스템에 저장된 데이터를 다양한 읽기 모델을 적용해 원하는 시점의 데이터를 추출하는 기법**

</aside>

각 이벤트를 저장할 때, **Version** 필드를 추가하여 이벤트 발행 시 `+1` 을 해준다. 이러한 **Version** 정보를 통해서 원하는 시점으로의 **시간 여행**이 가능해진다. 현재까지 `ver 10` 이고 만약 `ver 5`까지의 데이터를 조회하고 싶다면 `ver 5` 까지의 이벤트를 그대로 적용하면 된다. 

## 검색

검색 기능을 구현한다고 가정해보자. 사용자의 연락처 정보(이름, 성, 전화번호)는 업데이트가 가능하다. 만약 영업 담당자가 기존의 사용자 연락처를 통해서 조회하기 직전, 다른 담당자가 이 사용자의 연락처를 변경하면 조회하기 힘들 수 있다. 하지만 이벤트 소싱은 이러한 정보를 모두 기록한다.

```java
public class LeadSearchModelProjection {
	public long LeadId { get; private set;}
	public HashSet<string> FirstNames { get; private set;}
	public HashSet<string> LastNames { get; private set;}
	public HashSet<PhoneNumber> PhoneNumbers {get; private set;}
	public int Version { get; private set;}

	// 초기화 이벤트
	public void Apply(LeadInitialized@event) {
		LeadId = @event.LeadId;
		FirstNames = new HashSet<string>();
		LastNames = new HashSet<String>();
		PhoneNumbers = new HashSet<PhoneNumber>();

		FirstNames.Add(@event.FirstName);
		LastNames.Add(@event.LastName);
		PhoneNumbers.Add(@event.PhoneNumber);

		// Version 0 으로 시작
		Version = 0;
	}

	// 상세 정보 변경 이벤트
	public void Apply(ContackDetailsChanged@event) {
		FirstName.Add(@event.FirstName);
		LastName.Add(@event.LastName);
		PhoneNumbers.Add(@event.PhoneNumber);

		Version += 1;
	}
	....
}
```

상세 정보 변경 이벤트인 **ContackDetailsChanged** 를 통해서 사용자의 상세 정보 변경이 이루어진다. 이때 연락처가 변경되면 PhoneNumbers 에 변경 전과 후가 모두 저장이 될 것이다. 

```java
LeadId : 12
FirstNames : ['Casey']
LastNames : ['David', 'Davis'] // 변경
PhoneNumbers : ['555-2951', '555-8101'] // 변경
Version : 6
```

## 분석

특정 부서에서 지금까지 예제로 나온 리드 데이터 중에 **후속 전화**가 예약된 개수를 구하고 싶다고 해보자. Followup 상태인 리드 데이터의 개수를 구해야 하는데. 어떻게 해야 할까?

간단하다 분석용 Projection 클래스를 하나 만들자. 그리고 Followup 이벤트가 발생할 때 같이 저장하면 된다. 

```java
// 분석 모델
public class AnalysisModelProjection {
	public long LeadId { get; private set; }
	public int Followups { get; private set; } // 후속 전화 수
	public LeadStatus Status { get; private set; }
	public int Version { get; private set; }

	public void Apply(LeadInitialized @event) {
		LeadId = @event.LeadId;
		Followups = 0;
		Status = LeadStatus.NEW_LEAD;
		Version = 0;
	}

	public void Apply(Contacted@event) {
		Version +=1;
	}

	// 후속 전화 이벤트 처리
	public void Apply(FollowupSet@event) {
		Status = LeadStatus.FOLLOWUP_SET;
		Followups += 1;
		Version += 1;
	}
	...
}
```

위의 클래스를 통해 후속 전화 이벤트가 발생하면 그 카운트를 조회할 수 있게 된다. 

```java
LeadId : 12
Followups: 1
Status: Converted
Version : 6
```

그러나 위의 로직은 HashSet과 같이 메모리에 그 데이터들을 저장하는데. 실제로는 데이터베이스에 저장된다. 이를 가능케 하는 것이 CQRS 패턴이다.

## 원천 데이터

이벤트 소싱 패턴이 작동하려면 객체 상태의 모든 **변경 사항이 이벤트로 표현되고 저장되어야 한다.** 이러한 이벤트가 시스템의 원천 데이터가 된다. 이런 원천 데이터들을 저장한 데이터 베이스를 이벤트 스토어라 명한다.

## 이벤트 스토어

이벤트 스토어는 오직 추가만 가능한 저장소이다. 수정, 삭제는 이루어져서는 안된다. 아래의 인터페이스를 살펴보자. 

```java
interface IEventStore {
	IEnumerable<Event> Fetch(Guid instanceId); // 모든 이벤트 조회
	void Append(Guid instanceId, Event[] newEvents, int expectedVersion); // 이벤트 추가
}
```

Append 메서드의 마지막 매개변수의 `expectedVersion` 을 주목하자. 낙관적 동시성 제어를 위해 미리 조회한 예상 버전을 넣어줘야 한다. 만약 일치하지 않으면 동시성 예외가 발생된다. 

# 이벤트 소싱 도메인 모델

기존의 도메인 모델은 에그리게이트의 상태를 표현하고 비즈니스 로직이 실행됨에 따라 **도메인 이벤트**를 발생시킨다. 도메인 이벤트는 외부로 전달되거나 내부로 전달됨으로써 또다른 비즈니스 로직이 진행된다. 이와 반대로 **이벤트 소싱 도메인 모델**은 ***에그리게이트의 수명 주기를 나타내기 위해 독점적으로 도메인 이벤트를 사용한다***. 즉, 도메인 이벤트의 모든 상태 변화를 이벤트로 나타내고 기록한다.

일반적으로 이벤트 소싱 도메인 모델의 순서는 다음과 같다. 

1. **에그리게이트의 도메인 이벤트를 로드한다.(모든 도메인 이벤트 혹은 특정 시점의 도메인 이벤트를 가져온다.)**
2. **이벤트를 비즈니스 의사결정을 내리는데 사용할 수 있도록 프로젝션한다.**
3. **에그리게이트의 명령을 실행하여 비즈니스 로직을 실행하고 새로운 도메인 이벤트를 생성한다.**
4. **생성된 도메인 이벤트를 이벤트 스토어에 저장한다.** 

다음의 예제는 이벤트 소싱 도메인 모델의 순서에 대한 소스 코드다.

```java
public class TicketAPI {
	private ITicketsRepository _ticketsRepository;
	...

	public void RequestEscalation(TicketId id, EscalationReason reason) {
		var events = _ticketsRepository.LoadEvents(id); // 1. 에그리게이트의 도메인 이벤트를 로드한다.
		var ticket = new Ticket(events); // 2. 이벤트를 프로젝션한다.
		var originalVersion = ticket.Version; // 낙관적 락을 위한 현재 버전 조회
		var cmd = new RequestEscalation(reason);
		ticket.Execute(cmd); // 3. 비즈니스 로직을 실행하여 새로운 도메인 이벤트를 내부적으로 생성한다.
		_ticketsRepository.CommitChanges(tickets, originalVersion); // 4. 생성된 도메인 이벤트를 이벤트 스토어에 저장한다. 
	}
	
}
```

**이벤트 프로젝션** 부분인 `new Ticket(events)` 의 내부를 살펴보자. 어떻게 프로젝션하는 걸까? 

```java
public class Ticket {
	private List<DomainEvent> _domainEvents = new List<DomainEvent>();
	private TicketState _state;

	public Ticket(IEnumerable<IDomainEvents events) {
		_state = new TicketState();  // 이벤트의 결과물
		foreach(var e in events) {
			AppendEvent(e); // 이벤트 실행
		}
	}

	private void AppendEvent(IDomainEvent@event) {
		_domainEvents.Append(@event); // 이벤트 저장
		((dynamic) state).Apply((dynamic) @event); // 이벤트의 상태는 각각 다르기 때문에 동적으로 실행
	} 
}
```

로드된 **events** 는 Ticket의 생성자의 매개변수로 들어온다. 그리고 가장 먼저 티겟의 상태를 나타내는 TicketState의 인스턴스를 생성하여 기존 필드에 담아준다. 

그리고 **AppendEvent** 를 통해 각 이벤트를 순서대로 진행시킨다. 

AppendEvent 를 보면 로드한 이벤트를 새로 저장하고 각각의 이벤트를 동적으로 호출한 것을 볼 수 있다. 이벤트들은 각 상황에 맞게 다르게 만들어질 수 있기 때문에 이벤트에 맞게 동적으로 호출된다. 아래의 Apply 메서드를 살펴보면 각 매개변수 별로 오버로딩 된 것을 확인할 수 있다.

```java
public class TicketState {
	public TicketId id { get; private set; } 
	public int Version { get; private set; }
	public bool IsEscalated { get; private set; }

	public void Apply(TicketInitialize@event) {
		Id = @event.Id;
		Version = 0;
		IsEscalated = false;
	}

	public void Apply(TicketEscalated@event) {
		IsEscalated = true;
		Version += 1;
	}
}
```

## 장점

기존의 현재 상태만을 기록하는 도메인 모델과는 달리 이벤트 소싱 도메인 모델은 변경사항의 모든 과정을 기록하는 만큼 모델링하는 비용이 상대적으로 더 크다. 그럼에도 불구하고 이벤트 소싱 도메인 모델을 사용하는 것에 대한 장점이 많다. 몇 가지 살펴보자. 

### 시간여행

도메인 이벤트를 이용하여 에그리게이트의 현재 상태를 재구성할 수 있는 것처럼 이벤트 소싱 도메인 모델은 원하는 특정 시점의 상태로 되돌릴 수 있다. 이것은 큰 장점이 된다. 

저장된 기록을 봄으로써 시스템을 동작할 수 있을 뿐아니라 시스템의 의사결정을 검사할 수 있으며 분석과 검사를 통해 비즈니스 로직을 최적화 할 수 있다.

또 버그나 장애가 났을 때의 상황으로 돌아가 재현할 수있게 된다. 

### 심오한 통찰력

이벤트 소싱 도메인 모델은 유연한 모델을 제공한다. 시스템의 상태와 동작을 감시하기 위해 특정한 프로젝션을 위한 이벤트 모델을 구성할 수도 있다. 

### 감사 로그

에그리게이트의 상태 변화에 대해 강력하게 일관된 **감사 로그**(audit log)가 필요한 경우가 있다. 금융권의 자금 흐름 추적에 대해서 로그를 항시 저장하는 것이 이것인데. 이때 즉각적으로 제공이 가능하다.

### 고급 낙관적 동시성 제어

고급 낙관적 동시성 모델은 읽기 데이터가 기록되는 동안 다른 프로세스를 통해서 덮어쓰여지는 경우 예외를 발생시킨다. 또 현재 이벤트를 가져와 새 이벤트를 만들고 기록하는 과정에 충돌이 발생하면 예외를 발생시키거나 문제가 없으면 계속 진행하도록 비즈니스 도메인 주도 의사결정을 내릴 수 있다. 

## 단점

### 학습 곡선

이벤트 소싱 도메인 패턴이 데이터 관리하는 기술이 기존과는 확연히 차이가 있다는 명백한 단점이 있다. 이는 높은 학습 곡선을 요구하게 된다. 온전히 사용하기 위해선 팀교육과 새로운 사고방식을 주입해야 한다.

### 모델의 진화

이벤트 소싱 모델을 발전시키는 것은 어려울 수 있다. 이벤트 소싱의 정의를 엄밀하게 따지면 한번 정한 이벤트는 변경이 어렵다.  가능은 하겠지만 어렵다

### 아키텍처 복잡성

이벤트 소싱을 구현하면 수많은 아키텍처의 ‘유동적인 부분’이 도입되어 전체 설계가 더 복잡해진다. 즉 아키텍처가 변화에 적응하고 발전할 수 있는 부분이 어려워진다. 예를 들어, 기술의 진화, 구성요소의 다양화, 외부 환경 변화 등에 따라서 아키텍처가 새로운 요구사항을 수용하고, 최적화를 위해 재구성하고, 개선을 위해 발전하는 과정을 거쳐야 하는데. 이벤트 소싱은 수정하거나 추가하기 어렵다.