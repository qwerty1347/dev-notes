# 간단한 CRUD 예제 (FastAPI 라우터 + Repository)

특정 프로젝트와 무관하게, **FastAPI + SQLAlchemy 2.0 (async)** 으로
`라우터 → Repository → DB` 흐름의 최소 CRUD 예제. SQLite를 쓰므로 DB 설치 없이 바로 실행된다.

`created_at` / `updated_at` 타임스탬프 컬럼 포함.

**흐름**: HTTP 요청 → 라우터가 받음 → 의존성으로 세션/Repository 주입 → Repository가 DB 처리 → 응답

> **작업 순서(실무 기준)**: 모델 정의 → **alembic 마이그레이션으로 테이블 생성** → 앱 실행.
> 테이블 생성은 마이그레이션이 담당한다. (`create_all`은 빠른 연습용 — 4번 참고)

---

## 1. 설치

```bash
pip install "fastapi[standard]" sqlalchemy aiosqlite alembic
```

---

## 2. 전체 코드 (`main.py` 한 파일)

```python
from datetime import datetime
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException
from pydantic import BaseModel, ConfigDict
from sqlalchemy import DateTime, String, func, select
from sqlalchemy.ext.asyncio import (
    AsyncSession, async_sessionmaker, create_async_engine,
)
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


# ── 1) Base + 모델 ───────────────────────────────
class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(255), unique=True)

    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(), onupdate=func.now()
    )


# ── 2) 엔진 + 세션 팩토리 ─────────────────────────
engine = create_async_engine("sqlite+aiosqlite:///./test.db", echo=True)
async_session = async_sessionmaker(engine, expire_on_commit=False)


# ── 3) Pydantic 스키마 (요청/응답) ────────────────
class UserCreate(BaseModel):       # 입력: 생성
    email: str


class UserUpdate(BaseModel):       # 입력: 수정
    email: str


class UserRead(BaseModel):         # 출력: ORM 객체 → JSON
    model_config = ConfigDict(from_attributes=True)
    id: int
    email: str
    created_at: datetime
    updated_at: datetime


# ── 4) Repository (DB 처리 로직) ──────────────────
class UserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def create(self, email: str) -> User:
        user = User(email=email)
        self.session.add(user)
        await self.session.commit()
        await self.session.refresh(user)
        return user

    async def get(self, user_id: int) -> User | None:
        return await self.session.get(User, user_id)

    async def list(self) -> list[User]:
        result = await self.session.execute(select(User))
        return list(result.scalars().all())

    async def update(self, user: User, email: str) -> User:
        user.email = email
        await self.session.commit()
        await self.session.refresh(user)
        return user

    async def delete(self, user: User) -> None:
        await self.session.delete(user)
        await self.session.commit()


# ── 5) 의존성 (세션 → Repository 주입) ────────────
async def get_session():
    async with async_session() as session:
        yield session


SessionDep = Annotated[AsyncSession, Depends(get_session)]


def get_user_repo(session: SessionDep) -> UserRepository:
    return UserRepository(session)


UserRepoDep = Annotated[UserRepository, Depends(get_user_repo)]


# ── 6) 앱 ────────────────────────────────────────
#  테이블은 alembic 마이그레이션으로 만든다 (3번 참고).
#  create_all 은 쓰지 않음.
app = FastAPI()


# ── 7) 라우터: 요청 받아서 Repository에 전달 ──────
@app.post("/users", response_model=UserRead, status_code=201)
async def create_user(body: UserCreate, repo: UserRepoDep):
    return await repo.create(body.email)


@app.get("/users", response_model=list[UserRead])
async def list_users(repo: UserRepoDep):
    return await repo.list()


@app.get("/users/{user_id}", response_model=UserRead)
async def get_user(user_id: int, repo: UserRepoDep):
    user = await repo.get(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user


@app.patch("/users/{user_id}", response_model=UserRead)
async def update_user(user_id: int, body: UserUpdate, repo: UserRepoDep):
    user = await repo.get(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return await repo.update(user, body.email)


@app.delete("/users/{user_id}", status_code=204)
async def delete_user(user_id: int, repo: UserRepoDep):
    user = await repo.get(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    await repo.delete(user)
```

---

## 3. 마이그레이션으로 테이블 생성 (alembic) — 실행 전 먼저

실무에선 모델을 정의한 뒤 **테이블 생성은 마이그레이션이 담당**한다.
앱을 띄우기 전에 이 단계를 먼저 끝내야 한다.

> 핵심: **컬럼은 모델에만 적는다.** 마이그레이션 파일은 `--autogenerate`가 모델을 읽고
> 자동으로 만들어준다. (양쪽에 손으로 두 번 적는 게 아님)

### 3-1. 초기화

```bash
# 위 예제가 async 엔진이므로 async 템플릿으로 초기화
alembic init -t async alembic
```

실행하면 `alembic/` 폴더와 `alembic.ini`가 생긴다.

### 3-2. `alembic/env.py` 설정

autogenerate가 모델을 인식하려면 **`target_metadata`에 `Base.metadata`를 연결**해야 한다.

```python
# alembic/env.py 상단

# main.py 의 Base 와 모델을 import
#   ※ User 모델이 import 되어야 Base.metadata 에 등록됨
from main import Base, User  # noqa: F401

# 기본값 target_metadata = None  →  아래로 교체
target_metadata = Base.metadata
```

DB 주소는 `alembic.ini`의 `sqlalchemy.url`에 적는다 (async 드라이버 그대로):

```ini
# alembic.ini
sqlalchemy.url = sqlite+aiosqlite:///./test.db
```

> MySQL이라면 `mysql+aiomysql://user:pass@127.0.0.1:3306/dbname` 처럼 적으면 된다.
> (`-t async` 템플릿이라 async 드라이버 URL을 그대로 쓸 수 있다)

### 3-3. 마이그레이션 생성 & 적용

```bash
# 1) 모델 vs 현재 DB 를 비교해 마이그레이션 파일 자동 생성
alembic revision --autogenerate -m "create users table"

# 2) DB에 적용 (← 이 시점에 테이블이 만들어진다)
alembic upgrade head
```

생성된 파일(`alembic/versions/xxxx_create_users_table.py`)을 열면 alembic이 **자동으로 적어둔**
코드가 들어있다:

```python
def upgrade():
    op.create_table(
        "users",
        sa.Column("id", sa.Integer(), primary_key=True),
        sa.Column("email", sa.String(length=255), nullable=False),
        sa.Column("created_at", sa.DateTime(), server_default=sa.func.now()),
        sa.Column("updated_at", sa.DateTime(), server_default=sa.func.now()),
        sa.UniqueConstraint("email"),
    )

def downgrade():
    op.drop_table("users")
```

### 3-4. 이후 컬럼을 추가할 때 (예: `content`)

```python
# 1) 모델에만 컬럼 추가
from sqlalchemy import Text

class User(Base):
    ...
    content: Mapped[str | None] = mapped_column(Text, nullable=True)
```

```bash
# 2) 마이그레이션 생성 → 적용
alembic revision --autogenerate -m "add content to users"
alembic upgrade head
```

| 자주 쓰는 명령 | 설명 |
|---|---|
| `alembic revision --autogenerate -m "msg"` | 모델 변경 감지해 마이그레이션 생성 |
| `alembic upgrade head` | 최신까지 전부 적용 |
| `alembic downgrade -1` | 한 단계 되돌리기 |
| `alembic current` | 현재 적용된 버전 확인 |
| `alembic history` | 마이그레이션 이력 보기 |

> ⚠️ `--autogenerate`는 완벽하지 않다. 컬럼명 변경·타입 변경·일부 인덱스는 놓칠 수 있으니
> **생성된 마이그레이션 파일을 항상 한 번 열어 확인**하고 적용하자.

---

## 4. 실행

> 테이블이 이미 3번(`alembic upgrade head`)에서 만들어진 상태여야 한다.

```bash
uvicorn main:app --reload
```

서버가 뜨면 브라우저에서 **Swagger UI** 로 바로 테스트:

```
http://127.0.0.1:8000/docs
```

또는 curl 로:

```bash
# CREATE
curl -X POST http://127.0.0.1:8000/users \
  -H "Content-Type: application/json" \
  -d '{"email":"hong@test.com"}'

# READ (목록)
curl http://127.0.0.1:8000/users

# READ (단건)
curl http://127.0.0.1:8000/users/1

# UPDATE
curl -X PATCH http://127.0.0.1:8000/users/1 \
  -H "Content-Type: application/json" \
  -d '{"email":"new@test.com"}'

# DELETE
curl -X DELETE http://127.0.0.1:8000/users/1
```

응답 JSON 에서 `created_at` / `updated_at` 이 채워지고,
PATCH 후 `updated_at` 이 바뀌는 것을 확인할 수 있다.

> **빠른 연습용 대안 (마이그레이션 생략)**: alembic 없이 바로 돌려보고 싶으면,
> `lifespan`에서 `create_all`로 테이블을 만들 수도 있다. 단 **컬럼 추가/변경은 반영 안 되니**
> 연습용으로만. 실무에선 3번(alembic)을 쓴다.
> ```python
> from contextlib import asynccontextmanager
>
> @asynccontextmanager
> async def lifespan(app: FastAPI):
>     async with engine.begin() as conn:
>         await conn.run_sync(Base.metadata.create_all)
>     yield
>
> app = FastAPI(lifespan=lifespan)
> ```

---

## 5. 요청이 흐르는 경로

```
[HTTP 요청]
   │
   ▼
라우터 함수 (create_user 등)      ← 요청 body 받음 (Pydantic 검증)
   │  Depends(get_user_repo)
   ▼
get_user_repo                     ← 세션을 주입받아 Repository 생성
   │  Depends(get_session)
   ▼
get_session                       ← 요청마다 세션 1개 발급, 끝나면 자동 close
   │
   ▼
UserRepository (self.session)     ← 실제 DB 쿼리/커밋
   │
   ▼
[ORM 객체 반환] → response_model(UserRead) 로 JSON 직렬화 → [HTTP 응답]
```

- **라우터**는 요청을 받아 검증하고 Repository에 넘기는 역할만 한다 (얇게 유지).
- **Repository**는 DB 처리 로직만 담는다 (재사용/테스트 용이).
- **세션**은 의존성으로 요청마다 주입 → 전역 변수로 쓰지 않는다.

---

## 6. 타임스탬프 컬럼 핵심 정리

```python
created_at: Mapped[datetime] = mapped_column(
    DateTime, server_default=func.now()
)
updated_at: Mapped[datetime] = mapped_column(
    DateTime,
    server_default=func.now(),   # INSERT 시 채움 ← 빠뜨리면 생성 시 NULL
    onupdate=func.now(),         # UPDATE 시마다 갱신
)
```

| 옵션 | 동작 시점 | 없으면 |
|------|-----------|--------|
| `server_default=func.now()` | INSERT (생성) | 생성 시 값이 NULL |
| `onupdate=func.now()` | UPDATE (수정) | 수정해도 시각이 안 바뀜 |

- `func.now()` → **DB 서버 시간**으로 채워짐 (권장)
- `DateTime` → DB의 `DATETIME`(`YYYY-MM-DD HH:MM:SS`), 파이썬에선 `datetime` 객체로 읽힘

---

## 7. 다른 DB로 바꾸려면 (URL만 교체)

```python
# SQLite (기본, 위 예제)
create_async_engine("sqlite+aiosqlite:///./test.db")

# MySQL  (pip install aiomysql)
create_async_engine("mysql+aiomysql://user:pass@127.0.0.1:3306/dbname")

# PostgreSQL  (pip install asyncpg)
create_async_engine("postgresql+asyncpg://user:pass@127.0.0.1:5432/dbname")
```

모델 / Repository / 라우터 코드는 그대로 두고 URL과 드라이버만 바꾸면 된다.
(alembic을 쓴다면 `alembic.ini`의 `sqlalchemy.url`도 같이 바꿔야 한다)
```
