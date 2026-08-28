# Стек разработки AIM Рутина

## Frontend (Telegram Mini App)

### Фреймворк и язык
- **React 18+** с TypeScript
- **Vite** — сборка и dev-сервер
- **Telegram Mini Apps SDK** — интеграция с Telegram (@twa-dev/sdk)

### UI/UX библиотеки
- **Tailwind CSS** — стилизация (OLED Black theme)
- **Framer Motion** — анимации
- **Recharts** — радарные диаграммы (Spider Chart) и графики
- **Lucide React** — иконки

### Управление состоянием
- **Zustand** — глобальное состояние
- **TanStack Query** — серверное состояние и кеширование

---

## Backend

### Платформа
- **Supabase** (PostgreSQL + Auth + Realtime + Edge Functions + Storage)
  - Аутентификация через Telegram
  - Row Level Security для изоляции данных пользователей
  - Realtime для уведомлений
  - Edge Functions (Deno) для serverless логики

### AI-интеграция
- **Google Gemini API** (бесплатный тариф)
  - Gemini 1.5 Flash / Gemini 2.0 Flash для генерации плейлистов
  - Function calling для инструментов анализа

### Voice модуль
- **Whisper (OpenAI)** или **Deepgram** — Speech-to-Text
- **ElevenLabs** — Text-to-Speech

---

## Внешние API интеграции

### FACEIT API
```
Base URL: https://open.faceit.com/datav4
Endpoints:
- GET /players — информация об игроке
- GET /players/{player_id}/history — история матчей
- GET /players/{player_id}/stats/{game_id} — статистика игрока
- GET /matches/{match_id} — детальная информация о матче

Требования:
- API Key (Header: Authorization: Bearer {key})
- Rate limits: 500 запросов/час (бесплатный тариф)
```

### Tracker.gg API
```
Base URL: https://public-api.tracker.gg/v2
Endpoints:
- GET /csgo/standard/profile/{platform}/{identifier} — профиль CS2
- GET /valorant/standard/profile/{platform}/{identifier} — профиль Valorant
- GET /csgo/standard/profile/{platform}/{identifier}/sessions — сессии матчей

Требования:
- API Key (Header: TRN-Api-Key)
- Rate limits: 1000 запросов/день (бесплатный)
```

### Platega.io (Платежи)
```
Base URL: https://api.platega.io
Функционал:
- Прием карт (Visa, Mastercard, МИР)
- СБП (QR-код, оплата по ссылке)
- Криптовалюта (USDT, BTC, GRAM)
- Webhooks для уведомлений о платежах

Тарифы:
- Карты: от 5%
- СБП: от 4%
- Криптовалюта: от 1%
```

### Aim Lab API / Share Codes
```
Aim Lab не предоставляет публичный REST API.
Интеграция осуществляется через Share Code систему.

Share Code формат:
- Формат: XXXX-XXXX (8 символов, uppercase + цифры)
- Пример: AB12-CD34

Механизм работы:
1. Генерация Share Code на основе базы сценариев
2. Пользователь копирует код → открывает Aim Lab → код автоматически подгружает плейлист
3. Deep link для открытия: aimlab://loadscenario?sharecode=XXXX-XXXX

База сценариев (разметка):
- Категории: Flick, Track, Precision, Reaction, Multi-target, Switch
- Сложность: Easy, Medium, Hard
- Длительность: 3, 5, 8, 10, 15, 20 минут
- Упражнения: targets, borders, walls, ceilings, floors, moving, shrinking

Пример структуры плейлиста:
{
  "name": "Precision Flick — 8 мин",
  "scenarios": [
    { "share_code": "AB12-CD34", "type": "Flick", "reps": 50 },
    { "share_code": "EF56-GH78", "type": "Precision", "reps": 40 },
    { "share_code": "IJ90-KL12", "type": "Tracking", "reps": 30 }
  ],
  "total_duration_minutes": 8,
  "target_game": "CS2"
}

Генерация кода:
- Share Code хранится в таблице playlists (share_code VARCHAR)
- AI генерирует последовательность кодов через Function Calling
- База данных сценариев: table aim_lab_scenarios (id, name, share_code, tags, duration)
```

### KovaaK'3D API / Workshop Codes
```
KovaaK'3D не предоставляет публичный REST API.
Интеграция через Workshop Share Codes.

Share Code формат:
- Формат: 9-значный alphanumeric код
- Пример: 1A2B3C4D5

Механизм работы:
1. Генерация Workshop Code на основе базы KovaaK'3D сценариев
2. Пользователь копирует код → открывает KovaaK'3D → Workshop → Paste Code
3. Сценарии автоматически загружаются из Steam Workshop

Deep link для KovaaK'3D:
- kovaaaks://workshop?code=1A2B3C4D5
- Или через Steam: steam://link/609650 (ID игры)

База сценариев (разметка):
- Категории: Tracking, Clicking, Transition, Micro-adjust, Target-switching
- Сложность: Beginner, Intermediate, Advanced, Expert
- Длительность: 3, 5, 8, 10, 15, 20 минут
- Упражнения: walls, pillars, targets, moving, shrinking, invisible

Пример структуры плейлиста:
{
  "name": "Micro Precision — 10 мин",
  "scenarios": [
    { "code": "1A2B3C4D5", "type": "Micro-adjust", "reps": 60 },
    { "code": "6F7G8H9I0", "type": "Clicking", "reps": 45 },
    { "code": "2J3K4L5M6", "type": "Tracking", "reps": 35 }
  ],
  "total_duration_minutes": 10,
  "target_game": "Valorant"
}

Генерация кода:
- Workshop Code хранится в таблице playlists (share_code VARCHAR)
- AI генерирует последовательность кодов через Function Calling
- База данных сценариев: table kovaak_scenarios (id, name, workshop_code, tags, duration)
```

---

## База данных (Supabase PostgreSQL)

### Основные таблицы
```sql
users (
  id UUID PRIMARY KEY,
  telegram_id BIGINT UNIQUE,
  tier VARCHAR(20) DEFAULT 'bronze',
  created_at TIMESTAMP,
  settings JSONB
)

user_stats (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  flick_score INT,
  track_score INT,
  micro_adjust_score INT,
  switch_score INT,
  spray_score INT,
  updated_at TIMESTAMP
)

faceit_accounts (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  faceit_id VARCHAR,
  game VARCHAR,
  verified BOOLEAN
)

tracker_accounts (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  platform VARCHAR,
  identifier VARCHAR,
  game VARCHAR
)

match_history (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  match_id VARCHAR,
  platform VARCHAR,
  game VARCHAR,
  stats JSONB,
  analyzed BOOLEAN,
  created_at TIMESTAMP
)

playlists (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  share_code VARCHAR,
  aim_type VARCHAR[],
  duration_minutes INT,
  generated_at TIMESTAMP
)

-- База сценариев Aim Lab
aim_lab_scenarios (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  share_code VARCHAR(10) UNIQUE NOT NULL,
  categories VARCHAR[],           -- Flick, Track, Precision, Reaction, Multi-target
  difficulty VARCHAR,             -- Easy, Medium, Hard
  duration_minutes INT,
  reps INT,
  tags VARCHAR[],
  created_at TIMESTAMP DEFAULT NOW()
)

-- База сценариев KovaaK'3D
kovaak_scenarios (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  workshop_code VARCHAR(10) UNIQUE NOT NULL,
  categories VARCHAR[],           -- Tracking, Clicking, Transition, Micro-adjust
  difficulty VARCHAR,             -- Beginner, Intermediate, Advanced, Expert
  duration_minutes INT,
  reps INT,
  tags VARCHAR[],
  created_at TIMESTAMP DEFAULT NOW()
)

training_sessions (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  playlist_id UUID REFERENCES playlists(id),
  completed BOOLEAN,
  completed_at TIMESTAMP
)

payments (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  amount DECIMAL,
  currency VARCHAR,
  tier VARCHAR,
  status VARCHAR,
  platega_transaction_id VARCHAR,
  created_at TIMESTAMP
)
```

---

## Инфраструктура

### Хостинг и Deploy
- **Vercel** или **Cloudflare Pages** — фронтенд TMA
- **Supabase Cloud** — backend + database
- **GitHub Actions** — CI/CD

### Мониторинг
- **Sentry** — error tracking
- **Supabase Dashboard** — метрики БД и функций

---

## Техническое задание для AI-генерации проекта

### Проект: AIM Рутина — AI-тренер по аиму (Telegram Mini App)

#### 1. Описание проекта
Telegram Mini App для автоматизированного персонального коучинга стрельбы в FPS-шутерах (CS2, Valorant). Система анализирует статистику матчей через FACEIT/Tracker.gg API, генерирует персональные тренировочные плейлисты для Aim Lab/KovaaK's с помощью Google Gemini AI, и отслеживает прогресс игрока.

#### 2. Архитектура системы
```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Telegram TMA   │────▶│   Supabase API   │────▶│  FACEIT API     │
│  (React + Vite) │     │  (PostgreSQL +   │     │  Tracker.gg API │
└─────────────────┘     │  Edge Functions) │     └─────────────────┘
                        └──────────────────┘              │
                                 │                        │
                                 ▼                        │
                        ┌──────────────────┐             │
                        │  Google Gemini   │◀────────────┘
                        │  AI Coach        │
                        └──────────────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │  Platega.io      │
                        │  (Payments)      │
                        └──────────────────┘
```

#### 3. Функциональные требования

**3.1. Система аутентификации**
- Вход через Telegram (initData validation)
- Привязка FACEIT аккаунта по никнейму
- Привязка Riot ID для Valorant (Tracker.gg)
- Привязка Steam ID для CS2

**3.2. AI-коучинг (Google Gemini)**
- Анализ сыгранных матчей с выявлением слабых мест (Flick, Track, Micro-adjust, Switch, Spray)
- Генерация Share Code для Aim Lab/KovaaK's на основе анализа
- Голосовой ввод/вывод (STT/TTS)
- Function calling для инструментов:
  - `get_tracker_stats(player_id, platform)`
  - `analyze_match_performance(match_data)`
  - `generate_playlist_code(aim_type, duration)`

**3.3. Тарифная система**
- Bronze (Free): ручной ввод, 10 текстовых запросов/день, 1 плейлист/неделя
- Silver (Pro): безлимит текст + 30 голосовых/день, 3 плейлиста/неделя
- Elite (Ultimate): автосбор статистики, полный безлимит, авто-генерация после каждого матча

**3.4. Интеграция платежей (Platega.io)**
- Приём оплаты картами РФ, СБП, криптовалютой
- Webhooks для автоматической активации тарифа
- История платежей в профиле пользователя

**3.5. Дашборд и визуализация**
- Радарная диаграмма 5 навыков (Spider Chart)
- Daily Streak счетчик тренировок
- График динамики ELO/K/D
- Карточки с Share Code для копирования

#### 4. API Endpoints (Supabase Edge Functions)

```
POST /auth/telegram — валидация Telegram initData
POST /accounts/link-faceit — привязка FACEIT аккаунта
POST /accounts/link-tracker — привязка Tracker.gg
GET /stats/:userId — получение статистики пользователя
POST /analyze/match — анализ матча через Gemini AI
POST /playlist/generate — генерация плейлиста
POST /payments/create — создание платежа в Platega
POST /payments/webhook — обработка webhook от Platega
```

#### 5. Telegram Mini App SDK функции

- `WebApp.MainButton` — кнопка главного действия
- `WebApp.HapticFeedback` — тактильная обратная связь
- `WebApp.showPopup` — всплывающие окна
- `WebApp.openLink` — открытие внешних ссылок
- `WebApp.ready()` — инициализация приложения

#### 6. Цветовая палитра

```css
--bg-primary: #0F1015;      /* OLED Black */
--accent-cs2: #FF5500;      /* FACEIT Orange */
--accent-valorant: #FF4655; /* Valorant Red */
--accent-ai: #8A2BE2;       /* AI Violet */
```

#### 7. Лимиты и квоты API

| Сервис | Лимит | Тариф |
|--------|-------|-------|
| Gemini API | 1500 RPD | Free |
| FACEIT API | 500 req/hour | Free |
| Tracker.gg | 1000 req/day | Free |
| Supabase | 500MB DB, 1GB storage | Free |

#### 8. Этапы разработки

1. **Setup**: инициализация React + Vite проекта, подключение Telegram SDK, настройка Supabase
2. **Auth**: реализация входа через Telegram, привязка игровых аккаунтов
3. **Core UI**: дашборд, экран статистики, экран тренировок
4. **AI Integration**: подключение Gemini, промпты для коучинга, генерация плейлистов
5. **External APIs**: интеграция FACEIT и Tracker.gg, обработка данных матчей
6. **Payments**: подключение Platega.io, обработка платежей, система тарифов
7. **Voice**: STT/TTS модуль для голосового коучинга
8. **Polish**: анимации, haptic feedback, оптимизация

#### 9. Админ панель

### 9.1. Описание
Админ панель — отдельный экран в TG Mini App (доступен только пользователям с `role: 'admin'` или `role: 'superadmin'`). Реализована как встроенная страница внутри Mini App, а не отдельное веб-приложение.

### 9.2. Роли и доступ
```sql
-- Новая колонка в users
ALTER TABLE users ADD COLUMN role VARCHAR(20) DEFAULT 'user';
-- Возможные значения: 'user', 'admin', 'superadmin'

-- Супер-админ устанавливается вручную через SQL:
UPDATE users SET role = 'superadmin' WHERE telegram_id = <telegram_id>;
```

### 9.3. Функционал админ панели

#### 9.3.1. Управление пользователями
| Функция | Описание |
|---------|----------|
| Поиск пользователей | По telegram_id, имени, никнейму |
| Просмотр профиля | Tier, дата регистрации, привязанные аккаунты, история платежей |
| Ручное изменение подписки | Выбор нового tier (bronze/silver/elite), установка даты окончания, причина изменения |
| Блокировка/разблокировка | Временная или постоянная блокировка доступа |
| История действий | Лог всех изменений, выполненных админом |

#### 9.3.2. Система скидок (Promo Codes)
```sql
CREATE TABLE promo_codes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code VARCHAR(50) UNIQUE NOT NULL,
  discount_type VARCHAR(20) NOT NULL,  -- 'percentage' | 'fixed'
  discount_value DECIMAL(10, 2) NOT NULL,
  tier VARCHAR(20) NOT NULL,           -- 'silver' | 'elite' | 'all'
  max_uses INT DEFAULT 1,
  used_count INT DEFAULT 0,
  expires_at TIMESTAMP,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  is_active BOOLEAN DEFAULT TRUE
);
```

| Функция | Описание |
|---------|----------|
| Создание промокода | Код, тип скидки (% или фикс), скидка, тариф, лимит использований, срок действия |
| Редактирование | Изменение параметров активного промокода |
| Деактивация | Временная блокировка без удаления |
| Статистика | Количество использований, список пользователей |
| Генератор | Автогенерация случайных промокодов (например, `AIM-2024-X7K9`) |

#### 9.3.3. Пуш-уведомления
```sql
CREATE TABLE push_notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(100) NOT NULL,
  message TEXT NOT NULL,
  target_audience VARCHAR(20) NOT NULL,  -- 'all' | 'bronze' | 'silver' | 'elite' | 'custom'
  custom_tiers VARCHAR[],               -- если target_audience = 'custom'
  deep_link VARCHAR,                    -- ссылка для перехода внутри TMA
  sent_at TIMESTAMP,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP DEFAULT NOW(),
  status VARCHAR(20) DEFAULT 'draft'    -- 'draft' | 'sent' | 'failed'
);

CREATE TABLE push_delivery_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  notification_id UUID REFERENCES push_notifications(id),
  user_id UUID REFERENCES users(id),
  delivered BOOLEAN,
  delivered_at TIMESTAMP,
  error_message TEXT
);
```

| Функция | Описание |
|---------|----------|
| Создание уведомления | Заголовок, текст, аудитория (все / по tier / кастомная) |
| Deep link | Ссылка на конкретный экран TMA (например, `/training`) |
| Отправка | Через Telegram Bot API `sendMessage` каждому пользователю |
| История | Статус отправки, количество доставленных/недоставленных |
| Расписание | Отложенная отправка (cron через Supabase Edge Function) |

**Механизм отправки:**
```
Edge Function: /admin/send-push-notification
1. Получить список пользователей по target_audience
2. Для каждого: bot.telegram.sendMessage(chat_id, message, {
     parse_mode: 'HTML',
     reply_markup: { inline_keyboard: [[{ text: 'Открыть', url: deep_link }]] }]
   })
3. Записать результат в push_delivery_log
```

### 9.4. Админ эндпоинты (Edge Functions)

```
POST /admin/users/search        — поиск пользователей
GET  /admin/users/:id           — профиль пользователя
PUT  /admin/users/:id/tier      — изменение подписки
PUT  /admin/users/:id/ban       — блокировка/разблокировка

POST /admin/promo/create        — создание промокода
GET  /admin/promo               — список всех промокодов
PUT  /admin/promo/:id           — редактирование
PUT  /admin/promo/:id/deactivate — деактивация
GET  /admin/promo/:id/stats     — статистика использования

POST /admin/push/create         — создание уведомления
POST /admin/push/:id/send       — отправка уведомления
GET  /admin/push                — история уведомлений
GET  /admin/push/:id/deliveries — лог доставки

GET  /admin/dashboard           — общая статистика (пользователи, платежи, revenue)
```

### 9.5. Админ UI экраны

1. **Дашборд** — KPI: кол-во пользователей, активные подписки, доход за период, конверсия
2. **Пользователи** — таблица с поиском, фильтрами, пагинацией
3. **Промокоды** — список, создание, редактирование, статистика
4. **Пуш-уведомления** — конструктор, история, лог доставки
5. **Аналитика** — графики регистрации, платежей, retention

---

## Экраны TG Mini App

### 1. Экран приветствия (Onboarding)
**Назначение:** Знакомство пользователя с приложением, объяснение ценности.

**Контент:**
- Анимированный логотип AIM Рутина
- 3 слайда преимуществ:
  1. "AI анализирует твою игру" — иконка мозга/анализа
  2. "Генерирует персональные тренировки" — иконка плейлиста
  3. "Рост аима за 2 недели" — иконка графика роста
- Кнопка "Начать" (MainButton Telegram)
- После нажатия: автоматическая авторизация через Telegram

**Состояния:**
- Новый пользователь → переход на экран профиля
- Возвращающийся пользователь → переход на Dashboard

---

### 2. Главный экран (Dashboard)
**Назначение:** Основная точка входа, обзор прогресса и быстрые действия.

**Контент:**
- **Шапка:**
  - Аватар пользователя (Telegram)
  - Имя + никнейм
  - Иконка текущего тарифа (Bronze/Silver/Elite бейдж)
  - Кнопка настроек (шестерёнка)
- **Spider Chart (Радарная диаграмма):**
  - 5 осей: Flick, Track, Micro-adjust, Switch, Spray
  - Цвет: фиолетовый (#8A2BE2) для AI, оранжевый (#FF5500) для CS2, красный (#FF4655) для Valorant
  - Переключатель игры (CS2 / Valorant)
- **Daily Streak:**
  - Счётчик дней подряд: 🔥 X дней
  - Мини-календарь с отмеченными днями тренировок
- **Quick Actions (карточки):**
  - "Пройти разбор" → переход на AI чат
  - "Мой плейлист" → экран тренировок
  - "Статистика" → экран аналитики
  - "Привязать аккаунт" → экран интеграций
- **Рекомендация AI:**
  - Карточка с коротким советом: "Заметил падение flick-шотов после 15-й игры. Рекомендую 8-минутный сет на precision."
- **MainButton Telegram:** "AI Разбор" — быстрый запуск анализа

**Состояния:**
- Нет привязанных аккаунтов → баннер "Подключи FACEIT для автоанализа"
- Elite tier → кнопка "Автоанализ активен"

---

### 3. Экран AI-чата (Coach Chat)
**Назначение:** Интерактивный диалог с AI-тренером.

**Контент:**
- **История сообщений:**
  - Сообщения AI с аватаром и анимацией появления
  - Текстовые и голосовые сообщения пользователя
  - Голосовые ответы AI с плеером (play/pause, прогресс)
- **Ввод:**
  - Текстовое поле с подсказкой "Напиши или отправь голосовое..."
  - Кнопка микрофона (запись голосового сообщения)
  - Кнопка отправки
- **Quick Replies (чипы):**
  - "Разбери мою последнюю игру"
  - "Дай плейлист на flick"
  - "Что улучшить?"
  - "Покажи статистику"
- **Лимиты (для Bronze):**
  - Прогресс-бар: "X/10 запросов сегодня"
  - Кнопка "Повысить тариф"

**Состояния:**
- AI думает → анимация пульсации
- Голосовое воспроизведение → waveform анимация

---

### 4. Экран тренировок (Playlist Hub)
**Назначение:** Просмотр и управление тренировочными плейлистами.

**Контент:**
- **Текущий рекомендованный плейлист:**
  - Название: "Precision Flick — 8 мин"
  - Типы упражнений: [Flick] [Precision] [Tracking]
  - Share Code: `XXXX-XXXX` с кнопкой "Копировать"
  - Кнопка "Открыть в Aim Lab" / "Открыть в KovaaK's"
  - Таймер обратного отсчёта до следующего рекомендации
- **История тренировок:**
  - Список прошлых плейлистов с датами
  - Статус: "Пройден" / "Не пройден" / "Частично"
  - Кнопка "Повторить" для копирования Share Code
- **Фильтры:**
  - По типу: Flick, Track, Micro-adjust, Switch, Spray
  - По длительности: 5, 8, 15, 20 мин
  - По игре: CS2, Valorant
- **Лимиты (по тарифу):**
  - Bronze: "1 плейлист доступен на этой неделе"
  - Silver: "3/3 использовано"
  - Elite: "Безлимит"

**Определение платформы (Device Detection):**
```typescript
// Telegram Mini App SDK — определение платформы пользователя
import { WebApp } from '@twa-dev/sdk';

const platform = WebApp.platform;     // 'ios' | 'android' | 'macos' | 'web' | 'unknown'
const isDesktop = platform === 'macos' || platform === 'web' || platform === 'unknown';
const isMobile = platform === 'ios' || platform === 'android';

// Альтернатива через UserAgent
const isTelegramDesktop = /TelegramDesktop/i.test(navigator.userAgent);
const isTelegramIOS = /Telegram/i.test(navigator.userAgent) && /iPhone|iPad/i.test(navigator.userAgent);
const isTelegramAndroid = /TelegramAndroid/i.test(navigator.userAgent) || /Android/i.test(navigator.userAgent);

// Итоговое определение
const deviceType = isTelegramDesktop || isTelegramIOS || isTelegramAndroid ? 'desktop' : 'mobile';
```

**Поведение в зависимости от платформы:**

| Платформа | Поведение | Кнопка |
|-----------|-----------|--------|
| **Desktop** (Mac, Web, PC) | Показывает кнопку "Открыть в игре" с deep link | `aimlab://loadscenario?sharecode=XXXX-XXXX` или `kovaaaks://workshop?code=1A2B3C4D5` |
| **iOS** | Показывает кнопку "Копировать код" + подсказка "Вставь в Aim Lab / KovaaK's" | Копирование в буфер обмена + haptic feedback |
| **Android** | Показывает кнопку "Копировать код" + подсказка "Вставь в Aim Lab / KovaaK's" | Копирование в буфер обмена + haptic feedback |

**Desktop UX:**
- Крупная кнопка с иконкой игры: "Открыть в Aim Lab" (оранжевая) / "Открыть в KovaaK's" (синяя)
- При нажатии: `WebApp.openLink('aimlab://loadscenario?sharecode=XXXX-XXXX')`
- Если игра не установлена → fallback: показать скопированный код + инструкция "Откройте Aim Lab → Сценарии → Введите код"
- Haptic feedback: `WebApp.HapticFeedback.notificationOccurred('success')`

**Mobile UX:**
- Кнопка "Скопировать код" (фиолетовая) с иконкой буфера обмена
- После копирования: анимация успеха + текст "Код скопирован! Вставьте в Aim Lab"
- Копирование: `navigator.clipboard.writeText('XXXX-XXXX')`
- Haptic feedback: `WebApp.HapticFeedback.selectionChanged()`
- Дополнительно: кнопка "Поделиться" (share card с кодом и QR-кодом)

**QR-код (для мобильных):**
- Генерируется через библиотеку `qrcode.react`
- Содержит deep link: `aimlab://loadscenario?sharecode=XXXX-XXXX`
- Пользователь сканирует камерой → открывает Aim Lab на ПК

**Адаптивная карточка плейлиста:**
```
┌──────────────────────────────────┐
│  🎯 Precision Flick — 8 мин      │
│  [Flick] [Precision] [Tracking]  │
│                                  │
│  Share Code:  AB12-CD34          │
│                                  │
│  ┌──────────────────────────┐    │
│  │  [Открыть в Aim Lab]     │    │  ← Desktop
│  └──────────────────────────┘    │
│  ┌──────────────────────────┐    │
│  │  [Копировать код]        │    │  ← Mobile
│  │  [Поделиться ▸]          │    │
│  └──────────────────────────┘    │
└──────────────────────────────────┘
```

---

### 5. Экран статистики (Analytics)
**Назначение:** Детальная аналитика прогресса игрока.

**Контент:**
- **Общая статистика:**
  - ELO / Ранг (с графиком динамики)
  - K/D ratio
  - HS% (Headshot %)
  - KPR / DPR
- **Графики:**
  - Линейный график ELO за последние 30 дней
  - Столбчатая диаграмма HS% по неделям
  - Корреляция тренировок и результата (линия тренировок + линия ELO)
- **Разбивка по механикам:**
  - Таблица: Flick, Track, Micro-adjust, Switch, Spray
  - Оценки за последние 7/30 дней
  - Тренд: ↑ / → / ↓
- **Match History (Elite tier):**
  - Список последних 20 матчей
  - Результат (W/L), счёт, карта, K/D
  - AI вердикт: "Хороший первый патрон", "Падение точности во 2-й половине"
  - Кнопка "Разобрать" → переход в AI-чат с контекстом матча

**Состояния:**
- Нет привязанного аккаунта → баннер "Подключи FACEIT для автоанализа"
- Bronze tier → "Детальная статистика доступна в Silver+"

---

### 6. Экран интеграций (Account Linking)
**Назначение:** Привязка игровых аккаунтов для автоанализа.

**Контент:**
- **FACEIT:**
  - Иконка FACEIT
  - Поле ввода никнейма
  - Кнопка "Подключить"
  - Статус: "Не подключено" / "Подключено: player123" / "Ошибка: аккаунт не найден"
  - Отображение ELO, уровня, ранга после успешной привязки
- **Tracker.gg:**
  - Иконка Tracker.gg
  - Поле ввода Riot ID (для Valorant) или Steam ID (для CS2)
  - Кнопка "Подключить"
  - Статус аналогично FACEIT
- **Привязка Valorant:**
  - Riot ID: `username#tag`
  - Поддержка платформ: PC, KR, JP, LATAM, BR, EU, NA, AP
- **Информация о данных:**
  - "Мы получаем статистику матчей, K/D, HS% и другие метрики"
  - Ссылка на политику конфиденциальности

---

### 7. Экран профиля (Profile)
**Назначение:** Настройки аккаунта, тариф, привязанные аккаунты.

**Контент:**
- **Информация:**
  - Аватар, имя, telegram_id
  - Текущий тариф с бейджем
  - Дата регистрации
- **Текущий тариф:**
  - Название тарифа
  - Список включённых функций (чек-лист)
  - Кнопка "Повысить тариф" → переход на экран оплаты
  - Для Silver/Elite: дата окончания подписки
- **Привязанные аккаунты:**
  - FACEIT, Tracker.gg, Steam — статус и никнейм
  - Кнопка "Отвязать" для каждого
- **Настройки:**
  - Язык (RU / EN)
  - Целевая игра (CS2 / Valorant / Обе)
  - DPI и чувствительность (для точного анализа)
  - Уведомления (вкл/выкл)
  - Тема (OLED Black / Dark Gray)
- **Админ панель:**
  - Отображается только если `role: 'admin'` или `'superadmin'`
  - Кнопка "Админ панель" → переход на админ экран

---

### 8. Экран оплаты (Subscription)
**Назначение:** Выбор и оплата тарифа.

**Контент:**
- **Сравнение тарифов (3 колонки):**
  - **Bronze (Free)** — текущий, галочка "Текущий"
  - **Silver (Pro)** — кнопка "Повысить"
  - **Elite (Ultimate)** — кнопка "Повысить"
- **Функции каждого тарифа (чек-лист):**
  - Аналитика, лимиты AI, плейлисты, интеграции, голосовой коучинг
- **Ввод промокода:**
  - Поле ввода + кнопка "Применить"
  - Сообщение об успехе/ошибке
- **Способы оплаты (через Platega):**
  - Карты РФ (Visa, Mastercard, МИР)
  - СБП (QR-код)
  - Криптовалюта (USDT, BTC, GRAM)
- **Кнопка "Оплатить" — открывает платежный виджет Platega**

**После оплаты:**
- Webhook от Platega → обновление tier в БД
- Уведомление: "Тариф Elite активирован!"
- Анимация конфетти

---

### 9. Экран админ панели (Admin)
**Назначение:** Управление пользователями, промокодами, пуш-уведомлениями.

**Контент:**
- **Дашборд:**
  - Всего пользователей (активных / новых за день)
  - Активные подписки: Silver / Elite
  - Доход за период (график)
  - Конверсия Free → Paid
- **Пользователи:**
  - Поиск по telegram_id / имени
  - Таблица: аватар, имя, tier, дата регистрации, последний вход
  - Действия: изменить тариф, заблокировать, просмотреть профиль
  - Модальное окно изменения тарифа: выбор нового tier, дата окончания, причина
- **Промокоды:**
  - Таблица: код, тип скидки, значение, использований, срок, статус
  - Кнопка "Создать промокод" → модальное окно
  - Поля: код, тип (%/фикс), значение, тариф, лимит, срок
  - Кнопка "Сгенерировать" → автогенерация
- **Пуш-уведомления:**
  - Таблица: заголовок, аудитория, статус, дата отправки
  - Кнопка "Создать" → конструктор
  - Поля: заголовок, текст, аудитория (все / по tier), deep link
  - Кнопка "Отправить" → асинхронная отправка
  - Лог доставки: delivered / failed для каждого пользователя
- **Аналитика:**
  - График регистрации пользователей
  - График платежей
  - Retention: D1, D7, D30
  - Топ функций по использованию

---

### 10. Экран настроек (Settings)
**Назначение:** Глобальные настройки приложения.

**Контент:**
- **Аккаунт:**
  - Редактирование имени (если поддерживается Telegram)
  - Выход из аккаунта
- **Уведомления:**
  - Push-уведомления от бота (вкл/выкл)
  - Напоминания о тренировках (время)
  - Уведомление за 20 мин до прайма (Elite)
  - Anti-Tilt напоминания (Elite)
- **Конфиденциальность:**
  - Экспорт данных
  - Удаление аккаунта и данных
- **О приложении:**
  - Версия
  - Ссылки: Telegram-канал, поддержка, оферта
  - Рейтинг приложения (кнопка "Оценить")

---

### 11. Модальные окна

#### 11.1. Подтверждение действия
- Заголовок, описание, кнопки "Отмена" / "Подтвердить"
- Используется для: изменение тарифа, удаление аккаунта, отвязка аккаунта

#### 11.2. Успешное изменение
- Анимация галочки, заголовок, описание
- Используется для: успешная оплата, привязка аккаунта, применение промокода

#### 11.3. Ошибка
- Анимация "x", заголовок, описание, кнопка "Повторить"
- Используется для: ошибка API, неверный промокод, сбой платежа

---

### 12. Навигация

**Нижняя навигация (Tab Bar):**
1. 🏠 Главная (Dashboard)
2. 💬 AI Чат
3. 🎯 Тренировки
4. 📊 Статистика
5. 👤 Профиль

**Переходы:**
- MainButton Telegram → AI Чат
- Кнопки на карточках → соответствующие экраны
- Deep link из пуш-уведомления → целевой экран
- Swipe между экранами (Dashboard ↔ Статистика)

**Telegram MainButton:**
- На Dashboard: "AI Разбор" → AI Чат
- На Тренировках (Desktop): "Открыть в Aim Lab" → `WebApp.openLink('aimlab://...')`
- На Тренировках (Mobile): "Копировать код" → `navigator.clipboard.writeText()` + haptic feedback
- На Оплате: "Оплатить" → открытие платежного виджета

---

## Локальный запуск проекта

### 1. Требования

```
Node.js >= 18.0.0
npm >= 9.0.0 или pnpm >= 8.0.0
Docker >= 24.0.0 (для локального Supabase)
Git
```

### 2. Клонирование репозитория

```bash
git clone <repository_url>
cd aim-rutina
```

### 3. Структура проекта

```
aim-rutina/
├── frontend/                 # TG Mini App (React + Vite)
│   ├── src/
│   │   ├── components/      # UI компоненты
│   │   ├── pages/           # Экраны (Dashboard, Chat, Training, etc.)
│   │   ├── hooks/           # Custom hooks
│   │   ├── stores/          # Zustand stores
│   │   ├── lib/             # Утилиты, API клиенты
│   │   ├── types/           # TypeScript типы
│   │   └── App.tsx
│   ├── index.html
│   ├── vite.config.ts
│   ├── tailwind.config.js
│   ├── package.json
│   └── .env.local
├── backend/                  # Supabase Edge Functions
│   ├── functions/
│   │   ├── _shared/         # Общие модули
│   │   ├── auth/telegram/
│   │   ├── accounts/link-faceit/
│   │   ├── accounts/link-tracker/
│   │   ├── stats/
│   │   ├── analyze/match/
│   │   ├── playlist/generate/
│   │   ├── payments/create/
│   │   ├── payments/webhook/
│   │   ├── admin/users/search/
│   │   ├── admin/users/:id/tier/
│   │   ├── admin/promo/create/
│   │   ├── admin/push/create/
│   │   ├── admin/push/:id/send/
│   │   └── admin/dashboard/
│   ├── deno.json
│   └── deploy.sh
├── supabase/                 # Локальный Supabase
│   ├── config.toml
│   ├── migrations/
│   │   ├── 001_initial.sql
│   │   ├── 002_promo_codes.sql
│   │   ├── 003_push_notifications.sql
│   │   ├── 004_aim_lab_scenarios.sql
│   │   └── 005_kovaak_scenarios.sql
│   ├── seed.sql             # Начальные данные (сценарии Aim Lab / KovaaK)
│   └── schemas/
│       ├── rles.sql
│       └── policies.sql
├── .env.example              # Шаблон переменных окружения
├── docker-compose.yml        # Supabase локально
├── README.md
└── stack.md
```

### 4. Переменные окружения

**frontend/.env.local:**
```env
# Telegram Mini App
VITE_TELEGRAM_BOT_NAME=aim_rutina_bot
VITE_TELEGRAM_APP_URL=https://app.aimrutina.com

# Supabase
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key

# API Keys
VITE_FACEIT_API_KEY=your-faceit-key
VITE_TRACKER_API_KEY=your-tracker-key
VITE_GEMINI_API_KEY=your-gemini-key
VITE_PLETEGA_API_KEY=your-platega-key

# Voice (опционально)
VITE_WHISPER_API_KEY=your-whisper-key
VITE_ELEVENLABS_API_KEY=your-elevenlabs-key
```

**backend/functions/.env (для Edge Functions):**
```env
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
FACEIT_API_KEY=your-faceit-key
TRACKER_API_KEY=your-tracker-key
GEMINI_API_KEY=your-gemini-key
PLETEGA_API_KEY=your-platega-key
TELEGRAM_BOT_TOKEN=your-telegram-bot-token
```

**supabase/.env (локальный):**
```env
POSTGRES_PASSWORD=your-secret-postgres-password
DATABASE_URL=postgresql://postgres.your-secret-postgres-password@localhost:54322/postgres
```

### 5. Запуск локального Supabase

```bash
# Установка Supabase CLI
# Windows (PowerShell):
# scoop install supabase

# Инициализация локального проекта (первый раз)
supabase init

# Запуск локальных сервисов (Postgres, Auth, Storage, Edge Functions, Studio)
supabase start

# Просмотр логов
supabase logs

# Остановка
supabase stop

# Сброс и перезапуск (если сломалось)
supabase db reset
supabase stop && supabase start
```

### 6. Миграции базы данных

```bash
# Создание первой миграции
supabase migration new initial

# Применение миграций на локальный сервер
supabase db push

# Откат последней миграции
supabase db reset

# Просмотр текущих миграций
supabase migration list

# Синхронизация схемы сremote
supabase db push --linked
```

### 7. Запуск Edge Functions локально

```bash
# Деплой всех функций на локальный Supabase
supabase functions deploy --project-ref local

# Деплой конкретной функции
supabase functions deploy auth/telegram --project-ref local

# Локальный сервер функций (dev mode с hot reload)
supabase functions serve --project-ref local

# Проверка работы функции
curl http://localhost:54321/functions/v1/auth/telegram \
  -H "Authorization: Bearer your-anon-key" \
  -d '{"initData": "..."}'
```

### 8. Запуск фронтенда

```bash
cd frontend

# Установка зависимостей
npm install

# Запуск dev-сервера (http://localhost:5173)
npm run dev

# Сборка продакшн бандла
npm run build

# Предпросмотр сборки
npm run preview
```

### 9. Полный локальный стек

```bash
# Терминал 1: Supabase (Postgres + Auth + Functions + Studio)
supabase start

# Терминал 2: Edge Functions (dev mode)
supabase functions serve --project-ref local

# Терминал 3: Frontend (Vite dev server)
cd frontend && npm run dev
```

**Доступные сервисы:**
| Сервис | URL | Порт |
|--------|-----|------|
| Frontend (TMA) | http://localhost:5173 | 5173 |
| Supabase Studio | http://localhost:54323 | 54323 |
| Postgres DB | localhost:54322 | 54322 |
| Auth | http://localhost:9999 | 9999 |
| Storage | http://localhost:54321 | 54321 |
| Edge Functions | http://localhost:54321/functions/v1 | 54321 |
| GraphQL | http://localhost:54321/v1/graphql | 54321 |

### 10. Настройка Telegram Bot для локальной разработки

```bash
# 1. Открыть @BotFather в Telegram
# 2. Создать нового бота: /newbot
# 3. Получить токен бота
# 4. Создать Mini App: /newapp
# 5. Указать URL: https://<ngrok-url>/ (см. ниже)

# Для локального доступа через Telegram:
# Установить ngrok:
# winget install ngrok

# Проброс локального порта:
ngrok http 5173

# Скопировать https URL в BotFather
# Пример: https://abc123.ngrok.io
```

### 11. Seed данные (сценарии Aim Lab / KovaaK)

```sql
-- Запустить в Supabase SQL Editor или через psql
\i supabase/seed.sql

-- Проверка
SELECT count(*) FROM aim_lab_scenarios;   -- ~200 записей
SELECT count(*) FROM kovaak_scenarios;     -- ~150 записей
```

### 12. Полезные команды

```bash
# Полный цикл разработки
supabase start          # Запуск Supabase
supabase functions serve --project-ref local  # Dev mode функций
npm run dev             # Запуск фронтенда

# Деплой на продакшн
supabase link --project-ref <prod-project-ref>
supabase db push        # Миграции
supabase functions deploy --all  # Все функции
vercel deploy --prod    # Фронтенд

# Отладка
supabase logs           # Логи всех сервисов
supabase db reset       # Сброс БД
supabase status         # Проверка статуса сервисов
```

### 13. Troubleshooting

```bash
# Если Edge Functions не запускаются
supabase stop
supabase start
supabase functions deploy --all --project-ref local

# Если Postgres не подключается
supabase stop
supabase start

# Если Frontend не видит API
# Проверить CORS в supabase/config.toml
# Добавить:
# [api]
# enabled = true
# cors_origins = ["http://localhost:5173"]

# Если Telegram Mini App не открывается
# Проверить URL в BotFather
# URL должен быть HTTPS
# Использовать ngrok для локального теста
```
