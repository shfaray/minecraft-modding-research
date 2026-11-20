# Источники и методология

> [!NOTE]
> Этот документ — не библиография ради галочки. Здесь собрано, **откуда взято каждое утверждение** и **что перепроверить**, когда выйдет 26.4 или изменится инструмент. Если вы нашли ошибку — начните с первоисточника, а не с этой базы.

## Содержание

- [Как проверялись факты](#как-проверялись-факты)
- [Версии и релизы](#версии-и-релизы)
- [Fabric](#fabric)
- [NeoForge и Forge](#neoforge-и-forge)
- [Mixin и MixinExtras](#mixin-и-mixinextras)
- [Инструменты](#инструменты)
- [Legacy-стек](#legacy-стек)
- [Отладка, тесты, профилирование](#отладка-тесты-профилирование)
- [Что перепроверить после 26.4](#что-перепроверить-после-264)
- [Ограничения](#ограничения)

## Как проверялись факты

Дата среза: **2026-11-21**. Версия-ориентир: **Minecraft 26.3**, последняя обфусцированная — **1.21.11**.

Правила:

1. **Официальная документация и исходники** важнее статей и видео. Для каждого спорного утверждения искался первоисточник: docs.fabricmc.net, docs.neoforged.net, javadoc Mixin, исходники на GitHub.
2. **Версии сверялись по maven-metadata и GitHub Releases**, а не по памяти: NeoForge, NeoForm, Loom, Fabric Loader, Forge, Quilt Loader, Ornithe.
3. **Всё неподтверждённое помечено** `[draft]`, `[community]` или `[unverified]`. Это не украшение: в 26.x документация ещё молодая, и часть практик существует только в community.
4. **Код не запускался** в рамках исследования. Рецепты и примеры — карта изменений, а не протестированный мод. Проверяйте у себя: `-Dmixin.debug.export=true`, `-Dmixin.debug.verbose=true`, затем сборка.
5. **Имена API не выдумывались.** Где сигнатура не подтверждена, в тексте стоит пометка и совет свериться с `javap`/IDE.

## Версии и релизы

| Источник | Что брать |
| --- | --- |
| [Minecraft: new version numbering](https://www.minecraft.net/en-us/article/minecraft-new-version-numbering-system) | Почему версии стали `26.x`, что означают части номера |
| [Java Edition 26.1](https://www.minecraft.net/en-us/article/minecraft-java-edition-26-1) | Tiny Takeover, Java 25 |
| [Java Edition 26.2](https://www.minecraft.net/en-us/article/minecraft-java-edition-26-2) | Vulkan, Graphics API, reversed depth |
| [Java Edition 26.3](https://www.minecraft.net/en-us/article/minecraft-java-edition-26-3) | Renderpearl, SDL3, OIT, форматы паков |
| [Java Edition version history](https://minecraft.wiki/w/Java_Edition_version_history) | Даты релизов, соответствие drop'ов версиям |
| [Java Edition 26.3](https://minecraft.wiki/w/Java_Edition_26.3) | Data version 5023, pack format 97.1/121.0, Java 25 |
| [Mojang version manifest](https://piston-meta.mojang.com/mc/game/version_manifest_v2.json) | Первичный список версий и ссылки на JSON |
| [Removing obfuscation in Java Edition](https://www.minecraft.net/en-us/article/removing-obfuscation-in-java-edition) | Официальное объявление деобфускации |

## Fabric

| Источник | Что брать |
| --- | --- |
| [Fabric Documentation](https://docs.fabricmc.net/develop/) | Основной гайд: проект, запуск, порт |
| [Fabric Wiki](https://wiki.fabricmc.net/) | Mixin tutorial, glossary, mappings |
| [Fabric: Removing Obfuscation](https://fabricmc.net/2025/10/31/obfuscation.html) | Конец Yarn/Intermediary, official names |
| [Fabric: Migrating Mappings](https://docs.fabricmc.net/develop/porting/mappings/) | Loom `migrateMappings` и Ravel |
| [Fabric: Class Tweakers](https://docs.fabricmc.net/develop/class-tweakers/) | `classTweaker v1`, Loader 0.18 / Loom 1.12 |
| [Fabric Loom releases](https://github.com/FabricMC/fabric-loom/releases) | Split plugin IDs в 1.14, Gradle 9.2, удаление FernFlower в 1.15 |
| [Fabric Loader](https://github.com/FabricMC/fabric-loader) | `Knot`, `MixinServiceKnot`, версии |
| [Fabric API porting guide 26.1](https://docs.fabricmc.net/develop/porting/) | Переименования Fabric API (`ItemGroupEvents` → `CreativeModeTabEvents`) |
| [Fabric example mod](https://github.com/FabricMC/fabric-example-mod) | Актуальный шаблон под 26.3 |
| [Loom options](https://docs.fabricmc.net/develop/loom/options) | `decompilers`, `mixin`, `runs`, `accessWidenerPath` |

## NeoForge и Forge

| Источник | Что брать |
| --- | --- |
| [NeoForge Documentation](https://docs.neoforged.net/) | Реестры, события, networking, AT, datagen |
| [NeoForge 26.1 release notes](https://neoforged.net/news/26.1release/) | Java 25, `ItemStackTemplate`, `GuiGraphicsExtractor`, MDG 2 |
| [NeoForge 21.4 release notes](https://neoforged.net/news/21.4release/) | Split `GatherDataEvent.Client/Server` |
| [NeoForge primers](https://docs.neoforged.net/primer/docs/) | Пошаговые миграции: 1.20.5, 1.21.x, 26.1–26.3 |
| [ModDevGradle](https://github.com/neoforged/ModDevGradle) | DSL, `loadedMods`, breaking changes 2.x |
| [NeoGradle](https://github.com/neoforged/NeoGradle) | Альтернативный плагин, 7.x |
| [NeoForm](https://github.com/neoforged/NeoForm) | Пайплайн исходников, версии `26.3-1` |
| [ModLauncher plumbing](https://github.com/MinecraftForge/ModLauncher/blob/master/plumbing.md) | `ILaunchPluginService`, трансформеры |
| [Forge MDKs](https://github.com/MinecraftForge/MDK) | Legacy-шаблоны 1.12.2–1.20.1 |
| [Forge Maven](https://maven.minecraftforge.net/net/minecraftforge/forge/maven-metadata.xml) | Актуальные версии Forge, включая `26.3-66.x` |

## Mixin и MixinExtras

| Источник | Что брать |
| --- | --- |
| [SpongePowered Mixin](https://github.com/SpongePowered/Mixin) | `At.java`, `ModifyVariable.java`, `InjectionPoint.java` — истина о точках |
| [Mixin wiki](https://github.com/SpongePowered/Mixin/wiki) | Концепции, debugging, конфликты |
| [MixinExtras wiki](https://github.com/LlamaLad7/MixinExtras/wiki) | `@WrapOperation`, `@ModifyExpressionValue`, `@Local`, `@Expression` |
| [MixinExtras source](https://github.com/LlamaLad7/MixinExtras) | Точные сигнатуры и поведение |
| [Fabric Mixin Glossary](https://wiki.fabricmc.net/tutorial:mixin_glossary) | Термины, `@Share`, `@Local` |
| [Fabric: Mixins and Obfuscation (draft)](https://wiki.fabricmc.net/drafts:mixin_obfuscation) | Refmap, `remap=false`, 26.1 — **черновик** |

## Инструменты

| Источник | Что брать |
| --- | --- |
| [Minecraft Development](https://mcdev.io/) | Возможности плагина IDEA |
| [MixinMCP](https://github.com/muon-rw/MixinMCP) | 25 инструментов, Gradle-плагин, ограничения |
| [jdx](https://github.com/MohammadMD1383/jdx) | Команды, MCP, лицензия |
| [Ravel](https://github.com/badasintended/ravel) | Миграция mappings, Kotlin, Mixin |
| [Vineflower](https://github.com/Vineflower/vineflower) | Флаги, качество вывода |
| [CFR](https://www.benf.org/other/cfr/) | Альтернативный декомпилятор |
| [Recaf](https://github.com/Col-E/Recaf) | Редактор байткода |
| [Bytecode Viewer](https://github.com/Konloch/bytecode-viewer) | Сравнение декомпиляторов |
| [Linkie](https://linkie.shedaniel.dev/) | Поиск по mappings до 1.21.11 |
| [mappings.dev](https://mappings.dev/) | Сравнение Yarn/Mojmap/SRG |

## Legacy-стек

| Источник | Что брать |
| --- | --- |
| [RetroFuturaGradle](https://github.com/GTNewHorizons/RetroFuturaGradle) | 1.7.10/1.12.2, Gradle 7.6–8.8 |
| [Unimined](https://github.com/unimined/Unimined) | Мульти-версия и legacy 1.8–1.11 |
| [Legacy Fabric](https://legacyfabric.net/) | 1.3–1.14 snapshots |
| [Ornithe](https://ornithemc.net/) | Feather, Calamus, Ploceus |
| [Babric](https://babric.github.io/) | Beta 1.7.3, StationAPI |
| [Rift](https://github.com/DimensionalDevelopment/Rift) | 1.13, архив |

## Отладка, тесты, профилирование

| Источник | Что брать |
| --- | --- |
| [Mixin Debugging wiki](https://github.com/SpongePowered/Mixin/wiki/Mixin-Debugging) | Флаги `mixin.debug.*` |
| [MixinTrace](https://github.com/comp500/mixintrace) | Миксины в стектрейсах |
| [Crash Assistant](https://github.com/KostromDan/Crash-Assistant) | Разбор crash-report |
| [spark](https://spark.lucko.me/) | Профилирование CPU/heap |
| [JFR](https://docs.oracle.com/en/java/javase/25/jfr/) | Встроенный профайлер JVM |
| [GameTest](https://docs.fabricmc.net/develop/automatic-testing) | Автотесты в мире |
| [MC-Runtime-Test](https://github.com/headlesshq/mc-runtime-test) | Клиент в CI |
| [HeadlessMc](https://github.com/headlesshq/headlessmc) | Запуск без GPU/дисплея |

## Что перепроверить после 26.4

Каждый новый drop ломает что-то из этого списка. Чеклист на обновление базы:

- [ ] Номера `data pack` / `resource pack` форматов и синтаксис `pack.mcmeta`.
- [ ] `GuiGraphicsExtractor` и контракт `extractRenderState` / `Hud` — рендер меняется чаще всего.
- [ ] `ItemStackTemplate`, `ItemInstance`, data components — не появилось ли новых обязательных полей.
- [ ] Версии Loom / MDG / NeoGradle и минимальный Gradle.
- [ ] `compatibilityLevel` в `mixins.json` (сейчас `JAVA_25` для 26.x).
- [ ] Актуальные версии NeoForge, Forge, Fabric Loader, Quilt Loader.
- [ ] Не появилось ли новых `Renderpearl`-пакетов или переименований в `com.mojang.blaze3d`.
- [ ] Статус Yarn/Intermediary/Parchment: не объявили ли их возврат или новую замену.
- [ ] Инструменты: не сломался ли MixinMCP/jdx/Ravel на новой версии IDEA или Gradle.

## Ограничения

- **26.x — свежая эпоха.** Часть документации ещё в статусе draft, часть практик не устоялась. Если источник помечен `[unverified]`, это значит «мы не смогли подтвердить в живой игре».
- **Код не тестировался.** Ни один пример не собирался в рамках исследования; это сознательный выбор в пользу охвата, а не глубины по одному моду.
- **Версии инструментов меняются быстрее, чем этот текст.** Все точные номера — снимок на 2026-11-21; перед копированием в проект сверяйтесь с maven-metadata.
- **Мнения отделены от фактов.** Фразы вроде «берите `@WrapOperation`» — это рекомендация на основе практики сообщества, а не требование Mojang.

---

[← К оглавлению](README.md)
