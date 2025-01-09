# String Pool

> JVM의 String 관리 방식과 메모리 최적화에 대해 알아보자.

## String Pool이란?
String Pool은 JVM의 힙 영역에 존재하는 문자열 리터럴을 저장하는 특별한 메모리 영역이다. String의 불변성(immutability)을 활용하여 동일한 문자열 리터럴을 재사용함으로써 메모리를 효율적으로 관리한다.

## 구현 방식
String Pool은 내부적으로 Hash Table 자료구조를 사용하여 구현되어 있다. 문자열을 찾을 때 해시 함수를 통해 빠른 검색이 가능하며, 충돌(collision)이 발생할 경우 체이닝 방식으로 처리한다.

```java
String str1 = "hello";  // String Pool에 "hello" 저장
String str2 = "hello";  // String Pool에서 재사용
String str3 = new String("hello");  // 힙 영역에 새로운 객체 생성

System.out.println(str1 == str2);  // true
System.out.println(str1 == str3);  // false
```

## PermGen에서 Metaspace로의 변화
- Java 7 이전: String Pool은 PermGen 영역에 존재
  - PermGen은 고정된 크기를 가져 `OutOfMemoryError` 발생 가능성이 높았음
- Java 8 이후: Metaspace로 변경
  - Native 메모리 영역을 사용하여 동적으로 크기가 조절됨
  - `XX:MaxMetaspaceSize`로 크기 제한 가능

## intern() 메서드
`intern()` 메서드는 힙 영역에 생성된 String 객체를 String Pool에 등록하고 참조를 반환한다. 이미 Pool에 동일한 문자열이 있다면 해당 참조를 반환한다.

String 리터럴 방식(`String str = "hello"`)으로 문자열을 생성하면 내부적으로 `intern()` 메서드가 자동으로 호출된다. 따라서 리터럴 방식으로 생성된 동일한 문자열은 항상 같은 참조를 갖게 된다.

```java
String str1 = new String("hello").intern();
String str2 = "hello";  // 내부적으로 intern() 호출
System.out.println(str1 == str2);  // true

String str3 = "world";  // 내부적으로 intern() 호출
String str4 = "world";  // 내부적으로 intern() 호출
System.out.println(str3 == str4);  // true
```

## String Deduplication
Java 8u20부터 도입된 G1 GC의 String Deduplication 기능은 힙 메모리에서 중복된 String 객체를 자동으로 제거한다.

- `-XX:+UseStringDeduplication` 옵션으로 활성화
- Young GC 이후 Survivor 영역의 문자열 객체들을 검사
- 동일한 문자열을 가진 객체들은 하나의 char[] 배열을 공유하도록 변경

## String Pool 성능 테스트
String Pool의 성능을 측정하기 위해 대량의 문자열을 생성하고 intern()하는 테스트를 수행할 수 있다.

```java
public class StringPoolPerformanceMain {
    public static void main(String[] args) {
        Date start = new Date();
        List<String> strings = new ArrayList<>();

        // 1천만 개의 문자열 생성
        for(int i = 1; i < 10_000_000; i++) {
            String str = String.valueOf(i).intern();
            strings.add(str);  // GC 방지를 위한 참조 유지
        }

        Date end = new Date();
        System.out.println("실행 시간: " + (end.getTime() - start.getTime()) + "ms");
    }
}
```

JVM 옵션 `-XX:+PrintStringTableStatistics`를 사용하면 String Pool의 상태를 확인할 수 있다. 위 코드를 실행한 결과는 다음과 같다:

```text
실행 시간: 3669ms
SymbolTable statistics:
Number of buckets       :     32768 =    262144 bytes, each 8
Number of entries       :     23388 =    374208 bytes, each 16
Number of literals      :     23388 =    899176 bytes, avg  38.000
Total footprint         :           =   1535528 bytes
Average bucket size     :     0.714
Variance of bucket size :     0.708
Std. dev. of bucket size:     0.841
Maximum bucket size     :         7
```

테스트 결과를 분석해보면:
- 1천만 개의 문자열을 생성하고 intern()하는데 약 3.7초가 소요됨
- String Pool의 버킷 수는 32,768개로, 각 버킷은 8바이트를 차지
- 실제 저장된 문자열(엔트리) 수는 23,388개
- 평균 버킷 크기는 0.714로, 해시 충돌이 많지 않음을 보여줌
  - 버킷에는 0개 또는 1개 이상의 엔트리가 존재함 
  - 만약 모든 버킷에 균등하게 데이터가 분배되어 있다고 가정하면, 하나의 버킷에 0.714개의 엔트리가 존재함
- 최대 버킷 크기가 7이라는 것은 한 버킷에 최대 7개의 문자열이 체이닝되어 있다는 의미

이러한 성능 테스트를 통해 String Pool의 메모리 사용량과 검색 효율성을 확인할 수 있다.

### 주의점
- 대량의 문자열을 intern()할 경우 String Pool의 크기가 급격히 증가할 수 있음
- 실제 운영 환경에서는 필요한 경우에만 선별적으로 `intern()` 사용을 고려
- `intern()`의 과도한 사용은 String Pool의 크기를 증가시켜 성능 저하 가능
- String Pool의 문자열은 GC 대상이 되지만, 레퍼런스가 있는 한 메모리에서 해제되지 않음
- 대량의 문자열 처리 시에는 `StringBuilder`나 `StringBuffer` 사용을 고려

String Pool은 String 객체의 특성을 활용한 JVM의 메모리 최적화 방식으로, 적절히 활용하면 메모리 사용량을 줄일 수 있다. 하지만 과도한 사용은 오히려 성능 저하를 일으킬 수 있으므로 주의가 필요하다.

## Reference
[[Udemy] Discover how coding choices, benchmarking, performance tuning and memory management can optimize your Java applications](https://www.udemy.com/course/java-application-performance-and-memory-management/)
