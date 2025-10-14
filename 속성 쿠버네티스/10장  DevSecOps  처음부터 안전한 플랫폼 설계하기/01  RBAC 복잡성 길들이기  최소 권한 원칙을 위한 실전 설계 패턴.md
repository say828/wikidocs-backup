## 01\. RBAC 복잡성 길들이기: 최소 권한 원칙을 위한 실전 설계 패턴

이제 우리는 쿠버네티스 보안의 심장, \*\*RBAC(Role-Based Access Control)\*\*의 세계로 들어왔다. RBAC는 이론적으로는 지극히 명쾌하다. "누가(Subject) 무엇을(Resource) 어떻게(Verb) 할 수 있는지"를 정의하는 것. 하지만 이 단순한 개념은 현실의 복잡한 조직 구조와 수많은 마이크로서비스를 만나면서, 순식간에 누구도 이해하거나 관리할 수 없는 거대한 스파게티 덩어리로 변해버리기 일쑤다. 잘못 설계된 RBAC는 보안 구멍을 만들거나, 반대로 개발자의 모든 작업을 가로막는 족쇄가 되어 생산성을 파괴한다.

RBAC의 복잡성을 길들이는 유일한 길은, 처음부터 \*\*최소 권한의 원칙(Principle of Least Privilege)\*\*이라는 북극성을 나침반 삼아, 명확한 설계 패턴에 따라 움직이는 것이다. 이 원칙은 간단하다. "모든 주체(사용자, 서비스 어카운트)는 자신의 임무를 수행하는 데 필요한 **최소한의 권한만을** 가져야 한다." 데이터베이스를 관리하는 서비스 어카운트가 시크릿을 조회할 권한은 필요하지만, 디플로이먼트를 삭제할 권한은 없어야 하는 것이 당연하다.

이 원칙을 실현하기 위해, 우리는 RBAC의 네 가지 구성 요소를 정교하게 조립해야 한다.

1.  **Role (역할):** 특정 **네임스페이스** 안에서만 유효한 권한의 집합이다. " `prod` 네임스페이스 안에서 파드(pods)를 조회(get, list, watch)할 수 있다"와 같은 규칙을 정의한다.
2.  **ClusterRole (클러스터 역할):** Role의 클러스터 버전이다. 네임스페이스에 종속되지 않는 클러스터 전역의 리소스(예: 노드(nodes), 퍼시스턴트 볼륨(persistentvolumes))에 대한 권한을 정의하거나, 여러 네임스페이스에 걸쳐 공통적으로 적용될 권한의 템플릿으로 사용된다.
3.  **RoleBinding (역할 바인딩):** Role을 특정 주체(사용자, 그룹, 서비스 어카운트)에게 **연결**하는 다리다. " 'alice'라는 사용자에게 'pod-reader' Role을 부여한다"와 같이, 권한의 정의와 그 권한의 소유자를 묶어준다. RoleBinding 역시 특정 네임스페이스 안에서만 유효하다.
4.  **ClusterRoleBinding (클러스터 역할 바인딩):** ClusterRole을 모든 네임스페이스에 걸쳐 특정 주체에게 연결한다. 클러스터 관리자에게 모든 권한을 부여하는 것과 같이 매우 강력한 연결이므로, 극히 신중하게 사용해야 한다.

이 네 가지 블록을 가지고, 우리는 다음과 같은 실전 설계 패턴을 따라야 한다.

-----

### 패턴 1: 네임스페이스를 권한의 경계로 삼아라

가장 기본적이고 중요한 패턴이다. 가능하면 모든 권한은 **Role**과 **RoleBinding**을 사용하여 특정 네임스페이스 안에 가두어야 한다. 개발팀 A는 `team-a` 네임스페이스의 완전한 주인일 수 있지만, `team-b` 네임스페이스의 리소스는 아예 보지도 못하게 해야 한다. ClusterRole과 ClusterRoleBinding은 플랫폼 관리자와 같이 클러스터 전체를 관리해야 하는 극소수의 역할에게만 허락되는 '절대 반지'와 같다는 것을 명심하라.

### 패턴 2: 역할(Role)은 작고, 구체적이며, 재사용 가능하게 만들어라

"모든 것을 할 수 있는" 거대한 `god-mode` Role을 만들고 싶은 유혹을 뿌리쳐라. 대신, 역할의 책임을 명확하게 분리하라.

**나쁜 설계 (Anti-Pattern):**

```yaml
# monorepo-admin-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

**좋은 설계 (Best Practice):**

```yaml
# deployment-manager-role
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
# secret-reader-role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

이렇게 잘게 쪼개진 역할들은 레고 블록처럼 조합하여 사용할 수 있다. CI/CD 파이프라인을 위한 서비스 어카운트는 `deployment-manager-role`과 `secret-reader-role` 두 개를 바인딩해주면 된다.

### 패턴 3: ClusterRole을 역할 템플릿으로 활용하라

모든 네임스페이스의 개발자들에게 공통적으로 "자신의 네임스페이스 안에서 파드 로그를 볼 수 있는" 권한을 주고 싶다고 가정해보자. 각 네임스페이스마다 동일한 내용의 Role을 수십 개 만드는 것은 끔찍한 중복이다.

이때 바로 **ClusterRole**이 템플릿으로 빛을 발한다. 먼저, 권한의 내용만을 담은 ClusterRole을 하나 정의한다.

```yaml
# pod-log-reader-clusterrole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-log-reader
rules:
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list", "watch"]
```

그리고 각 네임스페이스에서는 이 ClusterRole을 참조하는 **RoleBinding**을 생성한다.

```yaml
# team-a-log-reader-binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-log-reader
  namespace: team-a
subjects:
- kind: Group
  name: team-a-developers
  apiGroup: rbac.authorization.k8s.io
roleRef: # Role이 아닌 ClusterRole을 참조한다!
  kind: ClusterRole
  name: pod-log-reader
  apiGroup: rbac.authorization.k8s.io
```

이로써 우리는 권한의 정의(`ClusterRole`)는 중앙에서 한 번만 하고, 그 권한의 부여(`RoleBinding`)는 각 네임스페이스의 자율에 맡기는, 매우 깔끔하고 확장 가능한 구조를 만들 수 있다.

RBAC를 설계하는 것은 플랫폼의 헌법을 제정하는 것과 같다. 처음에는 다소 번거롭고 답답하게 느껴질 수 있지만, 이 최소 권한의 원칙에 따라 잘 설계된 RBAC 체계는 시간이 지남에 따라 당신의 플랫폼을 내부의 실수와 외부의 위협으로부터 지켜주는 가장 든든한 방패가 되어줄 것이다.

이제 우리는 누가 무엇을 할 수 있는지를 통제하는 법을 배웠다. 다음으로는, 설령 권한이 있더라도 그 행동의 '내용'이 안전한지를 검증하여, 위험한 워크로드가 애초에 클러스터에 발을 들이지 못하도록 막는 파드 보안의 세계로 들어가 볼 것이다.