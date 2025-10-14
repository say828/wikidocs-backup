## 01\. Terraform vs AWS CDK: 철학과 선택의 기준

'클릭옵스'라는 죄악에서 벗어나기로 결심했다면, 우리는 이제 어떤 도구를 사용하여 코드로 인프라를 정의할지 선택해야 한다. IaC의 세계에는 여러 강력한 도구가 존재하지만, 현대 클라우드 환경에서는 사실상 두 거인이 시장을 양분하고 있다: 하시코프(HashiCorp)의 \*\*테라폼(Terraform)\*\*과 아마존 웹 서비스의 **AWS CDK(Cloud Development Kit)**.

이 둘은 단순히 문법만 다른 도구가 아니다. 그 기저에는 인프라를 바라보는 근본적인 **철학의 차이**가 존재한다. 어떤 도구를 선택하는가는 단순히 기술 스택을 결정하는 것을 넘어, 당신의 팀이 인프라를 어떻게 생각하고, 설계하며, 진화시켜 나갈 것인지에 대한 방향성을 결정하는 중요한 의사결정이다.

### 테라폼(Terraform): 선언적 상태 관리의 제왕

테라폼은 2014년 출시된 이래, 사실상의 업계 표준(De facto Standard)으로 자리매김했다. 테라폼의 핵심 철학은 **'선언적(Declarative)'** 접근 방식에 있다. 사용자는 최종적으로 원하는 인프라의 상태(State), 즉 '무엇(What)'을 HCL(HashiCorp Configuration Language)이라는 도메인 특화 언어로 정의한다. 그러면 테라폼 엔진이 현재 상태와 원하는 상태의 차이를 계산하여, 그 차이를 메우기 위해 '어떻게(How)' 실행할지 계획(Plan)을 세우고 적용(Apply)한다.

이는 마치 건축가에게 건물의 최종 청사진을 전달하는 것과 같다. 건축가는 "방 3개, 화장실 2개, 그리고 남향 창문이 있는 집을 원한다"고 선언할 뿐, 벽돌을 몇 개 쌓고, 파이프를 어떻게 연결할지 구체적인 작업 순서를 지시하지 않는다.

```terraform
# main.tf - Terraform HCL 예제
# "나는 'my-app-server'라는 이름을 가진 t3.micro 타입의 EC2 인스턴스를 원한다."
# "이 인스턴스는 ami-0c55b159cbfafe1f0 AMI를 사용해야 한다."

resource "aws_instance" "my_app_server" {
  ami           = "ami-0c55b159cbfafe1f0" # Amazon Linux 2 AMI
  instance_type = "t3.micro"

  tags = {
    Name = "MyAppServer"
  }
}
```

**테라폼의 강점:**

  * **클라우드 불가지론 (Cloud-Agnostic):** 단일 언어와 워크플로우를 통해 AWS뿐만 아니라 Azure, Google Cloud, 심지어 온프레미스 인프라까지 관리할 수 있다. 멀티 클라우드 전략을 가진 조직에게는 압도적인 장점이다.
  * **명시적인 상태 관리:** 테라폼은 인프라의 현재 상태를 `terraform.tfstate`라는 파일에 명시적으로 기록하고 관리한다. 이를 통해 복잡한 변경 사항을 적용하기 전에 어떤 리소스가 생성, 수정, 삭제될지 정확하게 예측(Plan)할 수 있다.
  * **성숙한 생태계와 커뮤니티:** 오랜 기간 업계 표준으로 군림해 온 만큼, 방대한 모듈 라이브러리와 풍부한 학습 자료, 거대한 커뮤니티를 보유하고 있다.

### AWS CDK: 프로그래밍 언어로 빚어내는 인프라

AWS CDK는 테라폼보다 훨씬 뒤늦게 등장했지만, AWS 생태계 내에서 폭발적인 성장세를 보이고 있다. CDK의 핵심 철학은 **'명령형(Imperative)'** 프로그래밍 방식을 통해 인프라를 정의하는 것이다. 개발자는 자신이 가장 익숙한 범용 프로그래밍 언어(TypeScript, Python, Java, Go 등)를 사용하여 코드를 작성한다. 이 코드는 최종적으로 AWS의 네이티브 IaC 서비스인 **클라우드포메이션(CloudFormation)** 템플릿으로 합성(Synthesize)되어 배포된다.

이는 마치 레고 블록 전문가에게 "이 부품들을 사용해서 성을 만들어 줘"라고 말하는 것과 같다. 전문가는 반복문, 조건문, 함수와 같은 프로그래밍 로직을 자유자재로 구사하여 훨씬 더 동적이고 정교한 구조물을 효율적으로 만들어낼 수 있다.

```typescript
// lib/my-stack.ts - AWS CDK TypeScript 예제
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';

export class MyStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // "나는 t3.micro 타입의 Amazon Linux AMI를 사용하는 EC2 인스턴스를 새로 만든다."
    new ec2.Instance(this, 'MyAppServer', {
      instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
      machineImage: new ec2.AmazonLinuxImage(),
      vpc: ec2.Vpc.fromLookup(this, 'DefaultVPC', { isDefault: true }),
    });
  }
}
```

**AWS CDK의 강점:**

  * **친숙한 프로그래밍 언어:** 개발자들은 새로운 DSL을 배울 필요 없이 기존의 프로그래밍 지식과 IDE, 테스트 프레임워크 등 익숙한 도구를 그대로 활용하여 인프라를 코딩할 수 있다.
  * **높은 수준의 추상화:** CDK는 `ec2.Vpc`, `ecs.Cluster`와 같은 고수준의 \*\*구성(Construct)\*\*을 제공한다. 단 몇 줄의 코드로 수백 줄의 클라우드포메이션 템플릿으로 변환되는 모범 사례(Best Practice)가 내장된 인프라 패턴을 생성할 수 있다.
  * **애플리케이션과 인프라 코드의 통합:** 애플리케이션 코드와 인프라 코드를 동일한 언어, 동일한 리포지토리에서 관리할 수 있어 진정한 의미의 "You Build It, You Run It" 데브옵스 문화를 실현하기에 용이하다.

### 철학과 선택의 기준

**[도표: Terraform vs AWS CDK 핵심 철학 및 기능 비교]**

| 구분 | Terraform | AWS CDK |
| :--- | :--- | :--- |
| **핵심 철학** | **선언적 (Declarative)**: 최종 상태를 정의 | **명령형 (Imperative)**: 생성 과정을 정의 |
| **사용 언어** | HCL (도메인 특화 언어) | TypeScript, Python, Java, Go 등 (범용 언어) |
| **추상화 수준**| 리소스 중심 (상대적으로 낮음) | 구성 중심 (높은 수준의 추상화 제공) |
| **상태 관리** | 명시적 (`tfstate` 파일) | 암묵적 (CloudFormation 스택에 위임) |
| **생태계** | **멀티 클라우드** 및 온프레미스 | **AWS 네이티브** (AWS 서비스에 최적화) |
| **최적의 사용자**| 인프라 엔지니어, SRE, 멀티 클라우드 환경 | 애플리케이션 개발자, 데브옵스, AWS 중심 환경 |

결론적으로, 어느 한쪽이 절대적으로 우월하다고 말할 수는 없다. 선택은 당신의 조직과 프로젝트가 처한 상황에 따라 달라진다.

  * 이미 여러 클라우드를 함께 사용하고 있거나, 인프라 엔지니어링 팀의 역량이 HCL에 집중되어 있다면 **테라폼**은 여전히 훌륭하고 안정적인 선택이다.
  * 주력 클라우드가 AWS이며, 애플리케이션 개발자들이 인프라 구축에 더 깊이 관여하기를 원하고, 프로그래밍 언어의 표현력을 활용하여 복잡하고 동적인 인프라를 구축하고자 한다면 **AWS CDK**는 비할 데 없는 생산성을 제공할 것이다.

이 책은 'AWS 클라우드 네이티브 아키텍처의 정수'를 탐구하는 것을 목표로 한다. 따라서 우리는 AWS 서비스와의 가장 깊고 유연한 통합, 그리고 애플리케이션 로직과 인프라의 경계를 넘나드는 현대적인 개발 경험을 제공하는 \*\*AWS CDK(TypeScript)\*\*를 우리의 표준 도구로 채택한다. 이제, 이 강력한 도구를 사용하여 실전 프로젝트를 어떻게 구조화할 것인지 살펴보자.