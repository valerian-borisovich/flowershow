### Цель: 
описание требований к внедрению функционала turnstile на платформе SG.

### Бизнес задача: 
нам необходимо отфильтровать ботовые и фейковые регистрации пользователей, так же как и бот логины. Причина в том что бот логины абузят платежки, что несет риски для проектов.

Решение: интегрировать функционал [turnstile](https://developers.cloudflare.com/turnstile/get-started/client-side-rendering/). 

## Обоснование: 
мы уже плотно интегрированы с CloudFlare. Добавление еще одной интеграции не ухудшит общую картину зависимости от их услуг.

## Функциональные требования: 
- мы хотим возможность включать и выключать функционал по требованию (через конфиги к примеру). 
- Мы хотим иметь возможность указать разные sitekey для разных api.hostnames. 
- Мы хотим вызывать turnstile клиентскую верификацию при попытке регистрации и при попытке логина. 
- Мы хотим использовать серверную верификацию токена turnstile на методы логина и регистрации
- Опционально: мы хотим так же проверять клиентскую и серверную верификацию при попытке депозита.

### Поддержка множества аккаунтов:
Мы как оператор имеем некоторое количество разных аккаунтов cloudflare на которых находятся разные домены. Мы не можем использовать один аккаунт cloudflare, по причине того, что лимиты cloudflare не позволяют засунуть все наши домены и pages в один аккаунт. При этом со стороны cloudflare существует привязка - `turnstile public key + turnstile secret key = 10 domain names`. 

Это значит что мы не сможем использовать к примеру наши лендинги с turnstile, т.к. мы не сможем в один ключ turnstile засунуть все наши домены разбросанные по разным аккаунтам.

Поэтому мы хотим, чтобы при необходимости инициализации turnstile public key получался с бекенда. При этом на бекенде мы бы хотели иметь возможность указать пару public/secret key для каждого hostname (оптимально в виде набора регулярок или глобов, но списком будет тоже ок).

## Предложенная схема реализации:

В конфиг [сайта](https://wlcgitlab.egamings.com/wlcalevcasino/web/-/blob/develop/config/backend/0.site.config.php?ref_type=heads#L243) добавить новую опцию: 
```php
$cfg['turnstile'] = true; // enable turnstile integration
$cfg['turnstileKeys'] => [
    'alev.casino' => ['publicKey','privateKey'],
    '*.alev.casino' => ['publicKey','privateKey'],
    '*.alevcasino.com' => ['publicKey','privateKey'],
    '*' => ['publicKey','privateKey'], // default key, used when no match
];
$cfg['turnstile_login'] = true; // server validation for login
$cfg['turnstile_signup'] = true; // for signup
$cfg['turnstile_deposit'] = true; // for deposit
```
Так же добавить новый метод:

GET `/api/v1/turnstile`
```json
{
	publicKey: "key",
}
```
Который бы возвращал публичный ключ, основываясь на домене с которого был запрос. В случае получения запроса с неизвестного домена — можно отдавать дефолтный ключ.

> Важно: желательно чтобы настройка приходила в отдельном методе, чтобы мы могли загружать ее к примеру на лендингах, делая один дешевый запрос (а не вытягивая огромный конфиг для всего сайта)

Мы не хотим запускать turnstile для всех пользователей. Причина в том, что он тоже стоит денег. Поэтому для нас оптимально имплементировать его как часть формы логина/регистрации. Такая реализация предпочтительна, потому что аттестация turnstile имеет время жизни, около 5и минут. Поэтому вызывать аттестацию (инициализировать проверку) имеет смысл непосредственно перед целевым действием. Таким образом мы и сэкономим бюджет на запусках аттестации и не будем лишний раз нагружать браузер пользователей.

То есть при рендере формы логина и регистрации мы бы хотели внутри формы рендерить элемент turnstile, который бы сразу запускал аттестацию и складывал токен аттестации в один из параметров запроса на бекенд. 

Бекенд, обладая private key, может проверить настоящий ли был использован токен. Если проверка успешна мы должны логинить или регистрировать пользователя, в случае ошибки верификации — возвращать ошибку.

### фронтенд часть
[документация](https://developers.cloudflare.com/turnstile/get-started/client-side-rendering/)
Пример скрипта установки и запуска turnstile:
```javascript
<script>
    (function() {
      console.log("ts loaded");
      var tsblock = document.createElement("div");
      tsblock.id = "ts";
      var modal = document.createElement("div");
      modal.style.display = "none";
      modal.appendChild(tsblock);
      document.body.appendChild(modal);
      var onloadTurnstileCallback = function () {
        var tsid = turnstile.render(tsblock, {
          appearance: "interaction-only",
          sitekey: "3x00000000000000000000FF",
          action: window.location.hostname.replaceAll('.','_'),
          callback: function (token) {
            console.log("Challenge Success token", token);
            modal.style.display = "none";
            turnstile.remove(tsid);
            sessionStorage.setItem("ts", true);
          },

          "before-interactive-callback": function (before) {
             modal.style =
               "position: fixed;width: 100vw;height: 100vh;left: 0;top: 0;z-index: 9999;background-color: #00000076;display: flex;justify-content: center;align-items: center;backdrop-filter: blur(6px);";
             dataLayer.push({ event: "turnStyleLaunched" });
          },
          "error-callback": function (err) {
            dataLayer.push({ event: "turnStyleError", turnStyleError: err });
            console.log("error", err);
          },
      });
    };
    const script = document.createElement('script');
    script.src = 'https://challenges.cloudflare.com/turnstile/v0/api.js??render=explicit';
    script.type = 'text/javascript';
    script.onload=onloadTurnstileCallback;
    script.defer = true;
    document.head.appendChild(script);
  })();
  </script>
```
Сейчас мы используем скрипт близкий к выше приведенному для установки turnstile на сайт.
[Посмотреть](https://liv-cdn.pages.dev/turnstile) - при загрузке страницы всегда будет показана тестовая аттестация.
### Важные для нас моменты:

Подключение библиотеки turnstyle должно осуществляться со специальным ключом `render=explicit`. Он необходим для того, чтобы turnstile не лез в наш дом и не начинал сам что-то в него добавлять. Мы хотим сами вызвать его, когда он нам нужен.

На продукте мы можем загружать его асинхронно, сильно позже загрузки основного сайта, но желательно до открытия формы регистрации/логина.

При открытии формы мы должны добавить в нее элемент, внутрь которого при необходимости будет отрендерен интерактивный блок прохождения каптчи. Это может быть скрытый элемент, как в примере выше.

После создания DOM элемента мы должны вызвать метод `turnstile.render(domElement, options)`, где domElement это скрытый элемент, внутрь которого мы рендерим turnstile. Так же нам нужно передать настройки запуска:
- `appearance: "interaction-only"`: для нас это самая важная настройка. Она говорит не показывать turnstile, если аттестация пользователя успешна. Благодаря ней только подозрительные пользователи получат запрос на подтверждение человечности.
- `sitekey: "xxx",`: сюда мы должны передать публичный ключ связанный с нашим доменом (к примеру из api метода предложенного ранее)
- `action: window.location.hostname.replaceAll('.','_')`: это информационное поле, в котором мы можем предать информацию, которую будет видно внутри админ панели cloudflare. Удобно передавать сюда hostname на котором запущен turnstile, чтобы иметь возможность смотреть статистику в разрезе разных доменов.
![[turnstile-action-analitics.png]]
Так же имеет смысл, передавать сюда тип проверки которая осуществляется, login, signup или deposit. Фактически это ключ для разделения статистики внутри админ панели cloudlare.
- `callback: function (token)` - это функция в которую нам будет передан токен. Его нужно прицепить к списку полей отправляемых вместе с формой. Собственно этот токен это и есть признак аттестации, который мы должны далее валидировать на бекенде.
- `"before-interactive-callback": function (before)`: этот вызов происходит перед запуском интерактивного взаимодействия с пользователем, если оно все же понадобилось. В этом месте мы можем показать раннее скрытый элемент, внутри которого отрендерен блок turnstile
- `"error-callback": function (err)`: тут будут ошибки, причем как прохождения аттестации, так и просто инициализации виджета.

### Аналитика
Следует обратить внимание, что мы дополнительно отправляем в аналитику некоторые события:
- `turnStyleLaunched`: нам нужно это событие, чтобы в аналитике видеть пользователей, которые стригерили ручную аттестацию.
- `turnStyleError`: нам нужно это событие, с дополнительным полем с описанием ошибки, чтобы понимать что у людей пошло не так. [Список возможных ошибок](https://developers.cloudflare.com/turnstile/troubleshooting/client-side-errors/)
> Нам ВАЖНО иметь эти данные в аналитике.

### Серверная валидация токена аттестации
Реализация проверки на сервере, [пример на php](https://clifford.io/blog/implement-cloudflare-turnstile-with-php/). 
Собственно для проверки необходимо всего лишь: приватный ключ, токен и ip с которого пришел запрос. На базе этой информации мы можем проверить корректность проведения аттестации клиентом.

> ВАЖНО: нам нужно сохранять все запросы которые были успешно аттестованы, с сохранением дополнительной информации в виде заголовка [cfray](https://developers.cloudflare.com/fundamentals/reference/http-request-headers/#cf-ray). Сохранение этого заголовка является НЕОБХОДИМЫМ УСЛОВИЕМ для рассмотрения со стороны CF инцидентов связанных с попытками обойти turnstile. Нам важно вести лог, всех успешных аттестаций и сопутствующих им заголовков CFRay.

### Опционально
После реализации защиты логина и регистрации можно заняться защитой инициализации вызова метода депозита.