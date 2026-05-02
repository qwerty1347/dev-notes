---
tags:
  - fastapi
  - jsonable_encoder
  - serialization
created: 2026-04-18
---

# FastAPI `jsonable_encoder` 역할

`jsonable_encoder` 는 FastAPI 유틸로, Python 객체를 JSON 직렬화 가능한 기본 타입으로 변환한다.
**임의의 Python 객체를 `json.dumps` 가 받을 수 있는 형태(dict/list/str/int/...) 로 변환**

## 왜 필요한가?
기본 `json.dumps` 는 다음 타입을 바로 직렬화할 수 없다:
- `datetime`, `date`, `time`, `timedelta`
- `UUID`, `Decimal`, `Enum`
- Pydantic `BaseModel`
- `bytes`, `set`, `frozenset`

`jsonable_encoder` 는 위 타입들을 JSON 호환 기본 타입으로 바꿔준다.

## 사용 예

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