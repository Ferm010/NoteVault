# Зависимости и Firebase — Вопросы и ответы для экзамена

## Базовые вопросы

### 1. Что такое Version Catalog (libs.versions.toml)? Зачем он нужен?

**Ответ:** Version Catalog — способ централизованного управления версиями зависимостей в Gradle:

```toml
# gradle/libs.versions.toml
[versions]
composeBom = "2024.01.0"
firebaseBom = "32.7.0"
retrofit = "2.9.0"

[libraries]
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref("composeBom") }
firebase-auth = { group = "com.google.firebase", name = "firebase-auth", version.ref("firebaseBom") }

[plugins]
android-application = { id = "com.android.application", version = "8.2.0" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version = "1.9.20" }
```

**Преимущества:**
- Единое место для всех версий
- Автодополнение в IDE (Ctrl+Space)
- Изменение версии в одном месте → применяется везде
- Группировка по категориям (plugins, libraries, versions)

### 2. Что такое BOM (Bill of Materials)? Как работает?

**Ответ:** BOM — файл с перечнем совместимых версий библиотек одного вендора. При использовании `platform(libs.xxx.bom)` Gradle автоматически подставляет правильные версии всех зависимостей из этого BOM:

```kotlin
// Указываем только платформу, версия берётся из BOM
implementation(platform(libs.firebase.bom))
implementation(libs.firebase.auth)      // Версия из firebase.bom
implementation(libs.firebase.firestore) // Версия из firebase.bom
implementation(libs.firebase.crashlytics)

// Без BOM пришлось бы указывать каждую версию отдельно!
```

### 3. В чём разница между implementation, testImplementation и debugImplementation?

| Конфигурация | Когда используется | Видимость |
|---|---|---|
| `implementation` | Основная сборка (release + debug) | Зависимость видима другим модулям |
| `api` | Библиотеки, которые экспортируются наружу | То же что implementation, но для библиотек |
| `testImplementation` | Только unit-тесты (`src/test/`) | Не попадает в APK |
| `androidTestImplementation` | Только instrumentation тесты (`src/androidTest/`) | Не попадает в APK |
| `debugImplementation` | Только debug сборка | Попадает только в debug APK |

```kotlin
// Зависит от Coil только для UI — не нужна в тестах
implementation("io.coil-kt:coil-compose:2.5.0")

// JUnit нужен только для тестов
testImplementation(libs.junit)

// Tooling только для debug (не в release!)
debugImplementation(libs.androidx.compose.ui.tooling)
```

## Вопросы по конкретным зависимостям

### 4. Почему в проекте используется Navigation3 вместо Navigation2?

**Ответ:**

| Feature | Navigation 2 | Navigation 3 |
|---|---|---|
| Компоненты | Fragment-based | Compose-native |
| Типобезопасность | Safe Args (XML generated) | Compiler-generated из sealed interface |
| ViewModel scope | Activity/Fragment | NavBackStackEntry |
| API стиль | Callback-based | Coroutine-first |
| Size | Больше (нужен Fragment) | Меньше (только Compose) |

Navigation3 — более современный подход, специально созданный для Compose. В проекте используется `sealed interface Destination` для type-safe навигации.

### 5. Как работает Firebase Auth в приложении? Опишите flow.

**Ответ:** 

1. **Регистрация Email/Password**:
   - Пользователь вводит email → `AuthRepository.checkEmailExists()` пытается создать аккаунт с временным паролем
   - Если email свободен — удаляем временный аккаунт, переходим к EULA
   - После принятия EULA — ввод пароля и имени → `registerUser()`

2. **Google Sign-In**:
   - Credential Manager → выбор Google аккаунта
   - Получаем ID Token → Firebase `GoogleAuthProvider.getCredential(idToken)`
   - Если новый пользователь → EULA → MainMenu
   - Если существующий → сразу MainMenu

3. **Проверка сессии**:
   - При старте → `FirebaseAuth.getInstance().currentUser`
   - Если есть user → проверяем профиль в Firestore
   - Профиль найден → MainMenu, нет → signOut → RegPage

### 6. Зачем нужен retrofit2-kotlinx-serialization-converter? Почему не обычный Gson?

**Ответ:** Обычный Retrofit использует Gson/Moshi для JSON. Этот конвертер позволяет использовать `kotlinx.serialization` — тот же формат сериализации, что и в остальном проекте:

```kotlin
// Модели с @Serializable — один подход везде!
@Serializable
data class ModrinthProject(val id: String, val title: String)

@Serializable
data class ModrinthSearchResponse(val results: List<ModrinthProject>)

// Retrofit + kotlinx.serialization converter
private val retrofit = Retrofit.Builder()
    .baseUrl(BASE_URL)
    .addConverterFactory(json.asConverterFactory("application/json"))
    .build()

interface ModrinthApiService {
    @GET("search")
    suspend fun searchProjects(@Query("query") query: String): ModrinthSearchResponse
}
```

**Преимущества:**
- Единый подход к сериализации во всём проекте
- Compile-time проверка (ошибки ловятся на компиляции)
- Меньше runtime overhead чем Gson (reflection)
- Kotlin-first дизайн (nullable, default values)

### 7. Как работает Coil для загрузки изображений? Почему он лучше Glide/Picasso?

**Ответ:** Coil — image loading library написанная полностью на Kotlin с Coroutines:

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

**Почему Coil лучше:**
- Kotlin-first — нет Java boilerplate
- Coroutines-native — не блокирует main thread
- Compose-native — `AsyncImage` для Compose (Glide/Picasso требуют View wrapper)
- Меньший размер APK (~200KB vs ~700KB у Glide)
- Auto-caching с Coil ImageLoader

### 8. Как работает Google Drive API в проекте?

**Ответ:** 

1. Пользователь авторизуется через Google Sign-In (Credential Manager)
2. Получаем `GoogleAccountCredential` с scope `DriveFile`
3. Создаём `com.google.api.services.drive.Drive` клиент
4. Загружаем `.mrpack` файл как `File` в Drive
5. Сохраняем ID файла в Firestore (`modpacks/{id}/driveFileId`)

```kotlin
// GoogleDriveRepository.kt
val drive = Drive.Builder(httpTransport, jsonFactory, credential)
    .setApplicationName("NexusForge")
    .build()

val fileMetadata = FileMetadata().setTitle(modpackName)
val content = ByteArrayContent("application/octet-stream", mrpackBytes)
drive.files().insert(fileMetadata, content).execute()
```

### 9. Что делает kotlinx.coroutines.play.services? Зачем она Firebase?

**Ответ:** Эта библиотека добавляет Coroutines-обёртки над Firebase `Task<T>`:

```kotlin
// Без coroutines — callback-based (ад колбэков)
auth.signInWithEmailAndPassword(email, password)
    .addOnSuccessListener { /* success */ }
    .addOnFailureListener { e -> /* error */ }

// С coroutines — чистый код!
try {
    auth.signInWithEmailAndPassword(email, password).await()  // await() из play-services
    // Success
} catch (e: FirebaseAuthException) {
    // Error
}
```

Без этой библиотеки пришлось бы использовать callback-based Firebase API или писать свои конвертеры.

### 10. Зачем нужен Theme Animator? Как он работает?

**Ответ:** `themeanimator:0.0.19` — библиотека для плавной анимации перехода между светлой и тёмной темой:

```kotlin
// При переключении темы в SettingsPage
ThemeAnimator.animateColorScheme(
    from = currentColorScheme,
    to = targetColorScheme,
    duration = 500
) { animatedColors ->
    // Обновляем тему с анимированными цветами
}
```

Без анимации — тема переключается мгновенно (flash). С анимацией — плавный переход (~500ms).

## Вопросы по архитектуре зависимостей

### 11. Почему зависимости разделены на категории? Как это помогает?

**Ответ:** Разделение на категории помогает:
- **Понимать архитектуру**: сразу видно какие библиотеки за что отвечают
- **Контролировать размер APK**: тестовые и debug-зависимости не попадают в release
- **Обновлять безопасно**: можно обновить одну категорию (например, Firebase BOM) без влияния на другие

### 12. Почему minSdk = 26? Какие это даёт преимущества?

**Ответ:** Android 8.0 (Oreo):
- Нет поддержки ~5% устройств с Android 7.x и ниже
- Можно использовать современные API без backport'ов (WorkManager, NotificationChannels)
- Better performance — Oreo оптимизирован для modern hardware
- Coroutines работают стабильно на всех версиях >= 26

### 13. Как устроена подпись приложения? Почему пароли в local.properties?

**Ответ:** 

```kotlin
// app/build.gradle.kts
signingConfigs {
    create("release") {
        storeFile = file("../nexusforge.keystore")
        storePassword = properties.getProperty("KEYSTORE_PASSWORD", "")  // Из local.properties
        keyAlias = "nexusforge"
        keyPassword = properties.getProperty("KEY_PASSWORD", "")
    }
}
```

**local.properties не коммитится в git** (указан в `.gitignore`), поэтому пароли остаются приватными. Keystore файл тоже не должен попадать в репозиторий — его нужно хранить отдельно и добавлять при сборке release.

### 14. Что будет, если удалить firebase-crashlytics из зависимостей?

**Ответ:** 
- Перестанет собираться информация о крашах
- Не будут приходить уведомления об ошибках от пользователей
- Приложение продолжит работать — Crashlytics не критичен для функциональности
- В `MainActivity` нужно будет убрать инициализацию `FirebaseCrashlytics.getInstance()`

### 15. Зачем нужны excludes в Google Drive API?

**Ответ:** 

```kotlin
implementation("com.google.apis:google-api-services-drive:v3-rev20240123-2.0.0") {
    exclude(group = "com.google.guava", module = "guava-jdk5")  // Старая версия Guava несовместима с Android!
}
```

`guava-jdk5` — старая JDK-специфичная версия Google Collections, которая использует классы недоступные на Android. Без `exclude` будет ошибка при сборке: `Duplicate class com.google.common...`.

## Вопросы по безопасности зависимостей

### 16. Какие зависимости отвечают за безопасность приложения?

**Ответ:**
- **SecurityCheck** (свой код) — проверка подписи APK, защита от модификаций
- **Firebase Auth** — безопасная аутентификация с токенами
- **local.properties** — хранение секретных ключей, не коммитится в git
- **nexusforge.keystore** — подпись release сборки, хранится отдельно

### 17. Почему Credential Manager лучше старого Google Sign-In SDK?

**Ответ:**
- Меньший размер APK (~2MB vs ~10MB)
- Единый API для всех провайдеров (Google, Apple, Email, Passkeys)
- Автоматическая обработка конфликтов аккаунтов
- Поддержка новых стандартов (Passkeys/FIDO2)
- Лучше поддерживается Google

### 18. Как защитить API ключи в приложении?

**Ответ:** В проекте NexusForge:
- Ключи хранятся в `local.properties` → читаются через `properties.getProperty()`
- `WEB_CLIENT_ID` попадает в `BuildConfig.WEB_CLIENT_ID` (читается на клиенте)
- Никогда не хардкодить ключи в коде
- Never commit `local.properties` в git

### 19. Что будет если Firebase BOM обновится до новой версии?

**Ответ:** 
- Все компоненты Firebase автоматически обновятся до совместимых версий
- Нужно проверить changelog на breaking changes
- Протестировать аутентификацию и Firestore после обновления
- Обычно обновление BOM безопасно — Google поддерживает backward compatibility

### 20. Как понять, какая зависимость вызывает увеличение размера APK?

**Ответ:** 
```bash
./gradlew app:dependencies              # Дерево зависимостей
./gradlew app:appDependenciesDebug     # Зависимости для debug
```

В Android Studio: Build → Analyze App Bundle → раздел "Dependency Size" показывает вклад каждой библиотеки в размер APK.
