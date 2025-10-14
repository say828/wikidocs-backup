## 02\. [실전] Kubebuilder를 이용한 나만의 오퍼레이터 개발

과거에는 커스텀 컨트롤러를 작성하는 것이 쿠버네티스 내부 코드에 대한 깊은 이해를 요구하는 매우 어려운 작업이었다. 하지만 오늘날에는 **Operator SDK**와 **Kubebuilder**와 같은 훌륭한 프레임워크 덕분에, 우리는 비즈니스 로직 자체에만 집중하여 훨씬 더 쉽게 우리만의 오퍼레이터를 개발할 수 있게 되었다.

여기서는 커뮤니티에서 널리 사용되는 **Kubebuilder**를 기반으로 오퍼레이터를 개발하는 과정을 개념적으로 살펴보자. Kubebuilder는 Go 언어를 사용하여 오퍼레이터를 개발하기 위한 프로젝트 구조, 보일러플레이트 코드, 그리고 빌드/배포 도구를 자동으로 생성해주는 CLI 도구다.

개발 과정은 크게 세 단계로 나뉜다.

**1단계: API 정의 (Define the API):**
먼저, `kubebuilder create api` 명령을 사용하여 우리의 CRD, 즉 `PostgreSQLDatabase`의 구조를 Go 코드로 정의한다. 우리는 `spec` 필드(우리가 원하는 상태)와 `status` 필드(컨트롤러가 보고할 현재 상태)를 Go 구조체(struct)로 명시한다.

```go
// api/v1alpha1/postgresqldatabase_types.go

// PostgreSQLDatabaseSpec defines the desired state of PostgreSQLDatabase
type PostgreSQLDatabaseSpec struct {
	Version        string `json:"version"`
	Replicas       int32  `json:"replicas"`
	BackupSchedule string `json:"backupSchedule"`
}

// PostgreSQLDatabaseStatus defines the observed state of PostgreSQLDatabase
type PostgreSQLDatabaseStatus struct {
	ReadyReplicas int32  `json:"readyReplicas"`
	CurrentVersion string `json:"currentVersion"`
	LastBackupTime metav1.Time `json:"lastBackupTime,omitempty"`
}
```

이 Go 코드는 Kubebuilder에 의해 자동으로 CRD YAML 파일로 변환된다.

**2단계: 조정 로직 구현 (Implement the Reconciler):**
이것이 오퍼레이터의 심장이다. `internal/controller/postgresqldatabase_controller.go` 파일 안에, 우리는 \*\*`Reconcile`\*\*라는 이름의 함수를 구현해야 한다. 이 함수는 `PostgreSQLDatabase` 리소스에 변경이 생길 때마다 쿠버네티스에 의해 호출된다.

```go
// internal/controller/postgresqldatabase_controller.go

func (r *PostgreSQLDatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)

	// 1. CRD 인스턴스를 가져온다.
	var pgdb v1alpha1.PostgreSQLDatabase
	if err := r.Get(ctx, req.NamespacedName, &pgdb); err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// 2. 현재 상태를 확인하고, 이상적인 상태와 비교한다.
	// 예를 들어, StatefulSet이 존재하는지 확인한다.
	var sts appsv1.StatefulSet
	err := r.Get(ctx, /* ... */, &sts)

	// 3. 만약 StatefulSet이 없다면? 새로 생성한다.
	if errors.IsNotFound(err) {
		newStatefulSet := r.desiredStatefulSetFor(&pgdb) // 헬퍼 함수로 원하는 상태의 StatefulSet 정의
		if err := r.Create(ctx, &newStatefulSet); err != nil {
			log.Error(err, "Failed to create new StatefulSet")
			return ctrl.Result{}, err
		}
		log.Info("Created new StatefulSet")
		return ctrl.Result{Requeue: true}, nil // 상태가 변경되었으니 즉시 재조정 요청
	}

	// 4. 만약 StatefulSet이 있는데 replicas 수가 다르다면? 업데이트한다.
    // ... update logic ...

	// 5. 모든 상태가 일치하면, 조정을 마친다.
	log.Info("All resources are in the desired state.")
	return ctrl.Result{}, nil
}
```

이 `Reconcile` 함수는 오퍼레이터의 모든 지식이 담기는 곳이다. 여기에는 스테이트풀셋을 만들고, 서비스와 연결하며, 백업을 위한 크론잡을 설정하는 모든 로직이 Go 코드로 구현된다.

**3단계: 빌드 및 배포 (Build & Deploy):**
모든 로직이 완성되면, `make docker-build docker-push`와 `make deploy`와 같은 간단한 명령어로 컨트롤러를 컨테이너 이미지로 빌드하고, 필요한 모든 RBAC, CRD, 디플로이먼트 YAML을 생성하여 클러스터에 배포할 수 있다.

이제 우리의 클러스터에는 `PostgreSQLDatabase`라는 새로운 언어를 이해하고 그에 따라 행동하는 지능적인 로봇이 살게 된 것이다.

하지만 만약 우리가 관리하고 싶은 대상이 쿠버네티스 클러스터 '안'이 아니라, 그 '밖'에 있는 클라우드 서비스라면 어떨까? 다음 절에서는 오퍼레이터 패턴을 한 단계 더 확장하여, Kubernetes API로 세상 모든 것을 관리하려는 야심 찬 프로젝트를 만나본다.