# 01. 도커 개념 정리

> 2026-11-04 학습. 실습 전 개념 정리.

## 1. 도커가 해결하는 문제: "내 PC에선 되는데요?"

코드는 같아도 **실행 환경**(JDK 버전, 설정값, 라이브러리, 포트)이 달라서 문제가 생긴다.

- 실제 경험: OpenIDEA로 TcmAdm 실행 시 32비트 구 JDK, `-DlocalConfig` 누락, context path 404, jdtls JDK 25 요구
- 실제 경험: 배치 서버에 jar 하나 누락 → `NoClassDefFoundError`로 조용히 실패

**도커의 아이디어: 프로그램만 보내지 말고, 실행 환경까지 통째로 상자에 담아서 보낸다.**
같은 상자를 열면 회사 PC, 맥북, 운영 서버 어디서든 똑같이 동작한다.

## 2. 이미지와 컨테이너

| 도커 | Java 비유 | 설명 |
|---|---|---|
| 이미지 (Image) | 클래스 | 실행 환경의 설계도이자 완성된 상자. **읽기 전용** |
| 컨테이너 (Container) | 객체 (인스턴스) | 이미지를 실제로 실행한 것. **같은 이미지로 여러 개** 만들 수 있음 |

```
MySQL 이미지 (mysql:8.4)          ← 클래스
   ├── 컨테이너 A: 학습용 DB       ← new MySQL()
   ├── 컨테이너 B: 테스트용 DB
   └── 컨테이너 C: 실험용 DB
```

- 컨테이너는 지워도 이미지는 남는다 → 다시 띄우면 깨끗한 초기 상태 (공부할 때 DB를 망가뜨려도 10초면 복구)

## 3. 레지스트리 (Docker Hub)

- 이미지 저장소. GitHub = 코드 저장소, Docker Hub = 이미지 저장소
- MySQL, Redis, Kafka는 공식 이미지가 있음: `docker pull mysql:8.4`
- `:8.4`는 태그(버전). 버전을 고정하면 언제 받아도 같은 환경 (회사 TcmApi의 Artemis 2.18.0 고정과 같은 이유)

## 4. Dockerfile: 내 앱의 이미지 레시피

```dockerfile
FROM eclipse-temurin:21-jre             # Java 21 실행 환경 이미지에서 시작
WORKDIR /app                            # 상자 안 작업 폴더
COPY build/libs/app.jar app.jar         # 빌드한 jar 복사
EXPOSE 8080                             # 8080 포트 사용 표시
ENTRYPOINT ["java", "-jar", "app.jar"]  # 컨테이너 시작 시 실행할 명령
```

- 손으로 맞추던 실행 환경 설정이 코드로 남는다
- **레이어**: 이미지는 각 줄이 층층이 쌓인 구조. jar만 바뀌면 JDK 층은 재사용 → 빌드가 빠름

## 5. 컨테이너 연결 4가지

컨테이너는 격리된 공간이라, 밖과 연결하려면 일부러 통로를 만들어야 한다.

### ① 포트 매핑 `-p 바깥:안`
```
docker run -p 3307:3306 mysql:8.4
```
- 내 PC `localhost:3307` → 컨테이너 안 `3306`
- 바깥 번호만 바꾸면 같은 프로그램을 여러 개 띄우거나 포트 충돌을 피할 수 있음

### ② 볼륨 `-v`: 컨테이너를 지워도 남는 데이터
- 컨테이너를 지우면 안의 데이터도 사라짐 → DB 데이터는 볼륨에 보관
- 비유: 컨테이너 = 호텔 방, 볼륨 = 개인 금고 (체크아웃해도 금고 물건은 남음)
- **이름 있는 볼륨**: `mysql-data:/var/lib/mysql` (도커가 관리, DB 데이터용)
- **바인드 마운트**: `./init.sql:/docker-entrypoint-initdb.d/init.sql` (내 PC 파일·폴더를 그대로 연결, 설정·초기 SQL용)

### ③ 환경 변수 `-e`: 실행할 때 넘기는 설정값
```
docker run -e MYSQL_ROOT_PASSWORD=비밀번호 -e MYSQL_DATABASE=commerce mysql:8.4
```
- 이미지는 그대로, 설정만 바꿔 실행 (local/dev/prod 설정 분리와 같은 개념)

### ④ 네트워크: 컨테이너끼리 대화
- 같은 네트워크면 **컨테이너 이름이 주소**: `jdbc:mysql://mysql:3306/commerce`

## 6. Docker Compose: 여러 컨테이너를 한 번에

```yaml
services:
  mysql:
    image: mysql:8.4
    ports: ["3307:3306"]
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: commerce
    volumes: [mysql-data:/var/lib/mysql]

  redis:
    image: redis:7
    ports: ["6379:6379"]

volumes:
  mysql-data:
```

```
docker compose up -d     # 전부 실행
docker compose down      # 전부 종료
```

## 7. 가상머신(VM)과의 차이 (면접 단골)

| | 가상머신 | 도커 컨테이너 |
|---|---|---|
| 구조 | OS를 통째로 하나 더 설치 | 호스트 OS의 **커널을 공유**, 필요한 부분만 격리 |
| 크기 | 수 GB | 수십~수백 MB |
| 시작 시간 | 분 단위 | 초 단위 |
| 비유 | 단독주택 한 채 더 짓기 | 아파트 한 호실 (건물은 공유, 집 안은 따로) |

- 핵심 답: **"커널을 공유하느냐"** → 그래서 가볍고 빠르다
- 맥·Windows는 리눅스가 아니라서 Docker Desktop이 작은 리눅스 VM 위에서 컨테이너를 돌린다 (Windows는 WSL)

## 8. 자주 쓰는 명령어

| 명령 | 뜻 |
|---|---|
| `docker pull mysql:8.4` | 이미지 받기 |
| `docker images` | 받아 둔 이미지 목록 |
| `docker run ...` | 컨테이너 만들어 실행 |
| `docker ps` / `docker ps -a` | 실행 중 / 전체 컨테이너 |
| `docker logs -f 이름` | 로그 보기 (catalina.out 보듯이) |
| `docker exec -it 이름 bash` | 컨테이너 안으로 들어가기 (SSH 접속하듯이) |
| `docker stop` / `docker rm` | 멈추기 / 지우기 |
| `docker compose up -d` / `down` | 묶음 전체 실행 / 종료 |

## 한 장 요약

```
Dockerfile ──build──▶ 이미지 ──run──▶ 컨테이너
(레시피)               (상자, 클래스)    (실행 중, 객체)
                          ▲
              Docker Hub에서 pull (mysql, redis …)

연결 4가지: 포트(-p) · 볼륨(-v) · 환경변수(-e) · 네트워크
여러 개 묶기: docker-compose.yml → docker compose up
```

## 확인 질문

1. 이미지와 컨테이너의 차이를 Java에 빗대면?
   → 이미지는 클래스(읽기 전용 설계도), 컨테이너는 객체(실행 중인 인스턴스, 여러 개 가능)
2. MySQL 컨테이너를 지웠다 다시 띄워도 데이터가 남으려면?
   → 볼륨 (DB 데이터는 이름 있는 볼륨)
3. `-p 3307:3306`일 때 내 PC의 DB 툴로 접속할 포트는?
   → 3307 (바깥 3307 → 안 3306)
