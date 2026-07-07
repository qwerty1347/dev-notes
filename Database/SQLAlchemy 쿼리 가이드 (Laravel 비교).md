# SQLAlchemy 2.x 쿼리 가이드 (Laravel 비교)

각 항목 구성:
- **Laravel** (Eloquent)
- **SQLAlchemy 2.x async** (`AsyncSession`)
- **SQLAlchemy 2.x sync** (`Session`)

공통 import:
```python
from sqlalchemy import select, update, delete, insert, func, and_, or_
from sqlalchemy.orm import Session, selectinload, joinedload
from sqlalchemy.ext.asyncio import AsyncSession
```

> 섹션은 주제별로 묶여 있음: **조회 기본 → 조건절 → 정렬/페이지네이션 → 집계/그룹 → JOIN/관계 → INSERT → UPDATE → DELETE → Upsert/동시성/트랜잭션 → 고급/Raw → 부록**.

---

# 조회 기본

---

## #1 전체 조회 / `User::all()`

```python
# Laravel
$users = User::all();

# async
async def get_all_users(db: AsyncSession) -> list[User]:
    result = await db.execute(select(User))
    return result.scalars().all()

# sync
def get_all_users(db: Session) -> list[User]:
    return db.execute(select(User)).scalars().all()
```

---

## #2 PK 단건 조회 / `User::find($id)`

```python
# Laravel
$user = User::find(1);

# async
async def get_user(db: AsyncSession, user_id: int) -> User | None:
    return await db.get(User, user_id)

# sync
def get_user(db: Session, user_id: int) -> User | None:
    return db.get(User, user_id)
```

> `db.get()`은 PK 전용. identity map 캐시 활용 → 같은 세션 내 중복 조회 시 SQL 안 날림.

---

## #3 PK 단건 조회 (없으면 예외) / `User::findOrFail($id)`

```python
# Laravel
$user = User::findOrFail(1);

# async
async def get_user_or_404(db: AsyncSession, user_id: int) -> User:
    user = await db.get(User, user_id)
    if user is None:
        raise NotFoundError()
    return user

# sync
def get_user_or_404(db: Session, user_id: int) -> User:
    user = db.get(User, user_id)
    if user is None:
        raise NotFoundError()
    return user
```

---

## #4 조건 단건 조회 / `User::where('email', $email)->first()` (PK 조회)

```python
# Laravel
$user = User::where('email', $email)->first();

# async
async def get_user_by_email(db: AsyncSession, email: str) -> User | None:
    stmt = select(User).where(User.email == email)
    return (await db.execute(stmt)).scalar_one_or_none()

# sync
def get_user_by_email(db: Session, email: str) -> User | None:
    stmt = select(User).where(User.email == email)
    return db.execute(stmt).scalar_one_or_none()
```

> `.scalar_one_or_none()` — 0개면 None, 1개면 객체, 2개+면 예외.
> `.scalar_one()` — 정확히 1개 강제 (없거나 여러 개면 예외).
> - (참고) except MultipleResultsFound:

---

## #5 단일·다중 컬럼 추출 / `User::pluck('email')`

```python
# Laravel
$emails = User::where('status', 'active')->pluck('email');     # ['a@x', 'b@x', ...]
$map    = User::pluck('name', 'id');                           # [id => name]
$rows   = User::select('id', 'name', 'email')->get();          # 여러 컬럼

# ── 단일 컬럼 → 값 리스트 (.scalars()) ─────────────────────────
# async
async def active_emails(db: AsyncSession) -> list[str]:
    stmt = select(User.email).where(User.status == "active")
    return (await db.execute(stmt)).scalars().all()            # ['a@x', 'b@x', ...]

# sync
def active_emails(db: Session) -> list[str]:
    stmt = select(User.email).where(User.status == "active")
    return db.execute(stmt).scalars().all()

# ── 2개 컬럼 → key=>value 맵 (.all() + dict 컴프리헨션) ─────────
# async — pluck('name', 'id') 대응
async def id_name_map(db: AsyncSession) -> dict[int, str]:
    rows = (await db.execute(select(User.id, User.name))).all()
    return {id_: name for id_, name in rows}                   # {1: 'kim', 2: 'lee'}

# ── 다중 컬럼 추출 ────────────────────────────────────────────
# (a) Row 튜플 리스트 — .scalars() 쓰면 첫 컬럼만 나오니 .all()
async def id_name_email_rows(db: AsyncSession) -> list[tuple[int, str, str]]:
    stmt = select(User.id, User.name, User.email).where(User.status == "active")
    return (await db.execute(stmt)).all()                      # [(1,'kim','a@x'), ...]

# (b) 언패킹해서 가공
async def to_labels(db: AsyncSession) -> list[str]:
    rows = (await db.execute(select(User.id, User.name))).all()
    return [f"#{id_} {name}" for id_, name in rows]            # 튜플 언패킹

# (c) 컬럼명으로 접근 (Row 는 named tuple → .id / .name / row[0])
async def first_label(db: AsyncSession) -> str:
    row = (await db.execute(select(User.id, User.name))).first()
    return f"{row.id}:{row.name}" if row else ""               # row[0], row[1] 도 가능

# (d) dict 리스트로 — .mappings().all()  (JSON 응답에 바로 쓰기 좋음)
async def user_dicts(db: AsyncSession) -> list[dict]:
    stmt = select(User.id, User.name, User.email)
    return [dict(m) for m in (await db.execute(stmt)).mappings().all()]
    # [{'id': 1, 'name': 'kim', 'email': 'a@x'}, ...]

# sync — 동일 (.all() / .mappings().all(), await 만 제거)
def id_name_email_rows(db: Session) -> list[tuple[int, str, str]]:
    return db.execute(select(User.id, User.name, User.email)).all()
```

> - **컬럼 1개** → `.scalars().all()` 로 풀면 값 리스트 (`select(User.email)` → `['a@x', ...]`).
> - **2개 (key⇒value)** → `.all()` 후 dict 컴프리헨션 (`{id_: name for id_, name in rows}`).
> - **다중 컬럼** → `.scalars()` 쓰면 **첫 컬럼만** 나오니 쓰지 말 것. `.all()` 로 Row 튜플 리스트를 받아
>   `for a, b, c in rows` 언패킹 / `row.name` 컬럼명 접근 / `.mappings().all()` 로 dict 리스트.
> - 전체 모델 말고 컬럼만 select 하면 네트워크/메모리 절약 + ORM 인스턴스 안 만듦.
> - (참고) 결과 추출 메서드 전체 표는 **부록 — 결과 추출 메서드 정리** 참고.

---

# 조건절 (WHERE)

---

## #6 다중 조건 (AND) / `User::where('status', 'active')->where('age', '>', 18)->get()`

```python
# Laravel
$users = User::where('status', 'active')->where('age', '>', 18)->get();

# async
async def get_active_adults(db: AsyncSession) -> list[User]:
    stmt = select(User).where(
        User.status == "active",
        User.age > 18,
    )
    return (await db.execute(stmt)).scalars().all()

# sync
def get_active_adults(db: Session) -> list[User]:
    stmt = select(User).where(User.status == "active", User.age > 18)
    return db.execute(stmt).scalars().all()
```

> `.where()`에 콤마로 여러 조건 → AND 결합. 명시적으로 `and_(...)`도 가능.

---

## #7 OR 조건 / `User::where(...)->orWhere(...)->get()`

```python
# Laravel
$users = User::where('role', 'admin')->orWhere('role', 'manager')->get();

# async
async def get_managers(db: AsyncSession) -> list[User]:
    stmt = (
        select(User)
        .where(
            or_(
                User.role == "admin",
                User.role == "manager"
            )
        )
    )
    return (await db.execute(stmt)).scalars().all()

# sync
def get_managers(db: Session) -> list[User]:
    stmt = (
        select(User)
        .where(
            or_(
                User.role == "admin",
                User.role == "manager",
            )
        )
    )
    return db.execute(stmt).scalars().all()
```

---

## #8 AND·OR 혼합 조건 / `->where('status','active')->where(fn($q) => $q->where(...)->orWhere(...))`

시나리오: **`status = 'active' AND (role = 'admin' OR age >= 18)`**
— 바깥은 AND, 괄호 안은 OR. **괄호(그룹핑)를 어떻게 거느냐**가 핵심.

```python
# Laravel — orWhere 를 클로저로 감싸야 괄호가 생김
$users = User::where('status', 'active')
    ->where(function ($q) {
        $q->where('role', 'admin')->orWhere('age', '>=', 18);
    })
    ->get();
# SQL: WHERE status = 'active' AND (role = 'admin' OR age >= 18)

# async
async def get_active_admins_or_adults(db: AsyncSession) -> list[User]:
    stmt = select(User).where(
        User.status == "active",
        or_(User.role == "admin", User.age >= 18),   # or_() 가 곧 괄호
    )
    return (await db.execute(stmt)).scalars().all()

# sync
def get_active_admins_or_adults(db: Session) -> list[User]:
    stmt = (
        select(User)
        .where(
            and_(
                User.status == "active",
                or_(User.role == "admin", User.age >= 18),
            )
        )
    )
    return db.execute(stmt).scalars().all()
```

> `.where(a, b)` 의 콤마는 AND. 그 안에 `or_(...)` 를 넣으면 그 부분만 `( ... OR ... )` 로 묶임.
> 반대로 **`(A OR B) AND (C OR D)`** 처럼 OR 그룹이 여러 개면 `and_(or_(...), or_(...))` 로 중첩:
> ```python
> stmt = select(User).where(
>     and_(
>         or_(User.role == "admin", User.role == "manager"),
>         or_(User.status == "active", User.status == "trial"),
>     )
> )
> # WHERE (role = 'admin' OR role = 'manager') AND (status = 'active' OR status = 'trial')
> ```
> `~` 로 부정(NOT)도 가능: `~or_(User.role == "admin", User.role == "manager")` → `NOT (role = 'admin' OR role = 'manager')`.
> 조건을 동적으로 모을 땐 리스트에 담아 언패킹: `conds = [User.status == "active"]; stmt = select(User).where(*conds)`.

---

## #9 IN 조건 / `User::whereIn('id', [1,2,3])->get()`

```python
# Laravel
$users = User::whereIn('id', [1, 2, 3])->get();

# async
async def get_users_in_ids(db: AsyncSession, ids: list[int]) -> list[User]:
    stmt = (
        select(User)
        .where(
            User.id.in_(ids)
        )
    )
    return (await db.execute(stmt)).scalars().all()

# sync
def get_users_in_ids(db: Session, ids: list[int]) -> list[User]:
    stmt = select(User).where(User.id.in_(ids))
    return db.execute(stmt).scalars().all()
```

---

## #10 LIKE / `User::where('name', 'like', '%kim%')->get()`

```python
# Laravel
$users = User::where('name', 'like', '%kim%')->get();

# async
async def search_users(db: AsyncSession, keyword: str) -> list[User]:
    stmt = select(User).where(User.name.ilike(f"%{keyword}%"))  # 대소문자 무시
    return (await db.execute(stmt)).scalars().all()

# sync
def search_users(db: Session, keyword: str) -> list[User]:
    stmt = select(User).where(User.name.like(f"%{keyword}%"))
    return db.execute(stmt).scalars().all()
```

> `.like()` 대소문자 구분, `.ilike()` 무시. 둘 다 와일드카드 `%`/`_` 사용.

---

## #11 BETWEEN / 범위·날짜 조건 / `whereBetween` · `whereDate`

```python
# Laravel
$users  = User::whereBetween('age', [20, 30])->get();
$orders = Order::whereDate('created_at', today())->get();
$orders = Order::whereBetween('created_at', [$start, $end])->get();

from datetime import date, datetime
from sqlalchemy import func

# async — BETWEEN
async def users_in_age_range(db: AsyncSession, lo: int, hi: int) -> list[User]:
    stmt = select(User).where(User.age.between(lo, hi))   # == (User.age >= lo) & (User.age <= hi)
    return (await db.execute(stmt)).scalars().all()

# async — 특정 날짜 (whereDate)
async def orders_on(db: AsyncSession, d: date) -> list[Order]:
    stmt = select(Order).where(func.date(Order.created_at) == d)
    return (await db.execute(stmt)).scalars().all()

# async — 날짜 범위 (인덱스 타려면 func.date 보다 범위 비교 권장)
async def orders_between(db: AsyncSession, start: datetime, end: datetime) -> list[Order]:
    stmt = select(Order).where(Order.created_at >= start, Order.created_at < end)
    return (await db.execute(stmt)).scalars().all()

# sync
def users_in_age_range(db: Session, lo: int, hi: int) -> list[User]:
    return db.execute(select(User).where(User.age.between(lo, hi))).scalars().all()
```

> `func.date(col) == d` 는 컬럼에 함수를 씌워서 **인덱스를 못 탐**. 데이터 많으면
> `col >= 그날 00:00 AND col < 다음날 00:00` 형태가 성능상 유리.

---

## #12 NULL 체크 / `User::whereNull('deleted_at')->get()`

```python
# Laravel
$users = User::whereNull('deleted_at')->get();

# async
async def get_active(db: AsyncSession) -> list[User]:
    stmt = select(User).where(User.deleted_at.is_(None))
    return (await db.execute(stmt)).scalars().all()

# sync
def get_active(db: Session) -> list[User]:
    stmt = select(User).where(User.deleted_at.is_(None))
    return db.execute(stmt).scalars().all()
```

> `is_(None)` 사용 (`== None`은 SQLAlchemy가 처리하지만 lint 경고).
> NOT NULL은 `.is_not(None)`.

---

## #13 Soft Delete / Laravel `use SoftDeletes;`

```python
# Laravel — 모델에 use SoftDeletes; → deleted_at 자동 처리
$users = User::all();                  # deleted_at IS NULL 자동 필터
$user->delete();                       # 실제 삭제 X → UPDATE deleted_at = now()
$users = User::withTrashed()->get();   # 삭제 포함
$users = User::onlyTrashed()->get();   # 삭제된 것만
$user->restore();                      # deleted_at = NULL

from datetime import datetime, timezone

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)

# 기본 조회 (= Laravel 기본: 살아있는 것만)
async def list_users(db: AsyncSession) -> list[User]:
    stmt = select(User).where(User.deleted_at.is_(None))
    return (await db.execute(stmt)).scalars().all()

# soft delete (= $user->delete())
async def soft_delete(db: AsyncSession, user_id: int) -> bool:
    user = await db.get(User, user_id)
    if user is None or user.deleted_at is not None:
        return False
    user.deleted_at = datetime.now(timezone.utc)
    await db.commit()
    return True

# 복구 (= restore())
async def restore(db: AsyncSession, user_id: int) -> None:
    await db.execute(
        update(User).where(User.id == user_id).values(deleted_at=None)
    )
    await db.commit()

# withTrashed = 조건 안 검 / onlyTrashed = User.deleted_at.is_not(None)
```

> SQLAlchemy 엔 Laravel 같은 **자동 글로벌 스코프가 없음** → `#12` 처럼 매 쿼리에 `deleted_at IS NULL`
> 을 명시하거나, 베이스 쿼리 헬퍼/리포지토리로 강제하는 게 보통.
> 진짜 자동화하려면 ORM 이벤트(`do_orm_execute`) + `with_loader_criteria` 패턴이 있지만 복잡함.

---

# 정렬·페이지네이션·결과 제어

---

## #14 정렬 / `User::orderBy('created_at', 'desc')->get()`

```python
# Laravel
$users = User::orderBy('created_at', 'desc')->get();

# async
async def get_users_sorted(db: AsyncSession) -> list[User]:
    stmt = select(User).order_by(User.created_at.desc())
    return (await db.execute(stmt)).scalars().all()

# sync
def get_users_sorted(db: Session) -> list[User]:
    stmt = select(User).order_by(User.created_at.desc())
    return db.execute(stmt).scalars().all()
```

> 오름차순은 `.asc()` 또는 그냥 컬럼만 (`User.created_at`).

---

## #15 페이지네이션 / `User::skip(20)->take(10)->get()`

```python
# Laravel
$users = User::skip(20)->take(10)->get();

# async
async def get_page(db: AsyncSession, page: int, size: int) -> list[User]:
    stmt = select(User).offset((page - 1) * size).limit(size)
    return (await db.execute(stmt)).scalars().all()

# sync
def get_page(db: Session, page: int, size: int) -> list[User]:
    stmt = select(User).offset((page - 1) * size).limit(size)
    return db.execute(stmt).scalars().all()
```

---

## #16 대량 스트리밍 / `User::chunk(500, ...)` · `User::cursor()`

```python
# Laravel
User::chunk(500, function ($users) { foreach ($users as $u) {...} });
User::lazy()->each(fn ($u) => ...);
User::cursor()->each(fn ($u) => ...);          # 단일 커넥션 스트리밍

# sync — yield_per (서버사이드 커서, 메모리 일정)
def process_all(db: Session) -> None:
    stmt = select(User).execution_options(yield_per=500)
    for user in db.execute(stmt).scalars():
        handle(user)                            # 500행씩 끊어 가져오며 순회

# async — db.stream (execute 아님!)
async def process_all(db: AsyncSession) -> None:
    stmt = select(User).execution_options(yield_per=500)
    result = await db.stream(stmt)
    async for user in result.scalars():
        handle(user)

# async — 청크 단위로 받기 (Laravel chunk 와 가장 유사)
async def process_in_chunks(db: AsyncSession) -> None:
    result = await db.stream(select(User).execution_options(yield_per=500))
    async for chunk in result.scalars().partitions(500):
        for user in chunk:                      # chunk: list[User]
            handle(user)
```

> 핵심: async 는 `db.execute` 가 아니라 **`db.stream`** 으로 받아야 진짜 스트리밍.
> `yield_per` 는 서버사이드 커서 + 배치 fetch → 수십만 행도 메모리 일정.
> 단, 스트리밍 순회 중 같은 세션으로 다른 쿼리를 날리면 커서가 깨질 수 있으니 주의.

---

## #17 DISTINCT / `Order::select('status')->distinct()->get()`

```python
# Laravel
$statuses = Order::select('status')->distinct()->get();
$users    = User::distinct()->get();

# async — 특정 컬럼 distinct
async def distinct_statuses(db: AsyncSession) -> list[str]:
    stmt = select(Order.status).distinct()
    return (await db.execute(stmt)).scalars().all()

# async — DISTINCT ON (Postgres 전용: 그룹별 첫 행)
async def latest_order_per_user(db: AsyncSession):
    stmt = (
        select(Order)
        .order_by(Order.user_id, Order.created_at.desc())
        .distinct(Order.user_id)                # PG: DISTINCT ON (user_id)
    )
    return (await db.execute(stmt)).scalars().all()

# sync
def distinct_statuses(db: Session) -> list[str]:
    return db.execute(select(Order.status).distinct()).scalars().all()
```

> `joinedload` + 1:N 과 같이 쓰면 행 중복이 생기는데, 이건 `.distinct()` 가 아니라
> `.unique()` (파이썬 레벨 dedupe) 로 처리해야 함 (#44 흔한 함정 참고).
> `distinct(col)` 인자 형태는 Postgres `DISTINCT ON` 으로만 동작.

---

# 집계·그룹

---

## #18 카운트 / `User::count()`

```python
# Laravel
$count = User::count();
$activeCount = User::where('status', 'active')->count();

# async
async def count_users(db: AsyncSession) -> int:
    return (await db.execute(select(func.count()).select_from(User))).scalar_one()

async def count_active(db: AsyncSession) -> int:
    stmt = select(func.count()).select_from(User).where(User.status == "active")
    return (await db.execute(stmt)).scalar_one()

# sync
def count_users(db: Session) -> int:
    return db.execute(select(func.count()).select_from(User)).scalar_one()
```

---

## #19 EXISTS / `User::where('email', $email)->exists()`

```python
# Laravel
$exists = User::where('email', $email)->exists();

# async
async def email_exists(db: AsyncSession, email: str) -> bool:
    stmt = select(User.id).where(User.email == email).limit(1)
    return (await db.execute(stmt)).first() is not None

# sync
def email_exists(db: Session, email: str) -> bool:
    stmt = select(User.id).where(User.email == email).limit(1)
    return db.execute(stmt).first() is not None
```

---

## #20 집계 (SUM/AVG/MAX/MIN) / `Order::sum('amount')`

```python
# Laravel
$total = Order::sum('amount');
$avg = Order::avg('amount');

# async
async def total_amount(db: AsyncSession) -> int:
    return (await db.execute(select(func.sum(Order.amount)))).scalar() or 0

async def avg_amount(db: AsyncSession) -> float:
    return (await db.execute(select(func.avg(Order.amount)))).scalar() or 0.0

# sync
def total_amount(db: Session) -> int:
    return db.execute(select(func.sum(Order.amount))).scalar() or 0
```

> SUM은 결과 없을 때 `None` 반환 → `or 0`으로 처리 권장.

---

## #21 GROUP BY / `Order::select('user_id', DB::raw('count(*) as cnt'))->groupBy('user_id')->get()`

```python
# Laravel
$grouped = Order::select('user_id', DB::raw('count(*) as cnt'))
    ->groupBy('user_id')->get();

# async
async def order_count_per_user(db: AsyncSession) -> list[tuple[int, int]]:
    stmt = (
        select(Order.user_id, func.count().label("cnt"))
        .group_by(Order.user_id)
    )
    return (await db.execute(stmt)).all()  # [(user_id, cnt), ...]

# sync
def order_count_per_user(db: Session) -> list[tuple[int, int]]:
    stmt = select(Order.user_id, func.count().label("cnt")).group_by(Order.user_id)
    return db.execute(stmt).all()
```

> 여러 컬럼 select 시엔 `.scalars()` 안 쓰고 `.all()`로 Row 튜플 받음.

---

## #22 HAVING / `->groupBy('user_id')->having('total', '>', 10000)`

```python
# Laravel
$rows = Order::select('user_id', DB::raw('SUM(amount) as total'))
    ->groupBy('user_id')
    ->having('total', '>', 10000)
    ->get();

# async
async def big_spenders(db: AsyncSession, threshold: int) -> list[tuple[int, int]]:
    total = func.sum(Order.amount).label("total")
    stmt = (
        select(Order.user_id, total)
        .group_by(Order.user_id)
        .having(total > threshold)              # 라벨/집계 표현식 그대로 사용
    )
    return (await db.execute(stmt)).all()

# sync
def big_spenders(db: Session, threshold: int) -> list[tuple[int, int]]:
    total = func.sum(Order.amount).label("total")
    stmt = select(Order.user_id, total).group_by(Order.user_id).having(total > threshold)
    return db.execute(stmt).all()
```

> `WHERE` = 그룹핑 **전** 행 필터, `HAVING` = 그룹핑 **후** 집계값 필터.
> `having()` 안에는 `func.sum(...) > x` 같은 집계 표현식을 직접 넣음.

---

## #23 윈도우 함수 (순위·누적) / `selectRaw('ROW_NUMBER() OVER (...)')`

```python
# Laravel — 보통 selectRaw 로 직접
$rows = DB::table('orders')->selectRaw(
    'user_id, amount, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn'
)->get();

from sqlalchemy import func

# async — 유저별 금액순 순위 (ROW_NUMBER)
async def ranked_orders(db: AsyncSession):
    rn = func.row_number().over(
        partition_by=Order.user_id,
        order_by=Order.amount.desc(),
    ).label("rn")
    stmt = select(Order.user_id, Order.amount, rn)
    return (await db.execute(stmt)).all()

# async — "유저별 최고액 주문만" (윈도우는 WHERE 에서 못 거름 → 서브쿼리로 감싸 필터)
async def top_order_per_user(db: AsyncSession):
    rn = func.row_number().over(
        partition_by=Order.user_id, order_by=Order.amount.desc()
    ).label("rn")
    ranked = select(Order.id, Order.user_id, Order.amount, rn).subquery()
    stmt = select(ranked).where(ranked.c.rn == 1)
    return (await db.execute(stmt)).all()

# async — 누적합 (running total)
async def running_total(db: AsyncSession):
    cum = func.sum(Order.amount).over(
        partition_by=Order.user_id,
        order_by=Order.created_at,
    ).label("cum")
    stmt = select(Order.user_id, Order.created_at, Order.amount, cum)
    return (await db.execute(stmt)).all()
```

> `func.<집계/순위>().over(partition_by=..., order_by=...)`. 윈도우 함수는 WHERE 에서 못 거르니
> `.subquery()` 로 감싼 뒤 바깥에서 `rn == 1` 필터. `rank()`, `dense_rank()`, `lag()`, `lead()` 도 동일.

---

# JOIN·관계

---

## #24 JOIN / `User::join('orders', 'users.id', '=', 'orders.user_id')->get()`

```python
# Laravel
$users = User::join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.*', 'orders.amount')->get();

# async
async def users_with_orders(db: AsyncSession):
    stmt = select(User, Order).join(Order, Order.user_id == User.id)
    return (await db.execute(stmt)).all()  # [(User, Order), ...]

# sync
def users_with_orders(db: Session):
    stmt = select(User, Order).join(Order, Order.user_id == User.id)
    return db.execute(stmt).all()
```

> 관계가 모델에 정의돼 있으면 `select(User).join(User.orders)`로 더 간결.

---

## #25 Eager Loading (N+1 방지) / `User::with('orders')->get()`

```python
# Laravel
$users = User::with('orders')->get();

# async
async def users_with_orders(db: AsyncSession) -> list[User]:
    stmt = select(User).options(selectinload(User.orders))
    return (await db.execute(stmt)).scalars().all()

# sync
def users_with_orders(db: Session) -> list[User]:
    stmt = select(User).options(selectinload(User.orders))
    return db.execute(stmt).scalars().all()
```

> `selectinload` — 별도 IN 쿼리로 N+1 방지 (대부분 케이스).
> `joinedload` — JOIN으로 한 번에 가져옴 (1:1, M:1 관계에 적합).
> 중첩 관계: `selectinload(User.orders).selectinload(Order.items)`.
> (자세한 로더 선택 기준은 **#44 관계 Eager Loading 종합** 참고)

---

## #26 관계 존재 조건 / `User::whereHas('orders')` · `doesntHave`

```python
# Laravel
$users = User::whereHas('orders')->get();                         # 주문 있는 유저
$users = User::whereHas('orders', fn($q) =>
    $q->where('status','paid'))->get();                           # 조건부
$users = User::doesntHave('orders')->get();                       # 주문 없는 유저
$users = User::has('orders', '>=', 3)->get();                     # 3건 이상

# async — .any() : 1:N / M:N 관계 존재 (상관 EXISTS 생성)
async def users_with_orders(db: AsyncSession) -> list[User]:
    stmt = select(User).where(User.orders.any())
    return (await db.execute(stmt)).scalars().all()

# async — 조건부 whereHas
async def users_with_paid(db: AsyncSession) -> list[User]:
    stmt = select(User).where(User.orders.any(Order.status == "paid"))
    return (await db.execute(stmt)).scalars().all()

# async — doesntHave (~ 로 부정)
async def users_without_orders(db: AsyncSession) -> list[User]:
    stmt = select(User).where(~User.orders.any())
    return (await db.execute(stmt)).scalars().all()

# async — M:1 / 1:1 관계는 .has()
async def orders_of_active_users(db: AsyncSession) -> list[Order]:
    stmt = select(Order).where(Order.user.has(User.status == "active"))
    return (await db.execute(stmt)).scalars().all()

# async — has('orders','>=',3) : 개수 조건은 EXISTS 로 안 됨 → 집계 서브쿼리
async def users_with_3plus_orders(db: AsyncSession) -> list[User]:
    cnt = (
        select(func.count())
        .select_from(Order)
        .where(Order.user_id == User.id)
        .scalar_subquery()
    )
    stmt = select(User).where(cnt >= 3)
    return (await db.execute(stmt)).scalars().all()

# sync — .any()/.has() 그대로
def users_with_paid(db: Session) -> list[User]:
    return db.execute(
        select(User).where(User.orders.any(Order.status == "paid"))
    ).scalars().all()
```

> - 컬렉션 관계(1:N, M:N) → **`.any(조건)`**, 단일 관계(M:1, 1:1) → **`.has(조건)`**.
>   둘 다 상관 `EXISTS` 로 렌더 → 부모 행 중복 없음.
> - `doesntHave` = `~relationship.any()`.
> - `has('orders','>=',n)` 같은 **개수 비교**는 EXISTS 로 표현 불가 → `func.count()` 스칼라 서브쿼리로 비교.
> - JOIN + DISTINCT 로도 되지만 1:N 에서 행 폭증 → 존재 여부만 필요하면 `.any()/.has()` 가 정석.

---

## #27 관계 카운트 / `User::withCount('orders')`

```python
# Laravel
$users = User::withCount('orders')->get();
foreach ($users as $u) { echo $u->orders_count; }
$users = User::withCount(['orders as paid_count' => fn($q) =>
    $q->where('status','paid')])->get();

from sqlalchemy import func
from sqlalchemy.orm import column_property

# async — 상관 스칼라 서브쿼리로 카운트 컬럼 추가
async def users_with_order_count(db: AsyncSession):
    order_count = (
        select(func.count())
        .select_from(Order)
        .where(Order.user_id == User.id)
        .correlate(User)
        .scalar_subquery()
        .label("order_count")
    )
    stmt = select(User, order_count)
    return (await db.execute(stmt)).all()       # [(User, order_count), ...]

# async — 조건부 카운트 (withCount + 필터)
async def users_with_paid_count(db: AsyncSession):
    paid_count = (
        select(func.count())
        .select_from(Order)
        .where(Order.user_id == User.id, Order.status == "paid")
        .scalar_subquery()
        .label("paid_count")
    )
    return (await db.execute(select(User, paid_count))).all()

# async — GROUP BY + OUTER JOIN (한 방 쿼리, 카운트로 정렬/필터 가능)
async def users_order_count_grouped(db: AsyncSession):
    stmt = (
        select(User, func.count(Order.id).label("order_count"))
        .outerjoin(Order, Order.user_id == User.id)
        .group_by(User.id)
        .order_by(func.count(Order.id).desc())
    )
    return (await db.execute(stmt)).all()

# 항상 같이 쓰면 모델에 박아두기 — column_property
class User(Base):
    ...
    order_count = column_property(
        select(func.count(Order.id))
        .where(Order.user_id == id)
        .correlate_except(Order)
        .scalar_subquery()
    )
```

> - 일회성이면 스칼라 서브쿼리 라벨, 항상 필요하면 `column_property`.
> - 카운트로 **정렬/필터**까지 하려면 `outerjoin + group_by` 방식이 편함 (`#22 having` 와도 조합).
> - `column_property` 는 매 SELECT 에 서브쿼리가 붙으니 무거우면 `deferred` 고려.

---

# INSERT

---

## #28 INSERT 단건 / `User::create([...])`

```python
# Laravel
$user = User::create(['name' => 'kim', 'email' => 'a@b.com']);

# async
async def create_user(db: AsyncSession, name: str, email: str) -> User:
    user = User(name=name, email=email)
    db.add(user)
    await db.commit()
    await db.refresh(user)  # DB가 채운 id, created_at 등 가져오기
    return user

# sync
def create_user(db: Session, name: str, email: str) -> User:
    user = User(name=name, email=email)
    db.add(user)
    db.commit()
    db.refresh(user)
    return user
```

---

## #29 INSERT 다건 / `User::insert([...])`

```python
# Laravel
User::insert([
    ['name' => 'a', 'email' => 'a@x'],
    ['name' => 'b', 'email' => 'b@x'],
]);

# async (ORM 객체 일괄)
async def bulk_create(db: AsyncSession, rows: list[dict]) -> None:
    db.add_all([User(**r) for r in rows])
    await db.commit()

# async (Core insert — 더 빠름, 객체 인스턴스 안 만듦)
async def bulk_create_core(db: AsyncSession, rows: list[dict]) -> None:
    await db.execute(insert(User), rows)
    await db.commit()

# sync
def bulk_create(db: Session, rows: list[dict]) -> None:
    db.execute(insert(User), rows)
    db.commit()
```

---

# UPDATE

---

## #30 UPDATE 단건 (조회 후 수정) / `$user->update(['name' => 'kim'])`

```python
# Laravel
$user = User::find(1);
$user->update(['name' => 'kim']);

# async
async def rename_user(db: AsyncSession, user_id: int, name: str) -> User | None:
    user = await db.get(User, user_id)
    if user is None:
        return None
    user.name = name
    await db.commit()
    return user

# sync
def rename_user(db: Session, user_id: int, name: str) -> User | None:
    user = db.get(User, user_id)
    if user is None:
        return None
    user.name = name
    db.commit()
    return user
```

---

## #31 UPDATE 다건 (조회 없이 일괄) / `User::where(...)->update([...])`

```python
# Laravel
User::where('status', 'pending')->update(['status' => 'active']);

# async
async def activate_pending(db: AsyncSession) -> int:
    stmt = update(User).where(User.status == "pending").values(status="active")
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount  # 영향받은 행 수

# sync
def activate_pending(db: Session) -> int:
    stmt = update(User).where(User.status == "pending").values(status="active")
    result = db.execute(stmt)
    db.commit()
    return result.rowcount
```

> 이건 SQL UPDATE 한 방. 조회 안 함 → 성능↑. 단, identity map 캐시는 무효화됨.

---

## #32 increment / decrement (원자적 증감) / `->increment('views')`

```python
# Laravel
Post::where('id', $id)->increment('views');
Post::where('id', $id)->increment('views', 5);
Product::where('id', $id)->decrement('stock', $qty);

# async — 원자적 증가 (조회 안 함 → 레이스 안전)
async def add_view(db: AsyncSession, post_id: int, n: int = 1) -> None:
    stmt = (
        update(Post)
        .where(Post.id == post_id)
        .values(views=Post.views + n)            # ★ 컬럼 자신을 참조 → DB 가 원자적 처리
    )
    await db.execute(stmt)
    await db.commit()

# async — 재고 차감 + 음수 방지 (조건부 UPDATE, 성공 여부 rowcount 로)
async def decrement_stock(db: AsyncSession, pid: int, qty: int) -> bool:
    stmt = (
        update(Product)
        .where(Product.id == pid, Product.stock >= qty)
        .values(stock=Product.stock - qty)
    )
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount == 1                  # 0 이면 재고 부족 → 차감 안 됨

# sync
def add_view(db: Session, post_id: int, n: int = 1) -> None:
    db.execute(update(Post).where(Post.id == post_id).values(views=Post.views + n))
    db.commit()
```

> ❌ `post = get(); post.views += 1; commit()` — 조회·증가 사이에 다른 트랜잭션 끼면 **lost update**.
> ✅ `values(views=Post.views + 1)` → `SET views = views + 1` 로 렌더 → DB 가 원자적 처리, 레이스 없음.
> 하한이 있으면 `WHERE stock >= qty` 를 같이 걸고 `rowcount` 로 성공 판정 (비관적 락 없이 처리).

---

# DELETE

---

## #33 DELETE 단건 / `$user->delete()`

```python
# Laravel
$user = User::find(1);
$user->delete();

# async
async def delete_user(db: AsyncSession, user_id: int) -> bool:
    user = await db.get(User, user_id)
    if user is None:
        return False
    await db.delete(user)
    await db.commit()
    return True

# sync
def delete_user(db: Session, user_id: int) -> bool:
    user = db.get(User, user_id)
    if user is None:
        return False
    db.delete(user)
    db.commit()
    return True
```

---

## #34 DELETE 다건 / `User::where(...)->delete()`

```python
# Laravel
User::where('status', 'inactive')->delete();

# async
async def delete_inactive(db: AsyncSession) -> int:
    stmt = delete(User).where(User.status == "inactive")
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount

# sync
def delete_inactive(db: Session) -> int:
    stmt = delete(User).where(User.status == "inactive")
    result = db.execute(stmt)
    db.commit()
    return result.rowcount
```

---

# Upsert·동시성·트랜잭션

---

## #35 firstOrCreate / `User::firstOrCreate(['email' => $e], ['name' => 'kim'])`

```python
# Laravel
$user = User::firstOrCreate(['email' => $email], ['name' => 'kim']);

# async
async def first_or_create(db: AsyncSession, email: str, name: str) -> User:
    user = (
        await db.execute(select(User).where(User.email == email))
    ).scalar_one_or_none()
    if user is None:
        user = User(email=email, name=name)
        db.add(user)
        await db.commit()
        await db.refresh(user)
    return user

# sync
def first_or_create(db: Session, email: str, name: str) -> User:
    user = db.execute(
        select(User).where(User.email == email)
    ).scalar_one_or_none()
    if user is None:
        user = User(email=email, name=name)
        db.add(user)
        db.commit()
        db.refresh(user)
    return user
```

> 동시성이 중요하면 DB unique 제약 + `INSERT ... ON DUPLICATE KEY UPDATE` 또는 `INSERT ... ON CONFLICT` 사용 권장.

---

## #36 updateOrCreate / `User::updateOrCreate(['email'=>$e], ['name'=>'kim'])`

```python
# Laravel
$user = User::updateOrCreate(
    ['email' => $email],                  # 찾을 조건
    ['name' => 'kim', 'plan' => 'pro'],   # 있으면 갱신 / 없으면 합쳐서 생성
);

# async — 조회 후 분기 (#35 firstOrCreate 의 update 버전)
async def update_or_create(db: AsyncSession, email: str, attrs: dict) -> User:
    user = (
        await db.execute(select(User).where(User.email == email))
    ).scalar_one_or_none()
    if user is None:
        user = User(email=email, **attrs)
        db.add(user)
    else:
        for k, v in attrs.items():
            setattr(user, k, v)
    await db.commit()
    await db.refresh(user)
    return user

# async — 동시성 안전: DB UPSERT (Postgres ON CONFLICT)
from sqlalchemy.dialects.postgresql import insert as pg_insert

async def upsert_user(db: AsyncSession, email: str, attrs: dict) -> None:
    stmt = pg_insert(User).values(email=email, **attrs)
    stmt = stmt.on_conflict_do_update(
        index_elements=["email"],                # email UNIQUE 제약 필요
        set_={k: stmt.excluded[k] for k in attrs},
    )
    await db.execute(stmt)
    await db.commit()

# sync
def update_or_create(db: Session, email: str, attrs: dict) -> User:
    user = db.execute(
        select(User).where(User.email == email)
    ).scalar_one_or_none()
    if user is None:
        user = User(email=email, **attrs); db.add(user)
    else:
        for k, v in attrs.items(): setattr(user, k, v)
    db.commit(); db.refresh(user)
    return user
```

> 조회 후 분기 방식은 동시 요청 시 둘 다 INSERT 시도 가능 → `email` UNIQUE 제약 +
> DB UPSERT(`on_conflict_do_update` / `on_duplicate_key_update`)가 진짜 안전 (#35, #40 와 같은 원리).

---

## #37 비관적 락 / `->lockForUpdate()` · `sharedLock()`

```python
# Laravel
DB::transaction(function () use ($id) {
    $acct = Account::where('id', $id)->lockForUpdate()->first();  # FOR UPDATE
    $acct->balance -= 100;
    $acct->save();
});
$row = Account::where('id', $id)->sharedLock()->first();          # FOR SHARE

# async — with_for_update (반드시 트랜잭션 안에서)
async def withdraw(db: AsyncSession, account_id: int, amount: int) -> bool:
    async with db.begin():
        stmt = (
            select(Account)
            .where(Account.id == account_id)
            .with_for_update()                    # SELECT ... FOR UPDATE
        )
        acct = (await db.execute(stmt)).scalar_one()
        if acct.balance < amount:
            return False
        acct.balance -= amount                    # 락 잡힌 동안 안전하게 갱신
    return True                                   # 블록 종료 시 commit + 락 해제

# 옵션들
select(Account).with_for_update(read=True)        # FOR SHARE (공유 락)
select(Account).with_for_update(nowait=True)      # 락 대기 안 함 → 못 잡으면 즉시 에러
select(Account).with_for_update(skip_locked=True) # 잠긴 행 건너뜀 (잡 큐 패턴)
select(Account).with_for_update(of=Account)       # JOIN 시 특정 테이블만 락

# sync
def withdraw(db: Session, account_id: int, amount: int) -> bool:
    with db.begin():
        acct = db.execute(
            select(Account).where(Account.id == account_id).with_for_update()
        ).scalar_one()
        if acct.balance < amount:
            return False
        acct.balance -= amount
    return True
```

> `with_for_update()` 는 **트랜잭션 안에서만** 의미 있음 (autocommit 이면 락이 바로 풀림).
> 행 락은 commit/rollback 시 해제.
> 잡 큐: `with_for_update(skip_locked=True)` + `limit(n)` 으로 워커가 서로 다른 행을 집어가게 하는 패턴이 정석.
> 단순 카운터 증감이라면 락보다 **#32** 의 `values(col=col+1)` 원자적 UPDATE 가 더 가볍다.

---

## #38 트랜잭션 / `DB::transaction(function () {...})`

```python
# Laravel
DB::transaction(function () {
    User::create([...]);
    Order::create([...]);
});

# async — context manager
async def transfer(db: AsyncSession):
    async with db.begin():       # 자동 commit/rollback
        db.add(User(...))
        db.add(Order(...))
    # 블록 정상 종료 → commit, 예외 → rollback

# sync
def transfer(db: Session):
    with db.begin():
        db.add(User(...))
        db.add(Order(...))
```

> `db.begin()` 컨텍스트는 명시적 `commit()` 호출 불필요. 예외 시 자동 rollback.

---

# 고급 / Raw

---

## #39 Raw SQL / `DB::select('SELECT * FROM users WHERE id = ?', [1])`

```python
# Laravel
$rows = DB::select('SELECT * FROM users WHERE id = ?', [1]);

from sqlalchemy import text

# async
async def raw_query(db: AsyncSession, user_id: int):
    result = await db.execute(
        text("SELECT * FROM users WHERE id = :id"),
        {"id": user_id},
    )
    return result.mappings().all()  # [{column: value, ...}, ...]

# sync
def raw_query(db: Session, user_id: int):
    result = db.execute(
        text("SELECT * FROM users WHERE id = :id"),
        {"id": user_id},
    )
    return result.mappings().all()
```

> 파라미터는 **반드시 named placeholder + dict**로. f-string으로 값 넣으면 SQL injection 취약점.

---

## #40 Bulk Insert (Laravel 배열 스타일) / `Model::insert([...])`

```python
# Laravel — 연관배열의 배열
User::insert([
    ['name' => 'kim',  'email' => 'kim@a.com',  'status' => 'active'],
    ['name' => 'lee',  'email' => 'lee@a.com',  'status' => 'pending'],
    ['name' => 'park', 'email' => 'park@a.com', 'status' => 'inactive'],
]);

# SQLAlchemy 도 동일한 모양 — list[dict] 그대로 넘김
rows = [
    {"name": "kim",  "email": "kim@a.com",  "status": "active"},
    {"name": "lee",  "email": "lee@a.com",  "status": "pending"},
    {"name": "park", "email": "park@a.com", "status": "inactive"},
]

# async — Core insert (가장 빠름, ORM 인스턴스 안 만듦, executemany 한 번)
async def bulk_insert_users(db: AsyncSession, rows: list[dict]) -> None:
    await db.execute(insert(User), rows)
    await db.commit()

# async — RETURNING 으로 생성된 PK 받기 (Postgres/SQLite 등 RETURNING 지원)
async def bulk_insert_with_ids(db: AsyncSession, rows: list[dict]) -> list[int]:
    result = await db.execute(insert(User).returning(User.id), rows)
    await db.commit()
    return [row[0] for row in result.all()]

# async — UPSERT (Postgres: ON CONFLICT DO UPDATE)
from sqlalchemy.dialects.postgresql import insert as pg_insert

async def bulk_upsert_users(db: AsyncSession, rows: list[dict]) -> None:
    stmt = pg_insert(User).values(rows)
    stmt = stmt.on_conflict_do_update(
        index_elements=["email"],                       # 충돌 기준 컬럼
        set_={
            "name":   stmt.excluded.name,               # EXCLUDED.name (INSERT 시도 값)
            "status": stmt.excluded.status,
        },
    )
    await db.execute(stmt)
    await db.commit()

# async — UPSERT (MySQL: ON DUPLICATE KEY UPDATE)
from sqlalchemy.dialects.mysql import insert as mysql_insert

async def bulk_upsert_mysql(db: AsyncSession, rows: list[dict]) -> None:
    stmt = mysql_insert(User).values(rows)
    stmt = stmt.on_duplicate_key_update(
        name=stmt.inserted.name,
        status=stmt.inserted.status,
    )
    await db.execute(stmt)
    await db.commit()

# sync
def bulk_insert_users(db: Session, rows: list[dict]) -> None:
    db.execute(insert(User), rows)
    db.commit()
```

> `db.execute(insert(Model), [dict, dict, ...])` 형태가 Laravel `Model::insert([...])`와 가장 가까움.
> 내부적으로 `executemany`로 1회 왕복에 처리 → 수천 건 단위에서 ORM `add_all` 대비 수 배 빠름.
> 충돌 처리는 dialect별 — Postgres `on_conflict_do_update`, MySQL `on_duplicate_key_update`.

---

## #41 복잡한 서브쿼리 / CTE + Raw 쿼리 비교

시나리오: **"전체 유저 중 평균 구매액 이상으로 산 헤비유저 + 각자의 총 주문 수/금액"**

```python
# Laravel — leftJoinSub + selectRaw + subquery
$totals = DB::table('orders')
    ->select('user_id', DB::raw('COUNT(*) as order_cnt'), DB::raw('SUM(amount) as total'))
    ->where('status', 'paid')
    ->groupBy('user_id');

$avgSub = DB::query()->fromSub($totals, 't')->selectRaw('AVG(total) as avg_total');

$rows = DB::table('users')
    ->joinSub($totals, 'ut', 'ut.user_id', '=', 'users.id')
    ->where('ut.total', '>', $avgSub)
    ->select('users.*', 'ut.order_cnt', 'ut.total')
    ->orderByDesc('ut.total')
    ->get();

# ============================================================
# 방법 A: SQLAlchemy ORM 서브쿼리 (.subquery / .scalar_subquery)
# ============================================================
async def heavy_buyers(db: AsyncSession):
    # 1) 유저별 합계 — FROM 절 서브쿼리
    user_totals = (
        select(
            Order.user_id.label("user_id"),
            func.count().label("order_cnt"),
            func.sum(Order.amount).label("total"),
        )
        .where(Order.status == "paid")
        .group_by(Order.user_id)
        .subquery()
    )

    # 2) 전체 평균 — 스칼라 서브쿼리 (WHERE 절에 인라인)
    avg_total = select(func.avg(user_totals.c.total)).scalar_subquery()

    # 3) 메인 쿼리
    stmt = (
        select(
            User,
            user_totals.c.order_cnt,
            user_totals.c.total,
        )
        .join(user_totals, user_totals.c.user_id == User.id)
        .where(user_totals.c.total > avg_total)
        .order_by(user_totals.c.total.desc())
    )
    return (await db.execute(stmt)).all()  # [(User, order_cnt, total), ...]

# ============================================================
# 방법 B: CTE (WITH ...) — 가독성 ↑, 같은 서브쿼리 재사용 시 유리
# ============================================================
async def heavy_buyers_cte(db: AsyncSession):
    totals_cte = (
        select(
            Order.user_id.label("user_id"),
            func.count().label("order_cnt"),
            func.sum(Order.amount).label("total"),
        )
        .where(Order.status == "paid")
        .group_by(Order.user_id)
        .cte("user_totals")
    )

    avg_total = select(func.avg(totals_cte.c.total)).scalar_subquery()

    stmt = (
        select(User, totals_cte.c.order_cnt, totals_cte.c.total)
        .join(totals_cte, totals_cte.c.user_id == User.id)
        .where(totals_cte.c.total > avg_total)
        .order_by(totals_cte.c.total.desc())
    )
    return (await db.execute(stmt)).all()

# ============================================================
# 방법 C: 동일 쿼리를 Raw SQL 로 — 진짜 복잡할 땐 그냥 이게 낫다
# ============================================================
async def heavy_buyers_raw(db: AsyncSession, status: str = "paid"):
    sql = text("""
        WITH user_totals AS (
            SELECT
                user_id,
                COUNT(*)   AS order_cnt,
                SUM(amount) AS total
            FROM orders
            WHERE status = :status
            GROUP BY user_id
        )
        SELECT
            u.id, u.name, u.email,
            ut.order_cnt,
            ut.total
        FROM users u
        JOIN user_totals ut ON ut.user_id = u.id
        WHERE ut.total > (SELECT AVG(total) FROM user_totals)
        ORDER BY ut.total DESC
    """)
    result = await db.execute(sql, {"status": status})
    return result.mappings().all()  # [{"id":..., "name":..., "order_cnt":..., "total":...}, ...]
```

> 서브쿼리 종류 정리:
> - `.subquery()` — `SELECT ... FROM (SELECT ...) AS alias`. 컬럼은 `subq.c.컬럼명`으로 접근.
> - `.scalar_subquery()` — 단일 값 반환. WHERE/SELECT 절에 인라인 가능 (`= (SELECT ...)`).
> - `.cte("name")` — `WITH name AS (...)`. 재사용/가독성 좋음. PG는 recursive CTE도 `cte.union_all(...)` 로 지원.
> - 복잡한 분석성 쿼리는 ORM 표현식 억지로 쓰는 것보다 `text()` raw SQL이 유지보수 더 편한 경우 많음.

---

## #42 CASE WHEN (switch/case) / `selectRaw('CASE WHEN ...')`

```python
# Laravel
$rows = User::select(
    'id', 'name',
    DB::raw("CASE
        WHEN age < 20 THEN 'teen'
        WHEN age < 60 THEN 'adult'
        ELSE 'senior'
    END AS age_group")
)->get();

from sqlalchemy import case

# async — SELECT 절에서 분기 (switch/case)
async def users_by_age_group(db: AsyncSession):
    age_group = case(
        (User.age < 20, "teen"),
        (User.age < 60, "adult"),
        else_="senior",
    ).label("age_group")

    stmt = select(User.id, User.name, age_group)
    return (await db.execute(stmt)).all()

# async — ORDER BY 에서 사용 (우선순위 정렬)
async def orders_by_priority(db: AsyncSession):
    priority = case(
        (Order.status == "urgent", 1),
        (Order.status == "normal", 2),
        else_=3,
    )
    stmt = select(Order).order_by(priority, Order.created_at.desc())
    return (await db.execute(stmt)).scalars().all()

# async — 조건부 집계 (SUM(CASE WHEN ...)) — 피벗/대시보드 쿼리에 자주 씀
async def status_breakdown(db: AsyncSession):
    stmt = select(
        func.count().label("total"),
        func.sum(case((Order.status == "paid",    1), else_=0)).label("paid_cnt"),
        func.sum(case((Order.status == "pending", 1), else_=0)).label("pending_cnt"),
        func.sum(case((Order.status == "paid", Order.amount), else_=0)).label("paid_sum"),
    )
    return (await db.execute(stmt)).one()

# async — UPDATE 에서 CASE 로 컬럼별 조건부 갱신
async def bulk_resync_status(db: AsyncSession):
    stmt = update(Order).values(
        status=case(
            (Order.amount >= 100_000, "vip"),
            (Order.amount >=  10_000, "normal"),
            else_="small",
        )
    )
    await db.execute(stmt)
    await db.commit()

# sync — 동일
def users_by_age_group(db: Session):
    age_group = case(
        (User.age < 20, "teen"),
        (User.age < 60, "adult"),
        else_="senior",
    ).label("age_group")
    return db.execute(select(User.id, User.name, age_group)).all()
```

> `case((조건1, 값1), (조건2, 값2), ..., else_=기본값)` — 튜플 리스트 형태. SQLAlchemy 1.4 까지의 dict/list 인자 방식은 deprecated.
> 조건은 SQLAlchemy 표현식 (`Col == 'x'`), 값은 리터럴 또는 다른 컬럼/표현식 모두 가능.

---

## #43 CONCAT / 문자열 연결

```python
# Laravel
$rows = User::select(
    'id',
    DB::raw("CONCAT(first_name, ' ', last_name) AS full_name")
)
->where(DB::raw("CONCAT(first_name, ' ', last_name)"), 'like', '%kim%')
->get();

# async — func.concat
async def users_with_full_name(db: AsyncSession, keyword: str):
    full_name = func.concat(User.first_name, " ", User.last_name).label("full_name")

    stmt = (
        select(User.id, full_name)
        .where(func.concat(User.first_name, " ", User.last_name).ilike(f"%{keyword}%"))
    )
    return (await db.execute(stmt)).all()

# async — Python "+" 연산자 (DB가 || 또는 CONCAT 으로 알아서 변환)
async def users_with_full_name_alt(db: AsyncSession):
    full_name = (User.first_name + " " + User.last_name).label("full_name")
    return (await db.execute(select(User.id, full_name))).all()

# async — CONCAT_WS (구분자 사용, NULL 무시) — 주소/이름 합칠 때 유용
async def address_line(db: AsyncSession):
    addr = func.concat_ws(", ", User.city, User.street, User.zip_code).label("addr")
    return (await db.execute(select(User.id, addr))).all()

# async — CASE + CONCAT 조합 (전체 이름 + 등급 라벨)
async def labeled_names(db: AsyncSession):
    label = func.concat(
        User.first_name, " ", User.last_name,
        " (",
        case(
            (User.age < 20, "teen"),
            (User.age < 60, "adult"),
            else_="senior",
        ),
        ")",
    ).label("display")
    return (await db.execute(select(User.id, label))).all()

# sync
def users_with_full_name(db: Session, keyword: str):
    full_name = func.concat(User.first_name, " ", User.last_name).label("full_name")
    stmt = (
        select(User.id, full_name)
        .where(func.concat(User.first_name, " ", User.last_name).ilike(f"%{keyword}%"))
    )
    return db.execute(stmt).all()
```

> - `func.concat(a, b, c)` — 표준 `CONCAT`. NULL 인자가 있으면 결과도 NULL인 DB 있음 (Postgres). MySQL은 그냥 빈 문자열로 처리.
> - `func.concat_ws(sep, a, b, c)` — `CONCAT_WS` (with separator). NULL 인자는 자동으로 건너뜀 → 안전.
> - 컬럼끼리 `+` 연산자 — Postgres/SQLite는 `||`, MySQL은 `CONCAT()`으로 dialect-aware 변환됨.
> - LIKE 안에서 동적 prefix/suffix가 필요하면 `func.concat('%', :kw, '%')` 처럼 DB 쪽에서 합치는 게 더 깔끔할 때도 있음.

---

## #44 관계 Eager Loading 종합 / `User::with('orders.items')`

Laravel `with`에 해당. **언제 어느 로더를 쓰는지**가 핵심.

```python
# Laravel — 다양한 with 패턴
$users = User::with('orders')->get();                              # 1단계
$users = User::with(['orders', 'profile'])->get();                 # 여러 관계
$users = User::with('orders.items')->get();                        # 중첩
$users = User::with(['orders' => fn($q) => $q->where('status','paid')])->get();  # 조건부
```

### 44-1. 기본 — 단일 관계

```python
from sqlalchemy.orm import selectinload, joinedload, contains_eager

# async
async def list_users_with_orders(db: AsyncSession) -> list[User]:
    stmt = select(User).options(selectinload(User.orders))
    return (await db.execute(stmt)).scalars().all()

# 이후 코드에서 user.orders 접근해도 추가 SQL 없음 (이미 로드됨)
# 만약 selectinload 없이 async 에서 user.orders 접근 → MissingGreenlet 에러
```

### 44-2. 여러 관계 동시 로드

```python
# Laravel: User::with(['orders', 'profile'])
async def list_users_full(db: AsyncSession) -> list[User]:
    stmt = select(User).options(
        selectinload(User.orders),
        selectinload(User.profile),     # 콤마로 나열, 각각 별도 쿼리
        joinedload(User.company),        # M:1 은 joinedload 가 보통 유리
    )
    return (await db.execute(stmt)).scalars().all()
```

### 44-3. 중첩 관계 (orders.items)

```python
# Laravel: User::with('orders.items.product')
async def list_users_deep(db: AsyncSession) -> list[User]:
    stmt = select(User).options(
        selectinload(User.orders)
            .selectinload(Order.items)
            .joinedload(OrderItem.product),  # 마지막 단은 M:1 → joinedload
    )
    return (await db.execute(stmt)).scalars().all()
```

### 44-4. 조건부 eager load (관계 필터)

Laravel에서 자주 쓰는 방식은 **모델에 필터된 관계 메서드를 따로 정의 → `with('함수명')` 호출**.
SQLAlchemy도 똑같이 모델에 필터된 `relationship` 을 추가하면 된다.

```php
// Laravel — 모델에 필터된 관계 추가
class User extends Model {
    public function orders() {
        return $this->hasMany(Order::class);
    }
    public function paidOrders() {                                 // ← 필터된 관계
        return $this->hasMany(Order::class)->where('status', 'paid');
    }
}

// 호출 — 그냥 with('함수명')
$users = User::with('paidOrders')->get();
```

```python
# SQLAlchemy — 모델에 필터된 relationship 추가
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)

    orders: Mapped[list["Order"]] = relationship(back_populates="user")

    # 같은 Order 테이블이지만 status='paid' 만 잡는 별도 관계
    paid_orders: Mapped[list["Order"]] = relationship(
        primaryjoin="and_(User.id == Order.user_id, Order.status == 'paid')",
        viewonly=True,     # 필터된 관계는 read-only 권장 (INSERT/UPDATE 혼동 방지)
    )

# 호출 — 그냥 selectinload(User.paid_orders)
async def users_with_paid_orders(db: AsyncSession) -> list[User]:
    stmt = select(User).options(selectinload(User.paid_orders))
    return (await db.execute(stmt)).scalars().all()
```

> 인라인으로 한 번만 쓸 거면 매번 관계 추가하는 게 부담스러우니 이렇게도 가능:
> ```python
> # Laravel 의 closure 형태와 대응 — User::with(['orders' => fn($q) => $q->where(...)])
> selectinload(User.orders.and_(Order.status == "paid"))
> ```
> 단, 여러 곳에서 같은 필터 쓰면 그냥 모델에 `paid_orders` 정의하는 게 깔끔.

### 44-5. 이미 JOIN 한 결과를 그대로 eager load 로 채우기 — `contains_eager`

조회 조건이 자식 테이블에 걸려있어서 어차피 JOIN 을 해야 할 때:

```python
async def users_who_have_paid(db: AsyncSession) -> list[User]:
    stmt = (
        select(User)
        .join(User.orders)
        .where(Order.status == "paid")
        .options(contains_eager(User.orders))   # JOIN 결과를 user.orders 에 채움
        .distinct()
    )
    return (await db.execute(stmt)).scalars().unique().all()
```

> 주의: `contains_eager`는 **JOIN의 결과만** 채워넣음. 위 예시에서 `user.orders`엔 `paid` 주문만 들어있고 다른 주문은 누락됨 → 의도한 동작인지 꼭 확인.

---

### `selectinload` vs `joinedload` 정리

| 로더 | SQL 모양 | 적합한 관계 | 장점 | 단점 |
|---|---|---|---|---|
| `selectinload` | 별도 쿼리 `WHERE parent_id IN (...)` | **1:N, M:N** | 행 폭증 없음, N+1 해결 표준 | 쿼리 1번 추가 (그래도 N+1보단 압도적으로 좋음) |
| `joinedload` | LEFT OUTER JOIN | **M:1, 1:1** | 쿼리 1번에 끝 | 1:N 에 쓰면 부모 행이 N배 중복 → `.unique()` 필수, 메모리/네트워크 낭비 |
| `contains_eager` | 사용자가 직접 짠 JOIN | JOIN을 어차피 하는 경우 | WHERE 조건도 자식에 걸 수 있음 | 관계의 "전체"가 아니라 "JOIN된 부분"만 채워짐 |
| `raiseload` | 접근 시 예외 | 디버깅용 | lazy load 실수 잡아냄 | 실제 사용은 X |

**선택 기준 한 줄 요약:**
- 부모→자식이 **여러 개** (1:N, M:N) → `selectinload`
- 부모→자식이 **하나** (M:1, 1:1) → `joinedload`
- WHERE 가 자식 컬럼에 걸려서 어차피 JOIN → `contains_eager`

### `selectinload`가 실제로 날리는 SQL

```python
stmt = select(User).options(selectinload(User.orders))
```

내부적으로 두 쿼리가 나감:

```sql
-- 1) 부모 먼저 가져옴
SELECT * FROM users;

-- 2) 부모 id 모아서 자식 IN 으로 한 번에
SELECT * FROM orders WHERE user_id IN (1, 2, 3, ..., 500);
```

→ `users.orders`에 자동 매핑. **N+1 (1 + N개 쿼리) → 1 + 1 = 2 쿼리.**

### 왜 async 에선 eager load가 "필수"인가

```python
# sync: 이거 그냥 됨 (대신 N+1 발생)
users = db.execute(select(User)).scalars().all()
for u in users:
    print(u.orders)           # 접근 시점에 lazy load SQL 자동 발생

# async: 이거 터짐
users = (await db.execute(select(User))).scalars().all()
for u in users:
    print(u.orders)           # ❌ MissingGreenlet 예외
```

async 컨텍스트에선 lazy load 시점에 동기 IO 호출이 일어나야 하는데, async 세션은 그걸 막아둠. 그래서 **async 에선 관계 접근하기 전에 무조건 `options(selectinload/joinedload)` 로 미리 로드해야 함.**

### 흔한 함정

```python
# ❌ joinedload + 1:N + .scalars().all() → 부모가 자식 수만큼 중복
stmt = select(User).options(joinedload(User.orders))  # User 1:N Order
users = (await db.execute(stmt)).scalars().all()
# → 주문 3개 있는 유저가 3번 나옴

# ✅ .unique() 추가
users = (await db.execute(stmt)).scalars().unique().all()

# ✅ 또는 selectinload 로 변경 (1:N 엔 이게 정석)
stmt = select(User).options(selectinload(User.orders))
users = (await db.execute(stmt)).scalars().all()
```

---

## #45 DateTime INSERT / UPDATE (`now()` / `Carbon::now()` 대응)

핵심은 **시간을 누가 만드느냐** — 앱 서버(Python `datetime`) vs DB 서버(`func.now()`).

### 45-1. 모델 정의 — 자동 created_at/updated_at (`$table->timestamps()`)

```python
from datetime import datetime
from sqlalchemy import DateTime, func
from sqlalchemy.orm import Mapped, mapped_column

class Post(Base):
    __tablename__ = "posts"
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    published_at: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)

    # Laravel: $table->timestamps()
    created_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(),            # INSERT 시 DB가 NOW() 채움
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, server_default=func.now(),
        onupdate=func.now(),                             # UPDATE 시 자동 갱신
    )
```

> `server_default=func.now()` → DDL `DEFAULT CURRENT_TIMESTAMP`. DB가 채움 → 앱-DB 시계 불일치 없음.
> `onupdate=func.now()` → **ORM** UPDATE(객체 수정/`update()`)에서만 자동. raw SQL엔 안 걸림 → 그땐 컬럼 DDL에 `ON UPDATE CURRENT_TIMESTAMP` 필요.

### 45-2. INSERT — 값 전달 3가지

```python
# Laravel
$post = Post::create(['title' => 'hi']);                              # 자동 타임스탬프
Post::create(['title' => 'hi', 'published_at' => now()]);             # Carbon 전달
Post::create(['title' => 'hi', 'published_at' => Carbon::now()->addDays(7)]);

from datetime import datetime, timedelta, timezone

# (1) 자동 — server_default 가 채움, 객체엔 값 안 줌
async def create_post(db: AsyncSession, title: str) -> Post:
    post = Post(title=title)
    db.add(post)
    await db.commit()
    await db.refresh(post)            # DB가 채운 created_at/updated_at 읽어옴
    return post

# (2) 파이썬 시간 전달 (now() / Carbon::now() 대응) — aware UTC 권장
async def publish_now(db: AsyncSession, title: str) -> Post:
    post = Post(title=title, published_at=datetime.now(timezone.utc))
    db.add(post); await db.commit(); await db.refresh(post)
    return post

# (3) DB 서버 시간 전달 (SQL NOW())
async def publish_db_now(db: AsyncSession, title: str) -> Post:
    post = Post(title=title, published_at=func.now())   # 컬럼에 SQL NOW() 들어감
    db.add(post); await db.commit(); await db.refresh(post)
    return post

# Carbon::now()->addDays(7) 대응
post = Post(title=title, published_at=datetime.now(timezone.utc) + timedelta(days=7))

# sync — await 만 제거
def create_post_sync(db: Session, title: str) -> Post:
    post = Post(title=title, published_at=datetime.now(timezone.utc))
    db.add(post); db.commit(); db.refresh(post)
    return post
```

### 45-3. UPDATE — datetime 갱신

```python
# Laravel
$post->update(['published_at' => now()]);
Post::where('status','draft')->update(['published_at' => now()]);

# async — 객체 수정 (updated_at 은 onupdate 로 자동)
async def mark_published(db: AsyncSession, post_id: int) -> Post | None:
    post = await db.get(Post, post_id)
    if post is None:
        return None
    post.published_at = datetime.now(timezone.utc)      # 또는 func.now()
    await db.commit()
    return post

# async — 일괄 update() (조회 없이)
async def publish_drafts(db: AsyncSession) -> int:
    stmt = (
        update(Post)
        .where(Post.status == "draft")
        .values(
            published_at=func.now(),                    # DB 시간
            updated_at=func.now(),                       # ★ 명시 필요 (아래 주석)
        )
    )
    result = await db.execute(stmt)
    await db.commit()
    return result.rowcount
```

> ★ 함정: `update(Model).values(...)` **Core 일괄 UPDATE는 `onupdate=func.now()` 가 자동 적용 안 됨.** `updated_at=func.now()` 를 직접 넣어야 함. (객체 수정 방식 #30 은 자동 적용됨)

### 45-4. 요청(API)에서 문자열로 받은 날짜 처리

```python
# 클라이언트가 "2026-05-18T09:30:00" 같은 문자열로 보냄

# 권장: Pydantic 이 datetime 으로 파싱 (FastAPI)
from pydantic import BaseModel

class PostIn(BaseModel):
    title: str
    published_at: datetime | None = None        # ISO 문자열 → datetime 자동 변환

async def create(db: AsyncSession, body: PostIn) -> Post:
    post = Post(
        title=body.title,
        published_at=body.published_at or datetime.now(timezone.utc),
    )
    db.add(post); await db.commit()
    return post

# Pydantic 안 쓰고 직접 파싱
dt = datetime.fromisoformat("2026-05-18 09:30:00")          # 표준 ISO
dt = datetime.strptime(raw, "%Y-%m-%d %H:%M:%S")            # 커스텀 포맷
```

> 문자열을 컬럼에 그대로 넣지 말 것 — dialect/포맷에 따라 깨짐. 항상 `datetime` 객체로 변환 후 전달.

### Laravel ↔ SQLAlchemy datetime 매핑

| Laravel/PHP | SQLAlchemy/Python | 시간 출처 | 비고 |
|---|---|---|---|
| `now()`, `Carbon::now()` | `datetime.now(timezone.utc)` | 앱 서버 | aware(UTC) 권장 |
| `Carbon::now()` (naive) | `datetime.now()` | 앱 서버 | naive — TZ 혼동, 비권장 |
| `DB::raw('NOW()')` | `func.now()` | DB 서버 | 앱-DB 시계 불일치 없음 |
| `$table->timestamps()` | `server_default=func.now()` + `onupdate=func.now()` | DB/ORM | created_at/updated_at 자동 |
| `Carbon::now()->addDays(7)` | `datetime.now(timezone.utc) + timedelta(days=7)` | 앱 | |
| `Carbon::parse($req->date)` | `datetime.fromisoformat(s)` / Pydantic | 입력값 | API 입력 파싱 |

> **타임존**: 저장은 UTC, 표시 시점에 로컬 변환 권장.
> MySQL `DATETIME` = TZ 정보 없이 입력값 그대로 저장 / `TIMESTAMP` = UTC 저장 + 세션 TZ 변환, 범위 1970~2038. 자동갱신·UTC 일관성엔 `TIMESTAMP`, 넓은 범위·명시 관리엔 `DATETIME` + UTC 규칙.
> `func.now()` 는 MySQL에서 `CURRENT_TIMESTAMP`(=`NOW()`)로 렌더. 표준 `func.utcnow()` 는 없음 → UTC가 필요하면 DB/세션 TZ를 UTC로 두거나 파이썬 `datetime.now(timezone.utc)` 사용.

---

# 부록

---

## 부록 — 결과 추출 메서드 정리

`db.execute(stmt)`의 반환값(`Result`)에서 데이터 꺼내는 방법:

| 메서드 | 반환 타입 | 용도 |
|---|---|---|
| `.scalars().all()` | `list[Model]` | ORM 모델 여러 개 |
| `.scalars().first()` | `Model \| None` | 첫 결과 (없으면 None) |
| `.scalar_one()` | `Model` | 정확히 1개 (아니면 예외) |
| `.scalar_one_or_none()` | `Model \| None` | 0개 또는 1개 (2개+면 예외) |
| `.scalar()` | `값` | 첫 행 첫 컬럼 (집계 결과 등) |
| `.all()` | `list[Row]` | Row 튜플 (여러 컬럼 select 시) |
| `.first()` | `Row \| None` | 첫 Row 튜플 |
| `.one()` | `Row` | 정확히 1개 Row (아니면 예외) |
| `.mappings().all()` | `list[dict]` | dict 형태 (raw SQL 등) |
| `.rowcount` | `int` | UPDATE/DELETE 영향 행 수 |

### 헷갈리기 쉬운 포인트

- `select(User)` → `Row`에 `User` 객체 1개 들어있음 → `.scalars()`로 풀어야 모델 직접 받음
- `select(User, Order)` → `Row`에 `(User, Order)` 튜플 → `.all()`로 받음 (`.scalars()` 쓰면 `User`만 나옴)
- `select(func.count())` → 단일 값 → `.scalar()` 또는 `.scalar_one()`

---

## 부록 — async/sync 공통 차이점

```python
# async                              # sync
result = await db.execute(stmt)      result = db.execute(stmt)
user = await db.get(User, 1)         user = db.get(User, 1)
await db.commit()                    db.commit()
await db.refresh(user)               db.refresh(user)
async with db.begin(): ...           with db.begin(): ...
```

API 표면은 거의 동일. `await` 추가/제거만 하면 변환 가능.

### async에서 주의할 점

- **lazy loading 불가** — 관계 컬럼은 반드시 `selectinload`/`joinedload`로 미리 로드
- **expire_on_commit 끄기** (`expire_on_commit=False`) — 안 끄면 commit 후 속성 접근 시 implicit refresh가 동기 호출이라 깨짐. 이 프로젝트는 이미 꺼둠

### sync에서 주의할 점

- **lazy loading 가능** — 관계 컬럼 접근 시 자동 SQL 발생. 의도 안 했으면 N+1
- 트랜잭션 끝난 후(commit 후)에도 `expire_on_commit=False`면 속성 그대로 접근 가능

---

## 정리

- `select()` + `db.execute()` + `.scalars()` 가 2.x 표준
- PK 단건은 `db.get(Model, pk)` 가 가장 짧음
- async/sync 코드 변환은 `await` 추가/제거 정도
- 관계 eager load는 `selectinload`(IN 쿼리) / `joinedload`(JOIN) 선택
- UPDATE/DELETE 일괄은 `update()`/`delete()` 문 사용 → 조회 안 거치고 빠름
- 트랜잭션은 `db.begin()` 컨텍스트 매니저
- 관계 존재 여부는 `.any()/.has()`(EXISTS), 카운트는 스칼라 서브쿼리/`column_property`
- 동시 갱신은 원자적 `values(col=col+1)` 또는 `with_for_update()` 락
