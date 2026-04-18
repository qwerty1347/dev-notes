---
tags:
  - fastapi
  - multipart
  - form
  - pydantic
created: 2026-04-18
---

# FastAPI multipart 파일 + 폼 처리

파일과 텍스트 필드가 함께 오는 요청은 `multipart/form-data` 로 전송된다. 이때 FastAPI는 Pydantic 모델 파라미터를 JSON body로 해석하므로 별도 처리가 필요하다.

## 잘못된 예: Pydantic 모델과 파일을 같이 선언

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

파일과 다른 필드(`title`, `user_id` 등)를 같은 요청에 담으면 **전체가 `multipart/form-data`** 가 된다. 이때 JSON body를 기대하는 Pydantic 모델을 그대로 주입하면 **422 Unprocessable Entity** 가 발생한다.

## 해결 방법 1: `Form()`으로 각 필드 분리

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

## 해결 방법 2: `as_form` 패턴으로 Pydantic 모델 재사용

필드가 많을 때는 `Form` 주입 가능한 Pydantic 모델로 변환하는 헬퍼를 만든다.

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

## 요약
- multipart 요청에서는 JSON body를 사용할 수 없다.
- 파일 + 폼 필드를 함께 쓸 때는 `Form()` 또는 `as_form` 패턴으로 처리한다.
