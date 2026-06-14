# Tistory API 오류 해결 정리

`/tistory` 라우터(스크래핑 → 요약 → 발행) 작업 중 발생한 오류들과 **어디를, 어떻게** 고쳤는지 정리한 문서입니다.

관련 파일
- 라우터: `app/api/v1/tistory/router.py`
- 요청 스키마: `app/schemas/tistory/request.py`, `app/schemas/tistory/article.py`
- 서비스: `app/services/scraper/finance_news.py`, `app/services/llm/news_summarize.py`, `app/services/tistory/news_post.py`

---

## 1. `FastAPIError: Invalid args for response field`

```
fastapi.exceptions.FastAPIError: Invalid args for response field!
Hint: check that <class '...FinanceNewsScrapService'> is a valid Pydantic field type.
```

- **원인**: 의존성을 `Depends` 없이 함수 파라미터 기본값으로 넣어, FastAPI가 `finance_scraper`를 요청 파라미터로 해석하고 그 타입으로 응답 필드를 만들려다 실패.
- **위치**: `router.py` 의 엔드포인트 파라미터
- **수정**: `Depends`로 감싸기

```python
# Before
finance_scraper: FinanceNewsScrapService = get_finance_news_scraper_service

# After
from fastapi import Depends
finance_scraper: FinanceNewsScrapService = Depends(get_finance_news_scraper_service)
```

---

## 2. `Depends`에 함수를 "호출"해서 넘김

- **원인**: `Depends(get_..._service())` 처럼 **괄호로 호출한 결과**(인스턴스)를 넘김. `Depends`는 매 요청마다 호출할 **함수 자체**를 받아야 함.
- **위치**: `router.py`
- **수정**: 괄호 제거

```python
# Before
finance_scraper = Depends(get_finance_news_scraper_service())

# After
finance_scraper = Depends(get_finance_news_scraper_service)   # 괄호 없음
```

---

## 3. `TargetClosedError` / XServer 없음 (Playwright)

```
[TargetClosedError] BrowserType.launch: Target page, context or browser has been closed
Looks like you launched a headed browser without having a XServer running.
Set either 'headless: true' or use 'xvfb-run ...'
```

- **원인**: Docker/리눅스 컨테이너에는 디스플레이(X서버)가 없는데 브라우저를 **headed(화면 있음)** 모드로 실행.
- **위치**: `app/modules/browser/playwright.py` 의 `launch` 호출
- **수정**: `headless=True` 로 실행 (또는 `xvfb-run` 사용)

```python
browser = await playwright.chromium.launch(headless=True)
```

---

## 4. `422 Validation Error` — body가 query로 인식됨

```json
{ "code": "422", "errors": [ { "field": ["query", "articles"], "message": "Field required" } ] }
```

- **원인**: 본문으로 받을 파라미터에 **타입이 없어서** FastAPI가 쿼리 파라미터로 처리. `field`가 `["query", ...]`인 게 단서.
- **위치**: `router.py` 의 `summarize` 파라미터
- **수정**: Pydantic 모델 타입을 지정해 본문으로 받기

```python
# Before
async def summarize(articles, ...):

# After
from app.schemas.tistory.request import FinanceSummarizeRequest
async def summarize(payload: FinanceSummarizeRequest, ...):
    ... = payload.articles
```

> 본문은 한 덩어리이므로 변수명은 `payload` 권장. 쿼리·폼은 개별 값이라 값 이름 그대로(`keyword`, `page`, `username` …) 쓰는 게 자연스러움.

---

## 5. `non-default argument follows default argument`

- **원인**: 기본값 있는 파라미터(`= Depends(...)`) **뒤에** 기본값 없는 파라미터를 둠 (Python 문법 위반).
- **위치**: `router.py` 파라미터 순서
- **수정**: 기본값 없는 파라미터(본문)를 **앞**, `Depends`를 **뒤**로

```python
async def summarize(
    payload: FinanceSummarizeRequest,                       # 기본값 없음 → 먼저
    finance_summarizer: NewsSummarizeService = Depends(...),# 기본값 있음 → 뒤
):
```