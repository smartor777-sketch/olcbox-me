# BUILD_REQUIREMENTS — OlcboxME

Этот файл фиксирует ключевые изменения, отличающие OlcboxME от оригинального Olcbox.
**Все сборки должны учитывать эти требования, чтобы не перезаписывать Olcbox.**

## 1. Имя приложения

| Платформа | Где задаётся | Значение |
|-----------|-------------|----------|
| Desktop (EXE, MSI) | `desktopApp/build.gradle.kts` — `packageName`, `menuGroup` | `OlcboxME` |
| Desktop (tray, window title) | `desktopApp/src/main/kotlin/main.kt` | `OlcboxME` |
| Android (app label) | `androidApp/src/main/AndroidManifest.xml` — `android:label` | `OlcboxME` |
| Android (QS tile) | `androidApp/src/main/res/values/strings.xml` — `qs_tile_label` | `OlcboxME` |
| Android (VPN session) | `OlcboxVpnService.kt` — `.setSession()`, notification channel | `OlcboxME` |
| iOS | `iosApp/iosApp/Info.plist` — `CFBundleDisplayName` | `OlcboxME` |
| Внутреннее (Compose) | `sharedUI/build.gradle.kts` — `GeneratedAppInfo.NAME` | `OlcboxME` |

## 2. Пути установки

| Платформа | Путь | Контролируется |
|-----------|------|----------------|
| Windows | `C:\Program Files\OlcboxME\` | `desktopPackageName` в `desktopApp/build.gradle.kts` |
| Windows (Start Menu) | `OlcboxME\` | `menuGroup` в `desktopApp/build.gradle.kts` |
| Android | `/data/app/org.olcbox.app-*` (неизменяем) | `applicationId = "org.olcbox.app"` (не трогать) |

## 3. Системные идентификаторы (должны быть `olcboxme`)

| Идентификатор | Файл | Текущее → Должно быть |
|---------------|------|----------------------|
| VPN-канал уведомлений | `OlcboxVpnService.kt` — `NOTIFICATION_CHANNEL_ID` | `olcbox_vpn` → `olcboxme_vpn` |
| WakeLock тег | `OlcboxVpnService.kt` — `"Olcbox::VpnWakeLock"` | `Olcbox` → `OlcboxME` |
| TAG логов | `OlcboxVpnService.kt` | `OlcboxVpnService` → `OlcboxMEVpnService` |
| Имена потоков | `OlcboxVpnService.kt` — thread name | `Olcbox*` → `OlcboxME*` |
| Имя TUN-интерфейса (Linux) | `LinuxTunController.kt` — `TUN_NAME` | `olcbox0` → `olcboxme0` |
| env var | `LinuxTunController.kt` — `OLCBOX_LINUX_PRIVILEGE` | `OLCBOX*` → `OLCBOXME*` |
| ARG | `main.kt` — `WINDOWS_ELEVATED_START_ARGUMENT` | `--olcbox-*` → `--olcboxme-*` |
| Имя файла логов (iOS) | `SwiftPlatformBridge.swift` | `olcbox-logs.txt` → `olcboxme-logs.txt` |
| Иконка Linux Desktop | `desktopApp/build.gradle.kts` — `Icon=olcbox` | `olcbox` → `olcboxme` |
| Репозиторий апдейтов | `AppUpdateService.kt` — `repositoryUrl` | `olcbox` → `olcboxme` |
| AppData path (Windows) | `DesktopPaths.kt` — `appDataDir()` | `Olcbox` → `OlcboxME` |
| AppData path (macOS) | `DesktopPaths.kt` — `appDataDir()` | `Olcbox` → `OlcboxME` |
| AppData path (Linux) | `DesktopPaths.kt` — `appDataDir()` | `.olcbox` → `.olcboxme` |

## 4. Тестовые данные

В тестах (`AppUpdateServiceTest.kt`, `DesktopProxyModeTest.kt`) все имена артефактов
и путей должны использовать `OlcboxME` / `olcboxme` вместо `Olcbox` / `olcbox`.

## 5. Что НЕ нужно менять

| Элемент | Причина |
|---------|---------|
| `applicationId = "org.olcbox.app"` (Android) | Менять = новый апп в Play Store |
| `namespace = "org.olcbox.app"` | Должен совпадать с applicationId |
| `bundleID = "org.olcbox.app.desktopApp"` (macOS) | Можно менять, но не критично |
| PROGUARD keep rules | Ссылаются на класс `OlcboxVpnService` — менять вместе с классом |
| JNI функции `olcbox_tun2socks_jni.c` | Ссылаются на `OlcboxVpnService` — менять вместе с классом |

## 6. Порядок полного ребрендинга

1. `desktopApp/build.gradle.kts` — `packageName`, `menuGroup`, `desktop` entries, icons
2. `desktopApp/src/main/kotlin/main.kt` — tray, window title, file dialogs, arg
3. `sharedUI/build.gradle.kts` — `GeneratedAppInfo.NAME`, `olcboxAppInfo` dir
4. `androidApp/build.gradle.kts` — gradle props `olcbox.*` (внутренние, не user-facing)
5. `OlcboxVpnService.kt` — notification channel, TAG, thread names, wakelock, split
6. `OlcboxVpnActions.kt` — intent strings (если не жалко обратную совместимость)
7. iOS: `Info.plist`, `SwiftPlatformBridge.swift`, `SwiftOlcRtcManager.swift`
8. Тесты: `AppUpdateServiceTest.kt`, `DesktopProxyModeTest.kt`
9. Документация: `README.MD`, `Makefile`

## 7. Проверка перед сборкой

```bash
# Убедиться что нет ссылок на оригинальное имя
rg -i "olcbox" android/olcbox/ --type-not md | grep -iv "olcboxme\|olcrtc"
# Должны остаться только:
# - packageName / namespace / applicationId (org.olcbox.app)
# - olcrtc
# - olcboxme (уже переименованные)
```
