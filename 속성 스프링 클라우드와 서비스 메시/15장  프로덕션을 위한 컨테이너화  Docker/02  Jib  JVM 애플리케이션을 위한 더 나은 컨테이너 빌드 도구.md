## Jib: JVM 애플리케이션을 위한 더 나은 컨테이너 빌드 도구

15장 01절에서 우리는 `Dockerfile`과 멀티 스테이지 빌드를 사용하여 컨테이너 이미지를 최적화했습니다. 이 방식은 매우 강력하고 보편적이지만, **JVM(Java/Kotlin) 애플리케이션**에 한해서는 더 좋고 간편한 대안이 존재합니다. 바로 구글이 만든 **Jib**입니다.

Jib은 `Dockerfile`을 작성하거나, 심지어 로컬 머신에 **도커 데몬(Docker Daemon)을 설치할 필요 없이**, 자바/코틀린 애플리케이션을 컨테이너 이미지로 빌드하고 푸시할 수 있게 해주는 Maven/Gradle 플러그인입니다.

-----

### 왜 Dockerfile보다 Jib이 더 나은가? 🤔

Jib은 JVM 애플리케이션의 구조를 깊이 이해하고 있기 때문에, `Dockerfile`로는 구현하기 까다로운 여러 최적화를 자동으로 수행합니다.

#### 1\. 도커 데몬 불필요 & Dockerfile 제거

Jib은 도커 데몬과 통신하지 않고, 자바 코드만으로 이미지 레이어를 생성하고 레지스트리로 직접 푸시합니다. 이는 다음과 같은 장점을 가집니다.

  * **CI/CD 환경 단순화:** CI 서버에 도커를 설치하고 설정하는 복잡한 과정이 필요 없어집니다.
  * **보안 강화:** 도커 데몬의 소켓 권한 문제 등에서 자유로워집니다.
  * **`Dockerfile` 제거:** 더 이상 `Dockerfile`을 작성하고 유지보수할 필요가 없습니다. 모든 빌드 설정은 `build.gradle.kts` 파일 안에서 코드로 관리됩니다.

#### 2\. 폭발적으로 빠른 빌드 (지능적인 레이어 분리) ⚡

`Dockerfile`을 사용한 빌드는 애플리케이션 코드 한 줄만 변경되어도, 거대한 애플리케이션 `.jar` 파일 전체가 포함된 레이어를 처음부터 다시 빌드하고 푸시해야 합니다. 이는 매우 비효율적입니다.

Jib은 이 문제를 해결하기 위해, JVM 애플리케이션을 **논리적인 단위로 자동 분리**하여 여러 개의 레이어로 만듭니다.

  * **1 레이어 (Dependencies):** 거의 변하지 않는 외부 라이브러리 (spring-boot-starter-web 등)
  * **2 레이어 (Resources):** 가끔 변하는 리소스 파일 (`application.yml` 등)
  * **3 레이어 (Classes):** **가장 자주 변하는** 우리 애플리케이션의 컴파일된 `.class` 파일

이러한 지능적인 분리 덕분에, 우리가 비즈니스 로직 코드(`*.kt`)만 수정했을 경우, Jib은 **3번 레이어(Classes)만 다시 빌드하고 푸시**합니다. 수백 MB에 달하는 1번 레이어(Dependencies)는 캐시된 버전을 그대로 재사용하므로, 빌드 속도가 놀랍도록 빨라집니다.

#### 3\. 재현 가능한 빌드 (Reproducible Builds)

`Dockerfile`로 이미지를 빌드하면, 빌드 시점에 따라 파일의 타임스탬프 등이 달라져 내용이 동일하더라도 이미지의 해시(Digest)값이 매번 바뀔 수 있습니다. Jib은 파일 타임스탬프를 고정하는 등, 소스 코드가 변경되지 않았다면 **언제, 어디서 빌드하든 항상 100% 동일한 이미지**를 생성하는 것을 보장합니다.

-----

### Jib 플러그인 설정 및 사용법

`member-service` 프로젝트에 Jib을 적용해 봅시다.

#### 1\. `build.gradle.kts`에 플러그인 추가

```kotlin
// member-service/build.gradle.kts
plugins {
    // ...
    id("com.google.cloud.tools.jib") version "3.4.2" // Jib 플러그인 추가
}

// ...

// Jib 플러그인 설정
jib {
    // 1. 이미지를 보낼 타겟(Target) 설정
    to {
        // 빌드된 이미지가 저장될 레지스트리와 이름
        image = "my-docker-hub-id/member-service"
        // 태그 설정 (버전)
        tags = setOf("0.0.1", "latest")
    }
    // (Private 레지스트리일 경우, from.auth 또는 to.auth에 인증 정보 설정)
    container {
        // JVM 옵션 등 컨테이너 실행 환경 설정
        jvmFlags = listOf("-Duser.timezone=Asia/Seoul")
        mainClass = "com.ecommerce.member.MemberApplication"
    }
}
```

#### 2\. Jib으로 이미지 빌드 및 푸시

이제 `Dockerfile`을 작성할 필요도, `docker build` 명령어를 사용할 필요도 없습니다. 다음 Gradle 태스크 하나면 모든 것이 끝납니다.

```bash
# 로컬 도커 데몬으로 이미지 빌드 (테스트용)
./gradlew jibDockerBuild

# 설정된 원격 레지스트리로 이미지 빌드 및 푸시
# (docker hub 등에 로그인 되어 있어야 함)
./gradlew jib
```

이 명령어 한 번으로, Jib은 애플리케이션을 빌드하고, 최적화된 레이어로 분리하여 컨테이너 이미지를 생성한 뒤, 지정된 도커 레지스트리로 푸시까지 완료합니다.

결론적으로, Jib은 `Dockerfile`을 직접 관리하는 복잡함을 제거하고, JVM 애플리케이션의 특성을 100% 활용하여 더 빠르고, 더 효율적이며, 더 안정적인 컨테이너 빌드 경험을 제공하는 강력한 도구입니다. 우리의 모든 스프링 부트 서비스는 이 Jib을 표준 빌드 방식으로 채택할 것입니다.