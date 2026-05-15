# Firebase Auth — Глубокое погружение

## 1. Архитектура аутентификации в NexusForge

### Flow авторизации

```
Пользователь открывает приложение
    ↓
MainActivity.onCreate()
    ↓ SecurityCheck.verifyAppIntegrity()
    ↓ FirestoreRepository.checkUserProfileExists()
    ↓
Есть профиль? → MainMenu (авторизован)
    ↓ Нет / нет аккаунта
RegPage (выбор: Email или Google)
```

### Два потока регистрации

#### Email/Password Flow

```
RegPage → EulaPage → PasswordPage → NameRegPage → MainMenu
```

1. Пользователь вводит email
2. `AuthRepository.checkEmailExists(email)` — проверяет, занят ли email
3. Если новый → переход к EULA
4. Принятие EULA → ввод пароля
5. Ввод имени → регистрация через `registerUser(email, password, displayName)`

#### Google Sign-In Flow

```
RegPage (Google) → EulaPage → MainMenu / NameRegPage
```

1. Пользователь нажимает "Sign in with Google"
2. Credential Manager → выбор аккаунта Google
3. Получаем ID Token → Firebase `GoogleAuthProvider.getCredential()`
4. Если новый пользователь → EULA → MainMenu (имя из Google)
5. Если существующий → сразу MainMenu

## 2. AuthRepository — Детальный разбор

### checkEmailExists() — Хак с временным паролем

```kotlin
suspend fun checkEmailExists(email: String): EmailCheckResult {
    val tempPassword = "TempCheck${System.currentTimeMillis()}!@#"
    
    try {
        // Пытаемся создать аккаунт
        val result = auth.createUserWithEmailAndPassword(email, tempPassword).await()
        
        // Если создался — email свободный, удаляем временный аккаунт
        result.user?.delete()?.await()
        return EmailCheckResult.NewUser
        
    } catch (e: FirebaseAuthException) {
        when (e.errorCode) {
            "ERROR_EMAIL_ALREADY_IN_USE" -> EmailCheckResult.ExistingUser(AuthMethod.EMAIL_PASSWORD)
            "ERROR_INVALID_EMAIL" -> EmailCheckResult.Error("ERROR_INVALID_EMAIL")
            else -> EmailCheckResult.Error(e.errorCode ?: "ERROR_GENERIC")
        }
    }
}
```

**Почему такой подход?** Firebase не предоставляет прямого API для проверки существования email. Приходится пробовать создать аккаунт и смотреть на результат.

### signInWithGoogle() — Проверка конфликта методов

```kotlin
suspend fun signInWithGoogle(idToken: String): GoogleSignInResult {
    try {
        val credential = GoogleAuthProvider.getCredential(idToken, null)
        val result = auth.signInWithCredential(credential).await()
        
        return GoogleSignInResult.Success(
            isNewUser = result.additionalUserInfo?.isNewUser ?: false,
            displayName = result.user?.displayName ?: "",
            email = result.user?.email.orEmpty()
        )
    } catch (e: FirebaseAuthException) {
        when (e.errorCode) {
            "ERROR_ACCOUNT_EXISTS_WITH_DIFFERENT_CREDENTIAL" -> 
                GoogleSignInResult.Error("ERROR_ACCOUNT_EXISTS_WITH_DIFFERENT_CREDENTIAL")
        }
    }
}
```

**Конфликт методов**: Если email зарегистрирован через Email/Password, а пользователь пытается войти через Google с тем же email — Firebase вернёт `ERROR_ACCOUNT_EXISTS_WITH_DIFFERENT_CREDENTIAL`. Это значит, что нужно предложить пользователю привязать аккаунт.

## 3. Firestore Repository

### checkUserProfileExists()

```kotlin
suspend fun checkUserProfileExists(): Result<Boolean> {
    val currentUser = FirebaseAuth.getInstance().currentUser ?: return Result.failure(...)
    
    val docRef = firestore.collection("users").document(currentUser.uid)
    val doc = docRef.get().await()
    
    return if (doc.exists()) {
        Result.success(true)
    } else {
        Result.success(false)
    }
}
```

### Структура Firestore

```
users/                          ← коллекция пользователей
  └── {userId}/                 ← документ UID пользователя
      ├── displayName: "Alex"
      ├── email: "alex@mail.com"
      ├── theme: "dark"
      ├── language: "ru"
      ├── favorites/            ← подколлекция избранных
      │   └── {projectId}/
      │       └── projectId: "abc123"
      └── modpacks/             ← подколлекция модпаков
          └── {modpackId}/
              ├── name: "My Modpack"
              ├── mods: [...]
              └── driveFileId: "xxx"
```

## 4. SecurityCheck — Безопасность

### verifyAppIntegrity()

```kotlin
fun verifyAppIntegrity(context: Context): Boolean {
    // Проверка подписи APK
    try {
        val packageInfo = context.packageManager.getPackageInfo(
            context.packageName, 
            PackageManager.GET_SIGNATURES
        )
        
        for (signature in packageInfo.signatures) {
            if (signature.toByteArray() == EXPECTED_SIGNATURE) {
                return true
            }
        }
    } catch (e: Exception) {
        return false
    }
    return false
}
```

**Зачем?** Защита от модифицированных/пересобранных APK. Если подпись не совпадает — приложение закрывается.

### isDeviceSecure()

```kotlin
fun isDeviceSecure(): Boolean {
    val keyguardManager = context.getSystemService(KeyguardManager::class.java)
    return keyguardManager.isDeviceSecure  // Есть lock screen?
}
```

В проекте отключена (закомментирован `return false`), но может быть включена для блокировки на root-устройствах.

## 5. Работа с Firebase в Compose

### Инициализация FirebaseAuth

```kotlin
// В AppNavigation.kt — проверка при запуске
androidx.compose.runtime.LaunchedEffect(Unit) {
    val currentUser = FirebaseAuth.getInstance().currentUser
    
    if (currentUser != null) {
        // Проверяем профиль в Firestore
        val result = firestoreRepository.checkUserProfileExists()
        
        if (result.isSuccess && result.getOrNull() == true) {
            startDestination = Destination.MainMenu
        } else {
            FirebaseAuth.getInstance().signOut()
            startDestination = Destination.RegPage
        }
    } else {
        startDestination = Destination.RegPage
    }
}
```

### ViewModel для Auth

```kotlin
class RegViewModel : ViewModel() {
    private val authRepo = AuthRepository()
    
    // Состояние UI
    var isGoogleFlow by mutableStateOf(false)
        private set
    
    fun signInWithEmail(email: String) {
        viewModelScope.launch {
            when (val result = authRepo.checkEmailExists(email)) {
                is EmailCheckResult.NewUser -> navigateToEula()
                is EmailCheckResult.ExistingUser -> navigateToLogin()
                is EmailCheckResult.Error -> showError(result.errorCode)
            }
        }
    }
    
    fun signInWithGoogle(idToken: String) {
        viewModelScope.launch {
            when (val result = authRepo.signInWithGoogle(idToken)) {
                is GoogleSignInResult.Success -> {
                    if (result.isNewUser) navigateToEula() 
                    else navigateToMainMenu()
                }
                is GoogleSignInResult.Error -> showError(result.errorCode)
            }
        }
    }
    
    fun signOut() {
        authRepo.signOut()
        // Навигация на RegPage происходит из UI layer
    }
}
```

## 6. Типичные ошибки Firebase Auth

### ❌ Забыть await()

```kotlin
// Плохо — не дождёмся результата
auth.signInWithEmailAndPassword(email, password)  

// Хорошо — ждём корутиной
auth.signInWithEmailAndPassword(email, password).await()
```

### ❌ Не обрабатывать исключения Firebase

```kotlin
// Плохо — необработанный FirebaseAuthException
val result = auth.signInWithEmailAndPassword(email, password).await()

// Хорошо — try-catch с обработкой errorCode
try {
    auth.signInWithEmailAndPassword(email, password).await()
} catch (e: FirebaseAuthException) {
    when (e.errorCode) {
        "ERROR_WRONG_PASSWORD" -> showError("Неверный пароль")
        "ERROR_USER_NOT_FOUND" -> showError("Пользователь не найден")
    }
}
```

### ❌ Не проверять currentUser перед операциями

```kotlin
// Плохо — NPE если пользователь не авторизован
val uid = FirebaseAuth.getInstance().currentUser?.uid!!

// Хорошо — безопасная проверка
val user = FirebaseAuth.getInstance().currentUser ?: return
val uid = user.uid
```
