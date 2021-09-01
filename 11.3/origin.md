## 11.3 쿠버네티스 애플리케이션 상태와 배포



#### 11.3.1 디플로이먼트를 통한 롤링 업데이트

테스트 또는 개발환경이 아닌 이상 pod 를 직접 생성하거나 종료할 일은 거의 없다. 디플로이먼트를 통해 replica set 을 관리하고, 이 때 --record 옵션을 통해 변경사항을 기록할 수 있다. 

##### Recreate 방식 

  기존 버젼의 포드를 모두 삭제한 뒤 새로운 버젼의 포드를 생성한다. 그러나 포드를 삭제하고 새롭게 생성하는 사이에는 사용자의 요청을 처리할 수 없기 때문에, 어플리케이션의 중단이 허용되지 않을 때에는 해당 방식이 적절하지 않을 수 있다.

```BASH
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-recreate
spec:
  replicas: 3
  strategy:
    type: Recreate  #  Recreate 방식의 deployment
...
```



##### Rolling 업데이트

  포드를 조금씩 삭제하고 생성하는 업데이트 기능. 업데이트 도중에도 사용자의 요청이 처리될 수 있다는 장점이 있다.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-rolling-update
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2    			# 업데이트 도중 전체의 포드의 추가적인 허용 개수(비중)
      maxUnavailable: 2			# 업데이트 도중 사용 불가능한 상태가 되는 포드의 최대 수(비중)
```

* Blue Green 배포 

  기존 버젼의 포드를 놔둔 상태에서 새로운 버젼의 포드를 미리 생성해둔 뒤 서비스의 라우팅만 변경하는 배포 방식을 의미한다. 쿠버네티스에서 공식적으로 지원하지는 않지만, 개념적으로 이용할 수는 있다. 





#### 11.3.2 포드의 생애 주기

포드의 상태

* Running: 포드에 포함된 컨테이너들이 모두 생성돼 포드가 정상적으로 실행된 상태. 일반적으로 바람직한 상태
* Completed: 포드가 정상적으로 실행되어 종료되었음을 의미한다. 포드 컨테이너의 init 프로세스가 종료 코드로서 0을 반환한 상태
* Error: 포드가 정상적으로 실행되지 않은 상태로 종료되었음을 의미한다. init 프로세스가 0이 아닌 종료 코드를 반환한 상태
* Terminating: 포드가 삭제되기 위해 삭제 상태에 머물러 있는 경우.



```bash
apiVersion: v1
kind: Pod
metadata:
  name: completed-pod-restart-never
spec:
  restartPolicy: Never   	# Completed 상태 이후로 포드가 다시 시작되지 않음.  Always / Never / OnFailure 중에서 선택할 수 있다.
  containers:
  - name: completed-pod-restart-never
    image: busybox
    command: ["sh"]
    args: ["-c", "sleep 10 && exit 0"]
```

위를 실행하면 정상적으로 종료되어 포드가 Completed 상태에 있게 된다. 마지막 줄의 exit 0 를 exit 1 로 수정하면 Error 상태가 된다.

만약 restartPolicy를 Always로 했다면, 자동적으로 포드가 다시 실행될 것이다. 이 때 Completed 와 Running 사이의 간격을 CrashLoopBackOff 상태라고 하고, 실패를 반복할수록 이 상태에 있는 시간이 길어진다.

포드를 생성했다고 해서 무조건 Running 상태가 되는 것은 아니기 때문에, 이를 확인하기 위한 기능들이 존재한다.

1. Init Container

   init Container는 포드의 컨테이너 내부에서 어플리케이션이 실행되기 전에 초기화를 수행하는 컨테이너를 의미한다. 이 컨테이너들은 포드의 어플리케이션 컨테이너와 거의 동일하게 사용할 수 있지만, 먼저 실행된다는 점이 다르다. 아래의 예시처럼 다른 서비스 또는 디플로이먼트가 생성될 때까지 init 컨테이너에서 대기하는 예시이다. 

   ```bash
   apiVersion: v1
   kind: Pod
   metadata:
     name: init-container-usecase
   spec:
     containers:
     - name: nginx
       image: nginx
     initContainers:   			# 여기에 초기화 컨테이너들을 정의한다.
     - name: wait-other-service
       image: busybox
       command: ['sh', '-c', 'until nslookup myservice; do echo waiting..; sleep 1; done;']
   ```

2.  postStart

   포드의 컨테이너가 실행되거나 삭제될 때, 특정 작업이 수행되도록 라이프사이클 훅을 지정할 수 있다. 시작될 때 실행되는 작업을 postStart 라고 하며, 컨테이너가 시작한 직후 특정 주소로 HTTP 요청을 보내는 방식과, 컨테이너 내부에서 특정 명령어를 실행하는 두 가지 방식이 있다.

   ```bash
   apiVersion: v1
   kind: Pod
   metadata:
     name: poststart-hook
   spec:
     containers:
     - name: poststart-hook
       image: nginx
       lifecycle:
         postStart:
           exec:
             command: ["sh", "-c", "touch /myfile"]
   ```

   컨테이너가 실행됨과 동시에 myfile 이라는 파일이 생성된다. 이 명령이 제대로 실행되지 않으면 컨테이너는 Running 상태로 전환되지 않고, restartPolicy에 의해 컨테이너가 재시작된다.

3. 애플리케이션의 상태 검사 - livenessProbe, readinessProbe

   livenessProbe: 컨테이너 내부의 어플리케이션이 살아있는지 검사하고, 실패하면 restartPolicy에 의해 재시작된다.
   readinessProbe: 컨테이너 내부의 어플리케이션이 사용자 요청을 처리할 준비가 되었는지 검사하고, 실패하면 컨테이너는 서비스의 라우팅 대상에서 제외된다.

   ```bash
   apiVersion: v1
   kind: Pod
   metadata:
     name: livenessprobe-pod
   spec:
     containers:
     - name: livenessprobe-pod
       image: nginx
       livenessProbe:  # 이 컨테이너에 대해 livenessProbe를 정의합니다.
         httpGet:      # HTTP 요청을 통해 애플리케이션의 상태를 검사합니다.
           port: 80    # <포드의 IP>:80/ 경로를 통해 헬스 체크 요청을 보냅니다. 
           path: /
   
   
   $ kubectl exec livenessprobe-pod --rm /usr/share/nginx/html/index.html
   # 임의로 index.html 삭제 -> 포드가 재시작 한다(restartPolicy Always)
   ```

   ```bash
     
   apiVersion: v1
   kind: Pod
   metadata:
     name: readinessprobe-pod
     labels:
       my-readinessprobe: test
   spec:
     containers:
     - name: readinessprobe-pod
       image: nginx       # Nginx 서버 컨테이너를 생성합니다.
       readinessProbe:    # <포드의 IP>:80/ 로 상태 검사 요청을 전송합니다.
         httpGet:
           port: 80
           path: /
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: readinessprobe-svc
   spec:
     ports:
       - name: nginx
         port: 80
         targetPort: 80
     selector:
       my-readinessprobe: test
     type: ClusterIP
     
     
   ```

   
