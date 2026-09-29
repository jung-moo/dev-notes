# JVM 메모리

## 핵심

JVM은 Java 프로그램을 실행하면서 필요한 메모리를 여러 영역으로 나누어 관리한다.

대표적인 JVM Runtime Data Area는 다음과 같다.

```
JVM Runtime Data Areas
├── Heap
├── Java Stack
├── Method Area
├── PC Register
└── Native Method Stack
```

백엔드 개발에서는 우선 **Heap과 Stack의 차이**를 이해하는 것이 중요하다.

일반적인 프로세스의 메모리 구조는 [[process-memory|프로세스 메모리]]에서 다룬다.

## Heap

객체와 배열 등이 주로 저장되는 영역이다.

```
Member member = new Member();
```

개념적으로 보면 다음과 같다.

```
Stack                         Heap

member ───────────────────→ Member 객체
```

- `member` : 객체를 참조하는 지역변수
    
- `new Member()` : Heap에 생성되는 객체
    

지역변수가 만들어진 위치와 객체가 만들어지는 위치를 구분해서 이해하는 것이 중요하다.

## 객체의 수명

Heap에 생성된 객체는 객체를 생성한 메서드가 종료된다고 바로 사라지지 않는다.

```
Member createMember() {
    Member member = new Member();
    return member;
}
```

`createMember()`가 종료되면 해당 메서드의 Stack Frame은 제거된다.

하지만 반환된 `Member` 객체를 다른 곳에서 계속 참조하고 있다면 Heap의 객체는 계속 사용할 수 있다.

```
메서드 종료
    ↓
Stack Frame 제거
    ↓
Heap 객체를 다른 곳에서 참조하고 있는가?
    ├── Yes → 계속 사용 가능
    └── No  → GC 대상이 될 수 있음
```

더 이상 도달할 수 없는 객체는 Garbage Collection의 대상이 될 수 있다.

## Stack

Java Stack은 **메서드 호출과 관련된 정보**를 관리한다.

메서드가 호출될 때마다 해당 호출을 위한 Stack Frame이 생성된다.

```
메서드 호출
    ↓
Stack Frame 생성
    ↓
메서드 실행
    ↓
메서드 종료
    ↓
Stack Frame 제거
```

Stack Frame에는 지역변수와 메서드 실행에 필요한 정보 등이 저장된다.

예를 들어 다음 코드가 있다.

```
void test() {
    int age = 20;
    Member member = new Member();
}
```

개념적으로 다음과 같이 이해할 수 있다.

```
Stack                         Heap

test() Stack Frame
├── age = 20
└── member ────────────────→ Member 객체
```

`test()`가 종료되면 `test()`의 Stack Frame과 그 안의 지역변수는 제거된다.

하지만 Heap의 `Member` 객체는 **다른 곳에서 계속 참조되고 있다면 유지될 수 있다.**

## Heap과 Stack을 나누는 이유

핵심적인 이유 중 하나는 **관리해야 하는 데이터의 수명이 다르기 때문**이다.

### Stack

메서드 실행 수명에 맞춰 관리한다.

```
메서드 호출
→ Stack Frame 생성
→ 메서드 종료
→ Stack Frame 제거
```

### Heap

객체의 수명은 객체를 생성한 메서드의 수명과 다를 수 있다.

```
객체 생성
→ 여러 곳에서 참조 가능
→ 생성한 메서드가 종료되어도 사용 가능
→ 더 이상 도달할 수 없으면 GC 대상
```

따라서 Stack은 **메서드 호출 중심**, Heap은 **객체의 수명 중심**으로 이해하면 좋다.

## Garbage Collection

Garbage Collection(GC)은 JVM이 더 이상 사용되지 않는 객체의 메모리를 회수하는 작업이다.

```
Heap에 객체 생성
    ↓
객체 사용
    ↓
더 이상 객체에 도달할 수 없음
    ↓
GC 대상
    ↓
GC가 메모리 회수
```

객체가 GC 대상이 되었다고 해서 **즉시 메모리에서 제거되는 것은 아니다.**

실제 메모리 회수 시점은 JVM의 Garbage Collector가 결정한다.

## Method Area

Method Area는 JVM이 사용하는 **클래스와 관련된 정보**를 저장하는 영역이다.

클래스의 구조, 메서드 정보 등 JVM이 클래스를 실행하기 위해 필요한 정보가 관리된다.

Java 8 HotSpot JVM에서는 클래스 메타데이터를 저장하기 위해 **Metaspace**를 사용한다.

## PC Register

각 스레드가 현재 실행하고 있는 JVM 명령어의 위치를 관리하기 위한 영역이다.

스레드마다 각각의 PC Register를 가진다.

## Native Method Stack

Java가 아닌 네이티브 코드를 실행할 때 사용되는 Stack 영역이다.

Java에서 JNI 등을 통해 네이티브 코드를 호출할 때 관련된다.

일반적인 Java 백엔드 개발에서는 Heap과 Java Stack에 비해 직접 다룰 일이 적다.

## 관련 JVM 옵션

### `-Xms`

JVM Heap의 **초기 크기**를 설정한다.

```
-Xms2048m
```

초기 Heap 크기를 약 2GB로 설정한다.

### `-Xmx`

JVM Heap의 **최대 크기**를 설정한다.

```
-Xmx2048m
```

최대 Heap 크기를 약 2GB로 설정한다.

### `-Xss`

각 스레드의 **Stack 크기**를 설정한다.

```
-Xss256k
```

각 스레드의 Stack 크기를 256KB로 설정한다.

## 정리

```
Heap
→ 객체와 배열 등이 주로 저장
→ 객체의 수명에 맞춰 관리
→ 더 이상 도달할 수 없는 객체는 GC 대상

Stack
→ 메서드 호출마다 Stack Frame 생성
→ 지역변수 등이 저장
→ 메서드 종료 시 Stack Frame 제거

-Xms
→ 초기 Heap 크기

-Xmx
→ 최대 Heap 크기

-Xss
→ 스레드 Stack 크기
```

일반적인 프로세스 수준의 Heap과 Stack은 [[process-memory|프로세스 메모리]]를 참고한다.