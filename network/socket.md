# Socket

## 핵심

소켓(Socket)은 **프로그램이 네트워크를 통해 데이터를 주고받기 위해 사용하는 통신 창구**이다.

애플리케이션은 네트워크에 직접 접근하는 것이 아니라 운영체제가 제공하는 소켓을 통해 통신한다.

```text
Application
    ↓
  Socket
    ↓
Operating System
    ↓
 Network
```

백엔드 개발자는 소켓의 내부 구현까지 알 필요는 없지만 다음 개념은 이해하는 것이 좋다.

- IP
- Port
- Socket
- LISTEN
- TCP Connection
- Server Socket / Client Socket
- `ss` 명령어

---

## Socket

소켓은 프로그램이 실제 네트워크 통신을 하기 위해 운영체제를 통해 사용하는 **통신 객체 또는 통신 창구**이다.

```text
Application
     ↓
   Socket
     ↓
     OS
     ↓
  Network
```

Port와 Socket은 같은 개념이 아니다.

```text
Port
→ 서비스를 구분하기 위한 번호

Socket
→ 프로그램이 실제 네트워크 통신에 사용하는 객체
```

---

## 서버가 요청을 기다리는 과정

웹 서버나 WAS가 실행되면 특정 포트에서 클라이언트의 연결을 기다린다.

예를 들어 Apache가 `8082` 포트를 사용한다고 가정한다.

```text
Apache(httpd)
      ↓
소켓 생성
      ↓
8082 포트에 연결
      ↓
LISTEN
      ↓
클라이언트 연결 대기
```

이때 연결 요청을 기다리고 있는 소켓을 **Listening Socket**이라고 한다.

---

## LISTEN

`LISTEN`은 서버 프로그램이 특정 포트에서 **클라이언트의 연결 요청을 기다리고 있는 상태**이다.

예를 들어:

```text
Apache
   ↓
Port 8082
   ↓
LISTEN
```

이라는 상태라면 Apache가 `8082` 포트를 통해 들어오는 TCP 연결을 받을 준비가 되어 있다는 의미이다.

---

## 클라이언트가 연결하면

서버가 LISTEN 상태라고 해서 아직 특정 클라이언트와 통신하고 있는 것은 아니다.

클라이언트가 서버에 연결하면 실제 통신을 위한 연결이 만들어진다.

```text
[연결 전]

Server
  ↓
8082
  ↓
LISTEN


[클라이언트 연결]

Client
   ↓
TCP Connection
   ↓
Server : 8082
   ↓
연결된 Socket
   ↓
데이터 송수신
```

즉 서버에는 크게 다음 두 가지 개념이 존재한다.

```text
Listening Socket
→ 새로운 연결을 기다림

Connected Socket
→ 특정 클라이언트와 실제 통신
```

---

## TCP Socket

웹 애플리케이션에서 일반적으로 접하게 되는 소켓은 TCP 기반 소켓이다.

TCP는 연결을 만든 뒤 데이터를 주고받는 방식이다.

```text
Client
   ↓
TCP 연결
   ↓
Server
   ↓
데이터 송수신
   ↓
연결 종료
```

HTTP 역시 일반적으로 TCP 기반의 통신을 사용한다.

따라서 Apache, Nginx, JBoss 같은 서버 프로그램도 네트워크 통신 과정에서 TCP 소켓을 사용하게 된다.

---

## ss 명령어

Linux에서는 `ss` 명령어를 사용하여 현재 서버의 소켓 상태를 확인할 수 있다.

```bash
ss -lntp
```

옵션은 다음과 같다.

```text
-l
→ LISTEN 중인 소켓

-n
→ Port 등을 이름으로 변환하지 않고 숫자로 표시

-t
→ TCP 소켓

-p
→ 해당 소켓을 사용하는 프로세스 표시
```

즉:

```bash
ss -lntp
```

는 다음 의미로 이해할 수 있다.

```text
현재 서버에서
어떤 TCP 포트를
어떤 프로세스가 열고
연결을 기다리고 있는가?
```

---

## Process / Port / Socket 관계

운영 환경에서는 다음 관계를 이해하는 것이 중요하다.

```text
Process
   ↓
Socket 사용
   ↓
Port에서 LISTEN
   ↓
Client 연결
   ↓
데이터 송수신
```

예를 들어 Apache라면:

```text
httpd Process
     ↓
TCP Socket
     ↓
Port 8082
     ↓
LISTEN
     ↓
HTTP 요청 수신
```

---

## Java에서의 Socket

Java에서도 소켓을 직접 사용할 수 있다.

대표적인 클래스는 다음과 같다.

```text
ServerSocket
→ 서버에서 연결 요청을 기다릴 때 사용

Socket
→ 실제 상대방과 통신할 때 사용
```

개념적으로는 다음과 같다.

```text
ServerSocket
     ↓
특정 Port에서 대기
     ↓
Client 연결
     ↓
Socket 생성
     ↓
데이터 송수신
```

Spring Boot 개발에서는 웹 서버가 이러한 네트워크 처리를 담당하기 때문에 `Socket`이나 `ServerSocket`을 직접 사용할 일은 많지 않다.

하지만 내부적으로는 결국 이러한 소켓을 통해 네트워크 통신이 이루어진다.

---

## 정리

```text
IP
→ 어떤 서버인가?

Port
→ 해당 서버의 어떤 서비스인가?

Socket
→ 프로그램이 네트워크 통신에 사용하는 통신 창구

LISTEN
→ 서버가 특정 Port에서 연결을 기다리는 상태
```

서버 프로그램의 기본적인 동작은 다음과 같이 이해할 수 있다.

```text
서버 프로그램 실행
       ↓
Socket 생성
       ↓
Port 사용
       ↓
LISTEN
       ↓
Client 연결
       ↓
연결된 Socket을 통해 통신
```

Linux에서는 다음 명령어를 통해 LISTEN 중인 TCP 소켓과 프로세스를 확인할 수 있다.

```bash
ss -lntp
```

특정 Port를 확인하려면:

```bash
ss -lntp | grep -E ':<포트>'
```

백엔드 개발자의 운영/트러블슈팅 관점에서는 **Process → Socket → Port → LISTEN → Connection**의 관계를 이해하는 것이 핵심이다.
