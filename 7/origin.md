# 7장 쿠버네티스 리소스의 관리와 설정

## 7.1 네임스페이스(NameSpace): 리소스를 논리적으로 구분하는 장벽

### 쿠버네티스 주요 오브젝트

**포드**: 컨테이너 애플리케이션의 기본 단위

**레플리카셋**: 일정 개수의 포드를 유지하는 컨트롤러

**디플로이먼트**: 컨테이너 어플리케이션을 배포하고 관리하는 역할

**서비스**: 디플로이먼트를 통해 생성된 포드에 접근할 수 있는 수단을 제공하는 요소

### 효율적으로 관리하기 위해 자주 사용되는 것

네임 스페이스(NameSpace)

컨피그맵(ConfigMap)

시크릿(Secret)

### NameSpace: 리소스를 논리적으로 구분하는 장벽

도커를 사용할 때는 컨테이너들을 논리적으로 구분하는 방법이 없었다. 

→ 따라서 docker ps 명령어를 사용하면 모든 컨테이너 목록을 확인

즉, Namespace는 포드, 레플리카셋, 디플로이먼트, 서비스 등과 같은 쿠버네티스 리소스들이 묶여 있는 하나의 가상 공간 또는 그룹이라고 생각하면 된다. (e.g. monitoring, testbed처럼 각자 목적에 맞는 리소스를 구분 가능)

기본적으로 3개의 namespace가 존재 (Default, Kube-public, Kube-system)

- Default: 기본 namespace
- Kube-system: 쿠버네티스 클러스터 구성에 필수적인 컴포넌트들과 설정값 등이 존재
- Kube-public: ?

```bash
kubectl get namespaces # namespace를 줄여서 ns 가능
kubectl get pods --namespace kube-system #Pod 조회 / --namespace를 -n으로 가능
kubectl get service -n kube-system #서비스 조회
```

**주의할 부분: 논리적으로 구분된 것이지, 물리적으로는 분리 X (Linux Namespace와 다름)**

→ 서로 다른 네임스페이스에서 생성된 포드가 같은 노드에 존재할 수 있다.

→ Linux Namespace: process가 네임스페이스를 공유하면 시스템의 리소스를 공유한다

[https://www.44bits.io/ko/keyword/linux-namespace](https://www.44bits.io/ko/keyword/linux-namespace)

[https://stackoverflow.com/questions/61628540/difference-between-kubernetes-namespace-and-linux-namespaces](https://stackoverflow.com/questions/61628540/difference-between-kubernetes-namespace-and-linux-namespaces)

### Namespace 사용하기

```bash
kubectl create namespace production #namespace 생성
kubectl get ns | grep production #namespace 조회
kubectl delete namespace production #namespace 삭제
```

특정 네임스페이스에 리소스 생성

```bash
apiVersion: apps/v1
kind:Deployment
metadata:
	name: hostname-deployment-ns
	namespace: production
spec:
...

--- # YAML에서 동시 정의 가능 - 실행시 kubectl apply -f 사용
apiVersion: v1
kind: Service
metadata:
	name: hostname-svc-clusterip-ns
	namespace: production
spec:
...
```

### 네임스페이스의 서비스에 접근하기

내부에서 서비스 이름을 통해서 접근할 때, 같은 네임스페이스 내의 서비스만 접근 가능

```bash
kubectl run -i --tty -rm debug --image=alicek106/ubuntu:curl --restart=Never -- bash
```

다른 네임스페이스 내로 접근하려면 <서비스 이름>.<네임스페이스 이름>.svc형식으로 네임스페이스 이름을 붙여주면 가능하다.

```bash
hostname-svc-cluterip-ns.production.svc
```

### 네임스페이스에 속하는/속하지 않는 오브젝트

보통 namespace에 속하는 오브젝트 (e.g. service, pod..)

보통 namespace에 속하지 않는 오브젝트: 클러스터 전반에 걸쳐서 사용되는 것들 (e.g. nodes, namespaces)

```bash
kubectl api-resources --namespaced=true #namespace에 속하는 오브젝트의 종류 확인
kubectl api-resources --namespaced=false #namespace에 속하지 않는 오브젝트의 종류 확인
```

## ConfigMap, Secret: 설정값을 Pod에 전달

YAML 파일에 모든 설정값을 하드코딩할 수 없기에 이를 지원하기 위한 ConfigMap(설정값), Secret(비밀값)

**ConfigMap을 사용하는 이점**

<img width="1037" alt="스크린샷_2021-07-28_오후_11 08 09" src="https://user-images.githubusercontent.com/26040955/127475668-c4d550cd-d260-4a3d-9431-fbd01f849dd4.png">


### ConfigMap

ConfigMap 생성: yaml 파일 또는 아래와 같은 명령어로도 생성 가능

```bash
#ConfigMap 생성
kubectl create configmap <컨피그맵 이름> <각종 설정값들>
kubectl create configmap log-level-configmap --from-literal LOG_LEVEL=DEBUG

#ConfigMap 조회
kubectl get configmap #cm으로도 가능
```

### ConfigMap을 Pod에서 사용하는 방법

- Configmap의 값을 컨테이너 환경변수로 사용(e.g. $LOG_LEVEL)
- Configmap의 값을 포드 내부의 파일로 마운트해 사용
    - nginx.conf 등의 파일을 통해 설정값을 읽어들인다면, 이 방법이 좋다

## Secret

SSH 키, 비밀번호 등과 같이 민감한 정보를 저장하기 위한 용도로 사용됨

→ 네임스페이스에 종속되는 쿠버네티스 오브젝트
