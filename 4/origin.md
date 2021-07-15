# 04. Docker Compose

> 다중 컨테이너 docker application을 정의하고 실행하는 툴

## 1. docker compose를 사용하는 이유

- 매번 run 명령어에 옵션을 설정해서 컨테이너를 생성하는 것이 아닌 여러 개의 컨테이너를 하나의 서비스로 정의해 컨테이너를 묶음으로 관리할 수 있게 해준다.
- 여러 개의 컨테이너 옵션과 환경을 정의한 파일을 읽어 컨테이너를 순차적으로 생성하는 방식으로 동작한다.
- 컨테이너의 수가 많아지고 정의할 옵션이 많아지면 사용하기 좋다.

```powershell
% docker run -d --name mysql \
alicek106/composetest:mysql \
mysqld
Unable to find image 'alicek106/composetest:mysql' locally
mysql: Pulling from alicek106/composetest
Image docker.io/alicek106/composetest:mysql uses outdated schema1 manifest format. Please upgrade to a schema2 image for better future compatibility. More information at https://docs.docker.com/registry/spec/deprecated-schema-v1/
d89e1bee20d9: Pull complete 
9e0bc8a71bde: Pull complete 
27aa681c95e5: Pull complete 
a3ed95caeb02: Pull complete 
7ab04d11bb96: Pull complete 
Digest: sha256:36eea6b0a8b767ef51dbc607c1249330ff762756c6c8ba72f8c565d2833818db
Status: Downloaded newer image for alicek106/composetest:mysql
22a5e319ee597322a8dd871aecf7173012c527f34fb8492521e5932cc55bd929

% docker run -d -p 80:80 \  
--link mysql:db --name web \
alicek106/composetest:web \
apachectl -DFOREGROUND
81a6306e1de9038b5b582ac7e204644ec876be3d7845c81e574996282b5ca791

% docker ps
CONTAINER ID   IMAGE                         COMMAND                  CREATED          STATUS          PORTS                               NAMES
81a6306e1de9   alicek106/composetest:web     "apachectl -DFOREGRO…"   27 seconds ago   Up 26 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   web
22a5e319ee59   alicek106/composetest:mysql   "mysqld"                 26 minutes ago   Up 26 minutes                                       mysql
```

## 3. docker compose 사용

### 3.1 기본 사용법

YAML 파일 작성

```yaml
# docker-compose.yml
version: '3.0'
services:
  web:
    image: alicek106/composetest:web
    ports:
      - "80:80"
    links:
      - mysql:db
    command: apachectl -DFOREGROUND
  mysql:
    image: alicek106/composetest:mysql
    command: mysqld
```

명령어 실행

```powershell
% pwd
docker-sample-app

% docker-compose up -d # docker-compose.yml 파일 읽어서 도커 엔진에 컨테이너 생성 요청
Pulling mysql (alicek106/composetest:mysql)...
mysql: Pulling from alicek106/composetest
Image docker.io/alicek106/composetest:mysql uses outdated schema1 manifest format. Please upgrade to a schema2 image for better future compatibility. More information at https://docs.docker.com/registry/spec/deprecated-schema-v1/
d89e1bee20d9: Pull complete
9e0bc8a71bde: Pull complete
27aa681c95e5: Pull complete
a3ed95caeb02: Pull complete
7ab04d11bb96: Pull complete
Digest: sha256:36eea6b0a8b767ef51dbc607c1249330ff762756c6c8ba72f8c565d2833818db
Status: Downloaded newer image for alicek106/composetest:mysql
Pulling web (alicek106/composetest:web)...
web: Pulling from alicek106/composetest
d89e1bee20d9: Already exists
9e0bc8a71bde: Already exists
27aa681c95e5: Already exists
a3ed95caeb02: Already exists
15bc302aa28e: Pull complete
7233974738a3: Pull complete
732ac06e8a0b: Pull complete
Digest: sha256:91e141799b618df4d665a1cade579c19c1f7e40e6c2ed5ff8acd87d834130b87
Status: Downloaded newer image for alicek106/composetest:web
Creating docker-sample-app_mysql_1 ... done
Creating docker-sample-app_web_1   ... done

% docker-compose ps 
          Name                     Command           State                Ports              
---------------------------------------------------------------------------------------------
docker-sample-app_mysql_1   mysqld                   Up                                      
docker-sample-app_web_1     apachectl -DFOREGROUND   Up      0.0.0.0:80->80/tcp,:::80->80/tcp

% docker-compose up -d --scale mysql=3 # 서비스에 여러 컨테이너 생성
Creating docker-sample-app_mysql_2 ... done
Creating docker-sample-app_mysql_3 ... done
docker-sample-app_web_1 is up-to-date

% docker-compose ps
          Name                     Command           State                Ports              
---------------------------------------------------------------------------------------------
docker-sample-app_mysql_1   mysqld                   Up                                      
docker-sample-app_mysql_2   mysqld                   Up                                      
docker-sample-app_mysql_3   mysqld                   Up                                      
docker-sample-app_web_1     apachectl -DFOREGROUND   Up      0.0.0.0:80->80/tcp,:::80->80/tcp

% docker-compose down # 프로젝트 삭제: 컨테이너 정지 -> 삭제
Stopping docker-sample-app_mysql_2 ... done
Stopping docker-sample-app_mysql_3 ... done
Stopping docker-sample-app_web_1   ... done
Stopping docker-sample-app_mysql_1 ... done
Removing docker-sample-app_mysql_2 ... done
Removing docker-sample-app_mysql_3 ... done
Removing docker-sample-app_web_1   ... done
Removing docker-sample-app_mysql_1 ... done
Removing network docker-sample-app_default
```

### 3.2 docker-compose 활용

**명령어 옵션**

- -p project_name
- -f docker_compose_file_location

**서비스 정의**

- image
- links
- environment
- command
- depends_on
- ports
- build
- extends

**네트워크 정의**

- driver
- ipam (IP Address Manager)
- external

볼륨 정의

ref) [https://ihp001.tistory.com/199](https://ihp001.tistory.com/199)

**YAML 파일 검증**

```powershell
% docker-compose config
```

## 4. Docker 학습을 마치며

컨테이너 기술이 특정 벤더 또는 회사에 의존적으로 개발되지 않도록 컨테이너 표준을 정의하는 OCI(Open Container Initiative) 발표.
