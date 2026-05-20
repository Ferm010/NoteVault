# Архитектура и технологический стек проекта NexusForge

## 1. Общая информация о проекте

**NexusForge** — Android-приложение для поиска, управления и создания модпаков Minecraft через API Modrinth с авторизацией пользователя, системой избранных проектов, пользовательскими шаблонами и генерацией `.mrpack` файлов.

---

## 2. Архитектурный паттерн: MVVM (Model-ViewModel-View)

Проект реализован по классической архитектуре **MVVM** с чётким разделением ответственности между слоями, что является отраслевым стандартом для Android-разработки, рекомендованным Google.

### 2.1. Слои приложения

| Слой | Пакет | Отвечает за | Файлов |
|------|-------|-------------|--------|
| **View (Представление)** | `frontend.*` | Jetpack Compose composables, отображение UI-состояния, обработка пользовательских жестов | ~20 файлов |
| **ViewModel** | `viewmodels.*` | Хранение и управление состоянием экрана, оркестровка бизнес-логики, взаимодействие с Repository | ~12 файлов |
| **Repository (Репозиторий)** | `repository.*`, `backend/` | Единственный источник данных для ViewModel, абстракция над Firestore и API | 3 файла |
| **Data (Модели данных)** | `data.*` | DTO-классы для сериализации/десериализации, модели домена | 4 файла |
| **Network (Сеть)** | `network.*` | Определение HTTP-эндпоинтов, конфигурация клиента | 1 файл |
| **Backend (Бэкенд)** | `backend/` | Навигация, авторизация, утилиты безопасности | ~8 файлов |
| **Utils (Утилиты)** | `utils.*` | Генерация .mrpack, санитизация ввода, debouncing | 4 файла |

### 2.2. Поток данных в MVVM

```
Пользователь взаимодействует с UI
        │
        ▼
┌───────────────────────┐
│   View (Compose)       │  ← collectAsState() → автоматическая перерисовка
│   frontend/*.kt        │     AsyncImage, LazyColumn, Material3 компоненты
└───────────┬───────────┘
            │ вызывает методы
            ▼
┌───────────────────────┐
│   ViewModel            │  ← StateFlow/mutableStateOf → публикация изменений
│   viewmodels/*.kt      │     viewModelScope.launch { } → безопасные корутины
└───────────┬───────────┘
            │ вызывает методы репозиториев
            ▼
┌───────────────────────┐
│   Repository           │  ← .await() → мост между Firebase Task и suspend
│   repository/*.kt      │     callbackFlow { } → реальное время через Firestore
└───────────┬───────────┘
            │
    ┌───────┴────────┐
    ▼                ▼
Firestore         Modrinth API
(облачная БД)     (Retrofit + kotlinx-serialization)
```

### 2.3. Конкретный пример потока данных: добавление в избранное

**View** (`ProjectDetailsPage.kt`):
```kotlin
// Composable проверяет состояние и вызывает callback ViewModel
val isFavorite = favoriteProjects.any { it.projectId == project.projectId }
Button(onClick = { favoritesViewModel.toggleFavorite(project) }) {
    Icon(if (isFavorite) Icons.Filled.Bookmark else Icons.Outlined.Bookmark)
}
```

**ViewModel** (`FavoritesViewModel.kt`):
```kotlin
fun toggleFavorite(project: ModrinthProject) {
    viewModelScope.launch {  // Корутина привязана к жизненному циклу ViewModel
        isLoading = true
        val result = if (isFavorite(project.projectId)) {
            repository.removeFromFavorites(project.projectId)
        } else {
            repository.addToFavorites(FavoriteProject.fromModrinth(project, userId!!))
        }
        result.onFailure { errorMessage = it.message }
        isLoading = false
    }
}
```

**Repository** (`FirestoreRepository.kt`):
```kotlin
suspend fun addToFavorites(project: FavoriteProject): Result<Unit> {
    val userId = getUserId() ?: return Result.failure(Exception("Не авторизован"))
    return try {
        firestore.collection("users").document(userId)
            .collection("favorites")
            .document(project.projectId)  // Doc ID = project_id для идемпотентности
            .set(projectData).await()     // await() — мост Firebase Task → suspend
        Result.success(Unit)
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

### 2.4. Преимущества выбранной архитектуры

1. **Тестируемость** — ViewModel принимает Repository через конструктор, что позволяет подменять репозиторий моком при unit-тестировании
2. **Отсутствие утечек памяти** — `onCleared()` в FavoritesViewModel вызывает `repository.clearAllListeners()`, останавливая Firestore snapshot-слушатели при уничтожении ViewModel
3. **Жизненный цикл-aware** — корутины работают в `viewModelScope`, автоматически отменяясь при уничтожении ViewModel
4. **Реактивность** — StateFlow обеспечивает автоматическую перерисовку Compose UI при изменении данных без ручного обновления

---

## 3. Навигация: Navigation3 с типобезопасными маршрутами

### 3.1. Архитектура навигации

Проект использует **AndroidX Navigation3** — следующее поколение навигации для Android, заменяющее устаревший NavGraph на сериализуемые типобезопасные маршруты.

```kotlin
@Serializable
sealed interface Destination : NavKey {
    // Auth экраны
    @Serializable data object RegPage : Destination
    @Serializable data object EulaPage : Destination
    @Serializable data class AuthPassPage(val email: String = "") : Destination
    
    // Основные экраны
    @Serializable data object MainMenu : Destination
    @Serializable data object FavoritePage : Destination
    
    // Экран с параметрами маршрута
    @Serializable data class ProjectDetailsPage(val projectId: String) : Destination
    @Serializable data class ModpackEditorPage(val modpackId: String?) : Destination
}
```

### 3.2. Двухуровневая стек-архитектура

```
Главный стек навигации (Main backStack)
├── RegPage → EulaPage → AuthPassPage → MainMenu
│                                                    │
│                          ┌─────────────────────────┘
│                          ▼ (nested tabBackStack)
└──── MainMenu ──────────────────────────────────────┐
    │  Local NavDisplay                              │
    │  ├─ tabBackStack[0] = MainMenu (поиск)         │
    │  ├─ tabBackStack[1] = FavoritePage             │
    │  ├─ tabBackStack[2] = ProfilePage              │
    │  └─ tabBackStack[3] = SettingsPage             │
    │                                                  │
    └──── Pushed screens (поверх табов):              │
         ├── ProjectDetailsPage(projectId)            │
         ├── ModpackEditorPage(modpackId?)            │
         ├── CreateModpackPage()                      │
         └── ...                                      │
```

### 3.3. Почему Navigation3 вместо NavGraph?

| Критерий | NavGraph (старый) | Navigation3 (новый) |
|----------|-------------------|---------------------|
| **Типобезопасность** | Строковые маршруты — ошибки выявляются только на рантайме | Sealed interface + компилятор Kotlin проверяет все маршруты на этапе компиляции |
| **Параметры маршрутов** | `navArgs` делегат, ручная десериализация аргументов | `@Serializable data class` — параметры встроены в тип, нулевой boilerplate |
| **Вложенная навигация** | Nested NavGraphs с XML-подобным описанием | `tabBackStack.clear() + push()` — программное управление стеком табов |
| **Deep Linking** | Требует отдельной конфигурации | Автоматически работает благодаря `@Serializable` |
| **Восстановление при смерти процесса** | Ручная сериализация аргументов | Авто-сериализация через kotlinx-serialization |

---

## 4. UI: Jetpack Compose + Material3

### 4.1. Почему Compose?

Jetpack Compose — декларативный UI-фреймворк от Google, заменивший императивную систему View/XML. В NexusForge используется **Compose BOM (Bill of Materials)** версии `2024.09.00`, который гарантирует совместимость всех Compose-библиотек между собой.

**Ключевые компоненты в проекте:**
- `LazyColumn` — рецитклируемые списки для модов и проектов
- `AsyncImage` (Coil) — асинхронная загрузка изображений с кэшированием
- `Scaffold` + `BottomNavigation` — Material3 шаблон основного экрана
- `AlertDialog`, `CircularProgressIndicator`, `LinearProgressIndicator` — стандартные Material3 компоненты
- `MaterialTheme` — динамические цвета (поддержка Android 12+ dynamic colors)

### 4.2. Управление темой

Тема управляется через **ThemeViewModel** с сохранением предпочтений пользователя между сеансами (SharedPreferences/DataStore). Поддерживается три режима: системная, светлая и тёмная тема.

---

## 5. Работа с сетью: Retrofit + kotlinx-serialization

### 5.1. Конфигурация HTTP-клиента

```kotlin
// ModrinthApiService.kt
private val okHttpClient = OkHttpClient.Builder()
    .connectTimeout(30, TimeUnit.SECONDS)   // Таймаут подключения — защита от зависаний
    .readTimeout(30, TimeUnit.SECONDS)      // Таймаут чтения — данные могут идти долго
    .writeTimeout(30, TimeUnit.SECONDS)     // Таймаут записи — для больших запросов
    .addInterceptor { chain ->
        chain.request().newBuilder()
            .addHeader("User-Agent", "NexusForge/1.0.0")  // Идентификация приложения
            .build()
    }

private val json = Json {
    ignoreUnknownKeys = true      // Устойчивость: API добавил поле — не крашимся
    coerceInputValues = true       // Null → значения по умолчанию из data class
}

private val retrofit = Retrofit.Builder()
    .baseUrl("https://api.modrinth.com/v2/")
    .client(okHttpClient)
    .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
    .build()
```

### 5.2. Определение API-эндпоинтов

```kotlin
interface ModrinthApiService {
    @GET("search")
    suspend fun searchProjects(
        @Query("query") query: String,
        @Query("facets") facets: String? = null,
        @Query("limit") limit: Int = 20,
        @Query("offset") offset: Int = 0,
        @Query("index") index: String = "relevance"
    ): ModrinthSearchResponse
    
    @GET("project/{id}")
    suspend fun getProject(@Path("id") projectId: String): ModrinthProject
}
```

### 5.3. Сериализация данных

Модели используют `@SerialName` для маппинга snake_case из API на camelCase в Kotlin:

```kotlin
@Serializable
data class ModrinthProject(
    @SerialName("project_id") val projectId: String = "",
    @SerialName("icon_url")   val iconUrl: String? = null,
    // ...
) {
    // Вычисляемое свойство — fallback для несовпадающих полей API
    val actualProjectId: String get() = projectId.ifEmpty { id }
}
```

### 5.4. Преимущества kotlinx-serialization над Gson/Moshi

| Критерий | Gson | Moshi | kotlinx-serialization |
|----------|------|-------|----------------------|
| **Производительность** | Reflection (медленно) | Annotation processing | Compile-time codegen (быстро, как ручной код) |
| **Обработка ошибок** | Тихие ошибки маппинга на рантайме | Частичная валидация | Ошибки компиляции при несовпадении типов |
| **Размер APK** | +~200KB (reflection) | +~150KB | +~30KB (минимальный) |
| **Kotlin-first** | Нет (Java-ориентирован) | Частично | Полная поддержка sealed classes, data classes, nullability |

---

## 6. Coroutines + Flow: реактивная работа с данными

### 6.1. Паттерн 1: Реалтайм-поток из Firestore (callbackFlow → stateIn)

```kotlin
// Repository возвращает cold Flow, привязанный к Firestore snapshot listener
fun getFavorites(): Flow<List<FavoriteProject>> = callbackFlow {
    val userId = getUserId() ?: return@callbackFlow
    val query = firestore.collection("users").document(userId)
        .collection("favorites")
        .orderBy("addedAt", Query.Direction.DESCENDING)
    
    val listener = query.addSnapshotListener { snapshot, error ->
        if (error != null) close(error); return@addSnapshotListener
        val favorites = snapshot?.documents?.mapNotNull { 
            it.toObject(FavoriteProject::class.java) 
        } ?: emptyList()
        trySend(favorites)  // Асинхронная отправка в Flow
    }
    
    awaitClose { listener.remove() }  // Очистка слушателя при завершении потока
}

// ViewModel.hot-connects поток с таймаутом остановки
val favoriteProjects: StateFlow<List<FavoriteProject>> = repository.getFavorites()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),  // Стоп через 5с после отключения UI
        initialValue = emptyList()
    )
```

**Ключевой момент**: `WhileSubscribed(5000)` — оптимизация потребления трафика и батареи. Когда пользователь уходит на другой экран, слушатель Firestore отключается через 5 секунд, а не сразу. Это позволяет избежать мерцания UI при быстром переключении между экранами (например, возврат к списку избранного).

### 6.2. Паттерн 2: Однократные вызовы в viewModelScope

```kotlin
fun addFavorite(project: ModrinthProject) {
    viewModelScope.launch {  // Корутина автоматически отменяется при onDestroy ViewModel
        isLoading = true
        val result = repository.addToFavorites(...)
        result.onFailure { errorMessage = it.message }  // Обработка через Result<T>
        isLoading = false
    }
}
```

### 6.3. Паттерн 3: Мост Firebase Task → Coroutines

```kotlin
// kotlinx-coroutines-play-services предоставляет .await() extension
firestore.collection("users").document(userId).get().await()
// Преобразует callback-based Firebase Task<T> в suspending function
```

---

## 7. Безопасность и санитизация ввода

### 7.1. Защита от path traversal (обход директорий)

```kotlin
fun sanitizeModpackName(name: String): String {
    var sanitized = name.replace(Regex("[<>:\"|?*\\/]"), "")   // Удаление файловых спецсимволов
    sanitized = sanitized.replace(Regex("\\.\\.[\\\\/]"), "")   // Блокировка ../ и ..\
    return sanitized.trim()
}
```

### 7.2. Верификация целостности файлов (двойной хеш)

При генерации `.mrpack` файла каждый загруженный мод проверяется по **двум алгоритмам**: SHA-1 И SHA-512 одновременно. При несовпадении любого из хешей файл удаляется и пропускается. Хеш-значения НЕ логируются для защиты от утечки метаданных.

### 7.3. Изоляция данных пользователей в Firestore

Все данные хранятся в подколлекциях `users/{uid}/...`. Это означает, что:
- Каждый пользователь видит ТОЛЬКО свои данные (путь к коллекции содержит UID)
- Нет необходимости в клиентской фильтрации — структура путей обеспечивает изоляцию
- Правила безопасности Firestore дополнительно защищают на серверном уровне

---

## 8. Технологический стек: сводная таблица

| Компонент | Выбор | Версия | Обоснование |
|-----------|-------|--------|-------------|
| **Язык** | Kotlin | 2.1.0 | Основной язык Android-разработки, null-safety, coroutines, sealed interfaces |
| **UI Toolkit** | Jetpack Compose | BOM 2024.09.00 | Современный декларативный фреймворк от Google, замена XML+Views |
| **Компоновка UI** | Material3 Design | BOM | Стандарт Material Design 3 с динамическими цветами и адаптивным дизайном |
| **Навигация** | Navigation3 | 1.0.0 | Типобезопасные сериализуемые маршруты, замена устаревшего NavGraph |
| **Архитектура** | MVVM + Repository | — | Официальная рекомендация Google Architecture Guide |
| **Асинхронность** | Kotlin Coroutines + Flow | 1.9.0 | Стандарт асинхронной разработки в Kotlin, лучше LiveData/RxJava по простоте |
| **HTTP-клиент** | Retrofit | 2.9.0 | Industry-standard: типобезопасные API описания, автоматическая сериализация |
| **Сериализация** | kotlinx-serialization | 1.8.1 | Compile-time codegen, нулевые reflection-накладные расходы, Kotlin-first |
| **HTTP Engine** | OkHttp | 4.12.0 | Connection pooling, transparent GZIP, таймауты — основа Retrofit |
| **Облачная БД** | Google Firestore | BOM 33.7.0 | Real-time sync через snapshots, noSQL, авто-масштабирование, интеграция с Firebase Auth |
| **Авторизация** | Firebase Auth + Credential Manager | BOM 33.7.0 / 1.3.0 | Email/Password + Google Sign-In, platform-agnostic API |
| **Загрузка изображений** | Coil | 2.5.0 | Kotlin-first, coroutine-native, zero-boilerplate для Compose |
| **Криптография хешей** | java.security.MessageDigest | stdlib | SHA-1 + SHA-512 верификация файлов из стандартной библиотеки JDK |

---

## 9. Структура модулей проекта

```
NexusForge/
├── build.gradle.kts              ← Плагин: android.application, kotlin.android, kotlin.compose
├── settings.gradle.kts           ← Module inclusion + version catalog (libs.versions.toml)
├── gradle/libs.versions.toml     ← Версионный каталог всех зависимостей (single source of truth)
└── app/
    ├── build.gradle.kts          ← namespace, compileSdk=36, minSdk=26, signing config
    └── src/main/java/com/ferm/nexusforge/
        ├── MainActivity.kt                    ← Точка входа: тема + навигация
        ├── NexusForgeApp.kt                   ← Application class (Firebase init)
        ├── backend/                           ← Навигация, Auth-репозиторий, утилиты
        │   ├── AppNavigation.kt               ← Главный хаб навигации (~550 строк)
        │   ├── NavigationData.kt              ← Sealed Destination + NavKey (~27 экранов)
        │   ├── AuthRepository.kt              ← Firebase Auth wrapper
        │   └── ...
        ├── frontend/                          ← Jetpack Compose screens
        │   ├── RegPageScreen.kt               ← Регистрация (email + Google Sign-In)
        │   ├── mainmenu/MainMenuPage.kt       ← Поиск модов Modrinth
        │   ├── FavoritePage.kt                ← Избранное с edit/delete
        │   ├── projectdetails/ProjectDetailsPage.kt
        │   └── ... (~20 экранов)
        ├── viewmodels/                        ← ViewModel для каждого экрана
        │   ├── MainMenuViewModel.kt           ← Поиск, список модов
        │   ├── FavoritesViewModel.kt          ← Real-time избранное (Flow)
        │   ├── CustomModpacksViewModel.kt     ← CRUD модпаков
        │   └── ... (~12 ViewModels)
        ├── repository/                        ← Data layer abstraction
        │   ├── FirestoreRepository.kt         ← ~484 строки: все операции с БД
        │   └── GoogleDriveRepository.kt       ← Загрузка ZIP на Google Drive
        ├── network/                           ← API definitions
        │   └── ModrinthApiService.kt          ← Retrofit interface + singleton client
        ├── data/                              ← Data models / DTOs
        │   ├── ModrinthProject.kt             ← Модели ответов Modrinth API
        │   ├── FirestoreModels.kt             ← Модели для Firestore документов
        │   ├── ModpackMod.kt                  ← Модификация внутри модпака
        │   └── ModpackTemplate.kt             ← Шаблоны генерации модпаков
        ├── utils/                             ← Утилиты
        │   ├── MrpackGenerator.kt             ← Генерация .mrpack (ZIP + modrinth.index.json)
        │   ├── InputSanitizer.kt              ← Санитизация пользовательского ввода
        │   └── DebounceUtils.kt               ← Debounce для поисковых запросов
        └── ui/theme/                          ← Material3 тема
            ├── Color.kt                       ← Палитра цветов
            ├── Theme.kt                       ← Light/Dark theme + dynamic colors
            └── Type.kt                        ← Типографика
```
