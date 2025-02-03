## docker compose
- 도커 컨테이너를 여러개 띄우고 이들간의 네트워크에서 소통이 필요하고 의존성이 있게 디자인 할 수 있습니다. 그런경우에 컨테이너를 띄우고 환경변수를 설정하는 행위는 반복될 수 밖에 없습니다. 그러면 이 과정에서 쉽게 실수나 에러가 발생할 수 있습니다. 그때 사용하는 것이 **docker compose** 입니다!

빠르게 예를 한번봅시다
``` yaml
version: '3.7'

services: # services들은 구성할 컨테이너를 뜻하고 이 컨테이너의 이름은 service의 이름을 따라갑니다.
  
  todo-web: # 컨테이너 이름 = 서비스 이름
    image: diamol/ch06-todo-list # 컨테이너의 이미지
    ports: 
      - "8020:80" #컨테이너의 포트: 호스트 컴퓨터의 내부 포트
    networks: # 해당 서비스가 사용할 도커 네트워크 이름. 
      - app-net

networks: # 도커에는 내부적으로 사용하는 도커 네트워크라는 네트워크가 존재합니다. 이를 정의합니다. 네크워크가 서비스와 마찬가지로 여러개 일 수 있습니다.
  app-net: # 도커 컴포즈에서 서비스를 구성할때 사용할 네트워크 이름.
    external:
      name: nat # 사전에 만들어놓은 외부 네트워크 이름
```

```
# 해당 컴포즈 파일이 있는 루트 디렉토리에서 입력
docker-compose up
```

- 여러 주석을 달아 보았는데, 도커 컴포즈를 이용하면 여러 서비스들과 네트워크들을 정의할 수 있는 yaml 파일을 기반으로 분산화된 어플리케이션을 하나의 컴퓨터에서 동작시킬 수 있습니다.

한번 여러개의 서비스들로 구성된 도커 컴포즈 파일을 봅시다.

```yaml
version: "3.7"

services:
  todo-db:
    image: diamol/postgres:11.5
    ports:
      - "5433:5432"
    networks:
      - app-net

  todo-web:
    image: diamol/ch06-todo-list
    ports:
      - "8030:80"
    environment:
      - Database:Provider=Postgres
    depends_on:
      - todo-db
    networks:
      - app-net
    secrets:
      - source: postgres-connection
        target: /app/config/secrets.json

networks:
  app-net:

secrets:
  postgres-connection:
    file: ./config/secrets.json
```
- 위 예시는 데이터베이스를 연결해서 간단하게 todo를 기록하는 어플리케이션입니다.
- `sercrets`에서 유추할 수 있듯이 환경변수의 경우는 `./config/secrets.json`에서 정의한 값들을 사용하게 됩니다.

### 도커 컴포즈를 사용해서 어플리케이션 묶음을 여러개 띄우는 방법
``` yaml
version: '3.7'

services:

  accesslog:
    image: diamol/ch04-access-log
    networks:
      - app-net

  iotd:
    image: diamol/ch04-image-of-the-day
    ports:
      - "80"
    networks:
      - app-net

  image-gallery:
    image: diamol/ch04-image-gallery
    ports:
      - "8010:80" 
    depends_on:
      - accesslog
      - iotd
    networks:
      - app-net

networks:
  app-net:
    external:
      name: nat
``` 
[docker_compose_app_structure](../images/docker-compose.png)
- 위와 같은 yaml 파일을 통해 어플리케이션을 관리하면 그림과 같이 실행되게 됩니다.
- 그런데 예를 들어 봅시다. 만약 내가 돌리는 app에서 iotd에 트래픽을 많이 가하는 경우에를 생각해봅시다.
  - 결국에 scale-up을 하기 위해 iotd의 수를 늘려야합니다.
  - 그러면 이와 같이 실행하면 됩니다. `docker-compose up -d --scale iotd=3`
  - 이렇게 실행하면 도커 컴포즈가 iotd를 3개 추가해서 실행하게 됩니다.
- 그러면 이러한 의문이 생깁니다.
  - 똑같은 컨테이너 이름을 갖는 컨테이너가 3개가 나오는데 image-gallery는 어떻게 구분해서 요청을 보내지?
  - 그리고 트래픽이 많아지면 로드 밸런싱을 어떻게 하지?
- 결론적으로 이렇게 결론이 나옵니다.
  - 각각의 컨테이너는 이름은 같지만, 모두 다른 ip 주소를 가지게 됩니다. 이는 가상의 도커 네트워크(app-net)에서 가상의 ip 주소를 가지게 됩니다. 그리고 ip하면 여기에 해당하는 DNS 어떻게 정의되는지 생각해볼 수 있는데. 바로 그것이 컨테이너 이름입니다. 아래 그림을 보시면 nslookup으로 조회했더니 3개의 ip가 나온다는것을 알 수 있습니다.
  - [nslookup결과](../images/nslookup_result.png)
  - 근데 여기서 또 nslookup을 수행하면 순서가 바뀌는것을 알 수 있습니다. 이렇게 해당 요청이 들어오면 내부적으로 도커 네트워크가 로드 밸런싱을 수행하게 됩니다.
  - 이 방법으로 컨테이너간의 트래픽을 분산시킬 수 있습니다.