# MongoDB 쿼리 가이드 (Laravel 비교)

각 항목 구성:
- **Laravel** (Eloquent — 익숙한 기준점)
- **Beanie** (async ODM, Pydantic 기반 — SQLAlchemy ORM 위치)
- **Motor** (raw async 드라이버 — dict 쿼리, 저수준)

> PyMongo(sync) = Motor 코드에서 `await` 만 제거하면 거의 동일 → 부록 참고.
> RDB ↔ Mongo 용어: 테이블=**컬렉션**, 행=**도큐먼트**, JOIN=**`$lookup`/임베드**, GROUP BY=**`$group`**, 트랜잭션=**replica set + session 필요**.

공통 셋업:
```python
# Beanie
from beanie import Document, init_beanie, PydanticObjectId, Link
from beanie.operators import (
    In, NotIn, Or, And, Not, Exists, RegEx,
    GT, GTE, LT, LTE, NE, Set, Inc, Push, Pull, AddToSet, Pop,
)
from motor.motor_asyncio import AsyncIOMotorClient
from pydantic import BaseModel

client = AsyncIOMotorClient("mongodb://localhost:27017")
db = client["app"]                                  # Motor raw 용 핸들

class User(Document):
    name: str
    email: str
    status: str = "active"
    age: int | None = None

    class Settings:
        name = "users"                              # 컬렉션 이름

# 앱 시작 시 1회
await init_beanie(database=db, document_models=[User])
```

> 섹션은 주제별로 묶임: **조회 기본 → 조건절 → 정렬/페이지네이션 → 집계/그룹 → 관계 → INSERT → UPDATE → DELETE → Upsert/동시성/트랜잭션 → 고급 → 부록**.

---

# 조회 기본

---

## #1 전체 조회 / `User::all()`

```python
# Laravel
$users = User::all();

# Beanie
async def get_all_users() -> list[User]:
    return await User.find_all().to_list()          # = User.all().to_list()

# Motor
async def get_all_users() -> list[dict]:
    return await db.users.find().to_list(length=None)
```

> `find()` 는 커서를 반환 → `.to_list()` 로 메모리에 적재. 대량이면 #16 스트리밍 사용.

---

## #2 `_id` 단건 조회 / `User::find($id)`

```python
# Laravel
$user = User::find(1);

# Beanie  (id 는 PydanticObjectId)
async def get_user(user_id: PydanticObjectId) -> User | None:
    return await User.get(user_id)

# Motor
from bson import ObjectId

async def get_user(user_id: str) -> dict | None:
    return await db.users.find_one({"_id": ObjectId(user_id)})
```

> Mongo PK 는 `_id` (기본 `ObjectId`). 문자열로 들어오면 `ObjectId(...)` 로 변환해야 매칭됨.
> Beanie `User.get()` 은 `_id` 전용.

---

## #3 단건 조회 (없으면 예외) / `User::findOrFail($id)`

```python
# Laravel
$user = User::findOrFail(1);

# Beanie
async def get_user_or_404(user_id: PydanticObjectId) -> User:
    user = await User.get(user_id)
    if user is None:
        raise NotFoundError()
    return user

# Motor
async def get_user_or_404(user_id: str) -> dict:
    user = await db.users.find_one({"_id": ObjectId(user_id)})
    if user is None:
        raise NotFoundError()
    return user
```

---

## #4 조건 단건 조회 / `User::where('email', $email)->first()`

```python
# Laravel
$user = User::where('email', $email)->first();

# Beanie
async def get_user_by_email(email: str) -> User | None:
    return await User.find_one(User.email == email)

# Motor
async def get_user_by_email(email: str) -> dict | None:
    return await db.users.find_one({"email": email})
```

> `find_one` 은 매칭 0개면 `None`, 여러 개면 그냥 첫 번째 (RDB `scalar_one` 처럼 예외 안 냄).

---

## #5 단일·다중 필드 추출 / `User::pluck('email')`

```python
# Laravel
$emails = User::where('status', 'active')->pluck('email');     # ['a@x', 'b@x', ...]
$map    = User::pluck('name', 'id');                           # [id => name]
$rows   = User::select('id', 'name', 'email')->get();          # 여러 필드

# ── 단일 필드 → 값 리스트 ──────────────────────────────────────
# Beanie — projection 모델
class EmailOnly(BaseModel):
    email: str

async def active_emails() -> list[str]:
    docs = await User.find(User.status == "active").project(EmailOnly).to_list()
    return [d.email for d in docs]                             # ['a@x', 'b@x', ...]

# Motor — projection (1=포함, _id 는 명시적으로 0)
async def active_emails_motor() -> list[str]:
    cur = db.users.find({"status": "active"}, {"email": 1, "_id": 0})
    return [d["email"] async for d in cur]

# 중복 제거면 distinct (단일 필드 전용, 결과는 set 처럼)
async def distinct_statuses() -> list[str]:
    return await User.distinct(User.status)                    # Motor: db.users.distinct("status")

# ── 2개 필드 → key=>value 맵 (pluck('name','id') 대응) ─────────
# Beanie
async def id_name_map() -> dict[PydanticObjectId, str]:
    docs = await User.find_all().project(IdName).to_list()     # IdName 정의는 아래
    return {d.id: d.name for d in docs}

# Motor — _id 는 ObjectId 라 str 키가 필요하면 변환
async def id_name_map_motor() -> dict[str, str]:
    cur = db.users.find({}, {"name": 1})                       # _id 는 기본 포함
    return {str(d["_id"]): d["name"] async for d in cur}

# ── 다중 필드 추출 ────────────────────────────────────────────
# (a) Beanie — projection 모델 (타입 안전, 추천)
class UserBrief(BaseModel):
    id: PydanticObjectId = Field(alias="_id")                  # _id → id 매핑
    name: str
    email: str
    model_config = {"populate_by_name": True}

class IdName(BaseModel):
    id: PydanticObjectId = Field(alias="_id")
    name: str
    model_config = {"populate_by_name": True}

async def user_briefs() -> list[UserBrief]:
    return await User.find(User.status == "active").project(UserBrief).to_list()

# (b) Beanie — 모델 없이 dict 로 (모델 컬렉션 핸들로 내려가서 projection)
async def user_dicts() -> list[dict]:
    coll = User.get_motor_collection()
    return await coll.find({"status": "active"}, {"name": 1, "email": 1}).to_list(None)

# (c) Motor — dict 리스트 (JSON 응답에 바로 쓰기 좋음)
async def user_dicts_motor() -> list[dict]:
    cur = db.users.find({"status": "active"}, {"name": 1, "email": 1, "_id": 1})
    out = []
    async for d in cur:
        d["_id"] = str(d["_id"])                               # ObjectId → str (직렬화용)
        out.append(d)
    return out

# (d) 중첩/임베드 필드만 뽑기 — 점 표기법
#     문서: {name, addr: {city, zip}}  →  name, addr.city 만
async def name_city() -> list[dict]:
    cur = db.users.find({}, {"name": 1, "addr.city": 1, "_id": 0})
    return await cur.to_list(None)                             # [{'name':..,'addr':{'city':..}}, ...]

# (e) 배열 필드 일부만 — $slice (배열 앞 3개만)
async def first_tags() -> list[dict]:
    return await db.users.find({}, {"name": 1, "tags": {"$slice": 3}}).to_list(None)
```

> - **단일 필드** → projection 후 `[d.field for d in docs]`. 중복 제거가 목적이면 `distinct` (단일 필드 전용).
> - **2개 (key⇒value)** → projection 후 dict 컴프리헨션. Motor 의 `_id` 는 `ObjectId` 라 문자열 키가 필요하면 `str(...)` 변환.
> - **다중 필드** → Beanie 는 **projection 모델**(`.project(Model)`)이 타입 안전해서 1순위. 모델 없이 빠르게면 Motor dict projection.
> - **projection 문법**: `{"field": 1}` = 포함만 / `{"field": 0}` = 제외만 — 한 쿼리에서 1과 0을 **섞을 수 없음**
>   (예외: `_id` 만 항상 명시적으로 끌 수 있음). `_id` 는 기본 포함이라 빼려면 `{"_id": 0}`.
> - **중첩 필드**는 점 표기법(`"addr.city": 1`), **배열 일부**는 `{"$slice": n}`.
> - Beanie projection 모델 필드명이 문서와 다르면 `Field(alias="_id")` + `populate_by_name=True` 로 매핑.
> - projection 으로 필요한 필드만 가져오면 네트워크/메모리 절약 + 인덱스만으로 처리되는 **covered query** 가능.
> - (참고) 결과 타입 비교는 **부록 — Beanie vs Motor(raw) 정리** 참고.

---

# 조건절 (filter)

---

## #6 다중 조건 (AND) / `->where('status','active')->where('age','>',18)`

```python
# Laravel
$users = User::where('status', 'active')->where('age', '>', 18)->get();

# Beanie — 인자 나열 = AND
async def get_active_adults() -> list[User]:
    return await User.find(User.status == "active", User.age > 18).to_list()

# Motor — 같은 dict 키 나열 = AND
async def get_active_adults() -> list[dict]:
    return await db.users.find({"status": "active", "age": {"$gt": 18}}).to_list(None)
```

> Beanie `find(a, b)` 콤마 = AND. Motor 는 dict 의 키들이 자동 AND.

---

## #7 OR 조건 / `->where(...)->orWhere(...)`

```python
# Laravel
$users = User::where('role', 'admin')->orWhere('role', 'manager')->get();

# Beanie
async def get_managers() -> list[User]:
    return await User.find(Or(User.role == "admin", User.role == "manager")).to_list()

# Motor
async def get_managers() -> list[dict]:
    return await db.users.find(
        {"$or": [{"role": "admin"}, {"role": "manager"}]}
    ).to_list(None)
```

---

## #8 AND·OR 혼합 조건 / `->where('status','active')->where(fn($q)=>$q->where(...)->orWhere(...))`

시나리오: **`status = 'active' AND (role = 'admin' OR age >= 18)`**

```python
# Laravel — orWhere 를 클로저로 감싸야 괄호가 생김
$users = User::where('status', 'active')
    ->where(fn ($q) => $q->where('role','admin')->orWhere('age','>=',18))
    ->get();

# Beanie — Or() 가 곧 괄호
async def get_active_admins_or_adults() -> list[User]:
    return await User.find(
        User.status == "active",
        Or(User.role == "admin", User.age >= 18),
    ).to_list()

# Motor — $and 안에 $or 를 한 원소로
async def get_active_admins_or_adults() -> list[dict]:
    return await db.users.find({
        "status": "active",
        "$or": [{"role": "admin"}, {"age": {"$gte": 18}}],
    }).to_list(None)
```

> `(A OR B) AND (C OR D)` 처럼 OR 그룹이 둘이면 **dict 최상위에 `$or` 키는 하나뿐**이라 `$and` 로 감싸야 함:
> ```python
> {"$and": [
>     {"$or": [{"role": "admin"}, {"role": "manager"}]},
>     {"$or": [{"status": "active"}, {"status": "trial"}]},
> ]}
> # Beanie: And(Or(...), Or(...))
> ```
> 같은 필드에 OR 이 여러 개면 `$or` 대신 `$in` 이 더 깔끔: `{"role": {"$in": ["admin","manager"]}}` (#9).

---

## #9 IN 조건 / `User::whereIn('id', [...])`

```python
# Laravel
$users = User::whereIn('id', [1, 2, 3])->get();

# Beanie
async def get_users_in_ids(ids: list[PydanticObjectId]) -> list[User]:
    return await User.find(In(User.id, ids)).to_list()

# Motor
async def get_users_in_ids(ids: list[str]) -> list[dict]:
    oids = [ObjectId(i) for i in ids]
    return await db.users.find({"_id": {"$in": oids}}).to_list(None)
```

> 반대는 `NotIn` / `{"$nin": [...]}`.

cf) 특정 seller 검색 시 지정된 blind seller 리스트에 포함되어 있지 않은 seller 검색 방법

{
    "seller": {
        "$nin": ["A", "B", ...]
    },
    "$and": [
        {
            "seller": "C"
        }
    ]
}

---

## #10 부분일치 / `User::where('name','like','%kim%')` → `$regex`

```python
# Laravel
$users = User::where('name', 'like', '%kim%')->get();

# Beanie
async def search_users(keyword: str) -> list[User]:
    return await User.find(RegEx(User.name, keyword, options="i")).to_list()  # i = 대소문자 무시

# Motor
async def search_users(keyword: str) -> list[dict]:
    return await db.users.find(
        {"name": {"$regex": keyword, "$options": "i"}}
    ).to_list(None)
```

> 정규식 특수문자가 섞일 수 있으면 `re.escape(keyword)` 권장.
> prefix 매칭(`kim%`)은 `^kim` 정규식으로 → 인덱스 탈 수 있음. `%kim%` (앞 와일드카드)는 인덱스 못 탐.
> 전문 검색이 필요하면 text index + `{"$text": {"$search": kw}}` 또는 Atlas Search 고려.

---

## #11 범위·날짜 조건 / `whereBetween` · `whereDate`

```python
# Laravel
$users  = User::whereBetween('age', [20, 30])->get();
$orders = Order::whereBetween('created_at', [$start, $end])->get();

from datetime import datetime, timezone

# Beanie
async def users_in_age_range(lo: int, hi: int) -> list[User]:
    return await User.find(User.age >= lo, User.age <= hi).to_list()

async def orders_between(start: datetime, end: datetime) -> list[Order]:
    return await Order.find(Order.created_at >= start, Order.created_at < end).to_list()

# Motor
async def users_in_age_range(lo: int, hi: int) -> list[dict]:
    return await db.users.find({"age": {"$gte": lo, "$lte": hi}}).to_list(None)

async def orders_between(start: datetime, end: datetime) -> list[dict]:
    return await db.orders.find(
        {"created_at": {"$gte": start, "$lt": end}}
    ).to_list(None)
```

> "특정 날짜" 조회는 `$gte 그날 00:00 AND $lt 다음날 00:00` 범위로. Mongo 엔 RDB `DATE()` 같은 게 없고
> aggregation `$dateTrunc` 를 써야 하므로 범위 비교가 단순하고 인덱스도 탐.
> datetime 은 **UTC aware** 로 저장 권장 (Mongo 는 내부적으로 UTC ms 로 저장).

---

## #12 NULL / 존재 체크 / `whereNull` · `whereNotNull`

```python
# Laravel
$users = User::whereNull('deleted_at')->get();

# Beanie
async def get_active() -> list[User]:
    return await User.find(User.deleted_at == None).to_list()   # noqa: E711

# Motor
async def get_active() -> list[dict]:
    return await db.users.find({"deleted_at": None}).to_list(None)
```

> Mongo 주의: `{"field": None}` 은 **"field 가 null 이거나 아예 없음"** 둘 다 매칭.
> "필드가 존재하느냐"와 "값이 null 이냐"를 구분하려면 `$exists` 사용:
> ```python
> {"deleted_at": {"$exists": False}}             # 필드 자체가 없음
> {"deleted_at": {"$ne": None}}                  # 값이 있고 null 아님
> # Beanie: Exists(User.deleted_at, False) / User.deleted_at != None
> ```

---

## #13 Soft Delete / Laravel `use SoftDeletes;`

```python
# Laravel — deleted_at 자동 필터/세팅
$users = User::all();                  # deleted_at IS NULL 자동
$user->delete();                       # UPDATE deleted_at = now()
$users = User::withTrashed()->get();   # 삭제 포함
$user->restore();                      # deleted_at = NULL

from datetime import datetime, timezone

class User(Document):
    name: str
    deleted_at: datetime | None = None
    class Settings:
        name = "users"

# 기본 조회 (= Laravel 기본: 살아있는 것만)
async def list_users() -> list[User]:
    return await User.find(User.deleted_at == None).to_list()   # noqa: E711

# soft delete (= $user->delete())
async def soft_delete(user_id: PydanticObjectId) -> bool:
    res = await User.find_one(User.id == user_id, User.deleted_at == None).update(
        Set({User.deleted_at: datetime.now(timezone.utc)})
    )
    return res.modified_count == 1

# 복구 (= restore())
async def restore(user_id: PydanticObjectId) -> None:
    await User.find_one(User.id == user_id).update(Set({User.deleted_at: None}))

# withTrashed = 조건 안 검 / onlyTrashed = User.deleted_at != None
```

> Laravel 같은 자동 글로벌 스코프는 Beanie 에 없음 → 매 쿼리에 `deleted_at == None` 명시하거나
> 베이스 쿼리 헬퍼/리포지토리로 강제. (대안: 별도 `archived_users` 컬렉션으로 이동시키는 패턴도 흔함)

---

# 정렬·페이지네이션·결과 제어

---

## #14 정렬 / `User::orderBy('created_at','desc')`

```python
# Laravel
$users = User::orderBy('created_at', 'desc')->get();

# Beanie  (- 접두사 = 내림차순)
async def get_users_sorted() -> list[User]:
    return await User.find_all().sort(-User.created_at).to_list()
    # 여러 키: .sort(-User.status, +User.created_at) 또는 .sort([("status",-1),("created_at",1)])

# Motor  (1 = asc, -1 = desc)
async def get_users_sorted() -> list[dict]:
    return await db.users.find().sort("created_at", -1).to_list(None)
    # 여러 키: .sort([("status", -1), ("created_at", 1)])
```

---

## #15 페이지네이션 / `User::skip(20)->take(10)`

```python
# Laravel
$users = User::skip(20)->take(10)->get();

# Beanie
async def get_page(page: int, size: int) -> list[User]:
    return await User.find_all().skip((page - 1) * size).limit(size).to_list()

# Motor
async def get_page(page: int, size: int) -> list[dict]:
    return await db.users.find().skip((page - 1) * size).limit(size).to_list(None)
```

> 깊은 페이지(`skip` 큰 값)는 느림 → 대용량은 **range pagination** 권장:
> `find(User.id > last_seen_id).sort(+User.id).limit(size)` (정렬 키 기준 커서 방식).

---

## #16 대량 스트리밍 / `User::chunk(500, ...)` · `cursor()`

```python
# Laravel
User::chunk(500, fn ($users) => ...);
User::cursor()->each(fn ($u) => ...);

# Beanie — async for (커서 스트리밍, 메모리 일정)
async def process_all() -> None:
    async for user in User.find(User.status == "active"):
        handle(user)                                # to_list() 안 함 → 한 건씩

# Motor — 커서 직접 순회
async def process_all() -> None:
    async for doc in db.users.find({"status": "active"}):
        handle(doc)

# Motor — 배치 단위 (Laravel chunk 와 유사)
async def process_in_chunks() -> None:
    cur = db.users.find({}).batch_size(500)
    batch: list[dict] = []
    async for doc in cur:
        batch.append(doc)
        if len(batch) == 500:
            handle_batch(batch); batch = []
    if batch:
        handle_batch(batch)
```

> `.to_list(None)` 은 전부 메모리 적재 → 대량이면 `async for` 로 스트리밍.
> 오래 도는 커서는 기본 10분 후 타임아웃 → 긴 작업은 `no_cursor_timeout=True` (쓰고 나면 꼭 닫기) 또는 range pagination.

---

## #17 DISTINCT / `Order::select('status')->distinct()`

```python
# Laravel
$statuses = Order::select('status')->distinct()->get();

# Beanie
async def distinct_statuses() -> list[str]:
    return await Order.distinct(Order.status, Order.amount > 0)   # 2번째 인자 = 필터(선택)

# Motor
async def distinct_statuses() -> list[str]:
    return await db.orders.distinct("status", {"amount": {"$gt": 0}})
```

> `distinct` 결과는 단일 배열 16MB 제한 있음. 카디널리티 크면 `$group` aggregation 으로 (#21).

---

# 집계·그룹

---

## #18 카운트 / `User::count()`

```python
# Laravel
$count = User::count();
$activeCount = User::where('status', 'active')->count();

# Beanie
async def count_users() -> int:
    return await User.find_all().count()

async def count_active() -> int:
    return await User.find(User.status == "active").count()

# Motor
async def count_users() -> int:
    return await db.users.count_documents({})
```

> 전체 건수만 대략 필요하고 빠르게 → `estimated_document_count()` (메타데이터 기반, 필터 불가).
> 정확한 필터 카운트는 `count_documents(filter)`.

---

## #19 존재 여부 / `User::where(...)->exists()`

```python
# Laravel
$exists = User::where('email', $email)->exists();

# Beanie
async def email_exists(email: str) -> bool:
    return await User.find_one(User.email == email) is not None

# Motor — count 보다 find_one + projection 이 보통 빠름
async def email_exists(email: str) -> bool:
    return await db.users.find_one({"email": email}, {"_id": 1}) is not None
```

---

## #20 집계 (SUM/AVG/MAX/MIN) / `Order::sum('amount')`

```python
# Laravel
$total = Order::sum('amount');
$avg   = Order::avg('amount');

# Beanie — 헬퍼 제공
async def total_amount() -> float:
    return await Order.find(Order.status == "paid").sum(Order.amount) or 0
    # .avg(Order.amount) / .max(...) / .min(...) 도 동일

# Motor — aggregation $group (_id: None = 전체)
async def total_amount() -> float:
    cur = db.orders.aggregate([
        {"$match": {"status": "paid"}},
        {"$group": {"_id": None, "total": {"$sum": "$amount"}}},
    ])
    rows = await cur.to_list(None)
    return rows[0]["total"] if rows else 0
```

---

## #21 GROUP BY / `->groupBy('user_id')`

```python
# Laravel
$grouped = Order::select('user_id', DB::raw('count(*) as cnt'), DB::raw('sum(amount) as total'))
    ->groupBy('user_id')->get();

# 결과 모델 (Beanie projection 용)
class UserTotal(BaseModel):
    user_id: PydanticObjectId
    cnt: int
    total: float

pipeline = [
    {"$group": {
        "_id": "$user_id",                          # GROUP BY user_id
        "cnt": {"$sum": 1},                         # COUNT(*)
        "total": {"$sum": "$amount"},               # SUM(amount)
    }},
    {"$project": {"_id": 0, "user_id": "$_id", "cnt": 1, "total": 1}},
]

# Beanie
async def order_stats() -> list[UserTotal]:
    return await Order.aggregate(pipeline, projection_model=UserTotal).to_list()

# Motor
async def order_stats() -> list[dict]:
    return await db.orders.aggregate(pipeline).to_list(None)
```

> `$group._id` 가 GROUP BY 키. 복합 키는 `{"_id": {"user_id": "$user_id", "status": "$status"}}`.
> 집계 연산자: `$sum`, `$avg`, `$min`, `$max`, `$first`, `$last`, `$push`(배열로 모음), `$addToSet`(중복 제거).

---

## #22 HAVING / `->groupBy(...)->having('total','>',10000)`

```python
# Laravel
$rows = Order::select('user_id', DB::raw('SUM(amount) as total'))
    ->groupBy('user_id')->having('total', '>', 10000)->get();

# Mongo — $group 다음 $match (그게 곧 HAVING)
pipeline = [
    {"$group": {"_id": "$user_id", "total": {"$sum": "$amount"}}},
    {"$match": {"total": {"$gt": 10000}}},          # ★ 그룹핑 "후" 필터 = HAVING
]

# Beanie
async def big_spenders() -> list[dict]:
    return await Order.aggregate(pipeline).to_list()

# Motor
async def big_spenders() -> list[dict]:
    return await db.orders.aggregate(pipeline).to_list(None)
```

> WHERE = `$group` **앞**의 `$match` (행 필터), HAVING = `$group` **뒤**의 `$match` (집계값 필터).
> 앞 `$match` 를 최대한 일찍 둬야 인덱스 타고 빠름.

---

## #23 윈도우 함수 (순위·누적) / `ROW_NUMBER() OVER (...)`

```python
# Laravel
DB::table('orders')->selectRaw(
  'user_id, amount, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn'
)->get();

# Mongo — $setWindowFields (4.4+)
pipeline = [
    {"$setWindowFields": {
        "partitionBy": "$user_id",
        "sortBy": {"amount": -1},
        "output": {
            "rn":  {"$documentNumber": {}},                 # ROW_NUMBER()
            "rnk": {"$rank": {}},                            # RANK()
            "cum": {"$sum": "$amount",                       # 누적합 (running total)
                    "window": {"documents": ["unbounded", "current"]}},
        },
    }},
]

# "유저별 최고액 1건만" = 윈도우 후 $match
top_per_user = pipeline + [{"$match": {"rn": 1}}]

# Beanie / Motor 동일
async def ranked(): return await Order.aggregate(pipeline).to_list()
async def top_per_user_q(): return await db.orders.aggregate(top_per_user).to_list(None)
```

> `$rank`/`$denseRank`/`$documentNumber` = SQL `RANK`/`DENSE_RANK`/`ROW_NUMBER`.
> 윈도우 결과로 필터하려면 그 뒤에 `$match` (SQL 에서 윈도우를 WHERE 에 못 쓰는 것과 동일 패턴).

---

# 관계 (참조 / 임베드)

> Mongo 엔 JOIN 이 1급이 아님. **임베드(문서 안에 포함)** 가 1순위, 정 필요하면 **`$lookup`** 또는 **참조+populate**.

---

## #24 $lookup (JOIN 대응) / `User::join('orders', ...)`

```python
# Laravel
$rows = User::join('orders', 'users.id', '=', 'orders.user_id')
    ->select('users.*', 'orders.amount')->get();

# Mongo — $lookup (left outer join 과 유사, 결과는 배열로 붙음)
pipeline = [
    {"$lookup": {
        "from": "orders",
        "localField": "_id",
        "foreignField": "user_id",
        "as": "orders",                             # users 문서에 orders: [...] 배열로 추가
    }},
    {"$unwind": "$orders"},                         # 1행:1주문 으로 펴기 (INNER JOIN 효과)
    {"$project": {"name": 1, "amount": "$orders.amount"}},
]

# Beanie / Motor
async def users_with_orders() -> list[dict]:
    return await User.aggregate(pipeline).to_list()
    # Motor: await db.users.aggregate(pipeline).to_list(None)
```

> `$lookup` 결과는 항상 **배열**. 1:1 처럼 펴려면 `$unwind`.
> 조건부 조인은 `$lookup` 의 `let` + `pipeline` 서브파이프라인 형태 사용 (#41).
> `$lookup` 은 비싸다 — 자주 같이 읽는 데이터는 처음부터 **임베드**(#25)가 정석.

---

## #25 임베드 vs 참조 + populate / `User::with('orders')`

```python
# Laravel
$users = User::with('orders')->get();               # 참조 + eager load

# 방식 A) 임베드 — 자식을 부모 문서 안에 그대로 (조인 불필요, 가장 흔함)
class Order(BaseModel):                              # Document 아님 — 서브문서
    amount: float
    status: str

class User(Document):
    name: str
    orders: list[Order] = []                        # 문서 안에 배열로 저장
    class Settings:
        name = "users"
# 조회 한 방 → user.orders 바로 접근 (조인/N+1 없음)
users = await User.find_all().to_list()

# 방식 B) 참조 + Beanie Link (RDB FK + eager load 에 가장 근접)
class Order(Document):
    amount: float
    status: str
    class Settings: name = "orders"

class User(Document):
    name: str
    orders: list[Link[Order]] = []
    class Settings: name = "users"

# eager load (= with('orders')) : fetch_links=True → 자동 $lookup
async def users_with_orders() -> list[User]:
    return await User.find_all(fetch_links=True).to_list()

# 단건만 나중에 채우기
user = await User.get(uid)
await user.fetch_link(User.orders)                  # 필요 시점에 로드
```

> 선택 기준: **같이 자주 읽고 무한정 안 커지면 임베드**, **독립적으로 크거나 공유되면 참조+Link**.
> `fetch_links=True` 미사용 시 `user.orders` 는 `Link` 객체(미해결) — RDB async lazy-load 막힘과 같은 함정.

---

## #26 관계 존재 조건 / `whereHas` · `doesntHave`

```python
# Laravel
$users = User::whereHas('orders')->get();                       # 주문 있는 유저
$users = User::whereHas('orders', fn($q)=>$q->where('status','paid'))->get();
$users = User::doesntHave('orders')->get();                     # 주문 없는 유저

# 임베드 모델이면 — 그냥 배열 조건
async def users_with_paid_embedded() -> list[User]:
    # orders 배열 안에 status=='paid' 인 원소가 하나라도 있으면
    return await User.find({"orders": {"$elemMatch": {"status": "paid"}}}).to_list()

async def users_without_orders_embedded() -> list[User]:
    return await User.find({"orders": {"$size": 0}}).to_list()

# 참조 모델이면 — $lookup 후 존재 여부로 필터
pipeline = [
    {"$lookup": {
        "from": "orders", "localField": "_id", "foreignField": "user_id",
        "pipeline": [{"$match": {"status": "paid"}}],   # 조건부 whereHas
        "as": "_paid",
    }},
    {"$match": {"_paid": {"$ne": []}}},              # 있으면 → whereHas
    # doesntHave 면:  {"$match": {"_paid": {"$eq": []}}}
    {"$project": {"_paid": 0}},
]
async def users_with_paid_ref() -> list[User]:
    return await User.aggregate(pipeline, projection_model=User).to_list()
```

> `$elemMatch` = 배열 안에 "조건을 모두 만족하는 원소 1개 이상". 단일 조건이면 `{"orders.status": "paid"}` 만으로도 됨
> (단, 서로 다른 원소에 분산 매칭될 수 있어 복합 조건엔 `$elemMatch` 필수).
> 개수 비교(`has('orders','>=',3)`)는 `{"$expr": {"$gte": [{"$size": "$orders"}, 3]}}` 또는 `$lookup` 후 `$size`.

---

## #27 관계 카운트 / `User::withCount('orders')`

```python
# Laravel
$users = User::withCount('orders')->get();          # $u->orders_count

# 임베드면 — $size 로 즉시
pipeline_embed = [
    {"$addFields": {"orders_count": {"$size": {"$ifNull": ["$orders", []]}}}},
]

# 참조면 — $lookup 후 개수만 (문서 안 끌고옴 → pipeline 으로 count 만)
pipeline_ref = [
    {"$lookup": {
        "from": "orders", "localField": "_id", "foreignField": "user_id",
        "pipeline": [
            {"$match": {"status": "paid"}},                 # 조건부 카운트
            {"$count": "n"},
        ],
        "as": "_c",
    }},
    {"$addFields": {"paid_count": {"$ifNull": [{"$arrayElemAt": ["$_c.n", 0]}, 0]}}},
    {"$project": {"_c": 0}},
]

async def users_with_count() -> list[dict]:
    return await User.aggregate(pipeline_ref).to_list()
    # Motor: await db.users.aggregate(pipeline_ref).to_list(None)
```

> 카운트로 정렬/필터까지 하려면 위처럼 `$addFields` 로 필드화한 뒤 `$sort`/`$match` 추가.
> 임베드면 `$size` 한 줄 — 참조보다 압도적으로 쌈 (조인 안 함).

---

# INSERT

---

## #28 INSERT 단건 / `User::create([...])`

```python
# Laravel
$user = User::create(['name' => 'kim', 'email' => 'a@b.com']);

# Beanie
async def create_user(name: str, email: str) -> User:
    user = User(name=name, email=email)
    await user.insert()                             # = await user.create()
    return user                                     # user.id 자동 채워짐

# Motor
async def create_user(name: str, email: str) -> str:
    res = await db.users.insert_one({"name": name, "email": email})
    return str(res.inserted_id)
```

---

## #29 INSERT 다건 / `User::insert([...])`

```python
# Laravel
User::insert([['name'=>'a','email'=>'a@x'], ['name'=>'b','email'=>'b@x']]);

# Beanie
async def bulk_create(rows: list[dict]) -> None:
    await User.insert_many([User(**r) for r in rows])

# Motor
async def bulk_create(rows: list[dict]) -> None:
    await db.users.insert_many(rows)                # ordered=False 면 일부 실패해도 나머지 진행
```

> `insert_many(..., ordered=False)` — 중복키 등으로 한 건 실패해도 나머지는 삽입. 순서 보장 X, 처리량 ↑.

---

# UPDATE

---

## #30 UPDATE 단건 (조회 후 수정) / `$user->update([...])`

```python
# Laravel
$user = User::find(1);
$user->update(['name' => 'kim']);

# Beanie — 객체 수정 후 save / 또는 set
async def rename_user(user_id: PydanticObjectId, name: str) -> User | None:
    user = await User.get(user_id)
    if user is None:
        return None
    user.name = name
    await user.save()                               # 또는: await user.set({User.name: name})
    return user

# Motor
async def rename_user(user_id: str, name: str) -> bool:
    res = await db.users.update_one(
        {"_id": ObjectId(user_id)}, {"$set": {"name": name}}
    )
    return res.modified_count == 1
```

> `save()` 는 문서 전체 치환에 가까움(변경 추적). 특정 필드만 원자적으로 → `.set({...})` / `$set`.

---

## #31 UPDATE 다건 (조회 없이 일괄) / `User::where(...)->update([...])`

```python
# Laravel
User::where('status', 'pending')->update(['status' => 'active']);

# Beanie
async def activate_pending() -> int:
    res = await User.find(User.status == "pending").update(
        Set({User.status: "active"})
    )
    return res.modified_count

# Motor
async def activate_pending() -> int:
    res = await db.users.update_many(
        {"status": "pending"}, {"$set": {"status": "active"}}
    )
    return res.modified_count
```

> `update_one` vs `update_many` 구분 명확히 — Mongo 는 기본 `update_one`(1건만). 일괄은 반드시 `update_many`.

---

## #32 원자적 증감 / 배열 연산 / `->increment('views')`

```python
# Laravel
Post::where('id',$id)->increment('views');
Post::where('id',$id)->decrement('stock', $qty);

# Beanie — $inc (원자적, 조회 안 함 → 레이스 안전)
async def add_view(post_id: PydanticObjectId, n: int = 1) -> None:
    await Post.find(Post.id == post_id).update(Inc({Post.views: n}))

# 재고 차감 + 음수 방지 (조건부 → 성공 여부 modified_count)
async def decrement_stock(pid: PydanticObjectId, qty: int) -> bool:
    res = await Product.find(Product.id == pid, Product.stock >= qty).update(
        Inc({Product.stock: -qty})
    )
    return res.modified_count == 1

# 배열 연산 (Mongo 특화) — Push / Pull / AddToSet
await User.find(User.id == uid).update(Push({User.tags: "vip"}))        # 끝에 추가
await User.find(User.id == uid).update(AddToSet({User.tags: "vip"}))    # 중복 없이 추가
await User.find(User.id == uid).update(Pull({User.tags: "vip"}))        # 값 제거

# Motor
async def add_view(post_id: str, n: int = 1) -> None:
    await db.posts.update_one({"_id": ObjectId(post_id)}, {"$inc": {"views": n}})
# $push / $addToSet / $pull / $pop 동일
```

> ❌ `doc = get(); doc.views += 1; save()` — 사이에 다른 갱신 끼면 **lost update**.
> ✅ `$inc` / `$push` / `$pull` 등은 **서버에서 원자적**으로 처리 → 레이스 없음.
> 하한 보장은 `find(stock >= qty)` 를 같이 걸고 `modified_count` 로 판정 (락 불필요).

---

# DELETE

---

## #33 DELETE 단건 / `$user->delete()`

```python
# Laravel
User::find(1)->delete();

# Beanie
async def delete_user(user_id: PydanticObjectId) -> bool:
    user = await User.get(user_id)
    if user is None:
        return False
    await user.delete()
    return True

# Motor
async def delete_user(user_id: str) -> bool:
    res = await db.users.delete_one({"_id": ObjectId(user_id)})
    return res.deleted_count == 1
```

---

## #34 DELETE 다건 / `User::where(...)->delete()`

```python
# Laravel
User::where('status', 'inactive')->delete();

# Beanie
async def delete_inactive() -> int:
    res = await User.find(User.status == "inactive").delete()
    return res.deleted_count

# Motor
async def delete_inactive() -> int:
    res = await db.users.delete_many({"status": "inactive"})
    return res.deleted_count
```

---

# Upsert·동시성·트랜잭션

---

## #35 firstOrCreate / `User::firstOrCreate(['email'=>$e], ['name'=>'kim'])`

```python
# Laravel
$user = User::firstOrCreate(['email' => $email], ['name' => 'kim']);

# Beanie — find_one or insert
async def first_or_create(email: str, name: str) -> User:
    user = await User.find_one(User.email == email)
    if user is None:
        user = User(email=email, name=name)
        await user.insert()
    return user

# Motor — 원자적: find_one_and_update + $setOnInsert + upsert
from pymongo import ReturnDocument

async def first_or_create(email: str, name: str) -> dict:
    return await db.users.find_one_and_update(
        {"email": email},
        {"$setOnInsert": {"email": email, "name": name}},   # 새로 만들 때만 세팅
        upsert=True,
        return_document=ReturnDocument.AFTER,
    )
```

> 동시성 중요하면 `email` 에 **unique index** + `find_one_and_update(upsert=True)` 가 안전.
> Beanie 의 find→insert 분기는 동시 요청 시 둘 다 insert 시도 가능 → unique index 로 방어 + 예외 처리.

---

## #36 updateOrCreate / `User::updateOrCreate(['email'=>$e], [...])`

```python
# Laravel
$user = User::updateOrCreate(['email' => $email], ['name' => 'kim', 'plan' => 'pro']);

# Beanie — upsert 헬퍼
async def update_or_create(email: str, attrs: dict) -> None:
    await User.find_one(User.email == email).upsert(
        Set(attrs),                                 # 있으면 이걸로 갱신
        on_insert=User(email=email, **attrs),       # 없으면 이 문서 생성
    )

# Motor — $set + upsert (있으면 갱신, 없으면 생성)
async def update_or_create(email: str, attrs: dict) -> dict:
    return await db.users.find_one_and_update(
        {"email": email},
        {"$set": attrs, "$setOnInsert": {"email": email}},
        upsert=True,
        return_document=ReturnDocument.AFTER,
    )
```

> `$set` 은 항상 적용(갱신), `$setOnInsert` 는 새 문서일 때만 → 둘 조합이 updateOrCreate 의 정석.
> 동시 호출 안전하려면 매칭 키(`email`)에 unique index 필수.

---

## #37 동시성 제어 / 비관적 락 대안

> Mongo 엔 RDB `SELECT ... FOR UPDATE` 같은 행 락이 없음. 대신 **단일 문서 연산은 원자적**이라는 점을 활용.

```python
# Laravel (RDB) — lockForUpdate
DB::transaction(fn() => {
    $acct = Account::where('id',$id)->lockForUpdate()->first();
    $acct->balance -= 100; $acct->save();
});

# 패턴 A) 조건부 원자 갱신 (락 없이 — 대부분 이걸로 충분)
async def withdraw(account_id: PydanticObjectId, amount: int) -> bool:
    res = await Account.find(
        Account.id == account_id, Account.balance >= amount
    ).update(Inc({Account.balance: -amount}))
    return res.modified_count == 1                  # 0 = 잔액 부족 → 차감 안 됨

# 패턴 B) 낙관적 동시성 (version 필드로 충돌 감지)
async def update_with_version(doc_id, expected_ver: int, changes: dict) -> bool:
    res = await Item.find(Item.id == doc_id, Item.version == expected_ver).update(
        Set({**changes, Item.version: expected_ver + 1})
    )
    return res.modified_count == 1                  # 0 = 그 사이 누가 바꿈 → 재시도

# 패턴 C) find_one_and_update — 읽기+갱신을 한 원자 연산으로
doc = await db.accounts.find_one_and_update(
    {"_id": oid, "balance": {"$gte": amt}},
    {"$inc": {"balance": -amt}},
    return_document=ReturnDocument.AFTER,
)
if doc is None:  ...                                 # 조건 불충족
```

> 핵심: **여러 문서에 걸친 불변식**이 아니면 보통 락 필요 없음 — 조건부 `$inc`/`find_one_and_update` 로 원자 처리.
> 진짜 다중 문서 일관성이 필요하면 #38 트랜잭션.

---

## #38 트랜잭션 / `DB::transaction(...)`

```python
# Laravel
DB::transaction(function () {
    User::create([...]); Order::create([...]);
});

# Beanie / Motor — replica set 필수 (단일 mongod standalone 은 트랜잭션 불가)
async def transfer() -> None:
    async with await client.start_session() as s:
        async with s.start_transaction():
            await User(name="a").insert(session=s)
            await Order(amount=100).insert(session=s)
        # 블록 정상 종료 → commit, 예외 → 자동 abort

# Motor raw
async def transfer_raw() -> None:
    async with await client.start_session() as s:
        async with s.start_transaction():
            await db.users.insert_one({"name": "a"}, session=s)
            await db.orders.insert_one({"amount": 100}, session=s)
```

> **모든 연산에 `session=s` 를 넘겨야** 트랜잭션에 포함됨 (빠뜨리면 트랜잭션 밖에서 실행됨 — 흔한 버그).
> 트랜잭션은 replica set / sharded cluster 에서만. 로컬 standalone 이면 `mongod --replSet` + `rs.initiate()` 필요.
> 트랜잭션은 비용이 큼 — 가능하면 #37 단일 문서 원자 연산으로 설계(임베드로 한 문서에 모으면 트랜잭션 자체가 불필요).

---

# 고급

---

## #39 Raw Aggregation Pipeline / `DB::raw(...)`

```python
# Laravel raw 와 대응 — 파이프라인이 곧 "쿼리 언어"
pipeline = [
    {"$match":   {"status": "paid"}},                       # WHERE
    {"$group":   {"_id": "$user_id", "total": {"$sum": "$amount"}}},  # GROUP BY + SUM
    {"$match":   {"total": {"$gt": 10000}}},                # HAVING
    {"$sort":    {"total": -1}},                            # ORDER BY
    {"$limit":   20},                                       # LIMIT
    {"$project": {"_id": 0, "user_id": "$_id", "total": 1}},# SELECT
]

# Beanie (projection_model 로 타입 보장)
class Row(BaseModel):
    user_id: PydanticObjectId
    total: float

async def heavy_buyers() -> list[Row]:
    return await Order.aggregate(pipeline, projection_model=Row).to_list()

# Motor
async def heavy_buyers_raw() -> list[dict]:
    return await db.orders.aggregate(pipeline).to_list(None)
```

> 단계 순서가 성능을 좌우 — `$match`/`$limit` 을 **최대한 앞으로** (인덱스 활용, 처리량 축소).
> 단계별 실행계획은 `aggregate(pipeline, explain=True)` 또는 `.explain()` 으로 확인.

---

## #40 Bulk Write / `Model::insert([...])` 대량·혼합

```python
from pymongo import InsertOne, UpdateOne, DeleteOne, ReplaceOne

# 혼합 연산을 한 번의 왕복으로 (insert/update/delete 섞기 가능)
async def sync_users(rows: list[dict]) -> None:
    ops = [
        UpdateOne(
            {"email": r["email"]},
            {"$set": r},
            upsert=True,                            # 없으면 insert (대량 upsert)
        )
        for r in rows
    ]
    await db.users.bulk_write(ops, ordered=False)   # ordered=False → 처리량 ↑, 일부 실패 무관

# Beanie 도 동일 — 컬렉션 핸들 직접 사용
await User.get_motor_collection().bulk_write(ops, ordered=False)
```

> 수천~수만 건 동기화는 `bulk_write` 가 정석 — 1회 왕복에 executemany.
> `ordered=False`: 순서 보장 X, 한 건 실패해도 나머지 진행, 가장 빠름.
> 대량 upsert 의 매칭 키엔 index 필수 (없으면 매 건 컬렉션 스캔 → 매우 느림).

---

## #41 복잡 Aggregation ($facet / 서브쿼리·조건부 조인)

```python
# 시나리오: 한 번의 쿼리로 "목록 + 전체 건수 + 상태별 통계" 동시에 ($facet)
pipeline = [
    {"$match": {"status": "paid"}},
    {"$facet": {
        "page":  [{"$sort": {"created_at": -1}}, {"$skip": 0}, {"$limit": 20}],
        "total": [{"$count": "n"}],
        "by_status": [{"$group": {"_id": "$status", "cnt": {"$sum": 1}}}],
    }},
]

# 조건부 $lookup (RDB 상관 서브쿼리 대응) — let + 서브파이프라인
lookup_correlated = [
    {"$lookup": {
        "from": "orders",
        "let": {"uid": "$_id"},
        "pipeline": [
            {"$match": {"$expr": {"$and": [
                {"$eq": ["$user_id", "$$uid"]},
                {"$gte": ["$amount", 100]},
            ]}}},
            {"$project": {"amount": 1}},
        ],
        "as": "big_orders",
    }},
]

async def dashboard() -> dict:
    rows = await db.orders.aggregate(pipeline).to_list(None)
    return rows[0]                                  # {"page":[...], "total":[{"n":..}], ...}
```

> `$facet` = 한 입력을 여러 서브파이프라인으로 동시 집계 (RDB 의 여러 서브쿼리/CTE 를 한 방에).
> `$lookup` 의 `let` + `pipeline` = 상관 서브쿼리(JOIN 조건에 부모 값 참조). `$$var` = 바깥에서 넘긴 변수.
> 정말 복잡하면 단계를 잘게 쪼개 각 단계 결과를 `$out`/`$merge` 로 임시 컬렉션에 떨구는 패턴도 있음.

---

## #42 조건 분기 ($cond / $switch) — CASE WHEN 대응

```python
# Laravel
DB::raw("CASE WHEN age < 20 THEN 'teen' WHEN age < 60 THEN 'adult' ELSE 'senior' END")

# Mongo — $switch (다중 분기) / $cond (2지 분기)
add_age_group = {"$addFields": {
    "age_group": {"$switch": {
        "branches": [
            {"case": {"$lt": ["$age", 20]}, "then": "teen"},
            {"case": {"$lt": ["$age", 60]}, "then": "adult"},
        ],
        "default": "senior",
    }},
}}

# $cond — 2지 (삼항 연산자)
flag = {"$addFields": {"is_vip": {"$cond": [{"$gte": ["$amount", 100000]}, True, False]}}}

# 조건부 집계 (SUM(CASE WHEN ...)) — 피벗/대시보드
breakdown = {"$group": {
    "_id": None,
    "paid_cnt":  {"$sum": {"$cond": [{"$eq": ["$status", "paid"]}, 1, 0]}},
    "paid_sum":  {"$sum": {"$cond": [{"$eq": ["$status", "paid"]}, "$amount", 0]}},
}}

async def by_age_group() -> list[dict]:
    return await User.aggregate([add_age_group]).to_list()
```

> aggregation 표현식은 **prefix 형**: `{"$lt": ["$age", 20]}` = `age < 20`. 필드 참조는 `"$필드명"`.
> `$switch` = 다중 CASE, `$cond` = 2지. 조건부 카운트/합계는 `$sum` 안에 `$cond` 중첩.

---

## #43 문자열 결합 ($concat)

```python
# Laravel
DB::raw("CONCAT(first_name, ' ', last_name) AS full_name")

# Mongo — $concat (모든 인자 string 이어야 함, null 있으면 결과 null)
add_full = {"$addFields": {
    "full_name": {"$concat": ["$first_name", " ", "$last_name"]},
}}

# null 안전하게: $ifNull 로 기본값
add_full_safe = {"$addFields": {
    "full_name": {"$concat": [
        {"$ifNull": ["$first_name", ""]}, " ",
        {"$ifNull": ["$last_name", ""]},
    ]},
}}

# CONCAT + 검색 (full_name LIKE '%kim%')
pipeline = [
    add_full,
    {"$match": {"full_name": {"$regex": "kim", "$options": "i"}}},
]
async def search_by_full_name() -> list[dict]:
    return await User.aggregate(pipeline).to_list()
```

> `$concat` 인자가 string 아니면 에러 → 숫자는 `{"$toString": "$n"}`, null 가능성은 `$ifNull` 로 감싸기.
> 구분자 join 은 `$concat` 직접 / 배열을 합치려면 `$reduce` 또는 `$concatArrays`.

---

## #44 인덱스 · 성능 노트 (eager loading 대응)

RDB 의 N+1 / eager load 고민이 Mongo 에선 **스키마 설계(임베드 vs 참조) + 인덱스**로 치환됨.

```python
# 인덱스 정의 — Beanie 모델 Settings
import pymongo
class User(Document):
    email: str
    status: str
    created_at: datetime
    class Settings:
        name = "users"
        indexes = [
            "email",                                          # 단일
            [("status", pymongo.ASCENDING),
             ("created_at", pymongo.DESCENDING)],              # 복합
            pymongo.IndexModel("email", unique=True),          # unique
        ]

# Motor — 직접 생성 (앱 부팅 시 1회)
await db.users.create_index("email", unique=True)
await db.users.create_index([("status", 1), ("created_at", -1)])
```

| RDB 고민 | Mongo 대응 |
|---|---|
| N+1 / eager load | **임베드**(조인 자체 제거) 또는 `fetch_links` / `$lookup` 1회 |
| `joinedload` vs `selectinload` | 임베드(한 문서) vs 참조+`$lookup`(별도 단계) |
| 인덱스로 WHERE 가속 | 동일 — `$match`/정렬 키에 인덱스, `explain()` 으로 검증 |
| SELECT 컬럼 최소화 | `$project` / projection 으로 필드 제한 |
| 깊은 페이지네이션 느림 | `skip` 대신 정렬 키 기준 range pagination (#15) |

> 복합 인덱스는 **ESR 규칙**(Equality → Sort → Range) 순으로 필드 배치하면 정렬·범위까지 한 인덱스로 커버.
> 쿼리가 인덱스를 타는지 항상 `.explain("executionStats")` 의 `IXSCAN`(좋음) vs `COLLSCAN`(나쁨) 확인.

---

## #45 DateTime 처리 (`now()` / `Carbon::now()` 대응)

```python
from datetime import datetime, timezone, timedelta
from pydantic import Field

# 모델 — 생성/수정 시각 (Laravel $table->timestamps())
class Post(Document):
    title: str
    published_at: datetime | None = None
    created_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    updated_at: datetime = Field(default_factory=lambda: datetime.now(timezone.utc))
    class Settings:
        name = "posts"

# (1) 앱 서버 시간 (now() / Carbon::now()) — UTC aware 권장
post = Post(title="hi", published_at=datetime.now(timezone.utc))
await post.insert()

# (2) DB 서버 시간 ($$NOW) — aggregation/update pipeline 에서만 사용 가능
await db.posts.update_many(
    {"status": "draft"},
    [{"$set": {"published_at": "$$NOW", "updated_at": "$$NOW"}}],   # 파이프라인 형태 update
)

# Carbon::now()->addDays(7)
post.published_at = datetime.now(timezone.utc) + timedelta(days=7)

# updated_at 자동 갱신 — Beanie 이벤트 훅
from beanie import before_event, Replace, Update, SaveChanges
class Post(Document):
    ...
    @before_event(Replace, Update, SaveChanges)
    def touch(self):
        self.updated_at = datetime.now(timezone.utc)

# API 입력 문자열 → datetime (Pydantic 자동)
class PostIn(BaseModel):
    title: str
    published_at: datetime | None = None            # "2026-05-18T09:30:00" 자동 파싱
```

| Laravel/PHP | Mongo/Python | 시간 출처 |
|---|---|---|
| `now()`, `Carbon::now()` | `datetime.now(timezone.utc)` | 앱 서버 (UTC aware 권장) |
| `DB::raw('NOW()')` | `"$$NOW"` (update/aggregation pipeline 내) | DB 서버 |
| `$table->timestamps()` | `default_factory` + `@before_event` 훅 | 앱 |
| `Carbon::now()->addDays(7)` | `datetime.now(timezone.utc) + timedelta(days=7)` | 앱 |
| `Carbon::parse($req->date)` | Pydantic `datetime` / `datetime.fromisoformat` | 입력값 |

> Mongo 의 `Date` 타입은 **내부적으로 UTC 밀리초**. naive datetime 넣으면 TZ 혼동 → 항상 `timezone.utc` aware 로.
> `$$NOW` 는 일반 `update_one({...},{...})` 의 `$set` 엔 못 씀 → **pipeline 형태 update**(`[{"$set": ...}]`)에서만.

---

# 부록

---

## 부록 — Beanie vs Motor(raw) 정리

| 항목 | Beanie (ODM) | Motor (raw) |
|---|---|---|
| 반환 타입 | `Document` 인스턴스 (Pydantic) | `dict` |
| 쿼리 표현 | `User.age > 18` (타입 체크됨) | `{"age": {"$gt": 18}}` |
| 단건 | `User.get(id)` / `find_one(...)` | `find_one({...})` |
| 다건 | `find(...).to_list()` | `find({...}).to_list(None)` |
| 결과 모델 | `.project(Model)` / `aggregate(.., projection_model=)` | 직접 dict 가공 |
| 검증/기본값 | Pydantic 으로 자동 | 없음 (직접) |
| 적합 | 모델 명확한 CRUD | 동적 스키마, 복잡 파이프라인, 최대 성능 |

> 보통 **Beanie 기본**, 복잡한 aggregation/`bulk_write`/동적 쿼리만 Motor 핸들(`User.get_motor_collection()`) 로 내려감.

---

## 부록 — PyMongo (sync) 변환

```python
# async (Motor)                           # sync (PyMongo)
doc = await db.users.find_one({...})      doc = db.users.find_one({...})
docs = await cur.to_list(None)            docs = list(cur)
await db.users.insert_one({...})          db.users.insert_one({...})
async for d in db.users.find({...}):      for d in db.users.find({...}):
```

> Motor 코드에서 `await` 제거 + `to_list(None)` → `list(cursor)` 면 거의 그대로 PyMongo. API 표면 동일.

---

## 부록 — Mongo 특화 함정

- `{"field": None}` 은 **null + 필드없음** 둘 다 매칭 → 구분하려면 `$exists` (#12)
- `update_one` 이 기본 — 일괄은 `update_many` 명시 (#31)
- `$set` 없이 `update_one({...}, {"name":"x"})` → **문서 전체 치환**(나머지 필드 날아감). 항상 `{"$set": {...}}`
- ObjectId vs str — URL/JSON 에서 온 id 는 `ObjectId(...)` 변환해야 매칭
- 트랜잭션은 replica set 필수 + 모든 연산에 `session=` 전달 (#38)
- `skip` 깊은 페이지 느림 → range pagination (#15)
- aggregation 단일 결과 문서 16MB 제한 — 대량은 `$out`/`$merge` 또는 커서 옵션
- `$lookup` 남발 금지 — 자주 같이 읽으면 임베드가 정답 (#25)

---

## 정리

- 단건 PK 는 `User.get(id)` / `find_one({"_id": ObjectId(id)})`
- 다건은 `find(...)` → 커서 → `.to_list()` 또는 `async for` 스트리밍
- 조건절: Beanie 연산자(`In`,`Or`,`GTE`...) ↔ Motor dict(`$in`,`$or`,`$gte`...)
- GROUP BY / HAVING / 윈도우 / JOIN 은 전부 **aggregation pipeline** (`$group`/`$match`/`$setWindowFields`/`$lookup`)
- 동시 갱신은 락 대신 원자적 `$inc`/조건부 `find_one_and_update`
- 관계는 **임베드 우선**, 참조면 `fetch_links`/`$lookup`
- upsert 는 `find_one_and_update(upsert=True)` + `$setOnInsert`
- 성능은 스키마 설계(임베드 vs 참조) + 인덱스 + `explain()` 으로
