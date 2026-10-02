# JBoss Request Size

## 핵심

JBoss EAP에서는 POST 요청의 최대 크기를 제한할 수 있다.

JBoss EAP 7.2의 Undertow Listener에서 `max-post-size`를 별도로 설정하지 않으면 기본값은 **10MB(10485760 bytes)**이다.

파일 업로드와 같이 POST 요청의 크기가 이 제한을 초과하면 요청이 정상적으로 처리되지 않을 수 있다.

엑셀 업로드 과정에서 해당 제한으로 문제가 발생했고, JBoss의 `ajp-listener`에 `max-post-size`를 설정하여 해결했다.

---

## 현재 확인된 요청 구조

현재까지 확인된 요청 구조는 다음과 같다.

```text
[앞단 구조 미확인]
        ↓
Apache
        ↓
mod_jk
        ↓
Load Balancer Worker
        ↓ AJP 1.3
JBoss EAP 7.2
        ↓
Application
```

Apache보다 앞단에 **Nginx, LB 등이 존재하는지는 아직 확인하지 않았다.**

따라서 전체 요청 구조를 확정하지 않고, 현재 확인된 **Apache → mod_jk → AJP → JBoss** 구간까지만 이해한다.

---

## WEB 서버 확인

WEB 서버에서 Apache 등의 웹서버 프로세스를 확인할 수 있다.

```bash
ps -ef | grep -E "httpd|apache|nginx"
```

실제 환경에서는 다음과 같이 `httpd` 프로세스가 실행되고 있는 것을 확인했다.

```text
<Apache_설치_경로>/bin/httpd -k start
```

### httpd

`httpd`는 **Apache HTTP Server를 실행하는 프로세스**이다.

`httpd`는 HTTP Daemon을 의미하며 클라이언트의 HTTP 요청을 받아 처리한다.

따라서 서버에서 `httpd` 프로세스가 실행되고 있다면 Apache HTTP Server가 실행되고 있다는 것을 파악할 수 있다.

여러 개의 `httpd` 프로세스가 나타날 수 있으며, Apache는 설정된 처리 방식에 따라 여러 프로세스 또는 스레드를 이용해 요청을 처리할 수 있다.

---

## Listen Port 확인 명령어

현재 서버에서 어떤 TCP 포트를 어떤 프로세스가 LISTEN하고 있는지 다음 명령어로 확인할 수 있다.

```bash
ss -lntp
```

- `ss` : 네트워크 소켓 상태를 확인하는 명령어 -> 현재 서버에서 어떤 포트를 어떤 프로세스가 사용하고 있는지 확인 

주요 옵션은 다음과 같다.

- `-l` : LISTEN 중인 소켓만 확인
- `-n` : 서비스 이름 대신 실제 포트 번호로 표시
- `-t` : TCP 소켓 확인
- `-p` : 해당 소켓을 사용하는 프로세스 확인

소켓에 대한 자세한 내용은 [[socket|Socket]] 참고

### 실제 환경에서 Apache Port 확인

WEB 서버가 일반적인 HTTP/HTTPS 포트인 `80`, `443`을 사용하고 있는지 확인하기 위해 다음 명령어를 실행했다.

```bash
ss -lntp | grep -E ':80|:443'
```

그런데 위 명령어의 `grep` 조건은 `80`, `443` 포트만 정확하게 검색하는 것이 아니다.

`:80`이라는 문자열이 포함되어 있으면 검색되기 때문에 다음과 같은 포트도 함께 조회될 수 있다.

```text
:80
:8080
:8081
:8082
:8083
```

실제 환경에서도 `8081`, `8082`, `8083` 등의 포트가 함께 조회되었고, 그중 `8082`에서 다음과 같이 `httpd` 프로세스를 확인했다.

```text
:::8082 ... users:(("httpd", ...))
```

즉 `8082` 포트를 처음부터 의도적으로 조회한 것은 아니고, `80`, `443` 포트를 확인하는 과정에서 `8082`가 함께 조회되면서 발견했다.

결과에서 다음 두 가지를 확인할 수 있다.

```text
:::8082
→ 8082 Port를 LISTEN 중

users:(("httpd", ...))
→ 해당 Socket을 httpd 프로세스가 사용 중
```

`httpd`는 Apache HTTP Server의 프로세스이므로 다음과 같이 연결할 수 있다.

```text
Apache(httpd)
      ↓
TCP Socket
      ↓
Port 8082
      ↓
LISTEN
```

따라서 현재 WEB 서버에서 **Apache가 8082 Port를 LISTEN하고 있다**는 것을 확인했다.

### 정확한 Port 검색

참고로 다음 명령어는 `:80`이라는 문자열을 포함하는 Port까지 조회한다.

```bash
ss -lntp | grep -E ':80|:443'
```

따라서 `8080`, `8081`, `8082` 등도 검색될 수 있다.

`80`, `443` Port만 정확하게 확인하려면 Port 번호 뒤의 공백까지 조건에 포함할 수 있다.

```bash
ss -lntp | grep -E ':(80|443)[[:space:]]'
```

이번 트러블슈팅에서는 다음과 같은 흐름으로 Apache의 Port를 확인했다.

```text
80 / 443 Port 확인
        ↓
808x Port도 함께 조회됨
        ↓
8082에서 httpd 발견
        ↓
httpd = Apache HTTP Server
        ↓
Apache가 8082 Port를
LISTEN하고 있음을 확인
```

---

## 요청 경로 추적 과정

아래는 문제 해결 과정에서 확인한 구성요소를 바탕으로 **WEB → WAS 요청 구조를 추가로 추적하고 이해한 과정**이다. 뒤의 `트러블슈팅 사례`는 증상에서 원인 후보를 좁혀 해결한 흐름이며, 아래 구조 추적 전체를 먼저 끝내야 `max-post-size`를 발견할 수 있었던 것은 아니다.

운영 환경을 처음부터 모두 알고 있지 않더라도 **현재 확인할 수 있는 정보에서 다음 질문을 만들어가는 방식**으로 요청 경로를 추적할 수 있다.

처음부터 `mod_jk`, `workers.properties`와 같은 구성요소나 설정 파일의 이름을 모두 알고 있을 필요는 없다.

### 1. 어떤 WEB 서버가 실행되고 있는지 확인

먼저 서버에서 실행 중인 프로세스를 확인한다.

```bash
ps -ef | grep -E "httpd|apache|nginx"
```

실제 환경에서는 다음과 같은 프로세스를 확인했다.

```text
<Apache_설치_경로>/bin/httpd -k start
```

여기서 `httpd`라는 이름을 모른다면 다음 질문으로 이어질 수 있다.

```text
httpd가 무엇인가?
        ↓
Apache HTTP Server의 프로세스
```

따라서 현재 WEB 서버에서 Apache가 실행되고 있다는 것을 파악할 수 있다.

### 2. Apache가 JBoss에 요청을 어떻게 전달하는지 확인

애플리케이션은 JBoss에서 실행되고 있는데 클라이언트 요청은 Apache가 받고 있다.

그러면 다음 질문으로 이어질 수 있다.

```text
Apache가 받은 요청을
JBoss에는 어떻게 전달하는가?
```

Apache와 WAS 사이의 연결 방식을 조사하면 Reverse Proxy, AJP, `mod_jk` 등의 개념을 확인할 수 있다.

실제 서버의 프로세스를 확인하는 과정에서도 다음과 같은 로그 설정이 발견되었다.

```text
<로그_경로>/jk/mod_jk.%Y%m%d.log
```

여기서 다시 다음 질문으로 이어질 수 있다.

```text
mod_jk가 무엇인가?
```

`mod_jk`는 Apache HTTP Server와 Tomcat/JBoss 등의 WAS를 **AJP 프로토콜로 연결할 때 사용하는 Apache 모듈**이다.

따라서 다음과 같은 가설을 세울 수 있다.

```text
Apache
   ↓
mod_jk
   ↓
AJP?
   ↓
JBoss
```

아직 이 단계에서는 실제 연결 설정을 확인한 것이 아니므로 설정을 통해 검증해야 한다.

### 3. mod_jk가 어느 WAS로 요청을 보내는지 확인

`mod_jk`가 Apache와 WAS를 연결한다는 것을 알게 되면 다음 질문으로 이어진다.

```text
mod_jk는 어느 WAS로
요청을 전달할지 어떻게 알 수 있는가?
```

어딘가에 WAS의 주소와 연결 방식 등을 관리하는 설정이 존재해야 한다.

`mod_jk`의 설정 방식을 확인하면 `workers.properties`라는 설정 파일을 사용한다는 것을 알 수 있다.

실제 서버에서 해당 파일을 찾는다.

```bash
find /app -name "workers.properties" 2>/dev/null
```

현재 Apache 인스턴스에서는 다음과 같은 위치에서 설정 파일을 확인했다.

```text
<Apache_설치_경로>/conf/workers.properties
```

즉 `workers.properties`라는 파일 이름을 처음부터 외워서 찾은 것이 아니라 다음과 같은 흐름으로 찾아갈 수 있다.

```text
Apache가 JBoss에 어떻게 요청을 전달하지?
        ↓
mod_jk라는 Apache 모듈 확인
        ↓
mod_jk는 WAS 정보를 어디서 관리하지?
        ↓
mod_jk 설정 방식 확인
        ↓
workers.properties 확인
        ↓
실제 서버에서 설정 파일 검색
```

### 4. workers.properties에서 실제 연결 방식 확인

설정 파일에서는 Load Balancer Worker를 확인했다.

```properties
worker.wlb.type=lb
```

또한 WAS와의 통신에 AJP 1.3을 사용하는 설정을 확인했다.

```properties
worker.template.type=ajp13
```

WAS 서버의 `host` 설정도 두 개 존재했다.

따라서 실제 환경에서 다음 구조를 사용하고 있다는 것을 확인할 수 있다.

```text
Apache
   ↓
mod_jk
   ↓
worker.wlb
(Load Balancer)
   ↓
AJP 1.3
   ↓
WAS 1 / WAS 2
```

### 5. JBoss에서는 AJP 요청을 어디서 받는지 확인

Apache에서 AJP 1.3을 이용하여 WAS로 요청을 전달한다는 것을 확인했다.

그러면 다음 질문으로 이어진다.

```text
JBoss에서는
AJP 요청을 어디서 받는가?
```

JBoss 설정을 확인하면 `ajp-listener`를 확인할 수 있다.

```xml
<ajp-listener
    name="ajp"
    record-request-start-time="true"
    max-post-size="41943040"
    max-header-size="41943040"
    socket-binding="ajp"/>
```

이를 통해 WEB 서버와 WAS의 설정을 연결할 수 있다.

```text
Apache
   ↓
mod_jk
   ↓
AJP 1.3
   ↓
JBoss ajp-listener
```

---

## 요청 경로 추적 시 중요한 점

운영 환경의 모든 구성요소와 설정 파일 이름을 외우고 있을 필요는 없다.

현재 확인되는 정보에서 **역할 → 연결 대상 → 설정 위치** 순서로 질문을 이어가며 추적할 수 있다.

```text
502 발생
 ↓
요청은 어떤 서버를 거치는가?
 ↓
WEB 서버에서는 어떤 프로세스가 실행되는가?
 ↓
httpd 발견
 ↓
httpd가 무엇인가?
 ↓
Apache HTTP Server
 ↓
Apache는 JBoss에 어떻게 요청을 전달하는가?
 ↓
mod_jk / AJP 확인
 ↓
우리 서버에서도 mod_jk를 사용하는가?
 ↓
mod_jk 로그 확인
 ↓
mod_jk는 WAS 정보를 어디서 관리하는가?
 ↓
workers.properties 확인
 ↓
실제 WAS host / AJP 설정 확인
 ↓
JBoss에서는 AJP를 어디서 받는가?
 ↓
ajp-listener 확인
```

중요한 것은 특정 설정 파일 이름을 모두 암기하는 것이 아니라 **모르는 구성요소가 나타났을 때 그 구성요소의 역할을 확인하고, 다음 연결 대상과 설정 위치를 추적하는 것**이다.

---

## Apache와 JBoss 연결 확인

### mod_jk

`mod_jk`는 Apache HTTP Server와 Tomcat/JBoss 등의 WAS를 AJP 프로토콜로 연결할 때 사용하는 Apache 모듈이다.

실제 환경에서는 WEB 서버의 프로세스를 확인하는 과정에서 다음과 같은 로그 설정을 확인했다.

```text
<로그_경로>/jk/mod_jk.%Y%m%d.log
```

이를 통해 Apache에서 `mod_jk`를 사용하고 있는 정황을 확인했다.

### workers.properties

`mod_jk`의 실제 연결 설정은 `workers.properties`에서 확인할 수 있다.

```text
<Apache_설치_경로>/conf/workers.properties
```

Load Balancer Worker가 존재한다.

```properties
worker.wlb.type=lb
```

WAS와의 통신에는 AJP 1.3을 사용한다.

```properties
worker.template.type=ajp13
```

WAS 서버의 `host` 설정도 두 개 존재했다.

현재 확인된 구조는 다음과 같다.

```text
Apache
   ↓
mod_jk
   ↓
worker.wlb
(Load Balancer)
   ↓
AJP 1.3
   ↓
WAS 1 / WAS 2
```

---

## JBoss AJP Listener

WAS에서는 JBoss EAP 7.2를 사용한다.

JBoss 설정에는 Apache에서 전달되는 AJP 요청을 받기 위한 `ajp-listener`가 존재한다.

```xml
<ajp-listener
    name="ajp"
    record-request-start-time="true"
    max-post-size="41943040"
    max-header-size="41943040"
    socket-binding="ajp"/>
```

WEB 서버의 `workers.properties`에서 `ajp13`을 사용하는 것을 확인했고, JBoss에서도 `ajp-listener`가 설정되어 있으므로 다음 연결 관계를 확인할 수 있다.

```text
Apache
   ↓
mod_jk
   ↓
AJP 1.3
   ↓
JBoss ajp-listener
```

---

## max-post-size

`max-post-size`는 POST 요청에서 허용할 최대 크기를 설정한다.

현재 설정은 다음과 같다.

```xml
max-post-size="41943040"
```

`41943040 bytes`는 약 **40MB**이다.

JBoss EAP 7.2에서는 `max-post-size`를 별도로 지정하지 않을 경우 기본값으로 약 **10MB**가 적용된다.

---

## 트러블슈팅 사례

### 증상

엑셀 파일 업로드 기능을 사용하는 과정에서 일정 크기 이상의 파일을 업로드하면 `502 Bad Gateway`가 발생했다.

작은 파일은 정상적으로 업로드되었지만 파일 크기가 커지면 문제가 발생했다.

### 원인 확인

작은 파일은 성공하고 큰 파일에서만 실패한다는 점을 단서로 **파일 크기와 관련된 문제일 것**이라는 가설을 세웠다.

여기서 "요청이 처리되는 과정 어딘가에 Request Body 크기 제한이 있는 것 아닐까?"라는 질문으로 이어졌고, WEB / WAS / Application 등 요청 처리 구간의 크기 제한 설정을 조사했다.

대용량 요청 문제에서는 다음과 같은 경로의 여러 구간을 조사 대상으로 생각할 수 있다.

```text
Client
   ↓
앞단 LB / Proxy (존재 여부 미확인)
   ↓
Apache
   ↓
mod_jk / AJP
   ↓
JBoss
   ↓
Application
```

이는 **Request Body / Upload Size 제한이 있는지 살펴볼 후보 구간**이며, 각 구간에 실제 제한이 있었다는 뜻은 아니다. Apache보다 앞단의 Nginx/LB 존재 여부는 아직 확인하지 않았고, 이번에 원인으로 확인한 것은 JBoss의 제한이다.

관련 설정을 조사하면서 JBoss AJP Listener에도 POST 요청 크기를 제한하는 **`max-post-size`** 설정이 있다는 것을 알게 되었다. 처음부터 이 설정 이름을 알고 찾아간 것은 아니다.

당시 `ajp-listener`에는 `max-post-size`가 없었다. 따라서 "설정하지 않으면 제한이 없는 것인가, 아니면 Default 값이 존재하는가?"를 확인하기 위해 사용 중인 **JBoss EAP 7.2의 기본값**을 조사했다.

그 결과 별도로 설정하지 않아도 기본 **10MB(10485760 bytes) 제한**이 적용된다는 것을 확인했다.

이후 문제가 발생한 엑셀 업로드 요청의 크기를 해당 기본값과 비교했고, 요청 크기가 이 제한을 초과하고 있음을 확인했다.

### 해결

JBoss의 `ajp-listener`에 `max-post-size`를 추가했다.

```xml
max-post-size="41943040"
```

POST 요청의 최대 크기를 **10MB → 40MB**로 변경했다.

설정 변경 후 기존에 실패하던 동일한 엑셀 파일로 재테스트했고, 정상적으로 업로드되는 것을 확인했다.

### 처리 흐름

```text
엑셀 업로드 실패
       ↓
작은 파일은 성공, 큰 파일은 실패
       ↓
파일 크기 관련 문제라는 가설 설정
       ↓
요청 처리 구간 어딘가의 Request Body 크기 제한 의심
       ↓
WEB / WAS / Application의 크기 제한 설정 조사
       ↓
JBoss AJP Listener의 POST 크기 제한 설정 발견
→ max-post-size
       ↓
현재 설정에 max-post-size가 없음
→ 미설정이면 무제한인가, Default 값이 있는가?
       ↓
JBoss EAP 7.2 기본값 조사 → 10MB 확인
       ↓
실패한 업로드 요청 크기가 기본 제한을 초과함을 확인
       ↓
max-post-size를 40MB로 설정
       ↓
동일한 엑셀 파일로 재테스트
       ↓
정상 업로드 확인
```

---

## 502 Bad Gateway에 대한 추가 확인

현재 확인된 것은 다음과 같다.

```text
Apache
   ↓
mod_jk
   ↓ AJP
JBoss
```

그리고 JBoss의 `max-post-size`를 변경한 뒤 엑셀 업로드 문제가 해결되었다.

다만 당시 `max-post-size` 제한을 초과했을 때 **왜 최종적으로 사용자에게 `502 Bad Gateway`가 반환되었는지에 대한 정확한 처리 과정은 아직 확인하지 않았다.**

따라서 다음과 같이 단정하지 않는다.

```text
max-post-size 초과
= 무조건 502 Bad Gateway
```

정확한 원인을 확인하려면 당시 Apache `mod_jk` 로그와 JBoss 로그 등을 추가로 확인해야 한다.

---

## 추가 확인 사항

확인 진행 상황은 [추가 확인 사항 목록](pending-checks.md#jboss-request-size)에서 관리한다.

현재 Apache보다 앞단의 구조는 확인하지 않았다.

```text
사용자
   ↓
Nginx / LB / 기타?   ← 미확인
   ↓
Apache              ← 확인
   ↓
mod_jk              ← 확인
   ↓
AJP 1.3             ← 확인
   ↓
JBoss EAP 7.2       ← 확인
```

추후 다음 내용을 추가로 확인할 수 있다.

- Apache 앞단에 Nginx가 존재하는지
- 별도의 L4/L7 Load Balancer가 존재하는지
- 외부 요청이 어떤 경로를 통해 Apache `8082`로 전달되는지
- `502 Bad Gateway`가 어느 구간에서 생성되었는지

---

## 정리

현재까지 확인한 핵심 구조는 다음과 같다.

```text
Apache
   ↓
mod_jk
   ↓
Load Balancer Worker
   ↓
AJP 1.3
   ↓
JBoss EAP 7.2
   ↓
ajp-listener
   ↓
Application
```

- WEB 서버에서 `httpd` 프로세스를 통해 Apache HTTP Server가 실행되고 있음을 확인했다.
- Apache가 `8082` 포트를 LISTEN하고 있다.
- Apache가 JBoss와 어떻게 통신하는지 추적하는 과정에서 `mod_jk`를 확인했다.
- `mod_jk`의 연결 설정을 추적하여 `workers.properties`를 확인했다.
- `workers.properties`에서 Load Balancer Worker와 AJP 1.3 사용을 확인했다.
- JBoss에서는 `ajp-listener`를 통해 AJP 요청을 받는다.
- `max-post-size`는 POST 요청의 최대 허용 크기를 설정한다.
- 기본 10MB 제한을 40MB로 변경하여 엑셀 업로드 문제를 해결했다.
- Apache보다 앞단의 Nginx/LB 존재 여부는 아직 확인하지 않았다.
- `502 Bad Gateway`가 반환된 정확한 과정 역시 추가 확인이 필요하다.
