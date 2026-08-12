# Elasticsearch (auth_apikey) 사용 정리

> `ES.txt`(REST DSL 쿼리)와 `ES 리스폰스.txt`(PHP 클라이언트 코드·응답)를 작업별로 합쳐 정리한 문서.
> 예시 인덱스: `api_keys` / `auth_apikey_dev`
>
> cf) Elasticsearch에서 작성하는 JSON 형태의 쿼리는 보통 Query DSL 이라고 부릅니다 (Domain Specific Language)
>
> 📌 **ES 쿼리가 낯설면 아래 대응표와 각 예제의 `MySQL 대응` 을 먼저 보세요.** (매핑·타입처럼 SQL 로 옮겨봐야 의미 없는 부분은 표로만 정리)

---

## 한눈에 보는 MySQL ↔ Elasticsearch 대응표

| MySQL | Elasticsearch |
|-------|---------------|
| 테이블 (table) | 인덱스 (index) |
| 행 (row) | 문서 (document) |
| 컬럼 (column) | 필드 (field) |
| 스키마 / `CREATE TABLE` | 매핑 (mapping) / `PUT index` |
| `PRIMARY KEY` | 문서 `_id` |
| `SELECT * FROM t` | `GET t/_search` |
| `WHERE` | `query` |
| `AND` / `OR` / `NOT` | `bool` 의 `must`·`filter` / `should` / `must_not` |
| `col = '값'` | `term` |
| `col IN (...)` | `terms` |
| `BETWEEN` / `>=` | `range` |
| `MATCH() AGAINST()` | `match` |
| `IS NULL` | `must_not` + `exists` |
| `ORDER BY` | `sort` |
| `LIMIT` / `OFFSET` | `size` / `from` |
| `GROUP BY` | `aggs` 의 Bucket (`terms` 등) |
| `COUNT`/`SUM`/`AVG` | `aggs` 의 Metric |
| `HAVING` | `bucket_selector` |
| `INSERT` | `index` / `create` |
| `UPDATE` | `update` (+ `doc`) |
| `DELETE` | `delete` / `_delete_by_query` |
| `JOIN` (1:N 자식 테이블) | 배열 필드 or `nested` 타입 |
| 뷰(VIEW) / 테이블 rename | alias |
| `COMMIT` 하면 즉시 보임 | **NRT — 기본 1초 뒤 검색 반영** (`refresh` 옵션) |

> ⚠️ **대응이 없는 3가지 (여기서 사고 남)**
> 1. **트랜잭션·롤백이 없음** — 여러 문서를 한 번에 원자적으로 바꿀 수 없습니다.
> 2. **조인이 없음** — 1:N 은 문서 안 배열/`nested` 로 미리 합쳐 저장해야 합니다.
> 3. **집계 숫자가 정확하지 않을 수 있음** — `cardinality`, `terms` 의 `doc_count` 는 근사치.

---

## 0. 인덱스 매핑 (Mapping)

| 필드 | 타입 | 비고 |
|------|------|------|
| `no` | integer |  |
| `apiKey` | keyword | 완전 일치 검색용 |
| `ips` | keyword | 배열 가능, 완전 일치 검색용 |

```json
PUT api_keys
{
  "mappings": {
    "properties": {
      "no": { "type": "integer" },
      "apiKey": { "type": "keyword" },
      "ips":    { "type": "keyword" }
    }
  }
}
```

> 📌 매핑은 MySQL 의 `CREATE TABLE` 자리입니다. (`integer` → `INT`, `keyword` → `VARCHAR`, `text` → `TEXT` + `FULLTEXT INDEX`)
> 이 문서의 SQL 예제는 `api_keys` 테이블(컬럼 `api_key`, `ip`, `no`) 기준으로 적었습니다.

> ⚠️ **핵심**: `text` 타입은 분석(analyze)되어 **완전 일치 체크가 안 될 수 있음**.
> 정확히 일치하는 값(apiKey, ip 등)은 반드시 `keyword`로 매핑해야 함.


### 주요 필드 타입과 특징

#### ① 문자열 타입 — `text` vs `keyword` (가장 중요)

| 타입 | 분석(analyze) | 완전 일치 | 부분 검색 | 정렬/집계 | 용도 |
|------|---------------|-----------|-----------|-----------|------|
| `text` | **O** (쪼갬) | ✕ | **O** | ✕ | 본문, 제목 등 **검색어로 찾는 글** |
| `keyword` | **X** (통째로) | **O** | ✕ | **O** | ID, 코드, 이메일, IP, 상태값 등 **값 그 자체** |

```
"Hello World" 저장 시

text    → ["hello", "world"] 로 쪼개서 저장   → "hello" 로 검색됨, "Hello World" 완전일치 ✕
keyword → "Hello World" 통째로 저장           → 정확히 "Hello World" 여야 매칭 ✓ (대소문자도 구분)
```

```sql (실예제)
-- MySQL 로 치면
SELECT * FROM posts WHERE title = 'Hello World';                 -- keyword + term
SELECT * FROM posts WHERE MATCH(title) AGAINST('Hello World');   -- text + match
```

> 💡 **판단 기준**: "이 필드로 **문장 검색**을 하나?" → `text` / "이 값이 **맞는지 대조**만 하나?" → `keyword`
> `apiKey`, `ips` 는 대조용이므로 `keyword`.

**둘 다 필요하면 멀티 필드**로 (검색은 `title`, 정렬·집계는 `title.keyword`)
```json
"title": {
  "type": "text",
  "fields": { "keyword": { "type": "keyword" } }
}
```

> 같은 컬럼에 일반 인덱스와 `FULLTEXT INDEX` 를 **둘 다 거는 것**과 같은 발상입니다.

**⚠️ `keyword`의 대소문자 함정 — `normalizer`**

`keyword`는 값을 통째로 저장하므로 `Abc@x.com` 과 `abc@x.com` 은 **서로 다른 값**입니다.
이메일·apiKey처럼 대소문자를 무시하고 대조해야 하는 필드는 `normalizer`로 소문자화해서 색인합니다.

```json
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lc": { "type": "custom", "filter": ["lowercase"] }
      }
    }
  },
  "mappings": {
    "properties": {
      "email": { "type": "keyword", "normalizer": "lc" }
    }
  }
}
```
> `normalizer`는 색인할 때와 `term` 검색어에 **똑같이** 적용되므로, 대소문자 아무렇게나 넣어도 매칭됩니다.
> **MySQL 의 콜레이션과 같은 개념** — `_ci`(대소문자 무시) vs `_bin`(구분) 을 고르는 것과 같습니다.
> (apiKey처럼 **대소문자를 구분해야 하는** 시크릿 값에는 쓰지 마세요.)

#### ② 숫자 타입

| 타입 | MySQL 대응 | 범위 / 특징 |
|------|------------|-------------|
| `integer` | `INT` | -21억 ~ 21억 (약 ±2^31). 일반적인 순번·ID |
| `long` | `BIGINT` | 매우 큰 정수 (±2^63). 타임스탬프(ms), 대용량 ID |
| `short` / `byte` | `SMALLINT` / `TINYINT` | -32768~32767 / -128~127. 작은 코드값 (저장 절약) |
| `float` / `double` | `FLOAT` / `DOUBLE` | 실수. 정밀도 double > float |
| `scaled_float` | `DECIMAL(15,2)` | 실수를 정수로 저장해 성능↑ (`scaling_factor` 필요). **금액**에 적합 |

> ⚠️ **숫자처럼 보여도 계산·범위검색을 안 하면 `keyword`가 나음** (전화번호, 우편번호, 사번 등).
> `keyword`가 정확 일치 검색이 더 빠름.

#### ③ 그 외 자주 쓰는 타입

| 타입 | MySQL 대응 | 예시 데이터 | 특징 / 용도 |
|------|------------|-------------|-------------|
| `boolean` | `TINYINT(1)` / `BOOLEAN` | `true` / `"true"` | `true` / `false` (문자열 `"true"`도 허용) |
| `date` | `DATETIME` / `TIMESTAMP` | `"2026-07-22 14:30:00"`, `1753160400000` | 날짜·시간. `format` 지정 가능, 범위 검색(`range`)·정렬에 사용 |
| `ip` | `VARBINARY(16)` + `INET6_ATON()` | `"192.168.0.1"`, `"::1"` | IPv4/IPv6 전용. **CIDR 대역 검색 가능** (`192.168.0.0/24`) |
| `object` | `JSON` 컬럼 (또는 컬럼 분리) | `{"user": {"name": "홍길동", "age": 30}}` | 중첩 JSON. 내부적으로 `user.name` 처럼 평탄화됨 |
| `nested` | **자식 테이블 (1:N) + JOIN** | `[{"name":"A","qty":1}, {"name":"B","qty":2}]` | 객체 **배열**의 각 원소를 독립 문서로 취급 (원소별 조건 조합이 정확) |
| `geo_point` | `POINT` + `SPATIAL INDEX` | `{"lat": 37.5665, "lon": 126.9780}` | 위/경도. 거리·반경 검색 |
| `binary` | `BLOB` | `"U29tZSBiaW5hcnkgZGF0YQ=="` | Base64 저장. 기본적으로 검색 불가 |

**타입별 저장 데이터 예시 (한 문서에 모아본 형태)**
```json
{
  "isActive":  true,
  "createdAt": "2026-07-22 14:30:00",
  "clientIp":  "192.168.0.1",
  "user":      { "name": "홍길동", "age": 30 },
  "items":     [ { "name": "A", "qty": 1 }, { "name": "B", "qty": 2 } ],
  "location":  { "lat": 37.5665, "lon": 126.9780 },
  "thumbnail": "U29tZSBiaW5hcnkgZGF0YQ=="
}
```

> ⭐ **`object` vs `nested` 차이** — 배열 안 객체를 다룰 때 결과가 달라짐.
> `object`는 필드별로 평탄화되어 **원소 간 짝이 깨짐**.
> ```
> 저장: [ {"name":"A","qty":1}, {"name":"B","qty":2} ]
>
> object → name: ["A","B"] , qty: [1,2] 로 따로 저장
>          → "name=A 이면서 qty=2" 검색이 ❌ 매칭됨 (잘못된 결과)
> nested → 각 객체를 별도 문서로 취급
>          → "name=A 이면서 qty=2" 검색이 ✅ 매칭 안 됨 (정확)
> ```
> 배열 객체의 **필드 조합 조건**이 필요하면 `nested`, 아니면 `object`(기본)로 충분.

> `nested` 는 MySQL 로 치면 **자식 테이블**입니다. 한 행 안에서 두 조건이 동시에 참이어야 하므로 결과가 정확합니다.

**date 예시**
```json
"createdAt": {
  "type": "date",
  "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
}
```

**ip 타입을 쓰면 대역 검색 가능** (지금 `ips`는 `keyword`라 정확 일치만 됨)
```json
"ips": { "type": "ip" }
```
```json
{ "term": { "ips": "192.168.0.0/24" } }   // 해당 대역 전체 매칭
```

#### ④ 배열은 별도 타입이 아님

ES는 **모든 필드가 기본적으로 배열을 허용**합니다. 타입은 그대로 두고 값만 배열로 넣으면 됩니다.

```json
"ips": { "type": "keyword" }          // 매핑은 단일 타입 그대로
"ips": ["127.0.0.1", "192.168.0.1"]   // 값은 배열로 저장 가능 ✓
```

> 단, **배열 안 원소들의 타입은 모두 같아야** 함.

> ⭐ **MySQL 에는 배열 컬럼이 없습니다.** 같은 데이터를 넣으려면 자식 테이블을 만들어 행을 늘리고
> 조회할 때 조인해야 하죠. ES 는 문서 하나에 배열로 담고 `term` 한 줄로 조회합니다.
> **1:N 데이터를 조인 없이 다루는 게 ES 를 쓰는 이유 중 하나입니다.**

#### ⑤ 매핑 관련 주의사항

- **한 번 만든 필드의 타입은 변경 불가.** 바꾸려면 새 인덱스를 만들고 `_reindex` 해야 함.
- 매핑을 안 주면 **동적 매핑(dynamic mapping)** 으로 자동 추론됨
  → 문자열이 `text` + `keyword` 멀티필드로 잡혀 의도와 달라질 수 있으니, **중요 인덱스는 매핑을 직접 정의**할 것.
- 필드 **추가**는 가능 (기존 필드 수정만 불가).
- 검색에 쓰지 않는 필드는 `"index": false` 로 색인을 꺼서 용량·성능 절약 가능.

**MySQL 과 다른 점 (여기서 제일 많이 당황함)**

| 작업 | MySQL | Elasticsearch |
|------|-------|---------------|
| 컬럼/필드 타입 변경 | `ALTER TABLE ... MODIFY col BIGINT` ✅ | **불가** → 새 인덱스 + `_reindex` |
| 컬럼/필드 추가 | `ALTER TABLE ... ADD COLUMN` ✅ | 가능 ✅ |
| 스키마 미정의 | 에러 (컬럼 없으면 못 넣음) | **동적 매핑으로 자동 생성** (의도와 다르게 잡힘) |
| 인덱스 끄기 | `INDEX` 안 걸면 됨 | `"index": false` |

**타입을 바꿔야 할 때 — alias + `_reindex` (무중단)**

애플리케이션이 처음부터 **실제 인덱스가 아닌 alias를 바라보게** 해두면, 교체 시 코드 수정 없이 전환됩니다.

```json
// 0) 최초 생성 시부터 alias를 걸어둔다
POST /_aliases
{ "actions": [ { "add": { "index": "auth_apikey_v1", "alias": "auth_apikey_dev" } } ] }

// 1) 새 매핑으로 v2 생성
PUT /auth_apikey_v2
{ "mappings": { "properties": { "ips": { "type": "ip" } } } }

// 2) 데이터 복사 (건수 많으면 wait_for_completion=false 로 백그라운드)
POST /_reindex
{
  "source": { "index": "auth_apikey_v1" },
  "dest":   { "index": "auth_apikey_v2" }
}

// 3) alias를 v2로 원자적 교체 (remove + add 가 한 번에 적용됨 — 무중단)
POST /_aliases
{
  "actions": [
    { "remove": { "index": "auth_apikey_v1", "alias": "auth_apikey_dev" } },
    { "add":    { "index": "auth_apikey_v2", "alias": "auth_apikey_dev" } }
  ]
}
```

> - `_reindex` 는 **매핑을 자동으로 안 만들어 줌.** 목적지 인덱스를 **먼저 새 매핑으로 생성**해야 함
>   (안 그러면 동적 매핑으로 잡혀서 하려던 타입 변경이 무의미해짐).
> - `_reindex` 중에도 원본에는 계속 쓰기가 들어올 수 있으므로, 복사 후 **누락분을 다시 한 번 반영**하거나
>   `createdAt` 기준으로 증분 재복사할 것.
> - 검증 후 구 인덱스 삭제. alias는 롤백 지점이기도 하므로 **바로 지우지 말고 며칠 두는 편이 안전**.

> 💡 MySQL 의 **무중단 테이블 재구성과 같은 절차**입니다.
> `CREATE TABLE 신규` → `INSERT INTO 신규 SELECT * FROM 기존`(= `_reindex`) → `RENAME TABLE`(= alias 교체).

---

## 1. 검색 (SEARCH) — 특정 apiKey & ip 조건

**REST DSL**
```json
GET api_keys/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "apiKey": "abc123" } },
        { "term": { "ips": "192.168.0.2" } }
      ]
    }
  }
}
```

**PHP 클라이언트**
```php
public function searchDocument()
{
    $apiKey = 'abcdedfgesdklfjl';
    $ip = '127.0.0.1';

    $response = $this->client->search([
        'index' => 'auth_apikey_dev',
        'body' => [
            'query' => [
                'bool' => [
                    'filter' => [
                        ['term' => ['apiKey' => $apiKey]],
                        ['term' => ['ips' => $ip]]
                    ],
                ],
            ],
        ],
    ]);

    $result = $response->asArray();
}
```

**응답(요약)**
```php
"hits" => [
  "total" => [ "value" => 1, "relation" => "eq" ],
  "hits"  => [
    0 => [
      "_id" => "fuGHXp8BE3j6GTNTMsnA",
      "_source" => [
        "id" => 1,
        "apiKey" => "abcdedfgesdklfjl",
        "ips" => ["127.0.0.1", "192.168.0.1"]
      ]
    ]
  ]
]
```

**쿼리 문법 ↔ MySQL 대응**

| ES | MySQL 대응 | 설명 |
|----|----------|------|
| `api_keys/_search` | `SELECT * FROM api_keys` | 인덱스에서 검색 |
| `query` | `WHERE` (절 전체) | 검색 조건이 들어가는 **최상위 컨테이너. 조건 개수와 무관하게 항상 필요** |
| `bool` | `WHERE A AND B` | 여러 조건을 AND/OR/NOT로 **조합**할 때 `query` 안에 넣는 절 |
| `must` | `A AND B AND C` | 모든 조건 만족 (검색 점수 계산 O), 배열로 감쌈 (`filter`·`should`·`must_not` 도 동일) |
| `term` | `WHERE no = 1` | 하나의 정확한 값 비교 |
| `terms` | `WHERE no IN (1,2,3)` | 찾을 값이 여러 개일 때 (IN) |
| `filter` | `WHERE` (AND) | 정확 일치만 필요할 때 (점수 계산 X, 캐싱 유리) |
| `range` | `WHERE price BETWEEN 10000 AND 50000` | 숫자·날짜 등 범위 조건 (`gte`, `lte`, `gt`, `lt`) |
| `match` | `WHERE MATCH(title) AGAINST('...')` | 문장 검색 (분석 후 비교) |
| `from` / `size` | `LIMIT size OFFSET from` | 페이지네이션 |
| `sort` | `ORDER BY` | 정렬 |
| `_source` | `SELECT col1, col2` (컬럼 지정) | 가져올 필드 선택 |
| `aggs` | `GROUP BY` + 집계함수 | 묶고 계산 (→ 15장) |

> ⚠️ **`query`와 `bool`은 택일 관계가 아님** (자주 하는 오해).
> `query`는 SQL의 `WHERE` 절 그 자체라 언제나 있어야 하고, 조건이 여러 개일 때 그 안에 `bool`을 넣는 구조.
> ```
> query               → WHERE 절 그 자체 (항상 필요)
>  ├ term / match     → 조건 1개
>  └ bool             → 조건 여러 개를 AND/OR/NOT로 조합 (하위 4개 모두 배열로 감쌈)
>     ├ must / filter → AND
>     ├ should        → OR
>     └ must_not      → NOT
> ```

> bool 쿼리 안에는 의미 있는 절이 최소 하나 있어야 함 (must, should, filter, must_not)
* must: AND
* filter: WHERE (AND)
* should: OR (다른 필드끼리의 OR. 같은 필드의 OR 은 `terms`)
* must_not: NOT

> 💡 **네 절(must, filter, should, must_not) 모두 배열 `[ ]` 로 감쌉니다.** (`must` 만 배열인 게 아님)
> 조건이 1개일 땐 배열을 생략하고 객체로 바로 써도 동작하지만, **조건이 늘어날 걸 감안해 항상 배열로 쓰는 편이 안전**합니다.
> ```json
> "filter": [ { "term": { "apiKey": "abc123" } } ]   // 권장 (조건 추가가 자유로움)
> "filter":   { "term": { "apiKey": "abc123" } }     // 동작은 함 (조건 1개 전용)
> ```

### ⭐ "정확한 값 2개를 **둘 다** 만족하는 문서" 를 찾으려면 → `filter`

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "apiKey": "abc123" } },
        { "term": { "ips": "192.168.0.2" } }
      ]
    }
  }
}
```
```sql (실예제)
-- MySQL 로는 그냥 이것
WHERE api_key = 'abc123' AND ip = '192.168.0.2'
```

`filter` 배열에 나열하면 **AND** 입니다. 조건이 2개든 5개든 나열하면 전부 AND 로 묶입니다.

| | 찾는 결과 | 점수 계산 | 캐싱 | 속도 |
|---|---|---|---|---|
| `must` | 같음 | **함** (안 쓸 값을) | ✕ | 느림 |
| `filter` | 같음 | 안 함 | **O** | 빠름 |

여기서 `must` 로 바꿔도 **찾아지는 문서는 완전히 똑같습니다.** 차이는 **점수 계산 여부**뿐입니다.
정확 일치 조건만 있고 순서가 필요 없으면 **전부 `filter`** 가 정답입니다.

> 왜 점수 계산이 붙는지, 점수가 언제 의미 있는지는 **→ 9장** 에서 자세히 다룹니다.

### ⭐ `A AND B AND (C OR D)` 를 만들려면 — `filter` + `should`

**"판매중이고, 10만원 이상이면서, 브랜드가 애플 or 삼성"** 을 찾는 쿼리입니다.

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term":  { "status": "OPEN" } },
        { "range": { "price": { "gte": 100000 } } }
      ],
      "should": [
        { "term": { "brand": "apple" } },
        { "term": { "brand": "samsung" } }
      ],
      "minimum_should_match": 1
    }
  }
}
```
```sql (실예제)
SELECT * FROM products
WHERE status = 'OPEN'                              -- filter[0]
  AND price >= 100000                              -- filter[1]
  AND (brand = 'apple' OR brand = 'samsung');      -- should + minimum_should_match: 1
```

#### ⚠️ `minimum_should_match` 를 빠뜨리면 OR 이 아닙니다 (가장 잘 걸리는 함정)

`should` 는 **혼자 쓰일 때만** "최소 1개 만족" 이 기본입니다.
`filter` 나 `must` 와 **같이 쓰면 `should` 는 그냥 가점 조건으로 바뀝니다.**

```sql (실예제)
-- minimum_should_match 를 안 주면 이렇게 동작함
SELECT * FROM products
WHERE status = 'OPEN' AND price >= 100000          -- ← 이 조건만 걸림
ORDER BY (brand IN ('apple','samsung')) DESC;      -- ← should 는 정렬 가점으로만
--   → LG 제품도 결과에 나옴 (순서만 뒤로 밀릴 뿐)
```

| 쓰임 | 동작 | SQL |
|------|------|-----|
| `should` 단독 | 최소 1개 만족해야 함 | `WHERE (C OR D)` |
| `filter`/`must` + `should` | **should 는 가점만** | `WHERE A AND B ORDER BY (C OR D) DESC` |
| `filter`/`must` + `should` + `minimum_should_match: 1` (filter/must와 함께 쓰고 OR 조건 사용하려면 필수) | 진짜 AND (C OR D) | `WHERE A AND B AND (C OR D)` |

> 💡 **같은 필드의 OR 이면 `should` 대신 `terms` 가 훨씬 간단합니다.**
> ```json
> "filter": [
>   { "term":  { "status": "OPEN" } },
>   { "terms": { "brand": ["apple", "samsung"] } }   // brand IN ('apple','samsung')
> ]
> ```
> `should` 는 **필드가 서로 다른 OR**(`brand = 'apple' OR price < 10000`)에 씁니다.

**`must_not` 까지 넣으면**

```json
{
  "query": {
    "bool": {
      "filter":   [ { "term": { "status": "OPEN" } } ],
      "should":   [ { "term": { "brand": "apple" } },
                    { "range": { "price": { "lt": 10000 } } } ],
      "minimum_should_match": 1,
      "must_not": [ { "term": { "isDeleted": true } } ]
    }
  }
}
```
```sql (실예제)
SELECT * FROM products
WHERE status = 'OPEN'                          -- filter
  AND (brand = 'apple' OR price < 10000)       -- should + minimum_should_match: 1
  AND isDeleted <> TRUE;                       -- must_not
```

> 괄호로 묶인 `OR` 덩어리가 필요할 때마다 **`should` + `minimum_should_match` 한 세트**라고 외우면 됩니다.
> `OR` 덩어리가 2개 이상이면 `bool` 안에 `bool` 을 중첩합니다 (SQL 의 괄호 중첩과 같음).

**예1) 조건 1개 — `bool` 없이 `query` 바로 아래**
```json
{
  "from": 0,
  "size": 1,
  "track_total_hits": true,
  "query": {
    "term": {
      "no": 54261872
    }
  },
  "sort": [
    { "no": { "order": "desc" } }
  ]
}
```
```sql (실예제)
-- MySQL 대응
SELECT * FROM api_keys
WHERE no = 54261872        -- query > term
ORDER BY no DESC           -- sort
LIMIT 1 OFFSET 0;          -- size / from

SELECT COUNT(*) FROM api_keys WHERE no = 54261872;   -- track_total_hits: true 에 해당
```

**예2) `object` 하위 필드 접근 — 점(`.`) 표기법**
```json
{
  "from": 0,
  "size": 1,
  "track_total_hits": true,
  "query": {
    "term": {
      "channel.dome": true
    }
  },
  "sort": [
    { "no": { "order": "desc" } }
  ]
}
```
```sql (실예제)
-- MySQL 대응 (JSON 컬럼의 하위 키를 찍는 것과 같음)
SELECT * FROM api_keys WHERE channel->>'$.dome' = 'true' ORDER BY no DESC LIMIT 1;
```

> ⚠️ 위는 `object` 타입의 하위 필드를 점 표기법으로 찍는 것이지 **`nested` 접근이 아님**.
> 필드가 `nested` 타입이면 `term`으로 바로 못 들어가고 `nested` 쿼리로 감싸야 함.
> ```json
> {
>   "nested": {
>     "path": "items",
>     "query": {
>       "bool": {
>         "filter": [
>           { "term": { "items.name": "A" } },
>           { "term": { "items.qty": 2 } }
>         ]
>       }
>     }
>   }
> }
> ```
> (MySQL 로 치면 자식 테이블을 조인해 **한 행 안에서** 두 조건을 보는 것과 같습니다.)

**예3) 여러 조건 조합 — `query` 안에 `bool`**
```json
{
  "from": 0,
  "size": 1,
  "track_total_hits": true,
  "query": {
    "bool": {
      "must": [
        { "term": { "no": 54261872 } }
      ]
    }
  },
  "sort": [
    { "no": { "order": "desc" } }
  ]
}
```
> 조건이 1개뿐이라 예1과 결과는 같습니다. `bool` 은 **조건을 늘릴 자리**를 만든 것뿐입니다.
> (`must` 에 항목을 추가하면 `AND` 가 늘어납니다.)

| 파라미터 | MySQL 대응 | 용도 |
|----------|------------|------|
| `from` | `OFFSET` | 페이지네이션 시작 위치 |
| `size` | `LIMIT` | 반환할 문서 개수 |
| `track_total_hits` | `SELECT COUNT(*)` 를 따로 날리는 것 | `true`면 정확한 total 개수 반환 (기본은 10,000에서 멈춤) |

> ⚠️ **JSON에 `//` 주석을 넣지 말 것.** Kibana Dev Tools는 관대하게 넘어가지만,
> curl·PHP 클라이언트로 그대로 보내면 파싱 에러(400)가 남. 주석은 설명용으로만.

> ⚠️ **`_id`로 정렬하지 말 것.** `sort: [{"_id": "desc"}]` 는 `_id` fielddata를 힙에 통째로 올려
> 메모리를 크게 먹고 ES에서 비권장(deprecated)하는 경로임. 게다가 자동 생성 `_id`는 랜덤 문자열이라
> **정렬해도 시간순이 아님**. 정렬은 `no` / `createdAt` 같은 전용 필드로,
> 순서가 상관없는 전량 스캔이면 `"sort": ["_doc"]` 이 가장 빠름.

> - apiKey·ips처럼 **정확히 일치하는 값만 조회**할 땐 `filter` 사용.
> - 조건이 1개면 `bool > filter` 생략 가능.
> - **문서 내용까지 필요** → `search()` (결과 + `hits.total` 을 한 번에 얻음)
> - **개수만 필요** → `count()` API
> - **존재 여부만 필요** → `search()` + `"size": 0` + `terminate_after: 1` (가장 가벼움)

### ⭐ term vs terms 차이 (자주 헷갈림)

핵심: **"내가 찾는 값(검색어)이 1개냐, 여러 개냐"** 의 차이. (필드가 배열이냐 아니냐가 아님!)

| 구분 | 찾는 값 | MySQL | 의미 |
|------|---------|-----|------|
| `term` | **1개** | `WHERE api_key = 'abc123'` | 이 값과 정확히 일치하는 문서 |
| `terms` | **여러 개(배열)** | `WHERE id IN (1, 2, 3)` | 이 값들 **중 하나라도** 일치하는 문서 |

**term — 값 하나**
```json
{ "term": { "apiKey": "abc123" } }
```
```sql (실예제)
SELECT * FROM api_keys WHERE api_key = 'abc123';
```

**terms — 값 여러 개 (OR / IN)**
```json
{ "terms": { "id": [1, 2, 3] } }
```
```sql (실예제)
SELECT * FROM api_keys WHERE id IN (1, 2, 3);
```

> 💡 `term`은 값이 **문자열/숫자 하나**, `terms`는 값이 **배열 `[ ]`** 이라는 점만 기억하면 됨.

#### 헷갈리는 포인트 — 필드가 배열(`ips`)일 때
`ips`는 값이 여러 개인 배열 필드지만, **검색어가 1개면 그냥 `term`을 씀**.
ES는 배열 필드에서 "원소 중 하나라도 일치하면" 매칭시켜 주기 때문.

```json
// 저장된 문서: "ips": ["127.0.0.1", "192.168.0.1"]

{ "term":  { "ips": "127.0.0.1" } }              // ✅ 매칭됨 (원소 중 하나와 일치)
{ "terms": { "ips": ["10.0.0.1", "127.0.0.1"] } } // ✅ 둘 중 하나라도 ips에 있으면 매칭
```

- 이 IP **하나**가 등록돼 있는지 확인 → `term` (`= 값`)
- 이 IP **목록 중 하나라도** 등록돼 있는지 확인 → `terms` (`IN (...)`)

**PHP 예시**
```php
// term : IP 1개 확인          →  WHERE ip = '127.0.0.1'
['term'  => ['ips' => '127.0.0.1']]

// terms : IP 여러 개 중 하나라도 →  WHERE ip IN ('10.0.0.1','127.0.0.1')
['terms' => ['ips' => ['10.0.0.1', '127.0.0.1']]]
```

#### 실전 예시 — API Key 인증 (요청 토큰 + 요청 IP 검증)

**"이 apiKey가 존재하고, 등록된 ips 목록 안에 요청 IP가 들어있는 문서"** 를 찾는 인증용 쿼리.

**MySQL 로 먼저 보면**
```sql (실예제)
SELECT EXISTS (
  SELECT 1 FROM api_keys
  WHERE api_key = ?   -- $request->bearerToken()
    AND ip      = ?   -- $request->ip()
  LIMIT 1             -- terminate_after: 1 (1건 찾으면 중단)
);
```
| ES 옵션 | MySQL 대응 |
|---------|-----------|
| `'size' => 0` | `SELECT 1` (본문 컬럼을 안 가져옴) |
| `'terminate_after' => 1` | `LIMIT 1` |
| `filter` | `WHERE` (점수 계산 없는 순수 조건) |

```php
$response = $this->client->search([
    'index' => $this->authApiKeyIndex,
    'body'  => [
        'size'             => 0,      // 문서 본문 불필요 — 존재 여부만 보면 됨
        'terminate_after'  => 1,      // 1건 찾으면 즉시 중단
        'track_total_hits' => false,  // 정확한 총계 불필요 (성능)
        'query' => [
            'bool' => [
                'filter' => [
                    ['term' => ['apiKey' => $request->bearerToken()]], // 요청 헤더의 토큰 1개
                    ['term' => ['ips'    => $request->ip()]]           // 요청 IP 1개
                ]
            ]
        ]
    ]
]);
```

> - 두 조건 모두 **찾는 값이 1개**(`bearerToken()`, `ip()`)라서 → `term`. (`terms` 아님)
> - `ips`는 배열 필드지만, **요청 IP 1개가 그 배열 안에 있는지**만 보면 되므로 `term`으로 충분.
>   → ES가 "배열 원소 중 하나라도 일치" 를 자동 처리함.
> - 정확 일치 + 점수 불필요 → `filter` 사용 (빠르고 캐싱됨).
> - 인증 성공 여부는 결과 존재로 판단:
>   ```php
>   $hit = $response['hits']['total']['value'] > 0;  // true 면 인증 통과
>   ```
> - ⚠️ `track_total_hits => false` 를 주면 `hits.total` 이 안 내려옴.
>   위처럼 `total` 로 판단할 거면 이 옵션은 빼거나, `count($response['hits']['hits']) > 0` 로 판단할 것.

**문서 내용도 함께 써야 한다면 `_source` 필터링으로 필요한 필드만**
```php
'size'    => 1,
'_source' => ['id', 'apiKey'],   // 이 필드만 가져옴 (ips 배열 전체를 안 실어옴)
```
> `SELECT *` 대신 필요한 컬럼만 적는 것과 같습니다.
> 인증처럼 **초당 호출이 많은 지점**은 문서 전체를 실어오는 비용이 누적됨.
> 존재 여부만 → `size: 0`, 일부 값만 필요 → `_source` 로 잘라 쓰기.

### ⭐ match vs term (가장 흔한 사고)

`term`은 **색인된 값과 글자 그대로** 비교합니다. 그래서 `text` 필드에 `term`을 쓰면 대부분 안 잡힙니다.

```
저장: "Hello World"   →  text 필드의 색인 결과: ["hello", "world"]

{ "term":  { "title": "Hello World" } }          ✕ 색인에 그런 토큰이 없음
{ "term":  { "title": "hello" } }                ✓ 토큰과 우연히 일치
{ "match": { "title": "Hello World" } }          ✓ 검색어도 분석해서 비교
{ "term":  { "title.keyword": "Hello World" } }  ✓ 멀티필드의 keyword 쪽
```

| 쿼리 | MySQL 대응 | 검색어 분석 | 대상 필드 | 용도 |
|------|-----------|-------------|-----------|------|
| `term` / `terms` | `WHERE col = '값'` / `IN (...)` | **X** (입력 그대로) | `keyword`, 숫자, 날짜, `boolean` | 값 대조 (ID, 코드, 상태값) |
| `match` | `WHERE MATCH(col) AGAINST('값')` | **O** (분석 후 비교) | `text` | 문장·단어 검색 |

> 💡 **"검색 결과가 0건인데 데이터는 분명히 있다"** 면 십중팔구 `text` 필드에 `term`을 쓴 경우.
> `apiKey`·`ips` 는 `keyword` 라서 `term` 이 맞음.

---

## 2. 전체 검색 (SEARCH ALL)

**REST DSL**
```json
GET api_keys/_search
{
  "size": 100,
  "query": { "match_all": {} }
}
```

**MySQL 대응**
```sql (실예제)
SELECT * FROM api_keys LIMIT 100;    -- match_all = WHERE 절 없음
```
> ⚠️ ES 는 `size` 를 안 주면 **기본 10건만** 나옵니다 (MySQL 이 `LIMIT` 없으면 전부 나오는 것과 반대).

**PHP 클라이언트**
```php
public function searchAllDocuments()
{
    $response = $this->client->search([
        'index' => 'auth_apikey_dev',
        'body' => [
            'size' => 100
        ]
    ]);

    $result = $response->asArray();
}
```

---

## 3. 생성 (INDEX / 저장)

**REST DSL**
```json
POST api_keys/_doc
{
  "apiKey": "abc123",
  "ips": ["192.168.0.1", "192.168.0.2", "10.0.0.1"]
}
```

**MySQL 대응**
```sql (실예제)
INSERT INTO api_keys (api_key, ips) VALUES ('abc123', '...');
```
> `POST .../_doc` 은 `_id` 를 ES 가 자동 생성 → MySQL 의 `AUTO_INCREMENT` PK 와 같은 자리.
> `PUT .../_doc/abc123` 처럼 `_id` 를 직접 주면 **PK 를 내가 지정해 INSERT** 하는 것과 같습니다.

**PHP 클라이언트**
```php
public function createDocument()
{
    $response = $this->client->index([
        'index' => 'auth_apikey_dev',
        'body'  => [
            'id' => 1,
            'apiKey' => 'abcdedfgesdklfjl',
            'ips' => ['127.0.0.1', '192.168.0.1']
        ]
    ]);

    $result = $response->asArray();
}
```

**응답(요약)**
```php
[
  "_index" => "auth_apikey_dev",
  "_id" => "d-H7Xp8BE3j6GTNTI8xD",
  "_version" => 1,
  "result" => "created"
]
```

### cf) 유니크 키(apiKey)로 중복 방지 저장
```php
$response = $this->client->index([
    'index' => $this->authApiKeyIndex,
    'id' => $request->input('apiKey'),   // 문서 _id 를 apiKey 로 지정
    'op_type' => 'create',               // ← 이미 있으면 409 에러 → 중복 원천 차단
    'body'  => [
        'id' => $request->input('id'),
        'apiKey' => $request->input('apiKey'),
        'ips' => $ips
    ]
]);
```

**MySQL 대응 — `_id` 를 apiKey 로 지정 = `api_key` 를 PK 로 두고 INSERT**

| ES | MySQL |
|----|-------|
| `'id' => $apiKey` (문서 `_id` 지정) | `PRIMARY KEY (api_key)` |
| `'op_type' => 'create'` | 그냥 `INSERT` (중복이면 에러) |
| 기본 `index()` | `REPLACE INTO` (있으면 통째로 덮어씀) |
| 409 Conflict | `1062 Duplicate entry` |

**예외 처리**
```php
use Elastic\Elasticsearch\Exception\ClientResponseException;

catch (ClientResponseException $e) {
    // "이미 저장된 apiKey입니다"   ← MySQL 이라면 PDOException code 23000 / 1062 잡는 자리
}
```

> 💡 문서 `_id` 는 **최대 512바이트**. apiKey처럼 긴 값을 `_id` 로 쓸 땐 길이를 확인하고,
> 값이 URL 경로에 그대로 노출된다는 점(로그·프록시에 남음)도 감안할 것.

### cf) 대량 저장은 `bulk` — 한 번의 요청으로 처리

문서를 하나씩 `index()` 로 넣으면 건당 HTTP 왕복이 발생해 건수가 늘수록 급격히 느려집니다.
`bulk` 는 여러 작업을 한 요청에 모아 보냅니다.

**REST DSL** (각 줄이 개행으로 구분된 NDJSON — 마지막 줄에도 개행 필요)
```
POST /_bulk
{ "index":  { "_index": "auth_apikey_dev", "_id": "abc123" } }
{ "id": 1, "apiKey": "abc123", "ips": ["127.0.0.1"] }
{ "create": { "_index": "auth_apikey_dev", "_id": "def456" } }
{ "id": 2, "apiKey": "def456", "ips": ["10.0.0.1"] }
{ "delete": { "_index": "auth_apikey_dev", "_id": "ghi789" } }
```

> MySQL 의 **다중 INSERT**(`VALUES (...), (...)`) 와 같은 발상입니다.
> 다만 `bulk` 는 **한 요청에 index/create/update/delete 를 섞어** 보낼 수 있습니다.

**PHP 클라이언트**
```php
$params = ['body' => []];

foreach ($rows as $row) {
    $params['body'][] = ['index' => [
        '_index' => 'auth_apikey_dev',
        '_id'    => $row['apiKey'],
    ]];
    $params['body'][] = [                 // 바로 다음 줄이 본문
        'id'     => $row['id'],
        'apiKey' => $row['apiKey'],
        'ips'    => $row['ips'],
    ];
}

$response = $this->client->bulk($params)->asArray();
```

| 액션 | 동작 | MySQL 대응 |
|------|------|------------|
| `index` | 있으면 덮어쓰기, 없으면 생성 | `REPLACE INTO` |
| `create` | 이미 있으면 실패(409) — 중복 방지 | `INSERT INTO` |
| `update` | 부분 수정 (`doc` 필요) | `UPDATE ... SET col = ...` |
| `delete` | 삭제 (본문 줄 없음) | `DELETE FROM ... WHERE pk = ?` |

> ⚠️ **`bulk` 는 일부만 실패해도 HTTP 200을 반환**합니다. 예외로 안 잡히니 응답을 직접 확인해야 함:
> ```php
> if ($response['errors']) {
>     foreach ($response['items'] as $item) {
>         $op = array_key_first($item);
>         if (isset($item[$op]['error'])) {
>             // $item[$op]['error']['reason'] 로그
>         }
>     }
> }
> ```
> 한 번에 보내는 양은 **5~15MB 또는 1,000~5,000건** 정도로 끊는 게 무난합니다.

---

## 4. 카운트 (COUNT)

**REST DSL**
```
GET /auth_apikey_dev/_count
```

**조건부 카운트**
```json
GET /auth_apikey_dev/_count
{
  "query": { "term": { "apiKey": "abc123" } }
}
```

**MySQL 대응**
```sql (실예제)
SELECT COUNT(*) FROM auth_apikey;                          -- GET /_count
SELECT COUNT(*) FROM auth_apikey WHERE api_key = 'abc123'; -- 조건부 count
```

**응답**
```php
[
  "count" => 1,
  "_shards" => [ "total" => 1, "successful" => 1, "skipped" => 0, "failed" => 0 ]
]
```

> **용도별 선택** (11장 존재 확인과 같은 기준)
> - **문서 내용까지 필요** → `search()` — 결과와 `hits.total` 을 한 번에 얻으므로 `count` 를 따로 부를 필요 없음
> - **정확한 전체 건수만 필요** → `count()`
> - **존재 여부만 필요** → `search()` + `"size": 0` + `terminate_after: 1` (전량을 세지 않아 가장 가벼움)

---

## 5. 삭제 (DELETE)

**REST DSL (조건 삭제)**
```json
POST /api_keys/_delete_by_query
{
  "query": { "term": { "apiKey": "abc123" } }
}
```

**MySQL 대응**
```sql (실예제)
DELETE FROM api_keys WHERE api_key = 'abc123';   -- _delete_by_query (조건 삭제)
DELETE FROM api_keys WHERE id = 1;               -- delete by _id     (PK 삭제)
```

| ES | MySQL |
|----|-------|
| `delete()` (`_id` 지정) | `DELETE ... WHERE pk = ?` |
| `deleteByQuery()` | `DELETE ... WHERE 조건` |
| `'refresh' => true` | **대응 개념 없음** (MySQL 은 커밋 즉시 보임) — ES 만의 NRT 이슈 |

**PHP 클라이언트 (_id 로 삭제)**
```php
public function deleteDocument()
{
    $id = 'Y-GLWp8BE3j6GTNTj69k';

    $response = $this->client->delete([
        'index' => 'auth_apikey_dev',
        'id' => $id,
        'refresh' => true          // 삭제 후 바로 검색 반영
    ]);

    $result = $response->asArray();
}
```

**응답(요약)**
```php
[
  "_index" => "auth_apikey_dev",
  "_id" => "xxxxxxx",
  "_version" => 2,
  "result" => "deleted",
  "forced_refresh" => true
]
```

### cf1) `_source.id` 로 삭제하려면 쿼리 사용
`delete`의 `id`는 문서의 `_id`를 사용함. 매핑된 `id`(=`_source.id`)로 지우려면 `deleteByQuery`.
```php
$response = $this->client->deleteByQuery([
    'index' => 'auth_apikey_dev',
    'body' => [
        'query' => [ 'term' => [ 'id' => 1 ] ]
    ]
]);
```
> ES 는 **`_id`(문서 주소)** 와 **`_source.id`(내가 저장한 값)** 가 별개라서 API 가 나뉩니다.
> MySQL 은 `DELETE ... WHERE id = 1` 하나로 끝나는 부분입니다.
> MySQL 로 치면 "내부 rowid 로 지우기" vs "컬럼 조건으로 지우기" 의 차이.

### cf2) 삭제 직후 조회하면 아직 나오는 함정 — NRT
> ES는 **Near Real-Time(NRT)** 이라 저장/삭제해도 즉시 검색에 반영하지 않음.
> 성능을 위해 변경을 모아뒀다가 **기본 1초마다 `refresh`** 할 때 검색 인덱스에 반영함.
>
> → 삭제 요청에 `'refresh' => true` 를 주면 **삭제를 검색에 즉시 반영한 뒤** 응답함.
>
> 💡 **MySQL 과 가장 크게 다른 지점.** MySQL 은 `DELETE` 후 `COMMIT` 하면 그 다음 `SELECT` 에 무조건 반영됩니다.
> ES 는 삭제가 끝났다고 응답해도 **약 1초간 검색 결과에는 남아 있을 수 있습니다.**
> (트랜잭션 격리 문제가 아니라, 검색 인덱스 갱신 주기 문제)

```php
$this->client->delete([
    'index'   => 'auth_apikey_dev',
    'id'      => $id,
    'refresh' => true,     // ← 삭제 후 바로 검색 반영
]);
```

---

## 6. 전체 삭제 (DELETE ALL, 인덱스는 유지)

**REST DSL**
```json
POST /auth_apikey_dev/_delete_by_query
{
  "query": { "match_all": {} }
}
```

**MySQL 대응**
```sql (실예제)
DELETE FROM auth_apikey;      -- _delete_by_query + match_all (한 행씩 지움, 느림)
TRUNCATE TABLE auth_apikey;   -- 인덱스 삭제 후 재생성 (훨씬 빠름) ← 아래 💡 와 같은 이야기
```

**PHP 클라이언트**
```php
public function deleteAllDocument()
{
    $response = $this->client->deleteByQuery([
        'index' => 'auth_apikey_dev',
        'body'  => [
            'query' => [
                'match_all' => (object)[],
            ],
        ],
    ]);

    $result = $response->asArray();
}
```

**응답(요약)**
```php
[
  "took" => 1,
  "total" => 0,
  "deleted" => 0,
  "batches" => 0,
  "version_conflicts" => 0,
  "failures" => []
]
```

### cf) `_delete_by_query` 운영 옵션

`_delete_by_query` 는 **스냅샷을 뜬 뒤 하나씩 지우는** 방식이라, 도중에 다른 요청이 같은 문서를 수정하면
버전 충돌(`version_conflicts`)로 그 문서만 조용히 건너뜁니다. 건수가 많으면 요청이 타임아웃도 납니다.

```
POST /auth_apikey_dev/_delete_by_query?conflicts=proceed&wait_for_completion=false&refresh=true
```

| 옵션 | MySQL 로 치면 | 의미 |
|------|--------------|------|
| `conflicts=proceed` | `DELETE IGNORE` | 버전 충돌이 나도 중단하지 않고 나머지를 계속 삭제 (기본은 중단) |
| `wait_for_completion=false` | 대량 DELETE 를 배치 잡으로 돌리는 것 | 즉시 `task` ID만 반환하고 백그라운드 실행 — **대량 삭제 시 필수** |
| `refresh=true` | (대응 없음 — ES 의 NRT 때문) | 삭제 완료 후 검색에 즉시 반영 |
| `scroll_size` | `DELETE ... LIMIT 1000` 을 반복하는 배치 크기 | 배치 크기 (기본 1000) |

```php
// 백그라운드 실행 후 진행상황 확인
$task = $response['task'];                       // 예: "oTUltX4IQMOUUVeiohTt8A:124"
$this->client->tasks()->get(['task_id' => $task]);
```

> 응답의 `version_conflicts` / `failures` 가 0이 아니면 **일부가 안 지워진 것**이므로 반드시 확인할 것.
>
> 💡 **인덱스를 통째로 비울 거면** `_delete_by_query` + `match_all` 보다
> **인덱스를 삭제하고 매핑과 함께 다시 만드는 편이 훨씬 빠릅니다** (문서를 하나씩 지우지 않으므로).
> 단, 이 경우 매핑·설정이 함께 날아가니 재생성 스크립트를 갖춰둘 것.

---

## 7. 페이지네이션 (Pagination)

```php
$pg = intval($request->input('pg', 1));   // 페이지 번호
$sz = intval($request->input('sz', 10));  // 페이지 크기

$params = [
    'track_total_hits' => true,           // 전체 건수 정확히 카운트
    'from' => ($pg - 1) * $sz,            // 시작 오프셋
    'size' => $sz,
    'query' => [
        'terms' => [
            'no' => $apiNos                // 여러 값 IN 검색
        ]
    ]
];
```

**MySQL 대응**
```sql (실예제)
-- pg=3, sz=10 이면
SELECT * FROM api_keys
WHERE no IN (?, ?, ?)          -- terms
ORDER BY no DESC
LIMIT 10 OFFSET 20;            -- size=10, from=(3-1)*10

SELECT COUNT(*) FROM api_keys WHERE no IN (?, ?, ?);   -- track_total_hits: true
```

| 파라미터 | MySQL 대응 | 설명 |
|----------|-----------|------|
| `from` | `OFFSET` | 시작 위치 `(pg - 1) * sz` |
| `size` | `LIMIT` | 가져올 개수 |
| `track_total_hits` | `SELECT COUNT(*)` 별도 조회 | `true`면 전체 건수를 정확히 반환 (기본은 10,000에서 멈춤) |

### ⚠️ 딥 페이징 한계 — `from + size ≤ 10,000`

`from + size` 가 **10,000을 넘으면 요청 자체가 실패**합니다 (`index.max_result_window` 기본값).
페이지 크기 10 기준 **1,000페이지가 한계**라, 운영에서 목록을 끝까지 넘기면 반드시 밟는 지점입니다.

```
Result window is too large, from + size must be less than or equal to: [10000]
```

원인은 성능입니다. `from: 100000` 은 각 샤드가 **10만 + size 건을 전부 정렬한 뒤 앞부분을 버리는** 방식이라
페이지가 뒤로 갈수록 급격히 무거워집니다. `max_result_window` 를 올리는 건 임시방편일 뿐 권장되지 않습니다.

> **MySQL 에서도 똑같이 겪는 문제입니다.**
> ```sql (실예제)
> SELECT * FROM api_keys ORDER BY no DESC LIMIT 10 OFFSET 100000;
> -- → 10만 건을 정렬해서 읽고 버린 뒤 10건만 반환 (페이지가 뒤로 갈수록 느려짐)
> ```
> 차이는 **MySQL 은 느려질 뿐이고, ES 는 아예 에러로 막는다**는 점입니다.

**대안 ① `search_after`** — 마지막 문서의 정렬값을 커서로 넘겨 다음 페이지를 받음 (무한 스크롤·전량 추출용)

```json
{
  "size": 10,
  "sort": [
    { "no": "desc" },
    { "_shard_doc": "asc" }     // 동점 방지용 tie-breaker (필수)
  ],
  "search_after": [54261872, 0] // 직전 마지막 문서의 sort 값 그대로
}
```
```php
$last = end($response['hits']['hits']);
$params['body']['search_after'] = $last['sort'];   // 다음 페이지 요청에 그대로 전달
```

**MySQL 대응 — 키셋(keyset) 페이지네이션과 완전히 같은 기법**
```sql (실예제)
-- 1페이지
SELECT * FROM api_keys ORDER BY no DESC LIMIT 10;

-- 2페이지: OFFSET 대신 "직전 마지막 값" 을 커서로 사용  (= search_after)
SELECT * FROM api_keys
WHERE no < 54261872          -- ← 직전 페이지 마지막 행의 no
ORDER BY no DESC LIMIT 10;

-- no 가 유니크하지 않다면 tie-breaker 필요 (= _shard_doc)
SELECT * FROM api_keys
WHERE (no, id) < (54261872, 5)
ORDER BY no DESC, id DESC LIMIT 10;
```
| ES | MySQL |
|----|-------|
| `from` / `size` | `LIMIT ... OFFSET ...` (깊어지면 느림) |
| `search_after` | 키셋 페이지네이션 (`WHERE no < 직전값`) |
| `_shard_doc` tie-breaker | 복합 정렬키 (`ORDER BY no DESC, id DESC`) |
| `_pit` (Point In Time) | 스냅샷 격리 트랜잭션(`REPEATABLE READ`) 로 커서 유지하는 것과 비슷 |

> - `search_after` 는 **임의 페이지 점프가 불가**(순차 이동만). 대신 깊이와 무관하게 일정한 속도.
> - 정렬 기준이 유니크하지 않으면 페이지 경계에서 문서가 누락·중복되므로 **tie-breaker를 반드시** 넣을 것.

**대안 ② 전량 추출은 `_pit`(Point In Time) + `search_after`** — 스냅샷을 고정해 페이징 중 데이터가 변해도 일관성 유지. (구버전의 `scroll` 대체)

---

## 8. 수정 (UPDATE)

> `index()`로 같은 `_id`에 다시 저장하면 **문서 전체가 덮어써짐(전체 교체)**.
> 일부 필드만 바꾸려면 `update()`를 사용.

**REST DSL (부분 수정)**
```json
POST /auth_apikey_dev/_update/Y-GLWp8BE3j6GTNTj69k
{
  "doc": {
    "ips": ["127.0.0.1", "10.0.0.5"]
  }
}
```

**MySQL 대응**
```sql (실예제)
-- update() + doc  = 지정한 컬럼만 수정
UPDATE auth_apikey SET ips = ? WHERE id = ?;

-- index() 로 같은 _id 에 다시 저장 = 행을 통째로 갈아끼움
REPLACE INTO auth_apikey (id, api_key, ips) VALUES (?, ?, ?);
--   → 안 넘긴 컬럼은 기본값으로 날아감. ES 의 "전체 교체" 와 정확히 같은 함정.
```

**PHP 클라이언트**
```php
$response = $this->client->update([
    'index' => 'auth_apikey_dev',
    'id'    => $id,
    'body'  => [
        'doc' => [
            'ips' => ['127.0.0.1', '10.0.0.5']   // 이 필드만 갱신
        ]
    ],
    'refresh' => true
]);
```

### upsert (있으면 수정, 없으면 생성)
```php
$response = $this->client->update([
    'index' => 'auth_apikey_dev',
    'id'    => $apiKey,
    'body'  => [
        'doc'           => [ 'ips' => $ips ],   // 있으면 이 값으로 수정
        'doc_as_upsert' => true                  // 없으면 doc 내용으로 새로 생성
    ]
]);
```

**MySQL 대응 — upsert**
```sql (실예제)
INSERT INTO auth_apikey (api_key, ips) VALUES (?, ?)
ON DUPLICATE KEY UPDATE ips = VALUES(ips);
--   있으면 UPDATE, 없으면 INSERT  =  'doc_as_upsert' => true
```

> `result` 값으로 결과 구분: `created`(신규) / `updated`(수정) / `noop`(변경 없음).
> MySQL 의 `ON DUPLICATE KEY UPDATE` 가 반환하는 affected rows (1=INSERT, 2=UPDATE, 0=변경없음) 와 같은 역할.

---

## 9. bool 쿼리 상세 (must / should / must_not / filter)

```json
GET api_keys/_search
{
  "query": {
    "bool": {
      "must":     [ { "term": { "apiKey": "abc123" } } ],
      "filter":   [ { "term": { "ips": "192.168.0.2" } } ],
      "should":   [ { "term": { "id": 1 } } ],
      "must_not": [ { "term": { "id": 99 } } ]
    }
  }
}
```

**MySQL 대응**
```sql (실예제)
SELECT *, (id = 1) AS score      -- should   : 만족하면 가점 (정렬용)
FROM api_keys
WHERE api_key = 'abc123'         -- must
  AND ip      = '192.168.0.2'    -- filter
  AND id     <> 99               -- must_not
ORDER BY score DESC;
```
> ⭐ `must` 와 `filter` 는 **SQL 로 옮기면 둘 다 그냥 `AND`** 입니다.
> 차이는 결과가 아니라 **점수 계산 여부**뿐이라, SQL 에는 대응 문법이 없습니다.
> `should` 도 마찬가지로 "결과에 포함될 조건" 이 아니라 **정렬 가중치**에 가깝습니다.

| 절 | 의미 | MySQL 대응 | 점수(score) |
|----|------|----------|-------------|
| `must` | 반드시 만족 (AND) | `AND` | O (점수 필요한 경우 사용) |
| `filter` | 반드시 만족 (AND) | `AND` | X (캐싱 유리, 빠름) |
| `should` | 만족하면 가점 (다른필드의 OR) | `OR` / `ORDER BY (조건) DESC` | O |
| `must_not` | 만족하면 제외 | `NOT` / `<>` | X |

> - **점수가 필요 없는 정확 일치 조건은 `filter`** 를 쓰는 게 성능상 유리 (결과 캐싱).
> - `should`만 단독으로 쓰면 그중 최소 1개는 만족해야 함(`minimum_should_match` 조절 가능).

### ⭐ "일치하는 걸 찾는 건데 점수 계산이 왜 필요하지?"

**정확 일치 검색에는 점수가 필요 없습니다.** 그래서 `filter` 가 따로 있는 겁니다.

`must` 는 애초에 **정확 일치용으로 만들어진 절이 아닙니다.**
"이 조건이 결과 순위에 영향을 준다" 는 뜻의 절이라 점수를 계산할 뿐입니다.
거기에 `term` 을 넣으면 **계산은 하는데 결과가 전부 똑같이 나옵니다.**

```
{ "must": [ { "term": { "apiKey": "abc123" } } ] }

→ 매칭된 문서 전부 _score = 1.0   (다 똑같음 → 정렬에 쓸모없음 → 계산 비용만 낭비)
```

#### 점수가 실제로 의미 있는 경우

점수는 **"얼마나 잘 맞는지가 문서마다 다를 때"** 필요합니다.

**① 단어가 몇 번 나오나 / 문서가 얼마나 짧은가** (`match`)
```
"아이폰" 검색

"아이폰"                       → _score 3.2   짧은 제목 = 그 단어가 핵심
"아이폰 16 Pro 256GB 자급제"    → _score 1.8
"삼성 갤럭시 (아이폰 아님)"      → _score 0.4   곁다리로 언급됨
```
셋 다 조건은 만족하지만 **사용자가 원하는 순서가 다릅니다.** 이 순서를 정하는 게 점수입니다.

**② 여러 조건 중 몇 개나 만족했나** (`should`)
```json
"should": [
  { "term": { "brand": "apple" } },
  { "term": { "isNew": true } }
]
```
둘 다 만족한 상품이 하나만 만족한 상품보다 위로 올라갑니다.

**③ 어느 필드에서 맞았나** (`boost`)
```json
{ "multi_match": { "query": "아이폰", "fields": ["title^3", "description"] } }
```
제목에서 맞으면 본문에서 맞은 것보다 3배 가점.

#### 정리

| 조건 | 문서마다 점수가 다른가 | 쓸 절 |
|------|------------------|-------|
| `term`, `terms`, `range`, `exists` | ✕ (맞거나 틀리거나) | **`filter`** |
| `match`, `multi_match`, `match_phrase` | O (얼마나 잘 맞는지) | **`must`** |

> **`term` 을 `must` 에 넣는 건 낭비입니다.** 이 문서의 apiKey·ips 조회는 전부 `filter` 가 맞습니다.
>
> 반대로 **검색창 검색(`match`)을 `filter` 에 넣으면** 결과는 나오지만 **순서가 뒤죽박죽**이 됩니다.
> 관련도 순 정렬이 사라지기 때문입니다.


ex)
예를 들어 사용자가 "아이폰" 을 검색했다고 해보겠습니다.

상품이 다음과 같다면

| 상품명 | status | price |
|--------| -------| ------|
| 아이폰 16 Pro | OPEN | 1,500,000 |
| 아이폰 케이스 | OPEN | 20,000 |
| 아이폰 15 | SOLDOUT | 1,200,000 |

검색 조건이

- 제목에 "아이폰"
- 판매중(OPEN)
- 가격 100,000원 이상

이라면, status와 price는 단순히 통과/탈락만 시킵니다.

| 상품명 | 결과 | 사유 |
|--------|------|------|
| 아이폰 16 Pro | ✅ | - |
| 아이폰 케이스 | ❌ | 가격 미달 |
| 아이폰 15 | ❌ | 품절 |

여기서

- 판매중인 상품이 더 높은 점수를 받을 필요가 있을까요? → ❌
- 가격이 비싸다고 더 높은 점수를 받을 필요가 있을까요? → ❌

그냥 조건을 만족하면 포함, 아니면 제외입니다.

이 경우 filter를 사용합니다.

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "title": "아이폰"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "status": "OPEN"
          }
        },
        {
          "range": {
            "price": {
              "gte": 100000
            }
          }
        }
      ]
    }
  }
}
```

**MySQL 대응**
```sql (실예제)
SELECT *,
       MATCH(name) AGAINST('아이폰') AS score   -- must  : 관련도 점수를 만드는 조건
FROM products
WHERE MATCH(name) AGAINST('아이폰')             -- must
  AND status = 'OPEN'                           -- filter : 통과/탈락만
  AND price >= 100000                           -- filter : range
ORDER BY score DESC;
```
> `status`·`price` 는 `ORDER BY` 에 안 들어갑니다 = **점수에 관여하지 않는다** = `filter`.
> `MATCH ... AGAINST` 만 점수에 들어갑니다 = `must`.

기준을 한 줄로 정리하면

| 목적 | 사용할 절 | MySQL 에서 그 조건이 놓이는 자리 | 대표 예시 |
|------|-----------|------------------------------|-----------|
| 조건을 **만족하는지만** 확인 | `filter` | `WHERE` 에만 | `status = 'OPEN'`, `price >= 100000`, `no = 1234` |
| 결과의 **순위(관련도)** 를 결정 | `must` | `WHERE` + `ORDER BY score` | `MATCH() AGAINST()` (= `match`, `multi_match`) |

---

## 10. 정렬 (SORT)

```php
$response = $this->client->search([
    'index' => 'auth_apikey_dev',
    'body'  => [
        'sort' => [
            ['id' => ['order' => 'desc']]   // id 내림차순
        ],
        'query' => [ 'match_all' => (object)[] ]
    ]
]);
```

**MySQL 대응**
```sql (실예제)
SELECT * FROM auth_apikey ORDER BY id DESC;
```
| ES | MySQL |
|----|-------|
| `'sort' => [['id' => ['order' => 'desc']]]` | `ORDER BY id DESC` |
| 정렬키 2개 | `ORDER BY a DESC, b ASC` |
| `"sort": ["_doc"]` | `ORDER BY` 없이 조회 (순서 보장 안 함, 가장 빠름) |

> `text` 필드는 기본적으로 정렬 불가 → 정렬이 필요하면 `keyword`(또는 숫자/날짜) 필드로 정렬.
> (MySQL 은 `TEXT` 컬럼도 `ORDER BY` 가 되므로, **여기서 ES 가 더 까다롭습니다.**)

---

## 11. 존재 여부 확인 (EXISTS)

```php
// _id 로 문서 존재 여부만 빠르게 확인
$exists = $this->client->exists([
    'index' => 'auth_apikey_dev',
    'id'    => $id
])->asBool();   // true / false
```

**MySQL 대응**
```sql (실예제)
SELECT EXISTS (SELECT 1 FROM auth_apikey WHERE id = ?) AS found;      -- exists() API
SELECT EXISTS (SELECT 1 FROM auth_apikey WHERE api_key = ? LIMIT 1);  -- 조건으로 존재 확인
```

> 조건(apiKey 등)으로 존재만 확인할 땐 `search` + `"size": 0` + `terminate_after: 1` 로 가볍게 조회하거나 `count` 사용.
> (MySQL 에서 `SELECT COUNT(*)` 대신 `SELECT EXISTS(... LIMIT 1)` 을 쓰는 것과 같은 이유 — 전부 세지 않으려고.)

### ⚠️ 이름이 비슷한 둘 — `exists()` API vs `exists` 쿼리

| 구분 | 확인 대상 | 예시 |
|------|-----------|------|
| `exists()` **API** | **문서**가 있는지 (`_id` 기준) | 위 PHP 코드 |
| `exists` **쿼리** | 문서 안에 **필드에 값이 있는지** | 아래 |

```json
// ips 필드에 값이 있는 문서만 (null, [] 는 제외됨)
{ "query": { "bool": { "filter": [ { "exists": { "field": "ips" } } ] } } }

// 반대로 ips 가 비어 있는 문서 찾기 (must_not)
{ "query": { "bool": { "must_not": [ { "exists": { "field": "ips" } } ] } } }
```

**MySQL 대응**
```sql (실예제)
SELECT * FROM api_keys WHERE ips IS NOT NULL;   -- exists 쿼리
SELECT * FROM api_keys WHERE ips IS     NULL;   -- must_not + exists
```

> ES에는 MySQL의 `IS NULL` 이 없습니다. `null` / `[]` / 필드 자체가 없음이 전부 **"값 없음"** 으로 동일 취급되며,
> `must_not` + `exists` 가 `IS NULL` 대응입니다.
> (MySQL 은 `NULL` 과 빈 문자열 `''` 을 구분하지만, **ES 는 구분하지 않는다**는 점이 차이.)

---

## 12. 클라이언트 초기화 (PHP 연결 설정)

```php
use Elastic\Elasticsearch\ClientBuilder;

$this->client = ClientBuilder::create()
    ->setHosts(['http://localhost:9200'])
    // ->setBasicAuthentication('elastic', 'password')  // 인증 사용 시
    // ->setApiKey('base64EncodedApiKey')               // API Key 인증 시
    ->build();
```

> 라이브러리: `elasticsearch/elasticsearch` (공식 PHP 클라이언트).
> `composer require elasticsearch/elasticsearch`

### cf) ES 7.x 코드와 다른 점 (8.x 기준)

7.x 예제를 복붙하면 바로 막히는 지점들입니다.

| 항목 | 7.x | 8.x (이 문서) |
|------|-----|---------------|
| 응답 타입 | `array` (바로 `$r['hits']`) | `Elasticsearch` 객체 → **`asArray()` / `asBool()` / `asString()` 필요** |
| 네임스페이스 | `Elasticsearch\` | `Elastic\Elasticsearch\` |
| 매핑 타입 | `_doc` 등 type 개념 잔존 | **완전 제거** (`PUT idx/_doc/1` 만) |
| 기본 통신 | http | **https + 보안 기본 활성화** (로컬 개발 시 인증서 설정 필요) |

```php
// 8.x 에서 응답을 배열처럼 바로 쓰면 에러
$response = $this->client->search([...]);
$total = $response['hits']['total']['value'];   // ArrayAccess 로 동작은 하지만
$result = $response->asArray();                 // 배열로 변환해 쓰는 쪽이 명확
```

> 자체 서명 인증서를 쓰는 개발 서버라면:
> ```php
> ClientBuilder::create()
>     ->setHosts(['https://localhost:9200'])
>     ->setBasicAuthentication('elastic', $password)
>     ->setCABundle('/path/to/http_ca.crt')   // 또는 ->setSSLVerification(false) — 개발 전용
>     ->build();
> ```

---

## 13. refresh 옵션 3종 정리

저장/수정/삭제 요청 시 `refresh` 값에 따라 검색 반영 시점이 달라짐.

| 값 | 동작 | 사용 상황 |
|----|------|-----------|
| `false` (기본) | 다음 주기(기본 1초)에 반영 | 일반적인 대량 처리 (성능 ↑) |
| `true` | **즉시** 반영 후 응답 | 저장/삭제 직후 바로 조회해야 할 때 |
| `'wait_for'` | 다음 refresh까지 **기다렸다가** 응답 | 즉시성 필요 + `true`의 부하는 피하고 싶을 때 |

> `refresh => true`는 매번 강제 refresh라 **자주 쓰면 성능 저하**. 실시간성이 꼭 필요한 지점에만 사용.
>
> 💡 **MySQL 에는 이 개념 자체가 없습니다.** `COMMIT` 하면 그 즉시 다음 `SELECT` 에 보이기 때문입니다.
> ES 는 "저장 완료 응답"과 "검색에 보임" 사이에 기본 1초의 간극이 있고, `refresh` 가 그 간극을 조절하는 옵션입니다.
> → **MySQL 감각으로 `저장 → 바로 조회` 코드를 짜면 ES 에서는 결과가 안 나옵니다.** (테스트 코드에서 특히 자주 걸림)

---

## 14. 예외 처리 정리

```php
use Elastic\Elasticsearch\Exception\ClientResponseException;   // 4xx (400/404/409 등)
use Elastic\Elasticsearch\Exception\ServerResponseException;   // 5xx (서버 오류)
use Elastic\Elasticsearch\Exception\MissingParameterException; // 필수 파라미터 누락

try {
    // ... ES 요청
} catch (ClientResponseException $e) {
    $status = $e->getResponse()->getStatusCode();  // 409 → 중복, 404 → 없음
    // 상태코드별 분기 처리
}
```

| 상태 코드 | 의미 | 대표 상황 | MySQL 대응 에러 |
|-----------|------|-----------|----------------|
| 400 | 잘못된 요청 | 매핑/쿼리 문법 오류 | `1064` (SQL syntax) / `1054` (Unknown column) |
| 404 | 없음 | 없는 `_id` 조회·삭제 | (에러 아님 — 0 rows affected) |
| 409 | 충돌 | `op_type => create` 중복 저장 | `1062` Duplicate entry |

---
---

# 📊 집계 (Aggregation)

> ### 여기부터는 **"찾기"가 아니라 "세고 · 묶고 · 계산하기"**
> MySQL의 `GROUP BY` + 집계함수(`COUNT`, `SUM`, `AVG`…)에 해당하는 영역.
> 검색(`query`)이 **"어떤 문서를 볼까"** 라면, 집계(`aggs`)는 **"그 문서들로 무엇을 계산할까"** 입니다.

```
                  ┌── query ──→ 조건에 맞는 문서 집합을 고름
검색 요청 ─────────┤
                  └── aggs  ──→ 그 집합을 대상으로 묶고 계산  ← 여기
```

---

## 15. 집계 기본 구조 — `size: 0` + `aggs`

**REST DSL** — 상태값별 문서 개수 세기 (`GROUP BY status`)
```json
GET products/_search
{
  "size": 0,                       // ← 문서 본문은 필요 없음 (집계 결과만)
  "aggs": {
    "by_status": {                 // ← 집계 이름 (내가 정함, 응답 키가 됨)
      "terms": { "field": "status" }
    }
  }
}
```

**MySQL 대응**
```sql (실예제)
SELECT status AS `key`, COUNT(*) AS doc_count
FROM products
GROUP BY status;
```
| ES | MySQL |
|----|-------|
| `"size": 0` | 원본 컬럼 없이 집계값만 SELECT |
| `terms` 집계 | `GROUP BY` |
| 응답의 `buckets` / `doc_count` | 결과 행 / `COUNT(*)` |

**응답**
```json
{
  "hits": { "total": { "value": 120 }, "hits": [] },   // size:0 이라 비어 있음
  "aggregations": {
    "by_status": {
      "doc_count_error_upper_bound": 0,
      "sum_other_doc_count": 0,
      "buckets": [
        { "key": "OPEN",    "doc_count": 95 },
        { "key": "SOLDOUT", "doc_count": 25 }
      ]
    }
  }
}
```

**PHP 클라이언트**
```php
$response = $this->client->search([
    'index' => 'products',
    'body'  => [
        'size' => 0,
        'aggs' => [
            'by_status' => [
                'terms' => ['field' => 'status']
            ]
        ]
    ]
])->asArray();

foreach ($response['aggregations']['by_status']['buckets'] as $b) {
    echo "{$b['key']} : {$b['doc_count']}건\n";   // OPEN : 95건
}
```

> ⚠️ **`size: 0` 을 빠뜨리면** 쓰지도 않을 문서 10건을 매번 같이 실어옵니다. 집계만 필요하면 항상 `size: 0`.
> `aggs` 는 `aggregations` 의 축약형이며 둘 다 동작합니다.

### 집계의 3가지 종류

| 종류 | 하는 일 | 결과 | 대표 | MySQL 대응 |
|------|---------|------|------|-----------|
| **Metric** | 숫자 하나로 **계산** | 값 | `sum`, `avg`, `cardinality` | 집계함수 `SUM()`, `AVG()`, `COUNT(DISTINCT)` |
| **Bucket** | 조건별로 **묶음(그룹)** 생성 | 버킷 목록 | `terms`, `range`, `date_histogram` | `GROUP BY` |
| **Pipeline** | 다른 집계의 **결과를 다시 가공** | 값/필터 | `bucket_selector`, `cumulative_sum` | `HAVING`, 윈도우 함수 |

> 실무의 대부분은 **Bucket 으로 묶고 → 그 안에 Metric 을 넣는** 조합입니다 (→ 18장).

---

## 16. Metric 집계 — 숫자 계산

```json
{
  "size": 0,
  "aggs": {
    "total_price": { "sum":         { "field": "price" } },
    "avg_price":   { "avg":         { "field": "price" } },
    "max_price":   { "max":         { "field": "price" } },
    "ip_count":    { "value_count": { "field": "ips" } },
    "uniq_ip":     { "cardinality": { "field": "ips" } },
    "price_stats": { "stats":       { "field": "price" } }
  }
}
```

**MySQL 대응**
```sql (실예제)
SELECT SUM(price), AVG(price), MAX(price),
       COUNT(ip)          AS ip_count,   -- value_count
       COUNT(DISTINCT ip) AS uniq_ip     -- cardinality (ES 는 ⚠️ 근사치)
FROM products;
```

| 집계 | MySQL 대응 | 설명 |
|------|----------|------|
| `sum` / `avg` / `min` / `max` | `SUM()` / `AVG()` / `MIN()` / `MAX()` | 숫자 필드 계산 |
| `value_count` | `COUNT(field)` | **값의 개수** (배열이면 원소를 각각 셈) |
| `cardinality` | `COUNT(DISTINCT field)` | **고유값 개수** — ⚠️ **근사치** |
| `stats` | `COUNT/MIN/MAX/AVG/SUM` 을 한 SELECT 에 | `count`, `min`, `max`, `avg`, `sum` 을 한 번에 |
| `extended_stats` | + `VARIANCE()`, `STDDEV()` | `stats` + `variance`, `std_deviation` |
| `percentiles` | (MySQL 8 윈도우함수로 흉내) | 응답시간 p95, p99 등 — ⚠️ 근사치 |
| `top_hits` | 그룹별 상위 N행 (`ROW_NUMBER() OVER (PARTITION BY ...)`) | 버킷 안에서 상위 N개 **문서 원본**을 꺼냄 |

**응답 형태 — Metric 은 `buckets` 가 없고 `value` 하나**
```json
"aggregations": {
  "uniq_ip":     { "value": 37 },
  "total_price": { "value": 18500000 },
  "price_stats": { "count": 120, "min": 1000, "max": 1500000, "avg": 154166.6, "sum": 18500000 }
}
```
```php
$uniq = $response['aggregations']['uniq_ip']['value'];   // 37
```

> ⚠️ **`cardinality` 는 정확한 값이 아닙니다.** HyperLogLog++ 알고리즘으로 메모리를 아끼는 대신 오차를 허용합니다.
> `precision_threshold`(기본 3000, 최대 40000) **이하 범위에서는 거의 정확**하고, 그 이상부터 오차가 생깁니다.
> ```json
> "uniq_ip": { "cardinality": { "field": "ips", "precision_threshold": 10000 } }
> ```
> **정산·과금처럼 정확한 distinct 가 필요하면** `composite` 집계로 전량을 훑거나 RDB에서 계산하세요.
>
> ⭐ **MySQL 과의 결정적 차이.** `COUNT(DISTINCT ip)` 는 **항상 정확한 값**이지만,
> ES 의 `cardinality` 는 **정확하지 않을 수 있습니다.** 숫자를 그대로 정산에 쓰면 안 됩니다.

---

## 17. Bucket 집계 — 그룹으로 묶기

### ① `terms` — 값별로 묶기 (가장 많이 씀)

```json
"by_status": {
  "terms": {
    "field": "status",
    "size": 20,                        // 상위 몇 개 버킷까지 (기본 10)
    "order": { "_count": "desc" }      // 정렬 기준
  }
}
```

**MySQL 대응**
```sql (실예제)
SELECT status, COUNT(*) AS cnt
FROM products
GROUP BY status
ORDER BY cnt DESC        -- order: { "_count": "desc" }
LIMIT 20;                -- size: 20
```

| `order` | MySQL 대응 | 의미 |
|---------|-----------|------|
| `{ "_count": "desc" }` | `ORDER BY COUNT(*) DESC` | 건수 많은 순 (기본) |
| `{ "_key": "asc" }` | `ORDER BY status ASC` | 값 이름순 |
| `{ "avg_price": "desc" }` | `ORDER BY AVG(price) DESC` | **하위 집계 결과 기준** 정렬 (→ 18장) |

### ② `range` / `histogram` — 숫자 구간으로 묶기

```json
"by_price": {
  "range": {
    "field": "price",
    "ranges": [
      { "to": 10000 },                        // ~ 10,000 미만
      { "from": 10000, "to": 100000 },        // 10,000 이상 ~ 100,000 미만
      { "from": 100000 }                      // 100,000 이상
    ]
  }
}
```
```json
"by_price_step": {
  "histogram": { "field": "price", "interval": 50000 }   // 5만원 단위로 자동 구간
}
```
> `from` 은 **이상(포함)**, `to` 는 **미만(제외)** 입니다.

**MySQL 대응**
```sql (실예제)
-- range agg = CASE WHEN 으로 구간 만들기
SELECT CASE WHEN price < 10000 THEN '~10000'
            WHEN price < 100000 THEN '10000~100000'
            ELSE '100000~' END AS price_range, COUNT(*)
FROM products GROUP BY price_range;

-- histogram agg = 나눗셈으로 구간 만들기 (5만원 단위)
SELECT FLOOR(price / 50000) * 50000 AS bucket, COUNT(*)
FROM products GROUP BY bucket ORDER BY bucket;
```

### ③ `date_histogram` — 날짜 단위로 묶기

```json
"daily": {
  "date_histogram": {
    "field": "createdAt",
    "calendar_interval": "1d",
    "time_zone": "+09:00",              // ⚠️ 한국은 필수 (아래 함정 참고)
    "format": "yyyy-MM-dd",
    "min_doc_count": 0,                 // 0건인 날도 버킷으로 출력
    "extended_bounds": {                // 데이터가 없는 앞뒤 구간까지 채움
      "min": "2026-08-01", "max": "2026-08-31"
    }
  }
}
```

**MySQL 대응**
```sql (실예제)
SELECT DATE(created_at) AS day, COUNT(*) AS doc_count   -- 1M 이면 DATE_FORMAT(.., '%Y-%m')
FROM products
GROUP BY day ORDER BY day;
```
> ⭐ `min_doc_count: 0` 은 **MySQL 에 대응이 없습니다.** "0건인 날도 0으로 표시" 하려면
> 날짜 테이블을 만들어 `LEFT JOIN` 해야 하는데, ES 는 옵션 한 줄이면 됩니다.

| 옵션 | 설명 |
|------|------|
| `calendar_interval` | `1m`, `1h`, `1d`, `1w`, `1M`, `1q`, `1y` — **달력 기준** (월의 길이 다름 반영) |
| `fixed_interval` | `30s`, `90m`, `24h` — **고정 길이** (달력 무시) |
| `min_doc_count: 0` | 데이터 없는 구간도 0으로 출력 (그래프용) |

### ④ `filters` — 내가 정의한 조건별로 묶기

`terms` 로 안 나뉘는 임의 조건을 그룹으로 만들 때 씁니다.

```json
"by_segment": {
  "filters": {
    "filters": {
      "고가":  { "range": { "price": { "gte": 1000000 } } },
      "품절":  { "term":  { "status": "SOLDOUT" } }
    }
  }
}
```

**MySQL 대응 — 조건별 카운트**
```sql (실예제)
SELECT SUM(price >= 1000000)   AS `고가`,
       SUM(status = 'SOLDOUT') AS `품절`
FROM products;
```
> `filters` 는 버킷이 **서로 겹칠 수 있습니다** (한 문서가 고가이면서 품절일 수 있음).
> 그래서 `CASE WHEN` 보다 위의 `SUM(조건)` 방식이 동작과 일치합니다.

### ⑤ `nested` — 배열 객체 안을 집계

`nested` 타입 필드는 **`nested` 집계로 감싸야** 안이 보입니다 (0장의 `object` vs `nested` 와 같은 이유).

```json
"items_agg": {
  "nested": { "path": "items" },
  "aggs": {
    "by_name": { "terms": { "field": "items.name" } }
  }
}
```

> MySQL 로 치면 `JOIN order_items ... GROUP BY i.name` 입니다.
> `"path": "items"` 가 조인 자리, 그 안의 `terms` 가 `GROUP BY` 자리.

---

## 18. 집계 중첩 & `query` 와의 조합

### 중첩 — Bucket 안에 Metric (실무의 90%)

**"상태별 건수 + 상태별 평균가 + 상태별 고유 IP 수"**

```json
GET products/_search
{
  "size": 0,
  "query": {                                   // ← 집계 대상 문서를 먼저 좁힘
    "bool": { "filter": [ { "range": { "createdAt": { "gte": "2026-08-01" } } } ] }
  },
  "aggs": {
    "by_status": {
      "terms": { "field": "status", "size": 10, "order": { "avg_price": "desc" } },
      "aggs": {                                // ← 각 버킷 안에서 다시 계산
        "avg_price": { "avg":         { "field": "price" } },
        "uniq_ip":   { "cardinality": { "field": "clientIp" } }
      }
    }
  }
}
```

**MySQL 대응 — 이 쿼리 하나로 전부 설명됩니다**
```sql (실예제)
SELECT status,                                     -- terms (Bucket)
       COUNT(*)                  AS doc_count,
       AVG(price)                AS avg_price,     -- 버킷 안 Metric
       COUNT(DISTINCT client_ip) AS uniq_ip        -- 버킷 안 Metric
FROM products
WHERE created_at >= '2026-08-01'                   -- query
GROUP BY status                                    -- terms
ORDER BY avg_price DESC                            -- order
LIMIT 10;                                          -- size
```

**응답**
```json
"by_status": {
  "buckets": [
    { "key": "OPEN",    "doc_count": 95, "avg_price": { "value": 180000 }, "uniq_ip": { "value": 31 } },
    { "key": "SOLDOUT", "doc_count": 25, "avg_price": { "value": 120000 }, "uniq_ip": { "value": 12 } }
  ]
}
```
```php
foreach ($response['aggregations']['by_status']['buckets'] as $b) {
    printf("%s: %d건, 평균 %s원, 고유IP %d\n",
        $b['key'], $b['doc_count'], number_format($b['avg_price']['value']), $b['uniq_ip']['value']);
}
```

> ⭐ **`query` 는 집계에도 그대로 적용됩니다.** 위 예시의 집계는 "8월 1일 이후 문서"만 대상으로 계산됩니다.
> SQL 로 치면 `WHERE createdAt >= '2026-08-01' GROUP BY status`.

### `HAVING` 이 필요하면 — `bucket_selector` (Pipeline)

버킷을 만든 **뒤에** 건수·합계로 버킷 자체를 걸러냅니다.

```json
"by_apikey": {
  "terms": { "field": "apiKey", "size": 100 },
  "aggs": {
    "call_count": { "value_count": { "field": "apiKey" } },
    "over_1000": {
      "bucket_selector": {
        "buckets_path": { "cnt": "call_count" },
        "script": "params.cnt > 1000"          // HAVING COUNT(*) > 1000
      }
    }
  }
}
```

**MySQL 대응**
```sql (실예제)
SELECT api_key, COUNT(*) AS call_count
FROM api_logs
GROUP BY api_key
HAVING call_count > 1000        -- ← bucket_selector 가 이 자리
LIMIT 100;                      -- terms 의 size: 100
```
> ⚠️ 순서 주의: `bucket_selector` 는 **`terms` 의 `size` 로 자른 뒤**에 걸러냅니다.
> MySQL 로 치면 `HAVING` 이 `LIMIT` **뒤에** 적용되는 셈이라, `size` 가 작으면 조건에 맞는 버킷이 누락될 수 있습니다.

---

## 19. ⚠️ 집계 함정 (여기서 대부분 틀림)

**① `text` 필드는 집계 불가**
```
Fielddata is disabled on text fields by default. Set fielddata=true ... or use a keyword field instead
```
→ `fielddata: true` 를 켜지 말고 **`.keyword` 멀티필드로 집계**하세요 (fielddata 는 힙을 크게 먹습니다).
```json
"terms": { "field": "title.keyword" }
```

> MySQL 은 `TEXT` 컬럼도 그냥 `GROUP BY` 가 되지만, ES 는 안 됩니다.

**② `terms` 는 기본 상위 10개만 나옵니다**
전체가 다 나온 줄 알고 합계를 내면 틀립니다. 응답의 `sum_other_doc_count` 가 **버킷에 안 담긴 나머지 건수**입니다.
```json
"sum_other_doc_count": 4821    // ← 0이 아니면 잘린 것
```
→ `size` 를 올리거나, 전량이 필요하면 `composite` 집계로 페이징하세요.

> **MySQL 감각으로 오면 반드시 틀리는 부분.** MySQL 은 `LIMIT` 이 없으면 그룹이 전부 나오지만,
> ES 의 `terms` 집계는 항상 `ORDER BY COUNT(*) DESC LIMIT 10` 이 붙어 있는 상태입니다.

**③ `terms` 의 `doc_count` 는 샤드 근사치일 수 있음**
각 샤드가 자기 상위 N개만 올려보내므로, 샤드가 여러 개면 순위·건수가 어긋날 수 있습니다.
`doc_count_error_upper_bound` 가 **오차 상한**입니다. 0이면 정확.
→ `shard_size` 를 `size` 보다 크게 주면 완화됩니다 (`"shard_size": 1000`).

> MySQL 의 `COUNT(*)` 는 **언제나 정확합니다.** ES 의 `doc_count` 는 샤드가 여러 개면 근사치일 수 있습니다.
> → **집계 숫자를 그대로 정산·과금에 쓰면 안 되는 이유.**

**④ `cardinality` / `percentiles` 는 근사치** (→ 16장)

**⑤ `date_histogram` 의 `time_zone` 미지정 = 날짜가 밀림**
ES 내부는 **UTC 기준**이라, 한국 시간 `2026-08-06 08:00` 은 UTC 로 `2026-08-05 23:00` 입니다.
`time_zone` 을 안 주면 **오전 9시 이전 데이터가 전날 버킷에 들어갑니다.**
```json
"time_zone": "+09:00"      // 일별·월별 집계에는 사실상 필수
```

**⑥ 버킷 수 폭발**
`terms` 의 `size` 를 크게 주거나 중첩을 깊게 하면 `search.max_buckets`(기본 65,536) 초과로 실패합니다.
```
Trying to create too many buckets. Must be less than or equal to: [65536]
```
→ 설정을 올리기 전에 **집계 범위를 `query` 로 먼저 좁히세요.**

**⑦ 집계는 검색보다 훨씬 무겁습니다**
전체 문서를 훑어 계산하므로, **`query` 로 대상을 좁히고 `size: 0`** 을 주는 것이 기본입니다.
실시간 화면에서 매번 돌릴 값이면 결과를 캐싱하거나 별도 집계 인덱스를 두는 편이 낫습니다.

---

## 20. MySQL ↔ 집계 대응표 & 실전 예시

| MySQL | Elasticsearch |
|-----|---------------|
| `GROUP BY status` | `terms` agg |
| `COUNT(*)` | 버킷의 `doc_count` |
| `COUNT(DISTINCT ip)` | `cardinality` (근사) |
| `SUM/AVG/MIN/MAX(price)` | `sum` / `avg` / `min` / `max` agg |
| `WHERE ... GROUP BY ...` | `query` + `aggs` |
| `HAVING COUNT(*) > 1000` | `bucket_selector` |
| `ORDER BY cnt DESC LIMIT 10` | `terms` 의 `order` + `size` |
| `GROUP BY DATE(createdAt)` | `date_histogram` (+ `time_zone`) |
| `GROUP BY CASE WHEN ...` | `range` / `filters` agg |
| `JOIN 자식테이블 ... GROUP BY` | `nested` agg |
| `SELECT` 절에 원본 컬럼 없음 | `"size": 0` |

### 실전 ① apiKey 인덱스 — 등록 현황 한눈에 보기

```php
$response = $this->client->search([
    'index' => $this->authApiKeyIndex,
    'body'  => [
        'size' => 0,
        'aggs' => [
            'uniq_ip'    => ['cardinality' => ['field' => 'ips']],       // 등록된 고유 IP 수
            'ip_total'   => ['value_count' => ['field' => 'ips']],       // 총 IP 등록 건수
            'top_ip'     => ['terms' => ['field' => 'ips', 'size' => 10]], // 많이 쓰인 IP TOP 10
        ]
    ]
])->asArray();

$agg = $response['aggregations'];
echo "고유 IP: {$agg['uniq_ip']['value']} / 총 등록: {$agg['ip_total']['value']}\n";
```

**MySQL 대응**
```sql (실예제)
SELECT COUNT(DISTINCT ip) AS uniq_ip,  -- cardinality
       COUNT(*)           AS ip_total  -- value_count
FROM api_keys;

SELECT ip, COUNT(*) AS doc_count       -- top_ip : 많이 쓰인 IP TOP 10
FROM api_keys GROUP BY ip ORDER BY doc_count DESC LIMIT 10;
```

> `ips` 는 배열 필드지만 집계에서는 **원소 하나하나가 개별 값으로 계산**됩니다.
> 그래서 `top_ip` 버킷의 `doc_count` 는 "그 IP를 등록한 apiKey 문서 수"가 됩니다.

### 실전 ② 중복 IP 사용 apiKey 찾기 (`HAVING`)

```json
{
  "size": 0,
  "aggs": {
    "by_ip": {
      "terms": { "field": "ips", "size": 1000 },
      "aggs": {
        "dup_only": {
          "bucket_selector": {
            "buckets_path": { "cnt": "_count" },
            "script": "params.cnt > 1"        // 2개 이상 apiKey가 쓰는 IP만
          }
        }
      }
    }
  }
}
```

**MySQL 대응**
```sql (실예제)
SELECT ip, COUNT(*) AS cnt
FROM api_keys
GROUP BY ip
HAVING cnt > 1                 -- ← bucket_selector
LIMIT 1000;                    -- terms 의 size
```

### 실전 ③ 일자별 등록 추이 (그래프용)

```json
{
  "size": 0,
  "query": { "range": { "createdAt": { "gte": "now-30d/d" } } },
  "aggs": {
    "daily": {
      "date_histogram": {
        "field": "createdAt",
        "calendar_interval": "1d",
        "time_zone": "+09:00",
        "format": "yyyy-MM-dd",
        "min_doc_count": 0
      }
    }
  }
}
```

```sql (실예제)
-- MySQL 대응
SELECT DATE(created_at) AS day, COUNT(*)
FROM auth_apikey
WHERE created_at >= CURDATE() - INTERVAL 30 DAY   -- "now-30d/d"
GROUP BY day ORDER BY day;
```

```php
foreach ($response['aggregations']['daily']['buckets'] as $b) {
    // key_as_string 이 format 적용된 사람이 읽을 수 있는 값
    echo "{$b['key_as_string']} : {$b['doc_count']}\n";   // 2026-08-06 : 12
}
```

> 버킷 응답의 `key` 는 **epoch milliseconds**, `key_as_string` 이 `format` 이 적용된 문자열입니다.

---

## 핵심 요약

- **완전 일치 검색이 필요한 필드는 `keyword`** 로 매핑 (`text` ✕). 대소문자 무시가 필요하면 `normalizer`.
- **`query`는 항상 필요한 `WHERE` 절 자체**, 조건이 여러 개일 때 그 안에 `bool`을 넣는 구조 (택일 아님).
- 정확 일치·다건 조회는 `term` / `terms` + `filter` 조합.
  **`text` 필드에 `term`을 쓰면 안 잡힘** → `match` 또는 `.keyword` 사용.
- **중복 방지**: **문서 `_id`** 를 유니크 값(apiKey)으로 지정 + `op_type => 'create'` (중복 시 409).
- **대량 저장은 `bulk`** — 단, 일부 실패해도 200이 오므로 `$response['errors']` 를 반드시 확인.
- **NRT 주의**: 저장/삭제 직후 즉시 반영이 필요하면 `refresh => true`.
- 전체 삭제는 인덱스를 지우지 않고 `_delete_by_query` + `match_all`
  (대량이면 `conflicts=proceed` + `wait_for_completion=false`).
- 페이지네이션은 `from`/`size`, 전체 건수는 `track_total_hits => true`.
  **`from + size ≤ 10,000` 한계**가 있으니 그 이상은 `search_after`.
- **수정**: `index()`는 전체 교체, 일부만 바꾸려면 `update()` + `doc`; 있으면수정/없으면생성은 `doc_as_upsert`.
- **bool 절**: 점수 필요 없으면 `filter`(빠름·캐싱), 가점은 `should`, 제외는 `must_not`.
- **refresh**: 즉시성 필요하면 `true`/`'wait_for'`, 평소엔 `false`(기본)로 성능 확보.
- **정렬은 `no`/`createdAt` 같은 `keyword`·숫자·날짜 필드로** (`text` 불가, **`_id` 정렬은 금지**).
- **존재 확인**: 문서 단위는 `exists()` API, 필드 값 유무는 `exists` 쿼리 (`IS NULL` = `must_not` + `exists`).
- **타입 변경은 불가** → alias + 새 인덱스 + `_reindex` 로 무중단 교체.

**📊 집계**

- **집계는 `size: 0` + `aggs`**, `query` 로 대상을 좁힌 뒤 계산 (`WHERE` + `GROUP BY`).
- **Bucket(`terms`·`date_histogram`)으로 묶고 그 안에 Metric(`avg`·`cardinality`)을 중첩**하는 게 기본형.
- **`text` 필드는 집계 불가** → `.keyword` 로 (`fielddata: true` 는 켜지 말 것).
- **`terms` 는 기본 상위 10개만** → `sum_other_doc_count` 가 0이 아니면 잘린 것. 전량은 `composite`.
- **`cardinality`·`percentiles` 는 근사치**, `terms` 의 `doc_count` 도 샤드 오차 가능(`doc_count_error_upper_bound`).
- **`date_histogram` 에 `time_zone: "+09:00"` 필수** — 없으면 UTC 기준이라 오전 9시 이전이 전날로 밀림.
- `HAVING` 은 `bucket_selector`, 버킷 응답의 날짜는 `key`(epoch) 말고 **`key_as_string`** 을 쓸 것.

**🧭 MySQL 쓰던 사람이 ES 에서 가장 자주 틀리는 5가지**

| # | MySQL 감각 | ES 에서는 |
|---|-----------|----------|
| 1 | `INSERT` 후 바로 `SELECT` 하면 보임 | **NRT — 1초간 안 보임.** 필요하면 `refresh => true` |
| 2 | `LIMIT` 없으면 전부 나옴 | **`size` 없으면 10건만**, `terms` 집계도 **상위 10개만** |
| 3 | `COUNT(*)`·`COUNT(DISTINCT)` 는 정확 | `doc_count`·`cardinality` 는 **근사치일 수 있음** |
| 4 | `ALTER TABLE MODIFY` 로 타입 변경 | **타입 변경 불가** → alias + `_reindex` |
| 5 | `WHERE col = '값'` 이면 다 찾아짐 | **`text` 필드에 `term` 쓰면 0건.** `keyword` / `match` 확인 |
