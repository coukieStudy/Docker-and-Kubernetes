# 6.1 쿠버네티스를 시작하기 전에

- `모든 리소스는 오브젝트 형태로 관리됩니다`
  - 오브젝트?
    - Kubectl api-resources
  - 모든 리소스를 다 외워버리겠다 X -> 자주 쓰는 것은 외우고 필요할 떄 document를 참고해서 찾아본다
  - kubectk explain {{pod}}
  - YAML(Yet Another Markup Language -> YAML Ain't Markup Language)를 사용한다
- `쿠버네티스는 여러 개의 컴포넌트로 구성돼 있습니다`
  - 컴포넌트?
    - API 서버(kube-apiserver)
    - 컨트롤러 매니저(kube-controller-manager)
    - 스케줄러(kube-scheduler)
    - DNS서버(coreDNS)
    - 프락시(kube-proxy)
    - 네트워크 플러그인(calico, flannel 등)
    - 도커 데몬(OCI 표준 구현한 다른 컨테이너도 사용 가능 )
    - 기타 등등
    - ![image-20210721110621273](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)
  - 컴포넌트는 기본적으로 도커 컨테이너로서 실행
    - docker ps on master node
  - 노드
    - 마스터
      - 클러스터를 관리
    - 워커
      - 애플리케이션 컨테이너 생성
    - kubelet
      - 클러스터 구성을 위해 모든 노드에 존재하는 에이전트
      - 컨테이너 생성, 삭제, 통신 등을 담당

# 6.2 포드(Pod): 컨테이너를 다루는 기본 단위

- 기본이 되는 컴포넌트
  - 포드
  - 레플리카셋
  - 서비스
  - 디플로이먼트

## 포드 사용하기

- 포드란

  - 쿠버네티스에서 컨테이너 애플리케이션의 기본 단위
  - 1개 이상의 컨테이너로 구성

- 예시 : Nginx 컨테이너로 구성된 포드 (책에서는 yaml을 사용하지만, 다큐멘트에서는 json도 가능하다고 함)

  - ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    	name: my-nginx-pod
    spec:
    	containers:
    	- name: my-nginx-container
    		image: nginx:latest
    		ports:
    		- containerPort: 80
    			protocol: TCP
    ```

  - apiVersion

    - 오브젝트의 api 버전

  - Kind

    - 리소스의 종류

    - `kubectl api-resources`

    - ```
      NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
      bindings                                                                      true         Binding
      componentstatuses                 cs                                          false        ComponentStatus
      configmaps                        cm                                          true         ConfigMap
      endpoints                         ep                                          true         Endpoints
      events                            ev                                          true         Event
      limitranges                       limits                                      true         LimitRange
      namespaces                        ns                                          false        Namespace
      nodes                             no                                          false        Node
      persistentvolumeclaims            pvc                                         true         PersistentVolumeClaim
      persistentvolumes                 pv                                          false        PersistentVolume
      pods                              po                                          true         Pod
      podtemplates                                                                  true         PodTemplate
      replicationcontrollers            rc                                          true         ReplicationController
      resourcequotas                    quota                                       true         ResourceQuota
      secrets                                                                       true         Secret
      serviceaccounts                   sa                                          true         ServiceAccount
      services                          svc                                         true         Service
      mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
      validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
      customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
      apiservices                                    apiregistration.k8s.io         false        APIService
      controllerrevisions                            apps                           true         ControllerRevision
      daemonsets                        ds           apps                           true         DaemonSet
      deployments                       deploy       apps                           true         Deployment
      replicasets                       rs           apps                           true         ReplicaSet
      statefulsets                      sts          apps                           true         StatefulSet
      tokenreviews                                   authentication.k8s.io          false        TokenReview
      localsubjectaccessreviews                      authorization.k8s.io           true         LocalSubjectAccessReview
      selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
      selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
      subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
      horizontalpodautoscalers          hpa          autoscaling                    true         HorizontalPodAutoscaler
      cronjobs                          cj           batch                          true         CronJob
      jobs                                           batch                          true         Job
      certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
      leases                                         coordination.k8s.io            true         Lease
      endpointslices                                 discovery.k8s.io               true         EndpointSlice
      events                            ev           events.k8s.io                  true         Event
      ingresses                         ing          extensions                     true         Ingress
      flowschemas                                    flowcontrol.apiserver.k8s.io   false        FlowSchema
      prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io   false        PriorityLevelConfiguration
      ingressclasses                                 networking.k8s.io              false        IngressClass
      ingresses                         ing          networking.k8s.io              true         Ingress
      networkpolicies                   netpol       networking.k8s.io              true         NetworkPolicy
      runtimeclasses                                 node.k8s.io                    false        RuntimeClass
      poddisruptionbudgets              pdb          policy                         true         PodDisruptionBudget
      podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
      clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
      clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
      rolebindings                                   rbac.authorization.k8s.io      true         RoleBinding
      roles                                          rbac.authorization.k8s.io      true         Role
      priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
      csidrivers                                     storage.k8s.io                 false        CSIDriver
      csinodes                                       storage.k8s.io                 false        CSINode
      csistoragecapacities                           storage.k8s.io                 true         CSIStorageCapacity
      storageclasses                    sc           storage.k8s.io                 false        StorageClass
      volumeattachments                              storage.k8s.io                 false        VolumeAttachment
      
      ```

  - metadata

    - 리소스에 대한 부가 정보

  - spec

    - 리소스를 생성하기 위한 정보

  - continerPort

    - 클러스터 내부에서는 접근 가능하지만, 클러스터 외부에서는 접근하지 못한다 (private ip 할당)
    - 할당한 private ip로 클러스터 내부에서 접근 가능하지만, 그건 하드코딩
    - 실제로 `Service` 라는 오브젝트를 사용해서 pod에 접근한다

- `kubectl apply -f`

  - f option이란?

  - ```bash
    kubectl apply -f ./my-manifest.yaml            # 리소스(들) 생성
    kubectl apply -f ./my1.yaml -f ./my2.yaml      # 여러 파일로 부터 생성
    kubectl apply -f ./dir                         # dir 내 모든 매니페스트 파일에서 리소스(들) 생성
    kubectl apply -f https://git.io/vPieo          # url로부터 리소스(들) 생성
    ```

- `kubectl get {{pods}}`

  - 오브젝트 목록 확인하기

  - ```shell
    NAME           READY   STATUS    RESTARTS   AGE
    my-nginx-pod   1/1     Running   0          68s
    ```

- `kubectl describe {{pods}} {{my-nginx-pod}}`

  - 자세한 정보 얻기

  - ```shell
    Name:         my-nginx-pod
    Namespace:    default
    Priority:     0
    Node:         minikube/192.168.64.2
    Start Time:   Wed, 21 Jul 2021 20:01:25 +0900
    Labels:       <none>
    Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                    {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"my-nginx-pod","namespace":"default"},"spec":{"containers":[{"image":"...
    Status:       Running
    IP:           172.17.0.3
    IPs:
      IP:  172.17.0.3
    Containers:
      my-nginx-container:
        Container ID:   docker://1e3cd8220e56e818deef95354ecf1007650cef5ee4f61ada9699ba6d19997a7e
        Image:          nginx:latest
        Image ID:       docker-pullable://nginx@sha256:353c20f74d9b6aee359f30e8e4f69c3d7eaea2f610681c4a95849a2fd7c497f9
        Port:           80/TCP
        Host Port:      0/TCP
        State:          Running
          Started:      Wed, 21 Jul 2021 20:01:51 +0900
        Ready:          True
        Restart Count:  0
        Environment:    <none>
        Mounts:
          /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-r9chh (ro)
    Conditions:
      Type              Status
      Initialized       True 
      Ready             True 
      ContainersReady   True 
      PodScheduled      True 
    Volumes:
      kube-api-access-r9chh:
        Type:                    Projected (a volume that contains injected data from multiple sources)
        TokenExpirationSeconds:  3607
        ConfigMapName:           kube-root-ca.crt
        ConfigMapOptional:       <nil>
        DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute for 300s
                                 node.kubernetes.io/unreachable:NoExecute for 300s
    Events:
      Type    Reason     Age   From               Message
      ----    ------     ----  ----               -------
      Normal  Scheduled  119s  default-scheduler  Successfully assigned default/my-nginx-pod to minikube
      Normal  Pulling    118s  kubelet, minikube  Pulling image "nginx:latest"
      Normal  Pulled     94s   kubelet, minikube  Successfully pulled image "nginx:latest" in 24.435337742s
      Normal  Created    93s   kubelet, minikube  Created container my-nginx-container
      Normal  Started    93s   kubelet, minikube  Started container my-nginx-container
    
    ```

- `kubectl exec {{-it}} {{my-nginx-pod}} {{bash}}`

  - 포드의 컨테이너에 명령을 전달
  - 포드의 컨테이너가 여러 개 있다면?

- `kubectl logs {{my-nginx-pod}}`

  - ```sh
    /docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
    /docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
    /docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
    10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
    10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
    /docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
    /docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
    /docker-entrypoint.sh: Configuration complete; ready for start up
    2021/07/21 11:01:51 [notice] 1#1: using the "epoll" event method
    2021/07/21 11:01:51 [notice] 1#1: nginx/1.21.1
    2021/07/21 11:01:51 [notice] 1#1: built by gcc 8.3.0 (Debian 8.3.0-6) 
    2021/07/21 11:01:51 [notice] 1#1: OS: Linux 4.19.182
    2021/07/21 11:01:51 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
    2021/07/21 11:01:51 [notice] 1#1: start worker processes
    2021/07/21 11:01:51 [notice] 1#1: start worker process 33
    2021/07/21 11:01:51 [notice] 1#1: start worker process 34
    ```

- `kubectl delete -f {{nginx-pod.yaml}}`

  - 만약 저 파일이 바뀌었다면?
    - `Note that the delete command does NOT do resource version checks, so if someone submits an update to a resource right when you submit a delete, their update will be lost along with the rest of the resource.`
    - 무시하고 삭제
  - 강제 종료하는 것일까?
    - `Some resources, such as pods, support graceful deletion. These resources define a default period before they are forcibly terminated`

## 포드 vs 도커 컨테이너

- 포드는 `여러 리눅스 네임스페이스(namespace)를 공유하는 여러 컨테이너들을 추상화된 집합으로 사용하기 위해서`

  - 리눅스 네임스페이스?

    - https://www.44bits.io/ko/keyword/linux-namespace
    - process가 네임스페이스를 공유하면 시스템의 리소스를 공유한다
    - 기본적으로 리눅스를 사용한다면 pid 1번의 process와 다른 process 모두 같은 네임스페이스를 공유한다
    - 쿠버네티스의 네임스페이스와는 다른 것
      - https://stackoverflow.com/questions/61628540/difference-between-kubernetes-namespace-and-linux-namespaces

  - 즉 한 포드에 여러 컨테이너가 있을 시 네임스페이스를 공유 -> 리소스를 공유

  - 예시2

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: my-nginx-pod
    spec:
      containers:
      - name: my-nginx-container
        image: nginx:latest
        ports:
        - containerPort: 80
          protocol: TCP
      - name: ubuntu-sidecar-container
        image: alicek106/rr-test:curl
        command: ["tail"]
        args: ["-f", "/dev/null"]
    ```

    

  - ex) 네트워크 네임스페이스를 공유하기 때문에 `localhost`로 다른 컨테이너에 접근 가능

    - 컨테이너가 여러 개 일때는

      - `kubectl exec -it my-nginx-pod -c ubuntu-sidecar-container bash`

    - [ ] 네임스페이스 확인해보기

      `ls -l /proc/$$/ns | awk '{print $1, $9, $10, $11}'`

    - my-nginx-continer

      ```sh
      lrwxrwxrwx. cgroup -> cgroup:[4026531835]
      lrwxrwxrwx. ipc -> ipc:[4026532444]
      lrwxrwxrwx. mnt -> mnt:[4026532546]
      lrwxrwxrwx. net -> net:[4026532447]
      lrwxrwxrwx. pid -> pid:[4026532548]
      lrwxrwxrwx. pid_for_children -> pid:[4026532548]
      lrwxrwxrwx. user -> user:[4026531837]
      lrwxrwxrwx. uts -> uts:[4026532547]
      ```

    - Ubuntu-sidecar-container

      ```sh
      lrwxrwxrwx. cgroup -> cgroup:[4026531835]lrwxrwxrwx. ipc -> ipc:[4026532444]lrwxrwxrwx. mnt -> mnt:[4026532549]lrwxrwxrwx. net -> net:[4026532447]lrwxrwxrwx. pid -> pid:[4026532551]lrwxrwxrwx. pid_for_children -> pid:[4026532551]lrwxrwxrwx. user -> user:[4026531837]lrwxrwxrwx. uts -> uts:[4026532550]
      ```

    - Network는 같은데, 다른 것들도 있다...왜?

    - https://speakerdeck.com/devinjeon/containerbuteo-dasi-salpyeoboneun-kubernetes-pod-dongjag-weonri?slide=65

  - 일단은 네트워크 인터페이스를 공유한다고 생각하자

- 어떻게 포드를 구성해야 할까?

  - 하나의 완전한 애플리케이션
  - 개별적으로 스케일링 가능하게
  - 웬만하면 pod과 컨테이너는 1대1
  - Main process와 sidecar 정도
    - ex) fluent를 달아서 로그 수집
    - istio
      - https://techcafe.tistory.com/133
  - https://act-coe.github.io/kubernetes-in-action-03/

## 레플리카셋(Replica Set): 일정 개수의 포드를 유지하는 컨트롤러

### 레플리카 셋을 사용하는 이유

- `정해진 수의 동일한 포드가 항상 실행되도록 관리`
- `노드 장애 등의 이유로 포드를 사용할 수 없다면 다른 노드에서 포드를 다시 생성합니다`
- -> 따라서 직접 포드를 관리할 필요가 없어 필수적이다

### 레플리카 셋 사용하기

- 예시

  - ```yaml
    apiVersion: apps/v1kind: ReplicaSetmetadata:	name: replicaset-nginxspec: 	replicas: 3	selector:		matchLabels:			app: my-nginx-pods-label  template:  	metadata:  		name: my-nginx-pod  		labels:  			app: my-nginx-pods-label    spec:    	containers:    		- name: nginx    			image: nginx:latest    			ports:    				- containerPort: 80
    ```

  - metadata.name

    - 포드 뿐만 아니라, 모든 쿠버네티스 오브젝트에 이름을 붙일 수 있다

  - spec.replicas

    - 포드를 몇 개 유지할지

  - spec.template

    - 포드를 생성할 때 사용할 템플릿

  - 다른 api 구성요소

    - https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#replicaset-v1-apps

- `kubectl get rs`

  ```sh
  NAME               DESIRED   CURRENT   READY   AGEreplicaset-nginx   3         3         2       9s
  ```

- `kubectl describe rs replicaset-nginx`

  ```sh
  Name:         replicaset-nginxNamespace:    defaultSelector:     app=my-nginx-pods-labelLabels:       <none>Annotations:  kubectl.kubernetes.io/last-applied-configuration:                {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{},"name":"replicaset-nginx","namespace":"default"},"spec":{"replica...Replicas:     3 current / 3 desiredPods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 FailedPod Template:  Labels:  app=my-nginx-pods-label  Containers:   nginx:    Image:        nginx:latest    Port:         80/TCP    Host Port:    0/TCP    Environment:  <none>    Mounts:       <none>  Volumes:        <none>Events:  Type    Reason            Age   From                   Message  ----    ------            ----  ----                   -------  Normal  SuccessfulCreate  66s   replicaset-controller  Created pod: replicaset-nginx-5tr9f  Normal  SuccessfulCreate  66s   replicaset-controller  Created pod: replicaset-nginx-jpkg7  Normal  SuccessfulCreate  66s   replicaset-controller  Created pod: replicaset-nginx-6dhrx
  ```

### 레플리카 셋의 동작 원리

- 어떻게 RS, PO가 연결되어 있을까?
  - Label Selector
    - 쿠버네티스 리소스를 분류할 떄 많이 사용
    - 서로 다른 오브젝트가 서로를 찾아야 할 때 사용
  - spec.selector.matchLabel
    - app: my-nginx-pods-label을 가지는 포드의 개수를 살핀다
  - 따라서 **따로 생성해도** 상관없다 -> Declarative
    - ex) 기존에 라벨을 가지고 있는 포드가 1개 있었다면, replicaset은 나머지 2개만 생성한다
- 결국 replicaSet은 declared 된 포드의 개수를 유지하는 목적 -> 더 많으면 삭제도 가능

### 다른 오브젝트와 비교

- vs 레플리케이션 컨트롤러
  - deprecated된 것
- vs Deployment
  -  **Warning:** In many cases it is recommended to create a [Deployment](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#deployment-v1-apps) instead of ReplicaSet.
