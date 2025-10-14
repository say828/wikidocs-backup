## Istio 라우팅 문제 해결: istioctl proxy-config와 analyze

Istio의 `VirtualService`와 `DestinationRule`은 매우 강력하지만, 선언적인 YAML로 동작하기 때문에 문제가 발생했을 때 "왜 트래픽이 내가 원하는 대로 흐르지 않는가?"에 대한 원인을 찾기 어려울 수 있습니다. 요청이 `503 Service Unavailable` 에러를 반환하거나, 의도치 않은 버전의 서비스로 라우팅되는 등의 문제는 모든 Istio 운영자가 마주하게 되는 흔한 시나리오입니다.

이러한 라우팅 문제를 해결하는 핵심은, \*\*"내가 YAML에 작성한 '의도'와, `istiod`가 Envoy 프록시에 실제로 내려보낸 '설정'이 일치하는가?"\*\*를 확인하는 것입니다. `istioctl`은 이 과정을 위한 강력한 진단 도구들을 제공합니다.

-----

### 디버깅 절차: 단계별 원인 분석

#### 1단계 (가장 먼저): 정적 분석 (`istioctl analyze`)

코드를 컴파일하기 전에 '린터(Linter)'를 돌려보듯이, YAML을 클러스터에 적용하기 전후에 `istioctl analyze`를 실행하는 것은 가장 기본적이고 효과적인 첫 단계입니다. 이 명령어는 클러스터의 Istio 리소스들을 스캔하여 흔히 발생하는 설정 오류를 찾아줍니다.

```bash
# default 네임스페이스의 모든 Istio 리소스를 분석
istioctl analyze -n default
```

**주요 발견 오류:**

  * `IST0108 RouteToNonExistentSubset`: `VirtualService`가 트래픽을 `v2` 서브셋으로 보내도록 설정했지만, `DestinationRule`에 `v2` 서브셋이 정의되어 있지 않은 경우. (가장 흔한 실수 중 하나)
  * `IST0107 MissingGateway`: `VirtualService`가 존재하지 않는 `Gateway`에 바인딩된 경우.
  * 기타 YAML 문법 오류나 잘못된 참조 등.

-----

### 2단계: 프록시 동기화 상태 확인 (`istioctl proxy-status`)

`analyze`에서 아무런 문제가 발견되지 않았다면, 다음은 모든 Envoy 사이드카 프록시들이 컨트롤 플레인(`istiod`)으로부터 최신 설정을 잘 받고 있는지 확인해야 합니다.

```bash
istioctl proxy-status
# 또는 단축 명령어: istioctl ps
```

**출력 예시:**

```
NAME                                CLUSTERS     ...   ISTIOD                     VERSION
member-service-5f5d6f6f88-abcde     10           ...   istiod-5f5d6f6f88-ghijk    1.21.0
order-service-6d7b8c8f9b-fghij      12           ...   istiod-5f5d6f6f88-ghijk    1.21.0
...
```

`ISTIOD` 컬럼에 각 프록시와 연결된 `istiod`의 버전이 표시됩니다. 만약 특정 프록시의 동기화 상태가 `SYNCED`가 아닌 \*\*`STALE`\*\*이라면, 해당 프록시는 컨트롤 플레인과의 통신 문제로 인해 최신 라우팅 규칙을 받지 못하고 있다는 의미이며, 이는 컨트롤 플레인 자체의 문제나 네트워크 문제를 의심해 봐야 합니다.

-----

### 3단계 (심층 분석): 프록시 실제 설정 확인 (`istioctl proxy-config`)

위 두 단계로도 문제가 해결되지 않았다면, 이제 Envoy 프록시의 '뇌' 속으로 직접 들어가 볼 차례입니다. `proxy-config`는 특정 Pod의 사이드카가 현재 **실제로 사용하고 있는** 라우팅 규칙과 클러스터 정보를 덤프하여 보여줍니다.

**중요:** `order-service`가 `product-service`를 호출하는 데 문제가 있다면, **출발지인 `order-service` Pod**의 프록시 설정을 확인해야 합니다.

```bash
# order-service Pod 중 하나의 이름을 확인
POD_NAME=$(kubectl get pod -l app=order-service -o jsonpath='{.items[0].metadata.name}')

# 1. 라우팅 규칙 확인 (route, r)
istioctl proxy-config route $POD_NAME --name 8082 -o json
```

  * `--name 8082`: `product-service`가 사용하는 포트(8082)로 나가는 트래픽에 대한 라우팅 규칙만 필터링합니다.
  * **확인할 것:** 출력된 JSON에서 `virtualHosts.routes` 부분을 살펴봅니다.
      * `match` 조건이 내가 `VirtualService`에 정의한 `uri`, `headers` 등과 일치하는가?
      * `route.cluster` 값이 내가 `DestinationRule`에 정의한 서브셋(예: `outbound|8082|v1|product-service...`)과 일치하는가?

<!-- end list -->

```bash
# 2. 클러스터(목적지) 상태 확인 (cluster, c)
istioctl proxy-config cluster $POD_NAME --fqdn product-service.default.svc.cluster.local -o json
```

  * `--fqdn`: 목적지 서비스의 FQDN을 지정합니다.
  * **확인할 것:** 출력된 JSON에서 `endpoints` 부분을 살펴봅니다.
      * `lbEndpoints` 목록에 내가 예상하는 `product-service`의 **Pod IP들이 올바르게 포함**되어 있는가?
      * 각 엔드포인트의 `healthStatus`가 `HEALTHY`인가? 만약 서킷 브레이커에 의해 차단되었다면 `UNHEALTHY`로 표시됩니다.
      * 만약 엔드포인트 목록이 비어있다면, `DestinationRule`의 `subset`에 정의된 `labels`가 실제 `product-service` Pod의 `labels`와 일치하지 않을 가능성이 높습니다.

-----

**결론적으로,** Istio 라우팅 문제 해결은 \*\*'추측'이 아닌 '검증'\*\*의 과정입니다. `istioctl analyze`로 명백한 설정 오류를 먼저 잡고, `proxy-status`로 동기화 상태를 확인한 뒤, 마지막으로 `proxy-config`를 통해 Envoy 프록시의 실제 동작 규칙을 직접 확인함으로써, 우리는 복잡한 서비스 메시 환경의 트래픽 문제를 체계적으로 진단하고 해결할 수 있습니다.