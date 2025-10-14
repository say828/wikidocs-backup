## 03\. 이 책의 표준: AWS CDK(TypeScript)를 활용한 베이스라인 구축

이론과 원칙, 그리고 잘 설계된 프로젝트 구조까지 갖추었다. 이제는 코드로 증명할 시간이다. 이 절에서는 우리가 앞서 정의한 표준 프로젝트 구조 위에서 'NextGen Commerce' 플랫폼의 가장 근간이 되는 \*\*아키텍처 베이스라인(Architecture Baseline)\*\*을 AWS CDK를 사용하여 직접 구축한다.

이 베이스라인은 앞으로 우리가 지을 모든 마이크로서비스라는 건축물을 떠받칠 단단한 대지와 같다. 여기에는 우리의 클라우드 데이터 센터가 될 \*\*VPC(Virtual Private Cloud)\*\*와 그 안의 혈관 역할을 할 **서브넷(Subnet)**, 그리고 외부와의 통신을 제어할 \*\*게이트웨이(Gateway)\*\*들이 포함된다. 우리는 이 과정을 통해 CDK의 고수준 추상화(High-level Abstraction)가 어떻게 단 몇 줄의 코드로 Well-Architected 원칙(특히 안정성과 보안)을 만족시키는 복잡한 네트워크를 생성하는지 직접 목격하게 될 것이다.

이것은 단순한 코드 예제가 아니다. 지금부터 작성하는 이 코드는 이 책의 모든 후속 챕터에서 사용될 실질적인 출발점이자, 당신이 현업에서 즉시 적용할 수 있는 재사용 가능한 패턴이다.

### 1\. 베이스라인의 핵심: CoreNetworkStack 정의하기

우리의 첫 번째 목표는 안정성의 핵심인 고가용성(High Availability)을 보장하는 네트워크 환경을 구축하는 것이다. 이를 위해 우리는 최소 2개 이상의 가용 영역(Availability Zone, AZ)에 걸쳐 리소스를 분산 배치할 것이다. 또한, 외부 인터넷에서 직접 접근할 수 있는 \*\*퍼블릭 서브넷(Public Subnet)\*\*과 데이터베이스처럼 내부적으로만 접근해야 하는 리소스를 위한 \*\*프라이빗 서브넷(Private Subnet)\*\*으로 네트워크를 격리하여 보안을 강화한다.

이 모든 요구사항을 `lib/stacks/core-network-stack.ts` 파일에 `CoreNetworkStack`이라는 이름의 스택으로 정의해 보자.

```typescript
// lib/stacks/core-network-stack.ts

import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export class CoreNetworkStack extends cdk.Stack {
  // 다른 스택에서 이 VPC를 참조할 수 있도록 public readonly 속성으로 선언
  public readonly vpc: ec2.Vpc;

  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // 💡 CDK의 고수준 Construct인 ec2.Vpc를 사용한다.
    // 이 단 하나의 Construct가 VPC, 서브넷, 라우팅 테이블,
    // 인터넷 게이트웨이, NAT 게이트웨이까지 모두 생성한다.
    this.vpc = new ec2.Vpc(this, 'NextGenCommerceVPC', {
      // VPC가 사용할 IP 주소 대역을 정의 (10.0.0.0/16)
      ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),

      // 이 VPC를 최대 2개의 가용 영역(AZ)에 걸쳐 생성
      // 이는 안정성 원칙의 핵심인 'SPOF 제거'를 위한 설계
      maxAzs: 2,

      // 서브넷 구성을 정의
      subnetConfiguration: [
        {
          // 'Public' 이라는 이름의 서브넷 그룹을 생성
          name: 'Public',
          // 퍼블릭 서브넷으로 지정. 이 서브넷의 리소스는 인터넷에서 직접 접근 가능
          subnetType: ec2.SubnetType.PUBLIC,
          // CIDR 블록의 서브넷 마스크를 24로 지정
          cidrMask: 24,
        },
        {
          // 'Private' 이라는 이름의 서브넷 그룹을 생성
          name: 'Private',
          // 프라이빗 서브넷으로 지정. 이 서브넷의 리소스는 외부에서 직접 접근 불가
          // NAT 게이트웨이를 통해 외부로 나가는 통신(예: 패치 다운로드)만 가능
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
          cidrMask: 24,
        },
      ],
    });

    // 스택의 결과물(Output)로 VPC ID를 출력하여 확인 용이성을 높임
    new cdk.CfnOutput(this, 'VpcId', {
      value: this.vpc.vpcId,
      description: 'The ID of the VPC',
    });
  }
}
```

위 코드가 바로 CDK의 강력함을 보여주는 대표적인 예시다. 우리는 단지 "2개의 AZ에 걸쳐 퍼블릭과 프라이빗 서브넷을 가진 VPC를 원한다"고 **선언**했을 뿐이다. 하지만 이 코드를 실행하면 CDK는 내부적으로 수십 개의 리소스(VPC, 6개의 서브넷, 라우팅 테이블, 인터넷 게이트웨이, 2개의 NAT 게이트웨이 등)를 생성하고 이들 간의 복잡한 라우팅 관계를 AWS 모범 사례에 맞게 자동으로 구성해 준다. 만약 이를 클라우드포메이션으로 직접 작성했다면 수백 줄의 코드가 필요했을 것이다.

### 2\. CDK 앱 진입점에서 스택 실행하기

이제 방금 정의한 `CoreNetworkStack`을 실제 배포 가능한 단위로 만들기 위해, `bin/nextgen-commerce-iac.ts` 파일에서 이 스택의 인스턴스를 생성해야 한다.

```typescript
#!/usr/bin/env node
// bin/nextgen-commerce-iac.ts

import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { CoreNetworkStack } from '../lib/stacks/core-network-stack';

const app = new cdk.App();

// 환경 변수나 cdk.json에 정의된 AWS 계정 및 리전 정보를 사용
const env = {
  account: process.env.CDK_DEFAULT_ACCOUNT,
  region: process.env.CDK_DEFAULT_REGION
};

// 우리의 베이스라인 네트워크 스택을 인스턴스화
new CoreNetworkStack(app, 'CoreNetworkStack-Dev', {
  env: env,
  description: 'Core network infrastructure for the Dev environment',
});
```

### 3\. 베이스라인 배포 및 검증

이제 모든 준비가 끝났다. 터미널에서 다음 명령어를 실행하여 우리의 첫 번째 인프라를 클라우드에 실제로 배포해 보자.

1.  **AWS 환경 준비 (최초 1회):** CDK를 사용하여 배포하려면, 대상 AWS 계정과 리전에 CDK가 필요한 자원(예: CloudFormation 템플릿을 저장할 S3 버킷)을 생성해야 한다. 이 과정을 `bootstrap`이라고 한다.

    ```bash
    cdk bootstrap
    ```

2.  **CloudFormation 템플릿 합성:** `cdk synth` 명령을 실행하면, 우리가 작성한 TypeScript 코드가 최종적으로 어떤 CloudFormation 템플릿으로 변환되는지 눈으로 직접 확인할 수 있다. 이는 코드가 실제로 어떤 리소스를 생성할지 검토하는 데 매우 유용하다.

    ```bash
    cdk synth CoreNetworkStack-Dev
    ```

3.  **베이스라인 배포:** 이제 `cdk deploy` 명령으로 실제 배포를 진행한다. CDK는 변경 사항을 적용하기 전에 어떤 리소스가 생성될 것이며, IAM 정책 등 보안에 민감한 변경이 있는지 요약해서 보여주고 사용자의 동의를 구한다.

    ```bash
    cdk deploy CoreNetworkStack-Dev
    ```

    배포가 성공적으로 완료되면, AWS Management Console의 VPC 섹션에 `NextGenCommerceVPC`라는 이름의 새로운 VPC와 그에 속한 서브넷, 라우팅 테이블 등이 완벽하게 구성된 것을 확인할 수 있다.

우리는 이제 코드 몇 줄로, 언제 어디서든 동일한 구성의 고가용성 네트워크를 단 몇 분 만에 재현할 수 있는 능력을 갖추게 되었다. 더 이상 수동 작업의 실수나 환경 불일치를 걱정할 필요가 없다. 이것이 바로 IaC가 제공하는 재현성과 진화의 힘이다.

이 견고한 대지 위에서, 우리는 드디어 아키텍처의 핵심 구성 요소인 '컴퓨팅'을 선택하고 배치할 준비를 마쳤다. 다음 장에서는 제어와 책임의 스펙트럼 위에서 EC2, ECS, EKS, 그리고 Lambda라는 각기 다른 특성을 가진 컴퓨팅 서비스들을 언제, 왜, 그리고 어떻게 선택해야 하는지에 대한 깊이 있는 트레이드오프 분석을 시작할 것이다.