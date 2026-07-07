---
tags:
  - docker
  - docker-compose
  - python
  - venv
  - uv
  - vscode
created: 2026-04-17
---

# Docker Named Volume으로 venv 격리

> 로컬 코드를 컨테이너에 bind mount 하는 환경에서 컨테이너 내부의 `.venv`(Linux 구조) 가 호스트(Windows) 에 새어 나와 VS Code가 Python 인터프리터를 못 잡는 문제. **Named Volume 을 하위 경로에 덮어씌워 `.venv` 만 격리**하는 표준 패턴으로 해결한다.

---

## 1. 문제 상황

### 단순 bind mount 구성

```yaml
services:
  app:
    build: .
    volumes:
      - ./:/app          # 로컬 폴더 전체를 컨테이너 /app에 덮어씌움
```

이 구조의 의미:

> 👉 **로컬 폴더 전체를 컨테이너 `/app` 에 그대로 덮어씌움** (양방향 동기화)

### 발생한 문제 흐름

1. 컨테이너 안에서 `uv venv` / `python -m venv` 실행 → `.venv` 생성
2. `./:/app` bind mount이므로 **컨테이너의 `.venv` 가 그대로 호스트에도 나타남**
3. 호스트의 VS Code 가 이 `.venv` 를 발견:
    > “아 이거 로컬 venv네?” 라고 착각
4. 그런데 컨테이너는 Linux → `.venv/bin/` 구조 (Windows의 `.venv/Scripts/` 없음)
5. VS Code가 Windows 기준 인터프리터 경로를 찾지 못해 **에러 발생**

| OS | venv 실행파일 경로 |
| --- | --- |
| Windows | `.venv/Scripts/python.exe` |
| Linux / macOS | `.venv/bin/python` |

`pyvenv.cfg` 를 열면 `home = /usr/local/bin` 처럼 **컨테이너 내부 경로**가 박혀 있어서 호스트에서는 애초에 사용 불가능한 상태다.

---

## 2. 해결 방식 — Named Volume을 `.venv` 경로에 덮어씌우기

### 적용한 `docker-compose.yml`

```yaml
services:
  app:
    build: .
    volumes:
      - ./:/app                  # 전체 코드 공유
      - venv_data:/app/.venv     # 🔥 .venv만 Named Volume으로 따로 격리

volumes:
  venv_data:
```

### 핵심 포인트

> 🔥 **`/app/.venv` 만 따로 격리** — 나머지 코드는 bind mount로 공유한 채, `.venv` 경로만 Docker가 관리하는 별도 저장소로 덮어쓴다.

---

## 3. 동작 원리 — "더 구체적인 경로가 우선"

Docker는 **mount 경로가 겹칠 경우, 더 하위(구체적) 경로의 mount가 우선 적용**된다.

```
./     → /app        (전체 공유, bind mount)
         /app/.venv  (따로 덮어쓰기, named volume)  ← 더 구체적
```

### 결과 매핑

| 컨테이너 경로 | 실제 저장 위치 |
| --- | --- |
| `/app` | 로컬 파일 (호스트의 `./`) |
| `/app/.venv` | Docker 관리 볼륨 (`venv_data`) |

즉, 컨테이너에서 `.venv` 에 무언가 쓰면 **호스트에는 전혀 나타나지 않고**, Docker가 관리하는 별도 저장소로 간다.

---

## 4. 이 구조의 이점

### ✅ 1. 로컬에 `.venv` 가 생기지 않음
- 호스트 프로젝트 디렉터리가 깔끔하게 유지된다.
- 백업, Git 관리, 파일 탐색기 성능 모두 개선.

### ✅ 2. VS Code 가 혼동하지 않음
- 호스트에 Linux 구조 venv이 보이지 않으므로 인터프리터 자동 감지가 꼬이지 않음.
- 로컬 개발은 별도로 `uv venv` 를 만들어 VS Code 에 연결.

### ✅ 3. 컨테이너는 정상적으로 venv 사용
- 컨테이너 재시작해도 `venv_data` 볼륨은 유지되므로 **매번 `uv sync` 를 다시 돌릴 필요 없음**.
- 의존성 설치 속도와 캐시 이점까지 얻는다.

---

## 5. 실무에서 표준 패턴인 이유

이 구조는 Python 프로젝트의 Docker 개발환경에서 **거의 표준 관용구**로 쓰인다.

> 👉 **"코드는 공유, 환경은 분리"**

| 대상 | 마운트 방식 | 이유 |
| --- | --- | --- |
| 코드 (`./`) | bind mount | 호스트 에디터에서 수정 → 컨테이너 즉시 반영 (hot reload) |
| 의존성 (`.venv`) | named volume | 컨테이너 OS 환경에 맞는 바이너리이므로 호스트와 분리 필요 |

### 비유

```
/app       = 프로젝트 코드    (공용 작업 공간)
.venv      = 런타임 환경      (컨테이너 전용 도구상자)
```

> 환경까지 공유하면 OS 차이, 아키텍처 차이(amd64 vs arm64), Python 버전 차이로 **반드시 충돌이 난다**.

### 같은 패턴이 적용되는 다른 경로들

동일한 "하위 경로 볼륨 격리" 는 다른 생태계에서도 자주 등장한다.

| 생태계 | 격리 대상 경로 | 이유 |
| --- | --- | --- |
| Node.js | `/app/node_modules` | 네이티브 애드온이 OS별로 다름 |
| Python | `/app/.venv`, `/app/__pycache__` | 바이너리/컴파일 결과 |
| Rust | `/app/target` | 대용량 빌드 산출물 |
| Go | `/go/pkg`, `/app/vendor` | 모듈 캐시 |
| Java | `/root/.m2`, `/app/build` | 의존성 캐시, 빌드 산출물 |

---

## 6. 주의사항 / 함정

### ① 볼륨 초기화 시점

Named volume 을 **처음 마운트할 때** 는 컨테이너 이미지 안의 `/app/.venv` 내용이 볼륨으로 복사된다. 이후부터는 볼륨이 "진실의 원천" 이 된다.

→ 그래서 Dockerfile 에서 미리 `uv sync` 로 `.venv` 를 만들어 두면, 첫 `up` 시 그 내용이 볼륨으로 올라간다.

### ② 의존성 추가 후 반영

`pyproject.toml` / `requirements.txt` 를 변경해도 볼륨은 자동 갱신되지 않는다.

```bash
# 컨테이너 안에서 재동기화
docker compose exec app uv sync

# 또는 볼륨을 날리고 재생성
docker compose down -v
docker compose up --build
```

### ③ `.dockerignore` 에 `.venv` 추가 권장

```gitignore
# .dockerignore
.venv/
__pycache__/
*.pyc
```

빌드 컨텍스트에 호스트의 `.venv`(있다면) 가 들어가는 것을 막아 이미지 빌드 속도와 정합성을 유지.

### ④ 로컬 개발용 별도 venv

호스트에서도 타입 체크나 테스트를 돌리고 싶다면 **로컬용 venv 을 따로** 만든다.

```bash
uv venv --python 3.12
uv sync
```

그리고 VS Code 인터프리터는 이 로컬 venv 을 선택하도록 `.vscode/settings.json` 에 고정:

```json
{
  "python.defaultInterpreterPath": ".venv/Scripts/python.exe"
}
```

---

## 7. 한 줄 핵심

> 👉 **`.venv` 를 로컬(bind mount) 이 아니라 Docker 전용 Named Volume 으로 분리해서 OS 구조 차이로 인한 충돌을 원천 차단한 구조.**

---

## 관련 노트

- [[2026-04-15 Docker vs Local venv 구조 차이 (uv sync 오류)]] — 본 패턴을 적용하게 된 원인 인시던트
- [[multipart 파일·폼 처리]] — 동일 프로젝트의 FastAPI 레이어

## 참고

- Docker 공식 문서 — Volumes: <https://docs.docker.com/storage/volumes/>
- Compose 파일 사양 — `volumes`: <https://docs.docker.com/compose/compose-file/07-volumes/>
