# VRF Exchange API — документация для разработчиков

REST API поверх Telegram-биржи токена **#VRF**. Позволяет сторонним
сайтам и приложениям читать рыночные данные и — по личному ключу
пользователя — управлять его балансом, сделками и стейкингом.

Базовый URL (после деплоя на Railway с включённым Public Networking):

```
https://<your-app>.up.railway.app
```

Локально: `http://localhost:8080` (или значение `$PORT`).

Все ответы — JSON. Успешные ответы содержат `"ok": true`, ошибки —
`"ok": false` и `"error": "..."`.

---

## Аутентификация

Приватные эндпоинты требуют персональный API-ключ пользователя.
Ключ создаётся самим пользователем в Telegram-боте:

```
/apikey [название]     — создать ключ (показывается один раз!)
/apikeys                — список ключей + отзыв
```

Ключ передаётся в заголовке:

```
Authorization: Bearer vrf_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

или

```
X-API-Key: vrf_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Ключ даёт доступ **только к аккаунту своего владельца** — как
персональный токен, а не мастер-ключ ко всей бирже. Разработчик не
может действовать от имени чужого пользователя.

⚠️ Никогда не встраивайте API-ключ пользователя в публичный фронтенд-код.
Ключ должен передаваться через ваш backend, либо пользователь сам
вводит его в вашем приложении (аналогично personal access token).

---

## Rate limiting

120 запросов / 60 секунд на IP (публичные эндпоинты) и дополнительно
на владельца ключа для торговых операций. При превышении — `429`.

---

## CORS

Все ответы содержат `Access-Control-Allow-Origin: *` (настраивается
через `API_CORS_ORIGIN`), так что API можно дёргать прямо с фронтенда.

---

## Публичные эндпоинты (без ключа)

### `GET /api/v1/price`
Текущая цена VRF/USD.

```json
{
  "ok": true,
  "pair": "VRF/USD",
  "price": 1.0342,
  "prev_price": 1.0301,
  "change_24h_pct": 3.42,
  "high_24h": 1.051,
  "low_24h": 0.981,
  "fee_pct": 0.01,
  "ts": "2026-07-02T18:04:11"
}
```

### `GET /api/v1/price/history?hours=24`
История цены (по умолчанию 24ч, максимум 720ч / 30 дней).

```json
{ "ok": true, "hours": 24, "points": [{"ts": "...", "price": 1.021}, ...] }
```

### `GET /api/v1/stats`
Общая статистика биржи.

```json
{
  "ok": true,
  "users": 512,
  "usd_in_circulation": 480213.5,
  "vrf_in_circulation": 91234.2,
  "active_stakes": 87,
  "vrf_staked": 34210.0,
  "total_trades": 2140,
  "price": 1.0342
}
```

### `GET /api/v1/tiers`
Тарифы стейкинга.

```json
{
  "ok": true,
  "tiers": [
    {"key": "flex", "label": "🔓 Гибкий", "apr": 0.08, "lock_days": 0, "early_penalty": 0.0},
    {"key": "7d",   "label": "📅 7 дней", "apr": 0.15, "lock_days": 7, "early_penalty": 0.10},
    {"key": "14d",  "label": "📅 14 дней","apr": 0.22, "lock_days": 14,"early_penalty": 0.12},
    {"key": "30d",  "label": "📅 30 дней","apr": 0.35, "lock_days": 30,"early_penalty": 0.15}
  ]
}
```

---

## Приватные эндпоинты (нужен `Authorization: Bearer <ключ>`)

### `GET /api/v1/me`
Баланс и активы владельца ключа.

```json
{
  "ok": true,
  "user_id": 123456789,
  "usd": 850.0,
  "vrf": 145.3,
  "vrf_value_usd": 150.28,
  "staked_vrf": 500.0,
  "pending_rewards_vrf": 1.42
}
```

### `POST /api/v1/trade/buy`
Купить VRF за USD.

```json
// запрос
{ "usd_amount": 100 }

// ответ
{ "ok": true, "usd": 750.0, "vrf": 245.3, "price": 1.0342 }
```

### `POST /api/v1/trade/sell`
Продать VRF за USD.

```json
{ "vrf_amount": 50 }
```

### `GET /api/v1/stakes`
Список активных стейков.

```json
{
  "ok": true,
  "stakes": [
    {
      "id": 17, "tier": "7d", "amount": 500.0, "apr": 0.15,
      "lock_days": 7, "start_ts": "2026-06-28T10:00:00",
      "accrued_vrf": 2.31, "matured": false, "seconds_left": 214000
    }
  ]
}
```

### `POST /api/v1/stakes`
Открыть новый стейк.

```json
// запрос
{ "tier": "30d", "amount": 1000 }

// ответ: 201 + свежий список стейков (как GET /api/v1/stakes)
```

### `POST /api/v1/stakes/{id}/claim`
Забрать накопленные проценты (только гибкий тариф, тело не нужно).

```json
{ "ok": true, "claimed_vrf": 3.14, "vrf_balance": 148.44 }
```

### `POST /api/v1/stakes/{id}/unstake`
Вывести стейк.

- Если срок истёк (или тариф гибкий) — выводится тело + все проценты сразу.
- Если срок ещё не истёк — сервер вернёт `409` с деталями штрафа;
  повторите запрос с `{"confirm_early": true}`, чтобы подтвердить
  досрочный вывод (проценты сгорают, применяется штраф).

```json
// досрочно, шаг 1 — сервер отвечает 409 с деталями
{ "ok": false, "error": "Lock not expired (214000s left). ... Resend with {\"confirm_early\": true} to proceed." }

// шаг 2
POST /api/v1/stakes/17/unstake
{ "confirm_early": true }

{ "ok": true, "early": true, "penalty_vrf": 60.0, "payout_vrf": 440.0, "vrf_balance": 588.44 }
```

---

## Примеры

### curl

```bash
curl https://your-app.up.railway.app/api/v1/price

curl -H "Authorization: Bearer vrf_live_xxx" \
     https://your-app.up.railway.app/api/v1/me

curl -X POST -H "Authorization: Bearer vrf_live_xxx" \
     -H "Content-Type: application/json" \
     -d '{"usd_amount": 100}' \
     https://your-app.up.railway.app/api/v1/trade/buy
```

### JavaScript (fetch)

```js
const API = "https://your-app.up.railway.app";

async function getPrice() {
  const r = await fetch(`${API}/api/v1/price`);
  return r.json();
}

async function buyVrf(apiKey, usdAmount) {
  const r = await fetch(`${API}/api/v1/trade/buy`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ usd_amount: usdAmount }),
  });
  return r.json();
}
```

### Python (requests)

```python
import requests

API = "https://your-app.up.railway.app"

price = requests.get(f"{API}/api/v1/price").json()

me = requests.get(
    f"{API}/api/v1/me",
    headers={"Authorization": "Bearer vrf_live_xxx"},
).json()
```

---

## Развёртывание (Railway)

1. В настройках сервиса включите **Public Networking** для порта,
   который слушает бот (`$PORT`, Railway подставляет автоматически).
2. Опционально задайте `API_CORS_ORIGIN` (домен вашего сайта вместо `*`).
3. API стартует автоматически вместе с ботом — отдельного процесса не нужно.

## Коды ошибок

| Код | Значение |
|-----|----------|
| 400 | Некорректный запрос / тело |
| 401 | Неверный или отсутствующий API-ключ |
| 404 | Ресурс не найден (например, чужой/несуществующий stake id) |
| 409 | Требуется подтверждение (досрочный вывод стейка) |
| 429 | Превышен лимит запросов |
| 500 | Внутренняя ошибка сервера |
