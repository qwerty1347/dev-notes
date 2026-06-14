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

## #4 조건 단건 조회 / `User::where('email', $email)->first()`

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

---

## #5 다중 조건 (AND) / `User::where('status', 'active')->where('age', '>', 18)->get()`

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

## #6 OR 조건 / `User::where(...)->orWhere(...)->get()`

```python
# Laravel
$users = User::where('role', 'admin')->orWhere('role', 'manager')->get();

# async
async def get_managers(db: AsyncSession) -> list[User]:
    stmt = select(User).where(or_(User.role == "admin", User.role == "manager"))
    return (await db.execute(stmt)).scalars().all()

# sync
def get_managers(db: Session) -> list[User]:
    stmt = select(User).where(or_(User.role == "admin", User.role == "manager"))
    return db.execute(stmt).scalars().all()
```

---

## #7 IN 조건 / `User::whereIn('id', [1,2,3])->get()`

```python
# Laravel
$users = User::whereIn('id', [1, 2, 3])->get();

# async
async def get_users_in_ids(db: AsyncSession, ids: list[int]) -> list[User]:
    stmt = select(User).where(User.id.in_(ids))
    return (await db.execute(stmt)).scalars().all()

# sync
def get_users_in_ids(db: Session, ids: list[int]) -> list[User]:
    stmt = select(User).where(User.id.in_(ids))
    return db.execute(stmt).scalars().all()
```

---

## #8 LIKE / `User::where('name', 'like', '%kim%')->get()`

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

## #9 NULL 체크 / `User::whereNull('deleted_at')->get()`

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

## #10 정렬 / `User::orderBy('created_at', 'desc')->get()`

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

## #11 페이지네이션 / `User::skip(20)->take(10)->get()`

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

## #12 카운트 / `User::count()`

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

## #13 EXISTS / `User::where('email', $email)->exists()`

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

## #14 집계 (SUM/AVG/MAX/MIN) / `Order::sum('amount')`

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

## #15 GROUP BY / `Order::select('user_id', DB::raw('count(*) as cnt'))->groupBy('user_id')->get()`

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

## #16 JOIN / `User::join('orders', 'users.id', '=', 'orders.user_id')->get()`

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

## #17 Eager Loading (N+1 방지) / `User::with('orders')->get()`

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

---

## #18 INSERT 단건 / `User::create([...])`

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

## #19 INSERT 다건 / `User::insert([...])`

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

## #20 UPDATE 단건 (조회 후 수정) / `$user->update(['name' => 'kim'])`

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

## #21 UPDATE 다건 (조회 없이 일괄) / `User::where(...)->update([...])`

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

## #22 DELETE 단건 / `$user->delete()`

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

## #23 DELETE 다건 / `User::where(...)->delete()`

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

## #24 firstOrCreate / `User::firstOrCreate(['email' => $e], ['name' => 'kim'])`

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

## #25 트랜잭션 / `DB::transaction(function () {...})`

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

## #26 Raw SQL / `DB::select('SELECT * FROM users WHERE id = ?', [1])`

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
