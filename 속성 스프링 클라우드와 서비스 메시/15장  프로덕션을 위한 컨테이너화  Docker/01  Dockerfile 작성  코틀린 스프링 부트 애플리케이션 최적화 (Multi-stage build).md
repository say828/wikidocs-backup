## Dockerfile 작성: 코틀린 스프링 부트 애플리케이션 최적화 (Multi-stage build)

15장 00절에서 Docker의 '왜'를 이해했으니, 이제 '어떻게' 우리의 코틀린 스프링 부트 애플리케이션을 위한 효율적인 `Dockerfile`을 작성할지 알아볼 차례입니다. `Dockerfile`은 우리 애플리케이션 이미지를 만들기 위한 '레시피'입니다.

가장 단순한 `Dockerfile`은 자바(JDK)가 설치된 OS 이미지 위에, 우리 소스 코드를 전부 복사하고, 그 안에서 Gradle로 빌드하는 방식일 것입니다. 하지만 이 방식은 최종 이미지에 소스 코드, Gradle 캐시, 전체 JDK 등 **실행에 전혀 필요 없는 파일들**이 모두 포함되어, 이미지 크기가 수백 MB에서 GB 단위로 매우 커지는 문제를 야기합니다.

이미지가 크다는 것은 곧, 배포 시간이 길어지고, 저장 비용이 증가하며, 보안에 취약해진다는 의미입니다. 우리는 \*\*멀티 스테이지 빌드(Multi-stage build)\*\*라는 기법을 사용하여 이 문제를 해결할 것입니다.

-----

### 멀티 스테이지 빌드: "빌드 환경과 실행 환경의 분리"

멀티 스테이지 빌드는 하나의 `Dockerfile` 안에 여러 개의 `FROM` 구문을 사용하여, 빌드 과정을 여러 단계(Stage)로 나누는 기법입니다.

1.  **빌드(Build) 스테이지:** 첫 번째 스테이지는 '빌더(Builder)' 역할을 합니다. 여기에는 소스 코드를 컴파일하고 `.jar` 파일을 만드는 데 필요한 모든 도구(JDK, Gradle 등)를 포함합니다.
2.  **실행(Runtime) 스테이지:** 두 번째 스테이지는 '실행'만을 위한 최소한의 환경으로 구성됩니다. 여기에는 JDK가 아닌 \*\*JRE(Java Runtime Environment)\*\*와, 빌드 스테이지에서 생성된 **`.jar` 파일 딱 하나**만 복사해 옵니다.

이 과정을 통해, 최종적으로 만들어지는 이미지는 소스 코드나 빌드 도구 없이, 오직 애플리케이션 실행에 필요한 최소한의 파일만 담게 되어 매우 가볍고 효율적입니다.

-----

### 최적화된 `Dockerfile` 작성 (`member-service` 기준)

`member-service` 프로젝트의 루트 디렉토리에 다음과 같이 `Dockerfile`을 작성합니다.

```dockerfile
# =================================================================
# 1단계: 빌드(Build) 스테이지 - 애플리케이션 .jar 파일 생성
# =================================================================
# Gradle과 JDK 17이 포함된 이미지를 기반으로 시작
FROM gradle:8.5-jdk17-alpine AS builder

# 작업 디렉토리 설정
WORKDIR /app

# 빌드 캐시 효율을 위해, 먼저 의존성 관련 파일만 복사하여 의존성을 다운로드
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle
RUN gradle dependencies --build-cache || return 0

# 전체 소스 코드 복사
COPY . .

# Gradle을 사용하여 애플리케이션을 빌드하고 실행 가능한 .jar 파일 생성
# './gradlew bootJar'와 동일한 명령어
RUN gradle bootJar

# =================================================================
# 2단계: 실행(Runtime) 스테이지 - 최종 이미지 생성
# =================================================================
# 실행에 필요한 최소한의 환경인 JRE 이미지를 기반으로 시작 (Alpine Linux)
FROM eclipse-temurin:17-jre-alpine

# 작업 디렉토리 설정
WORKDIR /app

# 1단계(builder)에서 생성된 .jar 파일을 복사
# 빌드 컨텍스트 내의 build/libs/*.jar 파일을 app.jar 라는 이름으로 복사
COPY --from=builder /app/build/libs/*.jar app.jar

# 컨테이너가 시작될 때 실행할 명령어
# "java -jar app.jar"
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Dockerfile 설명

  * **`FROM gradle:8.5-jdk17-alpine AS builder`**: 첫 번째 '빌더' 스테이지를 정의합니다. `AS builder` 구문을 통해 이 스테이지에 `builder`라는 이름을 붙여줍니다.
  * **`COPY --from=builder ...`**: 두 번째 스테이지의 핵심입니다. `--from=builder` 옵션을 사용하여, 이전 `builder` 스테이지의 빌드 결과물(`build/libs/*.jar`)을 현재 스테이지로 복사해옵니다.
  * **`FROM eclipse-temurin:17-jre-alpine`**: 최종 이미지는 `alpine` 리눅스 기반의 JRE 이미지를 사용합니다. Alpine은 경량화된 리눅스 배포판으로, 이미지 크기를 극적으로 줄여줍니다.
  * **`ENTRYPOINT ["java", "-jar", "app.jar"]`**: 이 이미지가 컨테이너로 실행될 때, `java -jar app.jar` 명령어를 실행하여 스프링 부트 애플리케이션을 시작합니다.

-----

### Docker 이미지 빌드 및 실행

이제 이 `Dockerfile`을 사용하여 실제 도커 이미지를 빌드해 봅시다. `member-service` 프로젝트의 루트 디렉토리에서 다음 명령어를 실행합니다.

```bash
# 1. Docker 이미지 빌드
# -t 옵션: 이미지에 이름과 태그(tag)를 부여 (예: my-registry/member-service:0.0.1)
docker build -t member-service:0.0.1 .

# 2. 빌드된 이미지로 컨테이너 실행
# -p 옵션: 호스트의 8081 포트와 컨테이너의 8081 포트를 매핑
docker run -p 8081:8081 member-service:0.0.1
```

이 명령어를 실행하면, 우리의 `member-service`는 이제 완전히 격리된 컨테이너 환경에서 실행됩니다. 멀티 스테이지 빌드 덕분에 최종 이미지의 크기는 수백 MB가 아닌 수십 MB 수준으로 최적화되었으며, 이는 더 빠른 배포와 효율적인 자원 사용으로 이어집니다.

다음 절에서는 이 방식보다 더 간편하게 JVM 애플리케이션 컨테이너를 빌드할 수 있는 `Jib`이라는 도구를 알아보겠습니다.