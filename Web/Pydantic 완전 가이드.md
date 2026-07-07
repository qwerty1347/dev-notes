# Pydantic 완전 가이드 (초보자용)

Pydantic 모델을 **만들고(검증) · 변환하고(dict/JSON) · 다루는** 법을 한 문서에 정리.
처음 보는 사람은 위에서부터 순서대로 읽으면 되고, 익숙하면 [2. 변환 치트시트](#2-변환-치트시트)만 봐도 된다.

> 관련 문서: 실전에서 터진 오류 모음은 [[Tistory API 오류 해결 정리]],
> 응답 민감필드 필터링은 [[response_model_filtering]] (미작성).

## 목차
1. [개념: Pydantic 모델이란 & dict가 모델이 되는 이유](#1-개념-pydantic-모델이란--dict가-모델이-되는-이유)
2. [변환 치트시트 (빠른 참조)](#2-변환-치트시트)
3. [`model_validate` — 데이터 → 모델 (들어올 때)](#3-model_validate--데이터--모델-들어올-때)
4. [`model_dump` — 모델 → dict/JSON (나갈 때)](#4-model_dump--모델--dictjson-나갈-때)
5. [리스트(`list[Model]`) 다루기](#5-리스트listmodel-다루기)
6. [FastAPI 안에서는 대부분 자동](#6-fastapi-안에서는-대부분-자동)
7. [값 규칙 검증 (`model_validator`)](#7-값-규칙-검증-model_validator)
8. [참고: Pydantic v1 → v2 이름 변경](#8-참고-pydantic-v1--v2-이름-변경)
9. [한 장 요약](#9-한-장-요약)

---

## 1. 개념: Pydantic 모델이란 & dict가 모델이 되는 이유

**Pydantic 모델 = "정해진 모양의 데이터"를 표현하는 클래스.** 들어온 데이터가 그 모양에 맞는지 **검증**하고, 맞으면 **객체로 변환**해준다.

```python
from pydantic import BaseModel

class FinanceArticle(BaseModel):
    article: str
```

### dict로 정의해도 오류가 안 나는 이유

`list[dict]` 형태의 데이터를 `list[FinanceArticle]`(모델)로 받아도 오류가 **안 나는 게 정상**이다. Pydantic이 dict를 모델로 **자동 변환(coercion)** 하기 때문. dict는 모델의 "재료"다 (JSON엔 클래스 인스턴스 개념이 없으니, Pydantic은 *항상* dict를 받아 모델로 만든다).

```python
class FinanceSummarizeRequest(BaseModel):
    articles: list[FinanceArticle]
```

입력이 `[{'article': '...'}, {'article': '...'}]` 일 때:
1. 리스트의 각 `dict`를 꺼낸다
2. dict의 `article` 키 → `FinanceArticle.article` 필드에 매핑
3. 타입(`str`) 검증 통과 → `FinanceArticle(article='...')` **객체로 변환**

### ⭐ 변환 후엔 dict가 아니라 "객체"다

검증이 끝나면 `payload.articles` 의 원소는 **`FinanceArticle` 객체**다. dict가 아니다.

```python
payload.articles[0].article      # ✅ 객체 → 속성 접근 (.)
payload.articles[0]['article']   # ❌ TypeError: 'FinanceArticle' object is not subscriptable
```

> 초보자가 가장 많이 하는 실수: 모델로 받아놓고 dict처럼 `['key']` 로 접근 → `not subscriptable` 에러.
> **모델은 `.key`, dict는 `['key']`.**

### 로그에 dict처럼 보여도 객체다

`print(payload)` 출력이 `articles=[{'article': '...'}]` 처럼 보여도 실제 원소는 객체다 (repr이 dict 형태로 보일 뿐). 확인:
```python
print(type(payload.articles[0]))   # <class '...FinanceArticle'>
```

---

## 2. 변환 치트시트

> 외울 거 딱 이것. 모델 ↔ dict ↔ JSON 6방향.

```
pydantic  →  dict       : model_dump()
pydantic  →  json(str)  : model_dump_json()

dict      →  pydantic   : Model.model_validate(d)   (== Model(**d))
json(str) →  pydantic   : Model.model_validate_json(s)

dict      →  json(str)  : json.dumps(d)
json(str) →  dict       : json.loads(s)
```

### 핵심 4개만 외우면 됨
```
모델 → dict   : .model_dump()
모델 → json   : .model_dump_json()
dict → 모델   : .model_validate(d)
json → 모델   : .model_validate_json(s)
```

### 기억법
- **`dump`** = 모델을 **밖으로 쏟아낸다** → 모델에서 데이터로 (**나가는** 방향)
- **`validate`** = 데이터를 **검증해서 모델로 받는다** → 데이터에서 모델로 (**들어오는** 방향)
- 뒤에 **`_json`** 이 붙으면 상대가 **dict가 아니라 JSON 문자열**

### 한눈 코드
```python
class SummarizedArticle(BaseModel):
    title: str
    content: str
    tags: str

article = SummarizedArticle(title='제목', content='본문', tags='AI')

article.model_dump()        # → {'title': '제목', 'content': '본문', 'tags': 'AI'}
article.model_dump_json()   # → '{"title":"제목","content":"본문","tags":"AI"}'

data = {'title': '제목', 'content': '본문', 'tags': 'AI'}
SummarizedArticle.model_validate(data)   # → SummarizedArticle(...)  (== SummarizedArticle(**data))
SummarizedArticle.model_validate_json('{"title":"제목","content":"본문","tags":"AI"}')
```

---

## 3. `model_validate` — 데이터 → 모델 (들어올 때)

손에 **dict나 외부 데이터**가 있고, **타입 검증된 모델로 만들고 싶을 때**.

### 3-1. dict → 모델
```python
data = {'article': '현대차 ETF 출시...'}
article = FinanceArticle.model_validate(data)
print(article.article)   # 현대차 ETF 출시...
```
`Model(**data)` 와 거의 같지만, `model_validate`는 dict가 아닌 입력(ORM 객체 등)도 처리한다.

### 3-2. 중첩 구조도 한 번에
```python
raw = {'articles': [{'article': '기사1'}, {'article': '기사2'}]}
req = FinanceSummarizeRequest.model_validate(raw)
print(type(req.articles[0]))   # <class 'FinanceArticle'>  ← dict가 객체로
```

### 3-3. JSON 문자열 → 모델
```python
article = FinanceArticle.model_validate_json('{"article": "삼성전자 급등..."}')
```

### 3-4. 검증 실패 시 (ValidationError)
```python
from pydantic import ValidationError

try:
    FinanceArticle.model_validate({'content': 'x'})   # 'article' 키 없음
except ValidationError as e:
    print(e)   # article: Field required
```

### 3-5. 실제 사용처 — LLM dict 결과를 모델로
```python
async def summarize_many(self, articles: list[FinanceArticle]) -> list[SummarizedArticle]:
    ...
    summarized = [{'title': '...', 'content': '...', 'tags': '...'}, ...]   # LLM이 만든 dict
    return [SummarizedArticle.model_validate(item) for item in summarized]
```

---

## 4. `model_dump` — 모델 → dict/JSON (나갈 때)

손에 **모델 객체**가 있고, **dict를 기대하는 곳**에 넘길 때.

### 4-1. 모델 → dict
```python
article = SummarizedArticle(title='제목', content='본문', tags='AI')
article.model_dump()   # {'title': '제목', 'content': '본문', 'tags': 'AI'}
```

### 4-2. 모델 → JSON 문자열
```python
article.model_dump_json()   # '{"title":"제목",...}'
```

### 4-3. 일부 필드만 / 제외
```python
article.model_dump(include={'title'})     # title만
article.model_dump(exclude={'content'})    # content 빼고
article.model_dump(exclude_none=True)      # 값이 None인 필드 제외
```

### 4-4. 실무 패턴 — DB/ORM 저장
```python
post = PostORM(**summarized.model_dump())   # 모델 → dict → ORM 펼쳐넣기
```

---

## 5. 리스트(`list[Model]`) 다루기

⚠️ `model_dump()` / `model_dump_json()` 은 **모델 한 개**에만 있는 메서드다. **리스트엔 없어서** 바로 부르면 에러난다.

```python
summarized_articles  # list[SummarizedArticle]
summarized_articles.model_dump_json()   # ❌ AttributeError: 'list' object has no attribute ...
```

### 해결법 3가지

```python
# 1) 각 요소에 적용 (간단, 디버그용)
[a.model_dump() for a in summarized_articles]        # dict 리스트

# 2) TypeAdapter — 리스트 통째 (정석)
from pydantic import TypeAdapter
adapter = TypeAdapter(list[SummarizedArticle])
adapter.dump_json(summarized_articles).decode()      # JSON 문자열 (dump_json은 bytes라 .decode())
adapter.validate_python(raw_list)                    # 반대: dict 리스트 → 모델 리스트

# 3) jsonable_encoder (FastAPI, datetime 등 섞여도 OK)
from fastapi.encoders import jsonable_encoder
jsonable_encoder(summarized_articles)                # dict 리스트
```

> `TypeAdapter`를 자주 쓰면 매번 만들지 말고 **모듈 최상단에 한 번** 만들어 재사용:
> ```python
> SummarizedListAdapter = TypeAdapter(list[SummarizedArticle])
> ```

| 대상 | 직접 `model_dump` | 대안 |
|------|------------------|------|
| 모델 1개 | ✅ | — |
| 모델 리스트 | ❌ | 위 3가지 |

---

## 6. FastAPI 안에서는 대부분 자동

가장 흔한 오해. **라우터 안에서는 보통 `model_validate`/`model_dump`를 직접 안 부른다.**

```python
@router.post('/summarize/finance')
async def summarize(payload: FinanceSummarizeRequest):  # ← 요청 JSON을 FastAPI가 자동 model_validate
    return SummarizedResponse(...)                       # ← response_model 있으면 자동 model_dump
```

- **요청 본문 → 모델**: FastAPI가 자동 `model_validate` (직접 호출 ❌)
- **모델 → 응답 JSON**: `response_model` 지정 시 자동 `model_dump` (직접 호출 ❌)

**직접 호출 = FastAPI가 안 해주는 경계에서만:**
- 노트북에서 데이터 다룰 때
- 서비스 내부에서 dict ↔ 모델 변환 (예: LLM 결과 → 모델)
- DB 저장 / 외부 API 전송 / 캐시 / 파일 / 테스트

### 빠른 결정 가이드
```
지금 내 손에 있는 게 무엇인가?
  dict / JSON / 외부 데이터  → 모델로 만들고 싶다  →  model_validate
  모델 객체                  → dict/JSON로 풀고 싶다 →  model_dump

이 변환을 누가 하나?
  FastAPI 라우터 요청/응답   →  자동. 직접 호출 ❌
  노트북·서비스·DB·외부      →  직접 호출 ✅
```

> ⚠️ 단, 라우터가 `JSONResponse`를 **직접 반환**하면 `response_model` 자동 직렬화가 무시된다.
> (자세한 건 [[response_model_filtering]] — 미작성)

---

## 7. 값 규칙 검증 (`model_validator`)

키·타입이 맞아도 **값 자체의 규칙**(예: "예약 날짜는 과거면 안 됨")을 검증하려면 `model_validator`. `ValueError` 를 raise 하면 FastAPI가 자동 **422**.

```python
from datetime import datetime as dt
from pydantic import BaseModel, model_validator

class ReservationData(BaseModel):
    type: str
    date: str
    time: str | None = None

    @model_validator(mode='after')
    def check_date_not_past(self):
        if self.date < dt.now().strftime('%Y-%m-%d'):   # 'YYYY-MM-DD' 사전식 비교 = 날짜 비교
            raise ValueError('예약 날짜가 오늘보다 이전입니다')
        return self
```

- `mode='after'` : 모든 필드 검증 **뒤** 실행 → `self.date` 접근 가능. 끝에 `return self` 필수.
- 단일 필드만이면 `@field_validator('date')` 도 가능.

### 검증을 어디 둘까 — 경계 vs 서비스

| 검증 종류 | 위치 |
|-----------|------|
| 형식/타입/필수값/enum/단순 값 규칙 | **스키마(Pydantic) · 라우터 경계** |
| 여러 엔티티·외부 상태가 얽힌 비즈니스 규칙 | **서비스/도메인** |

**핵심은 fail-fast** — 비싼 작업(브라우저 실행·로그인·DB) 전에 경계에서 거른다. 날짜 검증을 서비스 깊숙이 두면 로그인·글작성 다 한 뒤 실패해서 낭비다. 스키마로 올려 요청 파싱 단계에서 즉시 차단한다.

> 에러 포맷 주의: 스키마 검증은 **422**(Pydantic), 서비스 `BusinessException`은 **400**(커스텀)으로 다르다. 응답 계약을 정하고 배치할 것.

---

## 8. 참고: Pydantic v1 → v2 이름 변경

| v1 (deprecated) | v2 |
|-----------------|----|
| `instance.dict()` | `instance.model_dump()` |
| `instance.json()` | `instance.model_dump_json()` |
| `Model.parse_obj(d)` | `Model.model_validate(d)` |
| `Model.parse_raw(s)` | `Model.model_validate_json(s)` |

v1 메서드는 v2에서 deprecated. 새 코드는 `model_*` 사용.

---

## 9. 한 장 요약

- **dict로 들어와서 모델로 정의해도 안 난다** → Pydantic이 dict → 모델로 자동 변환.
- **변환 후엔 객체** → `['key']`(인덱싱) 말고 `.key`(속성 접근).
- **데이터 → 모델** = `model_validate` / **모델 → dict·JSON** = `model_dump` (`_json` 붙으면 문자열).
- **리스트는 직접 `model_dump` 불가** → 컴프리헨션 / `TypeAdapter` / `jsonable_encoder`.
- **라우터 안은 둘 다 자동**, 직접 쓰는 건 노트북·서비스·DB 같은 경계뿐.
- **값 규칙 검증**은 `model_validator`로 경계에서(→ 422), fail-fast.
