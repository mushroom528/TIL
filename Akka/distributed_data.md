# Distributed Data
- Akka 클러스터 내의 노드 간 데이터 공유가 필요할 때 사용하는 기능
- Distributed Data 저장소는 key-value 구조로 되어있고, key는 데이터의 타입 정보가 포함된 고유 식별자
- value는 CRDTs(Conflict Free Replicated Data Types)로 되어있음
- 모든 데이터는 메모리에 저장됨
## CRDTs(Conflict Free Replicated Data Types)
- 분산 시스템에서 데이터 일관성을 해결하기 위해 Akka에서 제공해주는 데이터 구조
- 제공해주는 타입 말고도 사용자가 직접 정의하여 타입을 구현할 수 있음
	- 직접 구현하는 경우 `AbstractReplicatedData`를 구현해야함
- 복제된 데이터 타입의 충돌을 해결하기 위해 타입마다 `monotonic merge function`를 통해 충돌을 해결
	- 결론적으로 같은 결과를 보장하기 위해 사용
### Counter
- `GCounter`: "grow only counter" -> increment만 가능한 counter, decrement는 지원하지 않음
	- 병합: Vetor clock과 유사한 방식, 각 노드마다 하나의 카운터 값을 유지하고 각 노드의 카운터 값의 합계가 병합된 값이 된다.
	- ex) 노드A, 노드B, 노드C이 있다고 가정
	  1. increment 요청 -> 노드 A가 처리 -> 노드 A 내부의 카운터 값은 1
	  2. 여러 요청에 대해 세 개의 노드가 분산 처리하고, 처리 결과는 각 노드 내부에서 관리
	    -> 노드 A: 3, 노드 B: 4, 노드 C: 1
	  3. 노드 A에서 읽기 요청 발생
		-> 노드 A는 자기 자신의 값만 참조하는 것이 아닌 다른 노드의 카운터 값도 고려
		-> 최종 응답: 3 + 4 + 1 = 8 
- `PNCounter`: 증가와 감소가 필요한 경우 사용
	- 병합: 노드 내부에 증가 카운터, 감소 카운터를 관리하여 읽기 요청 시 이를 참조하여 응답
### Sets
- `GSet`: grow-only set, 요소를 추가할 수 있기만 하고 remove는 불가한 데이터 타입(요소는 직렬화 가능한 모든 타입이 될 수 있음)
	- 병합: 노드 내부에 가지고 있는 Set의 합집합
		- 노드 A: {1, 2, 3}
		- 노드 B: {2, 3, 4}
		- `merged GSet`: {1, 2, 3, 4}
	- 요소가 한번 추가 되면 삭제될 수 없으므로, 복잡한 동기화 메커니즘 없이도 일관성을 유지할 수 있음
- `ORSet`: 요소 삭제하는 기능이 추가된 `Set`
	- 병합: 각 요소는 버전 정보도 같이 저장하고, 병합 시 해당 정보를 기반으로 병합한다.
### 그 외
- 위에서 명시한 데이터 타입 말고도 `Set`, `Counter`를 기반으로 구현한 `Map`
	- `ORMap`
	- `ORMultiMap`
	- `PNCounterMap`
	- `LWWMap`
	- 위 데이터 타입은 `delta-CRDT`를 지원함
- `Flag`
	- `true` or `false` 
	- 병합 시 `false` 보다 `true`를 우선함
- `Registers`
	- `LWWRegister`: 마지막 write가 우선(write 시 타임 스탬프 정보도 같이 저장)
	- 모든 직렬화가 가능한 값을 가지고 있을 수 있음
```java
final SelfUniqueAddress node = DistributedData.get(system).selfUniqueAddress();
final LWWRegister<String> r1 = LWWRegister.create(node, "Hello");
final LWWRegister<String> r2 = r1.withValue(node, "Hi");
System.out.println(r1.value() + " by " + r1.updatedBy() + " at " + r1.timestamp());
```
## 일관성
- 읽기 및 쓰기 작업 시 요청에 대해 성공적으로 응답 해야하는 노드 수를 지정함
- `WriteAll`, `ReadAll`은 가장 강력한 일관성을 보장하지만, 가장 느리고 가용성이 낮음
	- 하나의 노드라도 읽기 요청 응답에 실패하면 값을 받아올 수 없음
### 쓰기 일관성
- `writeLocal`: 로컬 복제본(노드)에만 즉시 write하고, 이후에 가십 프로토콜에 의해 다른 복제본으로 전파
	- `Replicator.writeLocal()`
- `WriteTo(n)`: 로컬 복제본을 포함하여 적어도 n개의 복제본에 write 작업 수행
	- `new Replicator.WriteTo(3, Duration.ofSeconds(3))`: write할 복제본의 수 3, timeout 3초 -> 3초 넘으면 write 실패로 간주함
- `WriteMajority`: 대다수의 복제본에 write 작업 수행(대다수: N/2 + 1, N: 클러스터 내의 복제본의 수)
- `WriteMajorityPlus`: `WriteMajority`와 유사하지만, 매개변수(`minCap`)를 통해 추가 계수를 설정할 수 있음
	- `new Replicator.WriteMajority(Duration.ofSeconds(3), 3)`
- `WriteAll`:즉시 모든 복제본에 write 작업 수행
	- `new Replicator.WriteAll(Duration.ofSeconds(3))` 


### 읽기 일관성
쓰기와 읽기 일관성이 `xxxMajority` 로 설정되어 있다면, 아래의 수식을 통해 Read가 항상 최근의 Write를 반영하도록 할 수 있음
> **(nodes_written + nodes_read) > N**

N: 클러스터의 총 노드 수 또는 Replicator 역할을 수행하는 노드의 수
**지원하는 읽기 일관성**
- `readLocal`: 값은 항상 로컬 복제본에서만 읽음
	- `Replicator.readLocal()`
- `ReadFrom(n)`: 값은 n개의 복제본에서 읽고 병합
	- `new Replicator.ReadFrom(3, Duration.ofSeconds(3))`: 3개의 복제본에서 읽고 병합, timeout 초과 시 읽기 실패
- `ReadMajority`: 값은 대다수의 복제본에서 읽고 병합(대다수: N/2 + 1, N: 클러스터 내의 복제본의 수)
- `ReadMajorityPlus`: `ReadMajority` 와 유사하지만, 매개변수(`minCap`)를 통해 추가 계수를 설정할 수 있음
	- `new Replicator.ReadMajority(Duration.ofSeconds(3), 3)`
- `ReadAll`: 값은 클러스터 내의 모든 노드에서 읽고 병합
	- `new Replicator.ReadAll(Duration.ofSeconds(3))`
병합(merge): CRDTs 에 따라 `monotonic merge function`에 의해서 merge

> `minCap`: 소규모 클러스터의 안정성을 높이기 위해 사용하는 매개변수
## 코드(ORSet)
```java
public class ORSetActor extends AbstractBehavior<ORSetActor.Command> {  
  
    public interface Command {  
    }  
    public interface InternalCommand extends Command {  
    }  
    record InternalGetResponse(Replicator.GetResponse<ORSet<String>> rsp) implements InternalCommand {  
    }  
    record InternalUpdateResponse(Replicator.UpdateResponse<ORSet<String>> rsp) implements InternalCommand {  
    }  
    public record AddElement(String element) implements Command {  
    }  
    public record RemoveElement(String element) implements Command {  
    }  
    public record GetElements() implements Command {  
    }  
    private final SelfUniqueAddress node;  
    private final ReplicatorMessageAdapter<Command, ORSet<String>> replicatorAdapter;  
    private final ORSetKey<String> dataKey;  
  
    public ORSetActor(ActorContext<Command> context, ReplicatorMessageAdapter<Command, ORSet<String>> replicatorAdapter, ORSetKey<String> dataKey) {  
        super(context);  
        this.node = DistributedData.get(this.getContext().getSystem()).selfUniqueAddress();  
        this.replicatorAdapter = replicatorAdapter;  
        this.dataKey = dataKey;  
    }  
  
    public static Behavior<Command> create(ORSetKey<String> dataKey) {  
        return Behaviors.setup(ctx ->  
                DistributedData.withReplicatorMessageAdapter(  
                        (ReplicatorMessageAdapter<Command, ORSet<String>> adapter) ->  
                                new ORSetActor(ctx, adapter, dataKey)));  
    }  
  
    @Override  
    public Receive<Command> createReceive() {  
        return newReceiveBuilder()  
                .onMessage(AddElement.class, this::onAddElement)  
                .onMessage(RemoveElement.class, this::onRemoveElement)  
                .onMessage(GetElements.class, this::onGetElements)  
                .onMessage(InternalGetResponse.class, this::onInternalGetResponse)  
                .onMessage(InternalUpdateResponse.class, msg -> {  
                    getContext().getLog().info("Received InternalGetResponse message: {}", msg.rsp());  
                    return Behaviors.same();  
                })  
                .build();  
    }  
  
    private Behavior<Command> onAddElement(AddElement command) {  
        getContext().getLog().info("add element: " + command.element);  
        replicatorAdapter.askUpdate(  
                replyTo -> new Replicator.Update<>(  
                        dataKey,  
                        ORSet.empty(),  
                        Replicator.writeLocal(),  
                        replyTo,  
                        curr -> curr.add(node, command.element))  
                , InternalUpdateResponse::new  
        );  
        return this;  
    }  
  
    private Behavior<Command> onRemoveElement(RemoveElement command) {  
        replicatorAdapter.askUpdate(  
                replyTo -> new Replicator.Update<>(  
                        dataKey,  
                        ORSet.empty(),  
                        Replicator.writeLocal(),  
                        replyTo, curr -> curr.remove(node, command.element))  
                , timeout -> new RemoveElement(command.element)  
        );  
        return this;  
    }  
  
    private Behavior<Command> onGetElements(GetElements command) {  
        replicatorAdapter.askGet(  
                replyTo -> new Replicator.Get<>(dataKey, Replicator.readLocal(), replyTo),  
                InternalGetResponse::new  
        );  
        return this;  
    }  
  
    private Behavior<Command> onInternalGetResponse(InternalGetResponse command) {  
        if (command.rsp instanceof Replicator.GetSuccess<ORSet<String>> success) {  
            ORSet<String> result = success.get(dataKey);  
            Set<String> elements = result.getElements();  
            getContext().getLog().info("Current elements in the ORSet: " + elements);  
            return this;  
        }        return Behaviors.unhandled();  
    }  
}
```
- `ORSet` 타입을 사용하고, `ORSet` 내부의 `Command` 타입의 메세지를 처리할 수 있는 `Actor`
- 메세지 종류
	- `Command`: `AddElement`, `RemoveElement`, `GetElements`
	- `InternalCommand`: `Actor` 내부에서만 사용하는 메세지(주로 콜백 용도로)
		- `InternalGetResponse`, `InternalUpdateResponse`
- `SelfUniqueAddress`: 클러스터 내의 각 노드에 대한 고유한 주소
- `ReplicatorMessageAdapter<Command, ORSet<String>>`: 특정 명령을 받고 `ORSet` 유형의 데이터를 처리할 때 사용
- `ORSetKey<String>`: Distributed Data에서 `ORSet`에 접근하기 위한 키
### Update
데이터를 수정하고 복제하기 위해서 `Replicator`에 `Replicator.Update` 메세지를 보냄
```java
private Behavior<Command> onAddElement(AddElement command) {  
    getContext().getLog().info("add element: " + command.element);  
    replicatorAdapter.askUpdate(  
            replyTo -> new Replicator.Update<>(  
                    dataKey,  
                    ORSet.empty(),  
                    Replicator.writeLocal(),  
                    replyTo,  
                    curr -> curr.add(node, command.element))  
            , InternalUpdateResponse::new  
    );  
    return this;  
}
```
- 필요한 값
	- 업데이트하려는 Key
	- Replicator가 해당 키에 대한 값이 없는 경우 사용할 데이터
	- 업데이트에 대한 write 일관성 수준
	- 업데이트가 완료되면 응답할 `ActorRef<Replicator.UpdateResponse<ORSet<String>>`
	- 이전 상태를 가져와 업데이트하는 수정 함수
- `askUpdate`의 두 번째 인자는 의해 업데이트 요청에 대한 응답이 돌아왔을 때 실행할 작업을 정의함(`InternalUpdateResponse::new`)
- 위와 작업을 수행한 뒤, 지정된 write 일관성 수준에 따라 복제 진행
- 항상 자신의 쓰기를 볼 수 있음
- 예외를 throw하여 복제를 중단할 수 있음 -> `Replicator.ModifyFailure` 가 응답으로 전송
### Get
데이터의 값을 검색하기 위해서 `Replicator`에 `Replicator.Get` 메시지를 보냄
```java
private Behavior<Command> onGetElements(GetElements command) {  
    replicatorAdapter.askGet(  
            replyTo -> new Replicator.Get<>(  
                    dataKey,  
                    Replicator.readLocal(),  
                    replyTo),  
            InternalGetResponse::new  
    );  
    return this;  
}
```
- 항상 자신의 쓰기를 볼 수 있음
- 읽기 실패 -> `Replicator.GetFailure` 전송
- 해당 key 값이 없음 -> `Replicator.NotFound`
```java
private Behavior<Command> onInternalGetResponse(InternalGetResponse command) {  
    if (command.rsp instanceof Replicator.GetSuccess<ORSet<String>> success) {  
        ORSet<String> result = success.get(dataKey);  
        Set<String> elements = result.getElements();  
        getContext().getLog().info("Current elements in the ORSet: " + elements);  
        return this;  
    } else if (command.rsp instanceof Replicator.GetFailure<ORSet<String>> failure) {  
        getContext().getLog().info("getFailure: " + failure);  
        return this;  
    }    return Behaviors.unhandled();  
}
```
- `Replicator`에서 전송하는 메세지를 처리하기 위해 위와 같은 방법을 사용
## 제한사항
- CRDT는 모든 문제 사항에서 적용할 수 없다. -> eventual consistency이 답이 아니다.
	- 때로는 strong consistency 가 필요한 경우도 있다.
- 대용량 데이터를 위해 설계된 것이 아니다.
	- 항목의 수는 100,000개를 넘으면 안된다.
	- 클러스터에 새 노드가 추가될 때마다 모든 항목들이 새 노드로 전송(gossiped)되므로 시간이 매우 오래 걸릴 수 있다.
	- 모든 데이터는 메모리에 저장되므로 대용량 데이터에 적합하지 않다.
## 참고
- [Akka Distributed Data](https://doc.akka.io/docs/akka/current/typed/distributed-data.html)
