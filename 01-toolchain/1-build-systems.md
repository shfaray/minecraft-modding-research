# Системы сборки модов

[← К оглавлению](../README.md)

> [!NOTE]
> **TL;DR.** Сборка мода — это не «Gradle вообще», а конкретная связка **лоадер → Gradle-плагин → Gradle → Java → mappings**, и каждая эпоха имеет свою. Для 26.3 это `net.neoforged.moddev` 2.x + NeoForge 26.3.0.x + Gradle 9.2.1 + Java 25 (или `net.fabricmc.fabric-loom` 1.19 без ремапа), для 1.20.1 — ForgeGradle 6 + Java 17 + official/Parchment, для 1.7.10–1.12.2 — RetroFuturaGradle + Java 8. Главная ошибка — тащить плагин, Gradle или Java из одной эпохи в другую: большинство «загадочных» падений сборки объясняется именно этим.

## Содержание

- [Как читать матрицу](#как-читать-матрицу)
- [Сводная матрица](#сводная-матрица)
- [Legacy: 1.7.10–1.12.2](#legacy-17101122)
- [Переход: 1.13–1.16.5](#переход-1131165)
- [1.17–1.20.1: official mappings](#1171201-official-mappings)
- [NeoForge 26.x: ModDevGradle 2.x](#neoforge-26x-moddevgradle-2x)
- [NeoGradle 7.x: чем отличается](#neogradle-7x-чем-отличается)
- [NeoForm и NFRT](#neoform-и-nfrt)
- [Fabric 26.x](#fabric-26x)
- [Fabric до 1.21.11](#fabric-до-12111)
- [Forge 1.20.1](#forge-1201)
- [Legacy-сборка: RetroFuturaGradle](#legacy-сборка-retrofuturagradle)
- [Мульти-лоадер](#мульти-лоадер)
- [Типичные ошибки сборки](#типичные-ошибки-сборки)
- [Что выбрать](#что-выбрать)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Как читать матрицу

Столбцы матрицы — это не независимые настройки, а **жёстко связанная цепочка**:

```text
версия Minecraft
   └── лоадер (Forge / NeoForge / Fabric)
         └── Gradle-плагин и его версия
               └── версия Gradle
                     └── версия Java (toolchain)
                           └── mappings (MCP / Yarn / official / нет)
```

Если хотя бы одно звено из другой эпохи, сборка обычно падает **до** запуска игры:

- Java новее, чем понимает Gradle → `Unsupported class file major version`.
- Мод собран под Java 25, а игра/лоадер ждёт 21 → `UnsupportedClassVersionError`.
- Плагин 26.x на версии 1.21.11 → Loom не ремапит, NeoForge-плагин не находит артефакт.
- Mappings одной системы смешаны с другой → миксины и `@Shadow` не компилируются или не применяются.

Обозначения в матрице:

- **✅** — проверено по первоисточнику (доки, MDK, release notes) на 2026-11-21.
- **⚠️** — верно для указанной версии, но линейка допускает вариации; сверяйте `build.gradle` своего MDK.
- **`[unverified]`** — оценка/логический вывод, первоисточник не даёт точной цифры.

> [!WARNING]
> Не подставляйте версии «по памяти». Для NeoForge версия NeoForm (`26.3-1`), версия самого NeoForge (`26.3.0.113`) и версия плагина (`2.0.148`) — **три разных числа**. Для Fabric версия Loom (`1.19`), Loader (`0.20.1`) и Fabric API (`0.166.2+26.3`) тоже не связаны между собой.

## Сводная матрица

| Эпоха / MC | Лоадер | Gradle-плагин: ID и версия | Gradle | Java | Mappings | Примечания |
| --- | --- | --- | --- | --- | --- | --- |
| 1.7.10 | Forge | `net.minecraftforge.gradle:ForgeGradle:1.2` (buildscript) | ≤ 4.4.1 | 8 | MCP stable | Исторический FG 1.2; не поддерживается, asset-download сломан. Для работы берите RFG |
| 1.7.10–1.12.2 | Forge (GTNH) | `com.gtnewhorizons.retrofuturagradle` 2.0.x | 7.6–8.8 | 8 (toolchain) | MCP stable | ✅ Официально только 1.7.10 и 1.12.2; промежуточные 1.8–1.11 не поддержаны |
| 1.8–1.11.2 | Forge | ForgeGradle 2.1 / 2.2 / 2.3 | 2.x–4.x | 8 | MCP | ⚠️ Ветки FG_2.1/2.2/2.3 существуют; точное соответствие минорной версии — по MDK |
| 1.12.2 | Forge | ForgeGradle `2.3` или `3.+` (поздний MDK) | 4.9 | 8 | MCP snapshot `20171003-1.12` | ✅ MDK 14.23.5.2859 уже на FG 3.+ |
| 1.13.2–1.15.2 | Forge | ForgeGradle `3.+` | 4.9–4.10.3 | 8 | MCP snapshot | ✅ MDK 1.13.2/1.14.4/1.15.2 |
| 1.16.5 | Forge | ForgeGradle `5.1.+` | 7.3.3 | 8 | official `1.16.5` (MCP ещё доступен) | ✅ MDK 36.2.34; FG 4.x — ранние сборки, поздний MDK на 5.1 |
| 1.17.1 | Forge | ForgeGradle `5.1.+` | 7.2 | 16 | official `1.17.1` | ✅ MDK 37.1.1; именно 16, не 17 |
| 1.18.2–1.19.4 | Forge | ForgeGradle `5.1.+` | 7.5.1 | 17 | official | ✅ MDK 1.18.2/1.19.2 |
| 1.20.1 | Forge | `net.minecraftforge.gradle` `[6.0,6.2)` | 8.8 | 17 | official / Parchment | ✅ MDK 47.4.0; Parchment через Librarian |
| 1.20.2–1.20.4 | NeoForge | `net.neoforged.gradle.userdev` 7.x | 8.x | 17 | official | ⚠️ Исторически NeoGradle; MDG появился позже (репозиторий создан 2024-05-23) |
| 1.20.6–1.21.11 | NeoForge | `net.neoforged.moddev` 1.x/2.x или NG 7.x | 8.8–9.x | 21 | official (+ Parchment) | 1.21.11 — последняя обфусцированная |
| 26.x (26.3) | NeoForge | `net.neoforged.moddev` `2.0.148` | 9.1+ (MDK: 9.2.1) | 25 | не нужны (unobfuscated); Parchment опционален | ✅ MDK-26.3, NeoForge 26.3.0.113 |
| 26.x (26.3) | NeoForge (альтернатива) | `net.neoforged.gradle.userdev` `7.1.39` (линейка 7.1.21+) | 8.14 (NG_7.1) | 25 | не нужны | NG 7.x — другой DSL, ближе к FG-опыту |
| Fabric 1.14–1.21.11 | Fabric | `net.fabricmc.fabric-loom-remap` `1.19` (до 1.14 — `fabric-loom`) | 9.2+ (1.14), 9.4+ (1.16) | по версии игры: 8/17/21 | Yarn или Mojmap; intermediary в рантайме | ✅ Ремап обязателен |
| Fabric 26.x (26.3) | Fabric | `net.fabricmc.fabric-loom` `1.19` | 9.4+ (шаблон: 9.8.2) | 25 | не нужны (unobfuscated) | ✅ Шаблон `fabric-example-mod` ветка `26.3` |
| 1.7.10–1.12.2, 1.8–1.11 | любой legacy | `xyz.wagyourtail.unimined` `2.0.0` | 8.x+ | по версии игры | MCP/Yarn/что настроите | ✅ Мульти-лоадер и мульти-версия, вплоть до alpha |

> [!NOTE]
> «Java» в таблице — это **toolchain**, которым компилируется мод, а не Java, которой запущен Gradle. Это разные вещи: Gradle 9.2.1 сам требует Java 17+, но собирать мод под 1.12.2 можно toolchain'ом Java 8. Если перепутать, получите либо `UnsupportedClassVersionError`, либо «invalid source release».

> [!NOTE]
> Начиная с 1.20.2 таблица показывает **NeoForge**, потому что это основная площадка Forge-семейства и именно под неё развиваются MDG/NG. Сборки классического Forge для 1.20.2+ тоже существуют, но для нового проекта на этих версиях выбирайте NeoForge: документация, MDK и экосистема модов сместились туда.

---

## Legacy: 1.7.10–1.12.2

### ForgeGradle 1.x/2.x/3.x

Хронология плагина:

| Линейка | Ветка репозитория | Для чего | Ограничения |
| --- | --- | --- | --- |
| FG 1.0–1.2 | `FG_1.0`–`FG_1.2` | 1.7.10 и ранние | Gradle ≤ 4.4.1, Java 8; несовместим с Gradle 8/9 |
| FG 2.1–2.3 | `FG_2.1`–`FG_2.3` | 1.8–1.12.2 | Старый Gradle, MCP |
| FG 3.0 | `FG_3.0` | 1.13–1.15 и поздние MDK 1.12.2 | Gradle 4.9–4.10.3, Java 8 |

Legacy Modding Wiki прямо говорит: «Minecraft 1.7.10 is only supported by version 1.2», плагин больше не поддерживается, а загрузка ассетов не работает [21]. Поэтому в 2026 году собирать 1.7.10 на FG 1.2 — это археология, а не рабочий процесс.

Поздний официальный MDK для 1.12.2 (`14.23.5.2859`) уже использует **ForgeGradle 3.+** и Gradle 4.9, mappings — MCP snapshot `20171003-1.12` [25]. То есть «1.12.2 = FG 2.3» верно только для старых проектов; новые/обновлённые MDK ушли на FG 3.

Типичный legacy `build.gradle` (ForgeGradle 2.x/3.x, Groovy) выглядит так:

```groovy
buildscript {
    repositories {
        maven { url = 'https://maven.minecraftforge.net/' }
        mavenCentral()
    }
    dependencies {
        classpath group: 'net.minecraftforge.gradle', name: 'ForgeGradle', version: '3.+', changing: true
    }
}
apply plugin: 'net.minecraftforge.gradle'

minecraft {
    version = '1.12.2-14.23.5.2859'
    runDir = 'run'
    mappings channel: 'snapshot', version: '20171003-1.12'
}

dependencies {
    deobf 'some.mod:artifact:1.0:dev'
}

// FG 2.x: то же самое называлось deobfCompile.
// RFG: api(rfg.deobf("some.mod:artifact:1.0"))
```

Ключевое отличие от современных плагинов: **нет `plugins {}` с версией, нет `neoForge {}`/`loom {}`**, конфигурация живёт в `minecraft {}`, а зависимости модов деобфусцируются через `deobf` (FG 2.x — `deobfCompile`, RFG — `rfg.deobf`).

### RetroFuturaGradle

RetroFuturaGradle (RFG) — форк/переписывание ForgeGradle для legacy-эпохи. По README:

- цель — dev-окружение для **1.7.10**;
- поддержка Gradle **7.6–8.8**, Java toolchains, configuration cache, build cache;
- generic injection: `List<String>` вместо сырых `List`;
- deobfuscation transformer для зависимостей (`rfg.deobf`);
- задача запуска в обфусцированном окружении (`runObfClient`/`runObfServer`).

Документация RFG уточняет: поддерживаются только **1.7.10 и 1.12.2**; 1.8.9, 1.9.4, 1.10.2, 1.11.2 — нет (issue #53 закрыт как not planned). Для этих версий смотрите Unimined.

Минимальный `build.gradle.kts` (по официальному примеру RFG):

```kotlin
plugins {
    id("java-library")
    id("com.gtnewhorizons.retrofuturagradle") version "2.0.2"
}

group = "com.example.examplemod"
version = "1.0.0"

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(8))
        vendor.set(org.gradle.jvm.toolchain.JvmVendorSpec.AZUL)
    }
}

minecraft {
    mcVersion.set("1.7.10")
    username.set("Developer")
    injectedTags.put("VERSION", project.version)
    extraRunJvmArguments.add("-ea:${project.group}")
}

repositories {
    maven {
        name = "GTNH Maven"
        url = uri("https://nexus.gtnewhorizons.com/repository/public/")
    }
}

dependencies {
    // пример деобфускации внешней зависимости
    // api(rfg.deobf("curse.maven:ic2-242638:2353971"))
}
```

Для 1.12.2 меняются две строки:

```kotlin
minecraft {
    mcVersion.set("1.12.2")
    mcpMappingChannel.set("stable")
    mcpMappingVersion.set("39-1.12")
}
```

> [!WARNING]
> RFG — не «ForgeGradle, но новее». Это отдельный плагин со своим DSL (`minecraft {}`, `injectedTags`, `rfg.deobf`). Его нельзя смешивать с `net.neoforged.moddev`, `net.neoforged.gradle.userdev` или Loom в одном проекте. И он не умеет NeoForge/26.x — там другой пайплайн (NeoForm).

### Почему современные плагины не подходят к 1.7.10–1.12.2

1. **ModDevGradle** создан для NeoForge и NeoForm. В README MDG прямо сказано: поддерживаются NeoForge версии «later than 21.0.x»; для 1.7.10 там просто нет ни NeoForm-версии, ни загрузчика.
2. **NeoGradle** — тот же класс: userdev-плагин тянет `net.neoforged:neoforge`, которого для 1.12.2 не существует.
3. **Loom** — плагин Fabric. Он версионно-независим и может собрать старую ваниль, но не даст Forge-API, `deobf`/`deobfCompile`, FML-загрузчик и Forge-события.
4. **FG 1.2** не работает на Gradle 8/9: использует удалённые API и старый механизм `buildscript`. Даже если запустить, скачивание ассетов сломано.
5. **Java 25** не сможет загрузить мод, скомпилированный под Java 8-игру, если перепутать toolchain: классы 69-й версии JVM просто не прочитаются.

---

## Переход: 1.13–1.16.5

Это эпоха **ForgeGradle 3–5** и **MCP mappings**. Именно здесь появился `mappings channel: ...`, который позже пережил FG 6.

| Версия | Плагин | Gradle | Java | Mappings в MDK |
| --- | --- | --- | --- | --- |
| 1.13.2 | FG `3.+` | 4.9 | 8 | `snapshot 20180921-1.13` |
| 1.14.4 | FG `3.+` | 4.9 | 8 | `snapshot 20190719-1.14.3` |
| 1.15.2 | FG `3.+` | 4.10.3 | 8 | `snapshot 20200514-1.15.1` |
| 1.16.5 | FG `5.1.+` | 7.3.3 | 8 | `official 1.16.5` |

Все четыре строки проверены по официальным MDK, скачанным с `maven.minecraftforge.net` [25].

Важные следствия:

- **MCP — это не Mojmap.** `func_12345_a` — SRG-имена (промежуточные Forge, на них основан MCP); `m_12345_` — обфусцированные официальные имена Mojang до 26.1; читаемые имена — результат official mappings. В 1.13–1.15 по умолчанию был MCP snapshot, поэтому старые туториалы и миксины используют MCP-имена.
- **ForgeGradle 4** — короткая переходная линейка для ранних 1.16.5. Поздний MDK 1.16.5 уже на FG 5.1, поэтому в новом проекте берите 5.1, а не 4.x.
- **Java 8** — обязательна для 1.13–1.16.5. Java 17 toolchain даст `class file major version 61` в игре, которая ждёт 52.
- **Parchment** для 1.13–1.15 практически не применялся; в 1.16.5 официальные mappings уже есть, и Parchment можно наложить через Librarian (FG ≥ 5.1.20).

Типовая ошибка при порте с 1.12.2 на 1.16.5 — оставить `mcp`-имена в миксинах и `@Shadow`. После смены канала на `official` их нужно переписать; `refmap` при этом генерируется заново, и старый JSON только мешает.

---

## 1.17–1.20.1: official mappings

С 1.17 Forge перешёл на **official Mojang mappings**; MCP остался историей. Это же время — переход на Java 17 (1.17 — Java 16, 1.18+ — Java 17).

| Версия | Плагин | Gradle | Java | Mappings |
| --- | --- | --- | --- | --- |
| 1.17.1 | FG `5.1.+` | 7.2 | 16 | `official 1.17.1` |
| 1.18.2 | FG `5.1.+` | 7.5.1 | 17 | `official 1.18.2` |
| 1.19.2 | FG `5.1.+` | 7.5.1 | 17 | `official 1.19.2` |
| 1.20.1 | FG `[6.0,6.2)` | 8.8 | 17 | `official` или Parchment |

> [!NOTE]
> В `00-glossary.md` для диапазона 1.17–1.20.4 указана Java 17. Это верно для 1.18+, но **1.17.1** по официальному MDK 37.1.1 собирается под **Java 16** — именно 16, а не 17. Для 1.18.2+ — Java 17.

Parchment добавляет поверх official **имена параметров и javadoc**. Для Forge/Parchment нужен плагин **Librarian** (`org.parchmentmc.librarian.forgegradle`), потому что сам ForgeGradle не умеет канал `parchment` без провайдера. Требования Librarian: ForgeGradle ≥ 5.1.20, Gradle ≥ 7.1.1 [24].

```groovy
plugins {
    id 'net.minecraftforge.gradle' version '[6.0,6.2)'
    id 'org.parchmentmc.librarian.forgegradle' version '1.+'
}

minecraft {
    // Вариант 1: чистый official
    // mappings channel: 'official', version: '1.20.1'

    // Вариант 2: official + Parchment
    mappings channel: 'parchment', version: '2023.09.03-1.20.1'
}
```

`2023.09.03-1.20.1` — пример формата `YYYY.MM.DD-MCVersion`; актуальный выпуск смотрите на parchmentmc.org. После 26.1 Parchment **опционален**: игра уже содержит имена параметров, Parchment остаётся полезен ради javadoc и уточнений.

> [!NOTE]
> На 1.20.1 ещё живёт огромная часть модовой экосистемы. Если ваша цель — именно 1.20.1, не «модернизируйте» сборку до NeoForge 26.x без необходимости: это не тот же API, не те же mappings и не тот же рендер.

---

## NeoForge 26.x: ModDevGradle 2.x

### Полный минимальный проект

Ниже — рабочий минимум для **NeoForge 26.3.0.113 / MDG 2.0.148 / Java 25**, собранный по официальному MDK `MDK-26.3-ModDevGradle` [3] и документации MDG [1][2].

#### `settings.gradle.kts`

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
    }
}

plugins {
    // Автоматически скачивает нужный JDK для toolchain
    id("org.gradle.toolchains.foojay-resolver-convention") version "1.0.0"
}

rootProject.name = "examplemod"
```

#### `gradle.properties`

```properties
# Память Gradle: декомпиляция NeoForm любит память
org.gradle.jvmargs=-Xmx2G
org.gradle.daemon=true
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true

# Версия NeoForge (26.3.0.113 — актуальная на 2026-11-21)
neoForge.version=26.3.0.113

# Кэш NFRT: ускоряет повторные сборки
neoForge.neoFormRuntime.enableCache=true

# Mod properties
mod_id=examplemod
mod_name=Example Mod
mod_license=All Rights Reserved
mod_version=1.0.0
mod_group_id=com.example.examplemod
```

> [!NOTE]
> `neoForge.version` в `gradle.properties` — валидный способ: по документации MDG 2 установка `neoForge.version` (или `neoForge.neoFormVersion`) включает моддинг и фиксирует версию. Официальный MDK называет свойство `neo_version` и передаёт его в `neoForge { version = ... }` — оба варианта рабочие. Если свойство не задано ни там, ни там, MDG не включит моддинг и зависимости NeoForge будут недоступны.

#### `build.gradle.kts`

```kotlin
plugins {
    `java-library`
    id("net.neoforged.moddev") version "2.0.148"
}

group = "com.example.examplemod"
version = "1.0.0"

base {
    archivesName.set("examplemod")
}

// Mojang поставляет Java 25 в 26.3 — мод должен целиться туда же
java {
    toolchain.languageVersion.set(JavaLanguageVersion.of(25))
}

neoForge {
    // Явная передача версии; можно опустить, если neoForge.version задан в gradle.properties
    version = providers.gradleProperty("neoForge.version").get()

    // Проверять, что Access Transformer целится в существующие члены
    validateAccessTransformers.set(true)

    runs {
        create("client") {
            client()
            systemProperty("neoforge.enabledGameTestNamespaces", "examplemod")
        }

        create("server") {
            server()
            programArgument("--nogui")
        }

        create("data") {
            clientData()
            programArguments.addAll(
                listOf(
                    "--mod", "examplemod",
                    "--all",
                    "--output", file("src/generated/resources/").absolutePath,
                    "--existing", file("src/main/resources/").absolutePath
                )
            )
        }

        create("gameTestServer") {
            type.set("gameTestServer")
        }

        configureEach {
            systemProperty("forge.logging.markers", "REGISTRIES")
            logLevel.set(org.slf4j.event.Level.DEBUG)
        }
    }

    mods {
        create("examplemod") {
            sourceSet(sourceSets.main.get())
        }
    }
}

// Ресурсы, сгенерированные datagen
sourceSets["main"].resources.srcDir("src/generated/resources")
```

#### `gradle/wrapper/gradle-wrapper.properties`

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-9.2.1-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

Создать wrapper:

```bash
./gradlew wrapper --gradle-version 9.2.1
```

#### Команды

```bash
./gradlew runClient       # клиент
./gradlew runServer       # сервер (примите EULA в run/eula.txt)
./gradlew runData         # datagen
./gradlew runGameTestServer
./gradlew build           # build/libs/examplemod-1.0.0.jar
```

### Что здесь важно

- **`version` в `neoForge {}` — это версия NeoForge, а не Minecraft и не NeoForm.** `26.3.0.113` = NeoForge для MC 26.3, сборка 113.
- **`mods {}` обязателен для dev-запуска.** Он говорит MDG, какой source set считать модом; без него `runClient` запустится без вашего кода.
- **`runs` создаются через `create(...)` в Kotlin DSL.** Groovy-примеры вида `client { client() }` работают за счёт динамического DSL; в `.kts` надёжнее `create("client") { client() }`.
- **`loadedMods` вместо `mods`.** В MDG 2 набор модов для конкретного запуска называется `loadedMods` (`loadedMods = [mods.examplemod]`), чтобы не путаться с блоком `mods {}`.
- **`neoForge.ideSyncTask`.** Если datagen должен обновляться при синхронизации IDE:

```kotlin
neoForge {
    ideSyncTask(tasks.named("generateModMetadata"))
}
```

- **AT по умолчанию.** Файл `src/main/resources/META-INF/accesstransformer.cfg` подхватывается без настройки; `validateAccessTransformers` включает проверку целей.
- **Parchment опционален.** Для 26.x имена параметров уже есть; Parchment нужен только ради javadoc. Проверьте, что для вашей версии игры артефакт вообще опубликован: на 2026-11-21 таблица Parchment заканчивается 1.21.11 [24].

```properties
# Только если для вашей 26.x есть артефакт Parchment:
neoForge.parchment.minecraftVersion=<minecraft-версия>
neoForge.parchment.mappingsVersion=<mappings-версия>
```

### Java 25 и Gradle 9

26.3 требует Java 25 (официальный MDK: «Mojang ships Java 25 to end users in 26.3»). MDK 26.3 идёт с Gradle **9.2.1**; ориентир для MDG 2.x — Gradle 9.1+. Gradle 8.x с 26.x не используйте: MDK и плагины рассчитаны на 9.x, а старый Gradle не понимает class file 69.

> [!WARNING]
> `UnsupportedClassVersionError: class file version 69.0` означает, что мод собран Java 25, а запускается на Java 21/17. `Unsupported class file major version 69` в выводе Gradle означает обратное: **Gradle слишком старый**, чтобы прочитать классы Java 25. Это две разные ошибки с разными лечениями.

---

## NeoGradle 7.x: чем отличается

NeoGradle (NG) — второй официальный плагин NeoForge. Его userdev-плагин выглядит как «классический ForgeGradle»:

```groovy
plugins {
    id 'net.neoforged.gradle.userdev' version '7.1.39'
}

dependencies {
    implementation 'net.neoforged:neoforge:26.3.0.113'
}
```

Ключевые отличия от MDG 2.x:

| Аспект | ModDevGradle 2.x | NeoGradle 7.x |
| --- | --- | --- |
| ID плагина | `net.neoforged.moddev` | `net.neoforged.gradle.userdev` |
| Подключение NeoForge | `neoForge { version = "..." }` | `dependencies { implementation "net.neoforged:neoforge:..." }` |
| Конфигурация | `neoForge {}` | `minecraft {}`, `runs {}`, `subsystems {}` |
| Runs | `neoForge { runs { create("client") { client() } } }` | `runs { client { } }`; по умолчанию создаётся run на каждый тип |
| Parchment | `neoForge { parchment {} }`, ключи `neoForge.parchment.*` | `subsystems { parchment {} }`, ключи `neogradle.subsystems.parchment.*` |
| Vanilla-mode | `neoFormVersion = "26.3-1"` | отдельный `net.neoforged.gradle.neoform` |
| NFRT | `neoFormRuntime {}` на верхнем уровне | встроено в userdev, настройки через `neogradle.subsystems.*` |
| Бинарный режим (без декомпиляции) | `enable { disableRecompilation = true }`; авто при `CI=true` (MDG ≥ 2.0.136) | `neogradle.subsystems.decompiler.enabled=false` или `CI=true` |
| Gradle | 9.1+ (MDK 26.3: 9.2.1) | 8.14 в ветке NG_7.1 |
| Стиль | новый, «Gradle best practices», configuration cache | ближе к ForgeGradle, больше конвенций |

Когда брать NeoGradle:

- у вас уже есть проект на NG и нет причины переезжать;
- нужны его конвенции (авто-создание runs, `LocalRuntime`, `runTypes`);
- нужна тонкая настройка декомпилятора/рекомпилятора через `neogradle.subsystems.*`;
- вы собираете не только мод, но и инфраструктуру вокруг NeoForge.

Когда брать MDG:

- новый проект на 1.20.6+/26.x;
- нужен современный Kotlin DSL и минимум магии;
- нужен vanilla-mode для общего модуля мульти-лоадерного проекта;
- нужен `jarJar`, `unitTest`, `interfaceInjectionData`, `ideSyncTask` из коробки.

> [!NOTE]
> Оба плагина — официальные и поддерживаются NeoForged. «Лучше» зависит от проекта, а не от идеологии. Но смешивать MDG и NG в одном проекте нельзя: они оба претендуют на одни и те же конфигурации и задачи.

---

## NeoForm и NFRT

**NeoForm** — форк MCP-пайплайна, который готовит компилируемые исходники Minecraft (и патчи NeoForge) для конкретной версии. **NFRT (NeoForm Runtime)** — исполняющая среда этого пайплайна: она скачивает ваниль, применяет патчи, деобфусцирует/подготавливает имена, декомпилирует и рекомпилирует.

Упрощённо:

```text
Mojang jars
  → merge client+server
  → apply NeoForge patches
  → deobfuscate / official names (26.x: имена уже официальные)
  → decompile (Vineflower)
  → recompile
  → minecraft-patched-<version>.jar + sources
```

Версионирование NeoForm:

| Пример | Что это |
| --- | --- |
| `26.3-1` | релиз NeoForm для MC 26.3, ревизия 1 |
| `26.1-1` | релиз NeoForm для MC 26.1 |
| `26.4-snapshot-10-1` | снапшот 26.4 № 10, сборка 1 |
| `1.21.11-20251209.172050` | старая схема с датой |

Это проверено по `maven-metadata.xml` NeoForm [8]: там есть и `26.1-1`, и `26.3-1`, и снапшоты вида `26.4-snapshot-10-1`.

Соответствие версий:

- **NeoForge** `26.3.0.113` — то, что вы указываете в `neoForge { version = ... }`.
- **NeoForm** `26.3-1` — то, что вы указываете в `neoForge { neoFormVersion = "26.3-1" }` **только в vanilla-mode** (без NeoForge).
- **NFRT** — отдельная версия инструмента; пиньтся через `neoForge.neoFormRuntime.version`, но обычно не трогается.

Vanilla-mode полезен для общего модуля мульти-лоадерного проекта: код компилируется против Minecraft, но без NeoForge API.

```kotlin
neoForge {
    neoFormVersion = "26.3-1"

    runs {
        create("client") { client() }
        create("server") { server() }
    }
}
```

В vanilla-mode доступны только `client`, `server`, `data`; никаких NeoForge-событий и реестров там нет.

Оптимизации NFRT:

```properties
# Кэш результатов пайплайна
neoForge.neoFormRuntime.enableCache=true
# Подробный лог (диагностика)
neoForge.neoFormRuntime.verbose=true
# Память декомпилятору задаётся через heap самого Gradle:
# org.gradle.jvmargs=-Xmx4G
```

MDG 2 умеет **полностью пропускать декомпиляцию/рекомпиляцию**:

```kotlin
neoForge {
    enable {
        version = "26.3.0.113"
        disableRecompilation = true
    }
}
```

Это быстрее и лучше для CI, но в IDE не будет исходников Minecraft для навигации. С MDG 2.0.136 при `CI=true` этот режим включается автоматически (MDG 2.0.124+ — опция доступна) [2].

> [!WARNING]
> Не путайте `neoForge.version` и `neoForge.neoFormVersion` в одном блоке: это взаимоисключающие режимы. `version` включает NeoForge, `neoFormVersion` — vanilla-mode. MDG 2 позволяет включить моддинг только один раз на проект.

---

## Fabric 26.x

### Что изменилось

С 26.1 Minecraft не обфусцирован. Loom 1.14 разделил плагин на два ID [12][10]:

- `net.fabricmc.fabric-loom` — для **необфусцированных** версий (26.1+), **ничего не ремапит**;
- `net.fabricmc.fabric-loom-remap` — для обфусцированных (≤ 1.21.11);
- `fabric-loom` — legacy-ID только для обратной совместимости с обфусцированными версиями.

Для 26.x это означает:

- **нет зависимости `mappings`** (Yarn/intermediary не нужны);
- **`implementation` вместо `modImplementation`** для Fabric Loader и Fabric API;
- **нет `remapJar`**: итоговый `jar` и есть релизный;
- `include` складывается в `jar`, а не в `remapJar`;
- Access Widener остаётся, но файл теперь называется **class tweaker** (`.classtweaker`).

### Минимальный `build.gradle`

По шаблону `FabricMC/fabric-example-mod`, ветка `26.3` [15].

`settings.gradle`:

```groovy
pluginManagement {
    repositories {
        maven {
            name = 'Fabric'
            url = 'https://maven.fabricmc.net/'
        }
        mavenCentral()
        gradlePluginPortal()
    }
}

rootProject.name = 'modid'
```

`build.gradle`:

```groovy
plugins {
    id 'net.fabricmc.fabric-loom' version "${loom_version}"
    id 'maven-publish'
}

repositories {
    // Loom сам добавляет Fabric, Mojang и Maven Central
}

dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    implementation "net.fabricmc:fabric-loader:${project.loader_version}"
    implementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_api_version}"
}

loom {
    splitEnvironmentSourceSets()

    mods {
        "modid" {
            sourceSet sourceSets.main
            sourceSet sourceSets.client
        }
    }
}

processResources {
    def version = project.version
    inputs.property "version", version
    filesMatching("fabric.mod.json") {
        expand "version": version
    }
}

tasks.withType(JavaCompile).configureEach {
    it.options.release = 25
}

java {
    withSourcesJar()
    sourceCompatibility = JavaVersion.VERSION_25
    targetCompatibility = JavaVersion.VERSION_25
}
```

`gradle.properties`:

```properties
org.gradle.jvmargs=-Xmx1G
org.gradle.parallel=true
org.gradle.configuration-cache=true

minecraft_version=26.3
loader_version=0.20.1
loom_version=1.19

fabric_api_version=0.166.2+26.3

version=1.0.0
group=com.example
```

> [!NOTE]
> Шаблон на 2026-11-21 пиннит `loom_version=1.19-SNAPSHOT`; стабильный тег `1.19` опубликован 2026-10-23. Для воспроизводимых сборок лучше фиксировать релиз, для отслеживания исправлений — SNAPSHOT.

### Access Wideners / class tweakers

Файл лежит в `src/main/resources`, обычно `<modid>.classtweaker`, и начинается с заголовка:

```text
classTweaker  v1  official
```

Пример записей:

```text
# сделать класс публичным
accessible    class    net/minecraft/client/gui/screens/Screen

# сделать метод доступным
accessible    method   net/minecraft/world/entity/player/Player   getInventory   ()Lnet/minecraft/world/entity/player/Inventory;

# разрешить менять final-поле
mutable       field    net/minecraft/world/level/Level   random   Lnet/minecraft/util/RandomSource;
```

Подключение:

```groovy
loom {
    accessWidenerPath = file("src/main/resources/example-mod.classtweaker")
}
```

```json
{
  "accessWidener": "example-mod.classtweaker"
}
```

Требования: Fabric Loader **0.18.0+**, Loom **1.12+** [16]. Имя свойства `accessWidenerPath` сохранено ради обратной совместимости, хотя файл теперь «class tweaker». Проверка целей:

```bash
./gradlew validateAccessWidener
```

> [!WARNING]
> Class tweakers работают только с **ванильными** классами Minecraft, не с классами других модов. Если нужен доступ к чужому моду, используйте миксин/accessor, а не AW.

### Чем 26.x отличается от `fabric-loom-remap`

| Аспект | `fabric-loom` (26.x) | `fabric-loom-remap` (≤1.21.11) |
| --- | --- | --- |
| Обфускация | нет | есть |
| `mappings` | не нужны | Yarn или `loom.officialMojangMappings()` |
| Зависимости модов | `implementation` | `modImplementation` / `modApi` |
| Ремап | нет | `remapJar`, `remapSourcesJar` |
| `include` | попадает в `jar` | попадает в `remapJar` |
| Refmap | не нужен | не нужен с Loom 1.12+, но возможен через `useLegacyMixinAp` |
| Mixin remap | имена уже рантаймные | TinyRemapper ремапит аннотации |
| Java | 25 | по версии игры (8/17/21) |

---

## Fabric до 1.21.11

### Плагин и mappings

```groovy
plugins {
    id 'net.fabricmc.fabric-loom-remap' version "${loom_version}"
}

dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"

    // Вариант A: Yarn
    mappings "net.fabricmc:yarn:${project.yarn_mappings}:v2"

    // Вариант B: official Mojang mappings
    // mappings loom.officialMojangMappings()

    modImplementation "net.fabricmc:fabric-loader:${project.loader_version}"
    modImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_api_version}"
}
```

Для 1.21.11 на 2026-11-21 актуальны Yarn `1.21.11+build.6` и Loader `0.20.1` (Fabric Meta API). Номер Yarn build обязательно сверяйте на fabricmc.net/develop: он меняется независимо от версии игры.

Yarn vs Mojmap:

| Критерий | Yarn | official Mojmap |
| --- | --- | --- |
| Имена | читаемые, «fabric-style» | официальные Mojang |
| Параметры | есть | нет (нужен Parchment) |
| Совместимость с туториалами | высокая для Fabric | высокая для Forge/NeoForge |
| Будущее | Yarn не поддерживается с 26.1 | official — единственные имена 26.x |
| Миксины | имена Yarn в аннотациях | имена Mojmap в аннотациях |

С 26.1 Fabric отказалась от поддержки сторонних mappings: Yarn остаётся только для старых версий. Для миграции есть `migrateMappings` (Loom) и плагин Ravel для IntelliJ; оба не идеальны и требуют ручной правки миксинов [17].

### Refmap и Mixin AP

Исторически Loom добавлял `net.fabricmc:fabric-mixin-compile-extensions` в `annotationProcessor`, и AP генерировал `*.refmap.json`. Refmap переводил «мягкие» строки в аннотациях (`method = "foo"`, `@At(target = "...")`) в обфусцированные имена.

С **Loom 1.12+** по умолчанию `useLegacyMixinAp = false`: вместо AP работает TinyRemapper, который ремапит сами миксины. Refmap больше не обязателен.

```groovy
loom {
    mixin {
        // true — вернуть старый AP и refmap (нужно для старых тулчейнов/модов)
        useLegacyMixinAp = true
        // имя refmap по умолчанию
        defaultRefmapName = "example.refmap.json"
    }
}
```

Практические правила:

- Новый проект на 1.21.x — **не включайте** `useLegacyMixinAp`. Меньше файлов, меньше рассинхрона.
- Порт старого мода с готовым refmap — либо удалите refmap и проверьте, что TinyRemapper справляется, либо включите AP и оставьте refmap.
- Если миксин «не применяется» и в логе есть `InvalidMixinException`/`refmap not found`, сначала проверьте, не смешаны ли refmap от Yarn и Mojmap.
- На 26.x refmap не нужен вообще: имена в миксине — это имена в рантайме.

Loom 1.15 добавил `loom.defaultMixinRemapType`, чтобы задать тип ремапа миксинов для необфусцированных версий; Loom 1.15 также **удалил FernFlower** (остались CFR/Vineflower) [13].

---

## Forge 1.20.1

Это «последняя большая» версия классического Forge. Сборка — ForgeGradle 6, Java 17, Gradle 8.8.

### `settings.gradle`

```groovy
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven {
            name = 'MinecraftForge'
            url = 'https://maven.minecraftforge.net/'
        }
    }
}

plugins {
    id 'org.gradle.toolchains.foojay-resolver-convention' version '0.7.0'
}
```

### `build.gradle`

```groovy
plugins {
    id 'eclipse'
    id 'idea'
    id 'maven-publish'
    id 'net.minecraftforge.gradle' version '[6.0,6.2)'
    // Для Parchment добавьте строкой ниже:
    // id 'org.parchmentmc.librarian.forgegradle' version '1.+'
}

version = mod_version
group = mod_group_id

base {
    archivesName = mod_id
}

// Mojang ships Java 17 to end users in 1.18+
java.toolchain.languageVersion = JavaLanguageVersion.of(17)

minecraft {
    mappings channel: mapping_channel, version: mapping_version
    // Или Parchment:
    // mappings channel: 'parchment', version: '2023.09.03-1.20.1'

    copyIdeResources = true

    runs {
        configureEach {
            workingDirectory project.file('run')
            property 'forge.logging.markers', 'REGISTRIES'
            property 'forge.logging.console.level', 'debug'

            mods {
                "${mod_id}" {
                    source sourceSets.main
                }
            }
        }

        client {
            property 'forge.enabledGameTestNamespaces', mod_id
        }

        server {
            property 'forge.enabledGameTestNamespaces', mod_id
            args '--nogui'
        }

        data {
            workingDirectory project.file('run-data')
            args '--mod', mod_id, '--all',
                 '--output', file('src/generated/resources/'),
                 '--existing', file('src/main/resources/')
        }
    }
}

repositories {
    maven { url = 'https://maven.parchmentmc.org' } // если используете Parchment
}

dependencies {
    minecraft "net.minecraftforge:forge:${minecraft_version}-${forge_version}"
}

tasks.withType(JavaCompile).configureEach {
    options.encoding = 'UTF-8'
}
```

`gradle.properties` (минимум):

```properties
org.gradle.jvmargs=-Xmx2G
org.gradle.parallel=true

minecraft_version=1.20.1
forge_version=47.4.0
mapping_channel=official
mapping_version=1.20.1

mod_id=examplemod
mod_name=Example Mod
mod_version=1.0.0
mod_group_id=com.example.examplemod
```

Проверено по официальному MDK `1.20.1-47.4.0` [25]: FG `[6.0,6.2)`, Gradle 8.8, Java 17, `mappings channel: mapping_channel, version: mapping_version`.

### Parchment

Порядок:

1. Добавить `maven.parchmentmc.org` в `pluginManagement.repositories` **и** в `repositories`.
2. Применить `id 'org.parchmentmc.librarian.forgegradle' version '1.+'` **после** ForgeGradle.
3. Поставить `mappings channel: 'parchment'`.

Parchment не заменяет official, а накладывается на него: имена классов/методов остаются Mojang, добавляются параметры и javadoc. Для 1.20.1 это до сих пор полезно, потому что official-имена параметров там отсутствуют.

> [!NOTE]
> `copyIdeResources = true` в `minecraft {}` — не косметика. Без него IntelliJ может запускать игру с устаревшими `resources`, и вы будете отлаживать «миксин не видит json», которого нет в classpath.

---

## Legacy-сборка: RetroFuturaGradle

RFG — практический стандарт для 1.7.10/1.12.2 в 2026. Почему:

- работает на Gradle 7.6–8.8, а не на Gradle 4.x;
- Java toolchain: можно запускать Gradle на Java 17/21, а компилировать мод под Java 8;
- configuration cache/build cache;
- deobf-трансформер зависимостей;
- generic injection (нормальные дженерики в API);
- задачи `runObfClient`/`runObfServer` для проверки в обфусцированном окружении.

Ограничения:

- официально только **1.7.10** и **1.12.2**;
- 1.8–1.11 — не поддержаны (используйте Unimined);
- DSL несовместим с FG/MDG/NG;
- для миксинов нужен внешний загрузчик (MixinBooter/Unimix) `[community]` или ручной `extraTweakClasses`.

Минимум для 1.12.2:

```kotlin
plugins {
    id("java-library")
    id("com.gtnewhorizons.retrofuturagradle") version "2.0.2"
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(8))
        vendor.set(org.gradle.jvm.toolchain.JvmVendorSpec.AZUL)
    }
}

minecraft {
    mcVersion.set("1.12.2")
    username.set("Developer")
    mcpMappingChannel.set("stable")
    mcpMappingVersion.set("39-1.12")
}

repositories {
    maven { url = uri("https://nexus.gtnewhorizons.com/repository/public/") }
}

dependencies {
    // deobf-зависимости
    // api(rfg.deobf("curse.maven:jei-238222:..."))
}
```

> [!WARNING]
> Не пытайтесь подключить RFG к 1.16.5/1.20.1 «потому что он современнее». RFG — про MCP-эпоху; для 1.16.5 есть FG 5.1, для 1.20.1 — FG 6, для 26.x — MDG/NG. Смешение даст либо отсутствие артефактов, либо конфликт mappings.

---

## Мульти-лоадер

Есть три подхода, и они не взаимозаменяемы.

### Architectury

Абстракция над лоадерами: общий модуль + тонкие платформенные модули (`fabric`, `neoforge`/`forge`). Использует **Architectury Loom** (`dev.architectury.loom`) — форк Fabric Loom, умеющий Forge/NeoForge.

- Документация: [docs.architectury.dev](https://docs.architectury.dev/loom/introduction).
- Генератор шаблонов: [generate.architectury.dev](https://generate.architectury.dev/).
- Плагины: `architectury-plugin` + `dev.architectury.loom` (plugin marker `1.17.493`).
- Плюс: один общий код, минимум `if (loader == ...)`.
- Минус: Architectury API — дополнительная зависимость и слой абстракции; IDE-поддержка ограничена IntelliJ/VSCode; проект помечен как experimental.
- Gradle: 8.1+ (по докам Architectury Loom).

Когда брать: мод с общей игровой логикой, которому нужны Fabric и NeoForge, и вы готовы зависеть от Architectury API.

### MultiLoader (source sets)

Шаблон [jaredlll08/MultiLoader-Template](https://github.com/jaredlll08/MultiLoader-Template): несколько subproject'ов, каждый со своим плагином.

```text
root
├── common   ← общий код, компилируется против ванили/NeoForm
├── fabric   ← Loom + Fabric API
└── neoforge ← MDG/NG + NeoForge
```

- `common` не знает про лоадеры; платформенные модули зависят от него.
- Каждый лоадер собирается **родным** плагином, без Architectury.
- Больше контроля и меньше магии, но больше ручной работы: события, реестры, networking нужно оборачивать самому.
- Для `common` удобен vanilla-mode MDG (`neoFormVersion = "26.3-1"`) или NeoGradle vanilla.

Когда брать: нужен жёсткий контроль над зависимостями и вы не хотите Architectury API.

### Unimined

`xyz.wagyourtail.unimined` 2.0.0 — плагин для мульти-лоадерных и мульти-версионных проектов, включая очень старые версии.

Поддерживаемые лоадеры (из README): Fabric, Quilt, Legacy Fabric, Babric, Ornithe, Forge, NeoForge, Cleanroom, Rift, LiteLoader, CraftBukkit/Spigot/Paper, Risugami's ModLoader, JarModding.

- Ставится как `id("xyz.wagyourtail.unimined") version "2.0.0"`.
- Требует **4 ГБ heap** у Gradle daemon (`org.gradle.jvmargs=-Xmx4g`), потому что ремап идёт в демоне.
- 2.0 — модульный rewrite (prerelease): `unimined-api`, `unimined-remap`, `unimined-minecraft`, `unimined-transform`, `unimined-runs`, `unimined-source`.
- Шаблоны: [unimined/Unimined-Templates](https://github.com/unimined/Unimined-Templates), CLI `unimined new <name>`.

Когда брать: 1.8–1.11, очень старые версии, Total Conversion, нестандартные лоадеры, одна кодовая база на много версий.

| Подход | Общий код | Лоадеры | Сложность | Когда |
| --- | --- | --- | --- | --- |
| Architectury | да, через API | Fabric + NeoForge/Forge | средняя | быстрый мульти-лоадер |
| MultiLoader | да, вручную | любые | высокая | контроль без API-слоя |
| Unimined | да, через source sets | почти всё, включая legacy | высокая | старые/экзотические версии |

---

## Типичные ошибки сборки

| Симптом | Причина | Что делать |
| --- | --- | --- |
| `UnsupportedClassVersionError: class file version 69.0` | Мод собран Java 25, запущен на Java 21/17 | Выровнять toolchain и Java рантайма; для 26.x — Java 25 |
| `Unsupported class file major version 69` в выводе Gradle | Gradle слишком старый для Java 25 | Обновить wrapper до 9.x (MDK 26.3: 9.2.1) |
| `invalid source release: 25` | `options.release`/toolchain новее, чем JDK Gradle | Указать toolchain 25 и foojay-resolver; либо `--release` под фактический JDK |
| `Plugin [id: 'fabric-loom'] was not found` | Нет `maven.fabricmc.net` в `pluginManagement` | Добавить Fabric maven в `settings.gradle(.kts)` |
| `Plugin [id: 'net.neoforged.moddev'] was not found` | Опечатка в ID/версии или закрытый репозиторий | Сверить ID с docs.neoforged.net; MDG есть на Gradle Plugin Portal |
| `Could not find net.neoforged:neoforge:...` | Версия NeoForge не соответствует MC | Сверить `26.3.0.x` с 26.3; для NeoForm — `26.3-1` |
| `No matching variant ...` / `Could not resolve net.minecraft:...` | Смешаны plugin ID и эпоха (например, remap-Loom на 26.x) | Для 26.x — `net.fabricmc.fabric-loom`; для ≤1.21.11 — `fabric-loom-remap` |
| Миксин не применяется, `refmap not found` | Включён/выключен `useLegacyMixinAp`, старый refmap | Удалить refmap (Loom 1.12+), либо включить AP и пересобрать |
| `Mixin apply failed ... target method not found` | Mappings не совпадают (Yarn vs Mojmap vs MCP) | Привести весь проект к одним mappings; переписать аннотации |
| `OutOfMemoryError` при `genSources`/декомпиляции | Мало heap у Gradle/NFRT | `org.gradle.jvmargs=-Xmx4G`, кэш NFRT, `disableRecompilation` в CI |
| `runClient` запускается без мода | Нет `mods {}` / неверный source set | Добавить `mods { create("modid") { sourceSet(...) } }` |
| `UnsupportedClassVersionError` только в тестах | `test` source set не получил toolchain | Настроить `java { toolchain { ... } }` глобально, а не только `compileJava` |
| `Could not find method ...` в Kotlin DSL | Groovy-синтаксис скопирован в `.kts` | Использовать `create(...)`, `set(...)`, `providers.gradleProperty(...)` |
| `Configuration cache` падает на кастомной задаче | Задача читает `project` во время выполнения | Использовать `Provider`/`inputs`/`outputs`, либо временно отключить cache |

Диагностика в три команды:

```bash
./gradlew --version
java -version
./gradlew build --stacktrace --info
```

Если непонятно, кто виноват, — сравните `./gradlew --version` с таблицей эпох. В 90% случаев «сломалось после обновления» — это несовпадение Gradle/Java/toolchain, а не баг мода.

> [!WARNING]
> `./gradlew build --refresh-dependencies` помогает при битом кэше Loom (`caches/fabric-loom`, `.gradle/loom-cache`), но **не** лечит несовпадение версий. Не начинайте с него: сначала проверьте версии, потом чистите кэш.

---

## Что выбрать

| Сценарий | Стек | Почему |
| --- | --- | --- |
| Новый мод на 26.3, NeoForge | NeoForge 26.3.0.113 + MDG 2.0.148 + Gradle 9.2.1 + Java 25 | Официальный MDK, современный DSL, configuration cache |
| Новый мод на 26.3, Fabric | `net.fabricmc.fabric-loom` 1.19 + Loader 0.20.1 + Java 25 | Нет ремапа, быстрее сборка, нет mappings |
| Порт мода с 1.20.1 на 26.3 | NeoForge 26.3 + MDG 2.x; mappings не нужны | Официальные имена в рантайме; переписать миксины/рендер, а не только импорты |
| Поддержка мода на 1.20.1 | ForgeGradle `[6.0,6.2)` + Java 17 + official/Parchment | Стабильная экосистема, MDK 47.4.x |
| Мод на 1.21.1/1.21.11 | NeoForge 21.x + MDG/NG; Fabric — `fabric-loom-remap` | 1.21.11 — последняя обфусцированная; Yarn ещё доступен |
| Fabric + NeoForge одновременно | Architectury Loom или MultiLoader | Общий код + платформенные модули |
| 1.7.10 / 1.12.2 | RetroFuturaGradle 2.0.x + Java 8 toolchain | Единственный поддерживаемый современный путь для MCP-эпохи |
| 1.8–1.11 legacy | Unimined 2.0.0 | RFG эти версии не поддерживает |
| Много версий одной кодовой базой | Unimined | Мульти-версия/мульти-лоадер, вплоть до alpha |
| Серверный плагин без лоадера | VanillaGradle / обычный Gradle | Не нужен mod-лоадер |

Быстрый выбор плагина NeoForge:

- **новый проект, 26.x** → MDG 2.x;
- **существующий NG-проект** → NeoGradle 7.x;
- **нужен vanilla-модуль для мульти-лоадера** → MDG vanilla-mode (`neoFormVersion`);
- **нужны конвенции и `runTypes`** → NeoGradle 7.x.

---

## Проверено на

Дата проверки: **2026-11-21**.

- **Minecraft**: 26.3 (релиз 2026-09-15), последняя обфусцированная — 1.21.11.
- **NeoForge**: 26.3.0.113 (актуальная на дату), линейка 26.1.0.x-beta.
- **ModDevGradle**: 2.0.148 (MDK-26.3-ModDevGradle).
- **NeoGradle**: 7.1.39 (userdev maven metadata), линейка 7.1.21+.
- **NeoForm**: `26.3-1`, `26.1-1`, снапшоты `26.4-snapshot-10-1` (по maven-metadata).
- **Java**: 25 для 26.3; 17 для 1.18–1.20.4; 16 для 1.17.1; 8 для ≤1.16.5.
- **Gradle**: 9.2.1 (NeoForge MDK 26.3), 9.8.2 (Fabric example-mod 26.3), 8.8 (Forge 1.20.1 MDK), 7.6–8.8 (RFG).
- **Fabric Loom**: 1.19 (стабильный тег 2026-10-23; шаблон 26.3 пиннит `1.19-SNAPSHOT`); Loom 1.14 — split ID + Gradle 9.2 [12]; 1.15 — удалён FernFlower [13]; 1.16 — минимум Gradle 9.4 [14].
- **Fabric Loader**: 0.20.1; **Fabric API**: 0.166.2+26.3; **Yarn для 1.21.11**: 1.21.11+build.6.
- **Forge**: 1.20.1-47.4.0 MDK; ForgeGradle `[6.0,6.2)`.
- **RetroFuturaGradle**: 2.0.2 в официальном примере; 2.0.6 в списке версий сайта GTNH.
- **Architectury Loom**: линейка 1.14.x–1.17.x, актуальный plugin marker 1.17.493.
- **Unimined**: 2.0.0.

## Источники

1. NeoForged Docs — ModDevGradle: https://docs.neoforged.net/toolchain/docs/plugins/mdg/
2. NeoForged Docs — ModDevGradle README / BREAKING_CHANGES: https://github.com/neoforged/ModDevGradle
3. NeoForgeMDKs — MDK-26.3-ModDevGradle: https://github.com/NeoForgeMDKs/MDK-26.3-ModDevGradle
4. NeoForged Docs — NeoGradle: https://docs.neoforged.net/toolchain/docs/plugins/ng/
5. NeoGradle repository: https://github.com/neoforged/NeoGradle
6. NeoForged Docs — Parchment: https://docs.neoforged.net/toolchain/docs/parchment
7. NeoForged Docs — Getting Started (26.1): https://docs.neoforged.net/docs/gettingstarted/
8. NeoForm maven metadata: https://maven.neoforged.net/releases/net/neoforged/neoform/maven-metadata.xml
9. NeoForge project listing: https://projects.neoforged.net/neoforged/neoforge
10. Fabric Docs — Loom: https://docs.fabricmc.net/develop/loom/
11. Fabric Docs — Loom options: https://docs.fabricmc.net/develop/loom/options
12. Fabric Loom 1.14 release notes: https://github.com/FabricMC/fabric-loom/releases/tag/1.14
13. Fabric Loom 1.15 release notes: https://github.com/FabricMC/fabric-loom/releases/tag/1.15
14. Fabric Loom 1.16 release notes: https://github.com/FabricMC/fabric-loom/releases/tag/1.16
15. Fabric example mod, branch 26.3: https://github.com/FabricMC/fabric-example-mod/tree/26.3
16. Fabric Docs — Class Tweakers: https://docs.fabricmc.net/develop/class-tweakers/
17. Fabric Docs — Migrating Mappings: https://docs.fabricmc.net/develop/porting/mappings/
18. Fabric Loom maven metadata: https://maven.fabricmc.net/net/fabricmc/fabric-loom/maven-metadata.xml
19. RetroFuturaGradle repository: https://github.com/GTNewHorizons/RetroFuturaGradle
20. RetroFuturaGradle API docs: https://www.gtnewhorizons.com/RetroFuturaGradle
21. Legacy Modding Wiki — Gradle Plugins: https://legacymoddingmc.github.io/wiki/gradle-plugins
22. Forge Docs — Getting Started 1.20.1: https://docs.minecraftforge.net/en/1.20.1/gettingstarted/
23. ForgeGradle 6.x docs: https://docs.minecraftforge.net/en/fg-6.x/
24. ParchmentMC — Getting Started: https://parchmentmc.org/docs/getting-started
25. ForgeGradle repository (ветки FG_1.2…FG_7.0): https://github.com/MinecraftForge/ForgeGradle
26. MultiLoader Template: https://github.com/jaredlll08/MultiLoader-Template
27. Architectury Loom docs: https://docs.architectury.dev/loom/introduction
28. Architectury Loom repository: https://github.com/architectury/architectury-loom
29. Unimined: https://github.com/WagYourOrg/unimined
30. ModDevGradle maven metadata: https://maven.neoforged.net/releases/net/neoforged/moddev-gradle/maven-metadata.xml
31. Fabric Meta API (Yarn/Loader): https://meta.fabricmc.net/
