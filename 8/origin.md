# 08.인그레스(Ingress)

##### **인그레스**는 일반적으로 외부에서 내부로 향하는 것을 지칭하는 단어

서비스 오브젝트가 외부 요청을 받아들이기 위한 것이었다면, 인그레스는 외부 요청을 어떻게 처리할 것인지 네트워크 7계층 레벨에서 정의하는 쿠버네티스 오브젝트

<img width="537" alt="image-20210804203749109" src="https://user-images.githubusercontent.com/66865817/128184418-bb1d882e-aa15-4721-a69a-e7548d3a40fa.png">

인그레스 오브젝트가 담당할 수 있는 기본적인 기능

- 외부 요청의 라우팅 : 특정 경로로 들어온 요청을 어떠한 서비스로 전달할지 정의하는 라우팅 규칙을 설정
- 가상 호스트 기반의 요청 처리 : 같은 IP에 대해 다른 도메인 이름으로 요청이 도착했을 때, 어떻게 처리할 것인지 정의
- SSL/TLS 보안 연결 처리 : 여러 개의 서비스로 요청을 라우팅할 때, 보안 연결을 위한 인증서를 쉽게 적용



## 8.1 인그레스를 사용하는 이유

NodePort, LoadBalancer 타입의 서비스를 사용해도 위 기능들을 구현하는 것이 불가능하지 않음

인그레스에 접근하기 위한 **단 하나의 URL만 존재**

클라이언트는 인그레스의 URL로만 접근, 요청은 인그레스에서 정의한 규칙에 따라 처리된 뒤 적절한 디플로이먼트의 포드로 전달

라우팅 정의나 보안 연결 등과 같은 세부 설정은 인그레스에 의해 수행됨 (서비스와 디플로이먼트가 아님)

디플로이먼트에 대해 일일이 설정할 필요 없이 하나의 설정 지점에서 처리 규칙을 정의하면 됨

외부 요청에 대한 처리 규칙을 쿠버네티스 자체의 기능으로 관리할 수 있다는 것이 인그레스의 핵심



## 8.2 인그레스의 구조

인그레스를 생성해도, 이것만으로는 아무 일도 일어나지 않음. 인그레스는 단지 요청을 처리하는 규칙을 정의하는 선언적인 오브젝트 일 뿐이기 때문

인그레스 컨트롤러(Ingress Controller)라고 하는 특수한 서버에 적용해야만 그 규칙을 사용할 수 있음

실제로 외부 요청을 받아들이는 것은 인그레스 컨트롤러 서버, 이 서버가 인그레스 규칙을 로드해 사용

대표적으로 Nginx 웹 서버 인그레스 컨트롤러



## 8.3 인그레스의 세부 기능 : annotation을 이용한 설정

쿠버네티스 클러스터 자체에서 기본적으로 사용하도록 설정된 인그레스 컨트롤러가 존재하는 경우가 있는데, 어떤 인그레스 컨트롤러를 사용할 것인지 반드시 인그레스에 명시해주는 것이 좋음



## 8.4 Nginx 인그레스 컨트롤러에 SSL/TLS 보안 연결 적용

인그레스 장점 중 하나가 앞쪽에 있는 인그레스 컨트롤러에서 편리하게 SSL/TLS 보안 연결을 설정할 수 있다는 점

요청이 전달되는 에플리케이션에 대해 모두 인증서 처리 가능, 일종의 gateway 역할

클라우드 환경에서 LoadBalancer 타입의 서비스를 사용할 계획이라면 클라우드 플랫폼 자체에서 관리해주는 인증서를 인그레스 컨트롤러에서 적용할 수도 있음

(직접 서명한 루트 인증서를 통해 Nginx 인그레스 컨트롤러에 적용하는 기초적인 방법)

##### 인증서를 통해 보안 연결을 설정했을 때는 http로 접근해도 자동으로 https로 리다이렉트 됨



## 8.5 여러 개의 인그레스 컨트롤러 사용하기
