## 2.3. 도커 이미지

docker search {이름} : docker hub에서 검색

#### 2.3.1 도커 이미지 생성

```bash
docker run -it --name commit_test ubuntu:14.04
root@b5f4021d710b:/# echo test_first >> first

docker commit -a "david1403" -m "first commit" \
> commit_test \
> commit_test:first

# docker commit [OPTIONS] CONTAINER [REPOSITORY:[TAG]]

~docker images
REPOSITORY    TAG                   IMAGE ID          CREATED         SIZE
commit_test   second              46575a3342d5        6 seconds ago       197MB
commit_test    first               0df3edf0571f        4 minutes ago       197MB
```

#### 2.3.2 이미지 구조 이해

```bash
 docker docker inspect ubuntu:14.04 | grep -A 20 Layers
            "Layers": [
                "sha256:f2fa9f4cf8fd0a521d40e34492b522cee3f35004047e617c75fadeb8bfd1e6b7",
                "sha256:30d3c4334a2379748937816c01f5c972a8291a5ccc958d6b33d735457a16196e",
                "sha256:83109fa660b2ed9307948505abd3c1f24c27c64009691067edb765bd3714b98d"
            ]
        }
    }
]
➜  docker docker inspect commit_test:first | grep -A 20 Layers
            "Layers": [
                "sha256:f2fa9f4cf8fd0a521d40e34492b522cee3f35004047e617c75fadeb8bfd1e6b7",
                "sha256:30d3c4334a2379748937816c01f5c972a8291a5ccc958d6b33d735457a16196e",
                "sha256:83109fa660b2ed9307948505abd3c1f24c27c64009691067edb765bd3714b98d",
                "sha256:c2b2febaa316e78220d490cafaba589d7a1ef751353686edd64c705c611d37ff"
            ]
        }
    }
]
➜  docker docker inspect commit_test:second | grep -A 20 Layers
            "Layers": [
                "sha256:f2fa9f4cf8fd0a521d40e34492b522cee3f35004047e617c75fadeb8bfd1e6b7",
                "sha256:30d3c4334a2379748937816c01f5c972a8291a5ccc958d6b33d735457a16196e",
                "sha256:83109fa660b2ed9307948505abd3c1f24c27c64009691067edb765bd3714b98d",
                "sha256:c2b2febaa316e78220d490cafaba589d7a1ef751353686edd64c705c611d37ff",
                "sha256:1c5f727f9617ed37c1829aa650e9920790aeb34e9d76728719a62a73dbf96390"
            ]
        }
    }
    
    
# docker history {image} 를 통해 커밋 내역 확인 가능
```

Commit_test:first 이미지를 삭제할 수 있을까? -> 그럴 수 없다 -> 이 이미지를 사용하고 있는 컨테이너가 존재하기 때문

해당 컨테이너를 삭제한 뒤, 이미지를 삭제할 수 있다.

```bash2
docker stop commit_test2 && docker rm commit_test2
docker rmi commit_test:first
Untagged: commit_test:first

그러나 docker images -a 로 확인해보면 해당 이미지의 레이어 파일이 삭제되지는 않았다는 것을 알 수 있다.
REPOSITORY    TAG                   IMAGE ID          CREATED         SIZE
<none>     <none>              0df3edf0571f        21 minutes ago      197MB
```

그 이유는 commit_test:first 이미지를 기반으로 하는 하위 이미지인 commit_test:second 가 존재하기 때문.

docker rmi commit_test:second 는 컨테이너도 없고 하위 이미지도 없기 때문에 바로 삭제가 가능하다.



#### 2.3.3 이미지 추출

```bash
docker save -o ubuntu_14_04.tar ubuntu:14.04
docker load -i ubuntu_14_04.tar

docker export -o rootFS.tar myContainer
docker import -o rootFS.tar myimage:0.0
```

이미지를 추출하고 로드하는 역할은 동일하다.

차이점

1. save / load : 추출되는 이미지는 원본 이미지와 동일 - layer 구조가 동일하다
   export /import : 추출되는 이미지는 원본 이미지를 하나의 다른 layer로 래핑된 새로운 이미지이다.
2. save / load : 추출되는 이미지는 이미지 원본 이미지를 구동할 당시에 했던 컨테이너 설정 정보까지 포함되어 있다.
   export / import : 추출되는 이미지는 설정 정보가 포함되어 있지 않다. 따라서 import 시 옵션을 추가해야 할 수도 있다.

