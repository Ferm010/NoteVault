# Kotlin: Продвинутые темы

## 1. Scope Functions (Функции области видимости)

### this vs it

| Функция | Receiver | Lambda arg | Возвращает |
|---|---|---|---|
| `run` | `this` | — | Результат лямбды |
| `let` | `this` | `it` | Результат лямбды |
| `apply` | `this` | — | Объект (`this`) |
| `with` | `this` | — | Результат лямбды |
| `also` | `this` | `it` | Объект (`this`) |

### Примеры использования

```kotlin
// run — объединить создание и конфигурацию
val result = mutableListOf<Int>().run {
    add(1)
    add(2)
    add(3)
    this.sum()  // возвращаем сумму
}

// let — безопасная работа с nullable
val name: String? = null
name?.let { println("Name is $it") }  // Не выполнится, если null

// apply — конфигурация объекта (builder pattern)
val user = User("Alex", "alex@mail.com").apply {
    // this — user
    role = "admin"
}

// also — побочные эффекты (логирование, анимации)
button.also { it.setOnClickListener { ... } }.animate()

// with — вызов методов объекта без this
val list = listOf(1, 2, 3)
with(list) {
    println(first())   // list.first()
    println(last())    // list.last()
}
```

### Как выбрать?

- Нужен результат лямбды + nullable безопасный вызов → `let`
- Нужно настроить объект и вернуть его → `apply`
- Нужно выполнить что-то после + вернуть тот же объект → `also`
- Нужно объединить создание + логику + результат → `run`
- Множественные вызовы одного объекта без this → `with`

## 2. Extension Functions (Расширяющие функции)

```kotlin
// Расширение для String
fun String.isEmail(): Boolean {
    return this.contains("@") && this.contains(".")
}

// Расширение с nullable
fun String?.orEmptyIfNull(): String {
    return this ?: ""
}

// Расширение для View/Composable
fun TextView.highlight() {
    this.setTextColor(Color.RED)
}

// Extension property (только get/set, нет backing field!)
val String.lastChar: Char
    get() = this[this.length - 1]
```

## 3.高阶 функции и функциональный стиль

### map, filter, reduce, flatMap

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// map — преобразование каждого элемента
numbers.map { it * 2 }           // [2, 4, 6, 8, 10]

// filter — отбор по условию
numbers.filter { it > 2 }       // [3, 4, 5]

// reduce — агрегация
numbers.reduce { acc, value -> acc + value }  // 15

// flatMap — flatten после map
val nested = listOf(listOf(1, 2), listOf(3, 4))
nested.flatMap { it }            // [1, 2, 3, 4]

// Комбинирование
numbers.map { it * 2 }.filter { it > 4 }  // [6, 8, 10]

// find — первый элемент по условию
val found = numbers.find { it > 3 }  // 4

// any, all, none
numbers.any { it > 4 }   // true (есть хотя бы один)
numbers.all { it > 0 }   // true (все больше нуля)
numbers.none { it < 0 }  // true (ни одного отрицательного)
```

### takeWhile, dropWhile, chunked

```kotlin
val nums = listOf(1, 2, 3, 4, 5)
nums.takeWhile { it < 4 }    // [1, 2, 3] — пока условие true
nums.dropWhile { it < 4 }    // [4, 5] — отбросить пока true

nums.chunked(2)              // [[1, 2], [3, 4], [5]]
```

## 4. Flow (Реактивные потоки данных)

Flow — последовательность асинхронных значений, безопасная для корутин:

```kotlin
import kotlinx.coroutines.flow.*

// Создание Flow
val numbers = flow {
    for (i in 1..5) {
        emit(i)      // Отправить значение
    }
}

// Hot vs Cold
// cold — начинает работать при collect
flow { ... }.collect { value -> ... }

// hot — всегда "слушает"
val sharedFlow = MutableSharedFlow<Int>()
sharedFlow.emit(1)  // Можно отправлять из любого места
```

### Типы Flow

| Тип | Описание |
|---|---|
| `Flow<T>` | Cold stream, collect начинает выполнение |
| `MutableStateFlow<T>` | Хранит текущее значение, hot stream |
| `SharedFlow<T>` | Публикует значения подписчикам, hot |
| `Channel<T>` | Канал для отправки значений по await |

### Операторы Flow

```kotlin
// Преобразование
flow.map { it * 2 }
flow.filter { it > 0 }
flow.flatMapConcat { ... }

// Агрегация
flow.reduce { acc, value -> acc + value }
flow.toList()
flow.first { it > 3 }

// Операторы для UI (StateFlow)
val state by viewModel.count.collectAsState()
```

### Flow в проекте

ViewModel'и используют `StateFlow`/`SharedFlow` для передачи данных в UI:
- `FavoritesViewModel.favoriteProjects` — StateFlow списка избранных
- `ThemeViewModel.isDarkTheme` — StateFlow темы

## 5. Context Receivers (Контекстные получатели)

```kotlin
// Kotlin 1.97+ / экспериментально
context(AnimationScope<T>)
@Composable
fun AnimatedText(text: String) {
    // Здесь доступен this как AnimationScope<T>
}
```

## 6. Sealed Interface vs Sealed Class

### Sealed Class (классический)

```kotlin
sealed class Result {
    data class Success(val data: String) : Result()
    object Error : Result()
}
// Все подтипы должны быть в том же файле
```

### Sealed Interface (современный)

```kotlin
sealed interface Result {
    data class Success(val data: String) : Result
    object Error : Result
}
// Подтипы могут быть в разных файлах и пакетах!
```

**Преимущество sealed interface**: Можно реализовать в разных модулях/файлах. В проекте `NavigationData.kt` использует `sealed interface Destination`.

## 7. Inline functions

```kotlin
// inline — код лямбды встраивается прямо в место вызова (no overhead)
inline fun <T> runIf(condition: Boolean, block: () -> T): T? {
    return if (condition) block() else null
}

// noinline — если лямбда передаётся как параметр другой функции
inline fun example(crossinline callback: () -> Unit) { ... }

// reified — доступ к типу внутри инлайн-функции
inline fun <reified T> Gson.fromJson(json: String): T {
    returnfromJson(json, T::class.java)
}
```

## 8. Reflection (Рефлексия)

```kotlin
// Класс
val clazz = User::class
clazz.simpleName        // "User"
clazz.members           // Все члены класса

// Свойства
val prop = User::name
prop.get(user)          // Получить значение
prop.set(user, "Bob")   // Установить значение
```

## 9. Destructuring Declarations

```kotlin
data class User(val name: String, val age: Int, val email: String)

fun getUser(): User = User("Alex", 25, "alex@mail.com")

// Извлечение только нужных полей
val (name, _, email) = getUser()

// В цикле for
map.forEach { (key, value) ->
    println("$key -> $value")
}

// Сопоставление с паттернами
when (getUser()) {
    is User -> {
        val (name, age, _) = it
        println("$name ($age)")
    }
}
```

## 10. Annotation Processing vs Kotlin Symbol Processing (KSP)

| Технология | Описание |
|---|---|
| **Annotation Processor** | Java-аннотации, работает во время компиляции |
| **KSP (Kotlin Symbol Processing)** | Современная замена, поддерживает Kotlin |
| **kapt** | Старый процессор аннотаций для Kotlin (deprecated) |

В проекте используется KSP через `kotlinSerialization` плагин.
