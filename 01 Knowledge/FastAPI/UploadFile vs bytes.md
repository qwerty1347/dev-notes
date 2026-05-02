---
tags:
  - fastapi
  - uploadfile
  - multipart
created: 2026-04-18
---

# FastAPI 파일 업로드: `UploadFile` vs `bytes`

FastAPI에서 파일 업로드는 `Content-Type: multipart/form-data` 로 전송된다. 이때 파일 타입을 `UploadFile` 또는 `bytes` 로 선언할 수 있다.

## `UploadFile`
- 스트리밍 방식으로 동작
- 내부적으로 `SpooledTemporaryFile` 사용
- 대용량 파일에 적합
- `await file.read()`, `file.filename`, `file.content_type` 제공

```python
from fastapi import FastAPI, UploadFile, File

app = FastAPI()

@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    contents = await file.read()
    return {"filename": file.filename, "size": len(contents)}
```

## `bytes`
- 요청 본문 전체를 메모리에 로드
- 작은 파일에만 권장
- 간단하고 직관적이지만, 대용량에서는 메모리 문제가 발생할 수 있다

```python
@app.post("/upload")
async def upload(file: bytes = File(...)):
    return {"size": len(file)}
```

## 언제 어떤 것을 쓸까?
- `UploadFile`: 업로드한 파일을 저장, 스트리밍 처리, 미들웨어/비동기 처리, 파일 메타데이터가 필요한 경우
- `bytes`: 테스트용이나 아주 작은 임시 파일, 빠른 프로토타이핑

## 주의점
- `UploadFile` 은 스트림이므로 `await file.read()` 를 여러 번 호출하면 두 번째 이후는 `b''` 가 된다. 재사용하려면 `seek(0)` 를 호출하거나 읽은 내용을 변수에 저장해야 한다.
