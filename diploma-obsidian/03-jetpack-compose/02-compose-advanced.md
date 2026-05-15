# Jetpack Compose — Продвинутые темы

## 1. ViewModel + Compose

### Подключение ViewModel

```kotlin
@Composable
fun MyScreen() {
    val viewModel: MyViewModel = viewModel()  // из androidx.lifecycle:lifecycle-viewmodel-compose
    
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    
    when (state) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> Content(state.data)
        is UiState.Error -> ErrorScreen(state.message)
    }
}
```

### ViewModel с navigation3

```kotlin
// В AppNavigation.kt — навигация передаёт viewModel через entryProvider
entry<Destination.MainMenu> {
    val vm: MainMenuViewModel = viewModel()  // Scoped к NavBackStackEntry
    MainMenuPage(vm = vm, onProfileClick = { ... })
}
```

### ViewModel с параметрами из навигации

```kotlin
entry<Destination.ProjectDetailsPage> { navEntry ->
    val projectId = navEntry.projectId  // Извлекаем параметр
    
    val viewModel: ProjectDetailsViewModel = viewModel()
    
    LaunchedEffect(projectId) {
        viewModel.loadProject(projectId)
    }
}
```

## 2. Navigation3 (AndroidX Navigation v3)

### Destination — типы навигации

```kotlin
@Serializable
sealed interface Destination : NavKey {
    // Data object — без параметров
    @Serializable data object MainMenu : Destination
    
    // Data class — с параметрами
    @Serializable data class ProjectDetailsPage(val projectId: String) : Destination
}
```

### NavGraph (entryProvider)

```kotlin
val backStack = rememberNavBackStack(startDestination)

NavDisplay(
    backStack = backStack,
    entryProvider = entryProvider {
        entry<Destination.MainMenu> {
            MainMenuPage(onProjectClick = { id ->
                backStack += Destination.ProjectDetailsPage(id)  // Push
            })
        }
        
        entry<Destination.ProjectDetailsPage> { navEntry ->
            val projectId = navEntry.projectId
            
            ProjectDetailsPage(
                onBackClick = { backStack.removeLastOrNull() },  // Pop
                projectId = projectId
            )
        }
    }
)
```

### Операции навигации

| Операция | Код | Описание |
|---|---|---|
| **Push** | `backStack += Destination.X` | Добавить наверх стека |
| **Pop** | `backStack.removeLastOrNull()` | Удалить верхний элемент |
| **Clear + Push** | `backStack.clear(); backStack += X` | Очистить стек → перейти |
| **Replace** | `backStack.clear(); backStack += Destination.MainMenu` | Заменить весь стек |

### Вложенная навигация (Nested Nav)

В проекте используется в `MainMenu`:
```kotlin
// Главный стек: RegPage, MainMenu, ProfilePage...
val backStack = rememberNavBackStack(startDestination)

// Внутри MainMenu — свой стек для табов
val tabBackStack = rememberNavBackStack(Destination.MainMenu)

Scaffold(
    bottomBar = { NavigationBar { ... } },  // Bottom nav
) {
    NavDisplay(backStack = tabBackStack, entryProvider = { ... })
}
```

## 3. Hilt (Dependency Injection)

### Базовая настройка

```kotlin
// Module — как создавать зависимости
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BASE_URL)
            .build()
    }
    
    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ModrinthApiService {
        return retrofit.create(ModrinthApiService::class.java)
    }
}

// Использование в Activity/Fragment
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    @Inject lateinit var apiService: ModrinthApiService
}
```

### Compose + Hilt

```kotlin
@Composable
fun MyScreen(
    viewModel: MyViewModel = hiltViewModel()  // Hilt ViewModel
) { ... }
```

## 4. Custom Composables (Создание своих компонентов)

### Pattern: Event + State

```kotlin
// Состояние (data class)
data class LoginState(
    val email: String = "",
    val password: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)

// События (sealed interface)
sealed interface LoginEvent {
    data class EmailChanged(val email: String) : LoginEvent
    data class PasswordChanged(val password: String) : LoginEvent
    object OnLoginClick : LoginEvent
}

// Компонент принимает state и event handler
@Composable
fun LoginForm(state: LoginState, onEvent: (LoginEvent) -> Unit) {
    Column {
        TextField(
            value = state.email,
            onValueChange = { onEvent(LoginEvent.EmailChanged(it)) },
            label = { Text("Email") }
        )
        
        TextField(
            value = state.password,
            onValueChange = { onEvent(LoginEvent.PasswordChanged(it)) },
            label = { Text("Password") },
            visualTransformation = PasswordVisualTransformation()
        )
        
        if (state.isLoading) CircularProgressIndicator()
        if (state.error != null) Text(state.error, color = Color.Red)
        
        Button(onClick = { onEvent(LoginEvent.OnLoginClick) }) {
            Text("Login")
        }
    }
}
```

### Pattern: Slot API (как Scaffold)

```kotlin
@Composable
fun CustomCard(
    modifier: Modifier = Modifier,
    title: @Composable () -> Unit,         // Slot для заголовка
    content: @Composable () -> Unit,        // Slot для контента
    actions: @Composable RowScope.() -> Unit = {}  // Slot с RowScope
) {
    Card(modifier = modifier) {
        Column {
            title()     // Рендерим slot заголовка
            content()   // Рендерим slot контента
            Row { actions() }  // Рендерим slot действий
        }
    }
}

// Использование
CustomCard(
    title = { Text("My Card") },
    content = { Text("Content here") },
    actions = {
        Button(onClick = {}) { Text("OK") }
        TextButton(onClick = {}) { Text("Cancel") }
    }
)
```

## 5. Performance Optimization

### Remember — кэширование вычислений

```kotlin
@Composable
fun ExpensiveList(items: List<String>) {
    // Без remember — пересчитывается при каждой перерисовке!
    val sorted = items.sorted()  
    
    // С remember — только когда items изменились
    val sortedItems = remember(items) { items.sorted() }
    
    LazyColumn { ... }
}
```

### key() — стабильная идентификация

```kotlin
@Composable
fun ThemeSwitcher(isDark: Boolean, content: @Composable () -> Unit) {
    // При переключении темы — полностью перестраиваем subtree
    key(isDark) {
        NexusForgeTheme(darkTheme = isDark) {
            content()
        }
    }
}
```

### Stability в Compose

Compose отслеживает стабильность параметров для оптимизации перерисовок:

| Тип | Стабильный? | Пример |
|---|---|---|
| `data class` | ✅ Да | `data class User(...)` |
| `sealed class/interface` | ✅ Да | `sealed interface Event` |
| `object` | ✅ Да | `object Singleton` |
| `interface` (без data) | ❌ Нет | `interface Api` |
| `class` (обычный) | ❌ Нет | `class User(...)` |
| `String`, `Int` | ✅ Да | Примитивы и стандартные типы |

### avoidRecomposition — когда не перерисовывать

```kotlin
@Composable
fun StableComponent(expensive: String, cheap: Int) {
    // Если expensive изменится, но cheap нет — всё равно перерисуется
}

// Оптимизация: стабильные данные в data class
data class ScreenState(val expensiveData: String, val cheapValue: Int)

@Composable
fun OptimizedComponent(state: ScreenState) {
    // Если state не изменился (data class сравнивает поля) — не перерисуется!
}
```

## 6. Accessibility (Доступность)

```kotlin
@Composable
function AccessibleButton() {
    Button(
        onClick = {},
        modifier = Modifier
            .semantics { 
                contentDescription = "Add new item"  // Описание для скринридеров
                role = Role.Button
            }
    ) {
        Icon(painter = painterResource(R.drawable.add), contentDescription = null)
        Text("Add")
    }
}

// Обработка свайпов
@Composable
fun SwipeToDelete(item: String, onDelete: () -> Unit) {
    Box(modifier = Modifier.swipeable(
        anchors = mapOf(0f to Reset, 100f to Delete),
        thresholds = { _, _ -> FractionalThreshold(0.3f) }
    )) {
        Text(item)
    }
}
```

## 7. Testing Compose UI

### Unit тесты ViewModel

```kotlin
@Test
fun `login with valid credentials shows success`() = runTest {
    val viewModel = LoginViewModel(authRepository)
    
    viewModel.onEvent(LoginEvent.EmailChanged("test@mail.com"))
    viewModel.onEvent(LoginEvent.PasswordChanged("password"))
    viewModel.onEvent(LoginEvent.OnLoginClick)
    
    viewModel.state.test {
        assertEquals(true, awaitItem().isLoading)
        val success = awaitItem()
        assertTrue(success is UiState.Success)
    }
}
```

### UI тесты (Compose Test Rule)

```kotlin
@get:Rule
val composeRule = createComposeRule()

@Test
fun loginButton_click_navigatesToHome() {
    composeRule.setContent { MyApp() }
    
    // Заполняем поля
    composeRule.onNodeWithText("Email").performTextInput("test@mail.com")
    composeRule.onNodeWithText("Password").performTextInput("password123")
    
    // Нажимаем кнопку
    composeRule.onNodeWithText("Login").performClick()
    
    // Проверяем навигацию
    composeRule.onNodeWithText("Home").assertIsDisplayed()
}
```

## 8. Material Design 3 в Compose

### Theme setup

```kotlin
// colors.kt — цвета приложения
private val Purple80 = Color(0xFFD0BCFF)
private val PurpleGrey80 = Color(0xFFCCC2DC)
private val Pink80 = Color(0xFFEFB8C8)

@Composable
fun NexusForgeTheme(darkTheme: Boolean, content: @Composable () -> Unit) {
    val colorScheme = when {
        darkTheme -> darkColorScheme(...)
        else -> lightColorScheme(...)
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,     // Шрифты из type.kt
        shapes = Shapes,             // Формы (скругления)
        content = content
    )
}
```

### Shapes (Формы/Скругления)

```kotlin
// В MaterialTheme.shapes доступны:
SmallCard()  // extraSmall — 4dp
MediumCard() // small — 8dp  
LargeCard()  // medium — 12dp
ExtraLargeCard() // large — 16dp
```

### Typography (Типографика)

```kotlin
// В MaterialTheme.typography доступны:
Text("Display Large", style = MaterialTheme.typography.displayLarge)
Text("Title Medium", style = MaterialTheme.typography.titleMedium)
Text("Body Small", style = MaterialTheme.typography.bodySmall)
```

## 9. Dynamic Color (Android 12+)

```kotlin
@Composable
fun NexusForgeTheme(darkTheme: Boolean, dynamicColor: Boolean = true) {
    val colorScheme = when {
        // Android 12+ с динамическими цветами из обоев
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) 
            else dynamicLightColorScheme(context)
        }
        
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }
    
    MaterialTheme(colorScheme = colorScheme, content = content)
}
```

## 10. Common Mistakes (Частые ошибки)

### ❌ Не hoist state наружу

```kotlin
// Плохо — каждый раз создаём новый mutableStateOf при перерисовке
@Composable
fun BadCounter() {
    val count = mutableStateOf(0)  // Создаётся заново!
}

// Хорошо — remember сохраняет между перерисовками
@Composable
fun GoodCounter() {
    var count by remember { mutableStateOf(0) }
}
```

### ❌ Не использовать remember с изменяемыми коллекциями

```kotlin
// Плохо — ссылка не меняется, Compose не узнает об изменении
@Composable
fun BadList() {
    val list = mutableListOf<String>()
    list.add("item")  // Не вызовет перерисовку!
}

// Хорошо — создаём новый список (data class immutable)
@Composable
fun GoodList() {
    var items by remember { mutableStateOf(listOf<String>()) }
    items = items + "new item"  // Новая ссылка → перерисовка!
}
```

### ❌ Запуск корутин без LaunchedEffect

```kotlin
// Плохо — запускается при каждой перерисовке!
@Composable
fun BadLoad() {
    lifecycleScope.launch { repository.loadData() }  // BAD!
}

// Хорошо — запускается только при создании/изменении ключа
@Composable
fun GoodLoad(id: String) {
    LaunchedEffect(id) { repository.loadById(id) }  // GOOD
}
```
