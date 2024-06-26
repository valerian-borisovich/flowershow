# Andorid
Предположение: Мы имеем готовую игру на flutter.

Мы покупаем домен и публикуем на нем Terms.
В игру встраивается кнопка `Term & Conditions`, по нажатию на которую должен открываться webview с T&C в виде веб страницы.
## SDK которые необходимо интегрировать
Необходимо интегрировать
- [appsflyer SDK](https://pub.dev/packages/appsflyer_sdk)
- [firebase cloud messaging sdk](https://pub.dev/packages/firebase_messaging)
- [IDFA плагин](https://pub.dev/packages/app_tracking_transparency)

При этом appsflyer должен обрабатывать deeplinks (корректно настроить перехват и permission для андроида)

Настраивать интеграцию appsflyer и FCM (треккинг удаления приложения) необязательно.

> Важно: [все методы получения атрибуции](https://github.com/AppsFlyerSDK/appsflyer-flutter-plugin/blob/master/doc/Guides.md) (и старые и новые) должны быть реализованны

При этом для FCM необходимо исследовать возможность конфигурирования данных для подключения (если получится менять данные FCM на лету — мы добавим их в api endpoind, если нет — увы зафиксируем статичные)

Запрос на отправку permissions нужно делать сразу при старте приложения. Отказ от выдачи разрешений не должен блокировать запуск приложения.

> Важно: мы не хотим подключать никакие части Firebase кроме CloudMessaging (экономим место)

Запрос на IDFA необходимо делать сразу при старте приложения.

## OnAttribution event
При получении данных по атрибуции из appsflyer мы хотим вызвать API

> Важно: url api метода должен быть вынесен в конфиг.

POST `/sendAttribution`
```json
{
  data: {... attribution data ...},
  advertisement: "", // idfa id
  appsflyerid: "1690...", // отдельно appsflyer id
  fcmid: "dwreerd", // token to send firebase notifications
  device: {
    device_dpr: 23, //
    device_width: 2323, //
    device_height: 2332, //
    model: "a10s", // phone model
    os_name: "android", // ios
    version: "SM-A107M",
    manufacture: "samsung",
  },
  app_bundle_id: "com.my.app", // id приложения в сторе для идентификации приложения
  app_version: "1.0.1", // версия нашего билда
  clienToken: "uuid", // у нас нет ид устройств, поэтоу это наш ид инсталяции, просто uuid сохраненный в appdata, чтобы он не менялся при перезапуске
}
```

В ответ мы получаем информацию о целевом действии:
```json
{"action": "none"}
```
В этом случаем мы запускаем игру как обычно.
либо:
```json
{
"action": "redirect",
"payload": "https://google.com"
}
```
В этом случае:
## Обработка action redirect
При получении команды на редирект мы должны открыть webview на весь экран с полученным урл. Опции закрыть webview не предусмотренно.

## Оптимизация размера
Необходимо переделать все ассеты на загружаемые с CDN для уменьшения размера игры


Вопросы: 
- хотим ли мы перехватить пуши и использовать собственную визуализацию пушей?
- нужен ли нам IDFA на андроиде (думаю да)?
- должны ли мы слать повторные запросы на notification permission
