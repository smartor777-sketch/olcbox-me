# Olcbox-me

Клиент для `olcrtc`-туннелей, построенный на Kotlin Multiplatform и Compose Multiplatform.

Форк [olcbox](https://github.com/alananisimov/olcbox) с поддержкой авторизации по логину/паролю (claims auth).

<p align="center">
  <img src="docs/screenshots/photo_2026-05-05_15-40-57.jpg" alt="Главный экран" width="200">
  <img src="docs/screenshots/photo_2026-05-05_15-40-59.jpg" alt="Настройки локации" width="200">
  <img src="docs/screenshots/photo_2026-05-05_20-40-45.jpg" alt="Раздельное туннелирование" width="200">
</p>

## Возможности

- Запуск/остановка туннеля одним нажатием с индикацией статуса.
- Сохранённые профили подключения с выбором активной локации.
- Провайдеры: Jazz, Telemost, WB Stream, Jitsi.
- Транспорты: DataChannel, VP8, SEI.
- Настройка VP8: FPS и размер батча.
- Проверка доступности отдельных локаций.
- Импорт конфига из буфера обмена/файла и экспорт в буфер обмена.
- Просмотр логов диагностики с экспортом.
- Android: режимы TUN/proxy, учётные данные SOCKS5, раздельное туннелирование.
- Авторизация по логину/паролю (claims auth) — уникальная фича этого форка.
- Миграция сети, переподключение, watchdog.

## Платформы

| Платформа | Режим | Стек |
| --- | --- | --- |
| Android | TUN, proxy | `VpnService`, `hev-socks5-tunnel`, локальный `olcrtc` SOCKS5 |
| iOS | Proxy | локальный `olcrtc` SOCKS5 через Swift-обёртку |
| macOS | Proxy | локальный `olcrtc` SOCKS5, PAC через ОС |
| Windows | TUN | локальный `olcrtc` SOCKS5, `hev-socks5-tunnel`, Wintun |
| Linux | TUN | локальный `olcrtc` SOCKS5, `hev-socks5-tunnel`, policy routes |

## Требования

| Область | Требование |
| --- | --- |
| JVM | JDK 17+ |
| Android | Android Studio, Android SDK, Android NDK |
| Desktop native assets | Go и локальный форк `olcrtc` |
| Linux TUN | `iproute2`, `/dev/net/tun`, root/Polkit/sudo |
| Windows TUN | UAC-повышение при запуске TUN, bundled Wintun |

Путь к локальному `olcrtc` по умолчанию:

```text
../olcrtc
```

Переопределение через `OLCRTC_REPO`:

```bash
OLCRTC_REPO=/path/to/olcrtc ./gradlew :desktopApp:run
```

Относительные пути разрешаются от корня проекта:

```bash
OLCRTC_REPO=../olcrtc-users ./gradlew :androidApp:assembleRelease
```

```powershell
$env:OLCRTC_REPO="C:\path\to\olcrtc"
.\gradlew.bat :desktopApp:run
```

## Сборка

### Android

```bash
./gradlew :androidApp:assembleDebug      # debug APK
./gradlew :androidApp:assembleRelease    # release APK (нужен keystore)
./gradlew :androidApp:installDebug       # установка на устройство
```

### Desktop

```bash
./gradlew :desktopApp:run                                    # запуск
./gradlew :desktopApp:hotRun --auto                          # hot reload
./gradlew :desktopApp:buildDesktopNativeAssets               # сборка нативных компонентов
./gradlew :desktopApp:packageReleaseDistributionForCurrentOS # пакет для текущей ОС
```

### Сборка по платформам

| Цель | Команда | Результат |
| --- | --- | --- |
| Android debug | `./gradlew :androidApp:assembleDebug` | `androidApp/build/outputs/apk/debug/androidApp-debug.apk` |
| Android release | `./gradlew :androidApp:assembleRelease` | `androidApp/build/outputs/apk/release/androidApp-release.apk` |
| macOS DMG | `./gradlew :desktopApp:packageReleaseDmg` | `desktopApp/build/compose/binaries/main-release/dmg/Olcbox-me-*.dmg` |
| Windows EXE | `.\gradlew.bat :desktopApp:packageReleaseExe` | `desktopApp\build\compose\binaries\main-release\exe\Olcbox-me-*.exe` |
| Windows MSI | `.\gradlew.bat :desktopApp:packageReleaseMsi` | `desktopApp\build\compose\binaries\main-release\msi\Olcbox-me-*.msi` |
| Linux AppImage | `./gradlew :desktopApp:packageReleaseLinuxAppImage` | `desktopApp/build/compose/binaries/main-release/appimage/Olcbox-me-*.AppImage` |

### CI/CD

Сборка автоматизирована через GitHub Actions (`.github/workflows/release.yml`).

При пуше в `main` или ручном запуске (`workflow_dispatch`) собираются:
- **Android** — подписанный APK (если настроены secrets) или debug APK
- **Windows** — EXE + MSI + portable ZIP
- **macOS** — DMG (arm64 + amd64)
- **Linux** — AppImage
- **iOS** — unsigned IPA

### Secrets для подписи Android

| Secret | Описание |
| --- | --- |
| `ANDROID_RELEASE_KEYSTORE_BASE64` | Keystore в base64 |
| `ANDROID_RELEASE_STORE_PASSWORD` | Пароль keystore |
| `ANDROID_RELEASE_KEY_ALIAS` | Alias ключа |
| `ANDROID_RELEASE_KEY_PASSWORD` | Пароль ключа |

## Заметки

- Android native ABIs: `armeabi-v7a`, `arm64-v8a`, `x86_64`.
- Release-сборки Android требуют `keystore.properties` в корне репозитория.
- macOS DMG не нотифицирован без отдельной настройки подписи.
- Windows-установщики собираются только на Windows (`jpackage` + WiX).
- Linux TUN-интерфейс: `olcbox0`.
- Linux: при остановке удаляются policy routes и восстанавливаются настройки `systemd-resolved`.

`keystore.properties`:

```properties
storeFile=/absolute/path/to/release.keystore
storePassword=...
keyAlias=...
keyPassword=...
```

Привилегии на Linux:

```bash
OLCBOX_LINUX_PRIVILEGE=sudo ./gradlew :desktopApp:run
OLCBOX_LINUX_PRIVILEGE=pkexec ./gradlew :desktopApp:run
```

## Лицензия

MIT
