## 02\. 실전 IaC 프로젝트 구조화: 모듈, 상태 관리, 그리고 협업

AWS CDK라는 강력한 검을 손에 쥐었다고 해서 곧바로 전장으로 뛰어들어서는 안 된다. 숙련된 검객이 검을 다루는 법식과 자세를 먼저 익히듯, 우리도 CDK를 효과적으로 사용하기 위한 체계적인 **프로젝트 구조**를 먼저 정립해야 한다. 코드가 수만 라인으로 늘어나고 여러 명의 개발자가 협업하는 상황에서 잘 정의된 구조가 없다면, IaC 프로젝트는 애플리케이션 코드베이스가 겪는 것과 똑같은 혼돈, 즉 '스파게티 인프라 코드'의 늪에 빠지게 된다.

훌륭한 IaC 프로젝트 구조는 단순히 파일들을 보기 좋게 정리하는 행위가 아니다. 그것은 아키텍처의 논리적 구성 요소를 코드로 명확하게 표현하고, 변경의 영향을 최소화하며, 팀의 협업 효율을 극대화하는 엔지니어링 규율이다. 이 절에서는 대규모 프로젝트에서도 흔들리지 않는 견고한 IaC 프로젝트를 위한 세 가지 핵심 원칙—**모듈화, 상태 분리, 그리고 협업 워크플로우**—에 대해 논하고, 이 책 전체에서 사용할 표준 디렉토리 구조를 제시할 것이다.

### 원칙 1: 모듈화 (Modularity) - 레고 블록처럼 아키텍처 조립하기

모든 것을 하나의 거대한 스택(Stack) 파일에 욱여넣는 것은 모놀리식 애플리케이션을 작성하는 것과 같은 실수다. 우리는 복잡한 아키텍처를 작고, 재사용 가능하며, 독립적으로 관리할 수 있는 논리적 단위로 분해해야 한다. CDK에서는 이러한 모듈화의 단위로 \*\*스택(Stack)\*\*과 \*\*구성(Construct)\*\*이라는 두 가지 강력한 개념을 제공한다.

  * **스택 (Stack):** 함께 배포되고 관리되는 AWS 리소스의 묶음이다. 예를 들어, 'NextGen Commerce'의 네트워크 인프라(VPC, 서브넷 등)는 `CoreNetworkStack`으로, 상품 서비스에 필요한 모든 리소스(ECS 클러스터, Fargate 서비스, DynamoDB 테이블 등)는 `ProductServiceStack`으로 분리할 수 있다. 스택 간의 의존성(예: 서비스 스택이 네트워크 스택을 참조)을 명시적으로 정의할 수 있으며, 이는 배포 순서를 보장하고 아키텍처의 관계를 명확하게 만든다.
  * **구성 (Construct):** 하나 이상의 AWS 리소스를 캡슐화하여 만든 재사용 가능한 아키텍처 패턴이다. 예를 들어, 우리 회사의 모든 마이크로서비스가 공통적으로 가져야 할 'ALB + Fargate 서비스 + Auto Scaling + CloudWatch 대시보드'라는 표준 패턴이 있다면, 이를 `SecureFargateService`라는 커스텀 Construct로 만들 수 있다. 이를 통해 개발자들은 내부 구현의 복잡성을 신경 쓸 필요 없이, 단 몇 줄의 코드로 회사의 보안 및 운영 표준을 준수하는 서비스를 손쉽게 생성할 수 있다.

### 원칙 2: 환경 분리 (Environment Isolation) - 설정은 코드가 아니다

개발(dev), 스테이징(staging), 프로덕션(production) 환경은 목적과 요구사항이 다르다. 프로덕션 환경은 높은 가용성을 위해 다중 AZ(Multi-AZ) 구성과 더 큰 인스턴스 타입을 사용해야 하지만, 개발 환경은 비용을 절감하기 위해 단일 AZ와 작은 인스턴스를 사용해야 한다. 이러한 환경별 차이점을 인프라 코드 내에 하드코딩하는 것은 최악의 안티패턴이다.

설정(Configuration)은 코드(Code)와 명확히 분리되어야 한다. 우리는 환경별 설정 파일(예: `config/dev.ts`, `config/prod.ts`)을 외부에 두고, CDK 애플리케이션의 진입점(Entrypoint)에서 현재 배포하려는 환경에 맞는 설정 객체를 동적으로 주입하는 방식을 사용해야 한다.

```typescript
// bin/nextgen-commerce-iac.ts (간략화된 예제)
import { devConfig } from '../config/dev-config';
import { prodConfig } from '../config/prod-config';
import { ProductServiceStack } from '../lib/stacks/product-service-stack';

const app = new cdk.App();

// 'cdk deploy -c env=dev ProductServiceStack' 와 같이 실행
const env = app.node.tryGetContext('env'); // 'dev' or 'prod'
const config = env === 'prod' ? prodConfig : devConfig;

new ProductServiceStack(app, `ProductServiceStack-${config.envName}`, {
  env: config.awsEnv,
  cpu: config.fargate.cpu, // 설정 파일에서 값을 주입
  memoryLimitMiB: config.fargate.memory,
});
```

이러한 접근 방식은 동일한 인프라 로직 코드(스택 정의)를 재사용하면서도, 각 환경에 맞는 인프라를 안전하고 일관되게 배포할 수 있도록 보장한다.

### 원칙 3: 협업 워크플로우 - GitOps 기반의 인프라 변경

코드로 관리되는 인프라의 최종 목표는 **Git 리포지토리**를 시스템의 \*\*단일 진실 공급원(Single Source of Truth)\*\*으로 만드는 것이다. 모든 인프라 변경은 반드시 Git을 통한 코드 변경과 코드 리뷰(Code Review) 프로세스를 거쳐야 한다.

1.  **브랜치 생성:** 개발자는 인프라 변경을 위해 새로운 기능 브랜치(feature branch)를 생성한다.
2.  **코드 수정:** CDK 코드를 수정하여 원하는 변경 사항(예: 새 S3 버킷 추가)을 반영한다.
3.  **변경 사항 검토:** Pull Request(PR)를 생성한다. 이때, CI/CD 파이프라인이 자동으로 `cdk diff` 명령을 실행하여 이 변경으로 인해 실제 AWS 환경에 어떤 변화가 생길지(생성/수정/삭제)를 PR 코멘트로 남겨준다.
4.  **동료 검토:** 다른 팀원들은 코드의 논리뿐만 아니라, `cdk diff`의 결과를 보고 인프라 변경의 타당성과 잠재적 위험을 함께 검토한다.
5.  **배포:** PR이 승인되고 메인 브랜치에 병합(merge)되면, CD 파이프라인이 자동으로 `cdk deploy` 명령을 실행하여 실제 환경에 변경 사항을 안전하게 적용한다.

이 워크플로우는 누가, 언제, 왜 인프라를 변경했는지에 대한 완벽한 감사 추적(Audit Trail)을 제공하며, 동료 검토를 통해 치명적인 실수를 사전에 방지하는 안전망 역할을 한다.

### NextGen Commerce를 위한 표준 프로젝트 구조

이상의 원칙들을 바탕으로, 이 책 전체에서 사용할 AWS CDK(TypeScript) 프로젝트의 표준 디렉토리 구조는 다음과 같다.

```plaintext
nextgen-commerce-iac/
├── bin/
│   └── nextgen-commerce-iac.ts  # 🚀 CDK 앱 진입점, 스택 인스턴스화 및 설정 주입
├── lib/
│   ├── stacks/                  # 🏗️ 개별 서비스 또는 리소스 그룹의 스택 정의
│   │   ├── core-network-stack.ts
│   │   └── product-service-stack.ts
│   └── constructs/              # 🧱 재사용 가능한 아키텍처 패턴 (커스텀 Construct)
│       └── secure-fargate-service.ts
├── config/
│   ├── dev-config.ts            # ⚙️ 개발 환경 설정
│   ├── prod-config.ts           # 🔒 프로덕션 환경 설정
│   └── types.ts                 # 📝 공통 설정 타입 정의
├── test/                          # 🧪 스냅샷 테스트 및 유닛 테스트
│   └── product-service-stack.test.ts
├── .github/workflows/             # 🤖 CI/CD 파이프라인 정의 (GitHub Actions)
│   └── deploy.yml
├── cdk.json
├── package.json
└── tsconfig.json
```

이 구조는 단순한 시작점이 아니다. 이것은 복잡성이 증가하더라도 질서를 유지하고, 팀의 생산성을 보호하며, 아키텍처를 지속적으로 진화시킬 수 있는 견고한 발판이다. 이제 이 발판 위에서, 우리의 첫 번째 아키텍처 베이스라인을 코드로 구축할 준비가 되었다.