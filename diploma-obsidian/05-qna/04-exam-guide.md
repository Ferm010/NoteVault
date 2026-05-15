# Гайд к экзамену — NexusForge (Дипломная работа)

## Структура проекта для защиты

```
NexusForge/
├── app/src/main/java/com/ferm/nexusforge/
│   ├── MainActivity.kt              # Точка входа
│   ├── backend/                     # Бизнес-логика
│   │   ├── AppNavigation.kt         # Навигация (553 строки!)
│   │   ├── NavigationData.kt        # sealed interface Destination
│   │   ├── SecurityCheck.kt         # Проверка целостности APK
│   │   └── LocaleHelper.kt          # Мультиязычность
│   ├── frontend/                    # UI экраны (Composables)
│   │   ├── mainmenu/MainMenuPage.kt
│   │   ├── FavoritePage.kt
│   │   ├── ProfilePage.kt
│   │   └── ...
│   ├── viewmodels/                  # ViewModel для каждого экрана
│   ├── repository/                  # Репозитории (Firestore, Drive)
│   ├── network/                     | Modrinth API
│   ├── data/                        | Модели данных (@Serializable)
│   ├── utils/                       | Утилиты
│   └── ui/theme/                    | Тема (Colors, Typography, Theme)
```

## Ключевые технологии для защиты

### 1. Kotlin
- **sealed interface** — type-safe навигация, обработка состояний
- **data class + @Serializable** — модели данных, сериализация параметров навигации
- **coroutines + flow** — асинхронность, реактивные данные
- **extension functions** — чистый код (isEmail(), orEmptyIfNull())

### 2. Jetpack Compose
- **Declarative UI** — описание ЧТО, а не КАК рисовать
- **State Hoisting** — вынесение состояния наружу компонентов
- **LazyColumn/LazyVerticalGrid** — ленивые списки для производительности
- **Scaffold + NavigationBar** — Material Design 3 layout

### 3. Architecture
- **MVVM**: View (Compose) → ViewModel → Repository → Data Source
- **Navigation3**: Type-safe навигация через sealed interface
- **StateFlow**: Реактивное состояние UI
- **Repository Pattern**: Абстракция над источниками данных

### 4. Dependencies
- **Firebase Auth + Firestore** — Backend as a Service
- **Retrofit + kotlinx.serialization** — HTTP API для Modrinth
- **Coil** — загрузка изображений (Kotlin-first)
- **Google Drive API** — облачное хранение модпаков

## Типовые вопросы на защите и ответы

### 🎓 "Опишите архитектуру вашего приложения"

**Ответ:** Приложение построено по архитектуре MVVM (Model-View-ViewModel):
- **View** — Jetpack Compose composables в пакете `frontend/`
- **ViewModel** — управляют состоянием UI, получают данные из Repository
- **Repository** — абстракция над источниками данных (Firestore, Modrinth API, Google Drive)
- Навигация через Navigation3 с type-safe sealed interface Destination

### 🎓 "Почему вы выбрали Jetpack Compose вместо XML?"

**Ответ:** 
1. Declarative подход — описываем ЧТО на экране, а не КАК рисовать
2. Меньше boilerplate кода (нет findViewById, нет XML)
3. Встроенная анимация и адаптивность
4. Kotlin-first — нет Java/Kotlin bridge overhead
5. Smart recomposition — перерисовываются только изменившиеся composables
6. Compose-native компоненты (AsyncImage, AnimatedVisibility)

### 🎓 "Как работает навигация в приложении?"

**Ответ:** Navigation3 с sealed interface Destination:
```kotlin
@Serializable
sealed interface Destination : NavKey {
    @Serializable data object MainMenu : Destination
    @Serializable data class ProjectDetailsPage(val projectId: String) : Destination
}
```
Это обеспечивает type-safe навигацию — компилятор знает все экраны и их параметры. Навигация через `backStack += Destination.X` (push), `backStack.removeLastOrNull()` (pop).

### 🎓 "Как вы организуете асинхронную работу?"

**Ответ:** Coroutines + Flow:
- `viewModelScope.launch(Dispatchers.IO)` — фоновые задачи
- `withContext(Dispatchers.IO)` — переключение на background для конкретной операции
- `StateFlow` — состояние UI (тема, данные)
- `collectAsStateWithLifecycle()` — подписка в Compose с остановкой когда экран не виден
- Firebase Task + `.await()` — интеграция Firebase с корутинами

### 🎓 "Как вы обрабатываете ошибки?"

**Ответ:** Sealed classes для всех результатов операций:
```kotlin
sealed class AuthResult { object Success : AuthResult(); data class Error(val code: String) : AuthResult() }
```
Компилятор требует обработать ВСЕ варианты в `when` — нельзя пропустить ошибку. Каждый API вызов обернут в try-catch с конкретными FirebaseAuthException кодами.

### 🎓 "Почему вы используете Firebase? Какие альтернативы?"

**Ответ:** Firebase выбран для:
- **Auth** — готовая реализация Email/Password + Google Sign-In
- **Firestore** — NoSQL cloud database, real-time sync
- **Crashlytics** — crash reporting из коробки

Альтернативы: Supabase, AWS Amplify, custom backend (Spring Boot + PostgreSQL). Firebase выбран за скорость разработки и бесплатность на старте.

### 🎓 "Как вы обеспечиваете безопасность данных?"

**Ответ:**
1. **SecurityCheck.verifyAppIntegrity()** — проверка подписи APK при запуске
2. **local.properties** — секреты не коммитятся в git
3. **Firebase Security Rules** — (в перспективе) защита доступа к Firestore
4. **Google Sign-In через Credential Manager** — современный безопасный подход

### 🎓 "Как работает интеграция с Modrinth API?"

**Ответ:** Retrofit + kotlinx.serialization:
```kotlin
interface ModrinthApiService {
    @GET("search")
    suspend fun searchProjects(@Query("query") query: String): ModrinthSearchResponse
}
```
Модели данных — data class с `@Serializable`, один подход ко всей сериализации в проекте.

### 🎓 "Как вы управляете состоянием приложения?"

**Ответ:** State Management через ViewModel + StateFlow:
- Состояние хранится в `MutableStateFlow` внутри ViewModel
- UI подписывается через `collectAsStateWithLifecycle()`
- State Hoisting — composables принимают state и event handlers как параметры
- Theme управляется отдельным ThemeViewModel с анимацией перехода

### 🎓 "Что такое State Hoisting и зачем он нужен?"

**Ответ:** State Hoisting — вынесение состояния из composables на уровень выше:
```kotlin
// ❌ Плохо — компонент управляет своим состоянием
@Composable fun NameField() { var text by remember { mutableStateOf("") } ... }

// ✅ Хорошо — состояние hoisted наружу
@Composable fun NameField(value: String, onValueChange: (String) -> Unit) { ... }
```
Преимущества: компонент становится pure, тестируемым и переиспользуемым.

### 🎓 "Почему вы используете sealed interface для навигации?"

**Ответ:** 
1. **Type-safe** — компилятор знает все возможные экраны
2. **Serializable** — параметры автоматически сериализуются/десериализуются
3. **Sealed interface** — подтипы могут быть в разных файлах (масштабируемость)
4. **NavKey** — интеграция с Navigation3 для type-safe навигации

## Что обязательно нужно знать

### Kotlin
- [ ] Разница val/var, nullable/non-null типы
- [ ] Data class — что генерирует автоматически
- [ ] Sealed class/interface — когда и зачем использовать
- [ ] Extension functions — что это, пример
- [ ] Coroutines — launch vs async, Dispatchers, suspend функции
- [ ] Flow — StateFlow vs SharedFlow, операторы map/filter/combine
- [ ] Scope functions — run, let, apply, also (когда какая)

### Jetpack Compose
- [ ] Declarative vs Imperative подход
- [ ] @Composable функции — ограничения
- [ ] Layout: Row, Column, Box, Scaffold
- [ ] Modifier — основные модификаторы
- [ ] State: remember, mutableStateOf, state hoisting
- [ ] LaunchedEffect, DisposableEffect
- [ ] LazyColumn — когда использовать
- [ ] Анимации: AnimatedVisibility, animate*AsState

### Firebase
- [ ] Auth flow в проекте (Email + Google)
- [ ] Firestore структура данных
- [ ] Как работает checkEmailExists() (хак с временным паролем)
- [ ] Crashlytics — зачем нужен

### Architecture
- [ ] MVVM паттерн — что есть в проекте
- [ ] Repository pattern — абстракция над источниками данных
- [ ] Navigation3 — type-safe навигация
- [ ] ViewModel lifecycle в Compose

## Структура презентации (5-7 минут)

### Слайд 1: Титульный
- Тема, ФИО, научный руководитель

### Слайд 2: Проблема и актуальность
- Создание модпаков Minecraft — сложная задача для обычных пользователей
- Существующие инструменты неинтегрированы (Modrinth + Drive + шаблоны)
- Решение: единое Android приложение

### Слайд 3: Цель и задачи
- **Цель**: Разработать Android-приложение для управления модпаками Minecraft
- **Задачи**: Изучить Kotlin/Compose, реализовать авторизацию, интеграцию с Modrinth API, работу с Google Drive

### Слайд 4: Архитектура приложения (MVVM)
- Схема: View → ViewModel → Repository → Data Source
- Навигация через Navigation3

### Слайд 5: Технологический стек
- Kotlin + Jetpack Compose (UI)
- Firebase Auth + Firestore (Backend)
- Retrofit + Modrinth API (поиск модов)
- Google Drive API (облачное хранение)

### Слайд 6: Ключевые экраны (скриншоты!)
- Авторизация, Главный экран, Создание модпака, Настройки

### Слайд 7: Особенности реализации
- Type-safe навигация через sealed interface
- State Management через ViewModel + StateFlow
- SecurityCheck — защита от модификаций APK

### Слайд 8: Результаты и выводы
- Реализованное приложение с полным функционалом
- Изучены современные подходы к Android разработке
- Перспективы: добавление Forge/Fabric профилей, синхронизация между устройствами

## Лайфхаки для защиты

1. **Покажите демо** — лучше один раз увидеть работающее приложение
2. **Подготовьте скриншоты** — если демо не работает на защите
3. **Знайте цифры** — количество экранов, строк кода, используемых библиотек
4. **Будьте честны** — если не знаете ответа, скажите "я об этом не думал, но предположу что..."
5. **Свяжите с теорией** — когда говорите о коде, упоминайте паттерны (MVVM, Repository)

## Вопросы, которые МОГУТ задать

### По коду:
- "Почему checkEmailExists создаёт временный аккаунт?" → Firebase не даёт API для проверки email
- "Зачем SecurityCheck?" → Защита от модифицированных APK
- "Почему sealed interface для Destination, а не sealed class?" → Подтипы в разных файлах

### По архитектуре:
- "Почему MVVM, а не MVI/MVP?" → MVVM — стандарт для Android, лучше баланс простоты и масштабируемости
- "Почему StateFlow, а не LiveData?" → Kotlin-first, больше операторов, coroutine-native

### По зависимостям:
- "Почему Retrofit, а не Ktor?" → Retrofit — стабильнее, больше сообщество, лучше документация
- "Почему Coil, а не Glide?" → Kotlin-first, Compose-native, меньше размер APK

### По безопасности:
- "Как защищены данные пользователей?" → Firebase Auth + Firestore Security Rules (в перспективе)
- "Где хранятся API ключи?" → local.properties, не коммитятся в git

## Чек-лист перед защитой

- [ ] Приложение запускается и работает на устройстве/эмуляторе
- [ ] Есть скриншоты всех основных экранов
- [ ] Знаю архитектуру проекта (MVVM + Navigation3)
- [ ] Могу объяснить выбор каждой основной зависимости
- [ ] Понимаю как работает авторизация (Email + Google)
- [ ] Знаю что такое sealed interface и зачем она в навигации
- [ ] Могу показать пример работы с Coroutines/Flow
- [ ] Подготовил презентацию (5-7 минут)
- [ ] Прорепетировал ответы на типовые вопросы
