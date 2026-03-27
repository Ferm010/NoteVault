### После компиляции релиза

1. **Соберите release APK** и получите хеш подписи из logcat
2. **Вставьте хеш** в SecurityCheck.kt → EXPECTED_SIGNATURE
3. **Создайте keystore** для подписи release версии
4. **Не коммитьте** keystore и local.properties в git