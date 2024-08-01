# Akka Cluster
- Akka Cluster를 통해 여러 서버를 하나의 클러스터로 묶을 수 있다.
- peer to peer 방식으로 클러스터를 구성한다. -> 중앙에서 클러스터를 관리하는 coordinator가 필요 없음
- `Cluster Membership API` 를 통해 멤버를 관리할 수 있다.
## 핵심 용어
- 노드: 클러스터의 논리적인 멤버(서버, 인스턴스 등등)
- 시드 노드(seed node): 새로운 노드가 클러스터에 조인할 때 시드 노드를 통해 진행한다. -> 먼저 시드 노드가 있어야 다른 노드가 클러스터에 조인할 수 있음
- Gossip 프로토콜: 노드 간의 정보를 전파하고 동기화하기 위한 메커니즘
## Cluster에 Join하기
- 노드 간에 네트워크로 접근이 가능해야하고, 같은 `ActorSystem` 이름을 사용해야한다.
- 새로운 노드가 클러스터에 join 하기 위해 먼저 시드 노드들에게 `Join Command`를 보낸다.
- 시드 노드는 클러스터 내에 모든 노드에게 새 노드가 Join 한다고 전파한다. 이때 새로운 노드의 상태는 `Joining`이 된다.
- 새로운 노드의 상태는 `Up` 으로 변경된다. 이러한 상태 변화도 클러스터 내에 모든 노드에게 전파된다.
## 코드
**application.conf**
```hocon
akka {  
  actor {  
    provider = "cluster"  
  }  
  remote {  
    artery {  
      enabled = on  
      transport = tcp  
      canonical.hostname = "10.11.10.29"  
      canonical.port = 0  
    }  
  }  
  
  cluster {  
    seed-nodes = []  
    downing-provider-class = "akka.cluster.sbr.SplitBrainResolverProvider"  
  }  
  loggers = ["akka.event.slf4j.Slf4jLogger"]  
  logging-filter = "akka.event.slf4j.Slf4jLoggingFilter"  
  loglevel = "DEBUG"  
}
```
- `akka.actor.provider`: provider를 cluster 모드로 설정한다. 이를 통해 여러 JVM에 걸쳐 있는 노드들을 하나의 논리적인 클러스터로 구성한다.
- `akka.remote.artery.canonical`: 현재 노드의 호스트 이름과 포트를 지정한다.
- `akka.cluster.seed-nodes`: 클러스터의 시드 노드를 설정한다. 현재는 비어두었고, 동적으로 시드 노드를 설정할 예정이다.

**ClusterListener.java**
```java
public class ClusterListener extends AbstractBehavior<ClusterEvent.ClusterDomainEvent> {

    public ClusterListener(ActorContext<ClusterEvent.ClusterDomainEvent> context) {
        super(context);
    }

    public static Behavior<ClusterEvent.ClusterDomainEvent> create() {
        return Behaviors.setup(ClusterListener::new);
    }

    @Override
    public Receive<ClusterEvent.ClusterDomainEvent> createReceive() {
        return newReceiveBuilder()
                .onMessage(ClusterEvent.MemberUp.class, this::onMemberUp)
                .onMessage(ClusterEvent.LeaderChanged.class, this::onLeaderChanged)
                .onMessage(ClusterEvent.MemberRemoved.class, this::onMemberRemoved)
                .onMessage(ClusterEvent.UnreachableMember.class, this::onUnreachableMember)
                .build();
    }

    private Behavior<ClusterEvent.ClusterDomainEvent> onMemberUp(ClusterEvent.MemberUp event) {
        getContext().getLog().info("Member Up: {}", event.member());
        return this;
    }

    private Behavior<ClusterEvent.ClusterDomainEvent> onLeaderChanged(ClusterEvent.LeaderChanged event) {
        getContext().getLog().info("Leader Changed: {}", event.getLeader());
        return this;
    }

    private Behavior<ClusterEvent.ClusterDomainEvent> onMemberRemoved(ClusterEvent.MemberRemoved event) {
        getContext().getLog().info("Member Removed: {}", event.member());
        return this;
    }

    private Behavior<ClusterEvent.ClusterDomainEvent> onUnreachableMember(ClusterEvent.UnreachableMember event) {
        getContext().getLog().info("Member detected as unreachable: {}", event.member().address());
        return this;
    }
}
```
- 클러스터 내에서 발생하는 이벤트들을 감지하기 위한 `Actor` 이다.
- `createReceive()` 메서드에서 이벤트 타입에 따라서 처리하는 메서드를 설정하였다.

**AkkaConfig.java**
```java
@Bean  
public Cluster cluster() {  
    Cluster cluster = Cluster.get(actorSystem());  
    cluster.subscriptions().tell(Subscribe.create(clusterListener(), ClusterEvent.ClusterDomainEvent.class));  
    return cluster;  
}
```
- `Cluster API` 를 사용하기 위한 인스턴스는 `Bean`으로 등록해놓았다.
- 추가로 리스너도 구독한 상태로 설정하였다.

**AkkaClusterManager.java**
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AkkaClusterManager {

    private final Cluster cluster;
    private final List<Address> seedNodes = new ArrayList<>();

    @PostConstruct
    void init() {
        seedNodes.add(AddressFromURIString.parse("akka://akka-system@127.0.0.1:2551"));
        cluster.manager().tell(new JoinSeedNodes(seedNodes));
    }

    public List<Member> getNodes() {
        Iterable<Member> members = cluster.state().getMembers();
        return StreamSupport.stream(members.spliterator(), false).toList();
    }
}

```
- `init()` 메서드에서 시드 노드를 설정해주었다.
- 따라서 해당 URL로 실행되는 서버는 시드 노드로 설정된다.
- 새로운 노드가 실행되면 시드 노드에 Join 요청을 보내게 된다.

**AkkaController.java**
```java
@RestController  
@RequiredArgsConstructor  
@Slf4j  
public class AkkaController {  
  
    private final AkkaClusterManager akkaClusterManager;  
  
    @GetMapping("/members")  
    public List<ClusterMember> members() {  
        return akkaClusterManager.getNodes().stream()  
                .map(m -> ClusterMember.of(m.address().toString(), m.status().toString()))  
                .toList();  
    }  
}
```
- 클러스터 내에 멤버(노드)들을 조회하기 위한 API
## 테스트
- 빌드해서 나온 jar 파일을 여러 개 실행한다.
```shell
java -jar -Dakka.remote.artery.canonical.port=2551 hello-akka-0.0.1-SNAPSHOT.jar --server.port=8080
java -jar -Dakka.remote.artery.canonical.port=0 hello-akka-0.0.1-SNAPSHOT.jar --server.port=8081
```
- 시드 노드의 역할을 하는 인스턴스를 먼저 띄운다.
- 일반 노드의 역할을 하는 인스턴스 1개를 띄운다.
- `/members`에 요청을 보내면 아래와 같은 결과를 리턴한다.
```json
[
  {
    "address": "akka://akka-system@127.0.0.1:2551",
    "status": "Up"
  },
  {
    "address": "akka://akka-system@127.0.0.1:54677",
    "status": "Up"
  }
]
```
## 정리
- 분산 시스템에서 사용할 Akka Cluster 기본적인 개념과 구현 방법을 이해할 수 있었다.
