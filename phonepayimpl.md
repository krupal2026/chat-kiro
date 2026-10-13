# PhonePe PG — Flutter + PHP Implementation Spec

Target: PhonePe Payment Gateway **Standard Checkout, Mobile SDK flow** (`phonepe_payment_sdk` on Flutter + own PHP backend).
Verified against PhonePe developer docs as of 2026-10-13. Links in [References](#references).

---

## 1. Architecture / Flow

```
Flutter                       PHP backend                     PhonePe
  |-- POST /payments/create ----->|
  |                               |-- POST /v1/oauth/token ------>|   (cached token)
  |                               |<-- access_token --------------|
  |                               |-- POST /checkout/v2/sdk/order >|
  |                               |<-- { orderId, token } --------|
  |<-- { orderId, token, mid } ---|
  |
  |-- PhonePePaymentSdk.init(...)
  |-- PhonePePaymentSdk.startTransaction(request, appSchema)
  |        -> PhonePe PayPage / UPI app switch -> returns SUCCESS|FAILURE|INTERRUPTED
  |
  |-- GET /payments/{merchantOrderId}/status --->|   (poll until terminal)
                                                 |-- GET /checkout/v2/order/{id}/status -->|
         <--------------- PhonePe webhook (S2S) --|
```

Rules that are non-negotiable:
- `amount` is computed **server-side** from the product/plan id. Never accept amount from the app.
- SDK `status` is a **UI hint only**. Order is `PAID` only when the webhook or Order Status API says `COMPLETED`.
- `client_secret` / webhook secret never leave the PHP server.

---

## 2. Credentials needed from PhonePe dashboard

| Key | Used in |
|---|---|
| `MERCHANT_ID` | PHP + Flutter (`init`) |
| `CLIENT_ID`, `CLIENT_VERSION`, `CLIENT_SECRET` | PHP only (auth token) |
| Webhook username/password **or** HMAC secret + webhook id | PHP only |

Store in env (`.env` / server env vars), not in code, not in the Flutter app.

## 3. Environments

| Purpose | Sandbox | Production |
|---|---|---|
| Auth token | `POST https://api-preprod.phonepe.com/apis/pg-sandbox/v1/oauth/token` | `POST https://api.phonepe.com/apis/identity-manager/v1/oauth/token` |
| Create SDK order | `POST https://api-preprod.phonepe.com/apis/pg-sandbox/checkout/v2/sdk/order` | `POST https://api.phonepe.com/apis/pg/checkout/v2/sdk/order` |
| Order status | `GET https://api-preprod.phonepe.com/apis/pg-sandbox/checkout/v2/order/{merchantOrderId}/status` | `GET https://api.phonepe.com/apis/pg/checkout/v2/order/{merchantOrderId}/status` |

Note the auth host differs between sandbox and prod (`pg-sandbox` vs `identity-manager`) — a common misconfiguration.

---

## 4. Backend (PHP)

### 4.1 DB table

```sql
CREATE TABLE payments (
  id                BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  merchant_order_id VARCHAR(64)  NOT NULL UNIQUE,   -- we generate this
  phonepe_order_id  VARCHAR(64)  NULL,              -- from create-order response
  user_id           BIGINT UNSIGNED NOT NULL,
  amount_paise      BIGINT       NOT NULL,
  status            ENUM('CREATED','PENDING','COMPLETED','FAILED') NOT NULL DEFAULT 'CREATED',
  transaction_id    VARCHAR(64)  NULL,
  raw_webhook       JSON         NULL,
  created_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at        TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 4.2 Auth token (cache it)

`expires_at` in the response is a unix timestamp. Cache the token and refresh ~60s before expiry; do not call this per payment.

```php
<?php
function phonepeToken(): string {
    $cacheFile = sys_get_temp_dir() . '/phonepe_token.json';   // use Redis in prod
    if (is_file($cacheFile)) {
        $c = json_decode(file_get_contents($cacheFile), true);
        if (!empty($c['access_token']) && $c['expires_at'] > time() + 60) {
            return $c['access_token'];
        }
    }

    $ch = curl_init(getenv('PHONEPE_AUTH_URL'));
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => ['Content-Type: application/x-www-form-urlencoded'],
        CURLOPT_POSTFIELDS     => http_build_query([
            'client_id'      => getenv('PHONEPE_CLIENT_ID'),
            'client_version' => getenv('PHONEPE_CLIENT_VERSION'),
            'client_secret'  => getenv('PHONEPE_CLIENT_SECRET'),
            'grant_type'     => 'client_credentials',
        ]),
        CURLOPT_TIMEOUT        => 15,
    ]);
    $res  = curl_exec($ch);
    $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    if ($code !== 200) throw new RuntimeException("PhonePe auth failed: $code $res");

    $data = json_decode($res, true);
    file_put_contents($cacheFile, json_encode($data), LOCK_EX);
    return $data['access_token'];      // token_type is "O-Bearer"
}
```

### 4.3 `POST /payments/create` (app-authenticated)

Request from app: `{ "plan_id": "PRO_MONTHLY" }` — **not** amount.

```php
<?php
$userId = requireAuthUser();                       // your existing auth
$in     = json_decode(file_get_contents('php://input'), true) ?: [];
$planId = $in['plan_id'] ?? '';
$amount = priceInPaiseForPlan($planId);            // server-side price lookup, throws on unknown plan
$merchantOrderId = 'ORD' . date('YmdHis') . bin2hex(random_bytes(4));

insertPayment($merchantOrderId, $userId, $amount); // status CREATED

$body = json_encode([
    'merchantOrderId' => $merchantOrderId,
    'amount'          => $amount,        // PAISE. 100 = Rs 1
    'expireAfter'     => 1200,           // seconds, optional
    'metaInfo'        => ['udf1' => (string)$userId, 'udf2' => $planId],
    'paymentFlow'     => ['type' => 'PG_CHECKOUT'],
]);

$ch = curl_init(getenv('PHONEPE_SDK_ORDER_URL'));
curl_setopt_array($ch, [
    CURLOPT_POST           => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => [
        'Content-Type: application/json',
        'Authorization: O-Bearer ' . phonepeToken(),
    ],
    CURLOPT_POSTFIELDS     => $body,
    CURLOPT_TIMEOUT        => 20,
]);
$res = curl_exec($ch);
curl_close($ch);

$d = json_decode($res, true);   // { orderId, state, expireAt, token }

updatePhonepeOrderId($merchantOrderId, $d['orderId']);

header('Content-Type: application/json');
echo json_encode([
    'merchantOrderId' => $merchantOrderId,     // our id, used for status polling
    'orderId'         => $d['orderId'],        // -> SDK
    'token'           => $d['token'],          // -> SDK
    'merchantId'      => getenv('PHONEPE_MERCHANT_ID'),
    'environment'     => getenv('PHONEPE_ENV'), // SANDBOX | PRODUCTION
]);
```

To restrict payment modes, add `paymentFlow.paymentModeConfig.enabledPaymentModes` with entries like `{"type":"UPI_INTENT"}`, `UPI_COLLECT`, `UPI_QR`, `NET_BANKING`, `CARD`. Omit to show all.

### 4.4 `GET /payments/{merchantOrderId}/status` (app-authenticated)

Return local status; if not terminal, hit PhonePe and update. Enforce that the row belongs to the calling user.

```php
<?php
$row = getPaymentForUser($merchantOrderId, requireAuthUser());
if (!$row) { http_response_code(404); exit; }

if (in_array($row['status'], ['COMPLETED', 'FAILED'], true)) {
    echo json_encode(['status' => $row['status']]); exit;
}

$url = rtrim(getenv('PHONEPE_BASE_URL'), '/')
     . "/checkout/v2/order/{$merchantOrderId}/status?details=false&errorContext=true";

$ch = curl_init($url);
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => [
        'Content-Type: application/json',
        'Authorization: O-Bearer ' . phonepeToken(),
    ],
    CURLOPT_TIMEOUT        => 15,
]);
$d = json_decode(curl_exec($ch), true);
curl_close($ch);

// state: COMPLETED | FAILED | PENDING
applyTerminalStatus($merchantOrderId, $d['state'], $d);   // idempotent, see 4.6
echo json_encode(['status' => $d['state']]);
```

### 4.5 `POST /webhooks/phonepe` (public, no app auth — verify signature)

Configure in Business Dashboard → Developer Settings → Webhook. Two auth types; pick one and implement only that.

**Option A — SHA (username/password).** PhonePe sends `Authorization: SHA256(username:password)`.

```php
<?php
$raw      = file_get_contents('php://input');
$received = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
$expected = hash('sha256', getenv('PHONEPE_WH_USER') . ':' . getenv('PHONEPE_WH_PASS'));

if (!hash_equals($expected, trim($received))) { http_response_code(401); exit; }
```

**Option B — HMAC.** Headers `x-phonepe-checksum-key-id` (identifies which secret) and `x-phonepe-checksum-signature` (HMAC of the raw body with that secret). Verify over the **raw** body, never over a re-encoded array.

```php
<?php
$raw   = file_get_contents('php://input');
$keyId = $_SERVER['HTTP_X_PHONEPE_CHECKSUM_KEY_ID'] ?? '';
$sig   = $_SERVER['HTTP_X_PHONEPE_CHECKSUM_SIGNATURE'] ?? '';
$secret = secretForKeyId($keyId);                   // reject unknown keyId
if (!$secret || !hash_equals(hash_hmac('sha256', $raw, $secret), $sig)) {
    http_response_code(401); exit;
}
```

Then process:

```php
<?php
$e = json_decode($raw, true);
// event e.g. checkout.order.completed / checkout.order.failed
$p = $e['payload'];
applyTerminalStatus($p['merchantOrderId'], $p['state'], $e);
http_response_code(200);
echo 'OK';
```

Confirm the exact HMAC string-to-sign with your PhonePe integration contact before go-live; the docs describe it as a checksum over the plain-text payload but do not publish a worked example.

### 4.6 `applyTerminalStatus` — idempotency

Webhook and polling will both fire. Make the write idempotent and guard against amount tampering:

```php
<?php
function applyTerminalStatus(string $moid, string $state, array $raw): void {
    // single UPDATE guarded by current status; fulfilment only on the first transition
    $stmt = db()->prepare(
        "UPDATE payments SET status = ?, raw_webhook = ?
         WHERE merchant_order_id = ? AND status IN ('CREATED','PENDING')"
    );
    $stmt->execute([$state, json_encode($raw), $moid]);

    if ($stmt->rowCount() === 1 && $state === 'COMPLETED') {
        // verify amount matches what we created, then fulfil (once)
        grantEntitlementOnce($moid);
    }
}
```

### 4.7 Reconciliation cron

Every 5 min: pick `payments` rows in `CREATED`/`PENDING` older than 2 min, call Order Status, apply result. Mark `FAILED` after the order's expiry window. This covers users who killed the app mid-UPI.

---

## 5. Flutter

### 5.1 Dependency

```yaml
dependencies:
  phonepe_payment_sdk: ^3.0.2   # publisher phonepe.com
```
Minimum Flutter 2.17.0.

### 5.2 Android config

Project-level `android/build.gradle` (or `settings.gradle` `dependencyResolutionManagement` on newer templates) — add the PhonePe maven repo:

```gradle
allprojects {
  repositories {
    google()
    mavenCentral()
    maven { url = uri("https://phonepe.mycloudrepo.io/public/repositories/phonepe-intentsdk-android") }
  }
}
```

`MainActivity` **must** extend `FlutterFragmentActivity` (not `FlutterActivity`) — the SDK uses fragments:

```kotlin
import io.flutter.embedding.android.FlutterFragmentActivity
class MainActivity : FlutterFragmentActivity()
```

`minSdkVersion 21`. The PhonePe page still lists `compileSdk/targetSdk 28` — that is stale and will fail Play Store policy; keep your project's current `compileSdk`/`targetSdk` (34+) and only honour the `minSdk 21` floor.

For UPI app discovery on Android 11+ (API 30), add to `AndroidManifest.xml` above `<application>`:

```xml
<queries>
  <intent><action android:name="android.intent.action.VIEW" />
    <data android:scheme="upi" /></intent>
  <package android:name="com.phonepe.app" />
</queries>
```

### 5.3 iOS config

`ios/Runner/Info.plist`:

```xml
<key>LSApplicationQueriesSchemes</key>
<array>
  <string>ppemerchantsdkv1</string>
  <string>ppemerchantsdkv2</string>
  <string>ppemerchantsdkv3</string>
  <string>paytmmp</string>
  <string>gpay</string>
</array>
```

Add a URL Type (CFBundleURLSchemes) for your app, e.g. `myappscheme` — letters/digits only, must start with a letter, `.` and `-` allowed. This exact string is passed as `appSchema` to `startTransaction`; a mismatch means the user never returns to your app.

`AppDelegate` must broadcast the return URL:

```objc
- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url
            options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options {
  NSMutableDictionary *userInfo = [NSMutableDictionary new];
  userInfo[@"options"] = options;
  userInfo[@"openUrl"] = url;
  [[NSNotificationCenter defaultCenter] postNotificationName:@"ApplicationOpenURLNotification"
                                                     object:nil userInfo:userInfo];
  return YES;
}
```

Then `cd ios && pod install`.

### 5.4 Service class

```dart
import 'dart:convert';
import 'package:phonepe_payment_sdk/phonepe_payment_sdk.dart';

class PaymentResult {
  final String status;     // COMPLETED | FAILED | PENDING
  final String? message;
  PaymentResult(this.status, [this.message]);
}

class PhonePeService {
  static const _appSchema = 'myappscheme';   // iOS only; must match Info.plist
  bool _inited = false;

  Future<void> _ensureInit(String env, String merchantId, String userId) async {
    if (_inited) return;
    _inited = await PhonePePaymentSdk.init(
      env,                 // 'SANDBOX' or 'PRODUCTION'
      merchantId,
      userId,              // flowId: alphanumeric, used by PhonePe for debugging
      false,               // enableLogging: false in release
    );
  }

  Future<PaymentResult> pay({required String planId}) async {
    // 1. create order on our server
    final order = await api.post('/payments/create', {'plan_id': planId});

    await _ensureInit(order['environment'], order['merchantId'], currentUserId);

    // 2. hand off to SDK
    final request = jsonEncode({
      'orderId': order['orderId'],
      'merchantId': order['merchantId'],
      'token': order['token'],
      'paymentMode': {'type': 'PAY_PAGE'},
    });

    Map<dynamic, dynamic>? res;
    try {
      res = await PhonePePaymentSdk.startTransaction(request, _appSchema);
    } catch (_) {
      // fall through to status check — payment may still have gone through
    }
    // res['status'] is SUCCESS | FAILURE | INTERRUPTED  -> hint only

    // 3. server is the source of truth
    return _confirm(order['merchantOrderId']);
  }

  /// Poll our backend until terminal. Backend falls back to PhonePe Order Status.
  Future<PaymentResult> _confirm(String merchantOrderId) async {
    for (var i = 0; i < 20; i++) {                 // ~60s
      final r = await api.get('/payments/$merchantOrderId/status');
      if (r['status'] == 'COMPLETED') return PaymentResult('COMPLETED');
      if (r['status'] == 'FAILED')    return PaymentResult('FAILED');
      await Future.delayed(const Duration(seconds: 3));
    }
    return PaymentResult('PENDING', 'Still processing. We will update you shortly.');
  }
}
```

### 5.5 UI requirements

- Non-dismissible "Verifying payment…" state during `_confirm`. Do not let the user retry here — that creates double charges.
- `PENDING` after timeout is a legitimate outcome: show "payment processing", do not show failure, do not grant access. Re-check on next app resume / via push.
- Disable the pay button from tap until `pay()` returns.
- On app resume, re-run `_confirm` for any locally stored in-flight `merchantOrderId`.

---

## 6. What changes vs the old `razorpay_flutter` integration

| Concern | Razorpay plugin (before) | PhonePe SDK (now) |
|---|---|---|
| Order creation | `order_id` from Razorpay Orders API | `orderId` + `token` from `checkout/v2/sdk/order` |
| Hand-off | `Razorpay.open(options)` | `PhonePePaymentSdk.init` then `startTransaction(json, appSchema)` |
| Client-side proof | `razorpay_signature` verified server-side | **none** — no client signature exists |
| Confirmation | signature verify + webhook | **webhook + Order Status API only** |
| iOS return path | handled by plugin | needs your own URL scheme + AppDelegate hook |
| Android base activity | `FlutterActivity` ok | must be `FlutterFragmentActivity` |

The important behavioural change: there is no payment signature to verify from the app. Any code path that previously trusted the client callback must be moved to webhook/status-API driven fulfilment.

---

## 7. Test checklist (sandbox, then one live low-value txn)

- [ ] UPI intent → PhonePe app → success → returns to app → `COMPLETED`
- [ ] UPI intent → press back inside PhonePe → `INTERRUPTED` from SDK, backend resolves `FAILED` or `PENDING`
- [ ] Kill the app while on the PayPage → relaunch → status resolves via cron/webhook
- [ ] Airplane mode right after payment → `_confirm` times out → `PENDING` UI, not a false failure
- [ ] Webhook delivered twice → entitlement granted exactly once
- [ ] Webhook with a tampered/absent signature → 401, no DB write
- [ ] `amount` modified in the app's create-payment request → ignored, server price used
- [ ] Expired auth token → auto-refresh, no user-visible failure
- [ ] iOS: wrong `appSchema` → verify you see the failure mode, then fix (common silent bug)
- [ ] Order status for an unknown `merchantOrderId` and for another user's order → 404, not data leak

---

## References

- Flutter SDK setup — https://developer.phonepe.com/payment-gateway/mobile-app-integration/standard-checkout-mobile/flutter/sdk-setup
- Authorization token — https://developer.phonepe.com/payment-gateway/mobile-app-integration/standard-checkout-mobile/api-reference/authorization
- Create Order Token — https://developer.phonepe.com/payment-gateway/mobile-app-integration/standard-checkout-mobile/api-reference/create-order-token
- Check Order Status — https://developer.phonepe.com/payment-gateway/mobile-app-integration/standard-checkout-mobile/api-reference/check-order-status
- Webhook handling — https://developer.phonepe.com/payment-gateway/mobile-app-integration/standard-checkout-mobile/webhook-handling
- Refunds — https://developer.phonepe.com/payment-gateway/mobile-app-integration/standard-checkout-mobile/api-reference/refund
- PHP Backend SDK (optional alternative to raw cURL) — https://developer.phonepe.com/payment-gateway/backend-sdk/php-be-sdk/introduction
- Android SDK setup (gradle/manifest details) — https://developer.phonepe.com/payment-gateway/mobile-app-integration/standard-checkout-mobile/android-sdk/sdk-setup
- UAT checklist / go-live — https://developer.phonepe.com/payment-gateway/uat-testing-go-live/uat-checklist
- Official Flutter sample app — https://github.com/PhonePe/phonepe-pg-sdk-flutter
- `phonepe_payment_sdk` on pub.dev — https://pub.dev/packages/phonepe_payment_sdk
