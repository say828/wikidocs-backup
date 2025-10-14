## 03\. 플랫폼 엔지니어의 무기: Crossplane을 활용한 클라우드 리소스 프로비저닝

지금까지 우리가 만든 오퍼레이터는 쿠버네티스 클러스터 **안의** 리소스들(StatefulSet, Service 등)을 관리했다. 하지만 현실의 애플리케이션은 클러스터 외부의 수많은 클라우드 서비스들—AWS RDS, S3, Google Cloud SQL, Azure Blob Storage—에 의존한다. 이 외부 리소스들은 보통 Terraform, Pulumi, 혹은 클라우드 제공업체의 웹 콘솔과 같은 별개의 도구와 워크플로우를 통해 관리된다.

이것은 플랫폼의 '진실의 원천'이 Git과 Terraform 저장소, 두 개로 분열되었음을 의미한다. 개발자가 새로운 마이크로서비스를 배포하려면, 먼저 Terraform으로 데이터베이스를 생성하고, 그 결과물(DB 엔드포인트, 암호)을 다시 쿠버네티스 시크릿으로 만들어 애플리케이션에 주입해야 하는 번거롭고 단절된 과정을 거쳐야 한다.

**Crossplane**은 이 분열된 세계를 쿠버네티스 API라는 하나의 위대한 언어 아래 통일하려는, 야심 차고 강력한 CNCF 프로젝트다. Crossplane은 오퍼레이터 패턴을 극단까지 밀어붙여, **모든 외부 클라우드 리소스를 쿠버네티스의 CRD처럼 보이게 만드는** 거대한 변환 계층이다.

Crossplane의 세계에서는 더 이상 Terraform 코드를 작성할 필요가 없다. AWS RDS 데이터베이스를 만들고 싶다면, 그저 다음과 같은 YAML 파일을 `kubectl apply`하거나 Git에 커밋하기만 하면 된다.

```yaml
apiVersion: database.aws.upbound.io/v1beta1
kind: RDSInstance
metadata:
  name: my-production-rds
spec:
  forProvider: # AWS API에 전달될 파라미터
    region: us-west-2
    dbInstanceClass: db.t3.small
    masterUsername: myadmin
    allocatedStorage: 20
    engine: postgres
    engineVersion: "15"
    skipFinalSnapshot: true
  writeConnectionSecretToRef: # 생성된 DB의 접속 정보를 이 시크릿에 저장
    namespace: default
    name: rds-connection-details
```

이것이 어떻게 가능할까?

1.  클러스터에는 각 클라우드 제공업체(AWS, GCP, Azure)를 위한 Crossplane \*\*프로바이더(Provider)\*\*가 설치되어 있다. 이 프로바이더는 해당 클라우드의 모든 리소스(RDS, S3, VPC 등)에 대한 CRD와 커스텀 컨트롤러(오퍼레이터)의 집합이다.
2.  당신이 `RDSInstance`라는 CRD를 클러스터에 생성하면, AWS 프로바이더에 포함된 `RDSInstance` 컨트롤러가 이를 감지한다.
3.  컨트롤러는 `spec.forProvider`의 내용을 읽고, 저장된 클라우드 자격 증명을 사용하여 당신을 대신해 AWS API를 직접 호출하여 실제 RDS 인스턴스를 프로비저닝한다.
4.  프로비저닝이 완료되면, 컨트롤러는 RDS의 엔드포인트 주소, 포트, 암호와 같은 접속 정보를 `spec.writeConnectionSecretToRef`에 지정된 쿠버네티스 시크릿(`rds-connection-details`)에 자동으로 기록해준다.

이제 애플리케이션 디플로이먼트는 이 시크릿을 직접 참조하여 데이터베이스에 연결하기만 하면 된다. 인프라와 애플리케이션의 배포가 단 하나의 쿠버네티스 API, 단 하나의 GitOps 워크플로우 안에서 완벽하게 통합된 것이다.

더 나아가, Crossplane의 **Composition** 기능을 사용하면, 플랫폼 팀은 `XPostgreSQLInstance`라는 조직만의 추상화된 API를 만들 수 있다. 개발자는 그저 "나는 'production' 등급의 PostgreSQL 데이터베이스가 필요해요"라고 이 추상 API를 요청하기만 하면, Crossplane은 그 뒤에서 이 요청이 AWS 환경에서는 고가용성 RDS 클러스터를, GCP 환경에서는 Cloud SQL 인스턴스를, 개발 환경에서는 클러스터 내부의 PostgreSQL 오퍼레이터를 통해 생성되도록 모든 복잡성을 숨겨줄 수 있다.

Crossplane은 쿠버네티스를 애플리케이션 오케스트레이터를 넘어, \*\*모든 클라우드 자원을 아우르는 범용 컨트롤 플레인(Universal Control Plane)\*\*으로 격상시키는, 플랫폼 엔지니어의 궁극의 무기다.

이제 우리는 쿠버네티스의 API를 확장하여, 상상할 수 있는 거의 모든 종류의 자동화를 구현할 수 있는 힘을 갖게 되었다. 하지만 애플리케이션이 복잡해지고 서비스 간의 통신이 기하급수적으로 늘어나면서, 우리는 또 다른 차원의 문제, 즉 애플리케이션 네트워킹의 혼돈과 마주하게 된다. 다음 장에서는 이 문제를 해결하기 위한 강력한 패러다임, **서비스 메시**의 세계로 떠나볼 것이다.