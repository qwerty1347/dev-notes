---
tags:
  - captcha
  - turnstile
  - hcaptcha
  - cloudflare
  - security
  - php
created: 2026-04-17
---

# Cloudflare Turnstile와 hCaptcha 통합

> 로그인·회원가입 등 폼 제출 지점에 봇 차단을 적용하는 두 대표적 CAPTCHA 서비스(Cloudflare **Turnstile**, **hCaptcha**) 의 발급 → 프론트 삽입 → 서버 검증 전체 흐름을 정리.

---

## 1. 두 서비스 한눈에

| 항목 | Cloudflare Turnstile | hCaptcha |
| --- | --- | --- |
| 사용자 경험 | **비보이는(invisible)** / 클릭 1회 / 자동 통과 다수 | 이미지 선택 또는 체크박스 |
| 무료 한도 | 사실상 무제한 (Cloudflare 계정만 있으면) | 월 1M req 무료 티어 |
| 프라이버시 | 쿠키·지문 수집 최소화 (Cloudflare 주장) | GDPR 친화 마케팅 |
| 엔드포인트 | `challenges.cloudflare.com/turnstile/v0/siteverify` | `api.hcaptcha.com/siteverify` |
| 프론트 위젯 클래스 | `cf-turnstile` | `h-captcha` |
| 응답 파라미터명 | `cf-turnstile-response` | `h-captcha-response` |

둘 다 **Site Key** (프론트 노출) + **Secret Key** (서버 전용) 쌍으로 동작하며, 서버가 토큰을 받아 각자의 siteverify 엔드포인트로 재검증하는 구조는 동일하다.

---

## 2. Key 발급 절차

### Cloudflare Turnstile

<https://dash.cloudflare.com/login>

1. 회원가입 및 로그인
2. **Protect & Connect → Turnstile** 메뉴 진입
3. **Add widget** 클릭
4. **Hostnames** 등록 (예: `example.com`, 로컬 테스트면 `localhost` 도 추가)
5. **Site Key** / **Secret Key** 발급

### hCaptcha

<https://dashboard.hcaptcha.com/login>

1. 회원가입 및 로그인
2. 대시보드에서 사이트 추가 → **Site Key** / **Secret Key** 발급

### `.env` 저장

```dotenv
TURNSTILE_SITE_KEY=0x4AAAAAAA...
TURNSTILE_SECRET_KEY=0x4AAAAAAA...
HCAPTCHA_SITE_KEY=xxxxxxxx-xxxx-xxxx
HCAPTCHA_SECRET_KEY=0x00000000000000000
```

> **Secret Key 는 절대 프론트에 노출 금지.** 프론트에는 Site Key만 내려보낸다.

---

## 3. 프론트엔드 적용

### 3-1. Turnstile

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Login</title>
  <!-- Turnstile 스크립트 로드 -->
  <script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
</head>
<body>
  <form method="POST" action="/captcha">
    @csrf
    <input type="text"     name="name">
    <input type="password" name="password">

    <!-- 위젯 삽입 지점 -->
    <div class="cf-turnstile"
         data-sitekey="<?= htmlspecialchars(env('TURNSTILE_SITE_KEY')) ?>">
    </div>

    <!-- 서버에서 분기 판단용 플래그 -->
    <input type="hidden" name="captcha" value="turnstile">
    <input type="submit" value="로그인">
  </form>
</body>
</html>
```

- `class="cf-turnstile"` 엘리먼트 자리에 위젯이 자동 렌더링됨
- 검증 성공 시 폼 제출 페이로드에 **`cf-turnstile-response`** 히든 필드가 자동 추가됨

### 3-2. hCaptcha

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Login</title>
  <!-- hCaptcha 스크립트 로드 -->
  <script src="https://js.hcaptcha.com/1/api.js" async defer></script>
</head>
<body>
  <form method="POST" action="/captcha">
    @csrf
    <input type="text"     name="name">
    <input type="password" name="password">

    <!-- 위젯 삽입 지점 -->
    <div class="h-captcha"
         data-sitekey="<?= htmlspecialchars(env('HCAPTCHA_SITE_KEY')) ?>">
    </div>

    <input type="hidden" name="captcha" value="hcaptcha">
    <input type="submit" value="로그인">
  </form>
</body>
</html>
```

- `class="h-captcha"` 자리에 위젯 렌더링
- 성공 시 **`h-captcha-response`** 필드가 자동 첨부됨

> 두 위젯 모두 `data-sitekey` 만 서버에서 내려주고, 사용자 응답 토큰은 JS가 숨은 필드에 자동 주입한다.

---

## 4. 서버 검증 (PHP)

프론트에서 전달된 **사용자 토큰 + 서버 보관 Secret Key** 를 각 공급자 엔드포인트로 POST 하면, `success: true/false` 가 담긴 JSON이 돌아온다.

### 4-1. 분기 후 검증

```php
public function verify(Request $request)
{
    // 1) 프론트에서 보낸 플래그로 어느 서비스인지 판단
    if ($request->post('captcha') === 'turnstile') {
        $token  = $request->post('cf-turnstile-response');
        $secret = env('TURNSTILE_SECRET_KEY');
        $url    = 'https://challenges.cloudflare.com/turnstile/v0/siteverify';
    } else {
        $token  = $request->post('h-captcha-response');
        $secret = env('HCAPTCHA_SECRET_KEY');
        $url    = 'https://api.hcaptcha.com/siteverify';
    }

    // 2) siteverify 엔드포인트로 전달할 페이로드
    $data = [
        'secret'   => $secret,
        'response' => $token,
        'remoteip' => $_SERVER['REMOTE_ADDR'],   // 선택값, 신뢰성 향상
    ];

    // 3) cURL POST
    $ch = curl_init($url);
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_POSTFIELDS     => http_build_query($data),
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 10,
    ]);
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    // 4) 결과 파싱 및 판정
    $result = json_decode($response, true);

    if ($httpCode !== 200 || empty($result['success'])) {
        return back()->withErrors(['captcha' => '캡챠 검증 실패']);
    }

    // 검증 통과 → 실제 로그인 처리
    // ...
}
```


### 4-2. siteverify 응답 예시

```json
{
  "success": true,
  "challenge_ts": "2026-04-17T10:20:30Z",
  "hostname": "example.com",
  "error-codes": []
}
```

실패 시 `error-codes` 에 아래와 같은 값이 들어온다:

| 코드 | 의미 |
| --- | --- |
| `missing-input-secret` | Secret Key 미전송 |
| `invalid-input-secret` | 잘못된 Secret Key |
| `missing-input-response` | 사용자 토큰 없음 |
| `invalid-input-response` | 토큰이 유효하지 않거나 **이미 사용됨 (1회성)** |
| `timeout-or-duplicate` | 토큰 만료 (≈ 5분) 또는 재사용 |

### 4-3. Laravel HTTP Client 를 쓴다면 (선택)

cURL 대신 Laravel 기본 HTTP 클라이언트가 가독성이 훨씬 좋다.

```php
use Illuminate\Support\Facades\Http;

$result = Http::asForm()
    ->timeout(10)
    ->post($url, [
        'secret'   => $secret,
        'response' => $token,
        'remoteip' => $request->ip(),
    ])
    ->json();

if (empty($result['success'])) {
    return back()->withErrors(['captcha' => '캡챠 검증 실패']);
}
```

---

## 5. 체크리스트 / 주의사항

### ✅ 해야 할 것
- **Secret Key 는 서버 전용** — `.env` 로 관리, `.gitignore` 에 포함
- **응답 토큰은 1회성** — 한 번 siteverify 에 쓴 토큰은 재사용 불가 (약 5분 내 소비)
- 서버 쪽에서 `hostname` 필드가 내 도메인과 일치하는지 **추가 확인** 권장
- 네트워크 장애로 siteverify가 실패할 수 있으므로 **타임아웃·에러 처리** 필수
- 위젯 렌더 실패 대비 폼 제출 전에 토큰 존재 여부를 JS 로 1차 체크

### ⚠️ 하지 말 것
- Site Key / Secret Key 를 바꿔치지 말 것 (프론트에 Secret 노출 금지)
- `captcha` 플래그를 **클라이언트 값만** 믿지 말 것 — 가능하면 서버에서 라우트별 고정 사용을 권장
- 하나의 토큰을 여러 요청에서 재사용 금지 (siteverify는 첫 호출만 성공)

---

## 6. 두 서비스 중 어떤 걸 쓸까

| 상황 | 추천 |
| --- | --- |
| UX 최우선, Cloudflare 생태계 이미 사용 중 | **Turnstile** |
| 이미지 선택형 친숙도가 중요하거나 GDPR 요구사항이 강한 조직 | **hCaptcha** |
| 공급자 장애 대비 이중화가 필요 | **둘 다 통합 후 플래그로 분기** (이 노트 예시 구조) |

---

## 관련 노트

- [[FastAPI 파일 업로드와 Form 처리]] — 다른 프레임워크에서 폼 제출 처리 참고
- [[Docker Named Volume으로 venv 격리]]

## 참고

- Cloudflare Turnstile 문서: <https://developers.cloudflare.com/turnstile/>
- hCaptcha 문서: <https://docs.hcaptcha.com/>
