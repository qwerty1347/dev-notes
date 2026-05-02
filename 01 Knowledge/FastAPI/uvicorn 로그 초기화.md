---
tags:
  - fastapi
  - uvicorn
  - logging
created: 2026-04-18
---

# uvicorn 로그 초기화 문제 (uvicorn 마이그레이션 0.34 → 0.44)

`uvicorn` 버전이 올라가면서 **0.34 → 0.44** `logging.basicConfig()`로 설정한 로그가 출력되지 않는 문제가 발생할 수 있다.

### 원인 — 초기화 순서 변화

| 버전 | 초기화 순서 | 결과 |
| --- | --- | --- |
| **uvicorn 0.34** | ① 앱 import → ② `setup_logging()` 호출 → ③ `basicConfig()` 가 root 로거에 핸들러 추가 → ④ uvicorn이 자기 로거만 설정 | ✅ 정상 출력 |
| **uvicorn 0.44** | ① uvicorn이 먼저 `dictConfig` 로 root 핸들러 부착 → ② 앱 import → ③ `basicConfig()` 가 "root에 이미 핸들러 있음" 으로 판단하고 **무시** | ❌ 로그 누락 |

## 원인
`logging.basicConfig()` 는 root 로거에 이미 핸들러가 있으면 아무 동작도 하지 않는다. uvicorn이 먼저 로깅을 초기화하면, 이후에 호출한 `basicConfig()` 는 무시된다.

### 버전별 초기화 순서
- **uvicorn 0.34**: 앱 import → `basicConfig()` → uvicorn이 자신의 로거 설정 → 정상 출력
- **uvicorn 0.44**: uvicorn이 먼저 `dictConfig`로 초기화 → 앱 import → `basicConfig()` 무시 → 로그 누락

### 해결 — `force=True` 사용

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s - %(message)s",
    force=True,            # ✅ 기존 핸들러가 있어도 덮어쓰기
)
```

- `force=True` 는 **기존 root 핸들러를 모두 제거하고 새로 설정**한다.
- uvicorn 0.34, 0.44 양쪽 모두에서 동일하게 동작하므로 안전.

### 권장 방법: `dictConfig`

운영 환경에서는 `basicConfig` 대신 **`dictConfig`** 로 명시적 설정을 쓰는 쪽이 충돌을 덜 일으킨다.

```python
import logging.config

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "default": {"format": "%(asctime)s %(levelname)s %(name)s - %(message)s"},
    },
    "handlers": {
        "console": {"class": "logging.StreamHandler", "formatter": "default"},
    },
    "root": {"level": "INFO", "handlers": ["console"]},
    "loggers": {
        "uvicorn": {"level": "INFO"},
        "uvicorn.access": {"level": "INFO"},
    },
}

logging.config.dictConfig(LOGGING)
```

## 요약
- 문제는 uvicorn 초기화 순서 차이로 인한 것.
- `force=True` 또는 `dictConfig` 로 로그 설정을 명시적으로 처리하면 해결된다.