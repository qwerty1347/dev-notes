# Elasticsearch (auth_apikey) 사용 정리

> `ES.txt`(REST DSL 쿼리)와 `ES 리스폰스.txt`(PHP 클라이언트 코드·응답)를 작업별로 합쳐 정리한 문서.
> 예시 인덱스: `api_keys` / `auth_apikey_dev`
>
> cf) Elasticsearch에서 작성하는 JSON 형태의 쿼리는 보통 Query DSL 이라고 부릅니다 (Domain Specific Language)

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

> 💡 **판단 기준**: "이 필드로 **문장 검색**을 하나?" → `text` / "이 값이 **맞는지 대조**만 하나?" → `keyword`
> `apiKey`, `ips` 는 대조용이므로 `keyword`.

**둘 다 필요하면 멀티 필드**로 (검색은 `title`, 정렬·집계는 `title.keyword`)
```json
"title": {
  "type": "text",
  "fields": { "keyword": { "type": "keyword" } }
}
```

#### ② 숫자 타입

| 타입 | 범위 / 특징 |
|------|-------------|
| `integer` | -21억 ~ 21억 (약 ±2^31). 일반적인 순번·ID |
| `long` | 매우 큰 정수 (±2^63). 타임스탬프(ms), 대용량 ID |
| `short` / `byte` | -32768~32767 / -128~127. 작은 코드값 (저장 절약) |
| `float` / `double` | 실수. 정밀도 double > float |
| `scaled_float` | 실수를 정수로 저장해 성능↑ (`scaling_factor` 필요). **금액**에 적합 |

> ⚠️ **숫자처럼 보여도 계산·범위검색을 안 하면 `keyword`가 나음** (전화번호, 우편번호, 사번 등).
> `keyword`가 정확 일치 검색이 더 빠름.

#### ③ 그 외 자주 쓰는 타입

| 타입 | 예시 데이터 | 특징 / 용도 |
|------|-------------|-------------|
| `boolean` | `true` / `"true"` | `true` / `false` (문자열 `"true"`도 허용) |
| `date` | `"2026-07-22 14:30:00"`, `1753160400000` | 날짜·시간. `format` 지정 가능, 범위 검색(`range`)·정렬에 사용 |
| `ip` | `"192.168.0.1"`, `"::1"` | IPv4/IPv6 전용. **CIDR 대역 검색 가능** (`192.168.0.0/24`) |
| `object` | `{"user": {"name": "홍길동", "age": 30}}` | 중첩 JSON. 내부적으로 `user.name` 처럼 평탄화됨 |
| `nested` | `[{"name":"A","qty":1}, {"name":"B","qty":2}]` | 객체 **배열**의 각 원소를 독립 문서로 취급 (원소별 조건 조합이 정확) |
| `geo_point` | `{"lat": 37.5665, "lon": 126.9780}` | 위/경도. 거리·반경 검색 |
| `binary` | `"U29tZSBiaW5hcnkgZGF0YQ=="` | Base64 저장. 기본적으로 검색 불가 |

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

#### ⑤ 매핑 관련 주의사항

- **한 번 만든 필드의 타입은 변경 불가.** 바꾸려면 새 인덱스를 만들고 `_reindex` 해야 함.
- 매핑을 안 주면 **동적 매핑(dynamic mapping)** 으로 자동 추론됨
  → 문자열이 `text` + `keyword` 멀티필드로 잡혀 의도와 달라질 수 있으니, **중요 인덱스는 매핑을 직접 정의**할 것.
- 필드 **추가**는 가능 (기존 필드 수정만 불가).
- 검색에 쓰지 않는 필드는 `"index": false` 로 색인을 꺼서 용량·성능 절약 가능.

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

**쿼리 문법 ↔ SQL 대응**

| ES | SQL 대응 | 설명 |
|----|----------|------|
| `api_keys/_search` | `SELECT * FROM api_keys` | 인덱스에서 검색 |
| `query` | `WHERE` | 검색 조건 |
| `bool` | `WHERE A AND B` | 여러 조건 조합 |
| `must` | `A AND B AND C` | 모든 조건 만족 (검색 점수 계산 O) |
| `term` | `WHERE no = 1` | 하나의 정확한 값 비교 |
| `terms` | `WHERE no IN (1,2,3)` | 찾을 값이 여러 개일 때 (IN) |
| `filter` | — | 정확 일치만 필요할 때 (점수 계산 X, 캐싱 유리) |


> bool 쿼리 안에는 의미 있는 절 최소 하나 있어야함 (must, should, filter, must_not)
* must: AND
* filter: WHERE (AND)
* should: OR (다른필드의 OR, 같은 필드의 OR = temrs)
* must_not: NOT

> - apiKey·ips처럼 **정확히 일치하는 값만 조회**할 땐 `filter` 사용.
> - 조건이 1개면 `bool > filter` 생략 가능.
> - **결과도 필요** → `search()`의 `hits.total` 사용 (추천)
> - **개수만 필요** → `count()` API 사용

### ⭐ term vs terms 차이 (자주 헷갈림)

핵심: **"내가 찾는 값(검색어)이 1개냐, 여러 개냐"** 의 차이. (필드가 배열이냐 아니냐가 아님!)

| 구분 | 찾는 값 | SQL | 의미 |
|------|---------|-----|------|
| `term` | **1개** | `WHERE apiKey = 'abc123'` | 이 값과 정확히 일치하는 문서 |
| `terms` | **여러 개(배열)** | `WHERE id IN (1, 2, 3)` | 이 값들 **중 하나라도** 일치하는 문서 |

**term — 값 하나**
```json
{ "term": { "apiKey": "abc123" } }
// apiKey 가 정확히 "abc123" 인 문서
```

**terms — 값 여러 개 (OR / IN)**
```json
{ "terms": { "id": [1, 2, 3] } }
// id 가 1 이거나 2 이거나 3 인 문서  →  WHERE id IN (1,2,3)
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

- 이 IP **하나**가 등록돼 있는지 확인 → `term`
- 이 IP **목록 중 하나라도** 등록돼 있는지 확인 → `terms`

**PHP 예시**
```php
// term : IP 1개 확인
['term'  => ['ips' => '127.0.0.1']]

// terms : IP 여러 개 중 하나라도 확인
['terms' => ['ips' => ['10.0.0.1', '127.0.0.1']]]
```

#### 실전 예시 — API Key 인증 (요청 토큰 + 요청 IP 검증)

**"이 apiKey가 존재하고, 등록된 ips 목록 안에 요청 IP가 들어있는 문서"** 를 찾는 인증용 쿼리.

```php
$response = $this->client->search([
    'index' => $this->authApiKeyIndex,
    'body'  => [
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

**예외 처리**
```php
use Elastic\Elasticsearch\Exception\ClientResponseException;

catch (ClientResponseException $e) {
    // "이미 저장된 apiKey입니다"
}
```

---

## 4. 카운트 (COUNT)

**REST DSL**
```
GET /auth_apikey_dev/_count
```

**응답**
```php
[
  "count" => 1,
  "_shards" => [ "total" => 1, "successful" => 1, "skipped" => 0, "failed" => 0 ]
]
```

> **존재 여부만 판단**할 거면 `count`보다 `search` 사용.

---

## 5. 삭제 (DELETE)

**REST DSL (조건 삭제)**
```json
POST /api_keys/_delete_by_query
{
  "query": { "term": { "apiKey": "abc123" } }
}
```

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

### cf2) 삭제 직후 조회하면 아직 나오는 함정 — NRT
> ES는 **Near Real-Time(NRT)** 이라 저장/삭제해도 즉시 검색에 반영하지 않음.
> 성능을 위해 변경을 모아뒀다가 **기본 1초마다 `refresh`** 할 때 검색 인덱스에 반영함.
>
> → 삭제 요청에 `'refresh' => true` 를 주면 **삭제를 검색에 즉시 반영한 뒤** 응답함.

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

| 파라미터 | 설명 |
|----------|------|
| `from` | 시작 위치 `(pg - 1) * sz` |
| `size` | 가져올 개수 |
| `track_total_hits` | `true`면 전체 건수를 정확히 반환 (기본은 10,000에서 멈춤) |

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

> `result` 값으로 결과 구분: `created`(신규) / `updated`(수정) / `noop`(변경 없음).

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

| 절 | 의미 | SQL 대응 | 점수(score) |
|----|------|----------|-------------|
| `must` | 반드시 만족 (AND) | `AND` | O |
| `filter` | 반드시 만족 (AND) | `AND` | X (캐싱 유리, 빠름) |
| `should` | 만족하면 가점 (다른필드의 OR) | `OR` | O |
| `must_not` | 만족하면 제외 | `NOT / != ` | X |

> - **점수가 필요 없는 정확 일치 조건은 `filter`** 를 쓰는 게 성능상 유리 (결과 캐싱).
> - `should`만 단독으로 쓰면 그중 최소 1개는 만족해야 함(`minimum_should_match` 조절 가능).


ex)
예를 들어 사용자가 "아이폰" 을 검색했다고 해보겠습니다.

상품이 다음과 같다면

| 상품명 | status | price |
|--------| -------| ------|
| 아이폰 16 Pr | OPEN | 1,500,000 |
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

기준을 한 줄로 정리하면

| 목적 | 사용할 절 | 대표 예시 |
|------|-----------|-----------|
| 조건을 **만족하는지만** 확인 | `filter` | `status = OPEN`, `category = "전자제품"`, `price >= 100000`, `no = 1234` |
| 결과의 **순위(관련도)** 를 결정 | `must` | `match`, `multi_match`, `query_string`, `match_phrase` |

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

> `text` 필드는 기본적으로 정렬 불가 → 정렬이 필요하면 `keyword`(또는 숫자/날짜) 필드로 정렬.

---

## 11. 존재 여부 확인 (EXISTS)

```php
// _id 로 문서 존재 여부만 빠르게 확인
$exists = $this->client->exists([
    'index' => 'auth_apikey_dev',
    'id'    => $id
])->asBool();   // true / false
```

> 조건(apiKey 등)으로 존재만 확인할 땐 `search` + `"size": 0` + `terminate_after: 1` 로 가볍게 조회하거나 `count` 사용.

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

---

## 13. refresh 옵션 3종 정리

저장/수정/삭제 요청 시 `refresh` 값에 따라 검색 반영 시점이 달라짐.

| 값 | 동작 | 사용 상황 |
|----|------|-----------|
| `false` (기본) | 다음 주기(기본 1초)에 반영 | 일반적인 대량 처리 (성능 ↑) |
| `true` | **즉시** 반영 후 응답 | 저장/삭제 직후 바로 조회해야 할 때 |
| `'wait_for'` | 다음 refresh까지 **기다렸다가** 응답 | 즉시성 필요 + `true`의 부하는 피하고 싶을 때 |

> `refresh => true`는 매번 강제 refresh라 **자주 쓰면 성능 저하**. 실시간성이 꼭 필요한 지점에만 사용.

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

| 상태 코드 | 의미 | 대표 상황 |
|-----------|------|-----------|
| 400 | 잘못된 요청 | 매핑/쿼리 문법 오류 |
| 404 | 없음 | 없는 `_id` 조회·삭제 |
| 409 | 충돌 | `op_type => create` 중복 저장 |

---

## 핵심 요약

- **완전 일치 검색이 필요한 필드는 `keyword`** 로 매핑 (`text` ✕).
- 정확 일치·다건 조회는 `term` / `terms` + `filter` 조합.
- **중복 방지**: `id`를 유니크 값으로 지정 + `op_type => 'create'` (중복 시 409).
- **NRT 주의**: 저장/삭제 직후 즉시 반영이 필요하면 `refresh => true`.
- 전체 삭제는 인덱스를 지우지 않고 `_delete_by_query` + `match_all`.
- 페이지네이션은 `from`/`size`, 전체 건수는 `track_total_hits => true`.
- **수정**: `index()`는 전체 교체, 일부만 바꾸려면 `update()` + `doc`; 있으면수정/없으면생성은 `doc_as_upsert`.
- **bool 절**: 점수 필요 없으면 `filter`(빠름·캐싱), 가점은 `should`, 제외는 `must_not`.
- **refresh**: 즉시성 필요하면 `true`/`'wait_for'`, 평소엔 `false`(기본)로 성능 확보.
- **정렬은 `keyword`/숫자/날짜** 필드로 (text 불가), 존재 확인은 `exists()`.
