# JBoss 프로세스

## 핵심

JBoss를 Standalone 모드로 실행하면 실행 스크립트와 실제 JBoss를 실행하는 Java 프로세스를 확인할 수 있다.

단순화하면 다음과 같은 구조로 이해할 수 있다.

```
standalone.sh
PID 1001
    │
    └── Java(JBoss)
        PID 1002
        PPID 1001
```

실제 JBoss 애플리케이션 서버가 동작하는 핵심 프로세스는 **Java(JVM) 프로세스**이다.

PID와 PPID 등 프로세스의 기본 개념은 [[process|프로세스]]를 참고한다.

## JBoss 프로세스 확인

Linux에서는 다음과 같이 JBoss 프로세스를 찾을 수 있다.

```
ps -ef | grep <검색할_키워드>
```

예를 들어 특정 JBoss 인스턴스를 식별할 수 있는 이름으로 검색할 수 있다.

```
ps -ef | grep server01
```

검색 결과에서는 실행 스크립트와 실제 Java 프로세스가 함께 보일 수 있다.

```
user  1001     1  ...  /bin/sh /path/to/jboss/bin/standalone.sh ...
user  1002  1001  ...  /path/to/java ... org.jboss.as.standalone ...
```

두 번째 프로세스의 `PPID`가 첫 번째 프로세스의 `PID`와 동일하다.

```
standalone.sh
PID = 1001
      │
      └── Java
          PID  = 1002
          PPID = 1001
```

이를 통해 두 프로세스의 부모-자식 관계를 확인할 수 있다.

Linux에서 프로세스를 조회하는 명령어는 [[process-command|Linux 프로세스 관련 명령어]]를 참고한다.

## standalone.sh

`standalone.sh`는 Linux 환경에서 JBoss를 Standalone 모드로 실행하기 위한 스크립트이다.

```
/path/to/jboss/bin/standalone.sh
```

스크립트에서는 JBoss 실행에 필요한 환경과 JVM 옵션 등을 구성하고 Java 프로세스를 실행한다.

따라서 `ps -ef` 결과에서 `standalone.sh`가 보인다고 해서 해당 프로세스 자체가 실제 JBoss JVM인 것은 아니다.

실제 JBoss가 동작하는 Java 프로세스도 함께 확인해야 한다.

## Java 프로세스

실제 JBoss는 JVM 위에서 동작한다.

따라서 실제 프로세스의 CMD를 보면 다음과 같은 구조를 확인할 수 있다.

```
/path/to/java
-Xms2048m
-Xmx2048m
-Xss256k
-Djboss.node.name=server01
-Djboss.server.base.dir=/path/to/server
...
-jar /path/to/jboss-modules.jar
...
org.jboss.as.standalone
...
```

긴 CMD 전체를 외울 필요는 없다.

프로세스를 확인할 때는 **어떤 Java가 실행되고 있는지, 어떤 JBoss 인스턴스인지, 어떤 설정으로 실행되고 있는지**를 중심으로 확인한다.

## CMD에서 확인할 내용

### Java 실행 경로

```
/path/to/java
```

어떤 Java 실행 파일을 사용하고 있는지 확인할 수 있다.

필요한 경우 Java 버전이나 JDK 경로를 파악하는 단서가 된다.

### JVM 메모리 옵션

```
-Xms2048m
-Xmx2048m
-Xss256k
```

대표적으로 다음과 같은 JVM 메모리 설정을 확인할 수 있다.

- `-Xms` : 초기 Heap 크기
    
- `-Xmx` : 최대 Heap 크기
    
- `-Xss` : 스레드 Stack 크기
    

자세한 내용은 [[jvm-memory|JVM 메모리]]를 참고한다.

### `-D` 옵션

`-D`는 Java의 **System Property**를 설정하는 JVM 옵션이다.

형식은 다음과 같다.

```
-D<프로퍼티_이름>=<값>
```

예를 들어 다음과 같은 설정이 있을 수 있다.

```
-Djboss.node.name=server01
-Dspring.profiles.active=prod
```

Java 코드에서는 `System.getProperty()`를 통해 System Property를 조회할 수 있다.

```
System.getProperty("<프로퍼티_이름>");
```

`-D`는 JBoss 전용 옵션이 아니라 Java에서 사용할 수 있는 일반적인 JVM 옵션이다.

## JBoss 관련 주요 설정

CMD에는 현재 실행 중인 JBoss 인스턴스를 식별하는 데 도움이 되는 여러 설정이 포함될 수 있다.

### Node Name

```
-Djboss.node.name=server01
```

JBoss 노드의 이름을 확인할 수 있다.

### Server Base Directory

```
-Djboss.server.base.dir=/path/to/server
```

해당 JBoss 인스턴스가 사용하는 서버 기본 디렉터리를 확인할 수 있다.

### Spring Profile

JBoss에서 Spring 애플리케이션을 실행하고 있다면 다음과 같은 설정을 볼 수도 있다.

```
-Dspring.profiles.active=prod
```

Spring의 활성 Profile을 확인할 수 있다.

### JBoss 설정 파일

```
-c standalone-ha.xml
```

JBoss가 어떤 설정 파일을 사용하여 실행되었는지 확인할 수 있다.

환경에 따라 `standalone.xml`, `standalone-ha.xml` 등의 설정 파일을 사용할 수 있다.

## JBoss Modules

CMD에서 다음과 같은 내용을 볼 수 있다.

```
/path/to/jboss-modules.jar
```

그리고 뒤쪽에 다음과 같은 값이 나타날 수 있다.

```
org.jboss.as.standalone
```

JBoss는 JBoss Modules라는 모듈 시스템을 사용한다.

따라서 `org.jboss.as.standalone`은 단순히 Java의 실행 클래스 이름으로 보기보다는 **JBoss Modules를 통해 실행되는 JBoss Standalone 관련 모듈 식별자**로 이해하는 것이 적절하다.

## 프로세스 분석 흐름

JBoss 프로세스에 문제가 발생했을 때는 우선 다음과 같은 흐름으로 프로세스 상태를 확인할 수 있다.

```
1. JBoss 관련 프로세스 검색

   ps -ef | grep <검색할_키워드>

           ↓

2. PID / PPID 확인

           ↓

3. standalone.sh와 Java 프로세스 관계 확인

           ↓

4. Java 프로세스의 CMD 확인

           ↓

5. JBoss 인스턴스와 실행 설정 확인

           ↓

6. 필요한 경우 로그 확인

   tail -100 <로그_파일_경로>
   tail -f <로그_파일_경로>
```

특히 프로세스를 종료하거나 재기동해야 하는 상황에서는 **실행 스크립트의 PID와 실제 Java 프로세스의 PID를 구분해서 확인하는 것이 중요하다.**

프로세스 종료 명령어는 [[process-command|Linux 프로세스 관련 명령어]]를 참고한다.

## 정리

```
standalone.sh
    ↓
Java(JVM) 프로세스 실행
    ↓
JBoss 동작
```

JBoss 프로세스를 확인할 때는 다음을 중심으로 본다.

- `PID` : 해당 프로세스의 ID
    
- `PPID` : 부모 프로세스 확인
    
- `CMD` : 실제 실행 명령어와 옵션
    
- Java 실행 경로
    
- JVM 옵션
    
- JBoss 인스턴스 관련 설정
    
- JBoss 설정 파일
    

`ps -ef`의 긴 CMD를 모두 이해하거나 외우기보다는 **현재 어떤 JBoss 인스턴스가 어떤 JVM과 설정으로 실행되고 있는지 파악할 수 있는 수준**으로 읽는 것이 우선 중요하다.