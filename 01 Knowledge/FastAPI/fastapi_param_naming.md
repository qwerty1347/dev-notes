# FastAPI 파라미터 변수명 컨벤션

요청 데이터를 받는 위치(Body / Query / Form / File)별 **변수명 추천**과 그 기준 정리.

---

## 추천 변수명 표

| 위치 | 추천 변수명 | 비고 |
|------|------------|------|
| Body(모델) | `payload` (또는 프로젝트 컨벤션상 `request_dto`) | 통째로 받음 |
| Query(개별) | `keyword`, `page`, `size` … | 값 이름 그대로 |
| Query(모델) | `query`, `filters` | `Depends()`로 묶을 때 |
| Form(개별) | `username`, `password` … | 필드명 그대로 |
| File | `file` / `files` | 단수/복수 |

---

## 기준 (왜 이렇게 나누나)

- **Body는 한 덩어리**라 `payload` / `request_dto` 처럼 **묶는 이름**이 자연스럽다.
- **Query·Form은 개별 값**이라 묶는 이름보다 **각 값의 의미 이름**이 자연스럽다.
- ⚠️ `request` (→ FastAPI `Request`와 충돌), `body` (→ `Body`와 혼동)는 피하는 게 안전하다.

---

## 예시

### Body (모델로 통째 받기)
```python
@router.post('/summarize/finance')
async def summarize(payload: FinanceSummarizeRequest, ...):
    ... = payload.articles
```

### Query (개별 값)
```python
@router.get('/articles')
async def list_articles(
    keyword: str | None = None,
    page: int = 1,
    size: int = 20,
):
    ...
```

### Query (모델로 묶기)
```python
@router.get('/articles')
async def list_articles(query: ArticleQuery = Depends()):
    ...
```

### Form (개별 필드)
```python
from fastapi import Form

@router.post('/login')
async def login(
    username: str = Form(...),
    password: str = Form(...),
):
    ...
```

### File
```python
from fastapi import UploadFile, File

async def upload(file: UploadFile = File(...)):   # 여러 개면 files: list[UploadFile]
    ...
```

---

## 개별 선언 vs 모델로 묶기 (개수 기준)

개별 파라미터는 **나열한 만큼 다 써야 한다.** 그래서 **개수가 많아지면 Pydantic 모델로 묶는다.**

| 파라미터 수 | 방식 | 변수명 |
|------------|------|--------|
| 적음 (2~4개) | 개별 선언 | `keyword`, `page` … (값 이름) |
| 많음 (5개+) | **모델로 묶기** | `query`, `filters`, `form` (묶는 이름) |

### Query 모델로 묶기 (FastAPI 0.115+)
```python
from typing import Annotated
from fastapi import Query
from pydantic import BaseModel

class ArticleQuery(BaseModel):
    keyword: str | None = None
    page: int = 1
    size: int = 20
    sort: str = 'latest'
    category: str | None = None

@router.get('/articles')
async def list_articles(query: Annotated[ArticleQuery, Query()]):
    ... = query.keyword, query.page
```
> 구버전이면 `query: ArticleQuery = Depends()` 형태.

### Form 모델로 묶기 (FastAPI 0.113+)
```python
from typing import Annotated
from fastapi import Form
from pydantic import BaseModel

class SignupForm(BaseModel):
    username: str
    password: str
    email: str
    nickname: str

@router.post('/signup')
async def signup(form: Annotated[SignupForm, Form()]):
    ... = form.username
```

### 모델로 묶으면 따라오는 이점
- **재사용** — 페이징(`page`/`size`) 같은 공통 쿼리를 여러 엔드포인트가 공유
- **검증** — Pydantic 타입/제약 검증 자동 적용
- **문서화** — `/docs`에 깔끔하게 표시

→ 실무에선 공통 쿼리(페이징·정렬 등)는 거의 모델로 빼둔다.

---

## 프로젝트 일관성

`body`를 `request_dto`로 통일하기로 했다면, query 모델은 `query_dto`, form 모델은 `form_dto` 식으로 **같은 접미사 규칙**을 맞추면 일관성이 좋아진다.

가장 중요한 건 이름 자체보다 **프로젝트 안에서의 일관성**이다.
