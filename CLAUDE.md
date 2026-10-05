# Tuvio VPN — per-app AmneziaWG для телевизора (карта)

Телевизор Tuvio TD43UFBSV1, YaOS = Android 11 (API 30). Приложение «VPN для ТВ» пускает через VPN
только выбранные приложения (YouTube и т.п.), остальное идёт напрямую. Протокол — официальный AmneziaWG.
**Это карта, не журнал.** История — `CHANGELOG.md` (не читать целиком: `rg -n -A8 'ключ' CHANGELOG.md`).
Новую хронологию дописывать туда, сюда — одну строку в «Где остановились».
Техническое описание, сборка, DNS/утечки, чек-лист приёмки — `docs/TECHNICAL.md` (в форке: `tv/README.md`).
GitHub: `jekfev/tuvio-amneziawg` — README, docs, CHANGELOG и APK в `releases/` (исходники модуля пока не выложены).

## Где остановились (05.10.2026)
- v1.0.0 стоит на ТВ, YouTube через VPN работает (владелец, 05.10).
- v1.1.0 (журнал + «Диагностика → Сохранить журнал на флешку») записана на флешку `USB-TV` как `VPN-TV.apk`,
  на ТВ ещё не установлена. Ставится поверх 1.0.0 — подпись та же, настройки сохраняются.
- Дальше: владелец ставит 1.1.0 → проверить сохранение журнала через системное окно YaOS.

## Главное ограничение
**На ТВ нет ADB** (в меню разработчика YaOS нет отладки по USB/сети). Удалённо ставить и проверять нельзя:
проверка — на эмуляторе, доставка — флешкой, логи с ТВ — через кнопку выгрузки журнала на флешку.

## Где что лежит

| Что | Где |
|---|---|
| Репозиторий (форк `amnezia-vpn/amneziawg-android`, ветка `tv-per-app`) | `~/tuvio-awg`, здесь симлинк `amneziawg-android` |
| Приложение для ТВ | `tv/` — id `tv.splitvpn.awg`, debug `tv.splitvpn.awg.debug` |
| Ядро: туннель, allowlist, переподключение | `tv/.../vpn/TunnelController.kt`, `PerAppConfig.kt` |
| Импорт `.conf` и Amnezia `vpn://` | `tv/.../data/ConfigRepository.kt`, `AmneziaVpnFormat.kt`, `ConfigImporter.kt` |
| Журнал | `tv/.../AppLog.kt` → на ТВ `files/logs/vpn.log` (+`.1`) |
| Тест-приложения для проверки per-app (не в APK) | `tools/probe` (flavors `a`, `b`) |
| JVM-тесты (в т.ч. на реальном файле: `-PtestVpnFile=…`) | `tv/src/test/.../ConfigPipelineTest.kt` |
| Готовый APK последней версии | `VPN-TV-<версия>.apk` в этой папке |
| Тестовый конфиг владельца (сервер «Riga», формат `vpn://`) | `test-config/amnezia_config.vpn` |

**Секреты:** `test-config/*.vpn` содержит приватный ключ — не выводить, не коммитить, в логи не писать.
Ключ подписи release — `~/tuvio-awg/tv/release.jks` + `tv/keystore.properties` (не в git).
Без него новые версии не встанут поверх установленной — у владельца должна быть резервная копия.

## Сборка
```bash
cd ~/tuvio-awg
export PATH="/opt/homebrew/opt/coreutils/libexec/gnubin:$PATH" \
  JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home \
  ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
./gradlew :tv:testDebugUnitTest :tv:assembleRelease
```
- APK: `tv/build/outputs/apk/release/tv-release.apk`. Версия — `tvVersionCode`/`tvVersionName` в `gradle.properties`,
  код поднимать на каждый релиз.
- **Путь без пробелов обязателен** — Makefile backend'а ломается на «Tuvio VPN», поэтому репо в `~/tuvio-awg`.
- Единственная правка официального кода: `tunnel/tools/libwg-go/Makefile` — Go 1.25.0 + `GOTOOLCHAIN=local`.
  Backend, парсер и криптографию не трогать.

## Проверка на эмуляторе
- AVD `tv30` (Android 11, arm64, без Google Play): `$ANDROID_HOME/emulator/emulator -avd tv30 -no-window -no-audio`.
- Виртуальная флешка: `adb shell sm set-virtual-disk true` → `sm list-disks` → `sm partition <disk> public`.
- Per-app: `probe.a` в allowlist, `probe.b` нет → `adb logcat -s PROBE:I` (IP через VPN и напрямую);
  UID-диапазон VPN: `dumpsys connectivity | grep -ioE "Uids: <[^>]*>"`.
- Системные диалоги эмулятора (запрос VPN) D-pad не берут — там тап; наши экраны — только D-pad.

## Флешка владельца
`USB-TV`, FAT32. На ней: `VPN-TV.apk`, `Riga.vpn`, `ИНСТРУКЦИЯ.txt`, `README.md`.
ТВ сам создаёт пустые служебные папки (Android, Movies, …) — это не мусор пользователя.
После записи с Mac: `dot_clean -m /Volumes/USB-TV` (иначе файлы `._*` на ТВ выглядят как второй битый APK), затем eject.

## Известные ограничения
- Точный ABI ТВ неизвестен → в APK `armeabi-v7a` + `arm64-v8a`; ABI виден в «Диагностике» и шапке журнала.
- Встроенный выбор файла на Android 11 видит `.conf` только с «Доступом ко всем файлам».
- Автоподключение после загрузки ТВ на YaOS не проверено (зависит от `BOOT_COMPLETED`).
- DoH/DoT внутри приложений VPN не перехватывает (трафик всё равно идёт через туннель).
