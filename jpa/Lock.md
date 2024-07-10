## Lock
- 동시성 제어를 위해 엔티티를 잠그는 데 사용한다.
- 데이터베이스에서 특정 레코드를 읽거나 수정할 때, 다른 트랜잭션이 동시에 접근 못하도록 할 수 있음
## `@Lock`
- Spring Data Jpa에서는 락 제어를 위해 아래와 같은 어노테이션을 제공함
```java
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Lock {  
    LockModeType value();  
}
```
- `LockModeType` 의 값을 설정하여 어떤 방식으로 동시성을 제어할 지 선택할 수 있음
```java
package jakarta.persistence;  
  
public enum LockModeType {  
    READ,  
    WRITE,  
    OPTIMISTIC,  
    OPTIMISTIC_FORCE_INCREMENT,  
    PESSIMISTIC_READ,  
    PESSIMISTIC_WRITE,  
    PESSIMISTIC_FORCE_INCREMENT,  
    NONE;  
  
    private LockModeType() {  
    }}
```
- `READ`: `OPTIMISTIC`과 같은 기능
- `WRITE`: `OPTIMISTIC_FORCE_INCREMENT`와 같은 기능
`READ`, `WRITE`는 잘 사용하지 않는다. JPA 구버전의 과거 호환성을 고려해서 남겨두었다.
## Optimistic Lock(낙관적 락)
- 주로 충돌이 드물다고 가정하고 사용하는 방식
- 트랜잭션이 데이터를 수정할 때 데이터의 변경 여부를 확인하여 충돌을 방지함
- 충돌이 발생하였을 때 애플리케이션에서 이를 처리하는 방식
- 주로 `@Version` 어노테이션과 같이 사용
**`OPTIMISTIC_FORCE_INCREMENT`**
-  엔티티가 수정되지 않아도, 트랜잭션이 커밋될 때 엔티티의 버전을 강제로 증가
- 여러 트랜잭션이 각각 엔티티의 상태 변화에 대해 정확히 알 필요가 있을 때 사용
## Pessimistic Lock(비관적 락)
- 데이터베이스에서 직접 레코드를 잠궈서 다른 트랜잭션이 접근할 수 없도록 하는 방식
- `PESSIMISTIC_READ`: 다른 트랜잭션이 데이터를 읽을 수 있지만, 수정 불가
- `PESSIMISTIC_WRITE`: 다른 트랜잭션에서 읽거나 수정 불가
- `PESSIMISTIC_FORCE_INCREMENT`: `PESSIMISTIC_WRITE` + 버전 기능이 추가된 것
