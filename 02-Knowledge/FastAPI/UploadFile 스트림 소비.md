---
tags:
  - fastapi
  - uploadfile
  - stream
created: 2026-04-18
---

# FastAPI `UploadFile` 스트림 소비 (`file.read()` 를 두 번 호출하면 빈 `b''` 반환)

`UploadFile` 내부는 **스트림(SpooledTemporaryFile)** 이다. 한 번 읽으면 커서가 끝으로 이동해 두 번째 `read()` 는 빈 바이트를 반환한다.

```python
@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    data1 = await file.read()
    data2 = await file.read()  # b''
```

## 해결 방법

1. **읽은 데이터를 변수에 저장해 재사용**

```python
data = await file.read()
hash_ = hashlib.sha256(data).hexdigest()
save_to_s3(data)
```

2. **스트림 커서를 처음으로 되돌리기** (`seek(0)`)

```python
data = await file.read()
await file.seek(0)
data_again = await file.read()
```

3. **여러 소비자에게 같은 파일을 넘길 때는 `BytesIO` 로 래핑**

```python
from io import BytesIO

buf = BytesIO(await file.read())
Image.open(buf)
buf.seek(0)
ocr_result = do_ocr(buf)
```

## 요약
- `UploadFile.read()` 는 한 번 사용하면 스트림이 소비된다.
- 재사용이 필요하면 `seek(0)` 하거나 메모리 변수에 저장한다.