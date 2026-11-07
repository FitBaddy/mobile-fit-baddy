# Промпты к ИИ — Android-клиент (`mobile-fit-baddy`)

---

## Сеанс 1

делаю нативное приложение на java (не kotlin, это требование), android sdk
нужен gradle-проект: minSdk 26, retrofit + okhttp + gson
как правильно указать адрес backend через BuildConfig, чтобы менять для эмулятора и телефона

---

открыла проект в android studio, sync падает:

```
Android Gradle plugin requires Java 17 to run. You are currently using Java 11.
Your current JDK is located in /Library/Java/JavaVirtualMachines/jdk-11.jdk/Contents/Home
You can try some of the following options:
 - changing the IDE settings.
 - changing the JAVA_HOME environment variable.
```

где это менять, в Settings -> Gradle -> Gradle JDK или в compileOptions? и targetSdk 34 норм или лучше 33

---

поменяла jdk на 17, теперь BuildConfig красный, Cannot resolve symbol 'BuildConfig'. buildConfigField же есть в build.gradle

---

а, в agp 8 его по умолчанию выключили, buildFeatures { buildConfig = true }. понятно, спасибо

---

эмулятор не ходит на http://10.0.2.2:8000, в логе CLEARTEXT communication not permitted. мне нужно разрешить http только для локалки, не для всего приложения. network_security_config?

---

а с телефона по ip компьютера тоже http, его тоже в конфиг добавлять? ip же меняется в разных сетях, неудобно каждый раз прописывать

---

**Задача.** DTO и интерфейс `FitQuestApi` по swagger backend (`/docs`).

**Требования.**
- Только эндпоинты, которые нужны экранам, остальное добавлю сама.
- `AuthInterceptor` подставляет `Authorization: Bearer` во все запросы, кроме `/auth/login` и `/auth/register`.
- Поля DTO — как в ответах backend (snake_case через `@SerializedName`).

**Вопрос.** Токен хранить в SharedPreferences — нормально для учебного проекта, или лучше EncryptedSharedPreferences?

---

ок, Encrypted. а если access протух — interceptor сам сходит в /auth/refresh или пусть экран ловит 401? мне кажется проще на 401 кидать на логин, refresh потом

---

добавила security-crypto, сборка падает:

```
Execution failed for task ':app:checkDebugAarMetadata'.
> A failure occurred while executing com.android.build.gradle.internal.tasks.CheckAarMetadataWorkAction
   > Dependency 'androidx.security:security-crypto:1.1.0-alpha06' requires libraries and applications that
     depend on it to compile against version 34 or later of the Android APIs.
```

compileSdk же 34 уже стоит? или он не тот модуль смотрит

---

в каждом фрагменте одинаковый enqueue с onResponse/onFailure на 20 строк. вынеси в ApiCalls.enqueue(call, onSuccess, onError), и чтобы колбэки приходили в главный поток

---

ошибки от backend сейчас показываются как есть: {"detail":[{"loc":["body","email"],"msg":"value is not a valid email"...}]}. разбери json и покажи человеку только msg, а если detail строка — её саму

---

## Сеанс 2

главный экран Den: питомец (уровень, xp, настроение) + сводка за неделю
разметку я сделала сама, помоги с фрагментом: загрузка данных, ошибки, пустое состояние

---

ещё на Den: серия дней, энергия, тренировка на сегодня, последний пульс и шаги. если пульса нет (null) — прочерк, а не 0

---

приложение вылетает сразу при открытии Den, в logcat:

```
FATAL EXCEPTION: main
Process: com.fitquest.app, PID: 8812
com.google.gson.JsonSyntaxException: java.lang.IllegalStateException: Expected BEGIN_OBJECT but was STRING at line 1 column 214 path $.today_schedule
```

today_schedule в сводке бывает объект, бывает null, а тут вообще строка? как лучше, JsonElement и разбирать руками или отдельный адаптер

---

xp-бар у тебя считает xp / 100, а порог уровня у backend level×100. возьми порог из ответа, не хардкодь

---

хочу вместо статичной картинки питомца гифку лисы fox.gif из assets. ImageView гифки не крутит, можно без glide? minSdk 26, AnimatedImageDrawable только с 28

---

ладно, на 28+ AnimatedImageDrawable, ниже просто первый кадр. не хочу тащить библиотеку ради одной картинки

---

студия подчёркивает ImageDecoder красным: Call requires API level 28 (current min is 26). я же проверку Build.VERSION.SDK_INT сделала, почему lint всё равно ругается

---

экран Quest: список программ, старт сессии, завершение, история
пока без ViewModel, экран простой. если будет мешать поворот экрана — переделаем

---

создание программы: название + упражнения из списка, подходы и повторы. диалог или отдельный экран? мне кажется диалога хватит, полей мало

---

после закрытия диалога и сворачивания приложения падает:

```
java.lang.IllegalStateException: Fragment WorkoutFragment{5c1a2e} not attached to a context.
    at androidx.fragment.app.Fragment.requireContext(Fragment.java:972)
    at com.fitquest.app.ui.workout.WorkoutFragment.lambda$loadPlans$2(WorkoutFragment.java:118)
```

ответ приходит когда фрагмента уже нет. проверять isAdded в колбэке или как правильно

---

секундомер тренировки локальный, но при сворачивании прилоения он сбивается. храни время старта, а не тикай счётчиком

---

история тренировок: дата, программа, длительность, xp. дату с backend получаю в utc, показывай в локальном времени телефона, а то тренировка в 23:30 улетает на следующий день

---

Progress и Notifications. уведомления: непрочитанные выделить, по тапу — POST read

---

после POST read список перезагружается целиком и прыгает наверх. обнови только один элемент в адаптере

---

на Progress недельная сводка: тренировки по дням столбиками. без библиотеки графиков, просто view с высотой по значению, норм?

---

## Сеанс 3

live watch: WebSocket /telemetry/ws, если сокет отвалился — опрос /telemetry/latest раз в 5 сек, а когда сокет вернётся, опрос остановить
backend ждёт токен первым сообщением после подключения: {"token": "..."}, а не в заголовке. как это сделать в okhttp WebSocketListener, в onOpen?
и что показывать на экране, пока идёт переключение сокет ↔ опрос

---

при обновлении пульса падает:

```
android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
    at com.fitquest.app.ui.telemetry.TelemetryFragment.showPoint(TelemetryFragment.java:141)
    at com.fitquest.app.ws.TelemetryWebSocket$1.onMessage(TelemetryWebSocket.java:64)
```

onMessage же не в главном потоке приходит. как правильно перекинуть

---

ушла с экрана live, а в логах опрос продолжается. Handler с postDelayed не снимается в onDestroyView? првоерь и сокет тоже закрывай

---

переподключение сокета сейчас сразу в onFailure, и если backend лежит, он долбит без остановки. сделай паузу, хотя бы 5 сек, и опрос в это время пусть работает

---

вход, регистрация, автовход если токен есть (splash)
профиль: адрес api можно поменять без пересборки, выход
нижняя навигация на 5 вкладок, один контейнер фрагментов

---

нажимаю на вкладку Pack и приложение закрывается:

```
android.content.res.Resources$NotFoundException: Resource ID #0x7f0800a3
    at android.content.res.ResourcesImpl.getValue(ResourcesImpl.java:248)
    at com.google.android.material.navigation.NavigationBarMenuView.buildMenuView
```

иконку я переименовала, но в menu xml вроде поправила. Build -> Clean Project не помог

---

на splash токен есть, но он уже протух, и меня кидает на Den с ошибкой 401 вместо логина. проверь токен через /auth/me перед тем как пускать дальше

---

при выходе чистить токен, и чтобы кнопкой назад нельзя было вернуться в приложение. FLAG_ACTIVITY_CLEAR_TASK?

---

вкладки: Den · Quest · Trail · Calls · Pack. при переключении фрагмент каждый раз создаётся заново и грузит данные, можно show/hide вместо replace?

---

поменяла адрес api в профиле, а retrofit всё равно ходит на старый. он же создаётся один раз в синглтоне, пересоздавай клиент при смене адреса

---

в профиле ещё подписка: тариф Free/Pro и кнопка оформить (mock checkout). после checkout статус не обновляется пока не перезайдёшь на вкладку, перезапроси сразу

---

## Сеанс 4

**Задача.** Вынести проверку форм из `AuthActivity` в чистый класс `AuthValidator` и покрыть unit-тестами.

**Требования.**
- Правила как у backend: email, пароль с буквами и цифрами, совпадение паролей.
- Класс без Android-зависимостей, тесты на JVM без эмулятора.

**Критерий приёмки.** `./gradlew testDebugUnitTest` проходит

---

тест на email падает:

```
java.lang.NullPointerException: Cannot invoke "java.util.regex.Pattern.matcher(java.lang.CharSequence)"
because "android.util.Patterns.EMAIL_ADDRESS" is null
    at com.fitquest.app.auth.AuthValidator.isValidEmail(AuthValidator.java:21)
```

это android класс, в jvm тестах его нет. замени на обычную regex, robolectric ради этого не подключай

---

ещё тесты на DTO (разбор json с null полями) и на выбор настроения питомца

---

в студии тесты запускаются зелёные, а через ./gradlew testDebugUnitTest в терминале пишет «No tests found for given includes». у меня папка test а не androidTest, всё правильно же?

---

ci в github actions: testDebugUnitTest, lint, сборка apk как артефакт
lint падает на предупреждениях, которые не относятся к нам (версии зависимостей). как сделать, чтобы он падал только на ошибках

---

в ci:

```
Run ./gradlew testDebugUnitTest
/home/runner/work/_temp/3f1c.sh: line 1: ./gradlew: Permission denied
Error: Process completed with exit code 126.
```

chmod в шаге делать или git update-index --chmod=+x? второе вроде правильнее

---

job собирается на каждый пуш в любую папку монорепы, а надо только когда менялся mobile. фильтр по путям сделай

---

сборка в ci 6 минут, gradle каждый раз качает всё заново. кэш добавь

---

README: запуск, тесты, структура. и SCREENS.md — какой экран какие эндпоинты дёргает

---

в SCREENS.md табличкой: экран | метод | эндпоинт | когда вызывается. и проверь по FitQuestApi что ничего не забыла

---

## Сеанс 5

**Задача.** Экран «Источники данных» (AND-09) вместо Live watch.

1. Подключённые устройства: последние данные, отвязка (`GET /devices`, `DELETE /devices/{id}`).
2. Raspberry Pi по коду: `POST /devices/pairing`, показать код и готовые строки запуска узла, ждать, пока узел появится в `GET /devices`.
3. Часы через Health Connect: шаги и пульс за 24 часа, 5-минутные точки -> `POST /telemetry/batch`.

**Ограничение.** Без новых зависимостей, если получится

---

health connect по разрешениям не понимаю: где их запрашивать, если пользователь отказал — что показывать? объясни, код потом напишу сама

---

на android 14+ health connect встроен в систему, а ниже нужно отдельное приложение. давай только 14+, а ниже понятное сообщение «не поддерживается на этой версии», без крашей

---

на эмуляторе api 34 запрос разрешений сразу возвращает отказ, окно даже не показывается. в logcat:

```
W/HealthConnectManager: Permission android.permission.health.READ_STEPS is not declared in the manifest
```

в манифесте я их добавила. может нужен ещё activity-alias с ACTION_SHOW_PERMISSIONS_RATIONALE?

---

**Задача.** Чистый класс `TelemetryBuckets`: записи Health Connect -> точки `TelemetryDTO`.

**Требования.**
- Корзины по 5 минут, точка на конец корзины.
- `steps` накопительные с полуночи по локальному времени, как на часах.
- `heart_rate` — среднее за корзину, значения вне 30–230 отбрасывать.
- Корзина без пульса пропускается: в контракте backend пульс обязателен.

**Критерий приёмки.** Unit-тесты без эмулятора, в том числе корзина через полночь

---

шаги в StepsRecord идут интервалом start-end, а интервал может попасть на две корзины. делить пропорционально или класть целиком в ту где конец? для 5 минут разница копеечная, делай как проще, но в коде напиши

---

копирую строку запуска pi с экрана, а там код в кавычках-ёлочках и bash ругается. сделай кнопку «скопировать» и обычные кавычки

---

пока ждём узел, экран опрашивает GET /devices раз в 3 сек. код живёт 10 минут — после этого опрос остановить и показать «код истёк, получите новый»

---

источник точки в списке сейчас пишется как raspberry_pi / health_connect. сделай по-человечески: «Raspberry Pi», «Часы (Health Connect)». в отдельный SourceLabels, чтоб тест написать

---

на pi указали BACKEND_URL=http://10.0.2.2:8000 как в приложении, и он не подключается. на экране кода надо подсказку: localhost если backend на самой pi, иначе ip компьютера в wi-fi

---

экран квестов и достижений, backend отдаёт /pet/quests и /pet/achievements
сравни QuestDto и AchievementDto с ответом backend, каких полей не хватает. часть приходит null

---

progress и target у квеста приходят, а period нет, он в quest.period вложенный. и у достижения unlocked_at может быть null если не получено, покахывай серым

---

прогресс квеста 3/3 а полоска не до конца, ProgressBar max по умолчанию 100, а ставлю progress=3. поставь max=target

---

## Сеанс 6

добавили картинку sleeping, когда питомец спит (pet.is_sleeping). выбор картинки по настроению вынеси в UiFormat

---

положила pet-sleeping.png в drawable, а сборка:

```
ERROR: /app/src/main/res/drawable/pet-sleeping.png: Resource name 'pet-sleeping' is not valid:
'-' is not a valid file-based resource name character: File-based resource names must contain only lowercase a-z, 0-9, or underscore
```

переименую в pet_sleeping, в коде где-то ещё на него ссылки поправь

---

is_sleeping главнее настроения, даже если mood happy. тест на это допиши

---

в доке README ещё старые названия вкладок и скриншот без sleeping, поправь текст, скрин я сама переделаю

---

## Сеанс 7

чек-лист подходов в сессии: упражнения -> подходы, отметка выполненного, прогресс сессии сверху
для сессии сделаем SessionViewModel, чтобы прогресс не терялся при повороте
таймер отдыха RestTimer — чистый класс без android-зависимостей, чтобы покрыть тестами. время подставлять снаружи, без System.currentTimeMillis внутри

---

new ViewModelProvider(this).get(SessionViewModel.class) не компилируется, Cannot resolve constructor. lifecycle-viewmodel надо отдельно добавлять или он в appcompat уже есть?

---

состояние чек-листа тоже в чистый класс SessionProgress, а ViewModel только держит его. статусы подхода: pending, done, skipped. прогресс считать по done + skipped или только done?

---

тесты RestTimer: старт, пауза, продолжение, окончание, повторный старт во время отдыха. и что будет если время пришло меньше предыдущего, не уходить в минус

---

тесты на SessionProgress: отметить, пропустить, пропустить всё упражнение целиком, прогресс 0 если упражнений нет (сейчас там деление на ноль)

---

в альбомной ориентации чек-лист и таймер не влезают. сделай layout-land: слева список, справа таймер

---

сделала layout-land, а при повороте падает:

```
java.lang.NullPointerException: Attempt to invoke virtual method
'void android.widget.TextView.setText(java.lang.CharSequence)' on a null object reference
    at com.fitquest.app.ui.workout.SessionActivity.renderTimer(SessionActivity.java:204)
```

в land разметке забыла tv_rest_timer или id другой? проверь что все id совпадают в обеих

---

при отметке подхода хочу вводить фактические повторы и вес, по умолчанию подставлять плановые. если поле пустое — отправлять плановое, а не null

---

отметка подхода -> POST /workouts/sessions/{id}/sets. если запрос упал, галочку откатить и показать тост, а не молча оставлять

---

вибрация по окончании отдыха. на api 26+ VibrationEffect, ниже Vibrator.vibrate(long) — нам ниже 26 не нужно, minSdk 26, упрости

---

разрешение VIBRATE в манифест не забудь, у меня без него падает SecurityException

---

getSystemService(Vibrator.class) студия зачёркивает как deprecated на 31+, говорит VibratorManager. переделывать или оставить с проверкой версии?

---

на эмуляторе вибрацию не проверить. можно в debug ещё короткий звук или хотябы лог, чтоб видно было что сработало

---

SessionActivity убрать из манифеста (ветка с ней ещё не готова), проверить, что сборка и тесты проходят
и верни API_BASE_URL по умолчанию на 10.0.2.2, я закоммитила свой ip

---

убрала из манифеста, сборка ок, а при старте тренировки краш:

```
android.content.ActivityNotFoundException: Unable to find explicit activity class
{com.fitquest.app/com.fitquest.app.ui.workout.SessionActivity}; have you declared this activity in your AndroidManifest.xml?
```

значит startActivity где-то в WorkoutFragment остался. найди все места и пока верни старое поведение без чек-листа
