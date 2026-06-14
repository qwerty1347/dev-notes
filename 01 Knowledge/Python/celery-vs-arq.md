# Celery vs arq (Redis 구성 비교)

기존 프로젝트는 Celery + Redis + Flower 구성 사용. `fastapi-tistory` 에 arq 도입 검토 시 비교.

## 비교 표

| 항목 | Celery (fastapi-ocr) | arq (fastapi-tistory 검토안) |
|---|---|---|
| redis 서비스 블록 | 동일 (그냥 redis 컨테이너) | **동일** |
| 접속 설정 | `CELERY_BROKER_URL` (URL) | `REDIS_URL` 또는 host/port — URL 방식 동일 가능 |
| result backend | **별도 필요** → `CELERY_RESULT_BACKEND` (DB 인덱스 분리, `/1`) | **불필요** — 결과도 같은 Redis 저장, 별도 변수 없음 |
| 워커 실행 명령 | `celery -A app worker` | `arq app.worker.WorkerSettings` |
| 모니터링 | Flower 서비스 (`flower`, 5555) | **없음** — 로그/`arq --watch`, 별도 대시보드 사용 |

## 핵심 요약

- **Redis 설정 방식은 다르지 않음.** URL 문자열로 접속하는 건 동일.
- **arq** 사용 시 fastapi-ocr의 `celery` + `flower` 두 서비스가 `worker` 하나로, env는 `CELERY_BROKER_URL`/`CELERY_RESULT_BACKEND` 두 개 → `REDIS_URL` 한 개로 축소.
 cf) redis **볼륨을 지정해야** 재시작 시 큐 유실되지 않음.

<br>

## Celery vs arq (사용법 비교)

개념은 동일: **함수 정의 → 큐에 넣기 → 워커가 꺼내 실행**. (arq는 의도적으로 더 미니멀)

### 작업 정의 / enqueue

```python
# Celery — 데코레이터로 app에 등록
@app.task
def publish(post_id): ...
publish.delay(post_id)

# arq — 평범한 async 함수, WorkerSettings.functions 에 나열
async def publish(ctx, post_id): ...        # 첫 인자 ctx 고정

class WorkerSettings:
    functions = [publish]                    # 여기 없으면 실행 불가
    redis_settings = RedisSettings.from_dsn(REDIS_URL)

redis = await create_pool(RedisSettings.from_dsn(REDIS_URL))
await redis.enqueue_job("publish", post_id)  # 함수 "이름 문자열" 로 enqueue
```

### 항목별 비교

| 항목 | Celery | arq |
|---|---|---|
| 함수 등록 | `@app.task` 등록된 것만 | `WorkerSettings.functions` 나열된 것만 — 워커가 모르는 함수명 enqueue 시 실패 |
| 큐 개수 | 다중 큐 + 라우팅/익스체인지로 동적 분배 | 큐 = Redis 키 1개. **워커 1개 = 큐 1개**(`queue_name`). 큐 나누려면 워커 별도 기동 |
| 라우팅 규칙 | `task_routes` 등 풍부 | 없음. `enqueue_job(..., _queue_name="x")` 수준 |
| 지연 실행 | `apply_async(countdown=, eta=)` | `enqueue_job(..., _defer_by=, _defer_until=)` |
| 주기 실행 | Celery Beat **별도 프로세스** | `WorkerSettings.cron_jobs` **내장** (별도 프로세스 불필요) |
| 재시도 | `autoretry_for`, `self.retry()` | `raise Retry(defer=...)`, `max_tries` |
| 결과 | result backend 별도 구성 | 같은 Redis 자동 저장, `await job.result()` |
| 중복 방지 | 직접 구현 | `_job_id` 지정 시 동일 잡 중복 차단 (내장) |
| 동시성 | prefork(프로세스)/gevent/스레드 | 단일 프로세스 asyncio, `max_jobs` 코루틴 동시 |

**요점**
- arq는 `WorkerSettings.functions` 에 **미리 등록한 함수만** 실행 가능, enqueue는 함수명(문자열)으로.
- 큐 라우팅은 Celery가 강력. arq는 "워커 1 = 큐 1" 단순 모델 → 큐 분리 시 워커 서비스 추가.
- 주기 실행(cron)은 arq가 더 간단(Beat 불필요) → tistory 정기 발행에 유리.
- 본 프로젝트 규모(발행 잡 1~2종 + 정기 실행)면 arq 단일 큐로 충분, 큐 라우팅 필요성 거의 없음.

## 참고) arg 모니터링

arq는 **Flower 같은 공식 웹 UI 없음** (Celery 대비 약점).

| 방법 | 설명 | 비고 |
|---|---|---|
| arq CLI / 로그 | `arq --check` 헬스체크, 잡 lifecycle 로그 | 기본 제공, UI 아님 |
| Python API 조회 | `ArqRedis` 로 `queued_jobs()`, `job.status()`, `job.result()` | 직접 엔드포인트/스크립트 작성 |
| arq-dashboard / arqmon | 커뮤니티 웹 대시보드 | **서드파티·비공식**, 성숙도/유지보수 확인 필요 |
| Redis GUI | RedisInsight, redis-commander | 범용, 잡 전용 뷰 아님 |
| 관측성 연동 | 로그→Sentry, 메트릭→Prometheus 커스텀 | 운영 환경용, 별도 구성 |

- 모니터링 UI가 필수면 → Flower 있는 Celery, 또는 **내장 웹 UI 있는 SAQ**(arq 유사 async 큐, 별개 라이브러리)도 후보.
- 현 발행 파이프라인 규모면 전용 UI 없이 **로그 + 발행 이력 DB 기록**으로 충분할 수 있음.
