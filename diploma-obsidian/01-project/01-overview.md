# NexusForge — Обзор проекта

## Общая информация

**NexusForge** — Android-приложение для создания и управления модпаками Minecraft. Приложение позволяет:
- Искать моды через Modrinth API
- Создавать кастомные модпаки
- Работать с шаблонами
- Сохранять модпаки на Google Drive или локально
- Генерировать `.mrpack` файлы

## Стек технологий

| Технология | Версия / Библиотека | Назначение |
|---|---|---|
| **Kotlin** | JVM Target 11 | Основной язык разработки |
| **Jetpack Compose** | BOM (latest) | Современный UI toolkit |
| **Navigation3** | androidx.navigation3 | Навигация между экранами |
| **Firebase Auth** | Firebase BoM | Авторизация (Email, Google) |
| **Firestore** | Firebase BoM | Облачное хранилище данных пользователей |
| **Retrofit** | 2.9.0 | HTTP-клиент для Modrinth API |
| **OkHttp** | 4.12.0 | Базовый HTTP-клиент |
| **kotlinx.serialization** | — | JSON сериализация/десериализация |
| **Coil** | 2.5.0 | Загрузка изображений |
| **Google Drive API** | v3-rev20240123-2.0.0 | Сохранение модпаков в облако |
| **Coroutines** | kotlinx.coroutines.play.services | Асинхронные операции с Firebase |

## Архитектура приложения

```
com.ferm.nexusforge/
├── MainActivity.kt              # Точка входа, инициализация
├── backend/                     # Бизнес-логика и навигация
│   ├── AppNavigation.kt         # Главная навигация (NavGraph)
│   ├── NavigationData.kt        # sealed interface Destination
│   ├── LocaleHelper.kt          # Управление языком
│   ├── SecurityCheck.kt         | Проверка целостности приложения
│   ├── EulaParser.kt            # Парсер EULA
│   └── NetworkUtils.kt          # Утилиты сети
├── frontend/                    # UI экраны (Composables)
│   ├── mainmenu/                # Главный экран
│   ├── components/              | Переиспользуемые компоненты
│   ├── projectdetails/          | Детали проекта Modrinth
│   └── ...                      | Остальные экраны
├── viewmodels/                  # ViewModel'и для каждого экрана
├── repository/                  # Репозитории работы с данными
│   ├── FirestoreRepository.kt   # Работа с Firestore
│   └── GoogleDriveRepository.kt # Работа с Google Drive
├── data/                        | Модели данных
├── network/                     | API-сервисы (Modrinth)
├── utils/                       | Утилиты
└── ui/theme/                    | Тема приложения
```

## Ключевые экраны и их назначение

| Экран | Описание |
|---|---|
| **RegPage** | Страница регистрации (выбор Email или Google) |
| **AuthPassPage** | Вход по паролю |
| **MainMenu** | Главный экран с поиском модов Modrinth |
| **FavoritePage** | Избранные проекты и модпаки |
| **ProfilePage** | Профиль пользователя |
| **SettingsPage** | Настройки (тема, язык, FAQ) |
| **CreateModpackPage** | Создание нового модпака |
| **GenerateModpackPage** | Генерация `.mrpack` / сохранение на Drive |
| **ProjectDetailsPage** | Детали мода с Modrinth API |

## Firebase интеграция

- **Authentication**: Email/Password + Google Sign-In через Credential Manager
- **Firestore**: Хранение пользовательских данных (профиль, избранное, модпаки)
- **Crashlytics**: Сбор краш-репортов
- **Analytics**: Аналитика использования

## Modrinth API

Базовый URL: `https://api.modrinth.com/v2/`

Эндпоинты:
- `GET search` — Поиск проектов
- `GET tag/game_version` — Список версий игры
- `GET project/{id}/version` — Версии проекта
- `GET project/{id}` — Информация о проекте
- `GET project/{id}/dependencies` — Зависимости проекта

## Безопасность

- Проверка целостности приложения (Signature verification)
- Проверка безопасности устройства (root detection, отключена опционально)
- Пароли для удаления аккаунта
- Ключи храним в `local.properties`, не коммитятся
