# Docker MySQL 비밀번호가 적용되지 않을 때

## 증상

- `docker compose up -d`로 컨테이너는 정상 기동됨
- 로그에 `mysqld: ready for connections.` 출력
- 그런데 `.env`의 자격증명으로 접속하면 거부됨

```
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```

추가로 로그에 다음과 같은 경고가 보일 수 있음:

```
[ERROR] Native table 'performance_schema'.'memory_summary_global_by_event_name' has the wrong structure
...
```

## 원인

MySQL 공식 이미지의 entrypoint는 **데이터 디렉터리(`/var/lib/mysql`)가 비어있을 때만** 다음 환경변수를 적용한다.

- `MYSQL_ROOT_PASSWORD`
- `MYSQL_DATABASE`
- `MYSQL_USER` / `MYSQL_PASSWORD`

데이터가 이미 존재하면 entrypoint는 초기화를 스킵하고 `mysqld`만 그대로 띄운다. 따라서 `docker-compose.yml`을 아무리 수정해도 디스크에 남아있는 옛 자격증명이 그대로 사용된다.

본 프로젝트의 `docker-compose.yml`은 다음과 같이 호스트 디렉터리를 마운트한다.

```yaml
mysql:
  image: mysql:5.7
  volumes:
    - ./.docker/mysql/data:/var/lib/mysql
  environment:
    MYSQL_ROOT_PASSWORD: root
    MYSQL_DATABASE: fastapi
    MYSQL_USER: admin
    MYSQL_PASSWORD: admin
```

`./.docker/mysql/data`에 이전 실행 시점의 초기화 데이터가 남아있으면 위 환경변수는 무시된다. 또한 이전 데이터가 다른 MySQL 버전 또는 손상된 상태에서 만들어졌다면 `performance_schema ... has the wrong structure` 에러가 함께 출력된다.

## 진단 절차

1. 컨테이너가 떠 있는지 확인

   ```bash
   docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
   ```

2. MySQL 자체가 준비되었는지 로그 확인

   ```bash
   docker logs fastapi_ocr-mysql --tail 30
   ```

   `mysqld: ready for connections.`가 보이면 프로세스 자체는 정상.

3. 컨테이너 내부에서 `.env`의 자격증명으로 직접 접속해 본다.

   ```bash
   docker exec fastapi_ocr-mysql mysql -uroot -proot -e "SHOW DATABASES;"
   ```

   여기서 `Access denied`가 나면 비밀번호 미적용이 확정된다.

4. 데이터 디렉터리 상태 확인

   ```bash
   ls -la ./.docker/mysql/data
   ```

   `auto.cnf`, `ibdata1`, `mysql/` 등이 존재하면 이미 초기화된 데이터가 남아있는 것.

## 해결

데이터 디렉터리를 비우고 컨테이너를 재생성하면, MySQL이 깨끗한 상태에서 재초기화되며 환경변수가 그대로 적용된다.

> 주의: 이 작업은 MySQL 데이터를 삭제한다. 보존할 데이터가 있다면 먼저 `mysqldump`로 백업할 것.

```bash
docker compose stop mysql
docker compose rm -f mysql
rm -rf ./.docker/mysql/data
docker compose up -d mysql
```

초기화가 끝날 때까지 대기 후 검증한다.

```bash
# 로그에 "MySQL init process done" 또는 두 번째 "ready for connections"가 찍힐 때까지 기다림
docker logs -f fastapi_ocr-mysql

# 자격증명 검증
docker exec fastapi_ocr-mysql mysql -uroot -proot -e "SHOW DATABASES; SELECT user, host FROM mysql.user;"
```

기대 결과:

- `fastapi` 데이터베이스 존재
- `root@%`, `admin@%` 사용자 등록 (컨테이너 간 접속 허용)
- `performance_schema` 에러가 더 이상 출력되지 않음

## 데이터를 보존해야 하는 경우

데이터 디렉터리를 삭제할 수 없다면 root 비밀번호를 직접 재설정한다.

```bash
docker compose stop mysql

# skip-grant-tables 모드로 임시 기동
docker run --rm -it \
  -v "$(pwd)/.docker/mysql/data:/var/lib/mysql" \
  mysql:5.7 \
  --skip-grant-tables --skip-networking &

# 다른 터미널에서 비밀번호 재설정
docker exec -it <임시컨테이너> mysql -uroot <<'SQL'
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
ALTER USER 'root'@'%' IDENTIFIED BY 'root';
FLUSH PRIVILEGES;
SQL

# 임시 컨테이너 종료 후 정상 기동
docker compose up -d mysql
```

## 핵심 규칙

- **`MYSQL_*` 환경변수는 빈 데이터 디렉터리에서만 적용된다.** 비밀번호를 바꿨는데 안 먹는다면 9할은 이 케이스다.
- `docker-compose.yml`을 수정해도 마운트된 호스트 디렉터리에 데이터가 남아있으면 변경이 반영되지 않는다.
- `Access denied` + `ready for connections`가 동시에 나오면 인증 정보 불일치를 먼저 의심한다.
