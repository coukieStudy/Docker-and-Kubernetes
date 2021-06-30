## 2.2.8 컨테이너 로깅

#### json-file 로그 사용하기

```bash
docker logs {컨테이너 이름}  # 컨테이너 이름에 해당하는 로그 출력
```



```bash
docker run -d --name no_password_mysql \
mysql:5.7
```

 docker ps 를 해도 컨테이너가 생성되었으나 실행되지 않은 것을 확인할 수 있음. docker start을 해도 마찬가지

```bash
➜  ~ docker logs no_password_mysql
[ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
    You need to specify one of the following:
    - MYSQL_ROOT_PASSWORD
    - MYSQL_ALLOW_EMPTY_PASSWORD
    - MYSQL_RANDOM_ROOT_PASSWORD
```

```bash
docker logs --tail 2 mysql			 # 로그파일의 마지막 두 줄만 출력
docker logs --since {Unix timestamp} mysql	 # 특정 시간 이후의 로그 출력
docker logs -t -f mysql 			# -t : 타임스탬프 표시 / -f : 실시간으로 출력되는 내용 확인

➜  ~ docker run -it --name logtest ubuntu:14.04
root@b9bbacdc4d1a:/# echo test!
test!
root@b9bbacdc4d1a:/# exit
exit

➜  ~ docker logs logtest  			# docker run -i -t 옵션을 사용한 컨테이너에 사용 가능
root@b9bbacdc4d1a:/# echo test!
test!
root@b9bbacdc4d1a:/# exit
exit


docker run -it \
--log-opt max-size=10k \			# 로그 파일의 최대 크기. default = -1
--log-opt max-file=3 \				# 로그 파일의 개수.  default = 1. 
--name logtest ubuntu:14.04
```

> JsonFile logging 참고 : https://docs.docker.com/config/containers/logging/json-file/



위 로그들의 저장 위치는 다음과 같다 

```bash
/var/lib/docker/containers/${CONTAINER_ID}/${CONTAINER_ID}-json.log
```

Mac에서 바로 접근이 불가해서 다음 링크를 참조

> https://gist.github.com/BretFisher/5e1a0c7bcca4c735e716abf62afad389



#### Syslog 사용하기

syslog는 유닉스 계열의 운영체제에서 로그를 수집하는 표준 중 하나로, 커널, 보안 등 시스템과 관련된 로그, 어플리케이션의 로그 등 다양한 로그를 수집해 저장한다. 

syslog - Mac에서 재현 불가 ㅠㅠ

```bash
~ docker run -d --name syslog_container \
--log-driver=syslog \
ubuntu:14.04 \
echo syslogtest

bc0cc308c56957b58628d65b482f082f6c6b4a2d6f52e115f19addc31f2fd336
docker: Error response from daemon: failed to initialize logging driver: Unix syslog delivery error.
```

rsyslog를 통해 원격 서버로 로그 보내기

```bash
docker run -it --name rsyslog_server \
-h rsyslogServer \				# 컨테이너의 host name
-p 514:514 -p 514:514/udp \			# 포트 포워딩 / tcp udp 모두 514번 포트 개방
ubuntu:14.04

root@rsyslogServer:/# vi /etc/rsyslog.conf

# provides UDP syslog reception
$ModLoad imudp		  	 	# 원래 주석 있는데 해제
$UDPServerRun 514			# 원래 주석 있는데 해제

# provides TCP syslog reception
$ModLoad imtcp			    	# 주석 해제
$InputTCPServerRun 514			# 주석 해제

# 이후 ifconfig로 ip 확인

➜  ~ docker run -it --name rsyslog_client \
--log-driver=syslog \
--log-opt syslog-address=tcp://172.17.0.3:514 \		# Server의 ip : port
--log-opt tag="client log" \			    	# 로그 접두사
ubuntu:14.04
root@7da2ac48caa1:/# echo test
test


# 다시 서버로 돌아와서 syslog 확인

root@rsyslogServer:/# tail /var/log/syslog
client log[1573]: #033]0;root@7da2ac48caa1: /#007root@7da2ac48caa1:/# echo test#015
client log[1573]: test#015

# --log-opt syslog-facility 로 로그 저장 파일 변경 가능
# /var/log/ 아래에 syslog-facility 로 생성한 로그 파일이 생성
```



### fluentd 로깅

fluentd는 각종 로그를 수집하고 저장할 수 있는 기능을 가진 오픈소스 도구로서, 도커 엔진의 컨테이너 로그를 fluentd를 통해 저장할 수 있도록 플러그인을 제공한다. 수집되는 데이터를 AWS, S3, MongoDB 등 다양한 저장소에 저장할 수 있다. 

도커 서버 -> fluentd 서버 -> MongoDB 서버 연결

1. MongoDB 서버 

   ```bash
   docker run --name mongoDB -d \
   -p 27017:27017 \
   mongo
   ```

2. FluentD 서버

   ```bash
   <source><source>
     @type forward
   </source>
   
   <match docker.**>  # docker로 시작하는 로그들 매치 
     @type mongo
     database nginx
     collection access
     host 172.17.0.3   # mongo 서버 ip
     port 27017
     flush_interval 10s
   </match>
   
   # 를 fluentd.conf 파일에 저장 후
   
   docker run -d --name fluentd -p 24224:24224 \
   -v $(pwd)/fluent.conf:/fluentd/etc/fluent.conf \
   -e FLUENT_CONF=fluent.conf \
   alicek106/fluentd:mongo # fluentd에 mongodb 플러그인을 설치한 이미지
   ```

3. Nginx 서버

   ```bash
   docker run -p 80:80 -d \
   --log-driver=fluentd \
   --log-opt fluentd-address=172.17.0.4:24424 \
   --log-opt tag=docker.nginx \
   nginx
   ```

   Localhost:80 접속해서 로그 남기고 확인

   ```bash
   docker exec -it mongoDB mongo
   > show dbs
   admin   0.000GB
   config  0.000GB
   local   0.000GB
   nginx   0.000GB
   
   > use nginx
   switched to db nginx
   > show collections
   access
   
   > db['access'].find()
   ... \"GET / HTTP/1.1\" 200 612 \"-\" \"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36\" \"-\"}
   ...
   ```

   



