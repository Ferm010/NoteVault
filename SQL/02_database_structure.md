# База данных Google Firestore: структура, проектирование и работа

## 1. Обзор базы данных

**Google Cloud Firestore** — noSQL документо-ориентированная база данных в реальном времени от Google, входящая в экосистему Firebase. В проекте NexusForge используется как основное хранилище пользовательских данных: профили, избранное, кастомные модпаки и шаблоны.

### Почему Firestore, а не локальная БД (Room/SQLite)?

| Критерий | Room + SQLite | Google Firestore |
|----------|---------------|------------------|
| **Синхронизация** | Нет — данные только на устройстве | Real-time sync между устройствами через облако |
| **Мультидевайс** | Невозможно без собственного бэкенда | Встроено — один пользователь = одни данные на всех устройствах |
| **Авторизация** | Ручная реализация | Интеграция с Firebase Auth — UID пользователя в пути к коллекции |
| **Реактивность** | LiveData/Flow нужно реализовывать вручную | `addSnapshotListener` — автоматическое обновление при изменении данных на сервере |
| **Масштабируемость** | Ограничена устройством | Авто-масштабирование Google Cloud |
| **Оффлайн-режим** | Встроен (данные локально) | Встроен (кэш + очередь операций при потере связи) |
| **Стоимость** | Бесплатно | Free tier 1GB storage, далее pay-as-you-go |

Для NexusForge Firestore — обязательный выбор, так как приложение требует:
- Сохранения избранного между устройствами одного пользователя
- Синхронизации кастомных модпаков и шаблонов
- Персонализации контента на основе учётной записи

---

## 2. Структура данных (Schema Design)

### 2.1. Общая схема коллекций

```
Firestore Root Database
│
└── users/                          ← Коллекция пользователей (top-level)
    │
    ├── {userId}                    ← Документ = UID из Firebase Auth
    │   │
    │   ├── email: String           ← Поля профиля (простые поля документа)
    │   ├── displayName: String     ← Отображаемое имя пользователя
    │   ├── createdAt: Timestamp    ← Дата регистрации (serverTimestamp)
    │   └── updatedAt: Timestamp    ← Последнее обновление профиля
    │   │
    │   ├── favorites/              ← Подколлекция: избранные проекты Modrinth
    │   │   ├── {projectId}/        ← Doc ID = project_id (уникальный идентификатор мода)
    │   │   │   ├── projectId: String
    │   │   │   ├── title: String
    │   │   │   ├── description: String
    │   │   │   ├── author: String
    │   │   │   ├── iconUrl: String?
    │   │   │   ├── downloads: Int
    │   │   │   ├── categories: List<String>
    │   │   │   ├── versions: List<String>
    │   │   │   ├── projectType: String ("mod" | "modpack")
    │   │   │   └── addedAt: Timestamp
    │   │   │
    │   │   └── {anotherProjectId}/ ← Каждый проект = отдельный документ
    │   │       └── ... (такая же структура)
    │   │
    │   ├── custom_modpacks/        ← Подколлекция: пользовательские модпаки
    │   │   ├── {auto-generated-id}/← Doc ID = автоматически сгенерированный Firestore ID
    │   │   │   ├── name: String
    │   │   │   ├── description: String
    │   │   │   ├── minecraftVersion: String  ← Версия Minecraft (1.20.4 и т.д.)
    │   │   │   ├── modLoader: String         ← Forge/Fabric/Quilt/NeoForge
    │   │   │   ├── iconUrl: String?
    │   │   │   ├── mods: List<ModReference>  ← Массив модов внутри модпака
    │   │   │   ├── createdAt: Timestamp
    │   │   │   └── updatedAt: Timestamp
    │   │   │
    │   │   └── {another-modpack-id}/
    │   │       └── ... (такая же структура)
    │   │
    │   └── templates/              ← Подколлекция: шаблоны генерации модпаков
        ├── {templateId}/           ← Doc ID = идентификатор шаблона
        │   ├── name: String
        │   ├── description: String
        │   ├── mcVersion: String
        │   ├── loader: String
        │   ├── mods: List<TemplateMod>
        │   ├── createdAt: Timestamp
        │   └── userId: String      ← Ссылка на автора (для будущей кросс-пользовательской логики)
        │
        └── {another-template-id}/
            └── ... (такая же структура)
```

### 2.2. ER-диаграмма (логическая модель)

```
┌──────────────┐         ┌─────────────────────────────┐
│   users      │         │  favorites/                 │
│  {userId}    │1     N  │  {projectId}/               │
│──────────────│─────────│─────────────────────────────│
│ email        │         │ projectId (PK = doc ID)     │
│ displayName  │         │ title                       │
│ createdAt    │         │ description                 │
│ updatedAt    │         │ author                      │
└──────────────┘         │ iconUrl                     │
                         │ downloads                   │
                         │ categories[]                │
                         └─────────────────────────────┘

┌──────────────┐         ┌─────────────────────────────┐
│   users      │1     N  │  custom_modpacks/           │
│  {userId}    │─────────│  {docId}/                   │
│──────────────│         │─────────────────────────────│
│              │         │ name                        │
│              │         │ description                 │
└──────────────┘         │ minecraftVersion            │
                         │ modLoader                   │
                         └─────────────────────────────┘
                                  │
                                  ▼
                          ┌─────────────────────────────┐
                          │  mods[] (embedded document)  │
                          │─────────────────────────────│
                          │ projectId: String           │
                          │ name: String                │
                          │ version: String             │
                          │ downloadUrl: String         │
                          │ iconUrl: String?            │
                          │ fileName: String            │
                          │ fileSize: Long              │
                          │ sha1: String                │
                          │ sha512: String              │
                          └─────────────────────────────┘

┌──────────────┐         ┌─────────────────────────────┐
│   users      │1     N  │  templates/                 │
│  {userId}    │─────────│  {templateId}/              │
│──────────────│         │─────────────────────────────│
│              │         │ name                        │
│              │         │ description                 │
└──────────────┘         │ mcVersion                   │
                         │ loader                      │
                         └─────────────────────────────┘
```

---

## 3. Детальное описание коллекций

### 3.1. Коллекция `users/{userId}` — Профили пользователей

**Назначение**: Хранение базовой информации аутентифицированного пользователя.

| Поле | Тип | Источник | Описание |
|------|-----|----------|----------|
| `email` | String | Firebase Auth | Email пользователя |
| `displayName` | String | Firebase Auth / обновление | Отображаемое имя |
| `createdAt` | Timestamp | `serverTimestamp()` | Момент регистрации (серверное время) |
| `updatedAt` | Timestamp | `serverTimestamp()` | Время последнего обновления профиля |

**Логика создания**: Документ создаётся автоматически при первой успешной регистрации через `createUserProfile()`. Проверка существования — `checkUserProfileExists()`, которая делает один read-operation для экономии ресурсов.

### 3.2. Подколлекция `favorites/{projectId}` — Избранные проекты

**Назначение**: Сохранение избранных проектов Modrinth с полными метаданными.

| Поле | Тип | Описание |
|------|-----|----------|
| `projectId` (Doc ID) | String | Уникальный ID проекта на Modrinth — используется как идентификатор документа для идемпотентных upsert-операций |
| `title` | String | Название мода/модпака |
| `description` | String | Описание с Modrinth |
| `author` | String | Автор проекта |
| `iconUrl` | String? | URL иконки (nullable) |
| `downloads` | Int | Количество скачиваний |
| `categories` | List\<String\> | Категории (library, crafting, combat и т.д.) |
| `versions` | List\<String\> | Поддерживаемые версии Minecraft |
| `projectType` | String | Тип: "mod", "modpack" или "plugin" |
| `addedAt` | Timestamp | Момент добавления в избранное (серверное время) |

**Почему Doc ID = projectId?** Это паттерн **upsert**: при повторном добавлении того же проекта документ просто перезаписывается, а не создаётся дубликат. Одной строкой: `db.collection("favorites").document(projectId).set(data)` — если документ существует, он обновляется; если нет — создаётся.

### 3.3. Подколлекция `custom_modpacks/{docId}` — Кастомные модпаки

**Назначение**: Пользовательские конфигурации модпаков с полным списком модификаций.

| Поле | Тип | Описание |
|------|-----|----------|
| `name` | String | Название модпака (санитизированное через InputSanitizer) |
| `description` | String | Описание |
| `minecraftVersion` | String | Целевая версия Minecraft |
| `modLoader` | String | Тип загрузчика: Forge, Fabric, Quilt, NeoForge |
| `iconUrl` | String? | Иконка модпака |
| `mods` | List\<ModReference\> | Вложенный массив модов (embedded documents) |

**Структура вложенного объекта `ModReference`:**
```kotlin
data class ModReference(
    val projectId: String,
    val name: String,
    val version: String,
    val downloadUrl: String,
    val iconUrl: String?,
    val fileName: String,
    val fileSize: Long,
    val sha1: String,
    val sha512: String
)
```

**Почему вложенный массив (embedded documents), а не отдельная коллекция?**
- Модпаки — небольшие структуры (обычно 10-50 модов)
- Вложенные данные всегда читаются вместе с родителем (нет запроса к отдельной коллекции)
- Firestore ограничивает размер документа 1MB — для типичного модпака это достаточный запас
- Один read-operation вместо нескольких (оптимизация стоимости и задержки)

### 3.4. Подколлекция `templates/{templateId}` — Шаблоны генерации

**Назначение**: Сохранённые конфигурации для быстрого создания модпаков по шаблону.

| Поле | Тип | Описание |
|------|-----|----------|
| `name` | String | Название шаблона |
| `description` | String | Описание шаблона |
| `mcVersion` | String | Версия Minecraft |
| `loader` | String | Тип загрузчика модов |
| `mods` | List\<TemplateMod\> | Список модов для этого шаблона |
| `createdAt` | Timestamp | Дата создания |
| `userId` | String | ID автора (для будущей кросс-пользовательской логики) |

---

## 4. Реализация работы с Firestore в проекте

### 4.1. FirestoreRepository — центральный слой данных

Файл: `repository/FirestoreRepository.kt` (~484 строки)

#### Мост Firebase → Coroutines

Firebase SDK использует callback-based API (`Task<T>`). Проект интегрирует **kotlinx-coroutines-play-services**, который предоставляет `.await()` extension:

```kotlin
// Было (callback-based):
firestore.collection("users").document(userId).get()
    .addOnSuccessListener { document -> /* обрабатываем */ }
    .addOnFailureListener { exception -> /* обрабатываем ошибку */ }

// Стало (suspend coroutine):
val document = firestore.collection("users").document(userId).get().await()
```

Это позволяет использовать Firestore в `suspend` функциях ViewModels без callback hell.

#### Обработка результатов через sealed class Result

```kotlin
suspend fun addToFavorites(project: FavoriteProject): Result<Unit> {
    val userId = getUserId() ?: return Result.failure(Exception("Пользователь не авторизован"))
    return try {
        firestore.collection("users").document(userId)
            .collection("favorites")
            .document(project.projectId)
            .set(projectData).await()
        Result.success(Unit)  // Успех
    } catch (e: Exception) {
        Result.failure(e)      // Ошибка с причиной
    }
}
```

ViewModel обрабатывает результат через `onSuccess`/`onFailure`, устанавливая UI-состояние (`isLoading = false`, `errorMessage = ...`).

### 4.2. Real-time синхронизация через callbackFlow

#### Механизм работы:

```
Firestore Server                  Repository              ViewModel                 UI (Compose)
     │                                │                       │                          │
     │  ←── document change ─────────│                       │                          │
     │  (snapshot listener triggers)  │                       │                          │
     │                                │       trySend()       │                          │
     │                                ├──────────────────────►│                          │
     │                                │   Flow<List<Fav>>     │                          │
     │                                │                       │    stateIn()             │
     │                                │                       ├──────────────────────────►│
     │                                │                       │   StateFlow<List<Fav>>   │
     │                                │                       │                          │    collectAsState()
     │                                │                       │                          ├─────► UI recomposition
```

#### Код реализации:

```kotlin
fun getFavorites(): Flow<List<FavoriteProject>> = callbackFlow {
    val userId = getUserId() ?: return@callbackFlow
    
    // Query с сортировкой по дате (новые — сверху)
    val query = firestore.collection("users")
        .document(userId)
        .collection("favorites")
        .orderBy("addedAt", Query.Direction.DESCENDING)
    
    // Регистрация слушателя изменений в реальном времени
    val listener = query.addSnapshotListener { snapshot, error ->
        if (error != null) {
            close(error)  // Завершаем поток с ошибкой
            return@addSnapshotListener
        }
        
        // Десериализация документов Firestore → Kotlin objects
        val favorites = snapshot?.documents
            ?.mapNotNull { doc -> doc.toObject(FavoriteProject::class.java) }
            ?: emptyList()
        
        trySend(favorites)  // Асинхронная отправка в Flow (не блокирует поток)
    }
    
    // Очистка слушателя при завершении потока
    awaitClose { listener.remove() }
}
```

### 4.3. Оптимизация потребления ресурсов: WhileSubscribed(5000)

```kotlin
val favoriteProjects: StateFlow<List<FavoriteProject>> = repository.getFavorites()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),  // Ключевая оптимизация!
        initialValue = emptyList()
    )
```

**Как это работает:**
1. Пользователь открывает экран избранного → Compose начинает собирать StateFlow → слушатель Firestore подключается
2. Пользователь уходит на другой экран → последний коллектор отключается
3. Через **5 секунд** (WhileSubscribed timeout) слушатель автоматически отключается
4. При быстром возврате (менее 5 секунд) слушатель не переподключается — данные из кэша

**Почему не 0 секунд?** Мгновенное отключение вызывает мерцание UI при навигации между экранами (например, из избранного → детали мода → обратно в избранное). Задержка в 5 секунд обеспечивает плавный UX без лишних сетевых запросов.

### 4.4. Управление жизненным циклом слушателей

```kotlin
class FavoritesViewModel(
    private val repository: FirestoreRepository = FirestoreRepository()
) : ViewModel() {
    
    // ... StateFlow properties ...
    
    override fun onCleared() {
        super.onCleared()
        repository.clearAllListeners()  // КРИТИЧНО: предотвращает утечку памяти
    }
}
```

Без этого вызова Firestore snapshot-слушатели продолжали бы работать даже после уничтожения ViewModel, что приводит к:
- Утечке памяти (listener удерживает ссылку на ViewModel)
- Лишним сетевым запросам и расходу батареи
- Возможным ошибкам при обращении к уже не существующим данным

---

## 5. Жизненный цикл операции записи

### Пошаговый процесс добавления в избранное:

```
1. Пользователь нажимает кнопку "В избранное" на экране проекта
   │
   ▼
2. UI (Compose) вызывает favoritesViewModel.toggleFavorite(project)
   │
   ▼
3. ViewModel определяет, есть ли уже в избранном через isFavorite(projectId)
   │
   ▼
4. Запускается корутина в viewModelScope:
   ├─ isLoading = true  (UI показывает прогресс-бар)
   ├─ Вызывается repository.addToFavorites(...)
   │   ├─ getUserId() → проверка авторизации
   │   ├─ Firestore: users/{uid}/favorites/{projectId}.set(data).await()
   │   │   └─ .await() преобразует Firebase Task в suspend function
   │   └─ Result.success/failure возвращается в ViewModel
   │
   ▼
5. ViewModel обрабатывает результат:
   ├─ onSuccess → isLoading = false (избранное обновлено автоматически через snapshot listener)
   └─ onFailure → errorMessage = error.message, isLoading = false
   │
   ▼
6. UI автоматически перерисовывается через collectAsState() → иконка меняется на заполненную/пустую
```

**Важный момент**: После успешной записи данные обновляются в UI **автоматически** через snapshot listener — не нужно вручную запрашивать обновление списка. Firestore push-уведомляет всех подписчиков об изменениях.

---

## 6. Безопасность данных на уровне архитектуры

### 6.1. Идентификация по пути (Path-based Isolation)

Все данные пользователей хранятся в подколлекциях, путь к которым содержит UID:

```
users/{uid}/favorites/     ← Только владелец uid может прочитать/записать
users/{uid}/custom_modpacks/
users/{uid}/templates/
```

Это означает, что даже без правил безопасности Firestore (Firebase Security Rules), клиентский код физически не может получить доступ к чужим данным — для этого нужен валидный UID.

### 6.2. Проверка авторизации перед каждой операцией

Каждый метод Repository начинается с проверки:

```kotlin
val userId = getUserId() ?: return Result.failure(Exception("Пользователь не авторизован"))
```

Если сессия Firebase Auth истекла или пользователь вышел, все операции с БД немедленно завершаются ошибкой без обращения к серверу.

### 6.3. Санитизация перед записью

Все пользовательские данные проходят санитизацию перед сохранением в Firestore:

```kotlin
// Модпак — проверка на path traversal и спецсимволы
val safeName = InputSanitizer.sanitizeModpackName(userInput)

// Поиск — ограничение длины и удаление control characters
if (!InputSanitizer.isValidSearchQuery(query)) { /* показать ошибку */ return }
val safeQuery = InputSanitizer.sanitizeSearchQuery(query)
```

### 6.4. serverTimestamp() для консистентности времени

Все временные метки задаются через `FieldValue.serverTimestamp()` — время определяется на сервере Firestore, а не на клиенте. Это гарантирует консистентность при:
- Синхронизации между устройствами (разное системное время)
- Сортировке по дате (orderBy работает корректно с serverTimestamp)
- Оффлайн-режиме (очерёдность операций сохраняется правильно)

---

## 7. Индексация и оптимизация запросов

### 7.1. Используемые индексы

| Запрос | Поле индексации | Тип |
|--------|-----------------|-----|
| `getFavorites()` | `addedAt` (DESC) | Single-field composite index (автоматически создаётся Firestore) |
| `getCustomModpacks()` | Без сортировки | Default (по doc ID) |
| `getTemplates()` | Без сортировки | Default (по doc ID) |

### 7.2. Оптимизация read-операций

1. **Один запрос на подколлекцию**: Все данные для экрана загружаются одним query к одной подколлекции
2. **Flow вместо однократных запросов**: Real-time listener заменяет множественные poll-запросы
3. **WhileSubscribed(5000)**: Экономия трафика при переключении экранов
4. **Doc ID как projectId для favorites**: Идемпотентность без дополнительных проверок на дубликаты

---

## 8. Альтернативные подходы и их анализ

### 8.1. Room (локальная БД) + собственный бэкенд

| Критерий | Firestore | Room + свой бэкенд |
|----------|-----------|---------------------|
| Время разработки | ~неделя на настройку | ~месяц+ (бэкенд + API + sync-логика) |
| Реалтайм-синхронизация | Встроена | Требуется WebSocket/SSE реализация |
| Оффлайн-режим | Встроенный кэш Firestore | Ручная реализация conflict resolution |
| Стоимость инфраструктуры | Free tier → pay-as-you-go | Сервер + база данных + DevOps |
| Масштабируемость | Google Cloud (авто) | Своя инфраструктура |

**Вывод**: Для дипломного проекта с ограниченным бюджетом и сроками Firestore — оптимальный выбор, предоставляющий production-ready облачную БД без написания серверного кода.

### 8.2. SharedPreferences / DataStore для всех данных

| Критерий | Firestore | Только локальное хранилище |
|----------|-----------|---------------------------|
| Мультидевайс | Да | Нет — данные привязаны к устройству |
| Синхронизация в реальном времени | Да | Ручная реализация через polling |
| Надёжность (потеря устройства) | Данные в облаке | Данные потеряны |

**Вывод**: Для приложения с учётными записями и персональными данными — Firestore обязательен. Локальное хранилище используется только для настроек (тема, язык).

---

## 9. Сводная таблица операций по коллекциям

| Коллекция | Create | Read | Update | Delete | Real-time |
|-----------|--------|------|--------|--------|-----------|
| `users/{uid}` | createUserProfile() | checkUserProfileExists() | updateDisplayName() | deleteAccount() | Нет (разовые запросы) |
| `favorites/` | addToFavorites() | getFavorites() | — (upsert по doc ID) | removeFromFavorites() | ✅ callbackFlow |
| `custom_modpacks/` | createCustomModpack() | getCustomModpacks() | updateCustomModpack() | deleteCustomModpack() | ✅ callbackFlow |
| `templates/` | saveTemplate() | getTemplates() | — (upsert по doc ID) | deleteTemplate() | ✅ callbackFlow |

---

## 10. Итоги: почему Firestore подходит для NexusForge

1. **Real-time sync** — пользователь видит изменения в избранном мгновенно, без обновления страницы
2. **Интеграция с Firebase Auth** — UID пользователя естественно встроен в структуру данных
3. **Минимальный код бэкенда** — вся серверная логика (авторизация, масштабируемость, uptime) обрабатывается Google
4. **Оффлайн-режим** — данные кэшируются, операции ставятся в очередь при потере связи
5. **Простота масштабирования** — от 1 до 100 000 пользователей без изменения кода приложения
6. **Free tier** — для дипломного проекта все ограничения бесплатного плана избыточны
