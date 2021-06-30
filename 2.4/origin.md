# 2.4 DockerFile

## 2.4.1 이미지를 생성하는 방법

Dockerfile을 통하면, 여러 수작업(직접 컨테이너 생성 or 이미지로 커밋) 없이 이미지를 생성할 수 있다.

## 2.4.2 DockerFile 작성

DockerFile - 아파치 웹 서버를 설치한 뒤, 로컬에 있는 test.html 파일을 웹 서버로 접근할 수 있는 컨테이너의 디렉터리인 /var/www/html에 복사합니다.

```
FROM ubuntu:14.04 //베이스될 이미지
MAINTAINER alicek106
LABEL "purpose"="practice"
RUN apt-get update 
RUN apt-get install apache2 -y
ADD test.html /var/www/html
WORKDIR /var/www/html
RUN ["/bin/bash", "-c", "echo hello >> test2.html"] //bash shell로 echo 명령어 수행해서 test2.html을 만든다
EXPOSE 80 
CMD apachectl -DFOREGROUND
```

→ Docker 엔진은 Dockerfile을 읽어 들일 때 기본적으로 현재 Directory의 Dockerfile이라는 이름을 가진 파일을 선택함.

**FROM**: 생성할 이미지의 베이스가 될 이미지

**MAINTAINER**: 이미지를 생성한 개발자의 정보

**LABEL**: 이미지에 메타데이터 (docker inspect 명령어로 이미지의 정보를 구해서 확인 가능)

→ 생성될 이미지의 라벨을 "purpose=practice"로 설정

**RUN**: 이미지를 만들기 위해 컨테이너 내부에서 명령어 실행

→ apt-get update, apt-get install apache2 -y 이기에 Apache 웹 서버 설치

→ -y는 Y/N를 선택하는 것임. 없을 경우 오류로 빌드 종료

**ADD**: 파일을 이미지에 추가합니다.

→ Dockerfile이 위치한 test.html에서 /var/www/html 디렉토리로 추가

→ ["추가할 파일 이름" .... "컨테이너 추가될 위치"]와 같이 배열로 사용 가능 (추가할 파일 이름은 여러 개 가능)

**EXPOSE**: Dockerfile의 빌드로 생성된 이미지에서 노출할 포트를 설정

→ EXPOSE는 run 명령어의 -P 플래그와 함께 사용된다. 

**CMD**: 컨테이너가 시작될 때마다 실행할 명령어를 설정하며, Dockfile에서 한번만 사용할 수 있다.

## 2.4.3 Dockerfile 빌드

### 2.4.3.1 이미지 생성

-t: 생성될 이미지의 이름(mybuild:0.0) 설정

./: build 명령어 끝에 Dockerfile이 저장된 경로를 입력한다

- local이기에 현재 디렉토리를 설정했지만, 외부 URL로부터 Dockerfile의 내용을 가져와 빌드할 수 있다.

```
docker build -t mybuild:0.0 ./
```

-P 옵션: 이미지에 설정된 EXPOSE의 모든 포트를 호스트와 연결하도록 설정 (여기서는 80 포트)

```
docker run -d -P --name myserver mybuild:0.0
```

```
docker port myserver //연결된 포트 확인 가능
docker images --filter "label=purpose=pratice" //해당 라벨 이미지만 보여줌
docker ps -f "name=myserver" //myserver 이름을 가진 애만 보여줌
```

### 2.4.3.2 빌드 과정 살펴보기

이미지 빌드를 하면 빌드 컨텍스트라는 걸 읽는데, Dockerfile이 위치한 Directory가 빌드 컨텍스트가 된다.

**빌드 컨텍스트란?** 각종 파일, 소스코드, 메타데이터 등을 담고 있는 디렉터리를 의미

→ 빌드 컨텍스트는 하위 디렉토리까지 모두 포함되지 때문에 불필요한 파일이 포함되지 않도록 한다. (깃 같은 외부 저장소나 루트 디렉토리(하지 않는 게 Best)에서 빌드하는 경우 조심)

**.dockerignore**: 빌드 시 이 파일에 명시된 이름의 파일을 컨텍스트에서 제외 (.gitignore와 유사한 기능)

```
test2.html
*.html      //모든 html 파일
*/*.html    //하위 디렉토리의 html 파일
test.htm?   //e.g. test.htma, test.htmb 등이 컨텍스트에서 제외
!test*.html //특수한 파일만 포함하고 싶은 경우, !를 붙인다.
```

### Dockerfile을 이용한 컨테이너 생성과 커밋

build 명령어를 사용하면 이미지를 생성해낸다. 중간 중간 명령어가 실행될 때마다 임시 Container가 실행되고 삭제 된다

→ Dockerfile의 명령어 줄 수만큼 이미지 레이어가 존재하게 되고, 중간에 컨테이너도 같은 수만큼 생성되고 삭제됨

<img width="991" alt="_2021-06-30__9 30 33" src="https://user-images.githubusercontent.com/26040955/123973961-2ffa1480-d9f7-11eb-9b5a-052e2b0b1bd7.png">

### 캐시를 이용한 이미지 빌드

이전에 빌드했던 Dockerfile에 같은 내용이 있다면 build 명령어는 이를 새로 빌드하지 않고 이전 캐시된 이미지 레이어를 사용하여 빌드.

![_2021-06-30__9 33 14](https://user-images.githubusercontent.com/26040955/123973969-338d9b80-d9f7-11eb-9cf8-30abdb1dc8f2.png)

### 캐시가 필요 없는 경우

매번 github 소스코드를 가져오는 경우, 캐시가 필요없는데 RUN git clone ... 을 사용해 이미지를 빌드했다면, 캐시를 하게 된다. 따라서 아래와 같이 —no-cache option을 추가

```
docker build --no-cache -t mybuild:0.0
```

![_2021-06-30__9 48 14](https://user-images.githubusercontent.com/26040955/123973982-36888c00-d9f7-11eb-9186-bbddceb2f49a.png)

ubuntu 이미지 받아오는 곳에 CACHED가 표시된 이유는 이미 다운로드되어있어서 그런 듯

### 2.4.3.3 멀티 스테이지를 이용한 Dockerfile 빌드

예시) Go로 작성된 소스코드를 빌드하기 위해서 Go와 관련된 빌드 툴과 라이브러리가 미리 설치되어 있어야 함. test하기 위한 소스코드와 Dockerfile

```go
package main
import "fmt"
func main() {
	fmt.Println("hello world")
}
```

```
FROM golang
ADD main.go /root
WORKDIR /root
RUN go build -o /root/mainApp /root/main.go
CMD ["./mainApp"]
```

docker image 크기가 굉장히 큰데, 각종 패키지 및 라이브러리 크기가 차지하고 있는 것이다.

![_2021-06-30__9 53 04](https://user-images.githubusercontent.com/26040955/123973998-3a1c1300-d9f7-11eb-85e5-ae15eba1a16b.png)

이는 **멀티 스테이지** 빌드 방법으로 해결 가능

→ 실행 파일만 최종 이미지 결과물에 포함시키게 된다 

→ alpine: 리눅스 배포판 이미지로서 경량화된 애플리케이션 이미지를 간단히 생성

```
FROM golang
ADD main.go /root
WORKDIR /root
RUN go build -o /root/mainApp /root/main.go
CMD ["./mainApp"]

FROM alpine:latest
WORKDIR /root
COPY --from=0 /root/mainApp . //첫번째 FROM에서 빌드된 이미지의 최종 상태를 alpine:lastest 이미지에 복사
CMD ["./mainApp"]
```

![_2021-06-30__10 05 51](https://user-images.githubusercontent.com/26040955/123974014-3daf9a00-d9f7-11eb-8102-411b30fef251.png)

**멀티 스테이지 빌드 시 2개 이상의 이미지 활용 가능**

## 기타 Dockerfile 명령어

[https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)

**ENV**: 환경변수 설정 (docker run 시에 -e로 재설정하면 기존 값은 덮어 쓰여짐)

```
ENV test /home

docker run -e test=myvalue
```

**VOLUME**: 호스트와 공유할 컨테이너 내부 디렉터리 설정

```
VOLUME /home/volume
VOLUME ["/var/www", "/var/log/apache2", "/etc/apache2"]
```

**ARG**: **build 명령어를 실행**할 때 추가로 입력을 받아 Dockerfile 내에서 사용될 변수의 값을 결정

```
FROM ubuntu:14.04
ARG my_arg
...

docker build --build-arg my_arge=/home ...
```

**USER**: 컨테이너 내에서 사용될 사용자 계정의 이름이나 UUID를 설정하면, 해당 권한으로 실행됨

→ 기본이 root 권한이기 때문에 웬만하면 사용자 계정으로 설정

```
USER alicek106
```

**ONBUILD**: 빌드된 이미지를 기반으로 하는 다른 이미지가 Dockerfile로 생성될 때 실행할 명령어

**STOPSIGNAL**: 컨테이너가 정지될 때 사용될 시스템 콜의 종류 지정

→ default: SIGTERM

**HEALTHCHECK**: 컨테이너에서 동작하는 애플리케이션의 상태를 체크

Ref. [https://nirsa.tistory.com/68](https://nirsa.tistory.com/68)

**COPY**: 로컬 디렉토리에서 읽어 들인 컨텍스트로부터 이미지에 파일을 복사하는 역할을 한다.

**ADD**는 외부및 tar 파일에서도 파일을 추가할 수 있다

→ ADD보다 COPY를 권장 한다. ADD를 써서 외부에서 추가할 경우 어떤 파일이 추가될지 정확히 모르기 때문에...

### ENTRYPOINT vs CMD

공통점: 컨테이너가 시작될 떄 수행할 명령을 지정한다.

다른점: Entrypoint는 커맨드를 인자로 받아 사용할 수 있는 스크립트의 역할을 할 수 있다

```bash
# 출력값: /bin/bash
# entrypoint의 인자가 CMD
# entrypoint를 설정하지 않는다면, CMD를 그대로 실행하지만 entrypoint를 설정하면 CMD는 인자가 된다.
docker run -i -t --entrypoint="echo" --name yes_entrypoint ubuntu:14.04 /bin/bash
```

**entrypoint를 이용한 스크립트 실행**

```bash
docker run -i -t --name entrypoint_sh --entrypoint="/test.sh" ubuntu:14.04 /bin/bash
```

### Dockerfile로 빌드할 때 주의할 점

- 하나의 명령어를 \로 나눠서 가독성을 높이는 것.
- .dockerignore 파일을 작성해 불필요한 파일을 빌드 컨텍스트에 포함하지 않는 것
- 빌드 캐시를 이용해 기존 이미지 레이어를 재사용하는 법

**도커 이미지 구조와 Dockerfile의 관계**

Dockerfile을 막 작성하면 저장 공간을 불필요하게 차지하는 이미지나 레이어가 너무 많은 이미지가 생성될 수 있다.

```bash
FROM ubuntu:14.04
RUN mkdir /test
RUN fallocate -l 100m /test/dummy
RUN rm /test/dummy
```

위처럼 가상으로 만들어서 컨테이너를 할당하고 바로 삭제하더라도 100M는 최종 이미지 사이즈에 잡히게 된다. (삭제하더라도 이전 이미지 레이어에 남아있기 때문에..)

이를 해결하려면, 여러 명령어를 하나의 RUN으로 합치면 된다

```bash
FROM ubuntu:14.04
RUN mkdir /test && \
fallocate -l 100m /test/dummy &&\
rm /test/dummy
```

→ 레이어 이미지 구조는 위 경우에는 단점으로 작용할 수 있지만, 서로 다른 Application이 이미지 레이러를 공유할 수 있는 경우에는 장점으로 작용한다
