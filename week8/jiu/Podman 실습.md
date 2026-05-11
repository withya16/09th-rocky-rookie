## 이미지 검색

```bash
podman search nginx
```

## 이미지 다운로드

```bash
podman pull nginx
```

## 이미지 목록 확인

```bash
podman images
```

## 컨테이너 실행

```bash
podman run -d --name web -p 8080:80 nginx
```

| 옵션 | 의미 |
| --- | --- |
| `-d` | 백그라운드 실행 |
| `--name web` | 컨테이너 이름 지정 |
| `-p 8080:80` | 호스트 8080 포트를 컨테이너 80 포트에 연결 |
| `nginx` | 실행할 이미지 |

## 실행 중인 컨테이너 확인

```bash
podman ps
```

## 전체 컨테이너 확인

```bash
podman ps -a
```

## 컨테이너 내부 접속

```bash
podman exec -it web /bin/bash
```

## 로그 확인

```bash
podman logs web
```

## 컨테이너 중지

```bash
podman stop web
```

## 컨테이너 삭제

```bash
podman rm web
```

## 이미지 삭제

```bash
podman rmi nginx
```

---

# 포트 매핑

## 목적

- 컨테이너 내부 포트를 호스트에서 접근 가능하게 연결
- 외부 요청을 컨테이너 애플리케이션으로 전달

## 예시

```bash
podman run -d --name web -p 8080:80 nginx
```

## 구조

```
Host:8080
   ↓
Container:80
```

## 확인

```bash
curl localhost:8080
```

![](./source/1.png)

---

# 볼륨 연결

## 목적

- 컨테이너 데이터를 호스트에 저장
- 컨테이너 삭제 후에도 데이터 유지
- 설정 파일 또는 정적 파일을 호스트에서 관리

## 예시

```bash
mkdir -p ~/nginx-html
echo "hello podman" > ~/nginx-html/index.html

podman run -d \
  --name web \
  -p 8080:80 \
  -v ~/nginx-html:/usr/share/nginx/html:Z \
  nginx
```

![](./source/2.png)

## 구조

```
Host: ~/nginx-html
   ↓
Container: /usr/share/nginx/html
```

## `:Z` 옵션

- SELinux 환경에서 사용
- 컨테이너가 해당 호스트 디렉터리에 접근할 수 있도록 라벨 조정
- RHEL 계열에서 볼륨 마운트 시 자주 사용
