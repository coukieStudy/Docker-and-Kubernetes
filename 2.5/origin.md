## 2.5 도커 데몬

#### 2.5.1 도커의 구조

도커 클라이언트 - 도커 서버

**도커 서버 (dockerd 프로세스)**

- 컨테이너 생성, 실행, 이미지 관리
- 도커 엔진은 API 입력으로 기능 수행
- 도커 프로세스가 실행되어 서버로서 입력 받을 준비가 된 상태를 도커 데몬

**도커 클라이언트**

- API 호출 할 수 있게 CLI 제공

- #docker 명령어는 도커 클라이언트 사용 입력된 명령어를 로컬 도커 데몬에게 API로서 전달

- 도커 클라이언트는 /var/run/docker.sock에 위치한 유닉스 소켓으로 도커 데몬 API 호출

순서

	1. 사용자가 docker version 같은 명령어 입력
	2. /usr/bin/docker는 /var/run/docker.sock 유닉스 소켓 사용해 도케 데몬에게 명령어 전달
	3. 도커 데몬은 이 명령어 파싱, 해당하는 작업 수행
	4. 수행결과 클라이언트에 리턴, 결과 출력

아무 설정 없을 때 일반적인 도커 데몬 제어 순서

![image](https://user-images.githubusercontent.com/66865817/125769805-805cf7aa-23ca-4099-9447-74e65fceefc8.png)



#### 2.5.2 도커 데몬 실행

도커 데몬 시작 / 정지 (우분투는 설치 후 자동 실행이 default)

#service docker start

#service docker stop

레드헷 계열은 자동 실행 등록 안됨 -> 자동 실행 등록 명령어 (당연히 맥에서 안됨)

#systemctl enable docker



도커 서비스를 정지하고 도커 데몬을 바로 사용 가능

#service docker stop

#dockerd *(mac에서 안됨)*

직접 실행하면 foreground 상태로 실행되서 service, systemctl 명령어를 통해 리눅스 서비스로 관리하는게 좋다



#### 2.5.3 도커 데몬 설정

부록 A 참고 (docker client에서 docker engine에 json 형식으로 설정, mac/window 기준)



#### 2.5.3.1 도커 데몬 제어: -H

#Dockerd

#Dockerd -H unix://var/run/docker.sock

위 두 명령어는 같은 명령어. 도커 데몬을 실행하면 유닉스 소켓 /var/run/docker.sock을 기본으로 사용하기 때문

-H에 IP 주소와 포트 번호를 입력하면 원격 API인 Docker Remote API로 도커를 제어할 수 있습니다. Remote API는 도커 클라이언트와는 다르게 로컬에 있는 도커 데몬이 아니더라도 제어할 수 있으며 Restful API 형식을 띠고 있으므로 HTTP 요청으로 도커를 제어할 수 있습니다.

다음과 같이 도커 데몬을 실행하면 호스트에 존재하는 모든 네트워크 인터페이스의 IP주소와 2375번 포트에 바인딩해 입력을 받습니다.

#Dockerd -H tcp://0.0.0.0:2375

위와 같이 명령어를 입력하면 소켓은 비활성화되므로 도커 클라이언트를 사용할 수 없게 되며, docker로 시작하는 명령어를 사용할 수 없습니다. 따라서 일반적으로 동시에 설정

#Dockerd -H unix://var/run/docker.sock -H tcp://0.0.0.0:2375

도커 클라이언트가 도커 데몬에게 명령어를 api를 통해서 전달할 때도 같은 api를 사용하므로, 모든 명령어를 api로 사용가능.

#Dockerd -H tcp://192.168.99.100:2375

#curl 192.168.99.100:2375/version --silent | python -m json.tool

```json
{
  "ApiVersion": "1.39",
  "Arch": "amd64",
  "BuildTime": "2019-02-10T03:42:13.000000000+00:00",
  "Components" :[
    {
      "Details": {
        "ApiVersion": "1.39",
        "Arch": "amd64",
...
```

= #docker version



#### 2.5.3.2 도커 데몬에 보안 적용: --tlsverify

기본적으로 보안 적용 안되어 있음 : IP와 포트만 알면 도커 제어 가능

TLS 보안 적용, 도커 클라이언트와 Remote API 클라이언트가 인증되지 않으면 도커 데몬 제어할 수 없도록 설정

보안 적용할 때 사용되는 파일 ca.pem, server-cert.pem, server-key.pem, vert.pem, key.pem 총 5개

양쪽에 키 설정 및 인증 사용 방법



#### 2.5.3.3 도커 스토리지 드라이버 변경: --storage-driver

도커 컨테이너와 이미지 관리하는 스토리지 백앤드

#docker info | grep "Storage Driver"

overlay2가 default.

#dockerd --storage-driver=devicemapper

와 같은 명령어로 변경 가능



##### 스토리지 드라이버의 원리

컨테이너 내부에서 읽기와 새로운 파일 쓰기, 기존의 파일 쓰기 작업이 일어날 때는 Copy-on-Write(CoW) 또는 Redirect-on-Write(RoW) 개념을 사용

따라서 도커 스토리지 드라이버는 CoW, RoW를 지원해야 함

https://tech.gluesys.com/blog/2020/12/16/storage_7_intro.html

AUFS, Devicemapper, OverlayFS, Btrfs, ZFS 드라이버 사용법

#### 2.5.3.4 컨테이너 저장 공간 설정

드라이버 별로 약간씩 다름

devicemapper, overlay2에 대해서 설명

DOCKER_OPTS=" ... --storage-driver=devicemapper --storage-opt dm.basesize=20G ..."



#### 2.5.4 도커 데몬 모니터링

#### 2.5.4.1 도커 데몬 디버그 모드

#dockerd -D

Remote API의 입출력을 비롯해서 도커 클라이언트에서 오가는 모든 명령어를 로그로 출력

#### 2.5.4.2 events, stats, system df 명령어

<img width="567" alt="image" src="https://user-images.githubusercontent.com/66865817/125769636-0f1ffab1-75e8-44b3-af82-40aeb172fcc4.png">


<img width="758" alt="image" src="https://user-images.githubusercontent.com/66865817/125769851-6875ad47-c48e-48dd-82bc-e84ba6ceffdc.png">

<img width="421" alt="image" src="https://user-images.githubusercontent.com/66865817/125769872-967a6c15-7300-4ea6-9670-21cb9a56c4ab.png">

#### 2.5.4.3 CAdvisor

구글이 만든 컨테이너 모니터링 도구

단일 도커 호스트만을 모니터링 할 수 있는 한계

여러개의 호스트로 도커를 사용하고 있으면, 보통 쿠버네티스나 스웜 모드 등으로 오케스트레이션 툴에서

프로메테우스나 InfluxDB 등에 데이터 수집하는게 일반적

#### 2.5.5 Remote API 라이브러리를 이용한 도커 사용

#### 2.5.5.1 자바 라이브러리

Dependency 수정

```java
package Main;

import com.spotify.docker.client.*;
import com.spotify.docker.client.exceptions.*;

public class Main {
  private static final String DOCKER_IP = "http://192.168.99.100:2375";
  
  public static void main(String[] args) throws DockerException, InterruptedException {
    DockerClient dc = new DefaultDockerClient(DOCKER_IP);
    
    System.out.println(dc.info());
    
    dc.close();
  }
}
```

출력 결과는 curl로 HTTP 요청을 보냈던 것과 같이 JSON 형태로 결과를 반환

```
Info{architecture=x86_64, clusterStore=etcd://192.168.99.100:12379, cgroupDriver=cgroufs, ...}
```



TLS 사용법

컨테이너의 80번 포트 - 호스트의 80번 포트 연결해 외부에 노출하는 nginx 컨테이너 생성법 (docker run -d -p 0.0.0.0:10080:80 --name mynginx nginx)

#### 2.5.5.2 파이썬 라이브러리

python 2.7(...)로 된 예제

