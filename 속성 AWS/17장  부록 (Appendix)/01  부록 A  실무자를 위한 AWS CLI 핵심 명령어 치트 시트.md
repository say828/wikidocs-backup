# 부록 A: 실무자를 위한 AWS CLI 핵심 명령어 치트 시트

이 부록은 AWS Management Console의 그래픽 인터페이스를 넘어, 커맨드 라인이라는 강력하고 효율적인 도구를 통해 AWS를 제어하고자 하는 모든 실무자를 위한 치트 시트다. AWS CLI(Command Line Interface)는 스크립팅을 통해 반복적인 작업을 자동화하고, IaC로는 번거로운 간단한 확인 작업을 빠르게 수행하며, 여러분의 클라우드 운영 능력을 한 차원 높여줄 필수적인 무기다.

여기서는 일상적인 운영 및 개발 과정에서 가장 빈번하게 사용되는 핵심 명령어들을 선별하여, 즉시 복사해서 사용할 수 있는 실용적인 예제와 함께 제공한다.

-----

## 00\. 기본 설정 및 프로파일 관리

AWS CLI를 사용하기 위한 가장 첫 단계는 당신의 자격 증명(Access Key)과 기본 설정을 구성하는 것이다.

**최초 설정 (자격 증명 및 기본값 구성):**

```bash
aws configure
```

> 이 명령어를 실행하면 AWS Access Key ID, Secret Access Key, 기본 리전(예: `ap-northeast-2`), 기본 출력 형식(예: `json`)을 순서대로 입력하라는 프롬프트가 나타난다.

**여러 계정을 다루기 위한 프로파일 생성:**

```bash
aws configure --profile my-dev-account
```

> `my-dev-account`라는 이름의 새로운 프로파일을 생성한다. 이후 모든 명령어에 `--profile my-dev-account` 옵션을 붙여 해당 계정의 자격 증명으로 명령을 실행할 수 있다.

**현재 설정된 자격 증명(호출자) 정보 확인:**

```bash
aws sts get-caller-identity
```

-----

## 01\. IAM 및 보안 자격 증명 관리

**특정 IAM 사용자 정보 조회:**

```bash
aws iam get-user --user-name my-test-user
```

**특정 IAM 역할(Role) 정보 조회:**

```bash
aws iam get-role --role-name my-app-role
```

**IAM 역할의 신뢰 정책(Assume Role Policy) 업데이트:**

```bash
aws iam update-assume-role-policy --role-name my-app-role --policy-document file://trust-policy.json
```

> `trust-policy.json` 파일에 정의된 내용으로 역할의 신뢰 정책을 업데이트한다.

-----

## 02\. EC2 및 VPC 제어

**특정 태그를 가진 EC2 인스턴스 정보 조회:**

```bash
aws ec2 describe-instances --filters "Name=tag:Project,Values=nextgen-commerce" "Name=tag:Environment,Values=production"
```

**EC2 인스턴스 시작 및 중지:**

```bash
aws ec2 start-instances --instance-ids i-0123456789abcdef0
aws ec2 stop-instances --instance-ids i-0123456789abcdef0
```

**나의 현재 IP 주소로 SSH(22번 포트) 접근을 허용하는 보안 그룹 규칙 추가:**

```bash
aws ec2 authorize-security-group-ingress --group-id sg-0abcdef1234567890 --protocol tcp --port 22 --cidr $(curl -s http://checkip.amazonaws.com)/32
```

**모든 VPC 목록과 CIDR 블록 조회:**

```bash
aws ec2 describe-vpcs --query "Vpcs[*].{VpcId:VpcId, CidrBlock:CidrBlock, IsDefault:IsDefault}" --output table
```

-----

## 03\. S3 및 DynamoDB 데이터 조작

**S3 버킷 목록 조회:**

```bash
aws s3 ls
```

**로컬 파일을 S3 버킷으로 복사:**

```bash
aws s3 cp my-local-file.txt s3://my-nextgen-bucket/
```

**S3 버킷과 로컬 디렉토리 동기화 (변경된 파일만 업로드):**

```bash
aws s3 sync ./my-local-assets/ s3://my-nextgen-bucket/assets/
```

**DynamoDB 테이블에서 특정 아이템 조회:**

```bash
aws dynamodb get-item --table-name product-catalog --key '{"productId": {"S": "product-123"}}'
```

**DynamoDB 테이블에 새로운 아이템 추가:**

```bash
aws dynamodb put-item --table-name user-sessions --item '{"sessionId": {"S": "session-xyz"}, "userId": {"S": "user-456"}, "ttl": {"N": "1665640000"}}'
```

-----

## 04\. Lambda 및 API Gateway 배포 및 관리

**모든 Lambda 함수 목록 조회:**

```bash
aws lambda list-functions
```

**Lambda 함수의 코드 업데이트 (ZIP 파일 사용):**

```bash
aws lambda update-function-code --function-name my-order-processor --zip-file fileb://function.zip
```

**Lambda 함수의 환경 변수 업데이트:**

```bash
aws lambda update-function-configuration --function-name my-order-processor --environment "Variables={LOG_LEVEL=DEBUG,TABLE_NAME=orders}"
```

**Lambda 함수 직접 호출 및 결과 확인:**

```bash
aws lambda invoke --function-name my-thumbnail-generator --payload '{"s3Bucket": "my-images", "s3Key": "input/image.jpg"}' response.json
```

**API Gateway의 모든 API 목록 조회:**

```bash
aws apigateway get-rest-apis
```