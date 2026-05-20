# Сравнительный анализ технологического стека NexusForge: аргументация выбора

## Введение

Данный документ содержит обоснование выбора каждого компонента технологического стека проекта NexusForge с сравнением альтернатив, что может быть использовано при защите выпускной квалификационной работы (диплома).

---

## 1. Язык программирования: Kotlin vs Java

### Сравнение

| Критерий | **Kotlin** ✅ | Java |
|----------|---------------|------|
| Null-safety | `String?` — компилятор запрещает null-доступ без проверки | Нulleable типы не проверяются → NPE на рантайме |
| Coroutines | Встроенные, лёгкие, структурированные | AsyncTask (устарел), RxJava (тяжёлый) |
| Data classes | `data class` генерирует equals/hashCode/toString автоматически | Ручная реализация или Lombok |
| Sealed interfaces | Ограничивают иерархию — exhaustive when на этапе компиляции | Abstract classes / enum (ограничения) |
| Interoperability с Java | 100% совместимость, можно вызывать Java-код из Kotlin | Нет обратной совместимости с Kotlin |
| Статус поддержки Google | Preferred language для Android (с 2019 года) | Legacy, только поддержка существующего кода |

### Почему Kotlin?

```kotlin
// Kotlin: null-safety на уровне компилятора
val name: String? = getUserDisplayName()  // compiler warns if accessed without check
val safeName = name ?: "Anonymous"         // elvis operator — безопасное разворачивание

// Kotlin: data class — один класс вместо 50 строк boilerplate
data class ModReference(
    val projectId: String,
    val name: String,
    val downloadUrl: String
)
// Автоматически генерируются: equals(), hashCode(), toString(), copy()

// Kotlin: sealed interface — exhaustive when на этапе компиляции
sealed class UiState {
    object Loading : UiState()
    data class Success(val items: List<Item>) : UiState()
    data class Error(val message: String) : UiState()
}
// При добавлении нового подтипа компилятор потребует добавить ветку в when {}
```

**Для диплома**: Kotlin — стандарт де-факто для современной Android-разработки, официально рекомендованный Google с 2019 года. Обеспечивает безопасность null-reference, сокращение boilerplate-кода на ~40% по сравнению с Java и встроенную поддержку асинхронности через coroutines.

---

## 2. UI Toolkit: Jetpack Compose vs XML Views

### Сравнение

| Критерий | **Jetpack Compose** ✅ | XML + Views |
|----------|------------------------|-------------|
| Парадигма | Декларативная (описываем ЧТО показывать) | Императивная (описываем КАК обновлять) |
| Объём кода UI | ~30-50% меньше | Больше boilerplate для простых экранов |
| Preview в IDE | Live preview с данными прямо в Android Studio | Layout Inspector — отдельный инструмент |
| Анимации | AnimatedVisibility, crossfade, tween — встроенные | AnimatorSet + manual management |
| Динамические цвета | Поддержка Material You (Android 12+) | Ограниченная поддержка через AppCompat |
| Состояние | Recomposition при изменении StateFlow/mutableStateOf | findViewById + manual updates |
| Обучение | Новая технология, меньше обучающих материалов | Огромное сообщество, legacy-контент |

### Пример в проекте:

```kotlin
// Compose: декларативный код — читаемо и компактно
@Composable
fun ProjectCard(project: ModrinthProject, onToggleFavorite: () -> Unit) {
    Card(modifier = Modifier.fillMaxWidth().padding(8.dp)) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = project.title, style = MaterialTheme.typography.titleMedium)
            AsyncImage(
                model = project.iconUrl,
                contentDescription = null,
                modifier = Modifier.size(48.dp)
            )
            Row {
                Button(onClick = onToggleFavorite) {
                    Icon(Icons.Filled.Bookmark)
                }
                Text(text = "${project.downloads} скачиваний")
            }
        }
    }
}

// XML Views: потребовал бы отдельный layout-файл + ViewHolder в Adapter
```

### Почему Compose?

1. **Будущее Android UI** — Google объявил Compose основным UI-фреймворком, XML-Views переходят в режим поддержки
2. **Реактивность** — автоматическая перерисовка при изменении состояния через StateFlow/mutableStateOf
3. **Material3** — нативная поддержка динамических цветов Android 12+ (dynamic colors)
4. **Compose BOM** — гарантирует совместимость всех Compose-библиотек

**Для диплома**: Jetpack Compose — современный декларативный UI-фреймворк, заменяющий устаревшую XML-систему. Позволяет создавать адаптивные интерфейсы с меньшим объёмом кода за счёт реактивной модели данных и встроенных анимаций.

---

## 3. Архитектурный паттерн: MVVM vs MVI vs MVP

### Сравнение архитектур

| Критерий | **MVVM** ✅ | MVI | MVP |
|----------|-------------|-----|-----|
| Сложность | Умеренная, понятна большинству разработчиков | Высокая (single source of truth, immutable state) | Низкая-средняя |
| Управление состоянием | StateFlow + mutableStateOf | Reducer + единый State object | Presenter → View callbacks |
| Тестируемость | ViewModel легко тестируется без Android | Отлично тестируется (чистые функции) | Зависит от реализации |
| Boilerplate | Умеренный | Минимальный (но сложная настройка) | Много интерфейсов View/Presenter |
| Сообщество | Стандарт де-факто для Android | Растёт (Redux-подход), но меньше ресурсов | Устаревает, не рекомендуется Google |
| Документация Google | Официальная Architecture Guide рекомендует MVVM | Нет официальной рекомендации | Нет — устарел |

### Почему MVVM?

```kotlin
// ViewModel: изолирует состояние экрана от UI
class MainMenuViewModel(
    private val repository: FirestoreRepository = FirestoreRepository()  // Dependency injection
) : ViewModel() {
    
    private val _searchUiState = MutableStateFlow<SearchUiState>(SearchUiState.Idle)
    val searchUiState: StateFlow<SearchUiState> = _searchUiState
    
    fun search(query: String) {
        viewModelScope.launch {
            _searchUiState.value = SearchUiState.Loading
            try {
                val projects = repository.searchProjects(query)
                _searchUiState.value = SearchUiState.Success(projects)
            } catch (e: Exception) {
                _searchUiState.value = SearchUiState.Error(e.message ?: "Ошибка")
            }
        }
    }
}

// UI: только отображение состояния и вызов callback-ов
@Composable
fun MainMenuScreen(viewModel: MainMenuViewModel = viewModel()) {
    val uiState by viewModel.searchUiState.collectAsState()
    
    when (uiState) {  // Exhaustive when — компилятор проверяет все ветки
        is SearchUiState.Loading -> CircularProgressIndicator()
        is SearchUiState.Success -> ProjectList(uiState.projects)
        is SearchUiState.Error -> Text((uiState as SearchUiState.Error).message)
        is SearchUiState.Idle -> SearchInput(onSearch = viewModel::search)
    }
}
```

**Для диплома**: MVVM (Model-ViewModel-View) — архитектурный паттерн, официально рекомендованный Google Android Architecture Guide. Обеспечивает чёткое разделение ответственности: ViewModel управляет состоянием и бизнес-логикой, View отвечает за отображение, Repository абстрагирует источники данных. Паттерн обеспечивает высокую тестируемость (ViewModel можно тестировать без эмулятора Android) и поддерживаемость кода.

---

## 4. Навигация: Navigation3 vs NavGraph (старый)

### Сравнение

| Критерий | **Navigation3** ✅ | Legacy NavGraph |
|----------|---------------------|-----------------|
| Определение маршрутов | Sealed interface с @Serializable — типобезопасно | Строковые URI — ошибки только на рантайме |
| Параметры навигации | Встроены в data class: `data class DetailsPage(val id: String)` | NavArgs делегат, ручная десериализация |
| Deep linking | Автоматически из @Serializable | Требует отдельной XML-конфигурации |
| Восстановление при смерти процесса | Авто через kotlinx-serialization | Ручное сохранение аргументов |
| Вложенные навигации | Программное управление стеком: `tabBackStack.clear() + push()` | Nested NavGraph в XML — жёсткая структура |

### Почему Navigation3?

```kotlin
// Navigation3: типобезопасная навигация на уровне компилятора
@Serializable
sealed interface Destination : NavKey {
    @Serializable data class ProjectDetailsPage(val projectId: String) : Destination
}

// Использование — IDE автодополнение, compile-time проверка типов
navController.navigate(ProjectDetailsPage(projectId = "abc123"))

// Получение параметра — напрямую из типа, без делегатов
composable<ProjectDetailsPage> { backStackEntry ->
    val projectId = backStackEntry.toDestination<ProjectDetailsPage>().projectId
    ProjectDetailsScreen(projectId = projectId)
}
```

**Для диплома**: Navigation3 — следующее поколение навигации в Android экосистеме, заменяющее устаревший NavGraph. Обеспечивает типобезопасность на уровне компилятора через sealed interface и Kotlin-сериализацию, что исключает ошибки с несуществующими маршрутами или несовпадением типов параметров — проблемы, характерные для строковой навигации.

---

## 5. HTTP-клиент: Retrofit vs OkHttp напрямую vs HttpURLConnection

### Сравнение

| Критерий | **Retrofit** ✅ | OkHttp напрямую | HttpURLConnection |
|----------|-----------------|-----------------|-------------------|
| API описания | Аннотации @GET, @POST — декларативно | Методы Call.execute() — императивно | Ручная настройка Connection |
| Сериализация | Плагин converter (Gson, kotlinx-serialization) | Ручной парсинг ResponseBody | Ручной парсинг InputStream |
| Корутины | `suspend fun` — нативная интеграция | Callback или обёртка через RxJava/Coroutines | Нет встроенной поддержки |
| Базовый URL | Настраивается один раз в Retrofit.Builder | Нужно указывать полный URL в каждом запросе | То же самое |
| Таймауты | Через OkHttp client config | Ручная настройка каждого call | Ручная настройка |

### Почему Retrofit?

```kotlin
// Retrofit: одно описание — все эндпоинты готовы
interface ModrinthApiService {
    @GET("search")
    suspend fun searchProjects(
        @Query("query") query: String,
        @Query("limit") limit: Int = 20
    ): ModrinthSearchResponse
    
    @GET("project/{id}")
    suspend fun getProject(@Path("id") projectId: String): ModrinthProject
}

// Использование — типобезопасно, с авто-сериализацией
val projects = ModrinthApi.retrofitService.searchProjects(query = "fabric", limit = 10)
```

**Для диплома**: Retrofit — industry-standard HTTP-клиент для Android от Square. Позволяет описывать API-эндпоинты через аннотации с автоматической сериализацией ответов в Kotlin data class. Интеграция с корутинами через `suspend fun` обеспечивает чистый асинхронный код без callback-hell.

---

## 6. Сериализация: kotlinx-serialization vs Gson vs Moshi

### Сравнение

| Критерий | **kotlinx-serialization** ✅ | Gson | Moshi |
|----------|------------------------------|------|-------|
| Механизм | Compile-time codegen (KSP/KAPT) | Reflection | Annotation processing |
| Производительность | Как ручной код (максимальная) | Reflection overhead (~20-30% медленнее) | Хорошо, но reflection для некоторых типов |
| Обработка ошибок | Ошибки на этапе компиляции | Тихие ошибки маппинга на рантайме | Частичная валидация |
| Размер APK | +~30KB | +~200KB (reflection) | +~150KB |
| Kotlin-first | Полная поддержка sealed classes, data class nullability | Java-ориентирован | Лучшая поддержка Kotlin чем Gson |
| Android Studio плагин | Да — валидация JSON на этапе разработки | Нет | Нет |

### Почему kotlinx-serialization?

```kotlin
// kotlinx-serialization: компиляция + полная проверка типов
@Serializable
data class ModrinthProject(
    @SerialName("project_id") val projectId: String = "",  // snake_case → camelCase
    @SerialName("icon_url")   val iconUrl: String? = null
)

// Json конфигурация для устойчивости к изменениям API
val json = Json {
    ignoreUnknownKeys = true      // API добавил поле — не крашимся
    coerceInputValues = true       // Null → default значения из data class
}
```

**Для диплома**: kotlinx-serialization — нативная система сериализации Kotlin, генерирующая код на этапе компиляции. В отличие от Gson (reflection-based), обеспечивает нулевые накладные расходы на рантайме и обнаружение ошибок маппинга JSON на этапе компиляции, а не во время выполнения приложения.

---

## 7. Асинхронность: Coroutines vs RxJava vs AsyncTask

### Сравнение

| Критерий | **Coroutines + Flow** ✅ | RxJava | AsyncTask (устарел) |
|----------|--------------------------|--------|---------------------|
| Синтаксис | `suspend fun` — выглядит как синхронный код | Операторы map/flatMap/subscribe — функциональный стиль | Устарел с Android 11 |
| Обучение | Низкий порог входа (простые конструкции) | Высокий (chain операторы, schedulers) | Низкий, но не рекомендуется |
| Backpressure | Flow: shareIn, buffer, conflate | Built-in (buffer, throttleFirst и т.д.) | Нет |
| Тестирование | runTest, coroutineTest — встроенные | TestScheduler — отдельный модуль | Невозможно без моков |
| Интеграция с Android | lifecycleScope, viewModelScope — нативная | RxAndroid — сторонняя библиотека | Встроен (но deprecated) |
| Размер библиотеки | ~100KB | ~500KB+ | 0 (deprecated в stdlib) |

### Паттерны использования в проекте:

```kotlin
// Pattern 1: Реалтайм-данные из Firestore → Flow → StateFlow
val favoriteProjects: StateFlow<List<FavoriteProject>> = repository.getFavorites()
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

// Pattern 2: Однократный HTTP-запрос в корутине
viewModelScope.launch {
    isLoading = true
    val result = try {
        repository.searchProjects(query)
        Result.success(it)
    } catch (e: Exception) {
        Result.failure(e)
    }
    isLoading = false
}

// Pattern 3: Debounce для поисковых запросов
searchQueryFlow.debounce(500).distinctUntilChanged().collect { query ->
    viewModel.search(query)  // Не чаще раза в 500мс
}
```

**Для диплома**: Kotlin Coroutines — встроенная в язык система асинхронной разработки, обеспечивающая лёгкие потоконезависимые корутины с структурированной конкурентностью. Flow предоставляет реактивный стрим данных с поддержкой backpressure. По сравнению с RxJava требует значительно меньше кода и имеет более низкий порог входа при сопоставимой функциональности.

---

## 8. Облачная база данных: Firestore vs PostgreSQL (Firebase) vs MongoDB Atlas

### Сравнение для проекта NexusForge

| Критерий | **Firestore** ✅ | PostgreSQL + Firebase | MongoDB Atlas |
|----------|------------------|----------------------|---------------|
| Тип данных | NoSQL документы | Реляционная (таблицы) | NoSQL документы |
| Реалтайм-синхронизация | Встроена (snapshots) | Нет — нужен Supabase/WebSocket | Есть, но менее зрелая |
| Авторизация | Интеграция с Firebase Auth | Ручная реализация правил | Ручная реализация |
| Оффлайн-режим | Встроенный кэш + очередь операций | Нет (нужна реализация) | Частичная поддержка |
| Масштабирование | Авто (Google Cloud) | Требует настройки | Auto (MongoDB Atlas) |
| Сложность запросов | Простые query (where, orderBy) | SQL JOIN, подзапросы | Aggregation pipeline |
| Стоимость для проекта | Free tier generous | Бюджетный VPS ($5/мес) | Free tier ограничен |

### Почему Firestore?

```kotlin
// Firestore: реальное время одной строкой
fun getFavorites(): Flow<List<FavoriteProject>> = callbackFlow {
    val listener = query.addSnapshotListener { snapshot, _ ->
        trySend(snapshot?.documents?.mapNotNull { it.toObject(FavoriteProject::class.java) } ?: emptyList())
    }
    awaitClose { listener.remove() }
}

// Пользовательские данные изолированы по UID — безопасность на уровне структуры пути
users/{uid}/favorites/  ← только владелец uid может прочитать
```

**Для диплома**: Google Firestore выбран как облачная база данных благодаря встроенной реалтайм-синхронизации через snapshot listeners, нативной интеграции с Firebase Auth для авторизации и автоматическому оффлайн-кэшированию. Это eliminates необходимость написания серверного кода (backendless architecture), что критически важно для дипломного проекта с ограниченными ресурсами.

---

## 9. Авторизация: Firebase Auth vs собственная реализация

### Сравнение

| Критерий | **Firebase Auth** ✅ | Собственная реализация |
|----------|----------------------|------------------------|
| Email/Password | Встроено | JWT tokens, bcrypt hashing, email verification — неделя разработки |
| Google Sign-In | 3 строки кода через Credential Manager | OAuth2 flow + Google API — день+ разработки |
| Безопасность | Google-уровень (SOC 2, шифрование) | Зависит от квалификации разработчика |
| Сессии | Автоматическое управление токенами | Ручная refresh token логика |
| Multi-factor Auth | Встроенный | Реализация с нуля |
| Rate limiting / Brute force protection | Встроено Google | Самостоятельная реализация |

### Почему Firebase Auth + Credential Manager?

```kotlin
// Google Sign-In через Credential Manager (современная замена deprecated API)
val intent = credentials.newGetCredentialRequest(
    listOf(GoogleID.ID, Password.ID), 
    nonce = null
).intentSenderIntent

startActivityForResult(intent, REQ_CREDENTIALS)  // Пользователь выбирает аккаунт
// Результат → Firebase Auth → signInWithCredential()
```

**Для диплома**: Google Firebase Auth обеспечивает production-ready систему аутентификации с поддержкой email/password и Google Sign-In. Интеграция с Credential Manager (AndroidX) предоставляет platform-agnostic API для входа, что является современным стандартом для Android-приложений. Безопасность уровня предприятия (SOC 2 compliance, шифрование данных) обеспечивается без написания серверного кода.

---

## 10. Загрузка изображений: Coil vs Glide vs Picasso

### Сравнение

| Критерий | **Coil** ✅ | Glide | Picasso |
|----------|-------------|-------|---------|
| Язык | Kotlin-first | Java (обёртка для Kotlin) | Java |
| Асинхронность | Coroutines-native | ExecutorService | AsyncDrawable |
| Compose интеграция | `coil-compose` — нативный компонент | `glide-compose` — сторонний | Нет официальной поддержки |
| Размер библиотеки | ~100KB | ~650KB | ~200KB |
| Синтаксис в Compose | `AsyncImage(model = imageUrl)` | `GlideImage(model = imageUrl)` | Нет нативной интеграции |
| Кэширование | Встроенное (Disk + Memory) | Встроенное | Встроенное |

### Почему Coil?

```kotlin
// Coil: одна строка для загрузки изображения в Compose
AsyncImage(
    model = project.iconUrl,  // URL, File, ResId — coil сам определит тип
    contentDescription = null,
    modifier = Modifier.size(48.dp)
)
// Автоматически: кэширование, загрузка, обработка ошибок, coroutines
```

**Для диплома**: Coil — Kotlin-first библиотека загрузки изображений, построенная на корутинах. Обеспечивает нативную интеграцию с Jetpack Compose через `coil-compose`, имеет минимальный размер библиотеки и zero-boilerplate API. В отличие от Glide/Picasso (Java-ориентированных), Coil является естественным выбором для проектов на Kotlin/Compose.

---

## 11. Генерация .mrpack: собственная реализация vs готовые решения

### Реализация в проекте

```
Пользователь выбирает моды → MrpackGenerator.kt
    │
    ├─ 1. Скачивание каждого мода по downloadUrl
    ├─ 2. Верификация SHA-1 + SHA-512 хешей (двойная проверка целостности)
    ├─ 3. Создание modrinth.index.json в формате Modrinth Pack Format
    │   {
    │     "formatVersion": 1,
    │     "game": {"version": "1.20.4"},
    │     "dependencies": {...},
    │     "files": [...]
    │   }
    └─ 4. Упаковка в ZIP-архив с опциональной overrides/ директорией
```

### Почему собственная реализация?

| Критерий | Собственная реализация ✅ | Готовые библиотеки |
|----------|---------------------------|-------------------|
| Контроль | Полный контроль над форматом .mrpack | Зависимость от сторонней библиотеки |
| Верификация хешей | Двойная проверка SHA-1 + SHA-512 | Может не поддерживать оба алгоритма |
| Размер APK | ~30KB (utils/MrpackGenerator.kt) | +200-500KB зависимость |
| Зависимости | Только stdlib (java.util.zip, java.security) | Дополнительные транзитивные зависимости |

**Для диплома**: Генерация `.mrpack` файлов реализована собственными средствами с использованием стандартной библиотеки Java (`java.util.zip.ZipOutputStream`, `java.security.MessageDigest`). Это обеспечивает минимальную зависимость от внешних библиотек, полный контроль над форматом и двойную верификацию целостности скачиваемых модов через SHA-1 и SHA-512.

---

## 12. Итоговая сравнительная таблица стека

| Компонент | Выбранный стек | Альтернатива 1 | Альтернатива 2 | Ключевое преимущество |
|-----------|---------------|----------------|----------------|----------------------|
| Язык | **Kotlin** | Java | Scala | Null-safety, coroutines, official Google language |
| UI | **Jetpack Compose** | XML Views | Flutter | Declarative, reactive, Material3 native |
| Архитектура | **MVVM + Repository** | MVI | MVP | Official Google recommendation, tested pattern |
| Навигация | **Navigation3** | Legacy NavGraph | Custom router | Type-safe sealed interface routes |
| HTTP Client | **Retrofit** | OkHttp raw | Ktor | Declarative API definitions, suspend functions |
| Serialization | **kotlinx-serialization** | Gson | Moshi | Compile-time codegen, zero reflection overhead |
| Async | **Coroutines + Flow** | RxJava | LiveData only | Lightweight structured concurrency |
| Database | **Firestore** | PostgreSQL + server | Room local only | Real-time sync, offline cache, no backend code |
| Auth | **Firebase Auth** | Custom JWT | OAuth library | Google-grade security, zero server code |
| Images | **Coil** | Glide | Picasso | Kotlin-first, coroutine-native, Compose native |

---

## 13. Аргументация для защиты диплома: ключевые тезисы

### Тезис 1: Современный стек соответствует индустриальным стандартам
Проект использует технологии, рекомендованные Google как стандарты Android-разработки (Kotlin, Compose, MVVM, Navigation3). Это обеспечивает актуальность разработанного решения и применимость описанных подходов в реальных коммерческих проектах.

### Тезис 2: Backendless архитектура — оптимальный выбор для дипломного проекта
Использование Firebase (Auth + Firestore) eliminates необходимость написания серверного кода, что позволяет сосредоточиться на клиентской части приложения при сохранении production-ready функциональности (реалтайм-синхронизация, оффлайн-режим, масштабируемость).

### Тезис 3: Реактивная архитектура обеспечивает плавный UX
Связка Coroutines + Flow + Compose создаёт полностью реактивный UI, который автоматически обновляется при изменении данных. Пользователь видит изменения в реальном времени без ручного обновления — ключевое преимущество для приложения с системой избранного и модпаков.

### Тезис 4: Типобезопасность на всех уровнях
Sealed interface навигации, kotlinx-serialization, sealed class UI states — все слои проекта используют компилятор Kotlin для обнаружения ошибок на этапе компиляции, а не на рантайме. Это повышает надёжность и снижает количество багов.

### Тезис 5: Безопасность данных встроена в архитектуру
Изоляция пользовательских данных через UID в пути Firestore, санитизация ввода (InputSanitizer), двойная верификация файловых хешей — безопасность реализована на уровне архитектуры, а не как дополнение.

### Тезис 6: Оптимизация ресурсов
WhileSubscribed(5000) для отключения слушателей при неактивности, debounce для поисковых запросов (500мс), .await() bridge для Firebase — все решения направлены на минимизацию потребления батареи и трафика.

---

## 14. Итоговая архитектура в одном изображении (текстовое представление)

```
┌─────────────────────────────────────────────────────────────┐
│                    USER INTERFACE                            │
│              Jetpack Compose + Material3                     │
│         (~20 экранов, реактивный UI через StateFlow)         │
└──────────────────────────┬──────────────────────────────────┘
                           │ collectAsState() / onClick callbacks
┌──────────────────────────▼──────────────────────────────────┐
│                      VIEWMODEL LAYER                         │
│        Coroutines + Flow (12 ViewModel, state management)    │
│  viewModelScope.launch { } → repository methods              │
└──────────────────────────┬──────────────────────────────────┘
                           │ suspend fun calls
┌──────────────────────────▼──────────────────────────────────┐
│                    REPOSITORY LAYER                          │
│     FirestoreRepository (real-time Flow, CRUD operations)    │
│     AuthRepository (Firebase Auth wrapper)                   │
│     GoogleDriveRepository (ZIP upload with progress)         │
└──────┬───────────────────────────┬──────────────────────────┘
       │                           │
┌──────▼───────────┐      ┌────────▼───────────────┐
│  GOOGLE FIRESTORE│      │   MODRINTH API          │
│  (Cloud NoSQL DB)│      │   (Retrofit +           │
│  Real-time sync  │      │    kotlinx-serialization)│
│  Offline cache   │      │   GET search/projects/   │
└──────────────────┘      └─────────────────────────┘

ФОН: Firebase Auth (Email/Password + Google Sign-In via Credential Manager)
```

