# 9. 퍼시스턴트 볼륨(PV)과 퍼시스턴트 볼륨 클레임(PVC)
- 데이터베이스를 필요로 하는 stateful한 애플리케이션의 경우에는 데이터를 어떻게 관리해야 할까?
- MySQL 디플로이먼트를 통해 포드를 생성해도 데이터는 영속적이지 않다. 디플로이먼트를 삭제하면 포드도 삭제되고 데이터도 삭제되기 때문이다.
- 도커에서는 `-v` 옵션이나 `docker volume` 명령어를 통해 디렉토리를 호스트와 공유함으로써 데이터를 보존했지만 이는 클러스터 환경에서는 사용할 수 없다. 특정 노드에 데이터를 보관하면 포드가 다른 노드로 옮겨가면 데이터를 사용할 수 없게 되기 때문이다.
- 해결책은 어느 노드에서든 접근할 수 있는 퍼시스턴트 볼륨을 사용하는 것이다.

## 9.1. 로컬 볼륨: hostPath, emptyDir
### 9.1.1. 워커 노드의 로컬 디렉토리를 볼륨으로 사용: hostPath
- 호스트의 디렉토리를 포드와 공유하는 방법 (도커의 `-v`와 같다)
 - 앞서 말했듯 바람직하지 않은 방법
 - 다만 모든 노드에 배치해야 하는 특수한 포드의 경우에는 사용할 만함 (ex. CAdvisor)
 ### 9.1.2. 포드 내의 컨테이너 간 임시 데이터 공유: emptyDir
 - 포드와 생명주기를 함께하는 임시 볼륨을 사용함

## 9.2. 네트워크 볼륨
- 온프레미스든 클라우드 플랫폼에서 제공하는 볼륨이든 포드에 마운트시킬 수 있지만, 여기서는 NFS 볼륨에 대해서만 설명함
- 원래는 NFS 서버와 클라이언트가 각각 필요하지만, 클라이언트는 워커 노드의 기능을 사용하면 되므로 서버만 구축하면 됨

## 9.3. PV, PVC를 이용한 볼륨 관리
### 9.3.1. PV와 PVC를 사용하는 이유
- 위에서처럼 볼륨으로 NFS를 사용한다고 해보자. 예를 들어 MySQL을 사용한다고 하면, MySQL 디플로이먼트를 정의한 YAML 파일에 NFS 서버 정보를 기입해야 한다.
- 이걸 혼자 쓰면 상관없는데 다른 팀에게 혹은 웹상에 배포한다면 그쪽에서도 똑같이 NFS를 써야만 한다. 당연히 서버 정보도 수정해야 한다.
- iSCSI나 GlusterFS 같은 다른 네트워크 볼륨 타입을 사용하게 하고 싶으면 별도의 YAML 파일을 만들어서 배포해야 한다.
- 즉 볼륨과 애플리케이션의 정의가 밀접하게 연관되어 버린다.
- 이런 불편을 해소하기 위해 PV와 PVC를 사용한다. PV와 PVC는 볼륨을 추상화시켜준다.
- PV와 PVC의 역할
	- 인프라 관리자와 개발자가 나뉘어 있다고 가정해보자. 인프라 관리자는 쿠버네티스 클러스터와 NFS 같은 스토리지를 관리하고, 개발자는 애플리케이션을 배포한다.
	- 인프라 관리자는 네트워크 볼륨의 정보를 이용해 PV 리소스를 미리 생성한다. 여기에는 스토리지 서버의 엔드포인트 등이 포함될 수 있다.
	- 개발자는 포드를 정의하는 YAML 파일에 PVC를 명시한다.
	  ```
	  ...
	  volumes:
	    - name: nfs-volume
	      persistentVolumeClaim:
	        className: mtpvc
	  ...
	  ```
	- 쿠버네티스는 인프라 관리자가 생성했던 PV의 속성과 개발자가 요청한 PVC의 요구사항이 일치한다면 두 리소스를 바인딩한다.
	- 여기서 중요한 것은 개발자가 YAML에 볼륨의 스펙을 정의하지 않아도 된다는 점이다.
### 9.3.2. PV와 PVC 사용하기
- 리소스 이름
	- PV: persistentvolume -> pv
	- PVC: persistentvolumeclaim -> pvc
#### AWS에서 EBS를 PV로 사용하기
- AWS에서 EBS를 생성한다. 이때 주의할 점은 EBS의 가용 영역과 리전이 반드시 쿠버네티스 워커 노드와 동일한 곳에 있어야 한다.
- YAML 파일을 이용해 PV를 등록한다. (파일 참고)
- YAML 파일을 이용해 PVC를 생성한다. (파일 참고)
- 위에서 생성한 PVC를 이용하는 포드를 띄운다.
### 9.3.3. PV를 선택하기 위한 조건 명시
- accessModes
	- ReadWriteOnce: 1:1 마운트만 가능, 읽기 쓰기 가능
	- ReadOnlyMany: 1:N 마운트 가능, 읽기 전용
	- ReadWriteMany: 1:N 마운트 가능, 읽기 쓰기 가능
- storage: 볼륨의 크기
- storageClassName: storageClass의 이름
- matchLabels: 라벨 셀렉터
### 9.3.4. PV의 라이프사이클과 Reclaim Policy
- Reclaim Policy: PV와 연결된 PVC가 삭제되었을 때 PV를 어떤 상태로 만들지에 대한 설정
	- Retain: 데이터를 계속 보존한다. PV는 Released 상태가 되어 다른 PVC에 바운드될 수 없다.
	- Delete: PV를 삭제하고 가능하면 외부 스토리지도 삭제한다.
	- Recycle: 데이터만 삭제하여 PV를 다시 바인딩할 수 있게 만든다. (deprecated)
### 9.3.5. StorageClass와 Dynamic Provisioning
- 스토리지 클래스 (storageClass): 쿠버네티스 오브젝트의 한 종류로 다이나믹 프로비저닝을 위한 볼륨 생성 명세를 담고 있다.
- 다이나믹 프로비저닝: PVC의 요구사항과 일치하는 PV가 없을 경우 스토리지 클래스의 정보를 참고해 PV와 외부 스토리지를 자동으로 생성해 바인딩하는 기능.