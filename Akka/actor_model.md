# Akka Actor Model
JVM 환경에서 동시성 및 분산형 애플리케이션을 쉽게 개발할 수 있도록 돕는 오픈소스 라이브러리
## Actor Model
> **everything is an actor** (모든 것은 액터다.)

Actor Model은 1974년 Carl Hewitt라는 사람이소개한 모델이다. Akka에서는 해당 모델을 통해 문제를 해결한다.
### Actor 특징
- 상태와 행동을 캡슐화하는 객체이다.
- Actor는 비동기 메세지를 통해서만 상호 작용한다. 직접적인 메서드 호출을 하지 않는다.(이벤트 기반)
- Actor는 다른 Actor를 생성할 수 있고, 다른 Actor에게 메세지를 보낼 수 있으며, 자신이나 자신이 생성한 Actor를중단 할 수 있다.
### Actor API
- classic Actor API
	- Akka에서 처음 도입된 방식
	- 메세지 유형에 제한이 없다. 모든 메세지를 Object 타입으로 받아서 사용한다.
	- 메세지 타입이 명시적이지 않아 런타임 에러가 발생할 수 있다.
- Typed Actor API
	- 타입 시스템을 사용
	- 메세지 타입이 명확히 정의되어 있어 컴파일 시점에 타입 검사가 이루어진다.
	- Actor가 주고 받는 메세지 타입이 명시적이므로, 프로토콜이 명확하다.
	- 타입을 명시적으로 정의해야 하기 때문에 코드가 더 길어질 수 있다.
## Code
`Typed API`를 사용한다.
- DB에 접근해서 데이터 조회 후 수정하는 Actor
```java
import akka.actor.typed.Behavior;  
import akka.actor.typed.javadsl.AbstractBehavior;  
import akka.actor.typed.javadsl.ActorContext;  
import akka.actor.typed.javadsl.Behaviors;  
import akka.actor.typed.javadsl.Receive;  
import example.helloakka.domain.count.CounterRepository;  
import lombok.extern.slf4j.Slf4j;  
  
@Slf4j  
public class CounterActor extends AbstractBehavior<CounterActor.Command> {  
  
    public interface Command {  
    }  
    public record IncrementCounter(Long id, int value) implements Command {  
    }  
    public static Behavior<Command> create(CounterRepository counterRepository) {  
        return Behaviors.setup(ctx -> new CounterActor(ctx, counterRepository));  
    }  
  
    private final CounterRepository counterRepository;  
  
    private CounterActor(ActorContext<Command> context, CounterRepository counterRepository) {  
        super(context);  
        this.counterRepository = counterRepository;  
    }  
  
    @Override  
    public Receive<Command> createReceive() {  
        return newReceiveBuilder()  
                .onMessage(IncrementCounter.class, this::onIncrementCounter)  
                .build();  
    }  
  
    private Behavior<Command> onIncrementCounter(IncrementCounter command) {  
        counterRepository.findById(command.id).ifPresent(counter -> {  
            counter.plus(command.value());  
            counterRepository.save(counter);  
            log.info("Counter incremented: {}/{}", counter.getName(), counter.getNumberValue());  
        });  
        return this;  
    }  
}
```
- Typed API에서 Actor를 정의할 때 `AbstractBehavior` 을 상속하여 정의한다. 타입 안정성을 위해 제네릭에는 해당 Actor가 처리하는 메세지의 타입을 명시해준다.
- `create()` 메서드를 통해 Actor에 `ActorContext` 전달 및 초기 세팅을 진행한다.
- `createReceive()` 메서드를 오버라이딩하여 메세지를 받았을 때의 메세지 처리 로직을 작성한다.
`CounterActor`를 사용하는 클라이언트에서는 아래와 같이 사용한다.
```java
@Service  
@RequiredArgsConstructor  
public class CounterService {  
  
    private final ActorRef<CounterActor.Command> counterActor;  
  
    @Transactional  
    public void plus(Long id, int value) {  
        counterActor.tell(new CounterActor.IncrementCounter(id, value));  
    }  
}
```
```java
@Configuration  
@RequiredArgsConstructor  
public class AkkaConfig {  
  
    private final CounterRepository counterRepository;  
  
    @Bean  
    public ActorSystem<Void> actorSystem() {  
        Config config = ConfigFactory.load();  
        return ActorSystem.create(Behaviors.empty(), "akka-system", config);  
    }  
  
    @Bean  
    public ActorRef<CounterActor.Command> counterActor() {  
        return actorSystem().systemActorOf(CounterActor.create(counterRepository), "counterActor", Props.empty());  
    }  
  
}
```
`tell()` 메서드를 통해 해당 Actor에게 메세지를 전달한다.
### 테스트
```java
@SpringBootTest  
class CounterServiceTest {  
  
    @Autowired  
    private CounterServiceV2 counterService;  
    @Autowired  
    private CounterRepository counterRepository;  
    private Counter counter;  
  
    @BeforeEach  
    void setup() {  
        counter = Counter.of("test");  
        counterRepository.saveAndFlush(counter);  
    }  
  
    @Test  
    void 동시에_올리기() throws InterruptedException {  
        int threadCount = 100;  
        ExecutorService executorService = Executors.newFixedThreadPool(32);  
        for (int i = 0; i < threadCount; i++) {  
            executorService.submit(() -> {  
                counterService.plus(counter.getId(), 1);  
            });  
        }        Thread.sleep(1000);  
  
        Optional<Counter> result = counterRepository.findById(counter.getId());  
  
        assertTrue(result.isPresent());  
        assertEquals(100, result.get().getNumberValue());  
    }  
  
}
```
- 동시에 100개의 스레드가 `plus()` 메서드를 실행하는 테스트
- Actor가 모든 작업을 수행하고 마치기 위해 1초간 sleep
- 예상하는 동작
	- 동시성 이슈로 인해 여러 스레드가 같은 데이터에 접근하여 원하는 결과를 얻어내지 못할 것이다.
- 실제 결과
```text
2024-07-31T14:08:20.092+09:00  INFO 56384 --- [hello-akka] [lt-dispatcher-3] e.helloakka.actor.counter.CounterActor   : Counter incremented: test/100
```
`lt-dispatcher-3` 이라는 스레드를 통해 메세지를 비동기적으로 처리하는 것을 확인 할 수 있다.
동시에 100개의 요청이 들어와도 기능이 정상적으로 수행되는 것을 확인할 수 있다.
- Actor 내부에 **메일 박스**라고 하는 메세지 큐가 있어 동시에 메세지를 받아도 순차적으로 처리할 수 있게 된다.
- 하지만 서버가 여러 대로 구성되어 있으면 위와 같은 방법으로는 동시성 이슈를 해결할 수 없다.
- n중화된 서버에서 Akka를 사용해서 동시성 이슈를 처리하기 위해서는 `Akka Cluster` 기능을 사용하는듯? 
> Akka Cluster: 여러 노드에 걸쳐 Actor를 배포하고 관리하는 기능
## 정리
- Actor 정의 방법: `AbstractBehavior` 의 상속을 통해 Actor를 정의한다.
- 비동기 동작: 별도의 스레드를 생성하여 비동기적으로 작업을 처리한다.
- 순차적 처리: Actor 내부의 메세지 큐를 통해 한 번에 하나의 메세지만 처리한다.
