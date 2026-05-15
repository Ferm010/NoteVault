# Jetpack Compose — Вопросы и ответы для экзамена

## Базовые вопросы

### 1. Что такое Jetpack Compose? Чем он отличается от View system?

**Ответ:** Jetpack Compose — declarative UI toolkit для Android, представленный в 2021 году.

| Характеристика | View System (XML) | Jetpack Compose |
|---|---|---|
| Подход | Imperative (ручное обновление) | Declarative (описание состояния) |
| Язык | XML + Kotlin/Java | Только Kotlin |
| Размер APK | Меньше | Немного больше (compose runtime) |
| Производительность | Хорошая (с оптимизациями) | Лучше (smart recomposition) |
| Анимации | Сложные, много кода | Простые, декларативные |
| Preview в IDE | Layout Inspector | Live preview с темами |

**Imperative vs Declarative:**
```kotlin
// Imperative — КАК рисовать
val text = findViewById<TextView>(R.id.text)
text.text = "Hello"
text.visibility = View.VISIBLE
text.setTextColor(Color.RED)

// Declarative — ЧТО рисовать
Text(
    text = "Hello",
    visibility = if (show) Visible else Gone,  // Не поддерживается напрямую
    color = Color.Red
)
```

### 2. Что такое @Composable функция? Какие у неё ограничения?

**Ответ:** `@Composable` функция — функция, которая описывает часть UI. Compose вызывает её для построения дерева компонентов.

**Ограничения:**
1. Должна вызываться из другой `@Composable` или корутины
2. Не может быть `private` (Compose генерирует код)
3. Не должна возвращать UI элементы — она их рисует сама
4. Не должна бросать исключения (нужно обрабатывать внутри)

```kotlin
@Composable  // ✅ Правильно
fun Greeting(name: String) {
    Text("Hello $name!")  // Рисуем, не возвращаем
}

// ❌ Неправильно — возвращаем UI элемент
@Composable
fun createButton(): Button { 
    return Button(...)  // Так нельзя!
}
```

### 3. В чём разница между Row, Column и Box?

**Ответ:**

| Layout | Расположение | Когда использовать |
|---|---|---|
| `Row` | Горизонтально (слева направо) | Кнопки в ряд, иконка + текст |
| `Column` | Вертикально (сверху вниз) | Список элементов, форма |
| `Box` | Наложение друг на друга | Карточка с текстом поверх изображения |

```kotlin
// Row — горизонтально
Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
    Icon(painterResource(R.drawable.add), null)
    Text("Add")
}

// Column — вертикально
Column(verticalArrangement = Arrangement.spacedBy(16.dp)) {
    TextField(...)
    Button(onClick = {}) { Text("Submit") }
}

// Box — наложение
Box {
    Image(painterResource(R.drawable.bg), null)  // Фон
    Text("Overlay", modifier = Modifier.align(Alignment.Center))  // Поверх
}
```

### 4. Что такое Modifier? Перечислите основные модификаторы.

**Ответ:** `Modifier` — цепочка операций, применяемых к composables для изменения их поведения и внешнего вида.

| Модификатор | Назначение |
|---|---|
| `fillMaxSize()` | Заполнить весь доступный space |
| `padding(16.dp)` | Внутренние отступы |
| `size(48.dp)` | Фиксированный размер |
| `clip(RoundedCornerShape(8.dp))` | Скруглить углы |
| `background(Color.Blue)` | Задать фон |
| `clickable {}` | Сделать кликабельным |
| `shadow(4.dp)` | Добавить тень |
| `aspectRatio(1f)` | Сохранить пропорции |

```kotlin
@Composable
fun StyledCard() {
    Box(modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp)
        .clip(RoundedCornerShape(8.dp))
        .background(Color.White)
        .shadow(4.dp)
        .clickable { /* action */ }
    ) {
        Text("Card")
    }
}
```

### 5. Что такое remember и mutableStateOf?

**Ответ:** 

- `remember {}` — кэширует значение между перерисовками composables
- `mutableStateOf(value)` — создаёт observable state, при изменении которого UI автоматически перерисовывается

```kotlin
@Composable
fun Counter() {
    // 1. remember сохраняет ссылку между перерисовками
    // 2. mutableStateOf делает значение observable
    var count by remember { mutableStateOf(0) }
    
    Column {
        Text("Count: $count")  // Перерисуется при изменении count!
        Button(onClick = { count++ }) { Text("+1") }
    }
}
```

**Паттерн деструктуризации**: `var count by remember { mutableStateOf(0) }` — позволяет использовать `count++` вместо `count.value++`.

## Вопросы на понимание Compose

### 6. Что такое State Hoisting? Почему это важно?

**Ответ:** State Hoisting — вынесение состояния из composables на уровень выше, чтобы сделать компонент предсказуемым и переиспользуемым.

```kotlin
// ❌ Плохо — компонент управляет своим состоянием (непредсказуем)
@Composable
fun NameField() {
    var text by remember { mutableStateOf("") }
    TextField(value = text, onValueChange = { text = it })
}

// ✅ Хорошо — состояние hoisted наружу (предсказуем, тестируем)
@Composable
fun NameField(value: String, onValueChange: (String) -> Unit) {
    TextField(value = value, onValueChange = onValueChange)
}

// Родитель управляет состоянием
@Composable
fun Parent() {
    var name by remember { mutableStateOf("") }
    NameField(value = name, onValueChange = { name = it })
}
```

**Преимущества hoisted state:**
- Компонент становится pure (одинаковый input → одинаковый output)
- Легче тестировать
- Можно использовать в разных контекстах
- Родитель контролирует lifecycle состояния

### 7. В чём разница между LaunchedEffect и DisposableEffect?

| Эффект | Когда срабатывает | Когда удаляется | Для чего |
|---|---|---|---|
| `LaunchedEffect(key)` | При создании + при изменении key | Когда composable удалён или key изменился | Запуск корутин, загрузка данных |
| `DisposableEffect(key)` | При создании + при изменении key | Когда composable удалён или key изменился | Регистрация/отписка от слушателей |

```kotlin
// LaunchedEffect — загрузка данных
@Composable
fun LoadProject(projectId: String) {
    LaunchedEffect(projectId) {  // Перезапускается при изменении projectId
        val project = repository.loadProject(projectId)
        viewModel.update(project)
    }
}

// DisposableEffect — подписка на слушателя
@Composable
fun SubscribeToAuth() {
    val auth = FirebaseAuth.getInstance()
    
    DisposableEffect(auth) {
        val listener = AuthStateListener { /* ... */ }
        auth.addAuthStateListener(listener)
        
        // Очистка при удалении эффекта
        onDispose { auth.removeAuthStateListener(listener) }
    }
}
```

### 8. Что такое LazyColumn и когда его использовать?

**Ответ:** `LazyColumn` — ленивый вертикальный список, который создаёт элементы только когда они видны на экране:

```kotlin
@Composable
fun ItemList(items: List<String>) {
    LazyColumn(
        verticalArrangement = Arrangement.spacedBy(8.dp),
        contentPadding = PaddingValues(16.dp)  // Отступы от краёв
    ) {
        items(items.size) { index ->
            Text(text = items[index])
        }
        
        // Фиксированный элемент
        item { Divider() }
        
        // С key для стабильной идентификации
        items(mods, key = { it.id }) { mod ->
            ModCard(mod)
        }
    }
}
```

**Когда использовать LazyColumn:**
- Списки с большим количеством элементов (10+)
- Динамические списки (добавление/удаление)
- Элементы с тяжёлой отрисовкой

### 9. Что такое Scaffold и из каких частей он состоит?

**Ответ:** `Scaffold` — базовый layout по стандарту Material Design, предоставляет области для всех стандартных элементов:

```kotlin
@Composable
fun MainScreen() {
    Scaffold(
        topBar = { 
            TopAppBar(title = { Text("NexusForge") })
        },
        bottomBar = { 
            NavigationBar { ... }
        },
        floatingActionButton = {
            FloatingActionButton(onClick = {}) { Icon(...) }
        },
        snackbarHost = { SnackbarHost(hostState) },
        drawer = { DrawerContent() }  // Опционально
    ) { padding ->
        // Контент с учётом отступов от topBar, bottomBar и FAB
        LazyColumn(modifier = Modifier.padding(padding)) { ... }
    }
}
```

**Области Scaffold:**
- `topBar` — верхняя панель (AppBar)
- `bottomBar` — нижняя панель (NavigationBar, BottomSheet)
- `floatingActionButton` — плавающая кнопка
- `snackbarHost` — область для snackbar сообщений
- `drawer` — боковое меню (NavigationDrawer)

### 10. Как работает анимация в Compose? Перечислите основные типы.

**Ответ:** Анимации в Compose декларативные — вы описываете начальное и конечное состояние, а Compose сам интерполирует:

| Тип | Назначение |
|---|---|
| `AnimatedVisibility` | Появление/исчезновение composables |
| `animate*AsState` | Анимация числовых значений (float, int, color) |
| `crossfade` | Плавный переход между composables |
| `animateContentSize` | Анимация при изменении размера |
| `AnimatedContent` | Анимация при смене контента по ключу |

```kotlin
// AnimatedVisibility — появление/исчезновение
@Composable
fun Toggle(show: Boolean) {
    AnimatedVisibility(visible = show) {
        Text("I appeared!")
    }
}

// animateFloatAsState — анимация значения
@Composable
fun GrowingBox(expanded: Boolean) {
    val scale by animateFloatAsState(
        targetValue = if (expanded) 1.5f else 1.0f,
        animationSpec = spring()  // Spring animation
    )
    
    Box(modifier = Modifier.size(100.dp).scale(scale))
}

// crossfade — переход между экранами
@Composable
fun SwitchScreen(screen: Int) {
    Crossfade(targetState = screen) { s ->
        when (s) { 0 -> ScreenOne(); 1 -> ScreenTwo() }
    }
}
```

## Продвинутые вопросы

### 11. Что такое recomposition? Как оптимизировать перерисовки?

**Ответ:** Recomposition — процесс повторной отрисовки composables при изменении state. Compose умный и перерисовывает только те composables, чьи зависимости изменились.

**Оптимизации:**

1. **remember** — кэшировать вычисления
```kotlin
val sorted = remember(items) { items.sorted() }  // Не сортировать при каждой recomposition
```

2. **key()** — стабильная идентификация subtree
```kotlin
key(isDarkTheme) { NexusForgeTheme(darkTheme = isDarkTheme) { content() } }
```

3. **Стабильные типы** — data class, sealed class, primitive types
```kotlin
// Stable — Compose не будет перерисовывать если ссылка не изменилась
data class UiState(val loading: Boolean, val data: String?)

@Composable
fun Screen(state: UiState) { ... }  // Перерисуется только при изменении state
```

4. **Разбиение на composables** — изолировать изменения
```kotlin
// Если изменился только header — не перерисуем content!
@Composable
fun Screen() {
    Header()   // Отдельный composable
    Content()  // Не перерисуется при изменении заголовка
}
```

### 12. В чём разница между collectAsState и collectAsStateWithLifecycle?

**Ответ:**

| Характеристика | `collectAsState()` | `collectAsStateWithLifecycle()` |
|---|---|---|
| Сбор Flow | Всегда, даже когда экран не виден | Останавливается при stop экрана |
| Экономия ресурсов | ❌ Нет | ✅ Да — экономит батарею и CPU |
| Сохранение состояния | ❌ Нет | ✅ Да — сохраняет последнее значение |
| Когда использовать | Редко, для простых случаев | **Всегда** для UI |

```kotlin
// ✅ Правильно — останавливается когда экран не виден
val state by viewModel.uiState.collectAsStateWithLifecycle()

// ❌ Плохо — продолжает собирать даже на background
val state by viewModel.uiState.collectAsState()
```

### 13. Что такое Composition Locals? Зачем они нужны?

**Ответ:** `CompositionLocal` — способ передать значение через дерево composables без параметризации каждого уровня:

```kotlin
// Определение (обычно в ui/theme/CompositionLocals.kt)
val LocalDarkTheme = staticCompositionLocalOf<Boolean> { true }

// Предоставление значения
CompositionLocalProvider(LocalDarkTheme provides isDarkTheme) {
    // Все дочерние composables могут читать LocalDarkTheme.current
}

// Чтение значения
val darkMode = LocalDarkTheme.current

// В теме MaterialTheme уже предоставляет:
val colors = MaterialTheme.colorScheme  // Через CompositionLocal
```

**Пример из проекта:** `CompositionLocalProvider(LocalDarkTheme provides themeViewModel.isDarkTheme)` — передача темы через всё дерево.

### 14. Как тестировать Compose UI?

**Ответ:**

**Unit тесты ViewModel:**
```kotlin
@Test
fun login_validCredentials_showSuccess() = runTest {
    val vm = LoginViewModel(repo)
    vm.onEvent(EmailChanged("test@mail.com"))
    vm.onEvent(PasswordChanged("pass123"))
    vm.onEvent(LoginClicked)
    
    assertEquals(UiState.Success, vm.state.value)
}
```

**UI тесты (Compose Test Rule):**
```kotlin
@get:Rule val composeRule = createComposeRule()

@Test
fun loginButton_click_navigatesToHome() {
    composeRule.setContent { MyApp() }
    
    // Ввод данных
    composeRule.onNodeWithText("Email").performTextInput("test@mail.com")
    composeRule.onNodeWithText("Password").performTextInput("pass123")
    
    // Нажатие кнопки
    composeRule.onNodeWithText("Login").performClick()
    
    // Проверка результата
    composeRule.onNodeWithText("Home").assertIsDisplayed()
}
```

### 15. Что такое Material Design 3 в Compose?

**Ответ:** MD3 (Material You) — дизайн-система Google с динамическими цветами:

```kotlin
@Composable
fun NexusForgeTheme(darkTheme: Boolean, content: @Composable () -> Unit) {
    val colorScheme = when {
        darkTheme -> darkColorScheme(
            primary = Purple80,
            secondary = PurpleGrey80,
            tertiary = Pink80
        )
        else -> lightColorScheme(
            primary = Purple40,
            secondary = PurpleGrey40,
            tertiary = Pink40
        )
    }
    
    MaterialTheme(
        colorScheme = colorScheme,  // Цвета
        typography = Typography,     // Шрифты (type.kt)
        shapes = Shapes,             // Скругления (shapes.kt)
        content = content
    )
}
```

**Dynamic Color (Android 12+):** Цвета берутся из обоев системы — приложение "подстраивается" под стиль пользователя.

## Вопросы по проекту NexusForge

### 16. Как в проекте организована навигация? Почему используется Navigation3?

**Ответ:** Навигация через `Navigation3` с `sealed interface Destination`:
- Type-safe: компилятор знает все экраны и их параметры
- kotlinx.serialization для сериализации параметров
- ViewModel scoped к NavBackStackEntry (не к Activity)
- Вложенная навигация в MainMenu (tabBackStack)

### 17. Как в проекте используется State Management?

**Ответ:** 
- **ViewModel** хранит state как `StateFlow`/`MutableStateFlow`
- UI получает state через `collectAsStateWithLifecycle()`
- События передаются через sealed interface или lambda callbacks
- Theme управляется через `ThemeViewModel` + `LaunchedEffect`

### 18. Как в проекте реализована смена темы?

**Ответ:** 
1. `ThemeViewModel` хранит `isDarkTheme: StateFlow<Boolean>`
2. `LaunchedEffect(Unit)` инициализирует тему при старте (сохраняет preference)
3. `key(themeViewModel.isDarkTheme)` — при переключении полностью перестраивается дерево тем
4. `CompositionLocalProvider(LocalDarkTheme provides ...)` — передача темы в subtree
5. `themeanimator:0.0.19` — плавная анимация перехода между темами

### 19. Как в проекте используются Lazy lists? Где?

**Ответ:** В проектах обычно используется `LazyColumn` для:
- Списка модов на MainMenuPage
- Избранных проектов на FavoritePage
- Списка шаблонов на TemplatesListPage

Каждый элемент имеет `key` для стабильной идентификации при обновлениях.

### 20. Как в проекте реализована загрузка изображений?

**Ответ:** Через Coil (`io.coil-kt:coil-compose:2.5.0`):
```kotlin
AsyncImage(
    model = imageUrl,
    contentDescription = "Mod icon",
    modifier = Modifier.size(48.dp),
    contentScale = ContentScale.Crop
)
```

Coil использует Coroutines для загрузки и автоматически кэширует изображения.
