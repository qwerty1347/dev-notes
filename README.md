---
tags:
  - index
  - moc
updated: 2026-07-07
---

# dev-notes

개발하며 얻은 지식과 로그를 모으는 저장소.
최상위는 **주제(도메인) 폴더**로 나누고, 시간순 기록만 `DevLog/`로 분리한다.

## 구조

| 폴더 | 축 | 범위 | 향후 들어올 것 |
| --- | --- | --- | --- |
| `DevLog/` | 시간 | 날짜별 개발 로그·트러블슈팅 (`YYYY-MM-DD 제목.md`) | 인시던트, 삽질 기록, 결정 로그 |
| `Web/` | 주제 | 웹/API 앱 런타임 (FastAPI: 요청·응답, 업로드, 폼, Pydantic, uvicorn) | Django/Flask, 미들웨어, 인증 라우팅 |
| `Database/` | 주제 | ORM/ODM 쿼리·모델링 (SQLAlchemy, MongoDB) | Alembic 마이그레이션, 인덱싱, Redis 캐시 |
| `Async/` | 주제 | 비동기 작업 큐·백그라운드 잡 | Celery/arq 운영, 스케줄, SAQ |
| `Infra/` | 주제 | 컨테이너·빌드·배포·로컬 개발환경 | Dockerfile 패턴, compose, CI, nginx |
| `Security/` | 주제 | 인증·봇 차단·시크릿 | JWT/OAuth, CORS, CSRF, rate limiting |

## 노트 목록

### DevLog
- [[2026-04-15 Docker vs Local venv 구조 차이 (uv sync 오류)]] — venv 격리 패턴을 도입한 인시던트

### Web
- [[jsonable_encoder (JSON 직렬화)]] — Python 객체 → JSON 직렬화 유틸
- [[Pydantic 검증 (model_validate)]] — 라우트 밖에서 dict/JSON 검증
- [[Pydantic 완전 가이드]] — 모델 생성·검증·변환 종합 가이드
- [[Annotated 의존성 주입 마이그레이션]] — `Depends`를 `Annotated`로 전환
- [[Annotated 사용처 정리]] — `Annotated` 메타데이터 패턴 개관
- [[FastAPI 파라미터 변수명 컨벤션]] — 경로/쿼리/바디 파라미터 네이밍
- [[multipart 파일·폼 처리]] — 파일+폼 동시 요청, `Form()`·`as_form`
- [[UploadFile vs bytes]] — 업로드 타입 선택
- [[UploadFile 스트림 소비]] — `read()` 2회 호출 시 빈 바이트
- [[uvicorn 로깅 초기화]] — 0.34→0.44 로그 누락, `force=True`/`dictConfig`
- [[Tistory API 오류 해결 정리]] — 외부 Tistory API 연동 오류 모음

### Database
- [[SQLAlchemy 쿼리 가이드 (Laravel 비교)]] — Eloquent ↔ SQLAlchemy 2.x (async/sync) 대응표
- [[MongoDB 쿼리 가이드 (Beanie·Motor)]] — Eloquent ↔ Beanie/Motor 대응표
- [[간단한 CRUD 예제 (라우터+Repository)]] — 라우터 + Repository 계층 CRUD
- [[관계(조인) CRUD 예제]] — 2개 테이블 조인·다컬럼 CRUD
- [[Elasticsearch 쿼리 가이드 (auth_apikey)]] — 매핑·Query DSL·PHP 클라이언트 사용 정리

### Async
- [[Celery vs arq]] — Redis 기반 큐 구성·사용법 비교

### Infra
- [[Docker Named Volume으로 venv 격리]] — bind mount 환경에서 `.venv` 분리
- [[Docker MySQL 비밀번호 미적용 문제]] — MySQL 컨테이너 env 비번이 안 먹을 때

### Security
- [[Cloudflare Turnstile·hCaptcha 통합]] — 발급 → 프론트 삽입 → 서버 검증

## 분류 규칙 (신규 노트 추가 시)

1. **주제 우선** — "FastAPI에서 썼다"가 아니라 "DB 쿼리 문제냐, 배포 문제냐"로 폴더 결정.
   예) SQLAlchemy 쿼리는 FastAPI에서 써도 `Database/`.
2. **날짜가 핵심인 기록은 `DevLog/`** — 재사용 지식은 주제 폴더, "그때 무슨 일이 있었나"는 DevLog.
3. **파일명은 주제를 서술** — slug(`celery-vs-arq`)나 `(예제)` 같은 접미사 대신 읽히는 제목.
4. **폴더당 1개여도 도메인 폴더 유지** — 같은 도메인 노트가 모일 자리.
5. **링크는 `[[노트 이름]]`** (Obsidian) — 폴더를 옮겨도 이름만 같으면 링크는 유지된다.
