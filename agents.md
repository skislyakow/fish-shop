# agents.md — Развёртывание и перенос разработки (Fish Shop)

Проект: интернет-магазин морепродуктов Дениса.
Состоит из двух частей:

- `cms/` — Strapi 5 (headless CMS, БД SQLite). Хранит каталог (Product/Category) и заказы (Order).
- `bot/` — Telegram-бот (grammy), заказывает товары через REST API Strapi.

---

## 1. Требования к окружению

Strapi 5 требует Node.js **20+** (engines: `>=20.0.0 <=26.x.x`).
Локально ставили через nvm:

```powershell
nvm install 24      # или 20
nvm use 24
node -v             # v24.x
npm -v
```

> На новой машине обязательно переключитесь на Node 20/24 перед `npm install`.

---

## 2. Локальный запуск

### CMS
```powershell
cd cms
npm install                 # если проект клонирован
cp .env.example .env        # если нет .env (он НЕ коммитится)
npm run develop             # http://localhost:1337/admin
```
Если `.env` отсутствует, Strapi сам сгенерирует его с новыми ключами при первом `develop`.

### Бот (когда появится папка bot/)
```powershell
cd bot
npm install
cp .env.example .env        # заполнить BOT_TOKEN и STRAPI_URL
node index.js
```

---

## 3. Что коммитим, а что нет

Strapi уже сгенерировала корректный `.gitignore`. В нём игнорируются:

| Игнорируется | Почему |
|---|---|
| `node_modules/` | зависимости ставятся через `npm install` |
| `.tmp/` | рабочая БД SQLite (`.tmp/data.db`) — **данные не уходят в git** |
| `.env` | секреты (ключи, JWT, БД) — **НЕ коммитить** |
| `build/`, `dist/` | собранная admin-панель |
| `public/uploads/*` | загруженные картинки товаров (оставлен только `.gitkeep`) |

**Коммитим:** `src/`, `config/`, `package.json`, `package-lock.json`, `.env.example`,
сгенерированные типы (`types/`), а также `agents.md` и `README.md`.

> `package-lock.json` НЕ игнорируется — коммитим его для воспроизводимой установки.

---

## 4. Push / Clone на другой машине

**Можно ли просто запушить и клонировать? — Да, для кода.**

Схема контент-типов (Product/Category/Order) живёт в `src/api/*` и поедет в git.
Поэтому на новой машине структура магазина восстановится автоматически.

**Но данные (созданные товары, заказы) лежат в `.tmp/data.db`, который игнорируется**,
поэтому админка после клона будет пустой. Реальные данные переносим отдельно (см. раздел 5).

### Рекомендуемый вариант — единый репозиторий на уровне `fish-shop/`

```powershell
# ОДИН РАЗ, локально:
cd C:\Code\fish-shop
# cms уже содержит вложенный .git — уберите его, чтобы не было вложенных репозиториев:
Remove-Item -Recurse -Force cms\.git
git init
git add .
git commit -m "init: Strapi cms + bot scaffold"
git branch -M main
git remote add origin <URL>
git push -u origin main
```

На другой машине:
```powershell
git clone <URL> fish-shop
cd fish-shop\cms
nvm use 24
npm install
cp .env.example .env        # или просто запустить develop — Strapi сгенерирует .env
npm run develop             # Strapi создаст пустую БД и схему из src/api

cd ..\bot
npm install
cp .env.example .env
node index.js
```

> Альтернатива: держать `cms` и `bot` как два независимых репозитория и клонировать их по отдельности.

---

## 5. Перенос реальных данных (товары / заказы)

1. **Strapi data transfer (рекомендуется)** — экспорт/импорт содержимого вместе с медиа:
   ```powershell
   # на исходной машине:
   cd cms
   npm run strapi export -- --file shop-data
   # -> создаёт exports/shop-data.tar.gz (в git НЕ попадает)

   # перенести файл на новую машину, затем:
   npm run strapi import -- --file shop-data.tar.gz
   ```
2. **Ручной ре-сидинг** — заново создать товары в админке или скриптом.
3. **Копия БД** — физически перенести `.tmp/data.db` (работает при одинаковой версии Strapi, но НЕ через git).

---

## 6. Полезные команды Strapi

| Команда | Назначение |
|---|---|
| `npm run develop` | dev-режим с авто-перезапуском при изменениях |
| `npm run build` + `npm run start` | продакшн-режим |
| `npm run strapi export/import/transfer` | перенос данных между машинами |
| `npm run strapi console` | отладочная консоль |

---

## 7. Проверка интеграции

- Каталог через API: `GET http://localhost:1337/api/products?populate=image,category`
- Заказ, оформленный в боте, появляется в админке → Content Manager → Order.
- Типичные ошибки:
  - `403` — нет прав у роли Public или не передан API-токен;
  - `401` — неверный/пустой токен;
  - пустой список товаров — забыли нажать **Publish** в Content Manager.
