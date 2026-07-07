# Annotated 의존성 주입 마이그레이션 가이드

기존 `param: Type = Depends(provider)` 방식을 FastAPI 권장 방식인
`param: Annotated[Type, Depends(provider)]` 로 전환하는 작업 정리.

## 왜 바꾸나

| | `= Depends(...)` (기존) | `Annotated[T, Depends(...)]` (권장) |
|---|---|---|
| FastAPI 권장 | 동작은 함 | ✅ 0.95+ 공식 권장 |
| 린트 경고(B008) | 발생 가능 | 없음 |
| 재사용 | 매번 반복 | **타입 별칭으로 재사용** |
| 인자 순서 제약 | "기본값 인자"라 제약 생길 수 있음 | 일반 인자라 자유 |

동작은 100% 동일하다. 스타일·재사용성·린트 개선이 목적.

## 두 가지 방식 — 별칭 vs 인라인

`Annotated` 적용에는 두 방식이 있다. **동작은 같고**, 재사용 빈도에 따라 고른다.

| 방식 | 적합한 경우 | 트레이드오프 |
|------|------------|-------------|
| **별칭** (`FinanceScraperDep = Annotated[...]`) | 같은 의존성을 여러 핸들러에서 반복 (`db`, `current_user` 등) | 정의↔사용 위치가 분리됨 |
| **인라인** (`Annotated[...]` 를 인자에 직접) | 그 핸들러에서 1회성으로만 사용 | 반복되면 길고 중복 |

- 현재 라우터는 각 의존성이 **핸들러마다 1번씩만** 쓰여서, 지금은 **인라인(방식 B)** 으로도 충분하다.
- 향후 연예뉴스 등 라우트가 늘어 `summarizer`/`post_service` 를 여러 곳에서 공유하면 **별칭(방식 A)** 으로 리팩토링하면 된다.

아래에 **방식 A(별칭)** 와 **방식 B(인라인)** 를 모두 정리한다. 하나만 골라 적용하면 된다.

---

# 방식 A — 별칭(Alias)으로 정의

## 수정 대상 파일 (2개)

| 파일 | 수정 내용 |
|------|-----------|
| `app/core/dependencies/tistory.py` | 의존성 별칭(`Annotated`) **추가 정의** |
| `app/api/v1/tistory/router.py` | 핸들러 인자를 별칭으로 **교체** |

---

## 1) `app/core/dependencies/tistory.py`

provider 함수 아래에 `Annotated` 별칭을 추가한다.
서비스 타입은 이미 import돼 있으므로, `Annotated` 와 `Depends` import만 추가하면 된다.

### 추가할 import (파일 상단)

```python
from typing import Annotated

from fastapi import Depends
```

### 파일 하단에 별칭 정의 추가

```python
# 의존성 별칭 (Annotated)
FinanceScraperDep = Annotated[FinanceNewsScrapService, Depends(get_finance_news_scraper_service)]
SummarizerDep     = Annotated[NewsSummarizeService, Depends(get_news_summarize_service)]
PostServiceDep    = Annotated[TistoryPostService, Depends(get_tistory_post_service)]
```

### 적용 후 파일 전체 모습

```python
from typing import Annotated

from fastapi import Depends

from app.modules.browser.playwright import PlaywrightManager
from app.modules.llm.groq import create_async_groq_client
from app.services.llm.news_summarize import NewsSummarizeService
from app.services.scraper.finance_news import FinanceNewsScrapService
from app.services.tistory.post import TistoryPostService


def get_finance_news_scraper_service() -> FinanceNewsScrapService:
    return FinanceNewsScrapService(PlaywrightManager(headless=True))


def get_news_summarize_service() -> NewsSummarizeService:
    return NewsSummarizeService(create_async_groq_client())


def get_tistory_post_service() -> TistoryPostService:
    return TistoryPostService(PlaywrightManager(headless=True))


# 의존성 별칭 (Annotated)
FinanceScraperDep = Annotated[FinanceNewsScrapService, Depends(get_finance_news_scraper_service)]
SummarizerDep     = Annotated[NewsSummarizeService, Depends(get_news_summarize_service)]
PostServiceDep    = Annotated[TistoryPostService, Depends(get_tistory_post_service)]
```

---

## 2) `app/api/v1/tistory/router.py`

### import 교체

기존:
```python
from app.core.dependencies.tistory import get_finance_news_scraper_service, get_news_summarize_service, get_tistory_post_service
```

변경:
```python
from app.core.dependencies.tistory import FinanceScraperDep, SummarizerDep, PostServiceDep
```

> `Depends` 를 다른 곳에서 안 쓰면 `from fastapi import APIRouter, Depends` 에서 `Depends` 도 제거 가능.
> 더 이상 사용하지 않는 서비스 타입 import(`FinanceNewsScrapService`, `NewsSummarizeService`, `TistoryPostService`)도 정리 대상.

### 핸들러 3곳 수정

**scarp (23~28줄)**
```python
@router.get('/scrap/finance', response_model=BaseResponse[list[NewsArticle]])
async def scarp(finance_scraper: FinanceScraperDep) -> JSONResponse:
    response = await finance_scraper.do_scraping()
    return success_response({'articles': response})
```

**summarize (31~37줄)**
```python
@router.post('/summarize/finance', response_model=BaseResponse[list[SummarizedArticle]])
async def summarize(
    payload: TistorySummarizeRequest,
    finance_summarizer: SummarizerDep,
) -> JSONResponse:
    summarized_articles = await finance_summarizer.summarize_many(payload.articles, FINANCE_NEWS_SYSTEM_PROMPT)
    return success_response({'summarized_articles': summarized_articles})
```

**publish (40~46줄)**
```python
@router.post('/publish/finance')
async def publish(
    payload: TistoryPublishRequest,
    tistory_post_service: PostServiceDep,
) -> JSONResponse:
    response = await tistory_post_service.do_posting(payload.summarized_articles, payload.reservation_data)
    return success_response(response)
```

---

# 방식 B — 인라인(Inline)

별칭을 만들지 않고, 핸들러 인자에 `Annotated[...]` 를 **직접** 쓴다.
**수정 파일은 `router.py` 1개뿐** (`dependencies/tistory.py` 는 그대로 둠 — 별칭 정의 불필요).

## `app/api/v1/tistory/router.py` 만 수정

### import — provider 함수는 그대로 유지

```python
from typing import Annotated

from fastapi import APIRouter, Depends
from app.core.dependencies.tistory import (
    get_finance_news_scraper_service,
    get_news_summarize_service,
    get_tistory_post_service,
)
```

> 방식 B는 provider 함수(`get_*`)를 그대로 import해서 쓴다. `Depends` 도 계속 필요하다.
> 추가되는 건 `from typing import Annotated` 한 줄뿐.

### 핸들러 3곳 수정

**scarp**
```python
@router.get('/scrap/finance', response_model=BaseResponse[list[NewsArticle]])
async def scarp(
    finance_scraper: Annotated[FinanceNewsScrapService, Depends(get_finance_news_scraper_service)],
) -> JSONResponse:
    response = await finance_scraper.do_scraping()
    return success_response({'articles': response})
```

**summarize**
```python
@router.post('/summarize/finance', response_model=BaseResponse[list[SummarizedArticle]])
async def summarize(
    payload: TistorySummarizeRequest,
    finance_summarizer: Annotated[NewsSummarizeService, Depends(get_news_summarize_service)],
) -> JSONResponse:
    summarized_articles = await finance_summarizer.summarize_many(payload.articles, FINANCE_NEWS_SYSTEM_PROMPT)
    return success_response({'summarized_articles': summarized_articles})
```

**publish**
```python
@router.post('/publish/finance')
async def publish(
    payload: TistoryPublishRequest,
    tistory_post_service: Annotated[TistoryPostService, Depends(get_tistory_post_service)],
) -> JSONResponse:
    response = await tistory_post_service.do_posting(payload.summarized_articles, payload.reservation_data)
    return success_response(response)
```

> 인라인 방식은 `FinanceNewsScrapService` 등 **서비스 타입을 계속 import해야 한다** (타입 자리에 직접 쓰므로). 방식 A에서는 라우터에서 이 타입들을 안 써도 됐지만, 방식 B는 필요하다.

## 두 방식 차이 한눈에

| | 방식 A (별칭) | 방식 B (인라인) |
|---|---|---|
| 수정 파일 | `dependencies/tistory.py` + `router.py` | `router.py` 만 |
| `dependencies` 별칭 정의 | 필요 | 불필요 |
| 라우터의 서비스 타입 import | 불필요 | **필요** |
| 라우터 인자 길이 | 짧음 | 긺 |
| 재사용 | 좋음 | 반복되면 중복 |

---

## 변경 전후 비교 (핸들러 1개 예시)

```python
# 전
async def scarp(
    finance_scraper: FinanceNewsScrapService = Depends(get_finance_news_scraper_service)
) -> JSONResponse:

# 후
async def scarp(finance_scraper: FinanceScraperDep) -> JSONResponse:
```

---

## 체크리스트

### 방식 A — 별칭
- [ ] `dependencies/tistory.py` 에 `Annotated`, `Depends` import 추가
- [ ] `dependencies/tistory.py` 하단에 별칭 3개 정의
- [ ] `router.py` import 를 별칭으로 교체
- [ ] 핸들러 3곳(`scarp`, `summarize`, `publish`) 인자 교체
- [ ] 안 쓰는 import 정리 (라우터의 `Depends`, 서비스 타입들)

### 방식 B — 인라인
- [ ] `router.py` 에 `from typing import Annotated` 추가
- [ ] provider 함수 import 유지 (`get_*`), 서비스 타입 import 유지
- [ ] 핸들러 3곳 인자를 `Annotated[Type, Depends(provider)]` 로 교체

### 공통
- [ ] `/docs` 스웨거에서 의존성 정상 주입 확인 (동작 동일해야 정상)

## 참고

- 향후 연예뉴스 스크래퍼(`EntNewsScrapService`)용 provider를 추가할 때도
  같은 패턴으로 `EntScraperDep = Annotated[...]` 별칭을 함께 정의하면 일관성 유지.
- 별칭이 여러 도메인으로 늘어나면 `app/core/dependencies/types.py` 로 분리 고려.
