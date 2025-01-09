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

JVM 옵션 `-XX:+PrintStringTableStatistics`를 사용하면 SymbolTable과 StringTable의 상태를 확인할 수 있다. 테스트 결과는 다음과 같다:

### SymbolTable 분석
SymbolTable은 JVM 내부에서 사용하는 심볼들을 관리하는 테이블이다.
```text
Number of buckets       :     32768 =    262144 bytes, each 8
Number of entries       :     23976 =    383616 bytes, each 16
Number of literals      :     23976 =    914912 bytes, avg  38.000
Average bucket size     :     0.732
Maximum bucket size     :         7
```
- 버킷 수는 32,768개로 각 버킷당 8바이트 사용
- 평균 버킷 사이즈는 0.732로 여유 공간이 충분함
- 최대 버킷 사이즈가 7이므로 체이닝이 길지 않음

### StringTable 분석
StringTable은 `String.intern()`으로 관리되는 실제 문자열 풀이다.
```text
Number of buckets       :   4194304 =  33554432 bytes, each 8
Number of entries       :  10001849 = 160029584 bytes, each 16
Number of literals      :  10001849 = 480121072 bytes, avg  48.000
Average bucket size     :     2.385
Maximum bucket size     :        13
Std. dev. of bucket size:     1.966
```
- 약 1천만 개의 문자열이 `intern()` 되어 있음
- 버킷 수가 4,194,304개로 SymbolTable보다 훨씬 큼
- 평균 버킷 사이즈는 2.385로 버킷당 평균 2~3개의 문자열이 존재
- 최대 버킷 사이즈는 13으로, 일부 버킷에서는 상당한 충돌이 발생

### 성능 분석
- 1천만 개의 문자열을 `intern()`하는데 약 3.6초 소요
- StringTable은 총 약 642MB(673,705,088 bytes)의 메모리를 사용
- 문자열 하나당 평균 48바이트 사용
- 버킷 사이즈의 표준편차(1.966)는 문자열들이 비교적 고르게 분포되어 있음을 보여줌

### 주의점
- 대량의 문자열을 `intern()`할 경우 메모리 사용량이 급격히 증가할 수 있음
- 평균 버킷 사이즈가 2.385인 것은 해시 충돌이 꽤 발생하고 있다는 의미로, 검색 성능에 영향을 줄 수 있음
- 실제 운영 환경에서는 필요한 경우에만 선별적으로 `intern()` 사용을 고려해야 함
- `intern()`의 과도한 사용은 String Pool의 크기를 증가시켜 성능 저하 가능
- String Pool의 문자열은 GC 대상이 되지만, 레퍼런스가 있는 한 메모리에서 해제되지 않음
- 대량의 문자열 처리 시에는 `StringBuilder`나 `StringBuffer` 사용을 고려

String Pool은 String 객체의 특성을 활용한 JVM의 메모리 최적화 방식으로, 적절히 활용하면 메모리 사용량을 줄일 수 있다. 하지만 과도한 사용은 오히려 성능 저하를 일으킬 수 있으므로 주의가 필요하다.

## Reference
[[Udemy] Discover how coding choices, benchmarking, performance tuning and memory management can optimize your Java applications](https://www.udemy.com/course/java-application-performance-and-memory-management/)
