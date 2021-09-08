# 13. 포드를 사용하는 다른 오브젝트들
- 디플로이먼트, 레플리카셋 등을 정의할 때 포드를 함께 사용했었다.
```yaml
apiVersion: apps/v1
kind: Deployment	
metadata:
  name: my-nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx
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
```
- 이와 같이 내부에서 포드를 사용할 수 있는 상위 오브젝트들이 더 있다.
## 13.1. 잡
- 잡이란, 특정 동작을 수행하고 종료하는 작업을 위한 오브젝트
- 잡의 바람직한 상태는 "포드가 실행되어 정상적으로 종료되는 것"이다.
- "Hello, World"를 출력하고 종료하는 잡의 예시
```
apiVersion: batch/v1
kind: Job
metadata:
  name: job-hello-world
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - image: busybox
        args: ["sh", "-c", "echo Hello, World && exit 0"]
        name: job-hello-world
```
- 어디에 쓸까? 배치 작업에 쓸 수 있다.
- 세부 옵션
	- spec.completions: 잡이 성공했다고 여겨지려면 몇 개의 포드가 성공해야 하는지 설정, 기본값은 1
	- spec.parallelism: 동시의 생성될 포드의 개수 설정, 기본값은 1
- 크론잡: 리눅스 크론 비슷한 걸 사용할 수 있다.
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob-example
spec:
  schedule: "*/1 * * * *"        # Job의 실행 주기
  jobTemplate:                 # 실행될 Job의 설정 내용 (spec)
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: cronjob-example
            image: busybox
            args: ["sh", "-c", "date"]
```
## 13.2. 데몬셋
- 데몬셋이란, 모든 노드에 동일한 포드를 하나씩 생성하는 오브젝트.
- 로깅, 모니터링, 네트워킹 등에 유용하게 사용할 수 있다.
- 모든 노드에 동일한 포드를 하나씩 배치하는 데몬셋의 예시
```
apiVersion: apps/v1
kind: DaemonSet                           # [1]
metadata:
  name: daemonset-example
spec:
  selector:
    matchLabels:
      name: my-daemonset-example         # [2.1] 포드를 생성하기 위한 셀렉터 설정
  template:
    metadata:                              # [2.2] 포드 라벨 설정
      labels:
        name: my-daemonset-example
    spec:
      tolerations:                           # [3] 마스터 노드에도 포드를 생성
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: daemonset-example
        image: busybox                      # 테스트를 위해 busybox 이미지 사용
        args: ["tail", "-f", "/dev/null"]
        resources:                           # [4] 자원 할당량을 제한
          limits:
            cpu: 100m
            memory: 200Mi
```
## 13.2. 스테이트풀셋
- 스테이트풀셋이란, 데이터베이스와 같이 상태를 갖는 포드를 관리하기 위한 오브젝트.
- 가축과 반려동물의 비유
	- 가축: 각 가축엔 이름이 없으며, 언제든 대체될 수 있음 ㅜㅜ -> 상태가 없는 포드
	- 반려동물: 각 반려동물에 이름을 지어주어 다른 개체와 구분해줌, 대체 불가능 -> 상태가 있는 포드
	- 3개의 포드로 구성된 스테이트풀셋의 예시
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-example
spec:
  serviceName: statefulset-service
  selector:
    matchLabels:
      name: statefulset-example
  replicas: 3
  template:
    metadata:
      labels:
        name: statefulset-example
    spec:
      containers:
      - name: statefulset-example
        image: alicek106/rr-test:echo-hostname
        ports:
        - containerPort: 80
          name: web
---
apiVersion: v1
kind: Service
metadata:
  name: statefulset-service
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None	# 헤드리스 서비스 설정
  selector:
    name: statefulset-example
 ```
 - 스테이트풀셋의 포드에 접근할 수 있는 서비스가 필요하다.
	 - 이 때 서비스를 통해 접근하는 포드는 고유하므로, 서비스는 클라이언트가 개별 포드에 접근할 수 있도록 해줘야 한다. 이를 헤드리스 서비스라 한다.
 - 헤드리스 서비스의 이름은 SRV 레코드로 쓰인다. 따라서 헤드리스 서비스의 이름을 통해 각 포드에 접근할 수 있는 IP 주소를 얻을 수 있다.
	 - SRV 레코드: DNS에서 각 서비스를 호스팅하는 머신을 식별하는 데 쓰이는 데이터 스펙
	 - `nslookup`: 도메인 네임을 질의하는 리눅스 명령어: SRV 레코드를 인자로 넣을 수 있다.
 - 퍼시스턴트 볼륨
	 - 실제 스테이트풀셋을 사용할 때는 상태가 존재하는 애플리케이션, 즉 퍼시스턴트 볼륨을 포드에 마운트해 사용한다.
	 - 쿠버네티스는 스테이트풀셋을 만들 때 포드마다 PVC를 자동으로 생성하여 다이나믹 프로비저닝을 지원한다.
	 - 스테이트풀셋에서 볼륨을 정의하는 예시
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-volume
spec:
  serviceName: statefulset-volume-service
  selector:
    matchLabels:
      name: statefulset-volume-example
  replicas: 3
  template:
    metadata:
      labels:
        name: statefulset-volume-example
    spec:
      containers:
      - name: statefulset-volume-example
        image: alicek106/rr-test:echo-hostname
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: webserver-files
          mountPath: /var/www/html/
  volumeClaimTemplates:
  - metadata:
      name: webserver-files
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: generic
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
kind: Service
metadata:
  name: statefulset-volume-service
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    name: statefulset-volume-example
```
- spec.volumeClaimTemplates에 정의한 대로 각 포드마다 PVC와 PV가 동적으로 생성된다.
- 스테이트풀셋을 삭제해도 spec.volumeClaimTemplates을 통해 생성된 볼륨은 삭제되지 않는다.