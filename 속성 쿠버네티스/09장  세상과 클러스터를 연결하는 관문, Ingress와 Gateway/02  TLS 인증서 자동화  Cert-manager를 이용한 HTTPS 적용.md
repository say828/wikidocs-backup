## 02\. TLS 인증서 자동화: Cert-manager를 이용한 HTTPS 적용

우리는 이제 클러스터의 관문을 지킬 강력한 Ingress Controller를 선택하고 배치했다. 하지만 이 관문이 평문 HTTP 트래픽을 그대로 통과시킨다면, 우리는 성문은 굳건히 지었으면서 그 위에 지붕을 얹는 것을 잊어버린 것과 같다. 인터넷이라는 전쟁터에서 암호화되지 않은 통신은 도청, 데이터 변조, 중간자 공격(Man-in-the-Middle)에 무방비로 노출되는 자살 행위나 다름없다. 오늘날 HTTPS는 선택이 아닌, 모든 서비스가 갖춰야 할 기본적인 예의이자 의무다.

그렇다면 이 HTTPS를 위한 TLS 인증서는 어떻게 관리해야 할까? 과거의 방식은 끔찍했다.

1.  수동으로 개인 키와 CSR(인증서 서명 요청)을 생성한다.
2.  Let's Encrypt와 같은 인증 기관(CA)에 접속하여 도메인 소유권을 증명한다. (웹 서버에 특정 파일을 올리거나 DNS 레코드를 추가하는 등)
3.  발급받은 인증서와 키 파일을 이용해 `kubectl create secret tls` 명령으로 쿠버네티스 시크릿을 생성한다.
4.  이 시크릿의 이름을 Ingress 리소스의 `tls` 섹션에 명시한다.
5.  **그리고 90일마다 이 모든 과정을 모든 도메인에 대해 반복한다.**

이 수동 프로세스는 지루할 뿐만 아니라, 인간의 실수를 유발하는 가장 확실한 방법이다. 인증서 만료로 인한 서비스 중단은 기술 회사에서 가장 흔하게 발생하는, 그러나 가장 부끄러운 장애 유형 중 하나다.

이 고통스러운 굴레에서 우리를 해방시키기 위해 탄생한 프로젝트가 바로 **cert-manager**다. cert-manager는 쿠버네티스 클러스터 내에서 TLS 인증서의 발급부터 갱신, 관리에 이르는 모든 생명주기를 완벽하게 자동화하는, 오늘날 논쟁의 여지가 없는 업계 표준 컨트롤러(오퍼레이터)다.

cert-manager는 다음과 같은 자체 CRD를 통해 우리의 '의도'를 이해하고 마법을 부린다.

  * **`Issuer` / `ClusterIssuer` (인증 기관):** 어떤 인증 기관(CA)으로부터 인증서를 발급받을지를 정의하는 리소스다. 우리는 여기에 Let's Encrypt의 ACME 서버 주소나, 내부 PKI를 위한 Vault 주소 등을 설정한다. `Issuer`는 특정 네임스페이스에서만 작동하고, `ClusterIssuer`는 클러스터 전체에서 작동하는 전역적인 리소스다.

  * **`Certificate` (인증서 요청서):** "나는 `myapp.example.com` 도메인을 위한 인증서가 필요하니, `letsencrypt-prod` Issuer를 통해 발급받아 `my-app-tls-secret`이라는 이름의 시크릿에 저장해줘"라는 구체적인 요청을 담는 리소스다.

cert-manager 컨트롤러는 이 `Certificate` 리소스의 생성을 감지하고, `Issuer`에 정의된 방식에 따라 ACME 프로토콜을 통해 Let's Encrypt와 통신하며 도메인 소유권 검증(Challenge)을 자동으로 수행한다. 검증이 완료되면, 발급받은 인증서와 개인 키를 지정된 이름의 TLS 시크릿에 안전하게 저장하고, 만료일이 다가오면 알아서 갱신까지 해준다.

하지만 cert-manager의 진정한 아름다움은 **ingress-shim**이라는 기능을 통해 Ingress 리소스와 완벽하게 통합된다는 점에서 드러난다. 우리는 `Certificate` 리소스를 직접 만들 필요조차 없다. 그저 우리의 Ingress 리소스에 특정 **어노테이션** 하나만 추가하면 된다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    # --- Cert-manager Magic Starts Here ---
    cert-manager.io/cluster-issuer: "letsencrypt-prod" # 1. 어떤 Issuer를 쓸지 지정
spec:
  ingressClassName: nginx
  tls: # 2. 이 Ingress는 TLS를 사용하며...
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-secret # 3. 인증서는 이 시크릿에 저장될 것이다.
  rules:
  - host: "myapp.example.com"
    http:
      paths:
      # ... (라우팅 규칙)
```

이 Ingress가 클러스터에 적용되는 순간, 다음과 같은 자동화된 연쇄 반응이 일어난다.

1.  cert-manager의 ingress-shim 컨트롤러가 저 어노테이션을 발견한다.
2.  Ingress의 `tls` 섹션에 명시된 호스트(`myapp.example.com`)와 시크릿 이름(`myapp-tls-secret`)을 읽어, 이를 바탕으로 `Certificate` 리소스를 **자동으로 생성**한다.
3.  cert-manager의 핵심 컨트롤러가 이 `Certificate`를 보고 Let's Encrypt와의 통신을 시작한다.
4.  도메인 검증(HTTP-01 챌린지)을 위해, cert-manager는 임시로 이 Ingress 리소스를 수정하여 챌린지용 파드로 향하는 경로를 추가한다.
5.  검증이 성공하면, 발급받은 인증서가 `myapp-tls-secret`에 저장되고, Ingress Controller는 이 시크릿을 읽어 HTTPS 트래픽을 처리하기 시작한다.

이제 우리는 더 이상 TLS 인증서를 신경 쓸 필요가 없다. 새로운 서비스를 추가하고 싶다면, 그저 `host`와 `secretName`을 추가한 Ingress YAML을 Git에 커밋하기만 하면 된다. cert-manager가 그 뒷단의 모든 복잡하고 지루한 작업을 완벽하게 처리해 줄 것이다.

우리는 이제 클러스터의 관문을 선택했고, 그 관문을 통과하는 모든 통신을 암호화하는 자동화된 방어 체계까지 구축했다. 하지만 Ingress API 자체의 근본적인 한계—모호한 역할 분담, 비표준 어노테이션의 남용—는 여전히 우리를 괴롭힌다. 다음 절에서는 이 모든 문제를 해결하고 쿠버네티스 네트워킹의 미래를 열어갈 차세대 표준, Gateway API의 세계로 나아갈 것이다.