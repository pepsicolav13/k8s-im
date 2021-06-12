# 11장 워크플로우 관리


### Custom Resource

```yaml
# mypod-crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: mypods.crd.example.com
spec:
  group: crd.example.com
  version: v1
  scope: Namespaced
  names:
    plural: mypods   # 복수 이름
    singular: mypod  # 단수 이름
    kind: MyPod      # Kind 이름
    shortNames:      # 축약 이름
    - mp
```

```bash
# crd 생성
kubectl apply -f mypod-crd.yaml
# customresourcedefinition.apiextensions.k8s.io/mypods.crd.example.com created

# crd 리소스 확인
kubectl get crd | grep mypods
# NAME                     CREATED AT          
# mypods.crd.example.com   2020-06-14T09:33:32Z          
```

```bash
# MyPod 리소스 생성
cat << EOF | kubectl apply -f -
apiVersion: "crd.example.com/v1"
kind: MyPod
metadata:
  name: mypod-test
spec:
  uri: "any uri"
  customCommand: "custom command"
  image: nginx
EOF
# mypod.crd.example.com/mypod-test created

kubectl get mypod
# NAME         AGE
# mypod-test   3s

# 축약형인, mp로도 조회가 가능합니다.
kubectl get mp
# NAME         AGE
# mypod-test   3s

# MyPod의 상세 정보를 조회합니다.
kubectl get mypod mypod-test -oyaml
# apiVersion: crd.example.com/v1
# kind: MyPod
# metadata:
#   ...
#   name: mypod-test
#   namespace: default
#   resourceVersion: "723476"
#   selfLink: /apis/crd.example.com/v1/namespaces/default/mypods/mypod-test
#   uid: 50dd0cc8-0c1a-4f43-854b-a9c212e2046d
# spec:
#   customCommand: custom command
#   image: nginx
#   uri: any uri

# MyPod를 삭제합니다.
kubectl delete mp mypod-test
# mypod.crd.example.com "mypod-test" deleted
```

### Custom Controller

```bash
# MyPod 정의
struct MyPod {
  Uri string
  CustomCommand string
	...
}


def main {
	# 무한루프
    for {
    	# 신규 이벤트
        desired := apiServer.getDesiredState(MyPod)
        # 기존 이벤트
        current := apiServer.getCurrentState(MyPod)
        
        # 변경점 발생시(수정,생성,삭제), 특정 동작 수행
        if desired != current {
            makeChanges(desired, current)	
        }
    }
}
```

## Argo workflow

```bash
kubectl create namespace argo
# namespace/argo created

# Workflow CRD 및 Argo controller 설치
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/v2.8.1/manifests/install.yaml
# customresourcedefinition.apiextensions.k8s.io/clusterworkflowtemplates.argoproj.io created
# customresourcedefinition.apiextensions.k8s.io/cronworkflows.argoproj.io created
# customresourcedefinition.apiextensions.k8s.io/workflows.argoproj.io created
# ...

# default 서비스계정에 admin 권한 부여
kubectl create rolebinding default-admin --clusterrole=admin \
      --serviceaccount=default:default
# rolebinding.rbac.authorization.k8s.io/default-admin created

# ingress 설정
cat << EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: argo-server
  namespace: argo
spec:
  rules:
  - host: argo.10.0.1.1.sslip.io
    http:
      paths:
      - backend:
          serviceName: argo-server
          servicePort: 2746
        path: /
EOF
```

```bash
kubectl get workflow  # wf
# NAME              STATUS      AGE
# fantastic-tiger   Succeeded   10m

kubectl describe workflow fantastic-tiger
# ...
```

```yaml
# single-job.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-
  namespace: default
spec:
  entrypoint: whalesay
  templates:
  - name: whalesay
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["hello world"]
      resources:
        limits:
          memory: 32Mi
          cpu: 100m
```

```bash
# Workflow 생성
kubectl create -f single-job.yaml
# workflow.argoproj.io/hello-world-tcnjj created

kubectl get wf
# NAME                STATUS      AGE
# wonderful-dragon    Succeeded   8m21s
# hello-world-tcnjj   Succeeded   17s

kubectl get pod
# NAME                READY   STATUS      RESTARTS   AGE
# wonderful-dragon    0/2     Completed   0          8m34s
# hello-world-tcnjj   0/2     Completed   0          31s

kubectl logs hello-world-tcnjj -c main
#  _____________
# < hello world >
#  -------------
#     \
#      \
#       \
#                     ##        .
#               ## ## ##       ==
#            ## ## ## ##      ===
#        /""""""""""""""""___/ ===
#   ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
#        \______ o          __/
#         \    \        __/
#           \____\______/
# 
# 
# Hello from Docker!
# This message shows that your installation appears to be working 
```

### 파라미터 전달

```yaml
# param.yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-parameters-
  namespace: default
spec:
  entrypoint: whalesay
  arguments:
    parameters:
    - name: message
      value: hello world through param

  templates:
  ###############
  # entrypoint
  ###############
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]
```

### Serial step 실행

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: serial-step-
  namespace: default
spec:
  entrypoint: hello-step
  templates:

  ###############
  # template job
  ###############
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]

  ###############
  # entrypoint
  ###############
  - name: hello-step
    # 순차 실행
    steps:
    - - name: hello1
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello1"
    - - name: hello2
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello2"
    - - name: hello3
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello3"
```

### Parallel step 실행

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: parallel-steps-
  namespace: default
spec:
  entrypoint: hello-step
  templates:

  ###############
  # template job
  ###############
  - name: whalesay
    inputs:
      parameters:
      - name: message
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]

  ###############
  # entrypoint
  ###############
  - name: hello-step
    # 병렬 실행
    steps:
    - - name: hello1
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello1"
    - - name: hello2
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello2"
      - name: hello3        # 기존 double dash에서 single dash로 변경
        template: whalesay
        arguments:
          parameters:
          - name: message
            value: "hello3"
```

- double-dash: 앞의 job을 이어서 순차적으로 실행(직렬)
- single-dash: 앞의 job과 동시에 병렬적으로 실행(병렬)

### 복잡한 DAG 실행

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond-
  namespace: default
spec:
  entrypoint: diamond
  templates:
  
  ###############
  # template job
  ###############
  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]

  ###############
  # entrypoint
  ###############
  - name: diamond
    # DAG 구성
    dag:
      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        dependencies: [B, C]
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
```

### Clean up

```bash
kubectl delete wf --all
kubectl delete -n argo -f https://raw.githubusercontent.com/argoproj/argo/v2.8.1/manifests/install.yaml
```
