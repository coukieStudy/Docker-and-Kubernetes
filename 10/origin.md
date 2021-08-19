# 10. 보안을 위한 인증과 인가: ServiceAccount와 RBAC
- RBAC(Role Based Access Control): 권한 부여
- ServiceAccount: 사용자 또는 애플리케이션 하나
- RBAC를 통해서 특정 명령을 실행할 수 있는 권한을 ServiceAccount에 부여한다. ServiceAccount는 부여받은 권한에 해당하는 기능만 사용 가능하다.

## 10.1 쿠버네티스의 권한 인증 과정
HumanUser(kubectl) or Pod(service Account) => API Server(authentication -> authorization -> admission control) => 요청 처리
- Authentication(인증): 쿠버네티스의 사용자가 맞는지.
- Authorization(인가): 사용자가 해당 기능을 사용할 권한이 있는지.
- Admission Controller: 요청이 올바른지.

지금까지 이 과정을 고려하지 않아도 된 이유는 설치시 자동으로 kubectl이 관리자 권한을 갖도록 `~/.kube/config`에 설정해두기 때문이다.

## 10.2 ServiceAccount와 Role, Cluster Role
![role_and_clusterrole](https://support.huaweicloud.com/intl/en-us/basics-cce/en-us_image_0261301557.png)
### Service Account
- 한 명의 사용자나 애플리케이션이라고 생각하자.
- 각 네임스페이스에 대해 default service account가 자동으로 생성된다.
```bash
$ kubectl get sa # serviceaccount도 사용가능
NAME    SECRETS AGE
default 1       1d

$ kubectl describe sa default
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-vssmw
Tokens:              default-token-vssmw
Events:              <none>
```
- Service Account 생성/삭제 가능
```bash
$ kubectl create sa sa-example
serviceaccount/sa-example created

$ kubectl get sa
NAME            SECRETS   AGE
default         1         1d
sa-example      1         2s
```
 
 ### Role
 - namespace 내의 오브젝트들에 대한 권한을 정의할 때 사용
 - Role 생성

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: role-example
rules:
- apiGroups: [""] # kubectl api-resources로 확인 가능
  resources: ["pods"] # The pod can be accessed.
  verbs: ["get", "list"] # The GET and LIST operations can be performed.
```

### RoleBinding
- user와 role의 관계를 정의
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rolebinding-example
  namespace: default
subjects: # Specified user
- kind: User # Common user
  name: user-example
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount # ServiceAccount
  name: sa-example
  namespace: default
roleRef: # Specified Role
  kind: Role
  name: role-example
  apiGroup: rbac.authorization.k8s.io
```
![rolebinding](https://support.huaweicloud.com/intl/en-us/basics-cce/en-us_image_0262051194.png)
![cross-namespace access](https://support.huaweicloud.com/intl/en-us/basics-cce/en-us_image_0261301593.png)

 ### Cluster Role & Cluster Role Binding
 - cluster 단위의 권한을 정의할 때 사용
 - namespace 정의 불필요.

 ### Cluster Role Aggregation
- 어떤 ClusterRole A의 aggregationRule.clusterRoleSelectors의 matchLabel과 일치하는 Label을 가진 ClusterRole B가 있을 때, A는 B의 권한을 모두 갖는다.
- https://github.com/alicek106/start-docker-kubernetes/blob/master/chapter10/clusterrole-aggregation.yaml

## 10.3 쿠버네티스 API 서버에 접근

### 10.3.1 ServiceAccount Secret 이용해 쿠버네티스 API 접근
- secret print
```bash
$ kubectl get secrets
NAME                        TYPE                                DATA    AGE
default-token-vssmw         kubernetes.io/service-account-token 3       1d
sa-example-token-xdlvn      kubernetes.io/service-account-token 3       4h
```
- serviceAccount secret은 ca.crt, namespace, token 총 3개의 데이터가 저장되어 있다. 
- ca.crt: 공개 인증서
- namespace: 해당 serviceAccount가 존재하는 namespace
- token: 쿠버네티스 Api와 JWT(Json Web Token) 인증에 사용되는 토큰
- Bearer token 사용해서 api 접근: *curl https://localhost:6443/apis --header "Authorization: Bearer $TOKEN" -k*

### 10.3.2 클러스터 내부에서 kubernetes 서비스를 통해 API 서버에 접근
- kubernetes service: 미리 생성된 클러스터 내부에서 API 서버에 접근할 수 있는 서비스 리소스
```bash
$ kubectl get svc
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes     ClusterIP   10.247.0.1       <none>        443/TCP          34
```
- pod을 생성하면 pod 내부에 자동으로 ServiceAccount의 secret을 마운트한다.
```bash
$ kubectl exec -it sa-example-pod -- /bin/sh
/ # ls /run/secrets/kubernetes.io/serviceaccount
ca.crt     namespace  token
```

### 10.3.3 쿠버네티스 SDK를 이용해 포드 내부에서 API 서버에 접근
- 일반적으로 간단한 테스트가 아닌 이상, curl 을 이용해서 직접 API를 호출하는 경우는 드물고, SDK를 사용하게 된다.
- serviceAccount를 명시적으로 지정해서 pod 생성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-example-pod
spec:  
  serviceAccountName: sa-example
  containers:
  - image: nginx:alpine             
    name: container-0 
```
- pod 실행
```bash
$ kubectl exec -it sa-example-pod -- /bin/sh
/ # vim list-pod-and-svc.py
...
/ # python3 list-pod-and-svc.py
...
```
- pod 내에서 쿠버네티스 API를 사용할 수 있는 코드를 작성: 
```python
from kubernetes import client, config

config.load_kube_config() # get service account token

v1=client.CoreV1Api()

print("Listing pods with their IPs:")
ret = v1.list_namespaced_pod(namespace='default')) 
# success

print("Listing services with their IPs:")
ret = v1.list_namespaced_service(namespace='default')) 
# forbidden: sa-example은 pod list하는 권한만 있음.

```
## 10.4  서비스 어카운트에 이미지 레지스트리 접근을 위한 시크릿 설정
- command line으로 docker-registry type의 시크릿 생성
```bash
kubectl create secret docker-registry registry-auth --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword>
```
- data `.dockerconfigjson` 안에 username, password 등이 인코딩된 상태로 들어가있다.
- `imagePullSecrets`로 service account의 docker-registry type의 시크릿을 지정
```yaml
apiVersion: v1
kine: ServiceAccount
metadata: 
    name: sa-registry-auth
    namespace: default
imagePullSecrets:
- name: registry-auth
```

## 10.5 kubeconfig 파일에 서비스 어카운트 인증 정보 설정
### 10.5.1 kubeconfig 파일
- kubectl 명령어를 사용해 쿠버네티스 클러스터를 제어할 때, kubeconfig파일을 통해 인증을 진행한다.
- ~/.kube/config 파일
```
apiVersion: v1
clusters:
- cluster: //클러스터 접속 정보
    certificate-authority-data: ...
    server: https://localhost:6443
  name: kubernetes

contexts:
- context: // 클러스터와 유저의 조합
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes

current-context: local-context

users:
- name: kubernetes-admin
  user: // 클러스터 인증 정보
    client-certificate-data: ...
    client-key-data:...


kind: Config
preferences: {}
```
### 10.5.2 새로운 context 생성
시크릿 생성 -> 디코딩한 토큰으로 새로운 유저 생성 -> 새로운 유저와 클러스터의 조합인 새로운 context 생성 -> 새로운 context를 사용하도록 설정.
```bash
$ kubectl config set-credetials new-user -token=$decoded_token
$ kubectl config set-context new-context --cluster=kubernetes --user=new-user
$ kubectl config use-context new-context
```

## 10.6 User와 Group의 개념
- service account 외에도 User, Group이 있다.
- User: 실제 사용자
- Group: 여러 유저의 집합
- yaml 파일에서 kind에 ServiceAccount 대신 User or Group을 적으면 된다.
- service Account도 사실 User 중 하나.

**다양한 인증 방법에서의 User, Group**
- 별도의 인증 서버를 사용해서 깃허브, 구글, LDAP의 데이터를 쿠버네티스 사용자 인증에 사용 가능하다.
- https://appscode.com/products/guard/0.2.1/guides/authenticator/github/ 
