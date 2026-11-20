# Отладка и диагностика рантайма

[← К оглавлению](../README.md)

> [!NOTE]
> TL;DR — Миксин врёт только в одном месте — в момент применения. Включайте `-Dmixin.debug.verbose=true` и `-Dmixin.debug.export=true`, читайте `.mixin.out/class`, а для «почему не сработало» идите в [`04-mixin-debugging.md`](../04-mixin-debugging.md). Для производительности есть spark, JFR и async-profiler; для автоматизации — GameTest, MC-Runtime-Test и HeadlessMc.

## Содержание

- [Системные свойства Mixin](#системные-свойства-mixin)
  - [Полная таблица](#полная-таблица)
  - [Чего не существует: `mixin.remap`](#чего-не-существует-mixinremap)
- [Куда вставлять флаги](#куда-вставлять-флаги)
  - [Gradle: ModDevGradle](#gradle-moddevgradle)
  - [Gradle: Fabric Loom](#gradle-fabric-loom)
  - [IntelliJ IDEA](#intellij-idea)
  - [`JAVA_TOOL_OPTIONS`](#java_tool_options)
  - [Прод-запуск](#прод-запуск)
- [Чтение логов Mixin](#чтение-логов-mixin)
  - [Формат и уровни](#формат-и-уровни)
  - [Характерные строки](#характерные-строки)
  - [Warning или fatal](#warning-или-fatal)
- [Инструменты](#инструменты)
  - [MixinTrace](#mixintrace)
  - [Crash Assistant и Not Enough Crashes](#crash-assistant-и-not-enough-crashes)
  - [`mixin.debug.export` + CFR/Vineflower](#mixindebugexport--cfrvineflower)
- [Hotswap](#hotswap)
  - [Ограничения HotSpot](#ограничения-hotspot)
  - [JBR и enhanced class redefinition](#jbr-и-enhanced-class-redefinition)
  - [Hotswap миксинов](#hotswap-миксинов)
- [Профилирование](#профилирование)
  - [spark](#spark)
  - [JFR](#jfr)
  - [async-profiler](#async-profiler)
  - [Что искать в моде](#что-искать-в-моде)
- [Автотесты](#автотесты)
  - [GameTest: vanilla, Fabric, NeoForge](#gametest-vanilla-fabric-neoforge)
  - [MC-Runtime-Test и HeadlessMc](#mc-runtime-test-и-headlessmc)
  - [Когда что](#когда-что)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Системные свойства Mixin

Все перечисленные ниже свойства — это **JVM system properties**, а не переменные окружения и не поля в `mixins.json`. Mixin читает их один раз при инициализации окружения (`MixinEnvironment`), поэтому:

- менять их из кода мода через `System.setProperty(...)` **поздно**;
- после изменения нужен перезапуск JVM;
- в именах важна каждая точка: `mixin.debug.export` и `mixin.debug.exports` — разные вещи (второго не существует).

Значения булевых флагов — `true`/`false`. Регистр значения не важен (`Boolean.parseBoolean`), регистр имени — важен.

### Полная таблица

Колонка «Значение» показывает, что писать после `=`. Для флагов это `true`, потому что по умолчанию `false`.

| Свойство | Значение | Что делает | Куда смотреть | Когда включать |
| --- | --- | --- | --- | --- |
| `mixin.debug` | `true` | Включает **все** debug-опции (`DEBUG_ALL`): export, verbose, verify, strict, profiler и т.д. | Всё ниже | Почти никогда: это не «побольше логов», а «весь отладочный ад одновременно» |
| `mixin.debug.export` | `true` | Экспортирует **post-mixin** байткод каждого target-класса, к которому применился миксин, в `.mixin.out/class` | `<runDir>/.mixin.out/class/net/minecraft/...` | Первое, что нужно, когда надо увидеть, что реально стало с классом |
| `mixin.debug.export.filter` | glob, например `net.minecraft.**` | Ограничивает экспорт классами, чьё имя подходит под шаблон. `*` — один или более символов кроме точки, `**` — любое число символов, `?` — ровно один | — | Большой пак: иначе экспорт пишет гигабайты |
| `mixin.debug.export.decompile` | `true` / `false` | Разрешает декомпиляцию экспортированных классов **Fernflower**, если он есть на classpath. По умолчанию наследует `mixin.debug.export`, то есть при включённом export равно `true`; `false` принудительно отключает декомпилятор | `<runDir>/.mixin.out/java/...` | Только если Fernflower реально есть в runtime; иначе см. CFR/Vineflower ниже |
| `mixin.debug.export.decompile.async` | `true` | Декомпилирует экспортированные классы в отдельном потоке | — | Старт быстрее, но при краше часть дампов может остаться недоделанной |
| `mixin.debug.export.decompile.mergeGenericSignatures` | `true` / `false` | Сливает generic-сигнатуры миксина в target при экспорте. `false` — отключить | — | Ломается generic-рефлексия на экспортированных классах |
| `mixin.debug.verbose` | `true` | Поднимает все `DEBUG`-сообщения Mixin до `INFO`. Именно после этого в обычном логе появляются строки `Mixing ...` | `logs/latest.log`, консоль | **Всегда первым делом** при отладке миксинов |
| `mixin.debug.countInjections` | `true` | Включает строгую проверку числа инъекций: если `expect` задан и не совпал, бросает `Injection validation failed` | Лог, exception | Ловить «тихие» промахи в dev |
| `mixin.debug.verify` | `true` | Прогоняет `CheckClassAdapter` по post-mixin байткоду и включает строгие проверки; некоторые warning'и становятся исключениями. Например, запись в `@Final`-поле начинает падать | Лог, exception | Только при разработке самого Mixin, не в обычном моде |
| `mixin.debug.strict` | `true` | Включает строгие проверки: `strict.unique` и `strict.targets` | Лог | Когда надо, чтобы опечатки в target'ах падали сразу |
| `mixin.debug.strict.unique` | `true` | Публичный `@Unique`-метод, конфликтующий с уже существующим в target, становится исключением, а не warning'ом | Лог | Конфликты `@Unique` |
| `mixin.debug.strict.targets` | `true` | `@Mixin target ... was not found` становится исключением, а не warning'ом | Лог | Проверка правильности `@Mixin(Target.class)` |
| `mixin.debug.profiler` | `true` | Включает профайлер **операций Mixin** (не игрового тика) | Лог | Диагностика долгого старта из-за миксинов |
| `mixin.dumpTargetOnFailure` | `true` | При падении применения дампит **входящий**, ещё не изменённый байткод target'а | `<runDir>/.mixin.out/class/<Class>.target.class` | Когда `InvalidMixinException` говорит «target не в том состоянии» |
| `mixin.checks` | `true` | Включает все проверки (`CHECK_ALL`), сейчас это `checks.interfaces` | — | Редко; обычно включают точечно |
| `mixin.checks.interfaces` | `true` | Аудит: какие методы интерфейса объявлены, но не реализованы (потенциальный `AbstractMethodError`) | `<runDir>/.mixin.out/audit/*.txt`, `*.csv` и stderr | Interface-миксины и duck-интерфейсы |
| `mixin.checks.interfaces.strict` | `false` | При включённом аудите пропускает абстрактные target-классы | — | Аудит абстрактных иерархий |
| `mixin.hotSwap` | `true` | Включает hot-swap-агент Mixin. **Сам по себе не работает**: нужен `-javaagent:<mixin.jar>` | Лог `Attempting to load Hot-Swap agent` | Итеративная разработка миксинов |
| `mixin.env.remapRefMap` | `true` | Ремапит существующий refmap в рантайме. В dev на Fabric включается автоматически, когда namespace мода и runtime различаются | Лог `Remapping refMap ...` | Dev-окружение ForgeGradle с SRG-refmap зависимостей |
| `mixin.env.refMapRemappingFile` | путь к `.srg` | Файл mappings для `remapRefMap` | — | Вместе с `mixin.env.remapRefMap=true` |
| `mixin.env.refMapRemappingEnv` | `searge` | Source-environment в refmap, который надо ремапить | — | Если source не `searge` |
| `mixin.env.allowPermissiveMatch` | `false` | Запрещает «permissive» fallback при ремапе refmap: при промахе по descriptor'у совпадение по имени больше не принимается | — | Ловить неверные descriptor'ы |
| `mixin.env.disableRefMap` | `true` | Отключает использование refmap: `getReferenceMapper()` возвращает default-mapper, soft-ссылки не переводятся | Лог | Когда refmap заведомо не нужен и мешает |
| `mixin.env.compatLevel` | `JAVA_17`, `JAVA_21`, `JAVA_25` | Default compatibility level, если он не задан в конфиге | — | Почти никогда: правильнее в `mixins.json` |
| `mixin.env.ignoreRequired` | `true` | Игнорирует `required` у всех конфигов | — | Только для диагностики: «что будет, если пропустить сломанный миксин» |
| `mixin.env.shiftByViolation` | `ignore`, `warn`, `error` | Поведение при превышении `@At(by = ...)`; по умолчанию `warn` | — | Ловить `by`-костыли |
| `mixin.ignoreConstraints` | `true` | Constraint violations становятся warning'ами, а не исключениями | — | Тестирование «чужих» target'ов |
| `mixin.initialiserInjectionMode` | `default`, `safe` | Режим инъекций в инициализаторы | — | Специфичные случаи `<init>`/`<clinit>` |
| `mixin.tunable.classReaderExpandFrames` | `true` | Читает байткод миксина с `ClassReader.EXPAND_FRAMES` (поведение ≤ 0.8.6) | — | Обход несовместимости с очень старыми миксинами |
| `mixin.remap` | — | **Не существует.** Mixin не читает такое свойство | — | Не использовать; см. ниже |

> [!WARNING]
> `-Dmixin.debug=true` включает `mixin.debug.export` **и** `mixin.debug.verify` **и** `mixin.debug.profiler`. На большом паке это легко даёт десятки гигабайт в `.mixin.out`, замедление старта в разы и новые падения там, где раньше был только warning. Не включайте его «на всякий случай» и не оставляйте в прод-скрипте.

> [!WARNING]
> `mixin.debug.export` очищает `.mixin.out/class` при создании экспортёра, то есть при следующем запуске. Если дамп нужен — скопируйте `.mixin.out` **до** перезапуска игры.

### Чего не существует: `mixin.remap`

В официальной таблице системных свойств Mixin ([Mixin Java System Properties]) свойства `mixin.remap` нет. В `MixinEnvironment.Option` (Mixin 0.8.7, коммит `4053421`) тоже нет опции с таким именем. Если передать `-Dmixin.remap=false`, Mixin молча проигнорирует флаг.

`remap` — это **атрибут аннотации**, а не системное свойство:

```java
@Mixin(value = SomeModClass.class, remap = false)
public class SomeModMixin { /* ... */ }

@Inject(method = "someMethod", at = @At("HEAD"), remap = false)
private void onSomeMethod(CallbackInfo ci) { /* ... */ }
```

Похожее по названию, но другое по смыслу: `mixin.env.remapRefMap` (ремап refmap), `mixin.env.disableRefMap` (не читать refmap), `mixin.env.refMapRemappingFile` (файл mappings). Подробно про `remap` и namespace — в [`04-mixin-debugging.md`](../04-mixin-debugging.md#refmap-и-namespace).

---

## Куда вставлять флаги

Общее правило: флаг должен попасть в **аргументы JVM процесса игры**, а не в аргументы `main`, не в `gradle.properties` и не в `org.gradle.jvmargs` (последнее настраивает JVM Gradle-демона, а не игру).

### Gradle: ModDevGradle

ModDevGradle (MDG) даёт типизированный `systemProperty` у run-конфигурации:

```groovy
neoForge {
    runs {
        client {
            client()
            systemProperty "mixin.debug.verbose", "true"
            systemProperty "mixin.debug.export", "true"
            // при необходимости — JVM-аргумент, а не system property:
            jvmArgument "-XX:+AllowEnhancedClassRedefinition"
        }
    }
}
```

То же самое для всех запусков сразу:

```groovy
neoForge {
    runs {
        configureEach {
            systemProperty "mixin.debug.verbose", "true"
        }
    }
}
```

После правки build-скрипта перегенерируйте run-конфигурации (в IDEA — Gradle sync / `neoForge.ideSyncTask`), иначе старая конфигурация останется без флага.

### Gradle: Fabric Loom

Loom 1.17+ использует property-based API; в старых версиях — `property(...)`/`vmArg(...)`:

```groovy
loom {
    runs {
        client {
            // Loom 1.17+
            systemProperties.put("mixin.debug.verbose", "true")
            systemProperties.put("mixin.debug.export", "true")

            // Старый API (до 1.17):
            // property "mixin.debug.verbose", "true"
            // property "mixin.debug.export", "true"

            // Если нужен именно JVM-аргумент:
            // jvmArguments.add("-XX:+AllowEnhancedClassRedefinition")
            // vmArg "-XX:+AllowEnhancedClassRedefinition"   // старый API
        }
    }
}
```

Для hotswap миксинов Loom-документация отдельно просит `-javaagent:<mixin.jar>`; это тоже JVM-аргумент, а не system property.

### IntelliJ IDEA

`Run` → `Edit Configurations...` → нужная конфигурация (`Minecraft Client` / `Minecraft Server` / Gradle-run) → поле **VM options**:

```text
-Dmixin.debug.verbose=true -Dmixin.debug.export=true
```

Не путайте с **Program arguments**: `-D...` там окажется аргументом `main` и до JVM не дойдёт. Если конфигурация генерируется Gradle-плагином, правка в IDEA может быть перезаписана при следующем sync — тогда меняйте `build.gradle`.

### `JAVA_TOOL_OPTIONS`

Самый быстрый способ «подкрасить» вообще все JVM, которые стартуют из текущей shell-сессии:

```bash
JAVA_TOOL_OPTIONS="-Dmixin.debug.verbose=true -Dmixin.debug.export=true" ./gradlew runClient
```

JVM напечатает в начале:

```text
Picked up JAVA_TOOL_OPTIONS: -Dmixin.debug.verbose=true -Dmixin.debug.export=true
```

> [!WARNING]
> `JAVA_TOOL_OPTIONS` наследуется **всеми** дочерними JVM: Gradle-демоном, Kotlin-компилятором, тестами, самой игрой. Это удобно для разового запуска и опасно как постоянная настройка: экспорт миксинов может включиться в сборочных процессах. Для постоянной работы используйте run-конфигурацию.

`_JAVA_OPTIONS` действует похожим образом, но имеет более высокий приоритет и тоже наследуется; в документации JVM он помечен как отладочный. Для воспроизводимости лучше не использовать ни то, ни другое.

### Прод-запуск

- **Обычный `java -jar`:** системные свойства идут **до** `-jar`:

  ```bash
  java -Dmixin.debug.verbose=true -Dmixin.debug.export=true -jar server.jar nogui
  ```

  После `-jar` они станут аргументами приложения и будут проигнорированы.

- **Forge/NeoForge server:** установщик создаёт `user_jvm_args.txt`. Свойства добавляются туда по одному на строку:

  ```text
  -Xmx6G
  -Dmixin.debug.verbose=true
  -Dmixin.debug.export=true
  ```

  Сам скрипт запускает `java @user_jvm_args.txt @libraries/.../unix_args.txt nogui`; `@`-файл разворачивается JVM до старта.

- **Панель хостинга:** часто редактируется только `JAVA_OPTS`/`JVM_ARGS`, а не командная строка. Проверьте, что панель реально передаёт значения в JVM, а не в аргументы сервера. Если нет — флаг не сработает, и это не баг Mixin.

- **Fabric server:** `java -D... -jar fabric-server-launch.jar nogui` (или `server.jar` в новом лаунчере).

В проде `mixin.debug.export` почти всегда плохая идея: диск, время старта и риск утечки внутренних классов. Для прода полезнее `mixin.debug.verbose` на один запуск и `mixin.dumpTargetOnFailure` при разборе конкретного краша.

---

## Чтение логов Mixin

### Формат и уровни

Mixin пишет через логгер `mixin` (Fabric: `[mixin/]`, NeoForge: `[mixin/]`, в старых сборках — `[Mixin]`). Смотреть нужно:

- `logs/latest.log` — основной лог;
- `logs/debug.log` — у Forge/NeoForge часто содержит больше;
- `crash-reports/crash-*.txt` — если игра упала;
- stdout консоли — если лог не успел сброситься на диск.

Первая строка при старте:

```text
[main/INFO] [mixin/]: SpongePowered MIXIN Subsystem Version=0.8.7 Source=... Service=... Env=CLIENT
```

Она подтверждает, что Mixin вообще загрузился, и показывает версию/сервис. Если этой строки нет — проблема не в миксине, а в лоадере/запуске.

Строка `Mixing ...` имеет уровень `DEBUG` и **не видна** в обычном логе. Её поднимает `mixin.debug.verbose=true`:

```text
[main/INFO] [mixin/]: Mixing example.mixin.MinecraftServerMixin from example.mixins.json into net.minecraft.server.MinecraftServer
```

### Характерные строки

Ниже — строки, которые реально формирует Mixin 0.8.7 (шаблоны из исходников; `<...>` — подстановки). Примеры с реальными именами, где они приводятся, взяты из issue-тредов и показывают формат, а не гарантируют те же имена в вашей версии.

| Строка (шаблон) | Уровень | Что означает |
| --- | --- | --- |
| `Mixing <mixin> from <config> into <target>` | INFO только при verbose, иначе DEBUG | Mixin пытается применить миксин к target-классу. Это **не** значит, что инъекция нашла точку — только что класс дошёл до этой стадии |
| `Critical injection failure: <описание> <handler><desc> in <mixin> failed injection check, (<N>/<M>) succeeded. Scanned <K> target(s). <refmap status>` | ERROR / exception | Инъектор нашёл меньше обязательных точек, чем требует `require`/`defaultRequire`. Обычно фатально |
| `Critical injection failure: <описание> could not find any targets matching <selector> in <target>. <refmap status>` | ERROR / exception | Не найден сам метод/поле, в которое надо инжектить |
| `Invalid descriptor on <injector>! CallbackInfoReturnable is required!` | ERROR / exception | У target'а есть возвращаемое значение, а handler объявил `CallbackInfo` |
| `Invalid descriptor on <injector>! Expected <desc> but found <desc>` | ERROR / exception | Сигнатура handler'а не совпала с сигнатурой target'а |
| `@Redirect conflict. Skipping <injector> with priority <N>, already redirected by <other> with priority <M>` | WARN | Два мода (или два миксина) целятся в одну и ту же инструкцию. Второй `@Redirect` пропущен |
| `@Redirect conflict: <injector> failed because target was already remapped by <other>` | ERROR / exception | Чужой redirect помечен `@Final`, перебить нельзя |
| `Method overwrite conflict for <method> in <mixin>, previously written by <owner>. Skipping method.` | WARN | Два `@Overwrite`/merged-метода на одно имя; второй пропущен |
| `Overwrite prohibited for @Final method <method> in <mixin>. Skipping method.` | WARN | Цель помечена `@Final` другим миксином |
| `@Mixin target <class> was not found <mixin>` | WARN (ERROR при strict.targets) | Класс-цель не найден: опечатка, не тот side, класс ещё не загружен/уже загружен, другой classloader |
| `@Shadow field <name> was not located in the target class <target>. <refmap status>` | ERROR / exception | Поле не найдено: переименовано, другой тип, не тот namespace, refmap не загружен |
| `@Shadow method <name> in <mixin> was not located in the target class <target>. <refmap status>` | ERROR / exception | То же для метода |
| `Reference map '<file>' for <config> could not be read. If this is a development environment you can ignore this message` | WARN | В конфиге указан `refmap`, но файла нет/он битый. В dev часто безобидно, в prod — тревожный признак |
| `Compatibility level <LEVEL> specified by <config> is lower than the default level supported by the current mixin service (<LEVEL>).` | DEBUG/INFO | Конфиг занижает уровень; не ошибка |
| `Compatibility level <LEVEL> specified by <config> is higher than the maximum level supported by this version of mixin (<LEVEL>).` | DEBUG/WARN | Конфиг завышает уровень; на старом Mixin может стать ошибкой |
| `Mixin config <name> does not specify "minVersion" or "requiredFeatures" property` | ERROR (но не fatal) | В конфиге нет `minVersion`/`requiredFeatures`. Совет автору, игра продолжается |
| `Mixin config <name> requires mixin subsystem version <v> but <v> was found. The mixin config will not be applied.` | WARN или exception | Конфиг требует более новый Mixin. Если `required: true` — падение |
| `Critical problem: <mixin> target <class> was loaded too early.` | ERROR / exception | Target загружен до того, как Mixin смог его обработать. Часто виноват другой early-трансформер |
| `Attempting to load Hot-Swap agent` | INFO | `mixin.hotSwap=true` и агент найден |
| `Hot-swap agent could not be loaded, hot swapping of mixins won't work. <Class>: <msg>` | INFO | Флаг есть, а агента нет: забыли `-javaagent` или версия не совпала |
| `Redefining mixin <class>` / `Re-transforming target class <class>` | INFO | Hotswap переприменяет миксин |
| `Mixin <class> cannot be reloaded, needs a restart to be applied: <msg>` | ERROR | Hotswap не смог: структурное изменение или ограничение JVM |
| `Mixin apply for mod <mod> failed <config>:<Mixin> from mod <mod> -> <target>: <exception>` | ERROR | Обёртка Fabric вокруг fatal-ошибки применения |
| `Mixin apply failed <config>:<Mixin> -> <target>: <exception>` | ERROR | Обёртка NeoForge/Forge вокруг fatal-ошибки |
| `MixinTransformerError: An unexpected critical error was encountered` | ERROR | Внешняя обёртка; настоящая причина — в последнем `Caused by:` |
| `MixinApplyError: Mixin [<config>:<Mixin>] from phase [<PHASE>] in config [<config>] FAILED during APPLY` | ERROR | Фаза применения; читайте вложенное исключение |

> [!NOTE]
> Строк `Injecting ...`, `Redirecting ...` и `Apply mixin ...` в Mixin 0.8.7 **нет** как отдельных сообщений. В старых 0.6/0.7 и в некоторых сторонних инструментах они встречались. В актуальном Mixin ориентируйтесь на `Mixing ...`, описания инъекторов вида `Mixin->@Inject::handler(...)` внутри ошибок и activity trail в квадратных скобках (`[PREINJECT Applicator Phase -> ...]`). Если вы ищете в логе `Apply mixin`, вы ищете строку не из этой версии.

Activity trail в исключении — самый полезный кусок, потому что показывает фазу:

```text
[PREINJECT Applicator Phase -> example.mixins.json:MinecraftServerMixin -> Prepare Injections ->
 handler$...$onTick(...) -> Parse -> Validate Targets]
```

Фазы, которые стоит различать:

- `PREPARE`/`PREINJECT` — миксин ещё не применялся: не найден target-класс, `@Shadow`, descriptor, конфликт аннотаций.
- `APPLY`/`INJECT_PREPARE` — ищем точки инъекции: не найден метод или `@At`.
- `INJECT_APPLY` — точки найдены, но при вставке что-то пошло не так (конфликт redirect, несовместимый handler).

### Warning или fatal

Уровень лога сам по себе **не** говорит, упала ли игра:

- `logger.error` в Mixin не всегда бросает исключение. Например, `Write access detected to @Final field ...` — это ERROR, но по умолчанию применение продолжается; исключение будет только при `mixin.debug.verify=true`.
- `Mixin config ... does not specify "minVersion"...` — ERROR, после которого игра спокойно стартует.
- `Reference map ... could not be read` — WARN, в dev почти всегда норма.
- `@Mixin target ... was not found` — WARN, миксин просто пропущен; при `mixin.debug.strict.targets=true` становится исключением.
- `Critical injection failure` — обычно `InjectionError`/`InvalidInjectionException`; если конфиг `required: false`, лоадер может поймать его, залогировать и продолжить. Тогда это «warning с ужасным текстом».

Практическое правило: **fatal — это наличие исключения в цепочке `Caused by:` и/или строки `FAILED during APPLY/PREPARE`, а не слово ERROR.** Если игра продолжила загрузку, а в логе остался WARN/ERROR без stack trace — это диагностика, а не причина падения.

Быстрый фильтр по логу:

```bash
grep -nE "Critical injection failure|Mixin apply|FAILED during|Caused by:.*(Mixin|Injection)|@Shadow|@Redirect conflict|was not found" logs/latest.log
```

---

## Инструменты

### MixinTrace

[MixinTrace] [community] (`comp500/mixintrace`; это не проект FabricMC, хотя идея позже обсуждалась для `fabric-crash-report-info-v1`) — Fabric/Quilt-мод, который добавляет в crash-report блок «Mixins in Stacktrace»: для каждого класса из стектрейса перечислены миксины и их config-файлы.

Пример формата из README проекта:

```text
Mixins in Stacktrace:
    net.minecraft.class_465:
        dev.emi.bunchotrinkets.mixin.AbstractContainerScreenMixin (bunchotrinkets.mixins.json)
        net.backslot.mixin.HandledScreenMixin (backslot.mixins.json)
```

Когда полезен: краш внутри ванильного метода, и непонятно, чей миксин его сломал. Когда бесполезен: ошибка на этапе применения (там и так есть имя конфига) и NeoForge (нужен форк; оригинал — Fabric/Quilt). Последний релиз на CurseForge — `1.1.1+1.17` (2022), поэтому на 1.21+/26.x проверяйте совместимость и форки; не считайте его обязательным.

### Crash Assistant и Not Enough Crashes

- **[Crash Assistant] [community]** (`KostromDan/Crash-Assistant`) — GUI после краша: показывает логи, crash-report, `hs_err`, умеет загружать их и анализировать известные причины, подсказывает auto-fix. Поддерживает Forge/NeoForge/Fabric (набор веток зависит от версии игры). Удобен для пользователей и для быстрого «что вообще произошло», но его анализ — эвристика, а не истина.
- **[Not Enough Crashes] [community]** (`natanfudge/Not-Enough-Crashes`) — возвращает в главное меню вместо закрытия игры, хранит отчёты локально, умеет собирать repro-ZIP с версиями модов и конфигами, показывает список замешанных модов и **mixin source filenames and line numbers** в стектрейсах. Есть Fabric- и NeoForge-сборки.

Для разработчика полезнее не «красивое окно», а сырой `crash-reports/*.txt` плюс `mixin.debug.verbose`: инструменты могут скрыть важную цепочку `Caused by`.

### `mixin.debug.export` + CFR/Vineflower

Самый надёжный способ понять, что миксин сделал с классом:

```bash
# 1. Запустить игру с экспортом (флаг — в run-конфигурации или JAVA_TOOL_OPTIONS)
# 2. Из рабочей директории запуска скопировать дамп, пока не перезапустили игру
cp -r .mixin.out /tmp/mixin-dump

# 3. Посмотреть байткод
javap -p -c -constants /tmp/mixin-dump/class/net/minecraft/server/MinecraftServer.class

# 4. Или декомпилировать
java -jar cfr.jar /tmp/mixin-dump/class/net/minecraft/server/MinecraftServer.class
java -jar vineflower.jar /tmp/mixin-dump/class/net/minecraft/server/MinecraftServer.class /tmp/decompiled
```

Что смотреть:

- появился ли handler-метод (`handler$...`/`modid$...`) в целевом классе;
- на месте ли вызов, в который вы целились;
- не переписал ли вызов другой миксин;
- совпадает ли descriptor с тем, что вы написали в `@At(target = ...)`.

`mixin.debug.export.decompile=true` умеет декомпилировать сам, но только если на classpath есть Fernflower. Loom/MDG используют Vineflower для исходников, а Mixin ищет именно Fernflower; в runtime его обычно нет, поэтому в логе появляется примерно такое:

```text
Fernflower could not be loaded, exported classes will not be decompiled. ClassNotFoundException: <...>
```

Это не ошибка вашего миксина. Используйте внешний CFR/Vineflower: он не зависит от того, что положил в runtime лоадер.

> [!WARNING]
> CFR и Vineflower тоже не всесильны: на class-файлах новее поддерживаемой версии они могут упасть с `Unsupported class file major version`. Для 26.x (class version 69) берите свежие сборки декомпиляторов; если не выходит — `javap` всегда покажет байткод.

---

## Hotswap

### Ограничения HotSpot

Стандартный HotSpot через `Instrumentation.redefineClasses`/`retransformClasses` позволяет менять **тела методов**, константный пул и некоторые атрибуты. Он не позволяет:

- добавлять/удалять/переименовывать поля и методы;
- менять сигнатуры методов;
- менять superclass/interfaces;
- менять `final`-модификаторы и, как правило, аннотации класса.

Типичные ошибки при попытке:

```text
java.lang.UnsupportedOperationException: class redefinition failed: attempted to add a method
java.lang.UnsupportedOperationException: class redefinition failed: attempted to change the schema (add/remove fields)
java.lang.UnsupportedOperationException: class redefinition failed: attempted to change superclass or interfaces
```

Важно: класс должен быть запущен **в debug-режиме** (IDE подключается по JDWP). В Run-режиме кнопка «Reload changed classes» не сработает, потому что нет агента/соединения.

### JBR и enhanced class redefinition

JetBrains Runtime (JBR 17/21/25) содержит DCEVM-подобную enhanced class redefinition. Она включается флагом:

```text
-XX:+AllowEnhancedClassRedefinition
```

Проверка, что ваш JVM это умеет:

```bash
java -XX:+AllowEnhancedClassRedefinition -version
```

Если команда падает с `Unrecognized VM option`, это не JBR (или сборка без DCEVM). JBR позволяет добавлять/удалять классы, методы и поля в большинстве случаев, но **смену superclass/interfaces** всё равно не гарантирует; для такого изменения нужен перезапуск.

| Изменение | Обычный HotSpot | JBR + `AllowEnhancedClassRedefinition` |
| --- | --- | --- |
| Тело метода | Да | Да |
| Константы, ветвления, вызовы внутри метода | Да | Да |
| Добавить метод | Нет | Обычно да |
| Добавить поле | Нет | Обычно да |
| Удалить метод/поле | Нет | Обычно да |
| Изменить сигнатуру метода | Нет | Частично/зависит от сборки |
| Сменить superclass/interfaces | Нет | Обычно нет |
| Новый класс | Нет | Да |
| Сменить аннотации класса | Ограниченно | Обычно да |

### Hotswap миксинов

Миксин — это не обычный класс: его тело исполняется не само по себе, а вставлено в target. Поэтому одного IDE-хотсвопа недостаточно. Нужен агент Mixin:

1. Найдите jar Mixin, который уже использует лоадер:

   ```bash
   find ~/.gradle/caches -iname "*mixin*.jar" | grep -Ei "sponge|mixin" | head
   ```

2. Добавьте в VM options run-конфигурации:

   ```text
   -javaagent:/absolute/path/to/sponge-mixin-<version>.jar
   -Dmixin.hotSwap=true
   ```

3. Запустите игру **в debug-режиме**.

В логе появятся сообщения (префикс логгера зависит от лоадера; уровни — INFO/DEBUG):

```text
Attempting to load Hot-Swap agent
Redefining mixin com.example.mixin.FooMixin
Re-transforming target class net/minecraft/server/MinecraftServer
```

Что меняется на лету:

- тело уже существующего handler'а (`@Inject`, `@Redirect`, `@WrapOperation`);
- константы и условия внутри него;
- логика, не добавляющая новых членов в target.

Что почти наверняка потребует рестарта:

- новый миксин-класс или новый config;
- новый `@Shadow`/`@Unique` член, которого не было в target;
- новый handler-метод (target получает новый метод — обычный HotSpot это запретит; с JBR может сработать);
- смена `@At`/`method`, если это приводит к структурному изменению target;
- смена `@Mixin` target'а;
- `@Overwrite` с другой сигнатурой.

> [!WARNING]
> Версия Mixin в `-javaagent` должна совпадать с версией, которую использует лоадер. Иначе при старте получите `Environment conflict, mismatched versions or you didn't call MixinBootstrap.init()` или странные падения при переприменении. Если сомневаетесь — не добавляйте агент, перезапускайте игру: это медленнее, но предсказуемо.

> [!NOTE]
> Для Fabric есть отдельная документация по hotswap миксинов с `-javaagent`; для NeoForge/Forge механика та же (Mixin agent), но свои пути к jar'у и свои ограничения ModLauncher. Проверяйте, что агент видит те же классы, что и трансформер лоадера.

---

## Профилирование

### spark

[spark] — самый практичный профайлер для Minecraft: работает на клиенте и сервере, есть готовый viewer, низкий overhead. На клиенте Forge/Fabric команда называется `/sparkc`, на сервере — `/spark`.

CPU:

```text
/spark profiler start --timeout 60
/spark profiler stop
/spark profiler info
/spark profiler cancel
/spark profiler start --thread "Render thread" --timeout 30
/spark profiler start --thread "Server thread" --only-ticks-over 50 --timeout 120
```

- `--timeout <сек>` — автоматически остановить;
- `--thread <name>` — только нужный поток; `--thread *` — все;
- `--only-ticks-over <ms>` — писать сэмплы только с медленных тиков (ловить lag spike, а не средний фон);
- `--interval <ms>` — период сэмплирования (по умолчанию 4);
- `--force-java-sampler` — если async-profiler не работает (например, в контейнере).

Память и аллокации:

```text
/spark profiler start --alloc --timeout 60
/spark heapsummary
/spark heapdump --compress gzip
/spark gc
/spark gcmonitor
```

`/spark heapsummary` — быстрый обзор классов и объёма; `/spark heapdump` — полный `.hprof` для Eclipse MAT/VisualVM. Команды `/spark heap` **нет**; есть `heapsummary` и `heapdump`.

Тики и здоровье:

```text
/spark tps
/spark tickmonitor --threshold-tick 50
/spark health show --memory
/spark ping
```

Что смотреть в viewer'е: не общий процент, а **листовые кадры своего мода**. Если 40% времени в `ServerLevel.tick` и 35% из них — в `com.example.mod.tick`, это ваш тик. Если проценты размазаны по `java.util.stream`/`HashMap` — смотрите аллокации.

> [!NOTE]
> spark на клиенте и сервере — разные процессы. В singleplayer интегрированный сервер живёт в том же JVM, что и клиент, но потоки разные: `Render thread` и `Server thread`. Профилируйте тот, который болит.

### JFR

JDK Flight Recorder встроен в JVM, не требует мода и умеет CPU, аллокации, локи, GC, класс-лоадинг. Включается на старте:

```bash
java -XX:StartFlightRecording=filename=run.jfr,settings=profile,dumponexit=true -jar server.jar nogui
```

- `settings=default` — низкий overhead, подходит для длительной записи;
- `settings=profile` — больше событий, выше overhead, для коротких сессий;
- `dumponexit=true` — записать файл при выходе;
- `duration=60s` — ограничить длительность;
- `maxsize=512m`/`maxage=1h` — ограничить дисковый буфер.

На живом процессе:

```bash
jcmd <pid> JFR.start name=mc settings=profile duration=60s filename=mc.jfr
jcmd <pid> JFR.dump name=mc filename=mc-now.jfr
jcmd <pid> JFR.stop name=mc
```

Анализ:

```bash
jfr summary mc.jfr
jfr print --events jdk.ExecutionSample mc.jfr | head
jfr print --events jdk.ObjectAllocationInNewTLAB mc.jfr | head
```

Для GUI — JDK Mission Control. Для модов JFR хорош тем, что не зависит от лоадера и позволяет поймать редкий лаг по `dumponexit`/`duration`, но в отличие от spark не даёт «магической» кнопки «покажи, кто тормозит»: нужно читать события.

### async-profiler

[async-profiler] — нативный сэмплер, на котором, в частности, работает spark. Можно запускать отдельно, без мода:

```bash
asprof -d 60 -e cpu -f cpu.html <pid>
asprof -d 60 -e wall -t -f wall.html <pid>
asprof -d 60 -e alloc -f alloc.html <pid>
asprof -d 60 -e cpu --alloc 2m --lock 10ms -f combined.jfr <pid>
```

- `-e cpu` — CPU-время (perf_events на Linux; в контейнере может быть недоступно);
- `-e wall` — wall-clock, полезно для старта и заблокированных потоков;
- `-e alloc`/`--alloc` — аллокации;
- `-e itimer`/`-e ctimer` — fallback, если perf_events запрещён;
- `-o jfr`/`-f *.jfr` — открыть в JMC;
- `-t` — разбивка по потокам.

Ограничения: нужен доступ к процессу (тот же пользователь/контейнер), на Linux часто нужен `kernel.perf_event_paranoid` ≤ 2 или `CAP_PERFMON`; в Docker — `--cap-add SYS_ADMIN`/`perf_event_paranoid=-1` (в зависимости от окружения). Для Minecraft это обычно не проблема на десктопе, но проблема в CI.

### Что искать в моде

| Симптом | Где смотреть | На что похоже |
| --- | --- | --- |
| Лаги сервера, низкий TPS | `Server thread` → `MinecraftServer.tickServer` → `ServerLevel.tick` | Ваш `BlockEntity`/`Entity`/`tick`-handler в листьях дерева |
| Фризы клиента | `Render thread` → клиентский render-цикл (`Minecraft`/`GameRenderer`/`LevelRenderer`) | `LevelRenderer.renderLevel`, ваш рендер-хук, `GuiGraphics` |
| Провалы при открытии GUI | `Render thread` → `Screen.render`/`extractRenderState` | Перестройка списков, загрузка текстур, `String.format` |
| Постоянный GC | `/spark gc`, `jdk.GCPhasePause`, alloc-профиль | `new` в тике, streams, boxed-коллекции, `ItemStack.copy()` |
| Утечка памяти | `/spark heapdump`, `jdk.OldObjectSample` (JFR) | Статические `Map`/`List`, кэши без eviction, слушатели |
| Долгий старт | JFR `jdk.ClassLoad`, `mixin.debug.profiler` | Тяжёлые миксины, огромные datapack'и, генерация |

Типичные ошибки профилирования мода:

- профилировать пустой мир и делать вывод о тике;
- смотреть общий CPU, а не конкретный поток;
- включать `settings=profile` + `--thread *` + `--alloc` одновременно и удивляться overhead;
- забывать, что в dev-окружении JIT прогревается иначе, а миксины применяются при загрузке;
- мерить рендер в VSync/ограничении FPS и путать ожидание с работой.

---

## Автотесты

### GameTest: vanilla, Fabric, NeoForge

**Vanilla GameTest** — фреймворк Mojang: тест-функция получает `GameTestHelper`, структура-сцена загружается из `.nbt`, тест выполняется в реальном мире на серверной стороне. Запуск в игре — `/test runall`, `/test run <name>`, `/test runfailed`; структура создаётся/экспортируется через `/test` и structure block. Это лучший способ проверить «блок реально плавит предмет», «энтити реально спавнится», а не только что метод вернул `true`.

**Fabric GameTest API** (`fabric-gametest-api-v1`):

- тесты регистрируются через entrypoint `fabric-gametest` в `fabric.mod.json`;
- метод помечается `@GameTest`, принимает `GameTestHelper`, должен быть `public`, не `static`, возвращать `void`;
- Loom умеет создать отдельный source set:

  ```groovy
  fabricApi {
      configureTests {
          createSourceSet = true
          modId = "example-mod-test"
          enableGameTests = true
          enableClientGameTests = true
          eula = true
      }
  }
  ```

- серверные тесты гоняются вместе с `build`; клиентские — `./gradlew runClientGameTest`;
- клиентские тесты (`FabricClientGameTest`) умеют создавать мир, ждать чанки, нажимать клавиши и делать скриншоты — это единственный вменяемый способ автоматизировать GUI.

**NeoForge GameTest**:

- `./gradlew runGameTestServer` — отдельный серверный запуск, код возврата равен числу обязательных упавших тестов;
- в других run-конфигурациях включается `neoforge.enableGameTest=true`;
- в MDK клиентская конфигурация часто содержит `neoforge.enabledGameTestNamespaces=<mod_id>`, чтобы был доступен `/test`;
- тесты описываются через `TestData`/`TestInstance` (datapack) или через `@GameTest`-методы; для тестового фреймворка NeoForge есть `@TestHolder`, `DynamicTest`, `@EmptyTemplate` (см. `NEOGAMETESTS.md`).

> [!NOTE]
> GameTest в dev-окружении работает с named-мэппингами и часто «прощает» то, что в prod-jar'е сломается из-за refmap/namespace. Поэтому зелёный GameTest **не заменяет** prod-smoke (см. ниже).

### MC-Runtime-Test и HeadlessMc

**[MC-Runtime-Test]** — GitHub Action (`headlesshq/mc-runtime-test`), который запускает **настоящий клиент** Minecraft в CI, заходит в singleplayer-мир, выполняет `/test runall`, затем выходит с кодом успеха/провала. Он использует HeadlessMc и умеет ставить vanilla/Fabric/Forge/NeoForge, моды, Fabric API. Это ближе всего к «пользователь поставил jar и запустил».

```yaml
- name: Run MC test client
  uses: headlesshq/mc-runtime-test@4.4.0
  with:
    mc: 1.21.4
    modloader: fabric
    regex: .*fabric.*
    mc-runtime-test: fabric
    java: 21
```

Версия action в примере взята из официальной документации; перед использованием проверьте актуальный тег.

Полезные свойства:

```text
-DMcRuntimeGameTestMinExpectedGameTests=1
```

— упасть, если GameTest'ов не нашлось (защита от «ноль тестов = ноль провалов»). Для Forge/NeoForge discovery структуры может требовать дополнительных ухищрений; в документации проекта это прямо признано.

**[HeadlessMc]** — CLI-лаунчер Minecraft: ставит версии, лоадеры, моды, запускает клиент без GPU/дисплея. Именно его использует MC-Runtime-Test. Для headless-запуска клиента применяется `-lwjgl` (отключает LWJGL), для JVM-свойств — `--jvm "-Dfoo=bar"`. Оффлайн-аккаунты допустимы только для headless CI, обычная игра требует валидного аккаунта.

### Когда что

| Инструмент | Что проверяет | Стоимость | Когда брать |
| --- | --- | --- | --- |
| JUnit + Fabric Loader JUnit | Чистая логика, codec'и, хелперы | Секунды | Всегда, где можно оторвать код от игры |
| Vanilla GameTest | Блоки, энтити, мир, тик | Минуты | Логика, которой нужен реальный `ServerLevel` |
| Fabric client gametest | GUI, ввод, скриншоты | Минуты + дисплей/Xvfb | Рендер и экраны |
| NeoForge `runGameTestServer` | Серверная логика NeoForge | Минуты | NeoForge-мод |
| MC-Runtime-Test | Prod-jar в реальном клиенте, загрузка модпака | Минуты, CI | Перед релизом; ловит refmap/namespace-проблемы |
| HeadlessMc | Запуск/установка в CI | Секунды-минуты | Инфраструктура для MC-Runtime-Test |
| spark/JFR/async-profiler | Производительность | Секунды-минуты | Когда есть конкретный лаг, а не «на всякий случай» |

Минимальный разумный набор для мода с миксинами:

1. unit-тесты на всё, что не требует игры;
2. один GameTest на ключевое поведение;
3. prod-smoke через MC-Runtime-Test (или ручной запуск собранного jar в чистом инстансе);
4. профилирование — по симптомам, не в CI.

---

## Проверено на

- **2026-11-21**, Mixin `0.8.7` (SpongePowered, коммит `4053421aa10aaac6127d969028a29c94fe3054f6`): имена опций, тексты сообщений, пути `.mixin.out/class`, `.mixin.out/java`, `.mixin.out/audit`, поведение `dumpTargetOnFailure`, hotswap-агент.
- **2026-11-21**, Fabric Loader `main` (клон на дату проверки): `FabricMixinBootstrap` автоматически выставляет `mixin.env.remapRefMap=true` в dev, когда namespace мода и runtime различаются.
- **2026-11-21**, документация Fabric Loom (property-based API с 1.17, `useLegacyMixinAp` по умолчанию выключен с 1.12), Fabric Docs по GameTest и hotswap.
- **2026-11-21**, документация NeoForge: `runGameTestServer`, `neoforge.enableGameTest`, `systemProperty` в MDG.
- **2026-11-21**, spark docs (команды `profiler`, `heapsummary`, `heapdump`), OpenJDK/JFR docs, JBR README, async-profiler docs.
- Версии Java в примерах: 17/21/25. Для 26.x минимальная — Java 25.

## Источники

### Первоисточники

- [Mixin Java System Properties] — официальная таблица флагов.
- [SpongePowered/Mixin: MixinEnvironment.java, 0.8.7] — определения `Option`, `CompatibilityLevel`, значения по умолчанию.
- [SpongePowered/Mixin: ExtensionClassExporter.java, 0.8.7] — `.mixin.out/class`, `.mixin.out/java`, Fernflower.
- [SpongePowered/Mixin: MixinProcessor.java, 0.8.7] — `dumpTargetOnFailure`, `.target.class`.
- [SpongePowered/Mixin: MixinConfig.java, 0.8.7] — тексты про compatibility level, `minVersion`, refmap.
- [SpongePowered/Mixin: InjectionInfo.java, 0.8.7] — `Critical injection failure`, `Injection validation failed`.
- [SpongePowered/Mixin: MixinApplicatorStandard.java, 0.8.7] — `Mixing ...`, overwrite conflicts.
- [SpongePowered/Mixin: MixinPreProcessorStandard.java, 0.8.7] — `@Shadow ... was not located`.
- [SpongePowered/Mixin: RedirectInjector.java, 0.8.7] — `@Redirect conflict`.
- [SpongePowered/Mixin: MixinAgent.java, 0.8.7] — hotswap-сообщения.
- [Fabric Loader: FabricMixinBootstrap.java] — dev-настройка `mixin.env.remapRefMap`.
- [Fabric Docs: Loom Options] — run-конфигурации, `useLegacyMixinAp`.
- [Fabric Docs: Launching the Game in IntelliJ IDEA] — hotswap, JBR, `-javaagent`.
- [Fabric Docs: Automated Testing] — GameTest и client gametest.
- [NeoForged Docs: Game Tests] — `/test`, `runGameTestServer`, `neoforge.enableGameTest`.
- [ModDevGradle docs] — `systemProperty`, `jvmArgument` в `runs`.
- [spark docs: Command Usage] — команды profiler/heap.
- [OpenJDK JFR: configuring] — `-XX:StartFlightRecording`, `settings`, `dumponexit`.
- [async-profiler docs] — `asprof`, режимы cpu/wall/alloc.
- [JetBrains Runtime README] — enhanced class redefinition и флаг.
- [Fabric Wiki: Mixins and Obfuscation (DRAFT)] [draft] — refmap, remap, 26.1.
- [Fabric Blog: Fabric for Minecraft 26.1] — деобфускация и новый Loom.
- [HeadlessMC docs: CI/CD] — MC-Runtime-Test.
- [MC-Runtime-Test] — GitHub Action и `/test runall`.

### Community / вторичные

- [MixinTrace] [community] — `comp500/mixintrace`.
- [Not Enough Crashes] [community] — `natanfudge/Not-Enough-Crashes`.
- [Crash Assistant] [community] — `KostromDan/Crash-Assistant`.
- [MixinExtras wiki] — `@WrapOperation`, `@ModifyExpressionValue` (пригодится при разборе конфликтов).
- [MixinMCP] [community] — `muon-rw/MixinMCP`, поиск target'ов и чужих миксинов.
- [Linen: Mixins on 26.1] [community] — практические заметки по 26.x (проверять по первоисточникам).

<!-- Ссылки ниже вынесены в конец, чтобы таблицы оставались читаемыми. -->

[Mixin Java System Properties]: https://github.com/SpongePowered/Mixin/wiki/Mixin-Java-System-Properties
[SpongePowered/Mixin: MixinEnvironment.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/main/java/org/spongepowered/asm/mixin/MixinEnvironment.java
[SpongePowered/Mixin: ExtensionClassExporter.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/main/java/org/spongepowered/asm/mixin/transformer/ext/extensions/ExtensionClassExporter.java
[SpongePowered/Mixin: MixinProcessor.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/main/java/org/spongepowered/asm/mixin/transformer/MixinProcessor.java
[SpongePowered/Mixin: MixinConfig.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/main/java/org/spongepowered/asm/mixin/transformer/MixinConfig.java
[SpongePowered/Mixin: InjectionInfo.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/main/java/org/spongepowered/asm/mixin/injection/struct/InjectionInfo.java
[SpongePowered/Mixin: MixinApplicatorStandard.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/main/java/org/spongepowered/asm/mixin/transformer/MixinApplicatorStandard.java
[SpongePowered/Mixin: MixinPreProcessorStandard.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/main/java/org/spongepowered/asm/mixin/transformer/MixinPreProcessorStandard.java
[SpongePowered/Mixin: RedirectInjector.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/main/java/org/spongepowered/asm/mixin/injection/invoke/RedirectInjector.java
[SpongePowered/Mixin: MixinAgent.java, 0.8.7]: https://github.com/SpongePowered/Mixin/blob/4053421aa10aaac6127d969028a29c94fe3054f6/src/agent/java/org/spongepowered/tools/agent/MixinAgent.java
[Fabric Loader: FabricMixinBootstrap.java]: https://github.com/FabricMC/fabric-loader/blob/master/src/main/java/net/fabricmc/loader/impl/launch/FabricMixinBootstrap.java
[Fabric Docs: Loom Options]: https://docs.fabricmc.net/develop/loom/options
[Fabric Docs: Launching the Game in IntelliJ IDEA]: https://docs.fabricmc.net/develop/getting-started/intellij-idea/launching-the-game
[Fabric Docs: Automated Testing]: https://docs.fabricmc.net/develop/automatic-testing
[NeoForged Docs: Game Tests]: https://docs.neoforged.net/docs/misc/gametest/
[ModDevGradle docs]: https://docs.neoforged.net/toolchain/docs/plugins/mdg/
[spark docs: Command Usage]: https://spark.lucko.me/docs/Command-Usage
[OpenJDK JFR: configuring]: https://dev.java/learn/jvm/jfr/configure/
[async-profiler docs]: https://github.com/async-profiler/async-profiler/blob/master/docs/ProfilerOptions.md
[JetBrains Runtime README]: https://github.com/JetBrains/JetBrainsRuntime
[Fabric Wiki: Mixins and Obfuscation (DRAFT)]: https://wiki.fabricmc.net/drafts:mixin_obfuscation
[Fabric Blog: Fabric for Minecraft 26.1]: https://fabricmc.net/2026/03/14/261.html
[HeadlessMC docs: CI/CD]: https://headlesshq.github.io/headlessmc/dev-ci-cd/
[MC-Runtime-Test]: https://github.com/headlesshq/mc-runtime-test
[MixinTrace]: https://github.com/comp500/mixintrace
[Not Enough Crashes]: https://github.com/natanfudge/Not-Enough-Crashes
[Crash Assistant]: https://github.com/KostromDan/Crash-Assistant
[MixinExtras wiki]: https://github.com/LlamaLad7/MixinExtras/wiki
[MixinMCP]: https://github.com/muon-rw/MixinMCP
[Linen: Mixins on 26.1]: https://www.linenide.com/docs/porting/mixins
