## 어플리케이션의 신뢰성 확보하기 위한 도커 세팅
만약 도커 컨테이너에서 돌고 있는 어플리케이션이 예상치 못한 에러 상황일때 이를 확인하기 위한 작업들이 필요하다.
그럴때 도커는 프로세스가 종료된 상황이라면 이를 보고 할수 있습니다. 하지만 프로세스는 그대로 실행중이지만 에러를 반환하는 상태라면 정상적인 상황은 아니고 이를 체크하기 위한 과정이 별도로 필요합니다.

### 헬스 체커
이때 헬스 체킹이라는 작업을 주기적으로 실행해 개발자가 커스터마이징하게 지정해놓은 어플리케이션의 로직을 수행하면서 도커 컨테이너에 올라가 있는 어플리케이션이 정상적으로 수행되고 있는 확인합니다.

HEALTHCHECK 도커 인스트럭션을 사용해서 도커 이미지를 통해 일정한 시간간격으로 이 작업을 수행할 수 있도록 정의할 수 있습니다.

```Dockerfile
FROM diamol/dotnet-sdk AS builder

WORKDIR /src
COPY src/Numbers.Api/Numbers.Api.csproj .
RUN dotnet restore

COPY src/Numbers.Api/ .
RUN dotnet publish -c Release -o /out Numbers.Api.csproj

# app image
FROM diamol/dotnet-aspnet

ENTRYPOINT ["dotnet", "/app/Numbers.Api.dll"]
HEALTHCHECK CMD curl --fail http://localhost/health # health check 작업하는 인스트럭션

WORKDIR /app
COPY --from=builder /out/ .
```

### 디펜더시 체크
만약 여러 어플리케이션을 실행하고 있고 이 실행되고 있는 어플리케이션들간에 의존성이 존재하면, 어플리케이션이 하나 죽었을때 이에 대한 체크를 통해 이 어플리케이션에 의존성을 가지고 있는 어플리케이션 정상적으로 종료시켜야합니다. 그렇다고 죽은 어플리케이션을 다시 살리기 위해 도커 컨테이너를 띄우는 행위는 굉장히 위험하기 때문입니다. 기존 데이터나 상태들을 버리게 되기 때문입니다. 그리고 기존 것을 삭제하고 도커 컨테이너를 띄우는 동안 정상적으로 실행될 수 없기 때문에 직관과 달리 모두 종료하는 것이 이 체크를 하는 이유입니다.

디펜더시 체크는 어플리케이션 즉 컨테이너를 만들때만 체크하게 됩니다.

이 또한 도커 이미지에서 수행할 수 있습니다.
```Dockerfile
FROM diamol/dotnet-sdk AS builder

WORKDIR /src
COPY src/Numbers.Web/Numbers.Web.csproj .
RUN dotnet restore

COPY src/Numbers.Web/ .
RUN dotnet publish -c Release -o /out Numbers.Web.csproj

# app image
FROM diamol/dotnet-aspnet

ENV RngApi:Url=http://numbers-api/rng

## 간접적으로 curl을 통해 api의 헬스를 체크하고 어플리케이션을 수행합니다.
CMD curl --fail http://numbers-api/rng && \
    dotnet Numbers.Web.dll

WORKDIR /app
COPY --from=builder /out/ .
```

이렇게 함으로써 의존성 있는 어플리케이션이 준비가 되지 않음을 미리 인지할 수 있고, 해당 어플리케이션을 정상적으로 종료시켜 예상치 못한 오류에서 벗어날 수 있게 합니다.

### 커스텀 유틸리티 이미지 만들기

경우에 따라 curl 과 같이 보안정책상의 문제가 있는 cmd는 사용하지 않는 방법을 위해 새롭게 도커 이미지를 만드는 방법도 있다.
그리고 셸 스크립트나 재시도 횟수 같이 커스텀적으로 설정할 수 있는 값들을 이미지를 별도로 만들면서 정의하기 쉬어진다.
```Dockerfile
FROM diamol/dotnet-sdk AS builder

WORKDIR /src
COPY src/Numbers.Api/Numbers.Api.csproj .
RUN dotnet restore

COPY src/Numbers.Api/ .
RUN dotnet publish -c Release -o /out Numbers.Api.csproj

# http check utility
FROM diamol/dotnet-sdk AS http-check-builder

WORKDIR /src
COPY src/Utilities.HttpCheck/Utilities.HttpCheck.csproj .
RUN dotnet restore

COPY src/Utilities.HttpCheck/ .
RUN dotnet publish -c Release -o /out Utilities.HttpCheck.csproj

# app image
FROM diamol/dotnet-aspnet

ENTRYPOINT ["dotnet", "Numbers.Api.dll"]
# health check 를 curl을 사용하지 않게 할 수 있다.
HEALTHCHECK CMD ["dotnet", "Utilities.HttpCheck.dll", "-u", "http://localhost/health"]

WORKDIR /app
COPY --from=http-check-builder /out/ .
COPY --from=builder /out/ .
```

### 도커 컴포즈에서 헬스 체크와 디펜더시 체크
``` yml

version: "3.7"

services:
  numbers-api:
    image: diamol/ch08-numbers-api:v3
    ports:
      - "8087:80"
    healthcheck:
      interval: 5s
      timeout: 1s
      retries: 2
      start_period: 5s
    networks:
      - app-net

  numbers-web:
    image: diamol/ch08-numbers-web:v3
    restart: on-failure
    environment:
      - RngApi__Url=http://numbers-api/rng
    ports:
      - "8088:80"
    healthcheck:
      test: ["CMD", "dotnet", "Utilities.HttpCheck.dll", "-t", "150"]
      interval: 5s
      timeout: 1s
      retries: 2
      start_period: 10s
    networks:
      - app-net

networks:
  app-net:
    external:
      name: nat

```

- 이 도커 컴포즈에서는 디펜더시 체크를 찾아 볼 수 없습니다. 디펜더시 체크는 단일 서버에 대해서만 지원이 됩니다.
그렇다보니 만약 의존성을 가지고 있는 어플리케이션이 분산서버로 이루어져있다면 디펜더시 체크를 정확히하는것이 어렵습니다.
- 따라서 `restart: on-failure`를 통해 정상적으로 실행될때 까지 컨테이너를 재시작하도록 하고 의존성을 가진 어플리케이션들이 모두 준비가 완료되었을때 비로소 어플리케이션이 실행되게 됩니다. 도커 교과서 서적에서 이 방법을 권장한다고 하는데 동의하게 되었습니다 

### 요약
- 헬스체크는 주기적으로 자주 실행되므로 시스템에 부하를 주는 내용이면 안됩니다.
- 디펜더시 체크는 어플리케이션이 실행될 때만 잘 수행하면 되므로 리소스는 신경을 너무 쓰지 않아도 됩니다. 따라서 간접적으로 컨테이너를 새롭게 띄우는 방법을 수행해서 분산 환경에서도 성공적으로 디펜더시 체크를 수행하는 방법도 좋은 거 같습니다.