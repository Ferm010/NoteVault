# Зависимости проекта NexusForge — Подробный обзор

## 1. Build Configuration

### build.gradle.kts (root)

```kotlin
// Version Catalog (libs.versions.toml)
alias(libs.plugins.android.application) apply false
alias(libs.plugins.kotlin.android) apply false
alias(libs.plugins.kotlin.compose) apply false
alias(libs.plugins.google.services) apply false
alias(libs.plugins.firebase.crashlytics) apply false
```

**Version Catalog** — централизованное управление версиями в `gradle/libs.versions.toml`:
- Все зависимости объявлены один раз
- Изменение версии в одном месте применяется везде
- Автодополнение в IDE

### app/build.gradle.kts — Plugins

| Плагин | Назначение |
|---|---|
| `android.application` | Android Application plugin (AGP) |
| `kotlin.android` | Kotlin компилятор для Android |
| `kotlin.compose` | Compose compiler интеграция |
| `kotlinSerialization` | kotlinx.serialization плагин |
| `google.services` | Firebase Google Services (config.json → google-services.json) |
| `firebase.crashlytics` | Crashlytics сбор краш-репортов |

## 2. Navigation3

```kotlin
implementation(libs.androidx.navigation3.ui)
implementation(libs.androidx.navigation3.runtime)
implementation(libs.androidx.lifecycle.viewmodel.navigation3)
```

**Navigation3** — новая версия Android Navigation с:
- **Type-safe navigation** через `@Serializable sealed interface`
- **Compose-native** API (`NavDisplay`, `entryProvider`)
- **ViewModel scoping** к NavBackStackEntry
- **Корутины-first** подход

### Сравнение Navigation2 vs Navigation3

| Feature | Navigation 2 (Fragment) | Navigation 3 (Compose) |
|---|---|---|
| Язык | Kotlin + Safe Args | Kotlin + kotlinx.serialization |
| Компоненты | Fragment | Composable functions |
| Типобезопасность | Safe Args (XML generated) | Compiler-generated from sealed interface |
| ViewModel scope | Activity/Fragment | NavBackStackEntry |

## 3. Lifecycle & ViewModel

```kotlin
implementation(libs.androidx.lifecycle.viewmodel.compose)
implementation(libs.androidx.lifecycle.viewmodel.navigation3)
implementation(libs.androidx.lifecycle.runtime.ktx)
```

- **lifecycle-viewmodel-compose** — `viewModel()` для Compose, `collectAsStateWithLifecycle()`
- **lifecycle-runtime-ktx** — LifecycleObserver, автоматическое управление жизненным циклом

### collectAsStateWithLifecycle vs collectAsState

| Метод | Назначение |
|---|---|
| `collectAsState()` | Базовый — собирает Flow в State |
| `collectAsStateWithLifecycle()` | Умный — останавливает сборку когда экран не виден (saved on stop) |

**Рекомендация**: Всегда используйте `collectAsStateWithLifecycle()` для экономии ресурсов.

## 4. Compose BOM (Bill of Materials)

```kotlin
implementation(platform(libs.androidx.compose.bom))
implementation(libs.androidx.compose.ui)
implementation(libs.androidx.compose.material3)
implementation(libs.androidx.compose.material.icons.extended)
```

**BOM** — гарантирует совместимость всех Compose-библиотек:
- Не нужно указывать версии для `libs.androidx.compose.*`
- Все компоненты из одной версии BOM автоматически совместимы
- Обновление одного `compose.bom.version` обновляет все зависимости

## 5. Firebase

```kotlin
implementation(platform(libs.firebase.bom))
implementation(libs.firebase.auth)
implementation(libs.firebase.firestore)
implementation(libs.firebase.crashlytics)
implementation(libs.firebase.analytics)
implementation(libs.kotlinx.coroutines.play.services)
```

### Firebase BOM

Позволяет использовать любой компонент Firebase без указания версии — BOM управляет совместимостью.

| Компонент | Назначение в NexusForge |
|---|---|
| **Firebase Auth** | Email/Password + Google Sign-In |
| **Firestore** | Хранение профилей, избранных проектов, модпаков |
| **Crashlytics** | Сбор краш-репортов для отладки |
| **Analytics** | Аналитика использования (события) |

### Coroutines + Firebase

```kotlin
// kotlinx.coroutines.play.services — интеграция Firebase с корутинами
auth.createUserWithEmailAndPassword(email, password).await()  // await() из coroutines-play-services
```

Без этой библиотеки пришлось бы использовать `FirebaseCompletionListener` (callback-based API).

## 6. Google Sign-In (Credential Manager)

```kotlin
implementation(libs.credentials)
implementation(libs.credentials.play.services.auth)
implementation(libs.googleid)
```

**Credential Manager** — современная замена old Google Sign-In SDK:
- Единый API для всех провайдеров (Google, Apple, Email)
- Минимальный размер APK
- Поддержка Passkeys

### Flow в проекте

1. Пользователь нажимает "Sign in with Google"
2. `CredentialsClient.request()` → открывает выбор аккаунта
3. Получаем `IdTokenCredential` с ID токеном
4. Токен отправляется в Firebase: `GoogleAuthProvider.getCredential(idToken, null)`
5. Firebase аутентифицирует и возвращает пользователя

## 7. Coil — Image Loading

```kotlin
implementation("io.coil-kt:coil-compose:2.5.0")
```

**Coil** — image loading library на Kotlin с Coroutines:

```kotlin
@Composable
fun ModImage(url: String) {
    AsyncImage(
        model = url,
        contentDescription = "Mod icon",
        modifier = Modifier.size(48.dp),
        contentScale = ContentScale.Crop,
        placeholder = painterResource(R.drawable.placeholder),
        error = painterResource(R.drawable.error)
    )
}
```

### Почему Coil?

| Feature | Coil | Glide | Picasso |
|---|---|---|---|
| Язык | Kotlin | Java | Java |
| Coroutines | ✅ Да | ❌ Нет | ❌ Нет |
| Compose | ✅ Native | ⚠️ Через View | ❌ Нет |
| Размер | ~200KB | ~700KB | ~300KB |

## 8. Retrofit + OkHttp + kotlinx.serialization

```kotlin
implementation("com.squareup.retrofit2:retrofit:2.9.0")
implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
implementation("com.squareup.okhttp3:okhttp:4.12.0")
```

### Архитектура API вызовов

```
ModrinthApiService (interface)
    ↓ Retrofit creates implementation
retrofit2-kotlinx-serialization-converter (JSON parser)
    ↓ OkHttp handles HTTP
OkHttpClient (HTTP client with interceptors)
```

### Почему retrofit2-kotlinx-serialization-converter?

Обычный Retrofit использует Gson/Moshi. Этот конвертер позволяет использовать `kotlinx.serialization` для JSON — тот же формат, что и в остальном проекте.

```kotlin
// Модели данных
@Serializable
data class ModrinthProject(
    val id: String,
    val title: String,
    val description: String,
    val logoUrl: String?
)

@Serializable
data class ModrinthSearchResponse(
    val results: List<ModrinthProject>
)
```

## 9. Google Drive API

```kotlin
implementation("com.google.android.gms:play-services-auth:21.0.0")
implementation("com.google.apis:google-api-services-drive:v3-rev20240123-2.0.0") {
    exclude(group = "com.google.guava", module = "guava-jdk5")
}
implementation("com.google.api-client:google-api-client-android:2.2.0") {
    exclude(group = "org.apache.httpcomponents")
    exclude(group = "com.google.guava", module = "guava-jdk5")
}
```

### Зачем нужны excludes?

`guava-jdk5` — старая версия Guava, несовместимая с Android. Исключаем её, чтобы не было конфликтов.

### Google Drive в проекте (GoogleDriveRepository.kt)

1. Пользователь авторизуется через Google Sign-In
2. Получаем `GoogleAccountCredential`
3. Создаём `Drive` клиент
4. Загружаем `.mrpack` файл как `File` в Google Drive
5. Сохраняем ID файла в Firestore

## 10. Theme Animator

```kotlin
implementation("io.github.gleb-skobinsky:themeanimator:0.0.19")
```

**ThemeAnimator** — библиотека для плавной анимации переключения между светлой и тёмной темой:
- Интерполяция цветов между темами
- Анимация всех UI элементов при смене темы
- Используется в `NexusForgeTheme` + `ThemeViewModel`

## 11. OSS Licenses

```kotlin
implementation(libs.play.services.oss.licenses)
classpath("com.google.android.gms:oss-licenses-plugin:0.10.6")
```

Автоматически генерирует список open-source зависимостей с лицензиями для экрана `OssLicensesPage`.

## 12. Testing Dependencies

```kotlin
testImplementation(libs.junit)                        // Unit тесты
androidTestImplementation(libs.androidx.junit)        // Android instrumentation tests
androidTestImplementation(libs.androidx.espresso.core) // UI тесты (View-based)
androidTestImplementation(platform(libs.androidx.compose.bom))
androidTestImplementation(libs.androidx.compose.ui.test.junit4)  // Compose UI тесты
debugImplementation(libs.androidx.compose.ui.tooling)          // Preview tooling
debugImplementation(libs.androidx.compose.ui.test.manifest)    // Test manifest
```

## 13. Android Configuration

### compileSdk = 36, targetSdk = 36

Последняя доступная версия SDK для разработки.

### minSdk = 26 (Android 8.0)

Минимальная поддерживаемая версия — Android 8.0 (Oreo). Это означает:
- Нет поддержки Android 7.x и ниже (~5% устройств)
- Можно использовать современные API без backport'ов

### Kotlin JVM Target = 11

Код компилируется в bytecode Java 11 совместимый.

## 14. Signing (Подпись приложения)

```kotlin
signingConfigs {
    create("release") {
        storeFile = file("../nexusforge.keystore")
        storePassword = properties.getProperty("KEYSTORE_PASSWORD", "")
        keyAlias = "nexusforge"
        keyPassword = properties.getProperty("KEY_PASSWORD", "")
    }
}
```

- **Keystore** хранится в корне проекта (`../nexusforge.keystore`)
- **Пароли** — в `local.properties`, не коммитятся в git
- **Build types**: release (с подписью) и debug (без minification)

## 15. Полный список зависимостей

| Категория | Зависимость | Версия | Назначение |
|---|---|---|---|
| Navigation | androidx.navigation3 | latest | Навигация Compose |
| Lifecycle | androidx.lifecycle | latest | ViewModel, Lifecycle |
| Compose BOM | androidx.compose.bom | latest | Управление версиями Compose |
| Material 3 | androidx.compose.material3 | — | UI компоненты MD3 |
| Firebase BOM | firebase.bom | latest | Управление версиями Firebase |
| Firebase Auth | firebase.auth | via BOM | Авторизация |
| Firestore | firebase.firestore | via BOM | Облачная БД |
| Crashlytics | firebase.crashlytics | via BOM | Краш-репорты |
| Analytics | firebase.analytics | via BOM | Аналитика |
| Coroutines | kotlinx.coroutines.play.services | — | Firebase + Coroutines |
| Credential Manager | credentials, googleid | latest | Google Sign-In |
| Image Loading | coil-compose | 2.5.0 | Загрузка изображений |
| HTTP Client | retrofit | 2.9.0 | REST API клиент |
| JSON Converter | retrofit2-kotlinx-serialization-converter | 1.0.0 | Kotlinx serialization для Retrofit |
| HTTP Engine | okhttp | 4.12.0 | Базовый HTTP |
| Google Drive | google-api-services-drive | v3-rev20240123-2.0.0 | Drive API |
| Theme Animator | themeanimator | 0.0.19 | Анимация тем |
| OSS Licenses | play.services.oss.licenses | — | Open source лицензии |
