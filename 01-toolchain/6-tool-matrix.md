# Сводная матрица инструментов

[← К оглавлению](../README.md)

> [!NOTE]
> **TL;DR.** Это карта инструментов, а не каталог всего на свете. Для каждой задачи здесь есть основной выбор, альтернатива и честная пометка «устарело/мертво». Начните с раздела [«Минимальный набор»](#минимальный-набор-под-задачу), если не хотите читать всю таблицу. Даты и версии — на 2026-11-21.

## Содержание

- [Как читать таблицу](#как-читать-таблицу)
- [Сборка](#сборка)
- [IDE и навигация](#ide-и-навигация)
- [Декомпиляция и байткод](#декомпиляция-и-байткод)
- [Отладка](#отладка)
- [Профилирование](#профилирование)
- [Mappings](#mappings)
- [Тесты](#тесты)
- [Ассеты](#ассеты)
- [Контент и миры](#контент-и-миры)
- [Скриптинг](#скриптинг)
- [Публикация](#публикация)
- [ИИ-ассистенты](#ии-ассистенты)
- [Что умерло или устарело](#что-умерло-или-устарело)
- [Минимальный набор под задачу](#минимальный-набор-под-задачу)
- [Как выбирать](#как-выбирать)
- [Лицензии и цена: оговорки](#лицензии-и-цена-оговорки)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Как читать таблицу

| Пометка | Значение |
| --- | --- |
| ✅ | Активно развивается, есть релизы за последние 12 месяцев |
| ⚠️ | Работает, но не обновляется или пригоден только для старых версий |
| ⛔ | Мёртв, архивирован или заменён; не начинайте новое на нём |
| `[unverified]` | Лицензию/цену не удалось подтвердить первоисточником |
| «см. репозиторий» | Лицензия есть, но точный SPDX не проверялся |

Колонка **«Когда брать»** — это не описание возможностей, а критерий выбора: если вы не попадаете в условие, инструмент вам, скорее всего, не нужен. Колонка **«Версии/лоадеры»** намеренно грубая: точные диапазоны смотрите в релизах.

---

## Сборка

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **Gradle** | сборка | Система сборки всей экосистемы: зависимости, таски, публикация | 9.2+ для Loom, 9.1+ для MDG; все лоадеры | Бесплатно, Apache-2.0 | Всегда; это фундамент, а не выбор | <https://gradle.org/> |
| **Fabric Loom** | сборка | Gradle-плагин Fabric: маппинги, ремап, run configs, jar-in-jar | Fabric/Quilt; `net.fabricmc.fabric-loom` для 26.1+, `-remap` для ≤1.21.11 | Бесплатно, MIT | Любой Fabric/Quilt-проект | <https://github.com/FabricMC/fabric-loom> |
| **ModDevGradle (MDG)** | сборка | Официальный плагин NeoForge; NeoForm, runs, datagen | NeoForge 1.20.2+; 2.x | Бесплатно, LGPL-2.1 | Новый NeoForge-проект; дефолт для 26.x | <https://github.com/neoforged/ModDevGradle> |
| **NeoGradle (NG)** | сборка | Альтернативный плагин NeoForge, ближе к FG-опыту | NeoForge; 7.x | Бесплатно, LGPL-2.1 | Если существующий билд уже на NG или нужен старый стиль | <https://github.com/neoforged/NeoGradle> |
| **ForgeGradle (FG)** | сборка | Исторический плагин Forge | Forge 1.20.1 и старше; для 1.21+ — только актуальная ветка FG, для NeoForge не использовать | Бесплатно, LGPL-2.1 | Только legacy-Forge; **не** для NeoForge | <https://github.com/MinecraftForge/ForgeGradle> |
| **RetroFuturaGradle (RFG)** | сборка | Плагин для legacy Forge с MCP mappings | Forge 1.7.10–1.12.2 | Бесплатно, LGPL-2.1 | GTNH-подобные проекты и старые сборки | <https://github.com/GTNewHorizons/RetroFuturaGradle> |
| **Unimined** | сборка | Единая среда для Fabric/Quilt/Forge/NeoForge и очень старых версий | 1.7.10+; мульти-лоадер | Бесплатно, LGPL-2.1 | Одна кодовая база на несколько версий/лоадеров | <https://github.com/unimined/Unimined> |
| **Architectury Loom** | сборка | Loom-форк для Architectury-проектов | Fabric + NeoForge/Forge | Бесплатно, см. репозиторий | Если используете Architectury API и общий код | <https://github.com/architectury/architectury-loom> |

**Практика.** Для нового проекта выбирайте ровно один плагин: Loom для Fabric, MDG для NeoForge. Не смешивайте FG и MDG в одном билде. `fabric-loom` без суффикса — legacy-ID; на 26.x используйте `net.fabricmc.fabric-loom`, иначе получите ремап там, где его не должно быть.

**Что ломается чаще всего.** Несовпадение Java и Gradle (Gradle 9 требует JDK 17+, но компилировать 26.x нужно на 25), забытый `settings.gradle` `pluginManagement`, и старый `gradle-wrapper.properties`. Последнее лечится `./gradlew wrapper --gradle-version 9.2`.

---

## IDE и навигация

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **IntelliJ IDEA** | IDE/навигация | Основная IDE для Java/Minecraft; Gradle-импорт, дебаггер, профилировщик | Все версии; Java 17/21/25 | Community — бесплатно (Apache-2.0); Ultimate — платная подписка | Почти всегда; Community достаточно для большинства задач | <https://www.jetbrains.com/idea/> |
| **Minecraft Development plugin** | IDE/навигация | Подсветка `mods.toml`/`fabric.mod.json`, генерация mixin/AT/AW, рефакторинги | NeoForge/Forge/Fabric; IDEA 2025.3+ для 26.1 | Бесплатный плагин; IDEA Ultimate — по подписке | Если пишете Java-мод в IDEA | <https://plugins.jetbrains.com/plugin/8327-minecraft-development> |
| **MixinMCP** | IDE/навигация | Расширяет MCP-сервер IntelliJ: поиск по всему classpath, байткод, mappings, поиск миксинов-конкурентов | IntelliJ 2026.1+, Gradle-плагин 1.2.x | Бесплатно, GPL-3.0 | Если работаете с ИИ-агентом в IDE или нужен поиск по чужим jar | <https://github.com/muon-rw/MixinMCP> |
| **jdx** | IDE/навигация | CLI/MCP для просмотра jar: классы, методы, mixin-конфиги, декомпиляция | Java 21+; любые jar | Бесплатно, Apache-2.0 | Быстро посмотреть чужой мод из терминала или дать агенту | <https://github.com/MohammadMD1383/jdx> |
| **Ravel** | IDE/навигация | Плагин IDEA для миграции исходников между mappings (Yarn → Mojmap) | Переход на 26.1+; Java и Kotlin | Бесплатно, MIT | Портирование Fabric-мода на 26.x | <https://github.com/badasintended/ravel> |

**Практика.** IDEA Community + Minecraft Development закрывают 90% задач. Ultimate нужен за Spring/HTTP-клиент/профилировщик, а не «за Minecraft». Parchment на 26.x теряет смысл: имена параметров уже приходят с игрой. Ravel, наоборот, нужен именно при переходе на 26.x — это инструмент миграции mappings, а не рантайм-зависимость.

**Не ставьте всё сразу.** Плагины, которые перехватывают одни и те же file types, иногда конфликтуют; если после установки Ravel перестала работать навигация по mixin-файлам, проверьте, не включена ли обработка `.java` двумя плагинами.

---

## Декомпиляция и байткод

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **Vineflower** | декомпиляция | Современный декомпилятор Java, форк FernFlower | Java 21+/25; все лоадеры | Бесплатно, Apache-2.0 | Дефолт: читать исходники Minecraft и модов | <https://github.com/Vineflower/vineflower> |
| **CFR** | декомпиляция | Альтернативный декомпилятор, силён на новом синтаксисе | Java 8+; все | Бесплатно, MIT | Когда Vineflower выдал нечитаемый код | <https://www.benf.org/other/cfr/> |
| **Recaf** | декомпиляция/байткод | GUI-редактор байткода: классы, методы, constant pool | Java 11+ (3.x); все | Бесплатно, MIT | Нужно править/смотреть байткод, а не только исходник | <https://github.com/Col-E/Recaf> |
| **Bytecode Viewer** | декомпиляция/байткод | Набор декомпиляторов и утилит в одном GUI | Java 8+; все | Бесплатно, GPL-3.0 | Сравнить вывод нескольких декомпиляторов | <https://github.com/Konloch/bytecode-viewer> |
| **javap** | декомпиляция/байткод | Дизассемблер JDK: сигнатуры, дескрипторы, opcodes | JDK 17/21/25 | Бесплатно, часть OpenJDK | Точные дескрипторы для `@At(target = ...)` | <https://docs.oracle.com/en/java/javase/25/docs/specs/man/javap.html> |
| **ASM** | декомпиляция/байткод | Библиотека чтения/записи байткода | Java 8+; все | Бесплатно, BSD-3-Clause | Свой инструмент/анализатор, не ручное чтение | <https://asm.ow2.io/> |
| **FernFlower** | декомпиляция | Старый декомпилятор JetBrains | ⛔ legacy | Бесплатно, Apache-2.0 | Не брать: заменён Vineflower | <https://github.com/JetBrains/fernflower> |

**Практика.** Порядок такой: Vineflower → CFR → `javap -c -p` для спорного метода. Если и это не помогло, смотрите байткод в Recaf: декомпилятор мог «съесть» synthetic-метод или lambda.

> [!WARNING]
> FernFlower больше не дефолт в Loom и не должен быть вашим первым выбором. Старые гайды, где `./gradlew genSources` даёт FernFlower, относятся к Loom ≤1.6.

**Про `ordinal`.** Если вы смотрите декомпилированный код, чтобы выбрать `ordinal`, вы уже на неправильном пути: порядок инструкций в декомпиляторе не гарантирован. Берите `@Expression` или якорь по вызову.

---

## Отладка

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **Mixin debug flags** | отладка | `-Dmixin.debug.export`, `verbose`, `countInjections`, `checks` | Все лоадеры с Mixin | Бесплатно, часть Mixin | Первое, что включать, если миксин не применился | <https://github.com/SpongePowered/Mixin/wiki/Mixin-Debugging> |
| **MixinTrace** | отладка | Добавляет миксины в стектрейсы и crash-report | Fabric/Forge/NeoForge; оригинал 2023, есть форки | Бесплатно, MIT | Краш непонятен, а в стеке только ванильные классы | <https://github.com/comp500/mixintrace> |
| **Crash Assistant** | отладка | GUI после краша, анализ логов, поиск виновника, auto-fix | 1.6.4–26.3; client-only | Бесплатно, custom license (MML 1.1.3) | Разбираете crash-report игрока/сборки | <https://github.com/KostromDan/Crash-Assistant> |
| **Hotswap / DCEVM / JBR** | отладка | Подмена классов в работающей JVM | Зависит от JDK/IDE | Бесплатно/по подписке IDE | Итерации по методу без перезапуска игры | <https://github.com/JetBrains/JetBrainsRuntime> |
| **`/debug` + IDE remote debug** | отладка | Точки останова в моде и ваниле | Все; нужен `-agentlib:jdwp` | Бесплатно | Логика, а не байткод | <https://docs.neoforged.net/> |

**Практика.** Сначала `mixin.debug.export=true` и `mixin.debug.verbose=true`, потом MixinTrace, потом дебаггер. Если краш в `<clinit>` чужого мода, Crash Assistant часто быстрее, чем чтение лога.

> [!TIP]
> `mixin.debug.countInjections=true` печатает, сколько раз применился каждый инъектор. Ноль — это ответ «миксин не нашёл цель», а не «Mixin сломался».

**Не путайте.** Hotswap меняет тело метода, но не добавляет поля/методы. Для структурных изменений нужен DCEVM/JBR с `-XX:+AllowEnhancedClassRedefinition`, и то не всегда.

---

## Профилирование

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **spark** | профилирование | CPU/memory/lag-spike профайлер для Minecraft | Fabric/Forge/NeoForge; 1.13+ | Бесплатно, GPL-3.0 | TPS просел, нужен flamegraph без выхода из игры | <https://spark.lucko.me/> |
| **JFR (Java Flight Recorder)** | профилирование | Встроенный профайлер JVM: CPU, alloc, locks | JDK 11+; все | Бесплатно, GPL-2.0+CE (OpenJDK) | Нет модов-профайлеров или нужен низкий overhead | <https://docs.oracle.com/javase/9/jfr/> |
| **async-profiler** | профилирование | Sampling CPU/alloc/lock с flamegraph | JDK 8+; Linux/macOS | Бесплатно, Apache-2.0 | Глубокий анализ вне игры, CI-бенчмарки | <https://github.com/async-profiler/async-profiler> |
| **VisualVM** | профилирование | GUI для heap/threads/CPU | JDK 8+ | Бесплатно, GPL-2.0+CE | Быстро посмотреть heap dump/потоки | <https://visualvm.github.io/> |

**Практика.** Для «лаг в мире» — spark (`/spark profiler start`, `/spark profiler stop`). Для «мод жрёт CPU» — JFR с `settings=profile`, затем откройте `.jfr` в JMC. Для «подозрение на аллокации» — async-profiler в alloc-режиме.

**Ограничение.** Профайлер показывает, *где* время, но не *почему*. Если 40% в `LevelChunk#getBlockState`, это может быть нормальный hot path, а не баг. Сравнивайте с ванилью на том же сейве.

---

## Mappings

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **Linkie** | mappings | Поиск по Yarn/Mojmap/MCP/Intermediary, зависимости, версии | 1.14–1.21.11; web + Discord | Бесплатно, open source (см. репозиторий) | Портируете мод или ищете имя класса между версиями | <https://linkie.shedaniel.dev/> |
| **mappings.dev** | mappings | Web-просмотрщик Mojmap/Yarn/Parchment | 1.14+ | Бесплатно, см. сайт | Сравнить имена в разных mappings | <https://mappings.dev/> |
| **Mojang official mappings** | mappings | Официальные имена Mojang; с 26.1 — имена рантайма | 1.14.4+; 26.x | Бесплатно, лицензия Mojang | Основа для NeoForge и 26.x | <https://piston-meta.mojang.com/mc/game/version_manifest_v2.json> |
| **Parchment** | mappings | Параметры и javadoc поверх Mojmap | 1.16.5–1.21.11; для 26.x опционален | Бесплатно, см. репозиторий | Нужны читаемые имена параметров до 26.1 | <https://parchmentmc.org/> |
| **MCPBot** | mappings | Исторический бот для MCP-имён | ⛔ мёртв | — | Не использовать | <https://github.com/MinecraftForge/MCPBot> |

**Практика.** На 26.x mappings как отдельный слой почти исчезли: игра деобфусцирована, имена в миксине — имена рантайма. Linkie всё ещё полезен для портов со старых версий и для поиска «как это называлось в 1.20.1».

> [!NOTE]
> `mappings.dev` и Linkie — справочники, а не зависимости. Не добавляйте их в `build.gradle`; это сайты.

---

## Тесты

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **GameTest** | тесты | Фреймворк Mojang для тестов игровой логики в мире | 1.17+; Fabric API/NeoForge | Бесплатно, часть Minecraft | Логика блоков/сущностей, которую нельзя проверить юнит-тестом | <https://docs.fabricmc.net/develop/automatic-testing> |
| **MC-Runtime-Test** | тесты | Запуск клиента в CI, smoke/GameTest-сценарии | Fabric/NeoForge/Forge; CI | Бесплатно, MIT | Нужен реальный клиент в GitHub Actions | <https://github.com/headlesshq/mc-runtime-test> |
| **HeadlessMc** | тесты | CLI-лаунчер Minecraft, клиент без GPU/дисплея | Все версии; CLI | Бесплатно, MIT | CI, серверные смоуки, автоматизация запуска | <https://github.com/headlesshq/headlessmc> |
| **JUnit 5** | тесты | Обычные unit-тесты Java | Любая | Бесплатно, EPL-2.0 | Чистая логика без реестров и мира | <https://junit.org/junit5/> |

**Практика.** Пирамида: JUnit для чистых функций → GameTest для мира → MC-Runtime-Test/HeadlessMc для «клиент вообще запускается». Не пытайтесь покрыть GameTest'ами весь мод: они медленные и хрупкие.

> [!WARNING]
> GameTest требует структур и мира; на 26.x API мог поменяться. Если тест не находится, проверьте namespace структуры и `@GameTestHolder`/`@PrefixGameTestTemplate`.

---

## Ассеты

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **Blockbench** | ассеты | Low-poly 3D-редактор, экспорт Java Block/Item JSON | 1.8–26.3; все лоадеры | Бесплатно, GPL-3.0 | Любая нестандартная модель блока/предмета | <https://www.blockbench.net/> |
| **Animated Java** | ассеты | Плагин Blockbench: риг и анимации в datapack | Java Edition; datapack | Бесплатно, open source (см. репозиторий) | Анимации через datapack, а не через Java-рендер | <https://github.com/Animated-Java/animated-java> |
| **GeckoLib Animation Utils** | ассеты | Плагин Blockbench: `.geo.json` + `.animation.json` | GeckoLib 4.x; NeoForge/Fabric | Бесплатно, см. репозиторий | Анимированные сущности/предметы с контролем из кода | <https://www.blockbench.net/plugins/geckolib> |
| **Misode** | ассеты | Генераторы JSON: рецепты, loot, модели, worldgen | 1.15–26.3 | Бесплатно, MIT | Быстро получить рабочий JSON и понять поля | <https://misode.github.io/> |
| **mcasset.cloud** | ассеты | Браузер ванильных ассетов по версиям | 1.14–26.3 | Бесплатно, см. сайт | Нужна ванильная текстура/модель/lang как референс | <https://mcasset.cloud/> |
| **Aseprite** | ассеты | Пиксель-арт и спрайты | Любая | Платная (≈$20), либо free-сборка | Рисуете текстуры с нуля | <https://www.aseprite.org/> |
| **Blockbench UV Editor** | ассеты | Встроенная развёртка | Входит в Blockbench | Бесплатно, GPL-3.0 | Точная UV-развёртка | <https://www.blockbench.net/> |

**Практика.** Blockbench — для геометрии, Aseprite — для текстур, Misode — для JSON. Не рисуйте `elements` руками: ошибки `uv`/`cullface` видны только в игре и стоят часа отладки.

**Граница.** Animated Java ≠ GeckoLib. Первый — datapack-анимации, второй — Java-библиотека. Если анимацией управляет код (состояния, атаки, интерполяция), берите GeckoLib; если команда/датапак — Animated Java.

---

## Контент и миры

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **NBT Studio** | контент/миры | Просмотр и редактирование NBT, SNBT, region-файлов | Формат NBT стабилен; последний push — 2024 | Лицензия не указана `[unverified]` | Смотрите block entity/entity/structure NBT | <https://github.com/tryashtar/nbt-studio> |
| **Amulet** | контент/миры | Редактор миров и Python-библиотека для save-форматов | Поддержка версий зависит от релиза | Бесплатно, open source (см. репозиторий) | Переносите/правите мир, а не код мода | <https://www.amuletmc.com/> |
| **WorldEdit** | контент/миры | Редактирование мира в игре, схемы, brushes | Fabric/Forge/NeoForge; 1.7.10–26.x | Бесплатно, GPL-3.0 (см. репозиторий) | Быстро построить тестовый полигон | <https://enginehub.org/worldedit> |
| **Axiom** | контент/миры | Строительство/моделирование в игре, blueprints | 1.20+; client-side | Бесплатно, closed-source `[unverified]` | Дизайн карт и презентаций, не разработка мода | <https://axiom.moulberry.com/> |
| **NBTExplorer** | контент/миры | Классический редактор NBT | ⚠️ старые версии | Бесплатно, см. репозиторий | Только legacy; для 26.x не рекомендуется | <https://github.com/jaquadro/NBTExplorer> |

**Практика.** Мододелу достаточно NBT Studio + структурного блока. Картостроителю — WorldEdit/Amulet, а Axiom — для скорости и красоты. Не тащите Axiom в зависимости мода: игроки не обязаны его ставить.

> [!CAUTION]
> Перед редактированием мира сделайте копию. Amulet/WorldEdit могут потерять data components и NBT block entity чужого мода; после правок проверяйте `/data get`.

---

## Скриптинг

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **KubeJS** | скриптинг | JavaScript для рецептов, тегов, loot, предметов, интеграций | 1.18–26.x (проверяйте конкретную ветку) | Бесплатно, LGPL-3.0 | Сборка/сервер, быстрые правки без Java | <https://github.com/kube-mods/kubejs> |
| **CraftTweaker** | скриптинг | ZenScript для рецептов, тегов, loot modifiers | 1.12–26.x | Бесплатно, MIT | Нужен стабильный DSL и документация | <https://github.com/CraftTweaker/CraftTweaker> |
| **GroovyScript** | скриптинг | Groovy-скрипты, рецепты, worldgen, интеграции | В основном 1.12.2/Cleanroom; современные версии ограниченно | Бесплатно, см. репозиторий | Legacy-сборки, где нужен полный Groovy | <https://github.com/CleanroomMC/GroovyScript> |

**Практика.** Скриптинг — для data-driven контента. Он не меняет AI, рендер и произвольные методы. Если задача звучит как «пусть зомби при ударе крадёт предмет», это Mixin, а не KubeJS.

| Задача | Инструмент |
| --- | --- |
| Рецепт/тег/loot/advancement | Datagen или KubeJS/CraftTweaker |
| Новый предмет с поведением | Java + datagen |
| Изменить поведение моба/блока | Mixin |
| GUI/рендер | Mixin/API лоадера |
| Совместимость с чужим модом | Mixin + version check |

---

## Публикация

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **Modrinth Minotaur** | публикация | Gradle-плагин загрузки на Modrinth | Gradle; все лоадеры | Бесплатно, LGPL-2.1 | Основная площадка — Modrinth | <https://github.com/modrinth/minotaur> |
| **mod-publish-plugin** | публикация | Публикация в Modrinth/CurseForge/GitHub/GitLab/Gitea/Discord | Gradle 8/9; все лоадеры | Бесплатно, MIT (2.2.1) | Несколько площадок из одного таска | <https://github.com/modmuss50/mod-publish-plugin> |
| **CurseGradle** | публикация | Старый плагин загрузки на CurseForge | ⛔ архивирован в 2021 | Бесплатно, MIT | Не брать для новых проектов | <https://github.com/matthewprenger/CurseGradle> |
| **GitHub Actions** | публикация | CI/CD: сборка, тесты, релизы, публикация | Любой репозиторий | Бесплатно для public; минуты для private | Любой проект, который выходит чаще раза в месяц | <https://github.com/features/actions> |
| **JitPack** | публикация | Сборка Maven-артефакта из GitHub-тега | GitHub; Java/Kotlin | Бесплатно для public; private — платно `[unverified]` | API-библиотека, а не игровой мод | <https://jitpack.io/> |
| **`maven-publish`** | публикация | Публикация в свой Maven/Nexus | Gradle; все | Бесплатно, Apache-2.0 (Gradle) | Внутренние зависимости и API | <https://docs.gradle.org/current/userguide/publishing_maven.html> |

**Практика.** Минимум: GitHub Actions + артефакт. Один релиз в месяц — Minotaur. Несколько площадок — `mod-publish-plugin`. CurseGradle не трогайте: он архивирован, и API CurseForge с тех пор менялся.

> [!TIP]
> `dryRun` в `mod-publish-plugin` и `debugMode` в Minotaur позволяют проверить, что именно уйдёт на площадку, не создавая релиз. Это дешевле, чем удалять ошибочную версию.

**Секреты.** `MODRINTH_TOKEN`, `CURSEFORGE_TOKEN`, `GITHUB_TOKEN`, `SIGNING_KEY`, `SIGNING_PASSWORD` — только через secrets. Токен в `build.gradle` попадёт в git и в лог.

---

## ИИ-ассистенты

| Инструмент | Категория | Что делает | Версии / лоадеры | Цена / лицензия | Когда брать | Ссылка |
| --- | --- | --- | --- | --- | --- | --- |
| **jdx (MCP)** | ИИ-ассистенты | Даёт агенту «IDE для jar»: outline, body, callers, diff, mixin-конфиги | Java 21+; любые jar | Бесплатно, Apache-2.0 | Агент читает чужой мод/библиотеку | <https://github.com/MohammadMD1383/jdx> |
| **MixinMCP** | ИИ-ассистенты | Расширяет MCP-сервер IntelliJ: поиск по classpath, байткод, mappings, миксины-конкуренты | IntelliJ 2026.1+, Gradle-плагин 1.2.x | Бесплатно, GPL-3.0 | Агент пишет миксины и ищет по чужим jar | <https://github.com/muon-rw/MixinMCP> |
| **LLM-клиенты (Copilot, Claude Code, Cursor, OpenCode)** | ИИ-ассистенты | Генерация кода, объяснение стектрейсов, рефакторинг | Любые | От бесплатных лимитов до подписки | Как ускоритель, не как источник истины | <https://opencode.ai/> |
| **Linkie MCP (community)** | ИИ-ассистенты | MCP-обёртка над mappings Linkie | 1.21.x и раньше | Бесплатно, см. репозиторий `[community]` | Агенту нужны mappings до 26.1 | <https://github.com/GRAINALCOHOL/linkie-mcp> |

**Практика.** ИИ полезен для «объясни этот стектрейс», «найди вызовы этого метода», «сгенерируй datagen-провайдер по образцу». Он плохо знает версии 26.x и склонен выдумывать API. Всегда проверяйте сигнатуры по исходникам и запускайте сборку.

> [!WARNING]
> Не давайте агенту право публиковать релиз или менять `gradle.properties` без ревью. MCP-сервер с токеном Modrinth — это доступ к вашему аккаунту, а не «ещё один инструмент».

---

## Что умерло или устарело

| Инструмент/практика | Статус | Чем заменить | Комментарий |
| --- | --- | --- | --- |
| **MCPBot** | ⛔ мёртв | Linkie / mappings.dev | Бот не обновляется; MCP-имена остались только в legacy |
| **FernFlower в Loom** | ⛔ legacy | Vineflower | Loom перешёл на Vineflower; старые гайды устарели |
| **CurseGradle** | ⛔ archived (2021) | `mod-publish-plugin` | Не используйте для 26.x |
| **MixinGradle** | ⛔ legacy | Встроено в Loom/MDG | Отдельный плагин больше не нужен |
| **ForgeGradle для NeoForge/новых версий** | ⛔ несовместимо | MDG / NeoGradle | NeoForge не собирается FG; для Forge 1.20.1 — можно |
| **`fabric-loom` (legacy ID)** | ⚠️ только backwards compat | `net.fabricmc.fabric-loom` / `-remap` | На 26.x старый ID может включить ремап |
| **NBTExplorer** | ⚠️ не обновляется | NBT Studio | Формат NBT стабилен, но GUI и фичи старые |
| **`runData` (NeoForge)** | ⚠️ историческое имя | `runClientData` / `runServerData` | В старых MDK — `runData`; в 26.x его нет |
| **Parchment для 26.x** | ⚠️ опционален | Встроенные имена | После деобфускации параметры уже есть |
| **Ravel для 26.x** | ⚠️ не нужен | — | Mappings больше не слой рантайма |

> [!NOTE]
> «Мёртв» не значит «не работает на старой версии». Если вы поддерживаете 1.12.2, RFG и NBTExplorer всё ещё могут быть рабочими. Пометка означает «не начинайте новое».

---

## Минимальный набор под задачу

| Задача | Обязательное | Полезное | Не нужно |
| --- | --- | --- | --- |
| Первый Fabric-мод | Gradle, Loom, IDEA, Vineflower | Minecraft Development, Linkie | NeoGradle, RFG, CurseGradle |
| Первый NeoForge-мод | Gradle, MDG, IDEA, Vineflower | Minecraft Development, MixinTrace | Loom, FG, MCPBot |
| Порт мода на 26.x | IDEA, Vineflower, Linkie, `mixin.debug.export` | jdx, CFR, Crash Assistant | Parchment, Ravel, MCPBot |
| Контент-мод (блоки/предметы) | Datagen, Blockbench, Misode | mcasset.cloud, Aseprite | spark, async-profiler |
| Оптимизация/перформанс | spark, JFR | async-profiler, VisualVM | Blockbench, KubeJS |
| Карта/мир | WorldEdit, NBT Studio | Amulet, Axiom | Loom, MDG |
| Сборка/сервер без Java | KubeJS или CraftTweaker | Misode, NBT Studio | Mixin, декомпиляторы |
| CI и релизы | GitHub Actions, `mod-publish-plugin` | Minotaur, JitPack | CurseGradle |

**Правило.** Если в вашем наборе больше двух декомпиляторов, вы, вероятно, отлаживаете не ту проблему. Один основной (Vineflower), один запасной (CFR) — достаточно.

---

## Как выбирать

1. **Сначала лоадер, потом плагин.** Fabric → Loom. NeoForge → MDG. Legacy Forge → RFG. Мульти-лоадер → Unimined/Architectury.
2. **Сначала данные, потом код.** Если задачу можно решить datagen'ом или KubeJS, не пишите миксин.
3. **Сначала встроенное, потом внешнее.** `mixin.debug.*` и JFR уже есть; не ставьте профайлер, пока не попробовали их.
4. **Сначала один инструмент на категорию.** Два профайлера, три декомпилятора и четыре паблишера — это не «надёжность», а рассинхрон.
5. **Проверяйте дату.** Инструмент, живой в 1.20.1, может быть мёртв в 26.3. Смотрите последний релиз, а не звёзды.

---

## Лицензии и цена: оговорки

- **«Бесплатно» ≠ «open source».** Axiom бесплатен, но closed-source; IDEA Community — open source, Ultimate — подписка.
- **Лицензия плагина ≠ лицензия IDE.** Minecraft Development бесплатен, но работает внутри IDEA, которая может быть платной.
- **GPL-3.0 у Blockbench/spark/WorldEdit** означает, что код можно использовать и модифицировать, но производные должны быть под GPL. Для мода, который просто *использует* Blockbench для экспорта модели, это не проблема: вы не линкуете GPL-код.
- **LGPL-2.1 у MDG/NeoGradle/Minotaur** позволяет использовать плагин в закрытом проекте; изменения самого плагина нужно шарить. Loom распространяется под MIT — это ещё мягче.
- **`[unverified]`** стоит там, где я не смог подтвердить лицензию первоисточником. Проверьте `LICENSE` в репозитории перед коммерческим использованием.
- **Цены меняются.** Aseprite, IDEA Ultimate и JitPack для private-репозиториев — проверяйте актуальный прайс, здесь указан порядок величины.

---

## Проверено на

| Компонент | Версия/состояние | Дата |
| --- | --- | --- |
| Minecraft Java | 26.3 (resource format 97.1, data format 121.0) | 2026-11-21 |
| NeoForge | 26.3.0.23-beta | 2026-11-21 |
| Fabric Loader / Loom | 26.3 / Loom 1.14+ | 2026-11-21 |
| Gradle | 9.2+ | 2026-11-21 |
| Java | 17 / 21 / 25 | 2026-11-21 |
| mod-publish-plugin | 2.2.1 | 2026-11-21 |
| Minotaur | 2.x | 2026-11-21 |
| Blockbench | актуальная на 2026-11 | 2026-11-21 |
| jdx | 1.7.0 | 2026-11-21 |

## Источники

**Сборка**

- Gradle: <https://gradle.org/>
- Fabric Loom (plugin IDs): <https://docs.fabricmc.net/develop/loom/>
- ModDevGradle: <https://github.com/neoforged/ModDevGradle>
- NeoGradle: <https://github.com/neoforged/NeoGradle>
- ForgeGradle: <https://github.com/MinecraftForge/ForgeGradle>
- RetroFuturaGradle: <https://github.com/GTNewHorizons/RetroFuturaGradle>
- Unimined: <https://github.com/unimined/Unimined>

**IDE, декомпиляция, отладка**

- IntelliJ IDEA: <https://www.jetbrains.com/idea/>
- Minecraft Development: <https://plugins.jetbrains.com/plugin/8327-minecraft-development>
- MixinMCP: <https://github.com/muon-rw/MixinMCP>
- jdx: <https://github.com/MohammadMD1383/jdx>
- Vineflower: <https://github.com/Vineflower/vineflower>
- CFR: <https://www.benf.org/other/cfr/>
- Recaf: <https://github.com/Col-E/Recaf>
- Bytecode Viewer: <https://github.com/Konloch/bytecode-viewer>
- Mixin debugging: <https://github.com/SpongePowered/Mixin/wiki/Mixin-Debugging>
- MixinTrace: <https://github.com/comp500/mixintrace>
- Crash Assistant: <https://github.com/KostromDan/Crash-Assistant>

**Профилирование, mappings, тесты**

- spark: <https://spark.lucko.me/>
- JFR: <https://docs.oracle.com/javase/9/jfr/>
- async-profiler: <https://github.com/async-profiler/async-profiler>
- Linkie: <https://linkie.shedaniel.dev/>
- mappings.dev: <https://mappings.dev/>
- Mojang version manifest (mappings внутри `version.json`): <https://piston-meta.mojang.com/mc/game/version_manifest_v2.json>
- Parchment: <https://parchmentmc.org/>
- GameTest (Fabric): <https://docs.fabricmc.net/develop/automatic-testing>
- MC-Runtime-Test: <https://github.com/headlesshq/mc-runtime-test>
- HeadlessMc: <https://github.com/headlesshq/headlessmc>

**Ассеты, контент, скриптинг**

- Blockbench: <https://www.blockbench.net/>
- Animated Java: <https://github.com/Animated-Java/animated-java>
- GeckoLib: <https://github.com/bernie-g/geckolib>
- Misode: <https://misode.github.io/>
- mcasset.cloud: <https://mcasset.cloud/>
- NBT Studio: <https://github.com/tryashtar/nbt-studio>
- Amulet: <https://www.amuletmc.com/>
- WorldEdit: <https://enginehub.org/worldedit>
- KubeJS: <https://github.com/kube-mods/kubejs>
- CraftTweaker: <https://github.com/CraftTweaker/CraftTweaker>
- GroovyScript: <https://github.com/CleanroomMC/GroovyScript>

**Публикация**

- Minotaur: <https://github.com/modrinth/minotaur>
- mod-publish-plugin: <https://github.com/modmuss50/mod-publish-plugin>
- CurseGradle (archived): <https://github.com/matthewprenger/CurseGradle>
- GitHub Actions: <https://docs.github.com/en/actions>
- JitPack: <https://docs.jitpack.io/>
