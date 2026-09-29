# Java와 플랫폼 독립성

## 핵심

Java의 대표적인 특징 중 하나는 **플랫폼 독립성(Platform Independence)**이다.

Java 소스 코드는 특정 운영체제나 CPU의 머신 코드로 바로 컴파일되는 것이 아니라 JVM이 실행할 수 있는 **바이트코드(Bytecode)**로 컴파일된다.

```
Java Source Code
       ↓
     javac
       ↓
Java Bytecode (.class)
       ↓
      JVM
       ↓
 Machine Code
       ↓
      CPU
```

각 플랫폼에 맞는 JVM이 바이트코드를 실행하기 때문에 동일한 Java 바이트코드를 여러 플랫폼에서 실행할 수 있다.

이를 흔히 **Write Once, Run Anywhere**라고 표현한다.

## 플랫폼

플랫폼(Platform)은 프로그램이 실행되는 환경을 의미한다.

Java의 플랫폼 독립성을 이야기할 때는 주로 다음과 같은 요소를 생각할 수 있다.

```
Platform
├── Operating System
│   ├── Windows
│   ├── Linux
│   └── macOS
│
└── CPU Architecture
    ├── x86-64
    └── ARM64
```

운영체제와 CPU 아키텍처가 달라지면 프로그램이 실행되는 환경도 달라진다.

## JVM

JVM(Java Virtual Machine)은 **Java 바이트코드를 실행하는 가상 머신**이다.

Java 프로그램은 운영체제에서 직접 실행되는 것이 아니라 JVM을 통해 실행된다.

```
Java Program
      ↓
     JVM
      ↓
Operating System
      ↓
     CPU
```

JVM이 운영체제와 CPU의 차이를 처리하기 때문에 Java 프로그램은 플랫폼의 차이에 상대적으로 덜 의존할 수 있다.

단, **JVM 자체는 플랫폼에 맞는 구현이 필요하다.**

```
같은 Java Bytecode

├── Windows용 JVM
├── Linux용 JVM
└── macOS용 JVM
```

Java 프로그램이 플랫폼 독립적인 것이지 JVM 자체가 플랫폼 독립적인 것은 아니다.

JVM의 메모리 구조는 [[jvm-memory|JVM 메모리]]에서 다룬다.

## Bytecode

바이트코드(Bytecode)는 **가상 머신이 실행할 수 있도록 만들어진 중간 형태의 명령어**이다.

Java에서는 `.java` 파일을 컴파일하면 `.class` 파일이 생성되고, 이 파일에 JVM이 실행할 Java 바이트코드가 들어 있다.

```
Main.java
   ↓ javac
Main.class
   ↓
Java Bytecode
```

바이트코드는 Java에만 존재하는 개념은 아니다.

일반적으로 가상 머신이나 런타임 환경에서 실행하기 위한 **중간 명령어 형태**를 바이트코드라고 부를 수 있다.

Java 바이트코드에는 다음과 같은 JVM 명령어가 존재한다.

```
iload
iadd
istore
invokevirtual
return
```

CPU가 이 명령어를 직접 실행하는 것은 아니며 JVM이 바이트코드를 실행한다.

## Machine Code

머신 코드(Machine Code)는 **CPU가 직접 실행할 수 있는 명령어**이다.

머신 코드는 주로 CPU 아키텍처에 따라 달라진다.

```
x86-64 CPU
→ x86-64 Machine Code

ARM64 CPU
→ ARM64 Machine Code
```

따라서 머신 코드가 달라지는 가장 직접적인 기준은 운영체제가 아니라 **CPU 아키텍처**이다.

하지만 실제 실행 파일은 운영체제의 실행 파일 형식, 시스템 호출, API 등의 영향도 받는다.

따라서 같은 CPU 아키텍처를 사용하더라도 Windows용 실행 파일과 Linux용 실행 파일을 그대로 서로 실행할 수 있는 것은 아니다.

## Java의 실행 과정

Java 프로그램의 실행 과정을 단순화하면 다음과 같다.

```
.java
  ↓
javac
  ↓
.class
  ↓
Bytecode
  ↓
JVM
  ↓
Machine Code
  ↓
CPU
```

개발자는 Java 코드를 작성하고 플랫폼에 공통적으로 사용할 수 있는 바이트코드를 만든다.

각 플랫폼의 JVM이 바이트코드와 실제 실행 환경 사이의 차이를 처리한다.

## C와의 차이

C와 비교하면 Java의 플랫폼 독립성을 이해하기 쉽다.

### C

C는 일반적으로 소스 코드를 대상 플랫폼에 맞는 네이티브 코드로 컴파일한다.

```
C Source Code
      ↓
   Compiler
      ↓
Target Platform에 맞는 실행 파일
      ↓
     CPU
```

플랫폼이 달라지면 해당 플랫폼을 대상으로 다시 컴파일해야 할 수 있다.

### Java

Java는 소스 코드를 JVM이 이해하는 바이트코드로 컴파일한다.

```
Java Source Code
      ↓
    javac
      ↓
   Bytecode
      ↓
      JVM
      ↓
 Machine Code
      ↓
     CPU
```

플랫폼마다 JVM 구현은 다르지만 JVM이 동일한 바이트코드를 실행할 수 있도록 중간 계층 역할을 한다.

따라서 Java에서는 동일한 바이트코드를 여러 플랫폼에서 실행하기가 상대적으로 쉽다.

## 플랫폼 독립성의 한계

Java를 사용한다고 해서 모든 코드가 무조건 플랫폼 독립적인 것은 아니다.

Java 프로그램이 다음과 같은 요소에 직접 의존하면 플랫폼 차이가 발생할 수 있다.

- 운영체제별 파일 경로
    
- 운영체제별 명령어
    
- 네이티브 라이브러리
    
- 특정 운영체제에서만 제공되는 기능
    

예를 들어 운영체제의 명령어를 Java 코드에서 직접 실행한다면 해당 코드가 다른 운영체제에서는 동작하지 않을 수 있다.

따라서 Java의 플랫폼 독립성은 **JVM에서 실행되는 Java 바이트코드를 중심으로 이해해야 한다.**

## 정리

```
Java Source Code
        ↓
      javac
        ↓
Bytecode (.class)
        ↓
       JVM
        ↓
   Machine Code
        ↓
       CPU
```

- Java는 소스 코드를 바이트코드로 컴파일한다.
    
- 바이트코드는 JVM이 실행한다.
    
- JVM은 플랫폼에 맞는 구현이 필요하다.
    
- 머신 코드는 CPU가 직접 실행하는 명령어이다.
    
- 머신 코드는 주로 CPU 아키텍처에 따라 달라진다.
    
- JVM이 플랫폼 차이를 중간에서 처리하기 때문에 동일한 Java 바이트코드를 여러 플랫폼에서 실행할 수 있다.