# Jetpack Compose — Основы

## 1. Что такое Jetpack Compose?

Jetpack Compose — современный declarative UI toolkit для Android. Вместо XML-разметки и `findViewById`, вы описываете UI как функцию, а Compose сама управляет обновлениями.

### Declarative vs Imperative

**Imperative (старый подход):**
```kotlin
// Вы вручную говорите КАК обновить UI
val textView = findViewById<TextView>(R.id.text)
textView.text = "Hello"
textView.visibility = View.VISIBLE
```

**Declarative (Compose):**
```kotlin
// Вы описываете ЧТО должно быть на экране
Text(text = "Hello")
```

## 2. @Composable функции

Любая функция, которая участвует в построении UI, должна быть аннотирована `@Composable`:

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello $name!")
}
```

### Правила Compose-функций

1. Должны вызываться из других `@Composable` или через Coroutine (`lifecycleScope.launch`)
2. Не должны возвращать UI-элементы — они их рисуют сами
3. Могут вызывать другие `@Composable` функции
4. Не должны быть `private` (Compose генерирует код для них)

## 3. Основные компоненты (Components)

### Текст и ввод

| Компонент | Назначение |
|---|---|
| `Text(text)` | Отображение текста |
| `TextField(value, onValueChange)` | Поле ввода |
| `OutlinedTextField` | Обведённое поле ввода |

```kotlin
@Composable
fun NameInput() {
    var name by remember { mutableStateOf("") }
    
    TextField(
        value = name,
        onValueChange = { name = it },
        label = { Text("Name") },
        placeholder = { Text("Enter your name") }
    )
}
```

### Кнопки

| Компонент | Описание |
|---|---|
| `Button(onClick)` | Основная кнопка с elevation |
| `TextButton(onClick)` | Плоская текстовая кнопка |
| `OutlinedButton(onClick)` | Обведённая кнопка |
| `IconButton(onClick)` | Кнопка-иконка |

```kotlin
@Composable
fun ActionButtons() {
    Button(onClick = { /* action */ }) {
        Icon(painter = painterResource(R.drawable.add), contentDescription = null)
        Text("Create")
    }
}
```

### Изображения

| Компонент | Описание |
|---|---|
| `Image(painter, contentDescription)` | Отображение изображения |
| `AsyncImage(model, contentDescription)` | Загрузка из сети (Coil) |

```kotlin
@Composable
fun Avatar(url: String) {
    AsyncImage(
        model = url,
        contentDescription = "Avatar",
        modifier = Modifier.size(48.dp),
        contentScale = ContentScale.Crop
    )
}
```

## 4. Layout (Расположение элементов)

### Row — горизонтальное расположение

```kotlin
@Composable
fun HorizontalRow() {
    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("Item 1")
        Text("Item 2")
        Text("Item 3")
    }
}
```

### Column — вертикальное расположение

```kotlin
@Composable
fun VerticalColumn() {
    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
        Text("Header")
        Text("Content")
        Text("Footer")
    }
}
```

### Box — наложение элементов

```kotlin
@Composable
fun ImageWithOverlay() {
    Box {
        Image(painter = painterResource(R.drawable.bg), contentDescription = null)
        Text(
            text = "Overlay",
            modifier = Modifier.align(Alignment.Center)
        )
    }
}
```

### ConstraintLayout — сложная разметка

```kotlin
@Composable
fun ComplexLayout() {
    ConstraintLayout(modifier = Modifier.fillMaxSize()) {
        val (header, content, footer) = createRefs()
        
        Text("Header", modifier = constrainAs(header) {
            top.linkTo(parent.top)
        })
        
        Text("Content", modifier = constrainAs(content) {
            top.linkTo(header.bottom)
            bottom.linkTo(footer.top)
        })
        
        Text("Footer", modifier = constrainAs(footer) {
            bottom.linkTo(parent.bottom)
        })
    }
}
```

### Scaffold — базовый layout с app bar, bottom nav

```kotlin
@Composable
fun MainScreen() {
    Scaffold(
        topBar = { TopAppBar(title = { Text("Title") }) },
        bottomBar = { NavigationBar { ... } },
        floatingActionButton = { FloatingActionButton(onClick = {}) { ... } }
    ) { padding ->
        // Контент с учётом отступов
        LazyColumn(modifier = Modifier.padding(padding)) {
            items(100) { index -> Item(index) }
        }
    }
}
```

## 5. Модификаторы (Modifiers)

Modifier — цепочка операций, применяемых к composables:

```kotlin
@Composable
fun StyledCard() {
    Box(
        modifier = Modifier
            .fillMaxWidth()           // Занять всю ширину
            .padding(16.dp)           // Внутренние отступы
            .aspectRatio(1f)          // Сохранить пропорции 1:1
            .clip(RoundedCornerShape(8.dp))  // Скруглить углы
            .background(Color.Blue)   // Фон
            .shadow(4.dp, RoundedCornerShape(8.dp))  // Тень
            .clickable { /* action */ }       // Кликабельность
    ) {
        Text("Card")
    }
}
```

### Популярные модификаторы

| Модификатор | Назначение |
|---|---|
| `Modifier.size(width, height)` | Фиксированный размер |
| `Modifier.fillMaxSize()` | Заполнить весь доступный space |
| `Modifier.wrapContentHeight()` | Обернуть содержимое (по умолчанию) |
| `Modifier.padding(innerPadding)` | Внутренние отступы |
| `Modifier.offset(x, y)` | Смещение |
| `Modifier.rotate(degrees)` | Поворот |
| `Modifier.scale(x, y)` | Масштабирование |
| `Modifier.animateContentSize()` | Анимация при изменении размера |

## 6. Lazy Lists (Ленивые списки)

### LazyColumn — вертикальный список

```kotlin
@Composable
fun ItemList(items: List<String>) {
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.spacedBy(8.dp),
        contentPadding = PaddingValues(16.dp)
    ) {
        items(items.size) { index ->
            Text(
                text = items[index],
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(vertical = 8.dp)
            )
        }
        
        // Разделитель между секциями
        item { Divider() }
        
        // Фиксированный элемент в конце
        item { Text("Footer") }
    }
}
```

### LazyVerticalGrid — сетка

```kotlin
@Composable
fun ModGrid(mods: List<Mod>) {
    LazyVerticalGrid(
        columns = GridCells.Fixed(2),  // 2 колонки
        horizontalArrangement = Arrangement.spacedBy(8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(mods, key = { it.id }) { mod ->
            ModCard(mod)
        }
    }
}
```

### Ключи (keys) в LazyList

```kotlin
// Используйте ключ для стабильной идентификации элементов
items(items, key = { it.id }) { item -> ... }

// Если нет уникального ID — используйте индекс
items(items.size) { index -> ... }
```

## 7. State Management (Управление состоянием)

### Что такое State в Compose?

State — данные, при изменении которых UI автоматически перерисовывается.

#### mutableStateOf (базовый state)

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Column {
        Text("Count: $count")
        Button(onClick = { count++ }) {
            Text("+1")
        }
    }
}
```

#### remember — кэширование значений

```kotlin
// Сохраняет значение между перерисовками
val value = remember { mutableStateOf(0) }

// С ключами — сбрасывается при изменении ключа
key(themeViewModel.isDarkTheme) {
    val theme by remember { mutableStateOf(...) }
}
```

#### StateHolder pattern (рекомендуемый)

```kotlin
@Composable
fun UserProfile() {
    // 1. Держим state в объекте
    var name by remember { mutableStateOf("") }
    var email by remember { mutableStateOf("") }
    
    val state = remember(name, email) { UserProfileState(name, email) }
    
    // 2. Используем state в UI
    TextField(value = state.name, onValueChange = { name = it })
}

data class UserProfileState(val name: String, val email: String)
```

### State Hoisting (Вынесение состояния)

Принцип: состояние поднимается на уровень выше, composables становятся pure.

```kotlin
// ❌ Компонент управляет своим состоянием (тightly coupled)
@Composable
fun NameField() {
    var text by remember { mutableStateOf("") }
    TextField(value = text, onValueChange = { text = it })
}

// ✅ Состояние hoisted наружу (loosely coupled)
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

## 8. Lifecycle в Compose

### LaunchedEffect — запуск корутин при изменении ключей

```kotlin
@Composable
fun LoadData(projectId: String) {
    // Запускается при создании + при изменении projectId
    LaunchedEffect(projectId) {
        val data = repository.loadProject(projectId)
        viewModel.update(data)
    }
}

// Запускается один раз
LaunchedEffect(Unit) {
    viewModel.init()
}
```

### DisposableEffect — очистка ресурсов

```kotlin
@Composable
fun SubscribeToAuth() {
    val auth = FirebaseAuth.getInstance()
    
    DisposableEffect(auth) {
        val listener = FirebaseAuth.AuthStateListener { /* ... */ }
        auth.addAuthStateListener(listener)
        
        // Очистка при удалении эффекта
        onDispose { auth.removeAuthStateListener(listener) }
    }
}
```

### SideEffect — синхронизация с платформой Android

```kotlin
@Composable
fun SetEdgeToEdge() {
    val activity = rememberActiveActivity()
    
    SideEffect {
        activity?.enableEdgeToEdge()
    }
}
```

## 9. Animation (Анимации)

### AnimatedVisibility — появление/исчезновение

```kotlin
@Composable
fun ToggleContent(show: Boolean) {
    AnimatedVisibility(visible = show) {
        Text("Visible!")
    }
}
```

### animate*AsState — анимация значений

```kotlin
@Composable
fun GrowingBox() {
    val scale by animateFloatAsState(
        targetValue = if (expanded) 1.5f else 1.0f,
        animationSpec = spring()
    )
    
    Box(modifier = Modifier.size(100.dp).scale(scale))
}
```

### Crossfade — плавный переход между экранами

```kotlin
@Composable
fun SwitchScreen(current: Int) {
    Crossfade(targetState = current) { screen ->
        when (screen) {
            0 -> ScreenOne()
            1 -> ScreenTwo()
        }
    }
}
```

### ThemeAnimator — анимация перехода тем

В проекте используется `themeanimator:0.0.19` для плавного переключения между светлой и тёмной темой.

## 10. Preview (Предпросмотр в IDE)

```kotlin
@Preview(showBackground = true)
@Preview(name = "Dark", uiMode = Configuration.UI_MODE_NIGHT_YES)
@Composable
fun GreetingPreview() {
    NexusForgeTheme(darkTheme = false) {
        Greeting("Preview")
    }
}
```

## 11. Composition Locals (Глобальные значения)

CompositionLocal — способ передать данные через tree composables без параметризации:

```kotlin
// Определение
val LocalDarkTheme = staticCompositionLocalOf<Boolean> { true }

// Предоставление
CompositionLocalProvider(LocalDarkTheme provides isDarkTheme) {
    // Весь дочерний доступ к LocalDarkTheme вернёт isDarkTheme
}

// Использование
val darkMode = LocalDarkTheme.current
```

В проекте используется в `CompositionLocal.kt` для передачи темы.

## 12. Testing Compose UI

```kotlin
@Composable
fun MyScreenTest() {
    val rule = createComposeRule()
    
    rule.setContent {
        MyApp()
    }
    
    // Проверка наличия элемента
    rule.onNodeWithText("Hello").assertIsDisplayed()
    
    // Взаимодействие
    rule.onNodeWithText("Button").performClick()
    
    // Проверка состояния
    rule.onNodeWithText("World").assert(hasText("World"))
}
```
