# puppypod-kubebuilder

- controller manager를 정의할 디렉토리를 생성합니다.
  - ex. mkdir -p cronjob-kubebuilder
- 생성한 하위 디렉토리에 접근하여 kubebuilder project를 생성합니다.
  - ex. kubebuilder init --domain puppypod.kubebuilder.io --repo puppypod.kubebuilder.io/project
  - 특별히 프로젝트 이름은 지정하지 않았는데, --project-name=<dns1123-label-string> 과 같이 옵션을 추가하지 않으면 폴더의 이름이 기본적으로 프로젝트 이름이 됩니다.
    - Makefile은 custom controller를 build하고 deploy할 수 있습니다.
    - config 디렉토리 아래에는 Kustomize 로 작성되어 CustomResourceDefinition, RBAC, WebhookConfiguration 등의 yaml 파일들이 정의됩니다.
    - 특히, config/manager 디렉토리에는 Cluster 에 Custom Controller 를 파드 형태로 배포할 수 있는 Kustomize yaml 이 있고, config/rbac 디렉토리에는 서비스 어카운트로 Custom Controller 의 권한이 정의되어 있습니다.
    - Custom Controller 의 Entrypoint 는 cmd/main.go 파일입니다.
```
✗ tree -L 2
.
├── Dockerfile
├── Makefile
├── PROJECT
├── README.md
├── cmd
│   └── main.go
├── config
│   ├── default
│   ├── manager
│   ├── prometheus
│   └── rbac
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── test
├── e2e
└── utils
```
- 처음 필요한 모듈을 임포트 한 것을 보면 아래 2개가 보입니다.
  - core controller-runtime 라이브러리
  - 기본 controller-runtime 로깅인 Zap
```go
package main

import (
    "flag"
    "fmt"
    "os"

    _ "k8s.io/client-go/plugin/pkg/client/auth"

    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"
    _ "k8s.io/client-go/plugin/pkg/client/auth/gcp"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/cache"
    "sigs.k8s.io/controller-runtime/pkg/healthz"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    // +kubebuilder:scaffold:imports
)
```
모든 컨트롤러에는 scheme이 필요하며, kind와 Go Type간의 매핑을 지원해줍니다.
```go
var (
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)

func init() {
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))

	//+kubebuilder:scaffold:scheme
}
```
- 쿠버네티스에서 API 에 대해서 이야기할 때는 groups, versions, kinds and resources 4개의 용어를 사용합니다.
  - API Group 은 단순히 관련 기능의 모음
  - Group 에는 하나 이상의 Version 이 있으며, 이름에서 알 수 있듯이 시간이 지남에 따라 API의 작동 방식을 변경할 수 있음
  - API group-version 에는 하나 이상의 API type 이 포함되며, 이를 Kind 라고 부릅니다.
  - Kind 는 Version 간에 양식을 변경할 수 있지만, 각 양식은 어떻게든 다른 양식의 모든 데이터를 저장할 수 있어야 함
    - 즉, 이전 API 버전을 사용해도 최신 데이터가 손실되거나 손상되지 않습니다.
  - Resource 란 간단히 말해서 API 안에서 Kind 를 사용하는 것
    - Kind 와 Resource 는 일대일로 매핑. ex) Pod Resource 는 Pod Kind 에 해당
    - 그러나 때로는 여러 Resource 에서 동일한 Kind를 반환할 수도 있다. 
      - Scale Kind 는 deployments/scale 또는 replicasets/scale 과 같은 모든 scale 하위 리소스에 의해 반환
      - 이것이 바로 Kubernetes HorizontalPodAutoscaler 가 서로 다른 resource 와 상호 작용할 수 있는 이유 
      - 그러나 CRD를 사용하면 각 Kind 는 단일 resource 에 해당
  - 아래 명령으로 새로운 Kind 를 추가
```shell
$ kubebuilder create api --group batch --version v1 --kind CronJob
```
```
✗ tree -L 2
.
├── Dockerfile
├── Makefile
├── PROJECT
├── README.md
├── api
│   └── v1
├── bin
│   └── controller-gen-v0.14.0
├── cmd
│   └── main.go
├── config
│   ├── crd
│   ├── default
│   ├── manager
│   ├── prometheus
│   ├── rbac
│   └── samples
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── internal
│   └── controller
├── test
│   ├── e2e
│   └── utils
└── vendor
    ├── github.com
    ├── go.uber.org
    ├── golang.org
    ├── gomodules.xyz
    ├── google.golang.org
    ├── gopkg.in
    ├── k8s.io
    ├── modules.txt
    └── sigs.k8s.io
```
이 경우 batch.puppypod.kubebuilder.io/v1 에 해당하는 api/v1 디렉토리가 생성됩니다.

- 웹훅 생성
```shell
$ kubebuilder create webhook --group batch --version v1 --kind CronJob --defaulting --programmatic-validation
```

# 참고문헌
- https://ssup2.github.io/blog-software/docs/programming/kubernetes-kubebuilder/#27-memcached-controller-local-%ea%b5%ac%eb%8f%99
- https://devocean.sk.com/blog/techBoardDetail.do?ID=165109