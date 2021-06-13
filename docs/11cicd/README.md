# 10장 CI / CD

## CI 파이프라인 (Github Action)


```bash
# checkout
git clone $PROJECT
git checkout $BRANCH

# build
docker build . -t $USERNAME/$PROJECT

# test
docker run --entrypoint /test $USERNAME/$PROJECT

# push
docker login --username $USERNAME --password $PASSWORD
docker push $USERNAME/$PROJECT
```


```yaml
# .github/workflows/main.yml
name: kubernetes CI/CD
on: push

jobs:
  build:
    name: Hello world action
    runs-on: ubuntu-latest    
    steps:
    - name: checkout source code
      uses: actions/checkout@main

    - name: Build the Docker image
      run: docker build . -t $GITHUB_ACTOR/app:$GITHUB_RUN_NUMBER

    - name: Test image
      run: docker run --entrypoint /test $GITHUB_ACTOR/app:$GITHUB_RUN_NUMBER

    - name: login
      run: docker login -u $GITHUB_ACTOR -p $PASSWORD
      env:
        PASSWORD: ${{ secrets.PASSWORD }}

    - name: push
      run: docker push $GITHUB_ACTOR/app:$GITHUB_RUN_NUMBER

    - name: change gitops config
      run:
```

## GitOps를 이용한 CD

### ArgoCD

```bash
kubectl create namespace argocd
# namespace/argocd created

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
# customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
# serviceaccount/argocd-application-controller created
# serviceaccount/argocd-dex-server created
# ...
```

```yaml
# argocd-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: http-issuer
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.10.0.1.1.sslip.io
    http:
      paths:
      - path: /
        backend:
          serviceName: argocd-server
          servicePort: https
  tls:
  - hosts:
    - argocd.10.0.1.1.sslip.io
    secretName: argocd-tls
```

```bash
kubectl apply -f argocd-ingress.yaml
# ingress.networking.k8s.io/argocd created
```

```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server \
    -o name | cut -d'/' -f 2
# argocd-server-6766455855-pzdrv    --> 비밀번호
```

#### ArgoCD 배포

마찬가지로 앞에서 배포한 `gitops-get-started` 레포지토리를 ArgoCD에서도 원천으로 사용해 봅시다.

- `Application Name`: gitops-get-started
- `Project`: default
- `Sync Policy`: Manual
- `Repository URL`: ` `
- `Revision`: HEAD
- `Path`: .
- `Cluster`: in-cluster
- `Namespace`: default
- `Directory Recurse`: Unchecked

### Clean up

```bash
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete ingress argocd -n argocd
kubectl delete deploy mynginx
kubectl delete svc mynginx
```


## 로컬 쿠버네티스 개발

#### skaffold 설치

```bash
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
sudo install skaffold /usr/local/bin/
```

### skaffold 세팅

```python
# app.py
from flask import Flask
app = Flask(__name__)


@app.route('/')
def hello():
    return "Hello World!"

if __name__ == '__main__':
    app.run()
```

- 도커파일

```Dockerfile
# Dockerfile
FROM python:3.7

RUN pip install flask
ADD app.py .

ENTRYPOINT ["python", "app.py"]
```

- YAML 정의서

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: skaffold-flask
spec:
  containers:
  - image: <USERNAME>/flask   # 각 사용자 docker hub 계정을 입력합니다.
    name: flask
```

```bash
ls
# app.py    Dockerfile    pod.yaml
```

```bash
skaffold init
# apiVersion: skaffold/v2beta5
# kind: Config
# metadata:
#   name: some-directory
# build:
#   artifacts:
#   - image: <USERNAME>/flask
# deploy:
#   kubectl:
#     manifests:
#     - pod.yaml
# 
# Do you want to write this configuration to skaffold.yaml? [y/n]: y
# Configuration skaffold.yaml was written

ls 
# app.py    Dockerfile    pod.yaml    skaffold.yaml
```

```bash
docker login
# Login with your Docker ID to push and pull images from Docker Hub. ..
# Username: <USERNAME>
# Password:
# WARNING! Your password will be stored unencrypted in ...
# Configure a credential helper to remove this warning. See
# https://docs.docker.com/engine/reference/commandline/...
# 
# Login Succeeded
```

### skaffold를 이용한 배포

```bash
skaffold run
# Generating tags...
#  - <USERNAME>/flask -> <USERNAME>/flask:latest
# Some taggers failed. Rerun with -vdebug for errors.
# ...
```

```bash
kubectl get pod
# NAME             READY   STATUS              RESTARTS   AGE
# skaffold-flask   0/1     ContainerCreating   0          2m21s
```

```bash
# 기존 pod 삭제
kubectl delete pod skaffold-flask
# pod "skaffold-flask" deleted

skaffold run --tail
# ...
# Press Ctrl+C to exit
# [skaffold-flask flask] starting app
# [skaffold-flask flask]  * Serving Flask app "app" (lazy loading)
# [skaffold-flask flask]  * Environment: production
# [skaffold-flask flask]    WARNING: This is a development server. Do not 
#                                  use it in a production deployment.
# [skaffold-flask flask]    Use a production WSGI server instead.
# [skaffold-flask flask]  * Debug mode: off
# [skaffold-flask flask]  * Running on http://127.0.0.1:5000/ 
#                                  (Press CTRL+C to quit)
```

```bash
# 기존 pod 삭제
kubectl delete pod skaffold-flask
# pod "skaffold-flask" deleted

skaffold dev
```

```python
from flask import Flask
app = Flask(__name__)


@app.route('/')
def hello():
    return "Hello World!2"

if __name__ == '__main__':
    print('start app')     # print문 추가 후 저장
    app.run()
```

```bash
skaffold dev
# ...
# Press Ctrl+C to exit
# [skaffold-flask flask] starting app
# [skaffold-flask flask]  * Serving Flask app "app" (lazy loading)
# [skaffold-flask flask]  * Environment: production
# [skaffold-flask flask]    WARNING: This is a development server. Do not 
#                                  use it in a production deployment.
# [skaffold-flask flask]    Use a production WSGI server instead.
# [skaffold-flask flask]  * Debug mode: off
# [skaffold-flask flask]  * Running on http://127.0.0.1:5000/ 
#                                  (Press CTRL+C to quit)
```