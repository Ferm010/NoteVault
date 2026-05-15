# Kotlin — Вопросы и ответы для экзамена

## Базовые вопросы

### 1. В чём разница между val и var?

**Ответ:**
- `val` — неизменяемая ссылка (immutable reference). Нельзя переназначить переменную после инициализации. Аналог `final` в Java.
- `var` — изменяемая ссылка (mutable reference). Можно переназначать сколько угодно раз.

```kotlin
val name = "Alex"    // Нельзя: name = "Bob"
var age = 25         // Можно: age = 26
```

**Важно**: `val` делает неизменной ссылку, но не объект. Если объект мутабельный (например, `MutableList`), его содержимое можно менять.

### 2. Что такое nullable типы в Kotlin?

**Ответ:** Nullable тип — это тип, значение которого может быть `null`. Обозначается знаком `?` после типа.

```kotlin
var name: String = "Alex"    // Не может быть null
var description: String? = null  // Может быть null
```

Для работы с nullable используются:
- **Безопасный вызов**: `description?.length` — вернёт `null` если `description == null`
- **Elvis operator**: `description?.length ?: 0` — если `null`, вернуть `0`
- **Null-safe cast**: `obj as? String` — вернёт `null` вместо бросания исключения

### 3. Что такое data class и чем он отличается от обычного class?

**Ответ:** Data class — это класс, автоматически генерирующий:
- `equals()` / `hashCode()` — по всем property в constructor
- `toString()` — в формате `User(name=Alex, email=test@mail.com)`
- `copy()` — создание копии с изменёнными полями
- `componentN()` — для деструктуризации

```kotlin
data class User(val name: String, val email: String)

val user1 = User("Alex", "alex@mail.com")
val user2 = user1.copy(name = "Bob")  // Копия с другим именем
user1 == user2                       // false (имена разные)

val (name, email) = user1            // Деструктуризация: name="Alex"
```

### 4. Что такое sealed class / sealed interface?

**Answer:** Sealed class/interface ограничивает иерархию типов — все подтипы известны при компиляции. Это позволяет использовать `when` без `else`:

```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    object Error : Result()
}

fun handle(result: Result) {
    when (result) {
        is Result.Success -> println(result.data)  // Компилятор знает, что это Success
        Result.Error -> println("Error")            // else не нужен!
    }
}
```

**Sealed interface** — более новая версия, подтипы могут быть в разных файлах.

### 5. В чём разница между lateinit и lazy?

| Характеристика | `lateinit var` | `by lazy { ... }` |
|---|---|---|
| Тип | `var` (mutable) | `val` (immutable) |
| Инициализация | Отложенная, вручную | Автоматическая при первом доступе |
| Поток | Main thread | По умолчанию — главный поток, можно указать dispatcher |
| Повторный доступ | Возвращает значение | Кэширует результат |
| Проверка | `::prop.isInitialized` | Всегда инициализирован после первого доступа |

```kotlin
lateinit var repository: UserRepository  // Инициализируем в onCreate()

val config by lazy { loadConfig() }      // Загружается при первом обращении
```

## Вопросы на понимание Kotlin-специфик

### 6. Что такое extension functions? Приведите пример.

**Ответ:** Extension function — функция, которая добавляет поведение существующему классу без наследования:

```kotlin
fun String.isEmail(): Boolean {
    return this.contains("@") && this.contains(".")
}

"test@mail.com".isEmail()  // true — вызываем как метод String!
```

### 7. Объясните scope functions: run, let, apply, also, with.

**Ответ:** Scope functions вызывают лямбду в контексте объекта:

| Функция | Receiver | Arg | Return | Когда использовать |
|---|---|---|---|---|
| `run` | `this` | — | результат | Конфигурация + возврат результата |
| `let` | `this` | `it` | результат | Nullable-безопасность, трансформации |
| `apply` | `this` | — | `this` | Builder pattern, настройка объекта |
| `also` | `this` | `it` | `this` | Побочные эффекты (логирование) |
| `with` | `this` | — | результат | Множественные вызовы одного объекта |

```kotlin
// let — nullable safe
name?.let { println(it) }

// apply — builder pattern
val user = User("Alex").apply { email = "alex@mail.com" }

// also — побочный эффект
button.also { it.visibility = View.VISIBLE }.setOnClickListener {}

// run — конфигурация + результат
val size = editText.run {
    setText("Hello")
    measuredWidth  // возвращаем результат
}
```

### 8. Что такое Coroutines и Dispatchers?

**Ответ:** Корутин — лёгкий поток, управляемый на уровне библиотеки (не ОС). Позволяет писать асинхронный код в синхронном стиле.

**Dispatchers** определяют, на каком потоке выполняется корутина:
- `Dispatchers.Main` — UI поток Android
- `Dispatchers.IO` — сеть, файловая система, БД
- `Dispatchers.Default` — тяжёлые вычисления (CPU-bound)
- `Dispatchers.Unconfined` — без ограничений (не использовать!)

```kotlin
lifecycleScope.launch(Dispatchers.Main) {
    val data = withContext(Dispatchers.IO) {
        repository.loadData()  // Background работа
    }
    textView.text = data  // Обновление UI
}
```

### 9. Что такое suspend функции?

**Ответ:** `suspend` функция — функция, которую можно приостановить и возобновить позже. Может вызываться только из coroutine или другой suspend функции:

```kotlin
// Может быть приостановлена (например, при сетевом запросе)
suspend fun fetchUser(id: String): User {
    return api.getUser(id)  // HTTP request — может ждать
}

// Вызов только внутри корутины
lifecycleScope.launch {
    val user = fetchUser("123")  // OK!
}
```

### 10. Что такое Flow? Как он отличается от LiveData?

**Ответ:** `Flow` — реактивный поток данных, безопасный для корутин:

| Характеристика | Flow | LiveData |
|---|---|---|
| Язык | Kotlin-only | Java/Kotlin |
| Стек | Coroutines | LifecycleObserver |
| Cold/Hot | Cold (начинает при collect) | Hot (всегда активен) |
| Операторы | map, filter, merge, combine | Без операторов |
| LifeCycle-aware | Нет (но есть collectAsStateWithLifecycle) | Да, автоматически |

```kotlin
// Flow — последовательность асинхронных значений
val flow = flow {
    for (i in 1..5) {
        emit(i)  // Отправляем значение
    }
}

flow.collect { value -> 
    println(value)  // 1, 2, 3, 4, 5
}
```

## Продвинутые вопросы

### 11. Что такое StateFlow и SharedFlow? В чём разница?

**Ответ:** Оба — hot streams (начинают работать сразу):

| Характеристика | StateFlow | SharedFlow |
|---|---|---|
| Хранит значение | ✅ Да, всегда актуальное | ❌ Нет (но можно настроить buffer) |
| Подписчики получают последнее | ✅ Да | ❌ Только новые значения |
| replay | Кол-во последних значений | Кол-во последних значений |
| Когда использовать | Состояние UI | События (навигация, toast) |

```kotlin
// StateFlow — состояние
private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
val uiState: StateFlow<UiState> = _uiState

_uiState.value = UiState.Success(data)  // Обновляем

// SharedFlow — события (одноразовые)
private val _event = MutableSharedFlow<Event>()
_event.emit(Event.NavigateToSettings)  // Один раз для каждого подписчика
```

### 12. Что такое reified type parameters?

**Ответ:** `reified` позволяет использовать тип внутри inline-функции (обычно типы стираются при компиляции):

```kotlin
inline fun <reified T> Gson.fromJson(json: String): T {
    return fromJson(json, T::class.java)  // Доступ к типу!
}

val user = gson.fromJson<User>(jsonString)
```

### 13. Объясните принцип SOLID на примере Kotlin.

**Ответ:**

- **S — Single Responsibility**: Класс должен иметь одну причину для изменения. `AuthRepository` отвечает только за аутентификацию, не за UI.
  
- **O — Open/Closed**: Открыт для расширения, закрыт для модификации. `sealed class Result` позволяет добавлять новые подтипы без изменения кода обработки.

```kotlin
sealed class Result {
    object Success : Result()  // Можно добавить новый тип без изменения when!
}
```

- **L — Liskov Substitution**: Подтипы должны заменять базовый тип. `class FirestoreRepository : Repository` может использоваться вместо `Repository`.

- **I — Interface Segregation**: Интерфейсы должны быть маленькими. Вместо одного большого `ApiService` лучше разбить на `ModrinthApiService`, `DriveApiService`.

- **D — Dependency Inversion**: Зависимости от абстракций, не от конкретики. ViewModel зависит от `AuthRepository` (интерфейс/класс), а не от конкретной реализации Firebase.

### 14. Что такое inline функции и зачем они нужны?

**Ответ:** Inline функция — код лямбды встраивается прямо в место вызова, без создания объекта Function:

```kotlin
inline fun runIf(condition: Boolean, block: () -> Unit) {
    if (condition) block()  // Нет overhead от создания лямбда-объекта!
}
```

**Преимущества:**
- Нет создания объектов для лямбд — лучше performance
- Можно использовать `return@` с именем функции
- Можно использовать `reified` типы

### 15. Что такое高阶 функции (higher-order functions)?

**Ответ:** Функция, которая принимает другую функцию как параметр или возвращает функцию:

```kotlin
// Принимает функцию
fun performOperation(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}
performOperation(3, 4) { x, y -> x + y }  // 7

// Возвращает функцию
fun multiplier(factor: Int): (Int) -> Int {
    return { it * factor }
}
val double = multiplier(2)
double(5)  // 10
```

## Вопросы по практике в проекте NexusForge

### 16. Как в проекте используется sealed interface для навигации?

**Ответ:** В `NavigationData.kt`:

```kotlin
@Serializable
sealed interface Destination : NavKey {
    @Serializable data object MainMenu : Destination
    @Serializable data class ProjectDetailsPage(val projectId: String) : Destination
}
```

Это обеспечивает type-safe навигацию — компилятор знает все возможные экраны и их параметры.

### 17. Как в проекте организован Auth? Почему checkEmailExists создаёт временный аккаунт?

**Ответ:** Firebase не предоставляет API для проверки существования email. Поэтому `AuthRepository.checkEmailExists()` пытается создать аккаунт с временным паролем:
- Если создался → email свободен (`NewUser`)
- Если ошибка `ERROR_EMAIL_ALREADY_IN_USE` → email занят (`ExistingUser`)

### 18. Зачем нужен SecurityCheck? Что он проверяет?

**Ответ:** `SecurityCheck.verifyAppIntegrity()` проверяет подпись APK — гарантирует, что приложение не было модифицировано или пересобрано кем-то другим. Если подпись не совпадает — приложение завершается.

### 19. Как в проекте используется coroutine + Firebase?

**Ответ:** Через `kotlinx.coroutines.play.services` — Firebase методы возвращают `Task<T>`, которые можно использовать с `.await()`:

```kotlin
auth.signInWithEmailAndPassword(email, password).await()
firestore.collection("users").document(uid).get().await()
```

### 20. Почему в проекте используются data class для моделей данных?

**Ответ:** Data classes автоматически генерируют `equals()`, `hashCode()`, `toString()`, `copy()` — это критично для:
- Сравнения объектов (Firestore documents)
- Кэширования (ViewModel state)
- Навигации с параметрами (`data class ProjectDetailsPage(val projectId: String)`)
