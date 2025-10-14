## CI(Continuous Integration): GitHub Actions를 이용한 자동 빌드 및 테스트

CI/CD 파이프라인의 첫 번째 단계이자 가장 중요한 심장은 \*\*CI(Continuous Integration, 지속적인 통합)\*\*입니다. CI는 개발자가 코드 변경 사항을 중앙 저장소(GitHub)에 푸시(Push)할 때마다, **자동화된 프로세스가 코드를 빌드하고 모든 테스트를 실행**하여 코드의 품질과 안정성을 검증하는 활동입니다.

이 자동화된 '검증 게이트'를 통해, 우리는 여러 개발자의 작업물이 통합될 때 발생할 수 있는 잠재적인 문제를 조기에 발견하여 항상 안정적인 코드베이스를 유지할 수 있습니다.

-----

### 왜 Jenkins가 아닌 GitHub Actions인가?

전통적으로 CI 서버의 최강자는 `Jenkins`였습니다. Jenkins는 매우 유연하고 강력하지만, 별도의 서버를 구축하고, 수많은 플러그인을 설치하며, 보안 설정을 하는 등 상당한 '관리 비용'이 발생합니다.

반면, **GitHub Actions**는 GitHub에 완벽하게 통합된 최신 CI/CD 플랫폼입니다.

  * **저장소 내 관리 (Pipeline as Code):** CI/CD 파이프라인의 모든 정의를 `.github/workflows` 디렉토리 아래의 `YAML` 파일로 코드와 함께 관리합니다.
  * **서버리스 (Serverless):** GitHub이 빌드 서버(Runner)를 직접 제공하고 관리해주므로, 별도의 CI 서버를 구축할 필요가 없습니다.
  * **쉬운 사용법:** `actions/checkout`, `actions/setup-java` 등 커뮤니티에 의해 검증된 수많은 재사용 가능한 '액션(Action)' 덕분에 파이프라인을 매우 쉽게 구성할 수 있습니다.

우리 프로젝트는 GitHub를 중심으로 진행되므로, GitHub Actions는 가장 자연스럽고 효율적인 선택입니다.

-----

### GitHub Actions 워크플로우(Workflow) 구축하기

`member-service`를 위한 CI 워크플로우를 만들어 봅시다. 이 워크플로우는 `main` 브랜치에 `push` 또는 `pull_request` 이벤트가 발생할 때마다 실행되어, 프로젝트를 빌드하고 모든 테스트(단위, 통합, 계약)를 수행할 것입니다.

#### 1\. 워크플로우 파일 생성

`member-service` 프로젝트의 루트에 `.github/workflows/ci.yml` 파일을 생성합니다.

```yaml
# .github/workflows/ci.yml

# 1. 워크플로우의 이름
name: Member Service CI

# 2. 워크플로우를 트리거할 이벤트 정의
on:
  push:
    branches: [ "main" ] # main 브랜치에 push될 때
  pull_request:
    branches: [ "main" ] # main 브랜치로 PR이 생성/수정될 때

# 3. 실행될 작업(Job)들의 묶음
jobs:
  # 4. 'build-and-test'라는 이름의 Job 정의
  build-and-test:
    # 5. Job이 실행될 가상 환경(Runner) 지정
    runs-on: ubuntu-latest
    
    # 6. Job 내부에서 실행될 단계(Step)들
    steps:
      # 7. actions/checkout@v4: Git 저장소의 코드를 Runner로 체크아웃
      - name: Checkout code
        uses: actions/checkout@v4

      # 8. actions/setup-java@v4: JDK 17 설치
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      # 9. (성능 최적화) Gradle 캐시 설정
      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-

      # 10. gradlew 파일에 실행 권한 부여
      - name: Grant execute permission for gradlew
        run: chmod +x ./gradlew

      # 11. Gradle로 빌드 및 테스트 실행
      - name: Build and Test with Gradle
        run: ./gradlew build
```

-----

### CI 파이프라인의 실제 동작

이제 개발자가 코드를 수정하고 `main` 브랜치로 Pull Request를 생성하면, GitHub은 이 `ci.yml` 파일을 읽어 자동으로 다음 과정을 수행합니다.

1.  GitHub은 가상 서버(Ubuntu Runner)를 하나 준비합니다.
2.  워크플로우의 각 `step`을 순서대로 실행합니다.
      * 소스 코드를 체크아웃하고, JDK 17을 설치합니다.
      * `./gradlew build` 명령어를 통해 애플리케이션 코드를 컴파일하고, 14장에서 작성한 모든 테스트 코드(`Kotest`, `Testcontainers`, `Spring Cloud Contract` 등)를 실행합니다.
3.  모든 과정이 성공하면, PR 페이지에 **녹색 체크 표시** ✅ 가 나타납니다. 이제 이 코드는 안전하게 병합(Merge)될 수 있음을 의미합니다.
4.  만약 테스트 하나라도 실패하면, **빨간색 X 표시** ❌ 가 나타나고, 개발자는 즉시 무엇이 잘못되었는지 피드백을 받아 코드를 수정해야 합니다.

이 CI 파이프라인을 통해, 우리는 `main` 브랜치가 항상 빌드 가능하고, 테스트를 통과하는 '안정적인 상태'로 유지됨을 보장받게 됩니다. 이는 여러 개발자가 동시에 협업하는 MSA 프로젝트의 코드 품질을 유지하는 가장 기본적인 안전장치입니다.