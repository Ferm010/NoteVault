# Корутин (Coroutines) — Глубокое погружение для экзамена

## 1. Основы Coroutines

### Что такое Coroutine?

Coroutine — лёгкий поток, управляемый на уровне библиотеки (не ОС). В отличие от `Thread`, корутина:
- Весит ~1KB vs ~1MB у потока
- Можно запустить миллионы одновременно
- Переключение между корутин — за микросекунды
- Не блокирует OS thread

### Структура корутины

```kotlin
// 1. Запуск в CoroutineScope
lifecycleScope.launch {
    // Корутина работает здесь
}

// 2. CoroutineScope — область жизни корутины
lifecycleScope      // Activity/Fragment lifecycle-aware
viewModelScope      // ViewModel lifecycle-aware
runBlocking         // Блокирует поток до завершения (только для тестов/main)
GlobalScope         // Живёт до конца приложения (не использовать!)

// 3. Dispatchers — где выполняется корутина
Dispatchers.Main    // UI поток
Dispatchers.IO      // Сеть, файлы, БД
Dispatchers.Default // Тяжёлые вычисления
```

## 2. launch vs async

### launch — "fire and forget"

```kotlin
lifecycleScope.launch {
    repository.loadData()  // Не возвращаем результат
}
```

### async + await — с результатом

```kotlin
val deferred = lifecycleScope.async(Dispatchers.IO) {
    repository.loadUser()  // Возвращает User
}

// Ждём результат (блокирует корутину, не поток!)
val user = deferred.await()
```

### Запуск нескольких async параллельно

```kotlin
val usersDeferred = async(Dispatchers.IO) { repository.getUsers() }
val settingsDeferred = async(Dispatchers.IO) { repository.getSettings() }

// Ждём оба результата (параллельно!)
val users = usersDeferred.await()
val settings = settingsDeferred.await()
```

## 3. withContext — переключение dispatcher'ов

```kotlin
lifecycleScope.launch(Dispatchers.Main) {
    // UI поток
    
    val data = withContext(Dispatchers.IO) {
        repository.loadData()  // Background работа
    }
    
    // Снова UI поток, data готов!
    textView.text = data.name
}
```

**Важно**: `withContext` не блокирует OS thread — он приостанавливает корутину и переключает на другой dispatcher.

## 4. Структура данных в проекте

### sealed classes для результатов операций

```kotlin
// AuthRepository.kt
sealed class EmailCheckResult {
    object NewUser : EmailCheckResult()
    data class ExistingUser(val authMethod: AuthMethod) : EmailCheckResult()
    data class Error(val errorCode: String) : EmailCheckResult()
}

sealed class AuthResult {
    object Success : AuthResult()
    data class Error(val errorCode: String) : AuthResult()
}

sealed class GoogleSignInResult {
    data class Success(val isNewUser: Boolean, val displayName: String, val email: String) : GoogleSignInResult()
    data class Error(val errorCode: String) : GoogleSignInResult()
}
```

**Почему sealed classes?** Компилятор требует обработать ВСЕ варианты в `when` — нельзя забыть ошибку.

### Result<T> из Kotlin stdlib

```kotlin
suspend fun checkUserProfileExists(): Result<Boolean> {
    return try {
        val doc = firestore.collection("users").document(uid).get().await()
        Result.success(doc.exists())
    } catch (e: Exception) {
        Result.failure(e)
    }
}

// Использование
when (val result = repository.checkUserProfileExists()) {
    is Result.Success -> if (result.getOrNull() == true) navigateToMainMenu()
    is Result.Failure -> showError(result.exception.message)
}
```

## 5. Flow в проекте

### StateFlow — состояние UI

```kotlin
// ViewModel
class ThemeViewModel : ViewModel() {
    private val _isDarkTheme = MutableStateFlow(false)
    val isDarkTheme: StateFlow<Boolean> = _isDarkTheme
    
    fun toggleTheme() {
        _isDarkTheme.value = !_isDarkTheme.value  // Обновляем
    }
}

// UI — collectAsStateWithLifecycle останавливает сборку когда экран не виден
@Composable
fun SettingsScreen(viewModel: ThemeViewModel) {
    val isDark by viewModel.isDarkTheme.collectAsStateWithLifecycle()
    
    Switch(
        checked = isDark,
        onCheckedChange = { viewModel.toggleTheme() }
    )
}
```

### SharedFlow — события (одноразовые)

Используется для событий навигации, показов snackbar:

```kotlin
// ViewModel
class MainViewModel : ViewModel() {
    private val _navigateEvent = MutableSharedFlow<Destination>()
    
    suspend fun navigateToSettings() {
        _navigateEvent.emit(Destination.SettingsPage)
    }
}

// UI — collect только один раз за сборку
lifecycleScope.launch {
    viewModel.navigateEvents.collect { dest ->
        navController.navigate(dest)  // Навигация по событию
    }
}
```

## 6. Операторы Flow

### map, filter — трансформации

```kotlin
// Фильтруем и трансформируем данные
viewModel.favoriteProjects
    .filter { it.isAvailable }
    .map { ModCardData(it.id, it.title, it.thumbnailUrl) }
    .collect { cards -> updateUI(cards) }
```

### combine — объединение нескольких потоков

```kotlin
// Объединяем состояние из двух ViewModel
combine(
    viewModel1.uiState,
    viewModel2.uiState
) { state1, state2 ->
    CombinedState(state1, state2)
}
.collect { combined -> /* обновить UI */ }
```

### catch — обработка ошибок в потоке

```kotlin
flow {
    emit(repository.loadData())
}.catch { e ->
    // Обрабатываем ошибку прямо в потоке
    _error.value = e.message
}
```

## 7. Coroutine Scopes в Android

| Scope | Lifecycle | Когда использовать |
|---|---|---|
| `lifecycleScope` (Activity/Fragment) | Живёт пока Activity активна | Загрузка данных при открытии экрана |
| `viewModelScope` (ViewModel) | Живёт пока ViewModel существует | Фоновые задачи, работа с БД |
| `reportDrawn()` scope | До первого кадра | Критичные для UX инициализации |

```kotlin
// ✅ Правильно — viewModelScope не утечёт при изменении конфигурации
class MyViewModel : ViewModel() {
    init {
        viewModelScope.launch {
            repository.loadData()  // Автоматически отменится при destroy ViewModel
        }
    }
}

// ✅ Правильно — lifecycleScope привязан к Activity
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        lifecycleScope.launch {
            checkUserProfile()  // Отменится при finish Activity
        }
    }
}
```

## 8. Coroutine + Compose

### LaunchedEffect — корутина в Composable

```kotlin
@Composable
fun LoadProject(projectId: String) {
    val viewModel: ProjectDetailsViewModel = viewModel()
    
    // Запускается при создании + при изменении projectId
    LaunchedEffect(projectId) {
        viewModel.loadProject(projectId)  // suspend функция
    }
}

// Unit — запускается один раз при создании composable
@Composable
fun InitScreen(viewModel: ThemeViewModel, context: Context) {
    LaunchedEffect(Unit) {
        viewModel.init(context, isSystemInDarkTheme())
    }
}
```

### collectAsStateWithLifecycle vs collectAsState

| Метод | Останавливается при stop? | Сохраняет последнее значение? |
|---|---|---|
| `collectAsState()` | ❌ Нет | ❌ Нет |
| `collectAsStateWithLifecycle()` | ✅ Да | ✅ Да |

```kotlin
// ✅ Всегда используйте collectAsStateWithLifecycle для UI
val state by viewModel.uiState.collectAsStateWithLifecycle()

// ❌ Не используйте — будет собирать даже на background
val state by viewModel.uiState.collectAsState()
```

## 9. Error Handling в Coroutines

### try-catch с suspend функциями

```kotlin
lifecycleScope.launch {
    try {
        val user = repository.loadUser(userId)
        _uiState.value = UiState.Success(user)
    } catch (e: FirebaseAuthException) {
        _uiState.value = UiState.Error(mapErrorCode(e.errorCode))
    } catch (e: IOException) {
        _uiState.value = UiState.Error("Нет подключения к сети")
    } catch (e: Exception) {
        _uiState.value = UiState.Error("Неизвестная ошибка")
    }
}
```

### Result<T> для передачи ошибок

```kotlin
// Repository — возвращает Result
suspend fun loadProject(id: String): Result<ModrinthProject> {
    return try {
        val project = api.getProject(id)
        Result.success(project)
    } catch (e: Exception) {
        Result.failure(e)
    }
}

// ViewModel — обрабатывает Result
fun loadProject(id: String) {
    viewModelScope.launch {
        when (val result = repository.loadProject(id)) {
            is Result.Success -> _uiState.value = UiState.Success(result.getOrNull()!!)
            is Result.Failure -> _uiState.value = UiState.Error(result.exception.message ?: "Error")
        }
    }
}
```

## 10. Common Mistakes с Coroutines

### ❌ Запуск корутин без LaunchedEffect в Compose

```kotlin
// BAD — запускается при каждой recomposition!
@Composable
fun BadLoad(id: String) {
    lifecycleScope.launch { repository.loadById(id) }  // Многократный запуск!
}

// GOOD — запускается только при изменении key
@Composable
fun GoodLoad(id: String) {
    LaunchedEffect(id) { repository.loadById(id) }  // Запускается один раз (или при смене id)
}
```

### ❌ GlobalScope в Android

```kotlin
// BAD — корутина не отменится, утечка памяти!
GlobalScope.launch { repository.loadData() }

// GOOD — привязано к lifecycle
lifecycleScope.launch { repository.loadData() }
```

### ❌ Не обрабатывать исключения Firebase

```kotlin
// BAD — необработанный FirebaseAuthException
auth.signInWithEmailAndPassword(email, password).await()

// GOOD — try-catch с конкретными ошибками
try {
    auth.signInWithEmailAndPassword(email, password).await()
} catch (e: FirebaseAuthException) {
    when (e.errorCode) {
        "ERROR_WRONG_PASSWORD" -> showError("Invalid password")
        "ERROR_USER_NOT_FOUND" -> showError("User not found")
    }
}
```

### ❌ Использовать runBlocking в UI коде

```kotlin
// BAD — блокирует main thread!
lifecycleScope.launch {
    val data = runBlocking { repository.loadData() }  // BLOCKS UI!
}

// GOOD — withContext не блокирует поток
lifecycleScope.launch {
    val data = withContext(Dispatchers.IO) { repository.loadData() }
}
```

## 11. Вопросы для самопроверки по Coroutines

### Q: Что произойдёт если вызвать suspend функцию из обычного метода?

**A:** Ошибка компиляции! Suspend функции могут вызываться только из coroutine или другой suspend функции.

### Q: В чём разница между launch и async?

**A**: `launch` — не возвращает результат ("fire and forget"), `async` — возвращает `Deferred<T>`, который можно.await() для получения результата.

### Q: Что делает .await() у Firebase Task?

**A:** Конвертирует callback-based `Task<T>` в suspend функцию — позволяет использовать `val result = task.await()` вместо `.addOnSuccessListener { }`.

### Q: Почему LaunchedEffect(projectId) перезапускается при изменении projectId?

**A:** Потому что `projectId` — ключ эффекта. Когда ключ меняется, предыдущая корутина отменяется и запускается новая с новым значением. Это предотвращает race conditions.

### Q: Что будет если ViewModel уничтожится пока корутина в viewModelScope работает?

**A:** Корутина автоматически отменится (cancellation). `viewModelScope` привязан к lifecycle ViewModel.
