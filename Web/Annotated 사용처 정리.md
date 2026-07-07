# Annotated 사용처 정리

`Annotated` 는 FastAPI/라우터 전용이 아니라 **파이썬 표준 `typing`** 의 범용 도구다.
의존성 주입은 그중 한 용도일 뿐. 핵심 원리는 하나:

```python
Annotated[실제_타입, 메타데이터1, 메타데이터2, ...]
```

- 첫 번째: 진짜 타입 (`int`, `str` ...) — 타입 체커가 보는 것
- 두 번째 이후: 부가 정보 — **그걸 읽는 도구**(Pydantic, FastAPI, SQLAlchemy 등)가 활용
- 순수 파이썬에선 두 번째 인자는 런타임에 무시됨. 해석하는 도구가 있을 때만 의미가 생긴다.

> 의존성 주입(`Depends`) 적용은 별도 문서 [[Annotated 의존성 주입 마이그레이션]] 참고.

---

## 1) Pydantic 모델 필드 검증 (가장 흔함)

라우터와 무관하게, 스키마에서 제약·메타데이터를 붙일 때.

```python
from typing import Annotated
from pydantic import BaseModel, Field

class Article(BaseModel):
    title: Annotated[str, Field(max_length=32)]            # 32자 제한
    views: Annotated[int, Field(ge=0)]                     # 0 이상
    tags: Annotated[str, Field(pattern=r'^[\w|]+$')]       # 정규식
```

> 이 프로젝트의 `EntertainmentArticle` 등 스키마에 적용 가능.
> 예: title 32자 제한(프롬프트 규칙)을 타입 레벨에서 강제.

---

## 2) FastAPI 파라미터 메타데이터

`Depends` 외에도 요청 파라미터 종류를 전부 `Annotated` 로 표현한다.

```python
from fastapi import Query, Path, Header, Cookie, File, UploadFile

async def search(
    q: Annotated[str, Query(max_length=50)],          # 쿼리스트링
    item_id: Annotated[int, Path(ge=1)],              # 경로 변수
    token: Annotated[str, Header()],                  # 헤더
    session: Annotated[str | None, Cookie()] = None,  # 쿠키
    file: Annotated[UploadFile, File()],              # 업로드 파일
):
    ...
```

---

## 3) Pydantic BeforeValidator / AfterValidator (커스텀 변환)

값을 검증 전후로 가공하는 함수를 타입에 붙인다.

```python
from typing import Annotated
from pydantic import BaseModel, BeforeValidator

def strip_spaces(v: str) -> str:
    return " ".join(v.split())   # 공백/줄바꿈 정리

CleanStr = Annotated[str, BeforeValidator(strip_spaces)]

class Article(BaseModel):
    article: CleanStr   # 입력 시 자동으로 공백 정리됨
```

> 스크래핑 텍스트 정리(`" ".join(content.split())`)를 **타입 레벨에서 자동화**할 수 있다.
> 스크래퍼 메서드마다 수동으로 정리하는 대신 스키마가 처리하게 만드는 선택지.

---

## 4) 타입 별칭 + 메타 정보 문서화

순수 파이썬에서 단위·의미를 주석처럼 남길 때.

```python
Won = Annotated[int, "단위: 원"]
Percentage = Annotated[float, "0.0 ~ 1.0"]

def calc_fee(price: Won) -> Won:
    ...
```

---

## 5) 라이브러리별 메타데이터 (SQLAlchemy 2.0 등)

ORM 컬럼 타입 정의에도 쓰인다.

```python
from typing import Annotated
from sqlalchemy.orm import Mapped, mapped_column

intpk = Annotated[int, mapped_column(primary_key=True)]

class User(Base):
    id: Mapped[intpk]   # 재사용 가능한 컬럼 타입
```

---

## 한눈에 정리

| 사용처 | 무엇을 붙이나 |
|--------|--------------|
| **Pydantic 필드** | `Field(...)` 제약 (길이·범위·정규식) |
| **FastAPI 파라미터** | `Query/Path/Header/Body/Depends...` |
| **Pydantic 검증기** | `BeforeValidator/AfterValidator` 변환 함수 |
| **순수 타입 별칭** | 단위·의미 문서화 문자열 |
| **ORM(SQLAlchemy 등)** | `mapped_column(...)` 등 라이브러리 메타 |

**공통 원리**: `Annotated[실제타입, 메타데이터]` 로 타입에 부가 정보를 붙이고,
그걸 읽는 도구가 활용한다.

## 이 프로젝트에서 바로 써먹을 만한 것

- **1번** — `EntertainmentArticle` / `FinanceArticle` 의 `title` 에 32자 제한
- **3번** — `article` 필드에 공백 자동 정리(`BeforeValidator`) → 스크래퍼 중복 로직 제거
