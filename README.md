# VPN Manager

Локальный Android-менеджер VLESS-конфигураций. Без backend и телеметрии: приложение само скачивает списки с
GitHub (raw), парсит, проверяет через реальный трафик, выбирает лучший сервер и поднимает full tunnel
(`VpnService` + Xray TUN).

## Архитектура
| Файл | Назначение |
|---|---|
| `Vless.kt` | Парсер VLESS URI (tcp/ws/grpc/xhttp, tls/reality), дедупликация, генерация Xray-конфигов |
| `Checker.kt` | DNS → TCP → реальный HTTPS-запрос через VLESS-outbound → 10 повторов (RTT, packet loss, стабильность) |
| `CoreBridge.kt` | Единственная точка контакта с Xray (`libv2ray.aar`) |
| `TunService.kt` | `VpnService`, TUN 0.0.0.0/0 + ::/0, fd передаётся в Xray |
| `Ctl.kt` | Состояние, загрузка, кэш, выбор сервера, автообновление (WorkManager, только Wi-Fi) |
| `Store.kt` | Локальная база (JSON в `filesDir`), кэш списков с датой |
| `MainActivity.kt` | Jetpack Compose UI, светлая/тёмная тема, системный шрифт |

Источники: `BLACK_VLESS_RUS_mobile.txt` (режим «Белые списки») и `BLACK_VLESS_RUS.txt` (режим «VPN»),
через `raw.githubusercontent.com`. Внешние запросы: GitHub, `gstatic.com/generate_204` (проверка),
`ipify.org` (внешний IP через туннель). Больше ничего.

## Сборка и раздача (без ручной возни для других)
1. Создайте репозиторий на GitHub и загрузите туда содержимое проекта.
2. **Один раз** создайте ключ подписи (чтобы обновления APK ставились поверх старых):
   `keytool -genkeypair -v -keystore release.jks -alias vpn -keyalg RSA -keysize 2048 -validity 10000`
   В Settings → Secrets → Actions добавьте: `KEYSTORE_B64` (результат `base64 -w0 release.jks`), `KEYSTORE_PASSWORD`, `KEY_ALIAS` (`vpn`), `KEY_PASSWORD`.
   Без секретов APK тоже соберётся и установится, но подписан временным ключом: каждая сборка будет с другой подписью, и обновление поверх старой версии не пройдёт (нужно удалять приложение).
3. Выпуск: `git tag v1.0.0 && git push --tags`. Actions соберёт APK и опубликует его в **Releases**. Остальным достаточно отправить ссылку на релиз.
4. Каждый push в `main` тоже собирает APK (Actions → Artifacts).

Локально: положите `libv2ray.aar` в `app/libs/` (`curl -L -o app/libs/libv2ray.aar https://github.com/2dust/AndroidLibXrayLite/releases/latest/download/libv2ray.aar`),
затем `gradle assembleRelease` (Gradle 8.9, JDK 17, Android SDK 34) → `app/build/outputs/apk/release/app-release.apk`.

Если API `libv2ray.aar` изменилось, правится только `CoreBridge.kt`. Для воспроизводимости зафиксируйте тег релиза вместо `latest` в `build.yml`.

## Раздельное туннелирование
Главный экран → «Раздельное туннелирование»: все приложения / только выбранные / все, кроме выбранных (на уровне приложений,
через `VpnService.Builder.addAllowedApplication/addDisallowedApplication`). Само приложение всегда вне туннеля.

## Установка (для пользователей)
Скачайте APK из Releases, разрешите установку из неизвестных источников, откройте файл. При первом подключении подтвердите диалог VPN.

## Ограничения
- Тест скорости (download/upload) не реализован: проверка идёт по RTT/loss/стабильности. В UI скорость показана как «н/д».
- DNS leak detection и отдельная проверка IPv6-доступности серверов не реализованы; внешний IPv4/IPv6 показывается через туннель.
- Публичные конфигурации принадлежат третьим лицам. Оператор сервера может видеть и изменять незашифрованный трафик.
  Прохождение технических проверок не означает безопасность сервера.
