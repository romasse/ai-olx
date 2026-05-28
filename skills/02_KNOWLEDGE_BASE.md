# OLX Partner API — База знаний для AI-асистента

> **Призначення документа:** ця база знань призначена для використання як контекст (system prompt / knowledge base) для AI-асистента, який допомагає розробникам та користувачам інтегруватися з OLX Partner API. Документ містить повний витяг офіційної документації OLX Partner API v2.0 з акцентом на ринок України та сценарії використання у сфері нерухомості.
>
> **Джерела:**
> - Офіційна OpenAPI 3.0 специфікація `partner_api.yaml` (OLX Europe)
> - Портал розробників OLX UA — `https://developer.olx.ua/api/doc`
>
> **Версія документа:** травень 2026
> **Версія API:** 2.0 (обов'язковий заголовок `Version: 2.0`)
> **Базовий URL для України:** `https://www.olx.ua/api/partner`

---

## 0. Як використовувати цей документ

Цей файл побудовано як референс. Кожна секція має заголовок другого рівня (`##`), що дозволяє асистенту швидко знаходити потрібний блок за ключовими словами. Структура:

1. Швидкий старт і базові поняття
2. Автентифікація (OAuth 2.0) — повний потік
3. Каталог ендпоінтів за функціональними групами
4. Життєвий цикл оголошення
5. Правила контенту та валідація (правила OLX)
6. Платежі, пакети, платні функції
7. Каталог помилок з рішеннями
8. Архітектурні рекомендації для multi-tenant SaaS
9. Спеціфіка для нерухомості (Україна, ринок риелторів)
10. Чекліст готовності до інтеграції

---

## 1. Швидкий старт

### 1.1. Що це таке

OLX Partner API — REST API від OLX Europe, який дозволяє партнерам:

- публікувати, редагувати та видаляти оголошення від імені користувачів OLX;
- читати статистику оголошень (перегляди, дзвінки, спостерігачі);
- спілкуватися з користувачами через внутрішню систему повідомлень OLX;
- купувати платні функції просування та пакети публікацій;
- працювати з каталогами (категорії, міста, регіони, валюти, мови).

API єдине для всіх країн Європи, де працює OLX, відмінності тільки на рівні базового URL та локальних правил (валюти, мови, доступні платні функції).

### 1.2. Обов'язкові заголовки кожного запиту

```
Authorization: Bearer <access_token>
Version: 2.0
Accept-Language: uk
Content-Type: application/json
```

Відсутність заголовка `Version` повертає 400 `Missing required 'Version' header!`

### 1.3. Базові поняття

| Термін | Значення |
|---|---|
| `client_id`, `client_secret` | Креденшіали додатку, видаються OLX вручну після подання заявки |
| `access_token` | Токен для виклику API. Живе **86400 сек (24 год)** |
| `refresh_token` | Токен для оновлення access. Живе **2592000 сек (30 днів)** |
| `code` | Одноразовий код авторизації. Живе **10 хвилин** |
| `scope` | Дозволи токена: `v2`, `read`, `write` (стандартно `v2 read write`) |
| `external_id` | Ваш внутрішній ID оголошення (в CRM). Ключове поле для ідемпотентної синхронізації |
| `external_url` | URL оголошення у вашій системі. **В Україні недоступне** (тільки PL/Jobs) |

---

## 2. Автентифікація (OAuth 2.0)

### 2.1. Три типи grant types

| Grant type | Контекст | Що дозволяє |
|---|---|---|
| `authorization_code` | Користувач (риелтор) | Постити оголошення, читати повідомлення, працювати від імені користувача |
| `client_credentials` | Додаток | Тільки читання довідників (категорії, міста, регіони, валюти) |
| `refresh_token` | Будь-який | Оновлення `access_token` без повторної авторизації користувача |

### 2.2. Повний потік OAuth для multi-user додатка (SaaS-кейс)

**Крок 1.** Користувач натискає в вашому додатку кнопку «Підключити OLX»:

```
https://www.olx.ua/oauth/authorize/
  ?client_id=<your_client_id>
  &response_type=code
  &state=<random_hash>
  &scope=read+write+v2
  &redirect_uri=https://yourapp.com/auth/olx/callback
```

- `state` — випадковий хеш, який ви зберігаєте і перевіряєте при поверненні (захист від CSRF).
- `redirect_uri` — обов'язковий, якщо у вашому додатку зареєстровано більше одного Callback URL.
- Уникайте спецсимволів `&` (multiple), `#` в `redirect_uri`.

**Крок 2.** Користувач логиниться в OLX і підтверджує підключення.

**Крок 3.** OLX редиректить його на `redirect_uri` з параметрами:

```
https://yourapp.com/auth/olx/callback?code=d34feg43g456g&state=<your_random_hash>
```

**Крок 4.** Ваш бекенд обмінює `code` на пару токенів:

```http
POST https://www.olx.ua/api/open/oauth/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "code": "<code_from_step_3>",
  "scope": "v2 read write",
  "redirect_uri": "https://yourapp.com/auth/olx/callback"
}
```

**Відповідь:**

```json
{
  "access_token": "454c3cd39a980500e9b787e03d53ec5c320f644b",
  "expires_in": 86400,
  "token_type": "bearer",
  "scope": "v2 read write",
  "refresh_token": "12a13560163da7901d7f74c89fc5df3ea7625942"
}
```

**Крок 5.** Зберігаємо `access_token` та `refresh_token` у вашій БД (зашифровано!) разом з user_id вашого додатку.

### 2.3. Оновлення токена

Коли `access_token` ось-ось протухне (рекомендую за 1 годину до `expires_in`):

```http
POST https://www.olx.ua/api/open/oauth/token

{
  "grant_type": "refresh_token",
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "refresh_token": "<saved_refresh_token>"
}
```

У відповіді отримаєте **новий** `access_token` і, можливо, **новий** `refresh_token` — обов'язково оновіть обидва в БД.

### 2.4. Client credentials (для довідників)

```http
POST https://www.olx.ua/api/open/oauth/token

{
  "grant_type": "client_credentials",
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "scope": "v2 read"
}
```

Цей токен **не має прив'язки до користувача** і працює тільки з ендпоінтами справочників. Якщо спробувати з ним викликати `/users/me` або `/adverts`, отримаєте `Invalid owner in token`.

---

## 3. Каталог ендпоінтів

Усі ендпоінти мають префікс `/api/partner`. Базовий URL: `https://www.olx.ua/api/partner`.

### 3.1. Користувачі

| Метод | Шлях | Опис | Scope |
|---|---|---|---|
| GET | `/users/me` | Отримати дані автентифікованого користувача | `read` |
| GET | `/users/{id}` | Отримати дані іншого користувача | `read` |
| GET | `/users/me/account-balance` | Баланс кредитів на рахунку | `read` |
| GET | `/users/me/payment-methods` | Доступні методи оплати | `read` |
| GET | `/users-business/me` | Дані бізнес-акаунту | `read` |
| GET | `/users-business/me/logos` | Логотипи бізнес-акаунту | `read` |
| POST | `/users-business/me/logos` | Завантажити логотип | `write` |
| GET | `/users-business/me/banners` | Банери бізнес-акаунту | `read` |

### 3.2. Географія

| Метод | Шлях | Опис |
|---|---|---|
| GET | `/regions` | Список областей |
| GET | `/regions/{regionId}` | Деталі області |
| GET | `/cities` | Список міст (підтримує `region_id`, `query` для пошуку, `limit` за замовч. 1000) |
| GET | `/cities/{cityId}` | Деталі міста |
| GET | `/cities/{cityId}/districts` | Райони міста |
| GET | `/districts` | Список усіх районів |
| GET | `/districts/{districtId}` | Деталі району |
| GET | `/locations?latitude=&longitude=` | Знайти місто/район за координатами |

**Важливо для України:** для великих міст (Київ, Дніпро, Львів, Харків, Одеса) `district_id` є обов'язковим при публікації оголошення.

### 3.3. Довідники

| Метод | Шлях | Опис |
|---|---|---|
| GET | `/languages` | Доступні мови |
| GET | `/currencies` | Доступні валюти |
| GET | `/categories` | Дерево категорій |
| GET | `/categories/{categoryId}` | Деталі категорії (включно з `is_leaf`, `photos_limit`) |
| GET | `/categories/{categoryId}/attributes` | Атрибути категорії (динамічна схема полів) |
| GET | `/categories/suggestion?title=` | Підказка категорії за назвою оголошення |

### 3.4. Оголошення (адверти) — серце API

| Метод | Шлях | Опис | Scope |
|---|---|---|---|
| GET | `/adverts` | Список ваших оголошень | `read` |
| POST | `/adverts` | Створити оголошення | `write` |
| GET | `/adverts/{advertId}` | Отримати оголошення | `read` |
| PUT | `/adverts/{advertId}` | Оновити оголошення | `write` |
| DELETE | `/adverts/{advertId}` | Видалити (тільки якщо не active) | `write` |
| POST | `/adverts/{advertId}/commands` | Команди: activate / deactivate / finish / extend | `write` |
| GET | `/adverts/{advertId}/statistics` | Статистика оголошення | `read` |
| DELETE | `/adverts/{advertId}/statistics/{statName}` | Очистити статистику | `write` |
| GET | `/adverts/{advertId}/moderation-reason` | Причина модерації (HTML) | `read` |
| GET | `/adverts/{advertId}/logos` | Логотипи на оголошенні | `read` |
| POST | `/adverts/{advertId}/logos` | Додати логотип | `write` |

**GET `/adverts` query параметри:**
- `offset` — зсув для пагінації;
- `limit` — кількість на сторінку;
- `external_id` — пошук за вашим внутрішнім ID (ключове для синхронізації з CRM);
- `category_ids` — список категорій через кому.

### 3.5. Повідомлення (чат OLX)

| Метод | Шлях | Опис | Scope |
|---|---|---|---|
| GET | `/threads` | Список тредів (підтримує `advert_id`, `interlocutor_id`, `offset`, `limit`) | `read` |
| GET | `/threads/{threadId}` | Деталі треду | `read` |
| GET | `/threads/{threadId}/messages` | Всі повідомлення в треді | `read` |
| POST | `/threads/{threadId}/messages` | Відправити повідомлення | `write` |
| GET | `/threads/{threadId}/messages/{messageId}` | Одне повідомлення | `read` |
| POST | `/threads/{threadId}/commands` | Команди над тредом (mark-as-read, favourite тощо) | `write` |

**Критично важливе обмеження:** вебхуків на нові повідомлення немає. Потрібно поллити `/threads?unread_count>0` за розкладом.

### 3.6. Платні функції та пакети

| Метод | Шлях | Опис |
|---|---|---|
| GET | `/paid-features` | Список доступних платних функцій (топ, виділення тощо) |
| GET | `/adverts/{advertId}/paid-features` | Активні платні функції для оголошення |
| POST | `/adverts/{advertId}/paid-features` | Купити платну функцію для оголошення |
| GET | `/packets` | Доступні пакети публікацій (підтримує `category_id`) |
| GET | `/zones` | Зони для платних функцій |
| GET | `/users/me/packets` | Куплені пакети користувача |
| POST | `/users/me/packets` | Купити пакет |
| GET | `/adverts/{advertId}/packets` | Пакети, прив'язані до оголошення |
| POST | `/adverts/{advertId}/packets` | Прив'язати оголошення до пакету |

### 3.7. Фінансові звіти

| Метод | Шлях | Опис |
|---|---|---|
| GET | `/users/me/billing` | Загальна біллінг-інформація |
| GET | `/users/me/prepaid-invoices` | Передплатні рахунки |
| GET | `/users/me/postpaid-invoices` | Постоплатні рахунки |

---

## 4. Життєвий цикл оголошення

Це найважливіший розділ. Помилки тут — найчастіша причина «оголошення зависає і не публікується».

### 4.1. Статуси оголошення

```
new        → оголошення створене, очікує модерації
active     → опубліковане, видиме всім користувачам
limited    → НЕ опубліковане, потрібно купити пакет та активувати
removed_by_user → видалене користувачем
```

### 4.2. Сценарій 1: Створення без необхідності пакета

```
POST /adverts → status: "new"
   ↓ (модерація OLX, зазвичай кілька секунд–хвилин)
status: "active" ✓
```

### 4.3. Сценарій 2: Створення в категорії з лімітом

```
POST /adverts → status: "limited"
   ↓
GET /packets?category_id=<cat> → знайти підходящий пакет
   ↓
POST /users/me/packets → купити пакет
   ↓
POST /adverts/{id}/packets → прив'язати оголошення до пакету
   ↓
POST /adverts/{id}/commands { "command": "activate" }
   ↓ (модерація)
status: "active" ✓
```

### 4.4. Деактивація оголошення

```http
POST /adverts/{advertId}/commands
{
  "command": "deactivate",
  "is_success": true
}
```

`is_success: true` — означає, що ви успішно продали об'єкт (це впливає на статистику аккаунту).
`is_success: false` — деактивуєте з інших причин.

### 4.5. Видалення оголошення

**Двоетапне:**

```http
1. POST /adverts/{id}/commands { "command": "deactivate", "is_success": false }
2. DELETE /adverts/{id}
```

Якщо викликати DELETE на active — отримаєте 400 `Invalid status`.

**Throttling cost = 5** (DELETE — найдорожчий запит).

### 4.6. Команди над оголошенням

| Команда | Що робить | Обмеження |
|---|---|---|
| `activate` | Активує неактивне оголошення | Потрібен пакет, якщо категорія лімітована |
| `deactivate` | Деактивує активне оголошення | Поле `is_success` обов'язкове |
| `finish` | Переводить limited у finished | — |
| `extend` | Продовжує період активності | **В Україні НЕДОСТУПНА** (та в PT) |
| (refresh) | Бамп оголошення | Не частіше ніж раз на **14 днів** |

---

## 5. Правила контенту та валідація OLX

Це чорний список правил, які OLX накладає на заголовки, описи, фото та інше. Якщо їх не дотримуватись — отримаєте 400 з валідацією.

### 5.1. Заголовок (title)

- ❌ Не більше **50% символів у верхньому регістрі** (CAPS LOCK).
- ❌ Заборонені email-адреси, www-адреси, телефонні номери в тексті.
- ❌ Не більше **2 підряд** знаків пунктуації: `! ? . , - = + # % & @ * _ > < : ( ) |`
- ❌ Спецсимволи з трьох та більше підряд — блокуються.

### 5.2. Опис (description)

- Ті самі правила, що для заголовка (капс, email/www, телефон, пунктуація).
- Для категорії **Jobs** (працевлаштування) дозволяється HTML: `<ul>`, `<li>`, `<strong>`, `<em>`, `<p>`. Для нерухомості — **тільки plain text**.

### 5.3. Категорія

- `category_id` має бути **листовою** (`is_leaf: true` у відповіді `GET /categories/{id}`).
- Помилка `Fix the category` означає, що вибрано не листову категорію.
- Для підказки правильної категорії за назвою — `GET /categories/suggestion?title=...`.

### 5.4. Атрибути (attributes)

- Кожна категорія має свій набір атрибутів. Отримати їх: `GET /categories/{id}/attributes`.
- Кожен атрибут має `code`, опціонально `value` (single) або `values` (multiple).
- Формат:
  ```json
  "attributes": [
    { "code": "type", "value": "fulltime" },
    { "code": "rooms", "value": "2" }
  ]
  ```
- Не додавайте інших ключів окрім `code` + (`value` або `values`) — отримаєте `This value is not valid: attributes`.

### 5.5. Локація

- `city_id` — обов'язковий.
- `district_id` — обов'язковий, якщо у міста є райони (Київ, Дніпро тощо).
- `latitude`/`longitude` — мають відповідати обраному місту/району. Перевірити: `GET /locations?latitude=&longitude=`.
- Помилка `Your coordinates are too far from picked location` — перевірте відповідність координат і `city_id`.

### 5.6. Фото

- Ліміт залежить від категорії: див. поле `photos_limit` у `GET /categories/{id}`.
- URL фотографій у полі `images` мають бути доступними публічно — OLX викачує їх до себе.
- Помилка `Remote file not exists` — URL недоступний/протух.
- Помилка `Image limit exceeded` — перевищено `photos_limit`.

### 5.7. Контакти

- `name` — обов'язкове.
- `phone` — список номерів через кому. Формат має бути валідним для країни (для UA — `+380XXXXXXXXX` або `380XXXXXXXXX`).
- Помилка `Invalid phone format` — телефон не пройшов локальну валідацію.

### 5.8. Ціна

```json
"price": {
  "value": 50000,
  "currency": "USD",
  "negotiable": false,
  "trade": false,
  "budget": false
}
```

- `negotiable` — «торг можливий».
- `trade` — «обмін».
- `budget` — позначка «бюджетна».
- Для нерухомості в Україні зазвичай `currency: "USD"` для продажу та `currency: "UAH"` для оренди (але це угода ринку, не вимога API).

---

## 6. Шаблон POST `/adverts` для нерухомості

Базовий приклад тіла запиту для публікації оголошення про продаж квартири у Дніпрі:

```json
{
  "title": "2-кімнатна квартира на Перемозі, 65 м²",
  "description": "Простора 2-кімнатна квартира з ремонтом, окрема кухня, балкон засклений. Поруч школа, садок, метро 5 хвилин. Документи готові до угоди.",
  "category_id": 1147,
  "advertiser_type": "business",
  "external_id": "R-2026-0531",
  "contact": {
    "name": "Роман",
    "phone": "+380501234567"
  },
  "location": {
    "city_id": 9,
    "district_id": 145,
    "latitude": 48.4647,
    "longitude": 35.0462
  },
  "images": [
    { "url": "https://cdn.yourcrm.com/photos/R-2026-0531/01.jpg" },
    { "url": "https://cdn.yourcrm.com/photos/R-2026-0531/02.jpg" }
  ],
  "price": {
    "value": 58000,
    "currency": "USD",
    "negotiable": true,
    "trade": false,
    "budget": false
  },
  "attributes": [
    { "code": "rooms", "value": "2" },
    { "code": "total_area", "value": "65" },
    { "code": "floor", "value": "3" },
    { "code": "floors_in_building", "value": "9" }
  ],
  "auto_extend_enabled": true
}
```

> ⚠️ `category_id`, `district_id` та коди атрибутів вище — приклади. Реальні значення треба отримати через `GET /categories`, `GET /cities/{cityId}/districts`, `GET /categories/{id}/attributes`.

---

## 7. Каталог помилок з рішеннями

### 7.1. Загальний формат помилки

```json
{
  "error": {
    "status": 400,
    "title": "Invalid request",
    "detail": "Data validation error occurred",
    "validation": [
      { "field": "title", "title": "Musisz podać tytuł" }
    ]
  }
}
```

### 7.2. HTTP-коди

| Код | Що означає |
|---|---|
| 400 | Валідація не пройшла (дивитися `validation` масив) |
| 401 | Невалідний/відсутній токен або scope |
| 403 | Зрозумілий запит, але виконання заборонене |
| 404 | Ресурс не знайдено |
| 406 | API не може повернути в форматі, який ви просите |
| 415 | Невідповідний Content-Type |
| 429 | Too Many Requests — спрацював throttling |
| 500 | Помилка на сервері OLX |

### 7.3. Помилки авторизації

| Помилка | Причина | Рішення |
|---|---|---|
| `Missing parameter: code is required` | У запиті обміну токена немає `code` | Передати `code` |
| `Authorization code doesn't exist or is invalid` | `code` протух (>10 хв) або вже використаний | Запустити OAuth-флоу заново |
| `The grant type was not specified` | Немає `grant_type` | Додати `grant_type` |
| `The scope requested is invalid` | Запитуєте scope, який не дозволено вашому акаунту | Використовуйте `read write v2` |
| `Client is not active` | Невірний `client_id` або клієнт деактивовано | Перевірити `client_id`, написати в OLX |
| `The client credentials are invalid` | Невірний `client_id`/`client_secret` | Перевірити креденшіали |
| `The access token provided is invalid` | Токен невалідний | Оновити через `refresh_token` |
| `Insufficient scope` | Токену не вистачає привілеїв | Запитати `read write v2` (не просто `read write`) |
| `Invalid owner in token` | Використовуєте `client_credentials` там, де потрібен `authorization_code` | Перейти на `authorization_code` |
| `The grant type is unauthorized for this client_id` | Партнерський акаунт не може використовувати цей grant type | Звернутися до OLX |
| `Invalid refresh token` | `refresh_token` протух (>30 днів) | Запустити OAuth-флоу заново для користувача |

### 7.4. Помилки публікації

| Помилка | Рішення |
|---|---|
| `Category with given ID doesn't exists` | Перевірити через `GET /categories/{id}` |
| `Fix the category` | Категорія не листова — потрібна `is_leaf: true` |
| `Error while decoding JSON data: Syntax error` | Перевірити JSON; не передавати зайвих полів |
| `Partner is not allowed to use external URL` | `external_url` доступне тільки в PL/Jobs |
| `Invalid value: district_id` | Перевірити `GET /cities/{cityId}/districts` |
| `Your coordinates are too far from picked location` | Перевірити `GET /locations?latitude=&longitude=` |
| `This value is not valid: attributes` | У `attributes` має бути тільки `code` + `value` або `values` |
| `Image error: Remote file not exists` | URL зображення недоступний |
| `Image error: Image limit exceeded` | Перевищили `photos_limit` категорії |
| `Unsupported API version` | Перевірити URL запиту |
| `Advert not found` | Невірний `advertId` |
| `You are not the owner of this ad` | Намагаєтесь керувати чужим оголошенням |
| `Too many capital letters` | >50% капсу в заголовку/описі |
| `Field is not valid. Emails and www addresses are not allowed` | Видалити email/www з тексту |
| `Field is not valid. Phone numbers are not allowed` | Видалити телефон з тексту |
| `Field contains to much punctuation` | Не більше 2 однакових знаків пунктуації підряд |
| `Ad has to be active` (deactivate) | Деактивувати можна тільки active оголошення |
| `Invalid status` (delete) | Перед DELETE викликати deactivate |
| `You cannot refresh ad more often than once in 14 days` | Чекати 14 днів від попереднього бампа |
| `City with given ID doesn't exists` | Перевірити через `GET /cities/{id}` |
| `Invalid phone format` | Привести телефон до національного формату |
| `Missing required 'Version' header!` | Додати `Version: 2.0` у заголовки |

### 7.5. Помилки оплати

| Помилка | Рішення |
|---|---|
| `No possibility to buy the packet` (`There is no variant with size X for category Y`) | Перевірити `GET /packets?category_id={id}` |
| `Invalid payment method` | Перевірити `GET /users/me/payment-methods` |
| `Payment method 'postpaid' is not activated` | Звернутися до OLX UA для активації постоплати |
| `Not enough credits` | Поповнити баланс або перейти на іншу оплату |

---

## 8. Архітектура для multi-tenant SaaS (рекомендації)

### 8.1. Зберігання токенів клієнтів

- **Шифрування at-rest:** використовуйте KMS / Vault / pgcrypto. `access_token` і `refresh_token` — це повний доступ до акаунту клієнта.
- **Структура таблиці:**
  ```
  olx_credentials
    user_id UUID         -- ID користувача у вашій системі
    olx_user_id INTEGER  -- ID користувача в OLX (з /users/me)
    access_token TEXT    -- зашифровано
    refresh_token TEXT   -- зашифровано
    access_expires_at TIMESTAMP
    refresh_expires_at TIMESTAMP
    scope TEXT
    created_at TIMESTAMP
    updated_at TIMESTAMP
  ```

### 8.2. Фоновий воркер оновлення токенів

- Раз на годину сканувати таблицю та оновлювати токени, які протухнуть протягом наступних 2 годин.
- Якщо `refresh_token` протух — позначити користувача як `needs_reauth` і повідомити його (email/push), що треба переавторизуватися.

### 8.3. Throttling та черга запитів

- Усі запити до OLX мають іти через **чергу** (Bull/Sidekiq/Celery) з retry-логікою.
- На 429 — експоненціальний backoff (1с → 2с → 4с → 8с → 16с → fail).
- Враховуйте `Throttling cost`: DELETE = 5, інші — припускати 1.
- Окремі черги на read-операції (можна паралелити) і write (обережніше).

### 8.4. Ідемпотентність через `external_id`

- Для кожного об'єкту у вашій CRM генеруйте стабільний `external_id` (наприклад, `R-2026-0531`).
- Перед публікацією: `GET /adverts?external_id=R-2026-0531`. Якщо є — `PUT`, якщо ні — `POST`.
- Це робить синхронізацію CRM → OLX повністю відновлюваною та безпечною.

### 8.5. Контент-валідатор перед публікацією

Не покладайтеся на OLX як на валідатор. Зробіть свій pre-flight checker:

- Перевірити заголовок і опис на: капс (>50%), email/www, телефонні номери, пунктуацію (>2 однакових підряд);
- Перевірити, що `category_id` — листова;
- Перевірити, що `district_id` відповідає `city_id`;
- Перевірити, що `latitude/longitude` у межах допустимої відстані від центру міста;
- Перевірити, що URL фото повертають 200 OK;
- Перевірити кількість фото проти `photos_limit`.

**Бонус:** LLM (Claude, GPT) можна підключити для автоматичної нормалізації описів — він прибере зайвий капс, перепише дозвонені «КУПУЙ!!!» в нормальний текст, обріже до правильної довжини.

### 8.6. Поллінг повідомлень (немає вебхуків!)

- Активні клієнти (відкриті оголошення з трафіком) — поллити `GET /threads` раз на 1–2 хвилини.
- Неактивні (немає активних оголошень) — раз на 15–30 хвилин.
- Зберігайте `last_thread_id` і `last_message_id`, щоб не обробляти повідомлення двічі.

### 8.7. Логування

Усі запити до OLX логуйте з:
- request ID;
- user_id клієнта;
- метод і шлях;
- статус відповіді;
- час відповіді;
- (для помилок) повний body відповіді.

---

## 9. Специфіка для нерухомості (Україна, риелтори)

### 9.1. Типові категорії OLX UA для нерухомості

> Точні `category_id` отримуйте через `GET /categories`. Нижче — приклади ієрархії.

- Нерухомість
  - Квартири
    - Продаж квартир
    - Довгострокова оренда квартир
    - Подобова оренда квартир
  - Будинки
    - Продаж будинків
    - Оренда будинків
  - Комерційна нерухомість
    - Продаж комерційної нерухомості
    - Оренда офісів / складів / магазинів
  - Земля
    - Продаж землі
    - Оренда землі
  - Гаражі і паркінги

### 9.2. Типові атрибути нерухомості

Точний список — через `GET /categories/{categoryId}/attributes`. Найчастіші:

| Атрибут (приклад коду) | Значення |
|---|---|
| `rooms` | Кількість кімнат |
| `total_area` | Загальна площа (м²) |
| `living_area` | Житлова площа |
| `kitchen_area` | Площа кухні |
| `floor` | Поверх |
| `floors_in_building` | Поверховість будинку |
| `building_type` | Тип будинку (цегла, панель, моноліт, новобудова) |
| `condition` | Стан (євроремонт, потребує ремонту, з ремонтом) |
| `heating` | Опалення (індивідуальне, центральне) |
| `furniture` | Меблі (є / немає) |
| `parking` | Паркінг |

### 9.3. Робота з ціною для нерухомості

- Для продажу зазвичай `USD`, для оренди — `UAH`.
- Для довгострокової оренди — місячна ціна.
- Для подобової — за добу.
- `negotiable: true` майже завжди для продажу (риелторська практика).

### 9.4. Платні функції просування

Через `GET /paid-features` отримаєте поточний список. Для нерухомості найкорисніші:
- **VIP / Топ** — підняття у топ списку категорії.
- **Виділення кольором** — оголошення виділяється в стрічці.
- **Бамп (refresh)** — підняття на перші позиції (але не частіше ніж раз на 14 днів).

Купівля через `POST /adverts/{advertId}/paid-features`.

### 9.5. Регуляторика (Україна)

- При обробці персональних даних користувачів OLX (включно з номерами телефонів, які приходять у threads) ви підпадаєте під Закон України «Про захист персональних даних».
- Якщо плануєте AI-чатбот для відповідей у threads — публічно інформуйте клієнтів, що відповідає AI (це етична норма та потенційно регуляторна вимога).
- Голосові дзвінки через AI (TTS-боти) у Європі поступово регулюються (AI Act EU); в Україні поки що м'який режим, але краще проектувати з прицілом на майбутні норми.

---

## 10. Чекліст готовності до інтеграції

### 10.1. Перед першим запитом

- [ ] Подана заявка в OLX, отримано `client_id` та `client_secret`.
- [ ] Зареєстровано всі `redirect_uri` для OAuth (production + staging).
- [ ] У всіх запитах є заголовок `Version: 2.0`.
- [ ] Всі запити йдуть з `Authorization: Bearer <token>`.
- [ ] Налаштовано шифрування зберігання токенів.

### 10.2. Перед першою публікацією

- [ ] Завантажено та закешовано довідник категорій (раз на добу оновлення).
- [ ] Завантажено довідник міст, районів, регіонів.
- [ ] Для кожної використовуваної категорії — закешовано схему атрибутів.
- [ ] Налаштовано pre-flight content validator (капс, email/www, телефон, пунктуація, фото).
- [ ] Налаштовано ідемпотентність через `external_id`.

### 10.3. Перед продакшеном

- [ ] Налаштована черга запитів з retry та backoff на 429.
- [ ] Фоновий воркер оновлення токенів працює.
- [ ] Налаштоване сповіщення користувачів про `needs_reauth`.
- [ ] Поллінг threads запущено з адекватним інтервалом.
- [ ] Логування всіх запитів/відповідей працює.
- [ ] Налаштовано моніторинг (Sentry/Datadog) на 4xx/5xx від OLX.
- [ ] Є окремий sandbox-акаунт OLX для тестів.

---

## 11. Корисні шаблони запитів (для асистента)

### 11.1. Створити оголошення з нуля

```http
POST https://www.olx.ua/api/partner/adverts
Authorization: Bearer <access_token>
Version: 2.0
Content-Type: application/json

{
  "title": "...",
  "description": "...",
  "category_id": <leaf_category_id>,
  "advertiser_type": "business",
  "external_id": "<your_internal_id>",
  "contact": { "name": "...", "phone": "+380..." },
  "location": { "city_id": ..., "district_id": ..., "latitude": ..., "longitude": ... },
  "images": [{ "url": "https://..." }],
  "price": { "value": ..., "currency": "USD", "negotiable": true, "trade": false, "budget": false },
  "attributes": [{ "code": "...", "value": "..." }],
  "auto_extend_enabled": true
}
```

### 11.2. Оновити існуюче оголошення (upsert по external_id)

```http
GET /adverts?external_id=R-2026-0531
   ↓ якщо є — отримуємо advertId
PUT /adverts/{advertId}    (те саме тіло, що для POST)
   ↓ якщо немає
POST /adverts
```

### 11.3. Купити пакет та активувати оголошення

```http
GET /packets?category_id=1147           → знайти потрібний пакет
POST /users/me/packets                  → купити
POST /adverts/{id}/packets              → прив'язати
POST /adverts/{id}/commands { "command": "activate" }
```

### 11.4. Відповісти на повідомлення в чаті OLX

```http
GET /threads?advert_id={advertId}       → знайти треди по оголошенню
GET /threads/{threadId}/messages        → прочитати листування
POST /threads/{threadId}/messages
{
  "text": "Доброго дня! Дякую за звернення..."
}
```

### 11.5. Витягнути статистику для звіту клієнту

```http
GET /adverts?limit=100&offset=0         → список усіх активних оголошень
   ↓ для кожного
GET /adverts/{id}/statistics            → {advert_views, phone_views, users_observing}
```

---

## 12. Глосарій

| Термін | Значення |
|---|---|
| **Advert** | Оголошення на OLX |
| **Thread** | Діалог (тред) у внутрішніх повідомленнях OLX |
| **Packet** | Пакет публікацій (передплачена кількість оголошень у категорії) |
| **Paid feature** | Платна функція просування (топ, виділення) |
| **Zone** | Географічна зона для деяких платних функцій |
| **Leaf category** | Кінцева категорія в дереві, у яку можна публікувати оголошення |
| **External ID** | Ваш внутрішній ID оголошення (для ідемпотентної синхронізації) |
| **Throttling cost** | «Вартість» запиту в системі обмежень OLX (різні ендпоінти коштують по-різному) |
| **Scope** | Дозволи токена (`read`, `write`, `v2`) |
| **Authorization code** | Одноразовий код OAuth, що обмінюється на токени (живе 10 хв) |

---

## 13. Куди звертатись

- **Документація:** https://developer.olx.ua/api/doc
- **Postman колекція:** https://app.getpostman.com/run-collection/3bf2713b8a249cdd917c
- **OAuth-документація:** https://www.oauth.com/oauth2-servers/definitions/
- **Підтримка партнерів OLX UA** — через форму заявки на отримання API-доступу.

---

*Документ створено як knowledge base для AI-асистента. Версія: 1.0. Дата: травень 2026.*
