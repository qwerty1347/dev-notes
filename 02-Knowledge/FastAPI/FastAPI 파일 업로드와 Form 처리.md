---
tags:
  - fastapi
  - python
  - uvicorn
  - multipart
  - logging
created: 2026-04-17
---

# FastAPI 파일 업로드와 Form 처리

> FastAPI로 파일 업로드 + 폼 데이터 처리, Pydantic 검증, 스트림 소비, uvicorn 로깅 초기화 순서, `jsonable_encoder` 까지 실무에서 자주 마주치는 이슈를 정리.

---

## 1. 폼 데이터에 파일이 포함된 경우 `UploadFile` 타입 명시

파일 업로드는 HTTP `Content-Type: multipart/form-data` 로 전송된다. FastAPI에서 파일을 받으려면 반드시 **`UploadFile`** (또는 `bytes`) 타입으로 매개변수를 선언해야 한다.

```python
from fastapi import FastAPI, UploadFile, File

app = FastAPI()

@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    contents = await file.read()
    return {"filename": file.filename, "size": len(contents)}
```

### `UploadFile` vs `bytes`
| 구분 | 설명 |
| --- | --- |
| `UploadFile` | 스트리밍(디스크 임시 저장) 방식. 대용량에 유리. `await file.read()`, `file.filename`, `file.content_type` 등 제공 |
| `bytes` | 메모리에 전부 로드. 작은 파일에만 사용 |

---

## 2. 파일 + 추가 필드가 함께 오면 `Form` / `Depends(as_form)` 필요

파일과 다른 필드(`title`, `user_id` 등)를 같은 요청에 담으면 **전체가 `multipart/form-data`** 가 된다. 이때 JSON body를 기대하는 Pydantic 모델을 그대로 주입하면 **422 Unprocessable Entity** 가 발생한다.

### ❌ 잘못된 예 (422 발생)

```python
from pydantic import BaseModel

class OcrRequest(BaseModel):
    lang: str
    engine: str

@app.post("/ocr")
async def ocr(
    file: UploadFile = File(...),
    ocr_dto: OcrRequest,   # ❌ FastAPI는 JSON body를 기대 → multipart와 충돌
):
    ...
```

**원인**: FastAPI는 Pydantic 모델 파라미터를 JSON body로 해석한다. 그러나 요청은 multipart이므로 JSON 파싱이 불가능하고, 유효성 검증 단계에서 422를 반환한다.

### ✅ 해결 방법 1 — `Form()` 으로 필드 분리

```python
from fastapi import Form

@app.post("/ocr")
async def ocr(
    file: UploadFile = File(...),
    lang: str = Form(...),
    engine: str = Form(...),
):
    ...
```

### ✅ 해결 방법 2 — `as_form` 패턴 (Pydantic 모델 재사용)

필드가 많아지면 모델을 `Form` 주입 가능하도록 변환하는 헬퍼를 만든다.

```python
import inspect
from fastapi import Form, Depends
from pydantic import BaseModel

def as_form(cls: type[BaseModel]):
    new_params = [
        inspect.Parameter(
            name=field_name,
            kind=inspect.Parameter.POSITIONAL_OR_KEYWORD,
            default=Form(... if field.is_required() else field.default),
            annotation=field.annotation,
        )
        for field_name, field in cls.model_fields.items()
    ]

    async def _as_form(**data):
        return cls(**data)

    _as_form.__signature__ = inspect.Signature(new_params)
    setattr(cls, "as_form", _as_form)
    return cls

@as_form
class OcrRequest(BaseModel):
    lang: str
    engine: str

@app.post("/ocr")
async def ocr(
    file: UploadFile = File(...),
    ocr_dto: OcrRequest = Depends(OcrRequest.as_form),
):
    ...
```

> **요약**: multipart에서는 JSON body를 못 쓴다 → 각 필드를 `Form`으로 쪼개거나, `as_form` 패턴으로 모델을 감싸야 한다.

---

## 3. `validate_model` 을 쓰는 이유 / 안 쓰는 경우

Pydantic v1의 `validate_model`(또는 v2의 `model_validate`) 은 **모델 인스턴스화 과정에서 수동으로 검증을 수행**할 때 사용한다.

### 쓰는 경우
- 라우트 바깥(서비스 계층, 배치 스크립트)에서 dict → 모델로 변환하며 검증하고 싶을 때
- 부분 업데이트 시 일부 필드만 검증하고 싶을 때
- 외부 시스템에서 받은 raw dict(Kafka, Redis, 외부 API 응답 등)를 신뢰 경계 안쪽으로 들일 때

```python
# Pydantic v2
user = User.model_validate(raw_dict)             # 검증 + 인스턴스 생성
user = User.model_validate_json(raw_json_bytes)  # JSON 문자열에서 바로
```

### 안 쓰는 경우
- 이미 FastAPI 라우트 파라미터로 Pydantic 모델을 선언했다면 **FastAPI가 자동 검증**하므로 따로 호출할 필요 없음
- 단순히 타입 힌트만 필요할 때 (`TypedDict`, `dataclass` 로 충분)

> **핵심**: 라우트 안에서는 자동 검증이 동작하니 중복 호출 금지. 바깥에서 dict를 신뢰할 수 없을 때만 명시적으로 검증.

---

## 4. `file.read()` 를 두 번 호출하면 빈 `b''` 반환

`UploadFile` 내부는 **스트림(SpooledTemporaryFile)** 이다. 한 번 읽으면 커서가 끝으로 이동해 두 번째 `read()` 는 빈 바이트를 반환한다.

```python
@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    data1 = await file.read()   # 실제 바이트
    data2 = await file.read()   # b''  ← 스트림이 소비됨
```

### 해결 방법

1. **읽은 데이터를 변수에 저장해 재사용**

    ```python
    data = await file.read()
    hash_ = hashlib.sha256(data).hexdigest()
    save_to_s3(data)
    ```

2. **스트림 커서를 처음으로 되돌리기** (`seek(0)`)

    ```python
    data = await file.read()
    await file.seek(0)          # 다시 읽을 수 있도록 커서 리셋
    data_again = await file.read()
    ```

3. **여러 소비자에게 같은 파일을 넘길 때는 `BytesIO` 로 래핑**

    ```python
    from io import BytesIO
    buf = BytesIO(await file.read())
    pillow_image = Image.open(buf); buf.seek(0)
    ocr_result   = do_ocr(buf);     buf.seek(0)
    ```

> 스트림은 "한 번 읽으면 끝" — 재사용하려면 `seek(0)` 하거나 메모리에 담아 두자.

---

## 5. uv 마이그레이션 후 로그가 안 찍히는 이슈 (uvicorn 0.34 → 0.44)

### 증상
- `uv` 로 마이그레이션하면서 uvicorn 버전이 **0.34 → 0.44** 로 올라간 뒤 `logging.basicConfig(...)` 로 설정한 로그가 **하나도 출력되지 않음**

### 원인 — 초기화 순서 변화

| 버전 | 초기화 순서 | 결과 |
| --- | --- | --- |
| **uvicorn 0.34** | ① 앱 import → ② `setup_logging()` 호출 → ③ `basicConfig()` 가 root 로거에 핸들러 추가 → ④ uvicorn이 자기 로거만 설정 | ✅ 정상 출력 |
| **uvicorn 0.44** | ① uvicorn이 먼저 `dictConfig` 로 root 핸들러 부착 → ② 앱 import → ③ `basicConfig()` 가 "root에 이미 핸들러 있음" 으로 판단하고 **무시** | ❌ 로그 누락 |

`logging.basicConfig()` 의 기본 동작이 "root 로거에 핸들러가 이미 붙어 있으면 아무것도 안 함" 이기 때문에 발생한다.

### 해결 — `force=True`

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

### 근본적으로 권장되는 방식

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

> **정리**: 코드 문제가 아니라 uvicorn의 **초기화 순서 변경**이 원인. `force=True` 또는 `dictConfig` 로 해결.

---

## 6. `jsonable_encoder` 의 역할 / 사용 이유

`jsonable_encoder` 는 FastAPI 제공 유틸로, **임의의 Python 객체를 `json.dumps` 가 받을 수 있는 형태(dict/list/str/int/...) 로 변환**한다.

```python
from fastapi.encoders import jsonable_encoder
from datetime import datetime
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    created_at: datetime

item = Item(name="pen", created_at=datetime.now())

data = jsonable_encoder(item)
# {"name": "pen", "created_at": "2026-04-17T12:34:56.789123"}
```

### 왜 필요한가?

Python의 기본 `json.dumps` 는 다음 타입을 **직접 직렬화하지 못한다**:
- `datetime`, `date`, `time`, `timedelta`
- `UUID`, `Decimal`, `Enum`
- Pydantic `BaseModel`
- `bytes`, `set`, `frozenset`

`jsonable_encoder` 는 이 타입들을 JSON 호환 기본 타입으로 변환해 준다.

### 사용 시나리오

1. **응답을 JSON으로 직접 반환하기 전에 변환**

    ```python
    @app.get("/item")
    def get_item():
        item = fetch_item()
        json_compat = jsonable_encoder(item)
        return JSONResponse(content=json_compat)
    ```

2. **MongoDB 등 NoSQL에 저장 전에 Pydantic 모델 변환**

    ```python
    doc = jsonable_encoder(user_model)
    await collection.insert_one(doc)
    ```

3. **캐시(Redis) 에 넣기 전 직렬화 가능한 dict 로 변환**

### `model_dump()` 와의 차이

| 메서드 | 범위 | JSON 호환 변환 |
| --- | --- | --- |
| `model.model_dump()` | Pydantic 모델만 | 일부 타입(`datetime` 등)은 그대로 객체로 남음 |
| `model.model_dump(mode="json")` | Pydantic 모델만 | **JSON 호환으로 변환** (`datetime` → ISO 문자열) |
| `jsonable_encoder(obj)` | **임의 객체** (모델, dict, list, dataclass 등) | 재귀적으로 JSON 호환으로 변환 |

> 일반적으로 라우트에서 **모델을 그대로 반환**하면 FastAPI가 내부적으로 직렬화하므로 수동으로 호출할 필요가 없다. 하지만 **`JSONResponse` 를 직접 만들거나, 외부 저장소로 넘길 때**는 `jsonable_encoder` 가 유용하다.

---

## 참고

- FastAPI 공식 문서: <https://fastapi.tiangolo.com/tutorial/request-forms-and-files/>
- Pydantic v2 마이그레이션: <https://docs.pydantic.dev/latest/migration/>
- uvicorn CHANGELOG: <https://www.uvicorn.org/release-notes/>

## 관련 노트

- [[2026-04-15 Docker vs Local venv 구조 차이 (uv sync 오류)]]
- [[Docker Named Volume으로 venv 격리]]
