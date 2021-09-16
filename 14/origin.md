# 14. 쿠버네티스 모니터링

- 프로메테우스(Prometheus): 쿠버네티스에서 가장 많이 사용되는 오픈소스 모니터링 데이터베이스
- 그라파나(Grafana): 데이터베이스에 수집된 데이터를 시각화하기 위한 도구
- AlertManager: ex) 슬랙 알람을 보내는 기능

## 14.1 모니터링 기본 구조
![image1](https://user-images.githubusercontent.com/40483081/133448494-2d74f3d8-4e4a-464f-bb63-2f15d10c9932.png)

**exporter**
- 모니터링 에이전트
- /metrics 경로를 외부에 노출시켜 데이터를 수집할 수 있도록 인터페이스를 제공하는 서버
- 요청 당시의 데이터를 리턴할 뿐 자체적으로 히스토리를 저장하는 기능은 없다.

**CAdvisor**
- exporter 중 하나
- 컨테이너에 관련된 모니터링 데이터를 확인할 수 있는 모니터링 도구

**프로메테우스의 메트릭 수집 방식(CAdvisor)**

1. CAdvisor의 /metrics 경로로 컨테이너 메트릭을 요청
2. CAdvisor가 프로메테우스 형식의 시계열 메트릭을 반환 (키-값 형태: 메트릭이름-메트릭값)
3. Prometheus는 CAdvisor로부터 수집된 메트릭을 저장


## 14.2 모니터링 메트릭 분류

**인프라 수준**
- 호스트 레벨 메트릭
- 호스트에서 사용 중인 파일 디스크립터 개수, 디스크 사용량, NIC에서의 패킷 전송량
- node exporter

**컨테이너 수준**
- 컨테이너 레벨에서의 메트릭
- 컨테이너별 CPU 사용량, 메모리 사용량, 컨테이너 프로세스의 상태, 컨테이너에 할당된 리소스 할당량
- Cadvisor는 컨테이너 수준의 메트릭 도구

**애플리케이션 수준**
- 인프라, 컨테이너를 제외한 메트릭
- 마이크로서비스에서 발생하는 트레이싱 데이터, 애플리케이션 로직에 종속적인 데이터, 서버 프레임워크에서 제공하는 모니터링 데이터

## 14.3 쿠버네티스 모니터링 기초

![image](https://user-images.githubusercontent.com/40483081/133469210-0d657260-1019-4dcf-9dad-bb50c4b65710.png)

**metrics-server**
- kubelet으로부터 주기적으로 노드, 포드 메트릭을 수집

**metric-server 동작 원리**
- metric server를 APIService로 등록 (https://github.com/alicek106/start-docker-kubernetes/blob/9ec196a0937667fff43d05926bac2cafea3cd874/chapter14/components.yaml#L43)
- API Server에 metrics.k8s.io로 요청
- metric-server로 요청 전달
- metric-server가 처리한 응답을 반환 (ex: 노드,포드의 리소스 사용량 메트릭)

**kube-state-metrics**
- 쿠버네티스 리소스의 상태에 관련된 메트릭을 제공하는 addOn
- ex: 포드의 상태, 디플로이먼트의 레플리카 개수

**node-exporter**
- 인프라 수준의 메트릭 제공을 위한 exporter
- 파일시스템, 네트워크 패킷 등과 같이 호스트 측면의 메트릭 제공


## 참고
- [프로메테우스 개념 구조 및 quick start](https://owin2828.github.io/devlog/2020/03/13/etc-5.html)
