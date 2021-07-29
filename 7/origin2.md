# 7 쿠버네티스 리소스의 관리와 설정

### 7.1 네임스페이스(Namespace) : 리소스를 논리적으로 구분하는 장벽

###### 도커나 도커 스웜 모드를 사용할 때에는 컨테이너를 논리적으로 구분하는 방법이 없었다.

쿠버네티스는 리소스를 논리적으로 구분하기 위해 네임스페이스(Namespace)라는 오브젝트를 제공합니다. 리소스들이 묶여 있는 하나의 가상 공간 또는 그룹이라고 이해하면 됩니다.



#### 네임스페이스 기본 개념 이해

```
$ kubectl get namespace(=ns)
```

<img width="231" alt="image-20210728195105949" src="https://user-images.githubusercontent.com/66865817/127470269-01e598ca-5b0b-481c-8bb2-3ecf224d46d6.png">



Namespace 안에 있는 pod를 확인하려면

```
$ kubectl get pods --namespace(=n) namespaceName
```



<img width="294" alt="image-20210728195230679" src="https://user-images.githubusercontent.com/66865817/127470349-5a48f740-824c-4fc3-92ba-18768e3e904f.png">

<img width="539" alt="image-20210728195356508" src="https://user-images.githubusercontent.com/66865817/127470359-1b3dd33d-e331-4976-b5fc-6786a7d612c7.png">

Kube-system 네임스페이스는 쿠버네티스 클러스터 구성에 필수적인 컴포넌트들과 설정값 등이 존재하는 네임스페이스

default와 논리적으로 구분돼 있기 때문에 지금까지 이 포드들을 본적이 없는 것입니다.

```
Kube-system 네임스페이스는 쿠버네티스에 대한 충분한 이해 없이는 건드리지 않는 것이 좋습니다. 예상치 못하게 쿠버네티스 클러스터가 동작하지 않을 수도 있기 때문입니다.
```

service에는 DNS서버의 서비스가 미리 생성돼 있습니다.

<img width="555" alt="image-20210728201829444" src="https://user-images.githubusercontent.com/66865817/127470461-0d196897-2122-4e16-acd2-38baa60ffc03.png">



논리적으로 구분되어 있지만 물리적으로 격리된 것은 아니라는 점을 알아 둬야 합니다. 예를 들어 서로 다른 네임스페이스에서 생성된 포드가 같은 노드에 존재할 수도 있습니다.

#### 라벨과의 차이점

이전에 서비스와 포드를 매칭시키기 위해 사용했던 라벨 또한 리소스를 분류하고 구분하기 위한 방법 중 하나

app=webserver라는 라벨을 가지는 포드만 출력하려면 다음과 같이 -l 옵션을 사용 할 수 있습니다.

``` 
$ kubectl get pods -l app=webserver
```

네임스페이스는 라벨보다 더 넓은 용도로 사용 가능

예를 들어, ResourceQuota라는 오브젝트를 이용해 특정 네임스페이스에서 생성되는 포드의 자원 사용량을 제한하거나, 애드미션 컨트롤러라는 기능을 이용해 특정 네임스페이스에 생성되는 포드에는 항상 사이드카 컨테이너를 붙이도록 설정할 수 있습니다. 사용 목적에 따라 포드, 서비스 등의 리소스를 격리함으로써 편리하게 구분할 수 있다는 특징도 있습니다.

#### 네임스페이스 사용하기

YAML파일에 정의해 생성할 수 있습니다.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```
$ kubectl apply -f production-namespace.yaml
```

또는

```
$ kubectl create namespace production
```



특정 네임스페이스에서 리소스를 생성하는 방법

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostname-deployment-ns
  namespace: production
spec:
  ...
---
apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-clusterip-ns
  namespace: production
spec:
  ...
```



```
하나의 YAML파일에 ---를 명시해 여러개의 리소스를 정의할 수 있습니다.
```



전체 네임스페이스 조회하고 싶으면 --all-namespaces 사용

<img width="631" alt="image-20210728203059483" src="https://user-images.githubusercontent.com/66865817/127474256-369db4e2-db79-4d18-b7bb-9cc068cb644d.png">



#### 네임스페이스의 서비스에 접근하기

쿠버네티스 클러스터 내부에서는 서비스 이름을 통해 포드에 접근할 수 있다 = 같은 네임스페이스 내의 서비스에 접근할 때에는 서비스 이름만으로 접근할 수 있다는 뜻

<img width="637" alt="image-20210728220729637" src="https://user-images.githubusercontent.com/66865817/127474312-a9171e18-847f-4b16-bac0-a95a1f47cc9f.png">

```
<p>Hello, hostname-deployment-ns-6965678d58-t48r6<\p>  <\blockquote>
```

```
서비스의 DNS이름에 대한 FQDN(Fully Qualified Domain Name)은 일반적으로 다음과 같은 형식으로 구성돼 있습니다.
<서비스 이름>.<네임스페이스 이름>.svc.cluster.local
```



네임스페이스는 kubectl delete -f <YAML 파일명> 또는 kubectl delete namespace 명령어로 삭제 가능

네임스페이스에 존재하는 모든 리소스 또한 함께 삭제되기 때문에 네임스페이스를 삭제하기 전에 다시 한번 리소스 목록을 확인해보는 것이 좋습니다.

<img width="309" alt="image-20210728221220289" src="https://user-images.githubusercontent.com/66865817/127474383-8c151dc1-7363-4777-9881-b8c2584e0175.png">



#### 네임스페이스에 종속되는 쿠버네티스 오브젝트와 독립적인 오브젝트

네임스페이스를 사용하면 쿠버네티스 리소스를 사용 목적에 따라 논리적으로 격리할 수 있지만, 모든 리소스가 네임스페이스에 의해 구분되는 것은 아닙니다.

포드, 서비스, 리플리카셋, 디플로이먼트는 네임스페이스 단위로 구분할 수 있습니다. A네임스페이스에서는 보이고 B네임스페이스에서는 보이지 않습니다. 이런 경우를 **오브젝트가 네임스페이스에 속한다(namespaced)**고 표현합니다. 네임스페이스에 속하는 오브젝트의 종류는 다음 명령어로 확인할 수 있습니다.

<img width="796" alt="image-20210728221534699" src="https://user-images.githubusercontent.com/66865817/127474440-cde5719b-9b6a-46c1-9e1b-e3fb531bfe40.png">

반대로 속하지 않는 오브젝트도 확인할 수 있습니다.

<img width="915" alt="image-20210728221624772" src="https://user-images.githubusercontent.com/66865817/127474470-c8286eeb-63d9-472e-877f-17b88c50ac62.png">


클러스터 관리를 위한 저수준의 오브젝트들은 네임스페이스에 속하지 않을 수도 있다는 것만 알아두면 됩니다.



### 7.2 컨피그맵(Configmap), 시크릿(Secret) : 설정값을 포드에 전달

애플리케이션은 대부분 설정값을 가지고 있을 것입니다. 예를 들어 애플리케이션의 로깅 레벨을 정의하는 LOG_LEVEL=INFO 와 같이 단순한 key-value 형태의 설정을 사용할 수도 있고, Nginx 웹 서버가 사용하는 nginx.conf처럼 완전한 하나의 파일을 사용해야 할 수도 있습니다.

이러한 설정값이나 설정 파일을 여러분의 애플리케이션에 전달하는 가장 확실한 방법은 도커 이미지 내부에 설정값 또는 설정 파일을 정적으로 저장해 놓는 것입니다. 하지만 도커 이미지는 일단 빌드되고 나면 불변의 상태를 가지기 때문에 이 방법은 상황에 따라 설정 옵션을 유연하게 변경할 수 없다는 단점이 있습니다.

이에 대한 대안으로 포드를 정의하는 YAML 파일에 환경 변수를 직접 적어 놓는 하드 코드 방식을 사용할 수도 있습니다. 예를 들어 디플로이먼트의 YAML 파일 중 포드 템플릿에서 아래와 같은 방식으로 환경 변수를 직접 설정할 수도 있습니다. 아래의 예시는 포드의 LOG_LEVEL이라는 이름의 환경변수를 INFO라는 값으로 설정합니다.

```yaml
...
  spec:
    containers:
      - name: nginx
        env:
        - name: LOG_LEVEL
          value: INFO
        image: nginx:1.10
...
```

포드 템플릿에 직접 명시하는 방법도 좋지만, 운영과 개발환경에서 각각 디플로이먼트를 생성하면 두가지 YAML이 존재해야합니다.

쿠버네티스는 YAML 파일 설정값을 분리할 수 있는 **컨피그맵(Configmap)**과 **시크릿(secret)**이라는 오브젝트를 제공합니다. 컨피그 맵에는 설정값을, 시크릿에는 노출되어서는 안 되는 비밀값을 저장할 수 있습니다.

![IMG_6068](https://user-images.githubusercontent.com/66865817/127474564-c6a2bcbd-39f9-4b5c-b8d7-65c19b126017.PNG)


#### 7.2.1 컨피그맵(Configmap)

##### 컨피그맵 사용 방법 익히기

컨피그맵은 일반적인 설정값을 담아 저장할 수 있는 쿠버네티스 오브젝트이며, 네임스페이스에 속하기 때문에 네임스페이스별로 컨피그맵이 존재합니다.

컨피그맵을 생성하는 방법은 매우 간단합니다. YAML 파일을 사용해 컨피그 맵을 생성해도 되지만, kubectl create configmap 명령어를 사용하면 쉽게 컨피그맵을 생성할 수 있습니다. 예를 들어 다음 명령어는 --from-literal이라는 옵션을 이용해 LOG_LEVEL 키의 값이 DEBUG인 컨피그맵을 생성하며, 컨피그맵의 이름은 log-level-configmap이 됩니다.

```
$ kubectl create configmap <컨피그맵 이름> <각종 설정값들>
```

<img width="562" alt="image-20210728225000004" src="https://user-images.githubusercontent.com/66865817/127474621-2a6c6e80-1dee-44e2-9954-16b850d5f741.png">

<img width="382" alt="image-20210728225020866" src="https://user-images.githubusercontent.com/66865817/127474648-200f95e5-e44d-4aeb-b11e-1cf5d6e379c0.png">


```
$ kubectl create configmap start-k8s --from-literal k8s=kubernetes --from-literal container=docker
```



컨피그맵을 생성했다면 다음은 컨피그맵의 값을 포드로 가져와 볼 차례입니다. 생성된 컨피그맵을 포드에서 사용하려면 디플로이먼트 등의 YAML 파일에서 포드 템플릿 항목에 컨피그맵을 사용하도록 정의하면 됩니다.

![IMG_6069 복사본](https://user-images.githubusercontent.com/66865817/127474676-e9876879-50e9-4288-b48e-7bb2fce35b2e.png)

그러나 컨피그맵을 사용하는 포드를 생성하기 전에, 컨피그맵을 포드에서 어떻게 사용하는지에 대해 먼저 살펴보겠습니다. 방법은 크게 두 가지가 있습니다.

- 컨피그맵의 값을 컨테이너의 환경변수로 사용

  시스템 환경 변수로부터 설정값을 가져온다면 이 방법을 사용하는 것이 좋습니다.

- 컨피그맵의 값을 포드 내부의 파일로 마운트해 사용

  nginx.cof등의 파일을 통해 설정값을 읽어들인다면 이 방법을 사용하는 것이 좋습니다.

##### 컨피그맵의 데이터를 컨테이너의 환경 변수로 가져오기

컨피그맵의 값을 환경 변수로 사용하는 포드를 생성해보겠습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: container-env-example
spec:
  containers:
   - name: my-container
     image: busybox
     args: ['tail', '-f', '/dev/null']
     envFrom:
     - configMapRef:
        name: log-level-config   <- key-value가 1개 존재하는 컨피그맵
     - configMapRef:
        name: start-k8s	 	   	   <- key-value가 2개 존재하는 컨피그맵
```

```
실제 운영환경에서는 대부분의 경우 디플로이먼트를 사용하지만, 책에서는 포드를 정의하는 YAML파일 기준으로 설명할 것, 이는 포드를 사용하는 쿠버네티스 오브젝트에 적용해 사용할 수 있다는 것을 의미합니다.
```

envFrom 항목은 하나의 컨피그맵에 여러개의 키-값 쌍이 존재하더라도 모두 환경 변수로 가져오도록 설정합니다. 따라서 총 3개의 키-값 쌍을 포드로 넘긴 셈입니다.

```
$ kubectl apply -f all-env-from-configmap.yaml
pod/container-env-exapmle created

$ kubectl exec container-env-example env
..
LOG_LEVEL=DEBUG   # 설정한 환경 변수
container=docker  # 설정한 환경 변수
k8s=kubernetes    # 설정한 환경 변수
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10/96.0.1:443
...
```

```
KUBERNETES_SERVICE_PORT처럼 여러개의 환경변수가 미리 설정된 것을 볼 수 있는데, 이는 쿠버네티스가 자동으로 서비스에 대한 접근 정보를 컨테이너의 환경 변수로 설정하기 때문
```



다른 방법으로 포드 생성

valueFrom과 configMapKeyRef를 사용하면 여러 개의 키-값 쌍이 들어있는 컨피그맵에서 특정 데이터만을 선택해 환경 변수로 가져올 수도 있습니다. Env 항목을 제외한 부분은 위의 YAML 파일과 동일하므로 파일의 일부분을 생략했습니다.

```
...
  env:
  - name: EVN_KEYNAME_1  # (1.1) 컨테이너에 새롭게 등록될 환경 변수 이름
    valueFrom:
      configMapKeyRef:
        name: log-level-configmap
        key: LOG_LEVEL
  - name: ENV_KEYNAME_2 # (1.2) 컨테이너에 새롭게 등록될 환경 변수 이름
    valueFrom:
      configMapKeyRef:
        name: start-k8s	# (2) 참조할 컨피그맵의 이름
        key: k8s				# (3) 가져올 데이터 값의 키
                        # 최종 결과 -> ENV_KEYNAME_2=$(k8s 키에 해당하는 값)
												#            ENV_KEYNAME_2=kubernetes
```

```
$ kubectl apply -f selective-env-from-configmap.yaml
pod/container-selective-env-example created
```



##### 컨피그맵의 내용을 파일로 포드 내부에 마운트하기

nginx.conf, mysql.conf 등과 같은 특정 파일로부터 설정 값을 읽어온다면 컨피그맵의 데이터를 포드 내부의 파일로 마운트해 사용할 수 있습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-volume-pod
spec:
    containers:
      - name: my-container
        image: busybox
        args: ["tail", "-f", "/dev/null"]
        volumeMounts:
        - name: configmap-volume		# volumes에서 정의한 컨피그맵 볼륨 이름
          mountPath: /etc/config		# 컨피그맵의 데이터가 위치할 경로

    volumes:
      - name: configmap-volumes			# 컨피그맵 볼륨 이름
      configMap:
        name: start-k8s							# 키-값 쌍을 가져올 컨피그맵 이름
```



```
$ kubectl apply -f volume-mount-configmap.yaml
pod/configmap-volume-pod created

$ kubectl exec configmap-volume-pod ls /etc/config
container
k8s

$ kubectl exec configmap-volume-pod cat /etc/config/k8s
kubernetes
```

컨피그맵에 저장돼 있던 두 개의 키-값 데이터가 파일로 존재하고 있음



여러개 중에 원하는 것만 가져오는 방법

```yaml
...
        volumeMounts:
        - name: configmap-volume		# volumes에서 정의한 컨피그맵 볼륨 이름
          mountPath: /etc/config		# 컨피그맵의 데이터가 위치할 경로
          
    volumes:
      - name: configmap-volumes			# 컨피그맵 볼륨 이름
      configMap:
        name: start-k8s							
        items:											# 컨피그맵에서 가져올 키-값의 목록을 나열합니다.
         - key: k8s									# k8s라는 키에 대응하는 값을 가져옵니다.
           path: k8s_fullname				# 최종 파일 이름은 k8s_fullname이 됩니다.
```



```
$ kubectl apply -f volume-mount-configmap.yaml
pod/configmap-volume-pod created

$ kubectl exec configmap-volume-pod ls /etc/config
k8s_fullname

$ kubectl exec configmap-volume-pod cat /etc/config/k8s_fullname
kubernetes
```



##### 파일로부터 컨피그맵 작성하기

```
$ kubectl create configmap <컨피그맵 이름> --from-file <파일 이름>
```

![image-20210729174539238](https://user-images.githubusercontent.com/66865817/127474806-3ac38100-7872-40e5-b65a-e4fbcf440e8d.png)


```
$ cat multiple-keyvalue.env
mykey1=myvalue1
mykey2=myvalue2
mykey3=myvalue3

$ kubectl create configmap from-envfile --from-env-file multiple-keyvalue.env
configmap/from-env-file created

$ kubectl get cm from-envfile -o yaml
apiVersion: v1
data:
	mykey1: myvalue1
	mykey2: myvalue2
	mykey3: myvalue3
...
```



##### YAML 파일로 컨피그맵 정의하기

```
$ kubectl create configmap my-configmap --from-literal mykey=myvalue --dry-run -o yaml

apiVersion: v1
data:
	mykey: myvalue
kind: ConfigMap
metadata:
	creationTimestamp: null
	name: my-configmap
```

출력 내용을 YAML 파일로 저장한 뒤, kubectl apply 명령어로 컨피그맵을 생성하면 됩니다.

```
$ kubectl create configmap my-configmap --from-literal mykey=myvalue --dry-run -o yaml > my-configmap.yaml

$ kubectl apply -f configmap.yaml
configmap/my-configmap created
```



#### 7.2.2 시크릿(Secret)

##### 시크릿 사용 방법 익히기

시크릿은 네임스페이스에 종속되는 쿠버네티스 오브젝트

```
$ kubectl create secret generic my-password --from-literal password=1q2w3e4r
secret/my-password created
```

컨피그맵에서 사용한 옵션 사용 가능

![image-20210729181137577](https://user-images.githubusercontent.com/66865817/127474840-ce8c6899-ad47-422a-882d-e0e72711e908.png)
![image-20210729181219713](https://user-images.githubusercontent.com/66865817/127474837-323e230e-dda2-45c0-a9ae-51509025b58d.png)


