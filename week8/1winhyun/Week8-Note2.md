# 도커

- 컨테이너를 사용하여 각각의 프로그램을 분리된 환경에서 실행 및 관리할 수 있는 툴
- 특정 프로그램을 다른 곳으로 쉽게 옮겨서 설치 및 실행할 수 있는 특성
- 장점
  - 매번 귀찮은 설치 과정을 일일이 거치지 않아도 된다.
  - 항상 일관되게 프로그램을 설치할 수 있다.
  - 프로그램 간에 충돌이 일어나지 않는다.

## 도커 컨테이너

- 하나의 컴퓨터 환경 내에서 독립적인 컴퓨터 환경을 구성해서, 각 환경에 프로그램을 별도로 설치할 수 있게 만든 개념
- 하나의 컴퓨터 환경 내에서 여러 개의 미니 컴퓨터 환경을 구성
- 독립성
  - 각 컨테이너마다 서로 각자의 저장 공간을 가지고 있다.
  - 각 컨테이너마다 고유의 네트워크를 가지고 있다. 각자의 IP 주소를 가지고 있다.

## 도커 허브

- 이미지를 저장, 다운로드 받을 수 있는 저장소의 역할을 한다.

## 도커 이미지

- 닌텐도의 게임 칩과 같은 역할을 한다고 보면 된다.
- 프로그램을 실행하는데 필요한 설치 과정, 버전 정보 등을 포함하고 있다.
- 프로그램을 실행하는데 필요한 모든 것을 포함하고 있다.
- 태그
  - 이미지의 특정 버전을 의미한다.
- nginx
  - 여러 기능을 가진 서버 중 하나
  - 웹 서버

## 도커 관련 명령어

- 이미지 확인
  - `docker image ls`
- 이미지 내려받기
  - `docker pull <이미지 이름>`
- 이미지 제거
  - `docker image rm <이미지 ID>`
  - 해당 이미지를 사용 중인 컨테이너(실행 중이거나 중단된 컨테이너 모두)가 없을 때만 삭제 가능
    - ※ 이미지 자체에는 실행/중단 상태가 없다. 컨테이너가 이미지를 사용 중인지 여부가 기준이다.
  - 강제로 삭제하려면 `-f` 옵션 추가 (단, 실행 중인 컨테이너가 사용 중인 이미지는 `-f`로도 삭제 불가)
- 이미지를 통한 컨테이너 생성
  - `docker create <이미지 이름>`
  - 이미지를 로컬에 내려받은 적이 없다면 자동으로 pull 후 컨테이너를 만든다.
  - 실행
    - `docker start <컨테이너 ID>`
- 컨테이너 생성과 실행을 한 번에 실행
  - `docker run <이미지 이름>` (포그라운드 실행, 백그라운드 실행하려면 `-d` 추가)
    - 추가 내용
      - 포그라운드
        - 내가 실행시킨 프로그램의 내용이 화면에서 실행, 출력되는 상태
        - 다른 명령어 입력을 못함
      - 백그라운드
        - 내가 실행시킨 프로그램이 컴퓨터 내부적으로 실행되는 상태
- 컨테이너에 이름을 붙여서 생성, 실행하는 방법
  - `docker run -d --name <새로운 컨테이너 이름> <이미지 이름>`
- 호스트의 포트와 컨테이너의 포트를 연결하기
  - `docker run -d -p <호스트 포트>:<컨테이너 포트> <이미지 이름>`
- `docker stop`과 `docker kill`의 차이점
  - stop
    - 정상적인 종료 방법 (SIGTERM 신호를 보내 프로세스가 정리 후 종료)
  - kill
    - 즉시 강제 종료 (SIGKILL 신호). 데이터 정리 없이 종료되므로 권장하지 않는다.
- 실행 중인 컨테이너 삭제 방법
  - `docker rm -f <컨테이너 ID>`
- 컨테이너 로그 조회
  - `docker logs <컨테이너 ID>`
    - `-f` 추가 시 실시간으로 생성되는 로그 확인 가능
- 실행 중인 컨테이너에 접속하기
  - `docker exec -it <컨테이너 ID> bash`

---

# 도커 볼륨

## 도커 볼륨

- 도커 컨테이너에서 데이터를 영속적으로 저장하기 위한 방법
- 컨테이너 자체의 저장 공간을 사용하지 않고, 호스트 자체의 저장 공간을 공유하는 형태
- 컨테이너의 문제점
  - 도커를 통해 특정 프로그램을 컨테이너로 띄울 수 있다. 여기에 기능이 추가되면 새로운 이미지를 만들어서 컨테이너를 실행해야 함.
  - 컨테이너를 새로운 컨테이너로 교체하면, 기존 컨테이너 내부에 있던 데이터도 같이 삭제
  - 만약 이 컨테이너가 MySQL을 실행시키는 컨테이너라면 데이터가 같이 삭제
- `docker run -v [호스트 디렉토리 절대경로]:[컨테이너 디렉토리 절대경로] [이미지명]:[태그명]`
  - 예시
    ```bash
    docker run -e MYSQL_ROOT_PASSWORD=6316 -d -p 3306:3306 \
      -v /Users/hanseunghyun/Documents/docker-mysql/mysql_data:/var/lib/mysql mysql
    ```
  - 다음과 같이 작성할 경우 원래 컨테이너에서 데이터베이스를 만든 후 컨테이너를 삭제했다가 다시 만들어 실행해도 이전에 만든 데이터베이스가 삭제되지 않고 남아 있다.
  - 다시 컨테이너 생성/실행 시 이전과 다른 password를 설정하면 안 된다.
    - Volume으로 설정해 둔 폴더에 이미 비밀번호 정보가 저장되었기 때문이다.
  - 호스트 디렉토리 경로에 디렉토리가 이미 존재할 경우, 호스트의 디렉터리가 컨테이너의 디렉터리를 덮어씌운다.
    - 즉, 원래 있던 거 삭제해라
- 도커 컨테이너로 MySQL 실행
  - `docker run -e MYSQL_ROOT_PASSWORD=6316 -d -p 3306:3306 mysql`
  - `mysql -u root -p` (MySQL을 실행하는 명령어, 컨테이너 안에서 실행해야 한다.)

## 도커 파일

- 도커 이미지를 만들게 해주는 파일

### FROM

- 베이스 이미지를 지정하는 역할
- 도커 컨테이너를 특정 초기 이미지를 기반으로 추가적인 세팅을 할 수 있다.
  - 베이스 이미지가 특정 초기 이미지
- 컨테이너를 새로 띄워서 환경을 구축할 때 기본 프로그램이 어떤 게 깔려있으면 좋겠는지 선택하는 옵션
- `FROM [이미지 명]`
  - 예시: `FROM openjdk:17-jdk`
- `ENTRYPOINT ["/bin/bash", "-c", "sleep 500"]`
  - 500초 동안 시스템을 일시정지 시키는 명령어
  - 컨테이너가 바로 종료되는 것을 막을 수 있다. 그런 후 `docker exec -it`를 활용해 컨테이너 내부에 직접 들어가서 디버깅을 하면 된다.

### COPY

- 호스트 컴퓨터에 있는 파일을 복사해서 컨테이너로 전달한다.
- `COPY [호스트 컴퓨터에 있는 복사할 파일의 경로] [컨테이너에서 파일이 위치할 경로]`
- 디렉토리를 복사할 경우 마지막에 `/`를 붙여준다.
  - 예시: `COPY my-app /my-app/`
- 모두 복사하고 싶다면 `*` 사용
  - 예시: `COPY *.txt /text-files/`
- 특정 파일을 제외하고 복사하고 싶다면
  - `.dockerignore` 파일을 생성한다.
    - ※ `.dodkcerignore`가 아닌 **`.dockerignore`** 가 올바른 파일명이다.
    - 그 안에 복사하고 싶지 않은 파일명을 작성한다.

### ENTRYPOINT

- 컨테이너가 시작할 때 실행되는 명령어
- 즉, 컴퓨터의 전원을 키고 나서 실행시키고 싶은 명령어

### RUN

- 이미지를 생성하는 과정에서 사용할 명령문 실행
- 예시: `RUN [명령문]`
- ENTRYPOINT는 컨테이너를 생성한 직후 실행, RUN은 이미지 생성 과정에서 실행

### WORKDIR

- 모든 명령문의 실행을 지정한 디렉터리를 기준으로 실행
- 예시: `WORKDIR [작업 디렉토리로 사용할 절대 경로]`

### EXPOSE

- 컨테이너 내부에서 사용 중인 포트를 문서화하기
- 컨테이너 내부에서 어떤 포트에 프로그램이 실행되는지를 문서화하는 역할만 한다.
- `-P` 플래그 사용 시 EXPOSE에 명시된 포트가 자동으로 매핑되므로 아예 없어도 되는 것은 아니다.
- 예시: `EXPOSE [포트 번호]`

## 백엔드 프로젝트를 도커 컨테이너로 올리기

도커 파일 작성

```dockerfile
FROM openjdk:17-jdk

COPY build/libs/*SNAPSHOT.jar app.jar

ENTRYPOINT ["java", "-jar", "/app.jar"]
```

---

# 도커 컴포즈

## Docker Compose

- 여러 도커 컨테이너들을 하나의 서비스로 정의하고 구성해 하나의 묶음으로 관리할 수 있게 도와주는 툴
- 장점
  - 여러 개의 컨테이너를 관리하는데 용이
  - 복잡한 명령어로 실행시키던 걸 간소화

## compose.yml

아래 코드는 `docker run --name webserver -d -p 80:80 nginx`와 동일하다.

```yaml
services: # 하나의 컨테이너를 service라고 부른다
  my-web-server: # 서비스의 이름
    container_name: webserver # 이미지 기반 컨테이너의 이름
    image: nginx # 어떤 이미지를 기반으로 할지
    ports:
      - 80:80 # 어떤 주소로 매핑할지
```

## Docker Compose 명령어

- `docker compose up`
  - compose.yml을 실행시키는 명령어
  - 백그라운드로 실행하려면 `-d` 추가
- `docker compose ps`
  - yml을 통한 컨테이너가 실행 중인지 확인 가능
  - 물론 `docker ps`도 가능
- `docker compose down`
  - 실행 중인 컨테이너 종료 및 삭제
- `docker compose logs`
  - 도커 컴포즈로 실행한 컨테이너 로그 확인 가능
- `docker compose up --build`
  - 이미지를 다시 빌드해서 컨테이너를 실행해야 할 때 사용
- `docker compose pull`
  - 도커 허브에 있는 최신 이미지를 다운받아서 업데이트한다.

## docker compose를 통해 MySQL 실행

아래 compose.yml은 `docker run -e MYSQL_ROOT_PASSWORD=6316 -d -p 3306:3306 -v /Users/hanseunghyun/Documents/docker-mysql/mysql_data:/var/lib/mysql mysql`과 동일하다.

```yaml
services:
  my-db:
    image: mysql
    environment: # 환경변수 설정
      MYSQL_ROOT_PASSWORD: 6316
    volumes:
      # compose 파일을 기준으로 경로 설정이 가능하다.
      # ./mysql_data 이름은 임의로 설정하면 되지만 /var/lib/mysql은 그대로 써야한다.
      - ./mysql_data:/var/lib/mysql
    ports:
      - 3306:3306
```

볼륨 설정을 하였기 때문에 컨테이너를 삭제해도 데이터는 남아있다.

## docker compose를 통해 스프링 부트 실행

```yaml
services:
  my-server:
    # Dockerfile을 통해 빌드한 이미지를 사용하겠다.
    # 경로는 compose.yml 기준으로 Dockerfile이 어디에 있는지 상대경로
    build: .
    ports:
      - 8080:8080
```

- `docker compose up -d --build`
  - 빌드할 때마다 이미지가 바뀐다. (jar 파일이 바뀌기 때문에)
  - `--build`를 통해 이미지를 다시 빌드하고 컴포즈를 띄운다.

---

# (실습) MySQL과 Spring Boot 컨테이너 동시에 띄우기

## application.yml

컨테이너로 띄울 MySQL 정보를 입력한다.

```yaml
spring:
  datasource:
    url: jdbc:mysql://my-db:3306/mydb
    username: root
    password: pwd1234
    driver-class-name: com.mysql.cj.jdbc.Driver
```

- `url: jdbc:mysql://my-db:3306/mydb`
  - **`localhost`가 아닌 `my-db`로 되어 있다.**
  - 만약 `localhost`라고 할 경우 Spring Boot가 존재하는 컨테이너의 3306 포트에는 아무것도 실행되지 않기 때문에 오류가 난다.
  - 따라서 compose.yml에서 정의한 서비스 이름으로 통신해야 한다.
- `driver-class-name: com.mysql.cj.jdbc.Driver`
  - MySQL 8.x 이상 사용 시 적용하는 JDBC 드라이버 클래스명이다.

## Dockerfile 작성

```dockerfile
FROM openjdk:17-jdk

COPY build/libs/*SNAPSHOT.jar app.jar

ENTRYPOINT ["java", "-jar", "/app.jar"]
```

## compose.yml 작성

```yaml
services:
  my-server:
    build: .
    ports:
      - 8080:8080
    depends_on:
      my-db:
        condition: service_healthy

  my-db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: pwd1234
      MYSQL_DATABASE: mydb
    volumes:
      - ./mysql_data:/var/lib/mysql
    ports:
      - 3306:3306
    healthcheck:
      test: [ "CMD", "mysqladmin", "ping" ]
      interval: 5s  # 5초마다
      retries: 10   # 10번까지 재시도
```

- 반드시 `my-db`가 실행된 후 Spring Boot가 실행되어야 제대로 작동한다.
  - `test: [ "CMD", "mysqladmin", "ping" ]`
    - 이 명령어가 정상적으로 실행되면 MySQL이 문제없이 동작한다고 판단한다.
    - MySQL이 제대로 작동하는지 신호를 날리는 것이다.

## 빌드 및 실행

```bash
# 작성한 코드를 빌드한다.
./gradlew clean build

# 작성한 도커 컴포즈 파일을 빌드하고 실행한다.
docker compose up -d --build
```
