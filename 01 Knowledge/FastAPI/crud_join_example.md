# 관계(조인) CRUD 예제 — 2개 테이블, 다(多)컬럼

실무에서 흔한 **1:N 관계**(카테고리 1 ── 게시글 N)를 SQLAlchemy 2.0 (async)로 구현한 예제.
외래키(FK), 관계(`relationship`), Enum, 다양한 컬럼 타입, 그리고 **조인 쿼리**를 다룬다.

```
Category (1) ───< Article (N)
  하나의 카테고리에 여러 게시글이 속한다
```

> 단순 단일 테이블 CRUD는 `simple_crud_example.md` 참고. 이 문서는 **관계/조인** 중심.

---

## 0. 실제 작업 순서 (중요)

**모델이 항상 먼저다.** 마이그레이션은 모델을 읽어서 생성되므로, 모델 없이는 마이그레이션을
만들 수 없다. 실무에서 진행하는 실제 순서:

아래 문서의 섹션 번호가 곧 실제 작업 순서다.

```
1. 의존성 설치
2. DB 연결 설정 — database.py
3. 모델 정의 — models.py                   ← 마이그레이션보다 먼저!
4. 마이그레이션 — alembic
   4-1) 초기화 (최초 1회) — init + env.py + ini
   4-2) 생성 — alembic revision --autogenerate   ← 모델을 읽어 생성
   4-3) 적용 — alembic upgrade head              ← 이 시점에 테이블 생김
5. 스키마 — schemas.py        ┐
6. Repository                 ├ 앱 코드
7. 라우터 — main.py           ┘
8. 실행 & 테스트
```

이후 **컬럼을 추가/변경할 때마다** 반복되는 사이클은 3 → 4-2 → 4-3:

```
모델 수정  →  alembic revision --autogenerate  →  alembic upgrade head
```

> 핵심: **DB 연결 → 모델 → 마이그레이션 → 앱 코드**. 모델이 마이그레이션의 입력이라 항상 먼저다.

---

## 1. 설치

```bash
pip install "fastapi[standard]" sqlalchemy aiosqlite alembic
```

---

## 2. DB 연결 설정 — `database.py`

먼저 DB 연결(engine)과 세션을 정해둔다. 이 URL은 뒤(4번)에서 alembic도 똑같이 쓴다.

```python
from sqlalchemy.ext.asyncio import (
    AsyncSession, async_sessionmaker, create_async_engine,
)

engine = create_async_engine("sqlite+aiosqlite:///./blog.db", echo=True)
async_session = async_sessionmaker(engine, expire_on_commit=False)


async def get_session():
    async with async_session() as session:
        yield session
```

---

## 3. 모델 정의 — `models.py`

테이블의 "정답지". 이 모델을 보고 4번에서 마이그레이션이 생성된다.

```python
from datetime import datetime
from enum import StrEnum   # Python 3.11+ : str + Enum 을 합친 베이스 클래스

from sqlalchemy import (
    Boolean, DateTime, Enum, ForeignKey, Integer, String, Text, func,
)
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, mapped_column, relationship,
)


class Base(DeclarativeBase):
    pass


# ── 상태 Enum (게시글 발행 상태) ───────────────────
# StrEnum 을 상속하면 값이 문자열처럼 동작 → JSON/DB 에 "draft" 로 깔끔하게 저장됨.
# 여기서 쓰는 StrEnum 은 '파이썬 열거형'이고,
# 아래 컬럼의 Enum(...) 은 'SQLAlchemy 컬럼 타입'이라 서로 다른 것이다. (이름만 비슷)
class ArticleStatus(StrEnum):
    DRAFT = "draft"           # 작성 중
    PUBLISHED = "published"   # 발행됨
    ARCHIVED = "archived"     # 보관됨

# [과거 버전] Python 3.10 이하에는 StrEnum 이 없어서 str + enum.Enum 을 직접 상속한다.
#   import enum
#   class ArticleStatus(str, enum.Enum):   # str 을 같이 상속해야 문자열처럼 동작
#       DRAFT = "draft"
#       PUBLISHED = "published"
#       ARCHIVED = "archived"
#
# ※ 이때 베이스를 enum.Enum 으로 쓴 이유: 위 import 의 'Enum'(SQLAlchemy 컬럼 타입)과
#   이름이 충돌하기 때문. 그냥 Enum 이라 쓰면 SQLAlchemy Enum 을 상속해 버린다.


# ── 부모 테이블: 카테고리 (1쪽) ────────────────────
# ★ 부모에는 ForeignKey 를 적지 않는다! relationship 만 둔다.
#   (FK 는 자식 쪽에만 — 아래 Article.category_id 참고)
class Category(Base):
    __tablename__ = "categories"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True)
    slug: Mapped[str] = mapped_column(String(120), unique=True, index=True)
    description: Mapped[str | None] = mapped_column(String(255), nullable=True)

    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), onupdate=func.now()
    )

    # 관계: 이 카테고리에 속한 게시글들 (1:N 의 N쪽)
    # (A) 어노테이션 Mapped[list["Article"]] 로 대상 추론 — 2.0 권장, 가장 간결
    #   back_populates="category" → 상대편(Article)의 relationship 속성 'category' 를 가리킴
    #   (서로 상대방의 변수 이름을 적어 짝을 맺는다: Category.articles ↔ Article.category)

    # ★ 이 줄 맨 앞의 'articles' 가 바로 속성(변수) 이름이다. 따로 어딘가 선언하는 게 아니라
    #   이 변수명 자체가 'articles' 이고, 자식 Article 의 back_populates="articles" 가 이걸 가리킨다.

    #   (id/name 처럼 Category 의 속성 하나인데, 컬럼이 아니라 relationship 일 뿐)
    articles: Mapped[list["Article"]] = relationship(
        back_populates="category",
        cascade="all, delete-orphan",   # 카테고리 삭제 시 하위 게시글도 삭제
    )

    # (B) relationship 첫 인자로 대상 클래스를 문자열로 명시하는 방식 (클래식, 동작 동일)
    #   articles: Mapped[list["Article"]] = relationship(
    #       "Article",
    #       back_populates="category",
    #       cascade="all, delete-orphan",
    #   )


# ── 자식 테이블: 게시글 (N쪽) ──────────────────────
# ★ FK 는 '많은 쪽(N=자식)'에만 둔다. 게시글 1개는 카테고리 1개에 속하므로 category_id 1개로 충분.
#   (부모에 두면 게시글 ID 여러 개를 담아야 해서 컬럼 하나로 표현 불가)
class Article(Base):
    __tablename__ = "articles"

    id: Mapped[int] = mapped_column(primary_key=True)

    # 외래키: categories.id 참조 (인덱스 필수 — 조인 성능)
    # ★ ForeignKey 에는 '테이블 이름'.컬럼 을 적는다 → "categories.id" (소문자 복수, __tablename__)
    #   cf. 아래 relationship 에는 '클래스 이름'(Category)을 적는다 — 둘이 다르니 주의!
    category_id: Mapped[int] = mapped_column(
        ForeignKey("categories.id", ondelete="CASCADE"), index=True
    )

    title: Mapped[str] = mapped_column(String(200))
    slug: Mapped[str] = mapped_column(String(220), unique=True, index=True)
    summary: Mapped[str | None] = mapped_column(String(500), nullable=True)
    content: Mapped[str] = mapped_column(Text)                 # 긴 본문 → TEXT
    thumbnail_url: Mapped[str | None] = mapped_column(String(500), nullable=True)

    status: Mapped[ArticleStatus] = mapped_column(
        Enum(ArticleStatus), default=ArticleStatus.DRAFT, index=True
    )
    view_count: Mapped[int] = mapped_column(Integer, default=0)
    is_featured: Mapped[bool] = mapped_column(Boolean, default=False)

    published_at: Mapped[datetime | None] = mapped_column(
        DateTime, nullable=True   # 발행 전엔 NULL
    )
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), onupdate=func.now()
    )

    # 관계: 이 게시글이 속한 카테고리 (1:N 의 1쪽)
    # ★ 자리마다 적는 '이름'이 다르다 — 헷갈리기 쉬움:
    #     ① ForeignKey("categories.id")  → 테이블 이름  (위 category_id)
    #     ② Mapped["Category"]            → 클래스 이름
    #     ③ back_populates="articles"     → 상대편(부모)의 relationship 속성(변수) 이름
    #        (= Category 클래스 안의 'articles' 속성. 테이블 이름도 클래스 이름도 아님!)
    # (A) 어노테이션으로 대상 추론
    category: Mapped["Category"] = relationship(back_populates="articles")
    # category: Mapped["부모클래스이름"] = relationship(back_populates="부모클래스 relationship 속성 이름")
    # (B) 대상 클래스를 문자열로 명시하는 방식 (동작 동일)
    #   ★ relationship 에는 '클래스 이름'을 적는다 → "Category" (대문자 단수)
    #     ↔ 위 ForeignKey 의 "categories.id"(테이블 이름)와 헷갈리지 말 것
    #   category: Mapped["Category"] = relationship("Category", back_populates="articles")
```

### 관계 설정 핵심
- **FK는 자식(N쪽) 테이블에만** 둔다(`Article.category_id`). 부모(Category)엔 FK 컬럼이 없다.
  - `relationship`은 **양쪽 모두**에 적지만, `ForeignKey`는 **자식 한쪽에만** 적는다 — 이게 핵심 차이.
  - 관계별 FK 위치: **1:N → N(자식)에 1개** / **1:1 → 한쪽에(+`unique=True`)** / **N:M → 중간(연결) 테이블에 FK 2개**.
- `relationship(back_populates=...)`를 **양쪽에 짝**으로 걸면 `category.articles` ↔ `article.category` 양방향 접근 가능.
  - 같은 부모를 가리켜도 자리마다 적는 **이름이 다르다**: ① `ForeignKey`=**테이블 이름**(`"categories.id"`) / ② `Mapped[...]`=**클래스 이름**(`"Category"`) / ③ `back_populates`=**상대편 relationship 속성 이름**(`"articles"`).
- `ondelete="CASCADE"`(DB 레벨) + `cascade="all, delete-orphan"`(ORM 레벨) → 카테고리 삭제 시 게시글도 함께 삭제.
- FK 컬럼엔 `index=True` 권장 (조인/필터 성능).
- **이름 주의**: `ForeignKey`엔 **테이블 이름**(`"categories.id"`, 소문자 복수), `relationship`엔 **클래스 이름**(`"Category"`, 대문자 단수)을 적는다. 서로 다르다.

---

## 4. 마이그레이션 (alembic) — 앱 코드 작성 전 먼저

DB 연결(2번)과 모델(3번)이 정해졌으면, **스키마/Repository/라우터를 만들기 전에
테이블부터 마이그레이션으로 생성**한다. 실무 순서: DB 연결 → 모델 → 마이그레이션 → 앱.

### 4-1. 초기화 (최초 1회)

프로젝트에서 처음 alembic을 쓸 때 한 번만 한다.

```bash
# async 엔진이므로 async 템플릿으로 초기화
alembic init -t async alembic
```

`alembic/env.py` — autogenerate가 모델을 인식하도록 metadata 연결:
```python
from models import Base, Category, Article  # noqa: F401  ← 두 모델 다 import
target_metadata = Base.metadata
```

`alembic.ini` — 2번의 DB URL과 동일하게:
```ini
sqlalchemy.url = sqlite+aiosqlite:///./blog.db
```

### 4-2. 마이그레이션 생성 (모델을 읽어 자동 생성)

```bash
alembic revision --autogenerate -m "create categories and articles"
```

> 관계가 있는 테이블은 **부모(categories) → 자식(articles)** 순으로 생성돼야 하는데,
> autogenerate가 FK 의존성을 보고 순서를 알아서 잡아준다. 생성된 파일을 열면
> `op.create_table("categories", ...)` 가 먼저, FK를 가진 `op.create_table("articles", ...)` 가
> 뒤에 오는 것을 확인할 수 있다.

### 4-3. 마이그레이션 적용 (← 이 시점에 테이블 생김)

```bash
alembic upgrade head
```

이후 **모델을 수정할 때마다** 4-2 → 4-3 만 반복한다 (4-1 초기화는 다시 안 함).

---

## 5. Pydantic 스키마 — `schemas.py`

요청/응답 스키마를 분리하고, 조인 결과를 **중첩(nested)** 으로 표현한다.

```python
from datetime import datetime

from pydantic import BaseModel, ConfigDict

from models import ArticleStatus


# ── Category ──────────────────────────────────────
class CategoryCreate(BaseModel):
    name: str
    slug: str
    description: str | None = None


class CategoryUpdate(BaseModel):
    name: str | None = None
    slug: str | None = None
    description: str | None = None


class CategoryRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    slug: str
    description: str | None
    created_at: datetime


# ── Article ───────────────────────────────────────
class ArticleCreate(BaseModel):
    category_id: int
    title: str
    slug: str
    summary: str | None = None
    content: str
    thumbnail_url: str | None = None
    is_featured: bool = False


class ArticleUpdate(BaseModel):
    title: str | None = None
    summary: str | None = None
    content: str | None = None
    status: ArticleStatus | None = None
    is_featured: bool | None = None


class ArticleRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    category_id: int
    title: str
    slug: str
    summary: str | None
    content: str
    thumbnail_url: str | None
    status: ArticleStatus
    view_count: int
    is_featured: bool
    published_at: datetime | None
    created_at: datetime
    updated_at: datetime


# 조인 결과: 게시글 + 카테고리 정보를 중첩으로
class ArticleWithCategory(ArticleRead):
    category: CategoryRead     # ← 조인된 카테고리 객체가 그대로 직렬화됨


# 조인 결과: 카테고리 + 하위 게시글 목록을 중첩으로 (1:N)
class CategoryWithArticles(CategoryRead):
    articles: list[ArticleRead]   # ← 자식 게시글들이 그대로 직렬화됨
```

---

## 6. Repository (조인 쿼리 포함) — `repositories.py`

```python
from datetime import datetime

from sqlalchemy import func, select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload

from models import Article, ArticleStatus, Category


class CategoryRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    # CREATE
    async def create(self, data: dict) -> Category:
        category = Category(**data)
        self.session.add(category)
        await self.session.commit()
        await self.session.refresh(category)
        return category

    # READ 단건
    async def get(self, category_id: int) -> Category | None:
        return await self.session.get(Category, category_id)

    # READ 단건 — 하위 게시글까지 함께 로딩 (1:N 조인)
    async def get_with_articles(self, category_id: int) -> Category | None:
        stmt = (
            select(Category)
            .where(Category.id == category_id)
            .options(selectinload(Category.articles))   # ← 자식 컬렉션 즉시 로딩
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    # READ 목록
    async def list(self) -> list[Category]:
        stmt = select(Category).order_by(Category.name)
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    # UPDATE — 부분 수정
    async def update(self, category: Category, data: dict) -> Category:
        for field, value in data.items():
            setattr(category, field, value)
        await self.session.commit()
        await self.session.refresh(category)
        return category

    # DELETE — cascade 설정으로 하위 게시글도 함께 삭제됨
    async def delete(self, category: Category) -> None:
        await self.session.delete(category)
        await self.session.commit()


class ArticleRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    # CREATE
    async def create(self, data: dict) -> Article:
        article = Article(**data)
        self.session.add(article)
        await self.session.commit()
        await self.session.refresh(article)
        return article

    # READ 단건 — 카테고리까지 함께 로딩 (조인)
    async def get_with_category(self, article_id: int) -> Article | None:
        stmt = (
            select(Article)
            .where(Article.id == article_id)
            .options(selectinload(Article.category))   # ← 관계 즉시 로딩
        )
        result = await self.session.execute(stmt)
        return result.scalar_one_or_none()

    # READ 목록 — 카테고리 slug 로 필터 (명시적 JOIN)
    async def list_by_category_slug(
        self, category_slug: str
    ) -> list[Article]:
        stmt = (
            select(Article)
            .join(Category, Article.category_id == Category.id)  # ← INNER JOIN
            .where(Category.slug == category_slug)
            .where(Article.status == ArticleStatus.PUBLISHED)
            .order_by(Article.published_at.desc())
            .options(selectinload(Article.category))
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())

    # 집계 — 카테고리별 게시글 수 (GROUP BY 조인)
    async def count_per_category(self) -> list[tuple[str, int]]:
        stmt = (
            select(Category.name, func.count(Article.id))
            .join(Article, Article.category_id == Category.id)
            .group_by(Category.id)
        )
        result = await self.session.execute(stmt)
        return [(name, cnt) for name, cnt in result.all()]

    # UPDATE — 부분 수정
    async def update(self, article: Article, data: dict) -> Article:
        for field, value in data.items():
            setattr(article, field, value)
        await self.session.commit()
        await self.session.refresh(article)
        return article

    # 발행 처리 — 상태 변경 + 발행시각 기록
    async def publish(self, article: Article, now: datetime) -> Article:
        article.status = ArticleStatus.PUBLISHED
        article.published_at = now
        await self.session.commit()
        await self.session.refresh(article)
        return article

    # DELETE
    async def delete(self, article: Article) -> None:
        await self.session.delete(article)
        await self.session.commit()
```

### ⚠️ async 에서 관계 로딩 — 가장 흔한 함정
async 세션에서는 **지연 로딩(lazy load)이 동작하지 않는다.** 아래처럼 그냥 접근하면 에러:

```python
article = await repo.get(article_id)
print(article.category.name)   # ❌ MissingGreenlet 에러 (lazy load 불가)
```

해결책 두 가지:
1. **`selectinload`** (위 예제 방식) — 쿼리 시 관계를 미리 로딩 (별도 SELECT). 권장.
   ```python
   .options(selectinload(Article.category))
   ```
2. **모델에 `lazy="selectin"`** 지정 — 항상 자동 로딩.
   ```python
   category: Mapped["Category"] = relationship(
       back_populates="articles", lazy="selectin"
   )
   ```

> `selectinload`(N+1 방지, 별도 쿼리) vs `joinload`(단일 쿼리 JOIN) 중 1:N 컬렉션엔
> 보통 **`selectinload`** 가 안전하고 효율적이다.

---

## 7. 라우터 — `main.py`

```python
from datetime import datetime, timezone
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from database import get_session
from repositories import ArticleRepository, CategoryRepository
from schemas import (
    ArticleCreate, ArticleRead, ArticleUpdate, ArticleWithCategory,
    CategoryCreate, CategoryRead, CategoryUpdate, CategoryWithArticles,
)

app = FastAPI()

SessionDep = Annotated[AsyncSession, Depends(get_session)]


def get_category_repo(session: SessionDep) -> CategoryRepository:
    return CategoryRepository(session)


def get_article_repo(session: SessionDep) -> ArticleRepository:
    return ArticleRepository(session)


CategoryRepoDep = Annotated[CategoryRepository, Depends(get_category_repo)]
ArticleRepoDep = Annotated[ArticleRepository, Depends(get_article_repo)]


# ── Category CRUD ─────────────────────────────────
@app.post("/categories", response_model=CategoryRead, status_code=201)
async def create_category(body: CategoryCreate, repo: CategoryRepoDep):
    return await repo.create(body.model_dump())


@app.get("/categories", response_model=list[CategoryRead])
async def list_categories(repo: CategoryRepoDep):
    return await repo.list()


# 조인: 카테고리 + 하위 게시글 목록 중첩 응답
@app.get("/categories/{category_id}", response_model=CategoryWithArticles)
async def read_category(category_id: int, repo: CategoryRepoDep):
    category = await repo.get_with_articles(category_id)
    if category is None:
        raise HTTPException(status_code=404, detail="Category not found")
    return category


@app.patch("/categories/{category_id}", response_model=CategoryRead)
async def update_category(
    category_id: int, body: CategoryUpdate, repo: CategoryRepoDep
):
    category = await repo.get(category_id)
    if category is None:
        raise HTTPException(status_code=404, detail="Category not found")
    return await repo.update(category, body.model_dump(exclude_unset=True))


@app.delete("/categories/{category_id}", status_code=204)
async def delete_category(category_id: int, repo: CategoryRepoDep):
    category = await repo.get(category_id)
    if category is None:
        raise HTTPException(status_code=404, detail="Category not found")
    await repo.delete(category)   # cascade 로 하위 게시글도 삭제됨


# ── Article CRUD ──────────────────────────────────


@app.post("/articles", response_model=ArticleRead, status_code=201)
async def create_article(body: ArticleCreate, repo: ArticleRepoDep):
    return await repo.create(body.model_dump())


# 조인: 게시글 + 카테고리 중첩 응답
@app.get("/articles/{article_id}", response_model=ArticleWithCategory)
async def read_article(article_id: int, repo: ArticleRepoDep):
    article = await repo.get_with_category(article_id)
    if article is None:
        raise HTTPException(status_code=404, detail="Article not found")
    return article


# 조인 필터: 카테고리별 발행 게시글 목록
@app.get("/categories/{slug}/articles", response_model=list[ArticleWithCategory])
async def list_by_category(slug: str, repo: ArticleRepoDep):
    return await repo.list_by_category_slug(slug)


# 집계: 카테고리별 게시글 수
@app.get("/stats/articles-per-category")
async def articles_per_category(repo: ArticleRepoDep):
    rows = await repo.count_per_category()
    return [{"category": name, "count": cnt} for name, cnt in rows]


@app.patch("/articles/{article_id}", response_model=ArticleRead)
async def update_article(article_id: int, body: ArticleUpdate, repo: ArticleRepoDep):
    article = await repo.get_with_category(article_id)
    if article is None:
        raise HTTPException(status_code=404, detail="Article not found")
    return await repo.update(article, body.model_dump(exclude_unset=True))


@app.post("/articles/{article_id}/publish", response_model=ArticleRead)
async def publish_article(article_id: int, repo: ArticleRepoDep):
    article = await repo.get_with_category(article_id)
    if article is None:
        raise HTTPException(status_code=404, detail="Article not found")
    return await repo.publish(article, datetime.now(timezone.utc))


@app.delete("/articles/{article_id}", status_code=204)
async def delete_article(article_id: int, repo: ArticleRepoDep):
    article = await repo.get_with_category(article_id)
    if article is None:
        raise HTTPException(status_code=404, detail="Article not found")
    await repo.delete(article)
```

---

## 8. 실행 & 테스트

```bash
uvicorn main:app --reload
# http://127.0.0.1:8000/docs
```

```bash
# 1) 카테고리 생성 (게시글의 부모 — 먼저 만들어야 함)
curl -X POST http://127.0.0.1:8000/categories \
  -H "Content-Type: application/json" \
  -d '{"name": "기술", "slug": "tech", "description": "개발 글"}'

# 2) 게시글 생성 (위에서 만든 category_id 사용)
curl -X POST http://127.0.0.1:8000/articles \
  -H "Content-Type: application/json" \
  -d '{
    "category_id": 1,
    "title": "FastAPI 입문",
    "slug": "fastapi-intro",
    "content": "본문 내용...",
    "summary": "요약"
  }'

# 3) 게시글 조인 조회 (카테고리 중첩 포함)
curl http://127.0.0.1:8000/articles/1

# 4) 카테고리 조인 조회 (하위 게시글 목록 중첩 포함)
curl http://127.0.0.1:8000/categories/1

# 5) 카테고리별 발행글 목록 (slug 로 필터)
curl http://127.0.0.1:8000/categories/tech/articles

# 6) 카테고리별 게시글 수
curl http://127.0.0.1:8000/stats/articles-per-category
```

> **경로 주의**: `GET /categories/{category_id}`(int)와 `GET /categories/{slug}/articles`는
> 경로 구조가 달라 충돌하지 않는다. 단 `/categories/{category_id}`는 정수 ID,
> `/categories/{slug}/articles`는 문자열 slug 기준이라는 점만 구분하면 된다.

---

## 9. 핵심 요약

| 주제 | 포인트 |
|------|--------|
| **FK 위치** | 자식 테이블(`Article.category_id`)에. 부모엔 없음 |
| **양방향 관계** | `relationship(back_populates=...)` 를 양쪽에 짝으로 |
| **삭제 전파** | DB: `ondelete="CASCADE"` + ORM: `cascade="all, delete-orphan"` |
| **async 관계 로딩** | lazy load 안 됨 → `selectinload` 또는 `lazy="selectin"` 필수 |
| **명시적 조인** | `select(A).join(B, A.b_id == B.id).where(...)` |
| **집계 조인** | `select(B.name, func.count(A.id)).join(...).group_by(B.id)` |
| **중첩 응답** | Pydantic 스키마에 관계 객체를 필드로 (`category: CategoryRead`) |
| **인덱스** | FK·자주 필터/정렬하는 컬럼(slug, status)에 `index=True` |
```
