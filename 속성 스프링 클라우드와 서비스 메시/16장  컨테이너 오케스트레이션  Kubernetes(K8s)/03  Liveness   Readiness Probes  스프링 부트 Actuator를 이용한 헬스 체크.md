## Liveness / Readiness Probes: 스프링 부트 Actuator를 이용한 헬스 체크

쿠버네티스는 컨테이너의 '프로세스'가 실행 중인지는 알 수 있습니다. 하지만 프로세스는 살아있는데, 애플리케이션 내부에 데드락(Deadlock)이 발생했거나, DB 커넥션 풀이 모두 고갈되어 더 이상 정상적인 응답을 할 수 없는 **'식물인간' 상태**가 되었다면 어떻게 될까요?

쿠버네티스는 기본적으로 이 상태를 감지하지 못하고, 계속해서 이 '죽은' 컨테이너로 사용자 트래픽을 보낼 것입니다. 또한, 컨테이너가 막 시작되어 스프링 애플리케이션이 초기화되는(DB 커넥션을 맺고, 캐시를 워밍업하는) 30초 동안은 정상적인 서비스가 불가능합니다. 이 때 트래픽이 들어오면 에러가 발생합니다.

이 문제를 해결하기 위해, 쿠버네티스는 \*\*프로브(Probe)\*\*라는 강력한 헬스 체크(Health Check) 메커니즘을 제공합니다.

-----

### Liveness Probe vs. Readiness Probe

쿠버네티스는 크게 두 가지 종류의 프로브를 통해 컨테이너 내부 애플리케이션의 상태를 정밀하게 파악합니다.

#### 1\. Liveness Probe (생존 프로브): "살아있는가?" ❤️

  * **목적:** 컨테이너 안의 애플리케이션이 \*\*정상적으로 살아있는지(Alive)\*\*를 검사합니다.
  * **비유:** 환자의 **'심박 측정기'**. 주기적으로 심장이 뛰는지 확인합니다.
  * **조치:** 만약 Liveness 프로브가 **실패하면**, 쿠버네티스는 해당 컨테이너가 회복 불가능한 '좀비' 상태라고 판단하고, 그 컨테이너를 가차없이 **죽인 뒤 새로 재시작**시킵니다.
  * **핵심 역할:** 데드락, 메모리 누수 등으로 인해 애플리케이션이 응답 불능 상태에 빠졌을 때, 이를 감지하고 \*\*자동으로 복구(Self-healing)\*\*하는 역할을 합니다.

#### 2\. Readiness Probe (준비 프로브): "손님 받을 준비 됐나?" 팻말

  * **목적:** 컨테이너가 \*\*트래픽을 받을 준비가 되었는지(Ready)\*\*를 검사합니다.
  * **비유:** 레스토랑의 **'Open/Closed' 팻말**. 주방에 불이 켜져 있고 요리사들이 출근했더라도(컨테이너 실행 중), 재료 준비가 끝나지 않았다면 'Closed' 팻말을 걸어두고 손님을 받지 않습니다.
  * **조치:** 만약 Readiness 프로브가 **실패하면**, 쿠버네티스는 컨테이너를 죽이지 않습니다. 대신, \*\*`Service` 오브젝트의 로드 밸런싱 목록에서 해당 컨테이너(Pod)의 IP를 일시적으로 '제거'\*\*합니다. 더 이상 새로운 트래픽이 이 Pod로 전달되지 않습니다. 나중에 애플리케이션 준비가 완료되어 프로브가 다시 성공하면, 쿠버네티스는 Pod를 로드 밸런싱 목록에 다시 추가합니다.
  * **핵심 역할:**
    1.  애플리케이션이 시작될 때, DB 커넥션 등 모든 초기화가 완료될 때까지 트래픽이 들어오는 것을 막아줍니다.
    2.  일시적으로 과부하가 걸리거나, 외부 시스템(DB 등)과의 연결이 끊어졌을 때, 스스로를 서비스에서 제외하여 장애가 확산되는 것을 방지합니다.

-----

### 스프링 부트 Actuator와의 연동

이 프로브들을 구현하는 가장 일반적인 방법은 쿠버네티스가 특정 HTTP 엔드포인트에 주기적으로 `GET` 요청을 보내 상태 코드를 확인하는 것입니다. **스프링 부트 Actuator**는 이를 위한 완벽한 기능을 제공합니다.

`spring-boot-starter-actuator` 의존성을 추가하면, `/actuator/health`라는 엔드포인트가 생성됩니다. 스프링 부트 2.3부터는 쿠버네티스 프로브를 위해 이 엔드포인트가 `/liveness`와 `/readiness` 두 그룹으로 분리되었습니다.

  * `/actuator/health/liveness`: 애플리케이션 자체의 생존 여부만 확인. (메모리 부족 등)
  * `/actuator/health/readiness`: 애플리케이션 자체뿐만 아니라, 연결된 데이터베이스나 다른 서비스의 상태까지 모두 확인하여 '서비스 가능' 여부를 알려줌.

#### `Deployment` YAML에 프로브 적용하기

`member-service-deployment.yaml` 파일에 Liveness/Readiness 프로브를 추가해 봅시다.

```yaml
# member-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: member-service
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: member-service-container
          image: ecommerce/member-service:0.0.1
          ports:
            - containerPort: 8081
          
          # --- Liveness Probe 설정 ---
          livenessProbe:
            httpGet: # 1. HTTP GET 방식으로 헬스 체크
              path: /actuator/health/liveness # 2. Liveness 전용 엔드포인트
              port: 8081
            initialDelaySeconds: 30 # 3. 컨테이너 시작 후 30초 뒤부터 검사 시작
            periodSeconds: 10       # 4. 10초마다 반복 검사
            failureThreshold: 3     # 5. 3번 연속 실패 시 컨테이너 재시작

          # --- Readiness Probe 설정 ---
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness # 6. Readiness 전용 엔드포인트
              port: 8081
            initialDelaySeconds: 15 # 7. 15초 뒤부터 검사 시작
            periodSeconds: 5        # 8. 5초마다 반복 검사
            failureThreshold: 3     # 9. 3번 연속 실패 시 서비스 목록에서 제외
```

  * **(설정)** `application.yml`에서 `management.health.livenessstate.enabled: true`와 `readinessstate.enabled: true`를 설정하여 각 엔드포인트를 활성화해야 합니다.

이제 쿠버네티스는 단순히 '프로세스가 살아있는지'를 넘어, 우리 애플리케이션의 '속사정'까지 파악하여 훨씬 더 지능적으로 시스템의 안정성을 유지해 줍니다. Liveness와 Readiness 프로브를 설정하는 것은 쿠버네티스 위에서 프로덕션급 서비스를 운영하기 위한 **필수적인 절차**입니다.