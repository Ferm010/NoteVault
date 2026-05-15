# Основы Kotlin для Android-разработки

## 1. Типы данных

### Примитивные типы (обёрнутые)

| Тип | Размер | Описание | Пример |
|---|---|---|---|
| `String` | — | Строка | `"Hello"` |
| `Int` | 32-bit | Целое число | `42`, `0xFF` |
| `Long` | 64-bit | Большое целое | `42L` |
| `Float` | 32-bit | Число с плавающей точкой | `3.14f` |
| `Double` | 64-bit | Двойная точность | `3.14` |
| `Boolean` | — | Логический тип | `true`, `false` |
| `Char` | 16-bit | Символ | `'A'` |

### Nullable и Non-Null типы

```kotlin
var name: String = "NexusForge"   // Не может быть null
var description: String? = null    // Может быть null

// Безопасный вызов
val length = description?.length  // Int? — вернёт null если description == null

// Elvis operator (оператор Эйлиса)
val safeLength = description?.length ?: 0  // Если null, вернуть 0

// !! — принудительное разимплементирование (не безопасно!)
val unsafeLength = description!!.length   // Бросит NPE если null
```

## 2. Переменные: val vs var

```kotlin
val constant = 42           // Immutable, нельзя переназначить
var mutable = "text"        // Mutable, можно переназначать
mutable = "new text"        // OK
// constant = 10            // ERROR! Нельзя

// val — ссылка неизменна, объект может быть мутабельным
val list = mutableListOf(1, 2, 3)
list.add(4)                 // OK — список мутируется
// list = mutableListOf()   // ERROR — ссылку нельзя сменить
```

**Правило**: Всегда используй `val` по умолчанию. `var` только если нужно переназначить.

## 3. Функции

```kotlin
// Базовая функция
fun greet(name: String): String {
    return "Hello, $name!"
}

// Single-expression (однострочная)
fun double(x: Int): Int = x * 2

// Без return — последнее выражение не нужно
fun sayHello(name: String) {
    println("Hello, $name!")  // Unit возвращается по умолчанию
}

// Значения по умолчанию
fun createUser(name: String, role: String = "user") { ... }
createUser("Alex")              // role = "user"
createUser("Alex", "admin")     // role = "admin"

// Именованные аргументы
createUser(name = "Alex", role = "admin")

// Extension functions (расширяющие функции)
fun String.isEmail(): Boolean {
    return this.contains("@")
}
"test@mail.com".isEmail()  // true
```

## 4. Классы и объекты

### Data Class

Автоматически генерирует `equals()`, `hashCode()`, `toString()`, `copy()`:

```kotlin
data class User(val name: String, val email: String)

val user1 = User("Alex", "alex@mail.com")
val user2 = user1.copy(name = "Bob")  // Копия с новым именем
```

### Sealed Class / Interface

Ограничивает иерархию типов — все подтипы известны в compile-time:

```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    data class Error(val message: String) : Result()
    object Loading : Result()
}

// Когда используем when, компилятор требует покрыть все варианты
fun handleResult(result: Result) {
    when (result) {
        is Result.Success -> println(result.data)
        is Result.Error -> println("Error: ${result.message}")
        Result.Loading -> println("Loading...")
    }
}
```

В проекте используется в `AuthRepository.kt`:
- `EmailCheckResult` — NewUser / ExistingUser / Error
- `GoogleSignInResult` — Success / Error
- `AuthResult` — Success / Error

### Object (Singleton)

```kotlin
object ModrinthApi {
    val retrofitService: ModrinthApiService by lazy { ... }
}
// Один экземпляр на всё приложение, доступ через ModrinthApi.retrofitService
```

## 5. Корутин (Coroutines)

### Основные концепции

```kotlin
import kotlinx.coroutines.*

// Запуск корутины в main thread (нужен CoroutineScope)
lifecycleScope.launch {
    // UI-операции
}

// Запуск на background thread
lifecycleScope.launch(Dispatchers.IO) {
    // Сетевые запросы, работа с БД
}

// Ожидание результата корутины
val result = withContext(Dispatchers.IO) {
    repository.loadData()  // Возвращаемое значение
}

// await() — для ожидания Deferred
val deferred = async(Dispatchers.IO) { loadData() }
val data = deferred.await()
```

### Dispatchers

| Dispatcher | Назначение |
|---|---|
| `Dispatchers.Main` | UI-поток (Android) |
| `Dispatchers.IO` | Сеть, файловая система, БД |
| `Dispatchers.Default` | Тяжёлые вычисления |
| `Dispatchers.Unconfined` | Без ограничений (не использовать!) |

### suspend функции

```kotlin
// Могут быть приостановлены и возобновлены
suspend fun fetchUser(id: String): User {
    return withContext(Dispatchers.IO) {
        api.getUser(id)  // Сетевой запрос
    }
}
```

## 6. Лямбды и higher-order функции

```kotlin
// Лямбда-выражение
val sum: (Int, Int) -> Int = { a, b -> a + b }

// Передача лямбды в функцию
fun performOperation(a: Int, b: Int, operation: (Int, Int) -> Int): Int {
    return operation(a, b)
}
performOperation(3, 4) { x, y -> x * y }  // 12

// last {} — лямбда как последний аргумент вне скобок
list.forEach { println(it) }

// run, apply, with (scope functions)
val user = User("Alex", "alex@mail.com").apply {
    // Контекст этого объекта доступен как this
}
```

## 7. Деструктуризация

```kotlin
data class Pair(val first: String, val second: Int)

val pair = Pair("key", 42)
val (key, value) = pair
println("$key -> $value")  // key -> 42
```

В проекте используется для навигации с параметрами через `NavKey`.

## 8. Аннотации в Android/Kotlin

| Аннотация | Назначение |
|---|---|
| `@Composable` | Функция Compose UI |
| `@Preview` | Предпросмотр в IDE |
| `@Serializable` | Сериализация kotlinx.serialization |
| `@OptIn(...)` | Разрешить использование experimental API |
| `@Suppress("...")` | Подавить предупреждения |

## 9. Интерфейсы

```kotlin
interface ApiService {
    suspend fun getUser(id: String): User
    
    // Default method (можно переопределить)
    fun getName(): String = "Default"
}

class ModrinthApiService : ApiService {
    override suspend fun getUser(id: String): User { ... }
    // getName() можно не переопределять — будет "Default"
}
```

## 10. Generics (Дженерики)

```kotlin
// Generic class
data class ApiResponse<T>(val data: T?, val error: String?)

// Использование
val userResponse = ApiResponse<User>(user, null)
val listResponse = ApiResponse<List<String>>(list, "timeout")

// Where clause
fun <T : Comparable<T>> List<T>.maxItem(): T { ... }
```
