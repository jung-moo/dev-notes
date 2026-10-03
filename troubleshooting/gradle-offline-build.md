
# Gradle 폐쇄망 환경에서 오프라인 빌드 구성

## 문제

외부망에 접근할 수 없는 PC에서 Gradle 프로젝트를 빌드하려 했으나,
외부 Repository에서 Dependency를 다운로드할 수 없어 빌드할 수 없었다.

### 환경

- PC1: 외부망 접근 가능
- PC2: 외부망 접근 불가
- PC1, PC2 모두 동일한 프로젝트
- Gradle Wrapper 사용

---

## 원인

Gradle은 빌드에 필요한 Dependency와 Plugin 등을 외부 Repository에서 다운로드한다.

PC2는 외부망에 접근할 수 없기 때문에 필요한 파일이 로컬 Gradle Cache에 존재하지 않으면
Dependency Resolution이 불가능하다.

또한 기본 Gradle User Home을 그대로 전달할 경우,

```text
C:\Users\<사용자>\.gradle
```

여러 프로젝트에서 사용한 Cache가 함께 저장되어 있어 불필요한 파일까지 전달하게 된다.

---

## 해결

### 1. PC1에서 프로젝트 전용 Gradle Cache 생성

기존 Gradle Cache와 분리하기 위해 `--gradle-user-home`을 사용하여
전달용 Gradle User Home을 별도로 지정했다.

프로젝트 Root Directory에서 실행:

```bash
./gradlew clean build --gradle-user-home <전달용_Gradle_User_Home>
```

예:

```bash
./gradlew clean build --gradle-user-home D:\gradle-transfer\project-gradle-cache
```

이를 통해 해당 프로젝트를 빌드하는 데 필요한 Dependency와 Gradle 관련 Cache를
별도의 디렉터리에 다운로드했다.

### 2. Gradle Cache를 PC2로 전달

PC1에서 생성한 전달용 Gradle User Home을 압축하여 PC2로 전달했다.
다행히도 깃헙 등 전달 방법이 몇 가지 있었다.

### 3. PC2에서 Offline Build

전달받은 Gradle User Home을 지정하고 `--offline` 옵션으로 빌드했다.

```bash
./gradlew clean build --offline --gradle-user-home <전달받은_Gradle_User_Home>
```

`--offline`
- 외부 Repository에 접근하지 않고 로컬 Cache만 사용

`--gradle-user-home`
- 해당 Gradle 실행에서 사용할 Gradle User Home을 명시적으로 지정

### 4. IntelliJ Gradle 연동

CLI에서는 Offline Build에 성공했지만 IntelliJ에서 Gradle 연동에 실패했다.

CLI에서 지정한 `--gradle-user-home`은 해당 명령 실행에만 적용되므로,
IntelliJ가 동일한 Cache를 사용하도록 별도로 설정했다.

```text
Settings
→ Build, Execution, Deployment
→ Build Tools
→ Gradle
→ Gradle user home
```

Gradle User Home:

```text
<전달받은_Gradle_User_Home>
```

으로 변경했다.

---

## 확인

PC2에서 다음 명령으로 외부망 연결 없이 빌드되는 것을 확인했다.

```bash
./gradlew clean build --offline --gradle-user-home <전달받은_Gradle_User_Home>
```

결과:

```text
BUILD SUCCESSFUL
```

IntelliJ의 Gradle User Home도 동일한 경로로 설정한 후
Gradle Sync 및 프로젝트 연동이 정상적으로 동작하는 것을 확인했다.

---

## 정리

```text
PC1 (외부망 가능)
    ↓
프로젝트 전용 Gradle User Home 지정
    ↓
프로젝트 Build 및 Dependency 다운로드
    ↓
Gradle Cache를 PC2로 전달
    ↓
PC2에서 --offline + --gradle-user-home으로 Build
    ↓
IntelliJ Gradle User Home 동일하게 설정
    ↓
Offline Build / IntelliJ Gradle Sync 정상 동작
```

### 핵심

- `.gradle` 전체를 전달하지 않고 프로젝트 전용 Gradle User Home을 생성하여 전달할 수 있다.
- `--gradle-user-home`은 Gradle이 사용할 Cache 위치를 지정한다.
- `--offline`은 외부 Repository 접근 없이 로컬 Cache만 사용하도록 한다.
- CLI의 `--gradle-user-home` 설정은 IntelliJ에 자동으로 적용되지 않는다.
- IntelliJ에서도 동일한 Gradle User Home을 사용하도록 별도로 설정해야 한다.