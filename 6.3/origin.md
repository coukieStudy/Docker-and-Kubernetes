## 6.4 Deployment

#### 6.4.1 Deployment 사용하기

```yaml
apiVersion: apps/v1
kind: Deployment							# Kind = Deployment
metadata:
  name: my-nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx							# pod의 nginx
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.10
        ports:
        - containerPort: 80

kubectl apply -f deployment-nginx.yaml 로 실행
```

포드 3개, 레플리카 셋은 동일하게 생성. 그 이외에 deploy 라는 서비스가 생성됨

```bash
kubectl get pods


NAME                                   READY   STATUS    RESTARTS   AGE
my-nginx-deployment-85d657c94d-dw4fc   1/1     Running   0          43s
my-nginx-deployment-85d657c94d-h5cbv   1/1     Running   0          43s
my-nginx-deployment-85d657c94d-xlf69   1/1     Running   0          43s
```

deploy가 생성되면서 replica set이 생성되었고, replica set이 생성되면서 pods가 생성되었기 때문에 deploy를 지우면 replica set, pods 가 모두 지워진다.



#### 6.4.2 Deployment를 사용하는 이유

Deployment 는 컨테이너 어플리케이션을 배포하고 관리하는 역할을 한다. 예를 들어 어플리케이션을 업데이트 할 때 변경 사항을 저장하는 revision을 남겨서 롤백을 가능하게 해주고, 포드의 롤링 업데이트 전략을 지정할 수도 있다.

```bash
kubectl apply -f deployment-nginx.yaml --record 		#Deployment 생성
	#nginx 라는 이미지를 가진 pod의 이미지를 nginx:1.11로 변경

kubectl get replicasets

NAME                             DESIRED   CURRENT   READY   AGE
my-nginx-deployment-67fff69f59   3         3         3       14s
my-nginx-deployment-85d657c94d   0         0         0       7m7s

# Revision 정보 확인
kubectl rollout history deployment my-nginx-deployment

#Rollback
kubectl rollout undo deployment my-nginx-deployment --to-revision={revision_version}
```

무중단 배포를 위한 롤링 업데이트 전략은 11장에서 다시 다룸.



## 6.5 Service

디플로이먼트를 통해 생성된 포드에 접근할 수 있는 수단을 제공하는 요소가 Service

-> Docker는 -p 옵션으로 외부에 포트를 노출할 수 있었음.

공통으로 사용되는 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostname-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver			#selector.matchLables.app == template.metadata.labes.app
  template:
    metadata:
      name: my-webserver
      labels:
        app: webserver
    spec:
      containers:
      - name: my-webserver
        image: alicek106/rr-test:echo-hostname # 컨테이너의 hostname 을 반환하는 웹서버
        ports:
        - containerPort: 80

```

```bash
kubectl apply -f deployment-hostname.yaml
kubectl get pods -o wide

NAME                                   READY   STATUS    RESTARTS   AGE   IP        
hostname-deployment-6965678d58-496jz   1/1     Running   0          25s   10.1.0.16  
hostname-deployment-6965678d58-g84dd   1/1     Running   0          25s   10.1.0.18   
hostname-deployment-6965678d58-tnwl4   1/1     Running   0          25s   10.1.0.17  

kubectl run -i --tty --rm debug --image=alicek106/ubuntu:curl --restart=Never

root@debug:/# curl 10.1.0.16
<!DOCTYPE html>
<meta charset="utf-8" />
<link rel="stylesheet" type="text/css" href="./css/layout.css" />
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">

<div class="form-layout">
	<blockquote>
	<p>Hello,  hostname-deployment-6965678d58-496jz</p>	</blockquote>
</div>
```



#### 6.5.1 Cluster IP - 쿠버네티스 내부에서만 포드에 접근

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-clusterip
spec:
  ports:
    - name: web-port
      port: 8080
      targetPort: 80
  selector:
    app: webserver
  type: ClusterIP

```

* spec.selector: 이 서비스에서 어떤 라벨을 가지는 포드에 접근할 수 있게 만들 것인지를 결정한다. 위의 deployment가 app: webserver 라는 라벨을 가지고 있으므로 해당 deployment의 포드에 접근할 수 있다.
* spec.ports.port: 생성된 서비스는 쿠버네티스 내부에서만 사용할 수 있는 고유한 IP를 할당받게 되고, 이 때 사용되는 포트이다.
* spec.ports.targetPort: 접근 대상이 된 포드들이 내부적으로 사용하고 있는 포트. Deployment-hostname.yaml의 containerPort 항목과 같은 값이어야 한다.
* spec.type: 서비스의 종류. ClusterIP, NodePort, LoadBalancer 등..



```bash
kubectl get services

NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
hostname-svc-clusterip   ClusterIP   10.104.145.133   <none>        8080/TCP   9s
kubernetes               ClusterIP   10.96.0.1        <none>        443/TCP    71m

kubectl run -i --tty --rm debug --image=alicek106/ubuntu:curl --restart=Never --bash

root@debug:/# curl 10.104.145.133:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-6965678d58-496jz</p>	</blockquote>
root@debug:/# curl 10.104.145.133:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-6965678d58-tnwl4</p>	</blockquote>
root@debug:/# curl 10.104.145.133:8080 --silent | grep Hello
	<p>Hello,  hostname-deployment-6965678d58-496jz</p>	</blockquote>
root@debug:/# curl hostname-svc-clusterip:8080 --silent | grep Hello   # 내부 DNS
	<p>Hello,  hostname-deployment-6965678d58-g84dd</p>	</blockquote>

```

![image](https://user-images.githubusercontent.com/37106166/126509131-d6c4f9fa-67e9-4e70-b600-971373088025.png)




#### 6.5.2 NodePort - 포드를 외부에 노출하기

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-nodeport
spec:
  ports:
    - name: web-port
      port: 8080
      targetPort: 80
  selector:
    app: webserver
  type: NodePort		#이 부분만 다름
```

```bash
kubectl get services
NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
hostname-svc-nodeport   NodePort    10.109.6.93   <none>        8080:31124/TCP   5s
kubernetes              ClusterIP   10.96.0.1     <none>        443/TCP          89m

#클러스터의 모든 노드에 31124 포트를 통해 접근하면 동일한 서비스에 연결할 수 있다.

#멀티 서버 환경에서는
#curl {WorkerNode InternalIP}:31124
curl localhost:31124 --silent | grep Hello
	<p>Hello,  hostname-deployment-6965678d58-496jz</p>	</blockquote>
	
```

그런데 NodePort 타입에도 CLUSTER-IP가 있다. NodePort가 ClusterIP를 포함하고 있어서 클러스터 내부에서는 내부 IP와 DNS 를 사용해 접근할 수 있다. 

![image](https://user-images.githubusercontent.com/37106166/126509220-03ddfc66-9af7-433c-a1ac-b11d6cb409be.png)




#### 6.5.3 LoadBalancer - 클라우드 플랫폼의 로드 밸런서와 연동

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-lb
spec:
  ports:
    - name: web-port
      port: 80 	# Port: 80
      targetPort: 80
  selector:
    app: webserver
  type: LoadBalancer
```

```bash
➜ kubectl get svc
NAME              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
hostname-svc-lb   LoadBalancer   10.106.117.139   localhost     80:30479/TCP   5s
kubernetes        ClusterIP      10.96.0.1        <none>        443/TCP        104m

curl localhost:80 # OK

```

30479번 포트는 NodePort 타입과 마찬가지로 각 노드가 개방한 포트이다. 따라서

<노드 ip> : 30479로 접근하면 개별 포드에 접근할 수 있다. Load Balancer 로 들어온 요청이 워커 노드로 전달될 때 사용되는 포트가 30479 포트이다.

ClusterIP도 마찬가지로 사용하고 있기 때문에, 클러스터 내부에서는 서비스  IP와 dns를 통해 포드에 접근할 수도 있다.

```bash
kubectl run -i --tty --rm debug --image=alicek106/ubuntu:curl --restart=Never
root@debug:/# curl hostname-svc-lb:80
```



#### 6.5.4 externalTrafficPolicy / ExternalName

LoadBalancer 타입의 서비스를 사용하면 외부로 들어온 요청은 각 노드 중 하나로 보내지며, 그 노드에서 포드 중 하나로 전달된다.

그런데 노드로 전달된 요청이 그 노드에서 처리되지 않고 다른 노드의 포드로 전달될 수도 있다.

이 때 각 노드로 들어오는 요청을 각 노드 내의 포드에서 처리하도록 하는 속성이 externalTrafficPolicy 이다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostname-svc-lb-local
spec:
  externalTrafficPolicy: Local
  ports:
    - name: web-port
      port: 80
      targetPort: 80
  selector:
    app: webserver
  type: LoadBalancer


```

externalTrafficPolicy 를 Local 로 설정하면 포드가 생성된 노드에서만 포드로 접근할 수 있으며, 로컬 노드에 위치한 포드 중 하나로 요청이 전달된다. 즉 추가적인 네트워크 홉이 발생하지 않는다.

그러나 이 방법이 무조건 효율적인 것은 아니다. 노드 간 포드가 고르지 않게 스케쥴링 되었고, (한 노드에 포드 2개, 다른 노드에 포드 1개) 각 노드에 요청이 50%로 동일하게 분산되는 경우가 있을 수 있다.



ExternalName은 쿠버네티스를 외부 시스템과 연동할 때 유용하게 사용될 수 있다.

아래의 설정은 쿠버네티스 내부의 포드들이 externalname-svc 로 요청을 보내면 my.database.com 에 접근하게 하도록 하는 설정이다. 쿠버네티스와 별개로 존재하는 레거시 시스템에 연동해야 하는 상황에서 유용하게 사용될 수 있다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: externalname-svc
spec:
  type: ExternalName
  externalName: my.database.com
```

