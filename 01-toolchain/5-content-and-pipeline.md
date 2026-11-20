# Контент, ассеты и публикация

[← К оглавлению](../README.md)

> [!NOTE]
> **TL;DR.** Всё, что можно описать JSON'ом — модели, blockstates, рецепты, теги, loot, advancements, lang — должно генерироваться datagen'ом, а не писаться руками. Модели делаются в Blockbench и раскладываются по структуре `assets/`, начиная с 1.21.4 предметы используют `items/*.json`. KubeJS/CraftTweaker закрывают data-driven контент, но не поведение — для поведения нужен Java + Mixin. Релиз — это не `build`, а версия, changelog, лицензия, тесты на чистом клиенте и сервере, и автоматическая публикация через Minotaur или `mod-publish-plugin`.

## Содержание

- [1. Datagen](#1-datagen)
  - [1.1. Зачем datagen](#11-зачем-datagen)
  - [1.2. NeoForge: `GatherDataEvent` и `runData`](#12-neoforge-gatherdataevent-и-rundata)
  - [1.3. Fabric: `FabricDataGenerator` и `runDatagen`](#13-fabric-fabricdatagenerator-и-rundatagen)
  - [1.4. Что генерировать, а что нет](#14-что-генерировать-а-что-нет)
  - [1.5. Грабли datagen](#15-грабли-datagen)
- [2. Модели и ассеты](#2-модели-и-ассеты)
  - [2.1. Blockbench](#21-blockbench)
  - [2.2. ItemModel 1.21.4+](#22-itemmodel-1214)
  - [2.3. Blockstates и модели блоков](#23-blockstates-и-модели-блоков)
  - [2.4. Texture atlas](#24-texture-atlas)
  - [2.5. `pack.mcmeta` и форматы](#25-packmcmeta-и-форматы)
- [3. Скриптинг без Java](#3-скриптинг-без-java)
- [4. NBT, структуры и миры](#4-nbt-структуры-и-миры)
- [5. Генераторы данных](#5-генераторы-данных)
- [6. Публикация и CI](#6-публикация-и-ci)
- [7. Версионирование мода](#7-версионирование-мода)
- [8. Чеклист релиза](#8-чеклист-релиза)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## 1. Datagen

### 1.1. Зачем datagen

Datagen — это Java-код, который на этапе сборки пишет JSON в `src/generated/resources` (NeoForge) или `src/main/generated` (Fabric). Вместо сотни файлов, в которых легко перепутать `minecraft:item/foo` и `minecraft:block/foo`, вы пишете типизированный код с автодополнением, а компилятор ловит опечатки в `ResourceLocation`.

Что это даёт:

- **Ссылочную целостность.** Тег ссылается на предмет, который реально зарегистрирован; loot table использует существующий блок; advancement — существующий предмет.
- **Единый источник правды.** Рецепт, его unlock-advancement и тег генерируются рядом и не расходятся.
- **Ревью.** В diff видно «добавлен рецепт», а не «изменены 40 JSON-файлов».
- **Меньше merge-конфликтов.** Файлы генерируются заново, а не правятся вручную.

> [!WARNING]
> Datagen **не создаёт текстуры, звуки и модели, нарисованные в Blockbench**. Он создаёт только JSON-описания. `.png`, `.ogg` и экспортированные `.json`-модели остаются обычными ресурсами в `src/main/resources`.

### 1.2. NeoForge: `GatherDataEvent` и `runData`

В NeoForge datagen — это отдельный run config, который проходит жизненный цикл мода до события реестров, затем вызывает `GatherDataEvent`, даёт провайдерам записать файлы и завершает процесс.

С 26.x событие разделено на два подтипа:

| Событие | Что можно регистрировать | Когда использовать |
| --- | --- | --- |
| `GatherDataEvent.Client` | Все провайдеры: и assets, и data | Единый `runClientData` (так делает MDK) |
| `GatherDataEvent.Server` | Только datapack-провайдеры | Отдельный `runServerData`, если клиентские ассеты не нужны |

Историческое имя `runData` осталось в старых туториалах и MDK для 1.20.x. В актуальном MDK run config называется `clientData`, поэтому задача — `runClientData`; если блок в `build.gradle` назван `data`, Gradle создаст `runData`. Отдельный `runServerData` появляется только если вы завели серверный data-run. Поэтому «`./gradlew runData`» может работать в одном шаблоне и отсутствовать в другом — смотрите `./gradlew tasks`.

#### Провайдеры NeoForge

| Класс | Метод | Что генерирует | Сторона |
| --- | --- | --- | --- |
| `ModelProvider` | `registerModels()` | Модели блоков/предметов, blockstates, client items | client |
| `LanguageProvider` | `addTranslations()` | `lang/*.json` | client |
| `SoundDefinitionsProvider` | `registerSounds()` | `sounds.json` | client |
| `SpriteSourceProvider` | `gather()` | `atlases/*.json` | client |
| `EquipmentAssetProvider` | `registerModels()` | Ассеты брони | client |
| `ParticleDescriptionProvider` | `addDescriptions()` | Описания партиклов | client |
| `RecipeProvider` | `buildRecipes()` (в 1.21.x — `buildRecipes(RecipeOutput)`) | Рецепты | server |
| `LootTableProvider` | `generate()` | Loot tables | server |
| `TagsProvider` (и подклассы) | `addTags(HolderLookup.Provider)` | Теги блоков/предметов/… | server |
| `AdvancementProvider` | `generate()` | Advancements | server |
| `DataMapProvider` | `gather()` | Data map entries | server |
| `GlobalLootModifierProvider` | `start()` | Global loot modifiers | server |
| `DatapackBuiltinEntriesProvider` | — | Worldgen, damage types, enchantments | server |
| `PackMetadataGenerator` | `add(...)` | `pack.mcmeta` | обе |
| `JsonCodecProvider` | `gather()` | Любой объект с codec | обе |

#### Пример для NeoForge 26.3

```java
// src/main/java/com/example/examplemod/datagen/DataGenerators.java
package com.example.examplemod.datagen;

import net.neoforged.bus.api.SubscribeEvent;
import net.neoforged.fml.common.EventBusSubscriber;
import net.neoforged.neoforge.data.event.GatherDataEvent;

@EventBusSubscriber(modid = ExampleMod.MOD_ID, bus = EventBusSubscriber.Bus.MOD)
public final class DataGenerators {
    private DataGenerators() {}

    @SubscribeEvent
    public static void gatherData(GatherDataEvent.Client event) {
        // Провайдеры, которым нужен только PackOutput
        event.createProvider(output -> new ExampleLanguageProvider(output, "en_us"));
        event.createProvider(ExampleModelProvider::new);

        // Провайдер, которому нужен HolderLookup.Provider
        event.createProvider(ExampleRecipeProvider.Runner::new);

        // Теги блоков и предметов одной строкой
        event.createBlockAndItemTags(ExampleBlockTags::new, ExampleItemTags::new);

        // Datapack-реестры (worldgen, damage types и т.п.)
        event.createDatapackRegistryObjects(
                new net.minecraft.core.RegistrySetBuilder()
                        .add(net.minecraft.core.registries.Registries.DAMAGE_TYPE,
                                ExampleDamageTypes::bootstrap)
        );

        // Отдельный datapack внутри мода
        var pack = event.getGenerator().getBuiltinDatapack(true, ExampleMod.MOD_ID, "example_pack");
        pack.addProvider(ExamplePackProvider::new);
    }
}
```

Провайдер-рецепт в 26.3 оборачивается в `Runner`, потому что `RecipeProvider` требует `HolderLookup.Provider` и `RecipeOutput`:

```java
public class ExampleRecipeProvider extends RecipeProvider {
    public ExampleRecipeProvider(HolderLookup.Provider registries, RecipeOutput output) {
        super(registries, output);
    }

    @Override
    protected void buildRecipes() {
        shaped(RecipeCategory.MISC, ExampleMod.RUBY.get())
                .pattern("###")
                .pattern("###")
                .pattern("###")
                .define('#', ExampleMod.RUBY_BLOCK.get())
                .unlockedBy("has_ruby_block", has(ExampleMod.RUBY_BLOCK.get()))
                .save(output);
    }

    public static class Runner extends RecipeProvider.Runner {
        public Runner(PackOutput output, CompletableFuture<HolderLookup.Provider> registries) {
            super(output, registries);
        }

        @Override
        protected RecipeProvider createRecipeProvider(HolderLookup.Provider registries,
                                                      RecipeOutput output) {
            return new ExampleRecipeProvider(registries, output);
        }
    }
}
```

`build.gradle` (ModDevGradle 2.x, схема может отличаться в 1.x):

```groovy
neoForge {
    runs {
        clientData {
            clientData() // тип run config для 1.21.4+; до 1.21.3 — data()
            programArguments.addAll '--mod', 'examplemod',
                    '--output', file('src/generated/resources').absolutePath,
                    '--existing', file('src/main/resources').absolutePath,
                    '--all'
        }
    }
}

// Сгенерированные ресурсы должны попасть в jar
sourceSets.main.resources.srcDir 'src/generated/resources'
```

Запуск:

```bash
./gradlew runClientData
# или, если настроен отдельный серверный run:
./gradlew runServerData
```

Ключевые аргументы генератора:

| Аргумент | Что делает |
| --- | --- |
| `--mod examplemod` | Для какого мода генерировать; обязателен при нескольких модах в проекте |
| `--output <path>` | Куда писать; по умолчанию `src/generated/resources` |
| `--existing <path>` | Учитывать существующие файлы (обычно `src/main/resources`) |
| `--existing-mod examplemod` | Учитывать ресурсы из jar этого мода |
| `--includeReports` | Дамп зарегистрированных объектов (для отладки) |
| `--all` | Включить все режимы генератора |

> [!TIP]
> Если генератор падает с `FileAlreadyExistsException` или «cannot find registry entry», почти всегда виноват порядок: провайдер, которому нужен `HolderLookup.Provider`, зарегистрирован раньше `createDatapackRegistryObjects`, либо в `--existing` не передан каталог с ручными ресурсами.

### 1.3. Fabric: `FabricDataGenerator` и `runDatagen`

В Fabric datagen — это entrypoint `fabric-datagen` и Gradle-задача `runDatagen`. Провайдеры складываются в `FabricDataGenerator.Pack`, а результат по умолчанию пишется в `src/main/generated`.

`build.gradle`:

```groovy
fabricApi {
    configureDataGeneration {
        client = true // нужен, если генерируем модели, lang и прочие client assets
    }
}

// Чтобы сгенерированные файлы попали в jar
sourceSets.main.resources.srcDirs += ['src/main/generated']
```

`fabric.mod.json`:

```json
{
  "entrypoints": {
    "main": ["com.example.examplemod.ExampleMod"],
    "fabric-datagen": [
      "com.example.examplemod.datagen.ExampleModDataGenerator"
    ]
  }
}
```

Сам генератор:

```java
public class ExampleModDataGenerator implements DataGeneratorEntrypoint {
    @Override
    public void onInitializeDataGenerator(FabricDataGenerator generator) {
        FabricDataGenerator.Pack pack = generator.createPack();

        pack.addProvider(ExampleModelProvider::new);
        pack.addProvider(output -> new ExampleLanguageProvider(output, "en_us"));
        pack.addProvider(ExampleRecipeProvider::new);
        pack.addProvider(ExampleBlockTagProvider::new);
        pack.addProvider(ExampleLootTableProvider::new);
        pack.addProvider(ExampleAdvancementProvider::new);
    }
}
```

Запуск:

```bash
./gradlew runDatagen
```

Провайдеры Fabric API:

| Класс | Что генерирует |
| --- | --- |
| `FabricModelProvider` (через `FabricBlockStateModelGenerator` и `FabricItemModelGenerator`) | Blockstates, модели блоков, `items/*.json` |
| `FabricLanguageProvider` | `lang/*.json` |
| `FabricRecipeProvider` | Рецепты |
| `FabricTagProvider.BlockTagProvider` / `ItemTagProvider` | Теги |
| `FabricLootTableProvider` / `SimpleFabricLootTableProvider` | Loot tables |
| `FabricAdvancementProvider` | Advancements |

> [!NOTE]
> Провайдер для `sounds.json` в Fabric API появлялся не во всех версиях. Если в вашей версии нет `FabricSoundProvider`, генерируйте `sounds.json` обычным `JsonCodecProvider`-подобным провайдером или держите файл в `src/main/resources` `[unverified]`.

> [!NOTE]
> На 26.3 Fabric-документация использует `configureDataGeneration { client = true }`. В старых проектах встречается `configureDataGeneration()` без аргументов и `fabricApi.configureDataGeneration()` — это тот же DSL, но с другим набором опций.

### 1.4. Что генерировать, а что нет

| Контент | Datagen? | Почему |
| --- | --- | --- |
| Blockstates, модели блоков/предметов | Да | Скучно, много, легко ошибиться в путях |
| `items/*.json` (ItemModel 1.21.4+) | Да | Ссылки на модели и tint sources |
| Рецепты | Да | Связаны с тегами и advancements |
| Теги | Да | Особенно `c:`-теги и теги совместимости |
| Loot tables | Да | Связаны с блоками, сущностями, data components |
| Advancements | Да | Ссылаются на предметы и критерии |
| `lang/en_us.json` | Да | Ключи не расходятся с кодом |
| `pack.mcmeta` | Да (опционально) | Формат меняется каждый drop |
| Текстуры `.png` | Нет | Рисуются вручную/Aseprite |
| Звуки `.ogg` | Нет | Записываются отдельно |
| Модели из Blockbench | Нет | Экспортируются как JSON и кладутся в `resources` |
| Структуры `.nbt` | Нет | Строятся в мире и сохраняются структурным блоком |

### 1.5. Грабли datagen

- **`clean` не удаляет `src/generated`.** Это не build-каталог. Если генератор изменил путь или вы удалили провайдер, старые файлы останутся и попадут в jar. Чистите каталог вручную или отдельной задачей.
- **Порядок провайдеров имеет значение.** `DatapackBuiltinEntriesProvider` должен быть зарегистрирован до провайдеров, которые на него ссылаются.
- **`HolderLookup.Provider` — не `RegistryAccess`.** В 26.x реестры datapack-driven; для тегов и рецептов нужен именно lookup, полученный из события.
- **`ItemStack` в рецептах и advancement'ах.** С 26.1 используйте `ItemStackTemplate`: `ItemStack` требует загруженных реестров и в datagen может вести себя не так, как в игре.
- **Advancement'ы не появляются без `unlockedBy`.** Рецепт сгенерируется, но не будет открыт в книге рецептов.
- **Клиентские провайдеры в `GatherDataEvent.Server`.** Модели и lang просто не сгенерируются; на NeoForge это ошибка конфигурации, а не «пустой файл».
- **Datagen — это код, который выполняется.** Если мод падает при инициализации, datagen упадёт вместе с ним. Иногда проще временно отключить проблемный provider, чем ловить стектрейс в генераторе.
- **Проверка в CI.** Дешёвый способ поймать рассинхрон:

```bash
# Fabric
./gradlew runDatagen
# NeoForge — раскомментируйте существующую задачу:
# ./gradlew runClientData   # или runData
git diff --exit-code -- src/generated/resources src/main/generated
```

---

## 2. Модели и ассеты

### 2.1. Blockbench

[Blockbench](https://www.blockbench.net/) — бесплатный редактор low-poly моделей, лицензия GPL-3.0. Для Java Edition нужен формат **Java Block/Item**: он умеет кубы, UV, `display`, `gui_light` и экспортирует JSON, совместимый с `models/block` и `models/item`.

Базовый маршрут:

1. Создать проект **Java Block/Item**.
2. Выставить размер текстуры (обычно 16×16 или 32×32) и нарисовать/импортировать PNG.
3. Смоделировать кубы, выставить UV.
4. `File → Export → Java Block/Item Model` — получить JSON.
5. Положить JSON в `src/main/resources/assets/<ns>/models/block|item/`, текстуру — в `assets/<ns>/textures/block|item/`.
6. Blockstate и item model definition — через datagen, а не руками.

Полезные плагины:

| Плагин | Зачем | Важная граница |
| --- | --- | --- |
| **Animated Java** | Анимации моделей и риг, экспорт в datapack-анимации | Это **datapack + resource pack**, а не Java-анимация сущности; для серверной логики и интерполяции в коде он не заменяет GeckoLib |
| **GeckoLib Animation Utils** | Экспорт `.geo.json` и `.animation.json` для GeckoLib | Нужен сам GeckoLib в зависимостях; для предметов/блоков с анимацией проверяйте поддержку версии |
| **Blockbench UV Editor** (встроен) | Точная развёртка | — |

> [!WARNING]
> Animated Java и GeckoLib решают разные задачи. Animated Java — про анимации в мире через datapack; GeckoLib — про Java-рендер сущностей/предметов с интерполяцией и контролем из кода. Выбирайте по тому, **кто управляет анимацией**: команда/датапак или Java-код.

### 2.2. ItemModel 1.21.4+

До 1.21.4 предмет описывался одним `models/item/<id>.json` с `parent: "item/generated"` и `textures.layer0`. С 1.21.4 появились **client item definitions**:

- `assets/<ns>/items/<id>.json` — что и как рендерить (тип модели, tint sources, conditions, select).
- `assets/<ns>/models/item/<id>.json` — геометрия/иконка (может наследовать `item/generated`).

Минимальный `items/ruby.json`:

```json
{
  "model": {
    "type": "minecraft:model",
    "model": "examplemod:item/ruby"
  }
}
```

`models/item/ruby.json`:

```json
{
  "parent": "minecraft:item/generated",
  "textures": {
    "layer0": "examplemod:item/ruby"
  }
}
```

Типы client item model:

| Тип | Назначение |
| --- | --- |
| `minecraft:model` | Обычная модель; поддерживает `tints` |
| `minecraft:composite` | Несколько моделей одновременно |
| `minecraft:select` | Выбор по свойству (например, `display_context`, `charge_type`) |
| `minecraft:range_dispatch` | Выбор по числу (damage, use duration) |
| `minecraft:condition` | Выбор по boolean-свойству (broken, damaged, has_component) |
| `minecraft:special` | Специальный рендер (сундук, шалкер, баннер, кровать) |
| `minecraft:empty` | Ничего не рисовать |

Своя tint source (например, цвет от погоды) требует `ItemTintSource` + codec и регистрации в `ItemTintSources.ID_MAPPER` на клиенте. Это один из немногих случаев, когда ассет требует Java-кода.

Типичные ошибки:

- Файл лежит в `models/item/`, а игра ищет `items/` — предмет «отсутствует» (missing model).
- В `items/*.json` указан `examplemod:models/item/ruby` вместо `examplemod:item/ruby`: путь модели — без `models/`.
- Забыт `parent: "item/generated"` — нет `layer0`, предмет белый/чёрный.
- Текстура не в атласе: для предметов обычно `textures/item`, но кастомные атласы требуют `atlases/*.json`.
- `tints` добавлены в `models/item`, а не в `items/*.json` — tint применяется на уровне client item, не модели.

### 2.3. Blockstates и модели блоков

Blockstate описывает, какая модель соответствует состоянию блока:

```json
{
  "variants": {
    "facing=north": { "model": "examplemod:block/ruby_lamp" },
    "facing=south": { "model": "examplemod:block/ruby_lamp", "y": 180 },
    "facing=east":  { "model": "examplemod:block/ruby_lamp", "y": 90 },
    "facing=west":  { "model": "examplemod:block/ruby_lamp", "y": 270 }
  }
}
```

Для блоков с комбинаторикой используется `multipart`:

```json
{
  "multipart": [
    { "apply": { "model": "examplemod:block/pipe_core" } },
    { "when": { "north": "true" }, "apply": { "model": "examplemod:block/pipe_arm", "y": 0 } },
    { "when": { "east": "true" },  "apply": { "model": "examplemod:block/pipe_arm", "y": 90 } }
  ]
}
```

Модель блока — обычный JSON с `parent`, `textures`, `elements`, `display`. Не пишите `elements` руками, если модель можно собрать в Blockbench: ошибки в UV и `cullface` видны только в игре.

> [!TIP]
> В datagen для blockstates используйте `BlockStateModelGenerator`: он сам создаст `cube_all`, `orientable`, `stairs`, `slab` и другие стандартные варианты. Ручной JSON нужен только для нестандартной геометрии.

### 2.4. Texture atlas

Атлас — это склейка текстур, которую игра грузит одним файлом. Для блоков и предметов ванильные атласы уже включают стандартные каталоги. Свой атлас нужен, если текстуры лежат вне `textures/block`, `textures/item` или вы делаете GUI-атлас.

`assets/examplemod/atlases/blocks.json`:

```json
{
  "sources": [
    {
      "type": "minecraft:directory",
      "prefix": "block/",
      "source": "block"
    },
    {
      "type": "minecraft:single",
      "resource": "examplemod:block/ruby_ore"
    }
  ]
}
```

Типы sources: `minecraft:directory`, `minecraft:single`, `minecraft:filter`, `minecraft:unstitch`, `minecraft:paletted_permutations`. Для динамических атласов (GUI, скины) проверяйте, какой атлас ожидает ванильный код: `minecraft:blocks`, `minecraft:gui`, `minecraft:paintings` и т.д.

### 2.5. `pack.mcmeta` и форматы

С 1.21.9 формат пака задаётся парой `min_format` / `max_format`; `supported_formats` удалён (кроме случая, когда пак должен грузиться на версиях ниже формата 65). Для 26.3:

- resource pack format — `97.1`;
- data pack format — `121.0`.

`src/main/resources/pack.mcmeta` для ресурсов 26.3:

```json
{
  "pack": {
    "description": "Example Mod Resources",
    "min_format": [97, 1],
    "max_format": 97
  }
}
```

`src/main/generated/pack.mcmeta` для данных:

```json
{
  "pack": {
    "description": "Example Mod Data",
    "min_format": [121, 0],
    "max_format": 121
  }
}
```

`max_format: 97` означает «любая минорная версия 97.x», то есть пак продолжит грузиться в 26.3.x. Если поставить `[97, 0]`, пак будет считаться слишком старым для 26.3.

> [!IMPORTANT]
> Номера форматов меняются каждый drop. Не копируйте `97.1` в проект под 26.4 — сверяйтесь с `version.json` внутри клиентского jar или с [Minecraft Wiki: Pack format](https://minecraft.wiki/w/Pack_format).

Языковые файлы: `assets/<ns>/lang/en_us.json`, `ru_ru.json` и т.д. Ключи предметов — `item.examplemod.ruby`, блоков — `block.examplemod.ruby_ore`, вкладок — `itemGroup.examplemod.main`. Форматирование — `%s`, `%d`, `%1$s`; не забывайте экранировать `%` как `%%`.

Звуки: `assets/<ns>/sounds.json` + `.ogg` в `assets/<ns>/sounds/`. Громкость и pitch задаются в коде при вызове `playSound`; `sounds.json` описывает только события и субтитры.

---

## 3. Скриптинг без Java

Скриптовые моды позволяют менять datapack-контент без пересборки Java-мода. Они полезны для сборок, серверов и быстрых экспериментов, но у них есть жёсткая граница: **скрипт не может переопределить произвольное поведение Java-класса**.

| Инструмент | Язык | Где лежит | Перезагрузка | Что умеет | Чего не умеет |
| --- | --- | --- | --- | --- | --- |
| **KubeJS** | JavaScript (Rhino) | `kubejs/server_scripts`, `client_scripts`, `startup_scripts` | `/reload` для server, перезапуск для startup/client | Рецепты, теги, loot, advancements, предметы/блоки (startup), интеграции с модами | Менять AI сущностей, рендер, произвольные методы |
| **CraftTweaker** | ZenScript | `scripts/*.zs` | `/reload` / `/ct reload` | Рецепты, теги, loot modifiers, JEI-описания | Добавлять блоки/предметы в новых версиях, менять поведение |
| **GroovyScript** | Groovy | `groovy/` | `/reload` (зависит от версии) | Рецепты, теги, worldgen, интеграции; мощный язык | В основном 1.12.2/Cleanroom; на 26.x не универсален |

### KubeJS

```js
// kubejs/server_scripts/recipes.js
ServerEvents.recipes(event => {
  event.shaped('examplemod:ruby_block', [
    'RRR',
    'RRR',
    'RRR'
  ], {
    R: 'examplemod:ruby'
  })

  event.smelting('examplemod:ruby', 'examplemod:ruby_ore')
})

ServerEvents.tags('item', event => {
  event.add('c:gems/ruby', 'examplemod:ruby')
})
```

```js
// kubejs/startup_scripts/items.js
StartupEvents.registry('item', event => {
  event.create('examplemod:ruby')
    .displayName('Ruby')
    .maxStackSize(64)
})
```

KubeJS хорош, когда нужно быстро собрать рецепты для сборки или сервера. Он не предназначен для замены Java-мода: API меняется между версиями, а скрипты сложно тестировать и отлаживать.

### CraftTweaker

```zenscript
// scripts/recipes.zs
craftingTable.addShaped("ruby_block", <item:examplemod:ruby_block>, [
  [<item:examplemod:ruby>, <item:examplemod:ruby>, <item:examplemod:ruby>],
  [<item:examplemod:ruby>, <item:examplemod:ruby>, <item:examplemod:ruby>],
  [<item:examplemod:ruby>, <item:examplemod:ruby>, <item:examplemod:ruby>]
]);
```

CraftTweaker стабильнее и лучше документирован для рецептов, но его DSL менее выразителен, чем JavaScript. Для генерации сотен рецептов из таблицы данных он неудобен.

### GroovyScript

```groovy
// groovy/postInit/recipes.groovy
crafting.shapedBuilder()
    .output(item('examplemod:ruby_block'))
    .matrix('RRR', 'RRR', 'RRR')
    .key('R', item('examplemod:ruby'))
    .register()
```

GroovyScript — мощный инструмент для legacy-сборок (особенно 1.12.2/Cleanroom), но не стоит закладываться на него как на универсальный слой для 26.x без проверки совместимости.

### Когда лучше миксин

| Задача | Инструмент |
| --- | --- |
| Добавить рецепт, тег, loot, advancement | Datagen или KubeJS/CraftTweaker |
| Добавить предмет/блок с новым поведением | Java + datagen |
| Изменить AI моба, урон, скорость | Mixin / события лоадера |
| Изменить рендер GUI или сущности | Mixin / API лоадера |
| Добавить совместимость с чужим модом | Mixin + проверка версии |
| Сделать datapack для карты | JSON/datagen, без мода |
| Быстро поправить рецепты на сервере | KubeJS/CraftTweaker |

---

## 4. NBT, структуры и миры

| Инструмент | Что делает | Кому нужен | Лицензия/статус |
| --- | --- | --- | --- |
| **NBT Studio** | Просмотр/редактирование NBT, SNBT, region-файлов | Мододелу: block entity, entity, structure | Лицензия в репозитории не указана `[unverified]`; последний push — 2024 |
| **NBTExplorer** | Классический редактор NBT | Наследие; для старых версий | Давно не обновляется, для 26.x не рекомендуется |
| **Amulet** | Редактор миров и библиотека для чтения/записи save-форматов | Картостроителю, тем кто переносит миры | Open source, лицензия — см. репозиторий |
| **WorldEdit** | Редактирование мира в игре, схемы, brushes | Картостроителю и мододелу для тестовых миров | GPL-3.0 (см. репозиторий) |
| **Axiom** | Строительство/моделирование в игре, blueprints | Картостроителю, дизайнеру | Бесплатно, closed-source `[unverified]`; не библиотека для мода |

### Что нужно мододелу

- **NBT Studio** — посмотреть, как игра реально хранит block entity, entity или предмет с data components. Полезно, когда код пишет «правильно», а в мире ничего нет.
- **Структурный блок** — сохранить тестовую постройку в `.nbt` и положить в `data/<ns>/structure/`. Структура содержит `DataVersion`; при переносе между версиями её лучше пересоздать, а не править.
- **`/data get`** — быстрый способ увидеть NBT без внешних инструментов:

```mcfunction
/data get block ~ ~ ~
/data get entity @e[type=minecraft:zombie,limit=1]
/data get entity @s SelectedItem
```

- **`--reports`** в datagen — дамп реестров и тегов, полезен для проверки, что тег действительно содержит нужный элемент.

### Что нужно картостроителю

- **WorldEdit** — массовые операции, `//set`, `//copy`, `//paste`, `//schem save`. Формат схем — Sponge `.schem`; для мода структуры удобнее `.nbt`.
- **Amulet** — открыть мир, заменить блоки, перенести регион, посмотреть палитру. Не все версии и моды поддерживаются; делайте бэкап.
- **Axiom** — быстрый билдинг и blueprints. Не является dependency мода и не должен требоваться игрокам.

> [!CAUTION]
> Любое редактирование мира — только на копии. Amulet и WorldEdit не гарантируют сохранение data components и block entity NBT чужого мода; после редактирования проверяйте `/data get` и логи.

---

## 5. Генераторы данных

| Ресурс | Что даёт | Ссылка |
| --- | --- | --- |
| **Misode** | Генераторы рецептов, loot tables, advancements, тегов, worldgen, item models, предикатов | <https://misode.github.io/> |
| **mcasset.cloud** | Браузер ванильных ассетов по версиям: текстуры, модели, lang, sounds | <https://mcasset.cloud/> |
| **Minecraft Wiki: Data values** | ID блоков/предметов, block states, damage types, enchantments | <https://minecraft.wiki/w/Data_values> |
| **minecraft-schemas** | JSON-схемы для автодополнения и валидации в VS Code | <https://github.com/misode/minecraft-schemas> |

Рабочий процесс:

1. Сгенерировать JSON в Misode для нужной версии.
2. Понять, какие поля обязательны, а какие — дефолты.
3. Перенести логику в datagen-провайдер, а не копировать JSON в `resources`.
4. Проверить имена через mcasset.cloud (текстуры/модели) и Wiki (ID/block states).
5. Добавить схему в VS Code, если JSON всё-таки правится руками (точный путь к схеме сверьте в репозитории `minecraft-schemas`):

```json
{
  "json.schemas": [
    {
      "fileMatch": ["src/main/resources/**/*.json"],
      "url": "https://raw.githubusercontent.com/misode/minecraft-schemas/main/schemas/26.3/recipe.json"
    }
  ]
}
```

> [!NOTE]
> Схемы и генераторы обновляются под конкретные версии. Для 26.3 проверяйте, что выбрана ветка/версия 26.3, а не 1.21.x: имена полей и типы рецептов между ними отличаются.

---

## 6. Публикация и CI

### 6.1. Куда публиковать

| Площадка | Через что | Когда |
| --- | --- | --- |
| Modrinth | Minotaur, `mod-publish-plugin` | Основная площадка для большинства модов |
| CurseForge | `mod-publish-plugin`, CurseGradle (legacy) | Крупные сборки и историческая аудитория |
| GitHub Releases | `mod-publish-plugin`, `softprops/action-gh-release` | Артефакты, исходники, changelog |
| JitPack | GitHub-теги | API-библиотеки, а не игровые моды |
| Maven (свой/Nexus) | `maven-publish` | Внутренние зависимости и API |

### 6.2. Modrinth Minotaur

Minotaur — Gradle-плагин, лицензия LGPL-2.1. Подключение:

```groovy
plugins {
    id 'com.modrinth.minotaur' version '2.+'
}

modrinth {
    token = System.getenv('MODRINTH_TOKEN')
    projectId = 'examplemod'
    versionNumber = project.version
    versionType = 'release' // release | beta | alpha
    uploadFile = remapJar  // с Loom обязателен remapJar, не jar
    gameVersions = ['26.3']
    loaders = ['fabric']
    changelog = rootProject.file('CHANGELOG.md').text
    dependencies {
        required.project 'fabric-api'
        optional.project 'sodium'
    }
}
```

Токен создаётся в настройках аккаунта Modrinth; нужны scopes `CREATE_VERSION` (для `modrinth`) и `PROJECT_WRITE` (для `modrinthSyncBody`). Секрет — только через переменную окружения, никогда в `build.gradle`.

Синхронизация описания:

```groovy
modrinth {
    syncBodyFrom = rootProject.file('README.md').text
}
tasks.modrinth.dependsOn(tasks.modrinthSyncBody)
```

`syncBodyFrom` перезаписывает тело проекта; сначала проверьте через `debugMode = true`.

### 6.3. `mod-publish-plugin`

Плагин `me.modmuss50.mod-publish-plugin` (MIT, версия 2.2.1 на сентябрь 2026) публикует в CurseForge, Modrinth, GitHub, GitLab, Gitea, Discord и поддерживает `dryRun`.

```groovy
plugins {
    id 'me.modmuss50.mod-publish-plugin' version '2.2.1'
}

publishMods {
    file = remapJar.archiveFile
    version = project.version
    type = STABLE // STABLE | BETA | ALPHA
    changelog = rootProject.file('CHANGELOG.md').text
    modLoaders.add('fabric')

    modrinth {
        projectId = 'examplemod'
        accessToken = providers.environmentVariable('MODRINTH_TOKEN')
        minecraftVersions.add('26.3')
    }

    curseforge {
        projectId = '123456'
        accessToken = providers.environmentVariable('CURSEFORGE_TOKEN')
        minecraftVersions.add('26.3')
    }

    github {
        repository = 'example/examplemod'
        accessToken = providers.environmentVariable('GITHUB_TOKEN')
        commitish = 'main'
    }
}
```

> [!TIP]
> `dryRun = true` (или `dryRun = providers.environmentVariable('CI').isPresent()`) печатает, что будет загружено, и не трогает площадки. Это дешевле, чем удалять ошибочный релиз.

### 6.4. CurseGradle — не для новых проектов

`matthewprenger/CurseGradle` — MIT, но репозиторий **архивирован в 2021**. Для 26.x используйте `mod-publish-plugin` (умеет CurseForge) или Minotaur + отдельную загрузку. CurseGradle оставьте только для legacy-проектов, где он уже работает.

### 6.5. GitHub Actions

Матрица Java соответствует поддерживаемым версиям игры: 17 — для 1.17–1.20.4, 21 — для 1.20.5–1.21.11, 25 — для 26.x. Не собирайте 26.x на Java 17 «для совместимости»: `UnsupportedClassVersionError` получите ещё до запуска игры.

> [!IMPORTANT]
> Матрица `[17, 21, 25]` оправдана только для проекта, который реально собирает несколько version-линий. Для одного мода под 26.3 оставьте один JDK 25: `./gradlew build` на JDK 17 не сможет скомпилировать код с `--release 25` и упадёт ещё до тестов. Матрица ниже — шаблон, а не обязательная конфигурация.

Ниже — пример для Fabric-проекта; для NeoForge замените `runDatagen` на `runClientData`, `remapJar` на `jar`, а `modrinth` — на задачу выбранного плагина публикации.

```yaml
name: build

on:
  push:
    branches: [main]
  pull_request:
  release:
    types: [published]

permissions:
  contents: read

concurrency:
  group: build-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        java: [17, 21, 25]

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK ${{ matrix.java }}
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: ${{ matrix.java }}

      - name: Set up Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Build
        run: ./gradlew build --no-daemon --stacktrace

      - name: Verify datagen is up to date
        if: matrix.java == 25
        run: |
          # Fabric: runDatagen. NeoForge: runClientData (или runData, если так назван run config)
          ./gradlew runDatagen --no-daemon
          git diff --exit-code -- src/generated/resources src/main/generated

      - name: Upload artifacts
        if: matrix.java == 25
        uses: actions/upload-artifact@v4
        with:
          name: mod-jar
          path: build/libs/*.jar
          if-no-files-found: error

  publish:
    if: github.event_name == 'release'
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 25

      - uses: gradle/actions/setup-gradle@v4

      - name: Publish to Modrinth
        env:
          MODRINTH_TOKEN: ${{ secrets.MODRINTH_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        # Если используете mod-publish-plugin, замените на ./gradlew publishMods
        run: ./gradlew modrinth --no-daemon
```

Что важно в этом workflow:

- `gradle/actions/setup-gradle` кэширует Gradle-дистрибутив и зависимости; отдельный `actions/cache` для `~/.gradle` больше не нужен.
- `fail-fast: false` — если падает Java 17, вы всё равно видите результат 25.
- Проверка datagen через `git diff --exit-code` ловит забытый `runDatagen` до релиза.
- `permissions: contents: read` по умолчанию; запись даётся только job'у публикации.
- Секреты не печатаются; `GITHUB_TOKEN` выдаётся автоматически, но для сторонних площадок нужны свои токены.

### 6.6. Подпись jar

Для модов подпись не обязательна: Modrinth и CurseForge проверяют хэши, а не PGP. Подпись нужна, если вы публикуете библиотеку в Maven Central или распространяете артефакты через собственный репозиторий.

```groovy
plugins {
    id 'signing'
}

signing {
    useInMemoryPgpKeys(
        findProperty('signingKey'),
        findProperty('signingPassword')
    )
    // Fabric: remapJar — то, что реально уходит игроку.
    // NeoForge 26.x: jar (реобфускации больше нет).
    sign remapJar
}
```

В CI:

```yaml
- name: Publish signed artifacts
  env:
    ORG_GRADLE_PROJECT_signingKey: ${{ secrets.SIGNING_KEY }}
    ORG_GRADLE_PROJECT_signingPassword: ${{ secrets.SIGNING_PASSWORD }}
  run: ./gradlew publish --no-daemon
```

Проверка:

```bash
jarsigner -verify -verbose build/libs/examplemod-1.0.0.jar
```

> [!WARNING]
> Не коммитьте приватный ключ и не передавайте его в аргументах командной строки — он попадёт в логи. Только secrets и in-memory keys.

### 6.7. JitPack

JitPack собирает проект из GitHub-тега и отдаёт Maven-артефакт. Это удобно для API-модов и библиотек, но **не** для распространения игровых модов.

`jitpack.yml`:

```yaml
jdk:
  - openjdk25
install:
  - ./gradlew publishToMavenLocal -x test
```

Потребитель добавляет:

```groovy
repositories {
    maven { url 'https://jitpack.io' }
}
dependencies {
    modImplementation 'com.github.example:examplemod:1.0.0'
}
```

Ограничения: сборка запускается по требованию (первый запрос медленный), нет гарантий воспроизводимости, не поддерживает сложные multi-project сборки без настройки, и это не замена Modrinth/CurseForge.

---

## 7. Версионирование мода

### 7.1. Схема версии

Для мода используйте SemVer: `MAJOR.MINOR.PATCH`, где MAJOR — несовместимые изменения API/контента, MINOR — новый контент, PATCH — исправления. Версию Minecraft лучше выносить в build metadata или в отдельное поле, а не в `version`:

```properties
# gradle.properties
mod_version=1.4.0
minecraft_version=26.3
minecraft_version_range=[26.3,26.4)
neo_version=26.3.0.23-beta
```

```groovy
version = "${mod_version}+mc${minecraft_version}"
```

`minecraft_version_range` — не «магическая» переменная: Gradle сам её никуда не подставляет. В MDK-шаблоне диапазон либо прописан в `neoforge.mods.toml` руками, либо подставляется через `processResources`. Если шаблон этого не делает, синхронизируйте `gradle.properties` и TOML вручную или добавьте подстановку:

```groovy
processResources {
    inputs.property 'minecraft_version_range', minecraft_version_range
    filesMatching('META-INF/neoforge.mods.toml') {
        expand 'minecraft_version_range': minecraft_version_range
    }
}
```

Для Fabric тот же диапазон в `fabric.mod.json` записывается в синтаксисе Loader (`~26.3`, `>=26.3 <26.4`), а не в Maven-синтаксисе — это два разных формата, и общей переменной тут не обойтись.

Modrinth принимает `1.4.0+mc26.3` как version number, но в идеале version number — чистый SemVer, а совместимость с игрой — в `gameVersions`.

### 7.2. `fabric.mod.json`

```json
{
  "schemaVersion": 1,
  "id": "examplemod",
  "version": "${version}",
  "name": "Example Mod",
  "environment": "*",
  "depends": {
    "fabricloader": ">=0.16.0",
    "minecraft": "~26.3",
    "java": ">=25",
    "fabric-api": "*"
  },
  "recommends": {
    "sodium": "*"
  },
  "breaks": {
    "old-examplemod": "<1.0.0"
  }
}
```

> [!NOTE]
> Числа в `depends` — пример, а не «правильные версии для 26.3». Минимальную версию Loader/Fabric API берите из релизов на момент вашей сборки: завышенный минимум отсечёт игроков, заниженный — даст `NoSuchMethodError`.

Синтаксис диапазонов Fabric Loader:

| Запись | Значение |
| --- | --- |
| `>=1.20.1` | 1.20.1 и новее |
| `~26.3` | 26.3.x, но не 26.4 |
| `^1.20` | 1.20.x (совместимые) |
| `26.3.x` | любая patch-версия 26.3 |
| `>=26.3 <26.5` | явный интервал |
| `*` | любая версия |

> [!NOTE]
> `fabric-api` лучше указывать конкретнее, если вы используете новый API: `>=0.120.0` вместо `*`. `*` означает «любая», и игрок с древней версией получит `NoSuchMethodError` вместо внятной ошибки зависимостей.

### 7.3. `neoforge.mods.toml`

```toml
modLoader = "javafml"
loaderVersion = "[4,)"
license = "MIT"
issueTrackerURL = "https://github.com/example/examplemod/issues"

[[mods]]
modId = "examplemod"
version = "${file.jarVersion}"
displayName = "Example Mod"
description = "Adds rubies."
logoFile = "examplemod.png"

[[mixins]]
config = "examplemod.mixins.json"

[[dependencies.examplemod]]
modId = "neoforge"
type = "required"
versionRange = "[26.3.0,)"
ordering = "NONE"
side = "BOTH"

[[dependencies.examplemod]]
modId = "minecraft"
type = "required"
versionRange = "[26.3,26.4)"
ordering = "NONE"
side = "BOTH"

[[dependencies.examplemod]]
modId = "examplelib"
type = "optional"
versionRange = "[1.2,2.0)"
ordering = "AFTER"
side = "BOTH"
```

`version = "${file.jarVersion}"` берётся из manifest `Implementation-Version`; MDG-шаблон подставляет туда `mod_version`. Если не хотите зависеть от этого, укажите версию явно и синхронизируйте с `gradle.properties`.

Диапазоны NeoForge — Maven-style: `[26.3,26.4)`, `[26.3.0,)`, `(,26.4]`. `type` — `required`, `optional`, `incompatible`, `discouraged`; `ordering` — `NONE`, `BEFORE`, `AFTER`; `side` — `BOTH`, `CLIENT`, `SERVER`.

### 7.4. Политика совместимости

- **Один jar — один minor Minecraft.** 26.3 и 26.4 — разные миры; не обещайте совместимость «на всех 26.x».
- **Не расширяйте диапазон без тестов.** `[26.1,26.4)` выглядит красиво, но если вы не запускали 26.2, это ложь.
- **Фиксируйте версии зависимостей.** `fabric-api` и `neoforge` меняются; `*` и открытые диапазоны скрывают поломки.
- **Changelog — часть версии.** Игрок и модпак-разработчик должны видеть, что изменилось и какая минимальная версия игры.

---

## 8. Чеклист релиза

### Метаданные

- [ ] Указана лицензия в `LICENSE`, `fabric.mod.json`/`neoforge.mods.toml` и на площадке.
- [ ] Иконка 128×128 или 512×512 PNG, не GIF, с прозрачностью или без — по требованиям площадки.
- [ ] Описание без обещаний «работает везде», с указанием версии Minecraft и лоадера.
- [ ] Ссылки: исходники, issue tracker, Discord/Matrix, Modrinth/CurseForge.
- [ ] Указаны обязательные и опциональные зависимости.
- [ ] Версия в `gradle.properties`, `fabric.mod.json`/`neoforge.mods.toml` и теге git совпадает.

### Контент

- [ ] `CHANGELOG.md` содержит «Added / Changed / Fixed / Removed» и известные ограничения.
- [ ] README объясняет установку, зависимости и совместимость.
- [ ] Галерея/скриншоты отражают текущую версию, а не 1.20.1.
- [ ] Указано, нужен ли мод на клиенте, сервере или с обеих сторон.

### Код и данные

- [ ] `./gradlew clean build` проходит без предупреждений о deprecated API.
- [ ] `./gradlew runDatagen` (Fabric) или `runClientData`/`runData` (NeoForge) не даёт diff.
- [ ] `./gradlew check` и GameTest (если есть) зелёные.
- [ ] Нет `-SNAPSHOT`, `dev`-классификаторов и локальных путей в jar.
- [ ] Нет client-only классов в общем коде; сервер запускается без `NoClassDefFoundError`.
- [ ] Mixin-конфиг и refmap (если нужен) попали в jar; `mixin.debug.export` не включён.
- [ ] Ресурсы и данные лежат в правильных namespace; нет `examplemod:item/../`.
- [ ] `pack.mcmeta` соответствует версии игры.

### Тесты

- [ ] Чистый клиент: только лоадер + мод (+ обязательные зависимости).
- [ ] Чистый сервер: `java -jar server.jar --nogui`, мод грузится, мир создаётся.
- [ ] Клиент + Sodium/Embeddium, Iris/Oculus — нет конфликтов рендера.
- [ ] Модпак-сборка с 50+ модами — нет дублирующихся миксинов.
- [ ] Java-версия соответствует игре (17/21/25).
- [ ] Проверены `latest.log` и `crash-reports/` на missing textures/models, mixin errors, registry errors.
- [ ] Проверены рецепты в JEI/REI, теги через `/tag`, loot через `/loot`.

### Публикация

- [ ] Тег `v1.4.0` создан и запушен.
- [ ] Артефакты приложены к GitHub Release (jar + sources jar).
- [ ] Modrinth/CurseForge: версия, changelog, game versions, loaders, dependencies.
- [ ] Проверена ссылка на скачивание с площадки, а не только локальный файл.
- [ ] Анонс содержит версию, лоадер, ссылку и предупреждения о несовместимости.

### Матрица smoke-тестов

| Сценарий | Клиент | Сервер | Что проверяем |
| --- | --- | --- | --- |
| Vanilla + loader + mod | ✅ | ✅ | Загрузка, регистрация, краш |
| + обязательные зависимости | ✅ | ✅ | Версии, classloading |
| + Sodium/Iris | ✅ | — | Рендер, mixin-конфликты |
| + 50 модов | ✅ | ✅ | Дубли реестров, теги, рецепты |
| Старый мир | ✅ | ✅ | Data fixer, missing entries |
| Новый мир | ✅ | ✅ | Генерация, loot, advancements |

> [!CAUTION]
> «Работает у меня в dev-окружении» — не тест. Dev-клиент грузит мод из классов и видит больше, чем production-jar: другие имена, другой classloader, другие ресурсы. Релиз проверяется на **собранном jar в чистом профиле**.

---

## Проверено на

| Компонент | Версия | Дата проверки |
| --- | --- | --- |
| NeoForge | 26.3.0.23-beta (MDK, `GatherDataEvent.Client`) | 2026-11-21 |
| Fabric Loader / Fabric API | Loader под 26.3; Fabric API — актуальная для 26.3 | 2026-11-21 |
| Fabric Loom | 1.14+ (plugin IDs `fabric-loom`, `fabric-loom-remap`) | 2026-11-21 |
| ModDevGradle | 2.x | 2026-11-21 |
| Gradle | 9.2+ | 2026-11-21 |
| Java | 25 (26.x), 21 (1.20.5–1.21.11), 17 (1.17–1.20.4) | 2026-11-21 |
| Minotaur | 2.x | 2026-11-21 |
| mod-publish-plugin | 2.2.1 | 2026-11-21 |
| Blockbench | актуальная на 2026-11 | 2026-11-21 |
| Pack format | resource 97.1, data 121.0 (26.3) | 2026-11-21 |

## Источники

- NeoForge Docs — Data Generation: <https://docs.neoforged.net/docs/resources/>
- NeoForge Docs — Data Maps: <https://docs.neoforged.net/docs/resources/server/datamaps/>
- NeoForge (исходники `GatherDataEvent`): <https://github.com/neoforged/NeoForge>
- Fabric Docs — Data Generation Setup (26.3): <https://docs.fabricmc.net/develop/data-generation/setup>
- Fabric Docs — Item Models: <https://docs.fabricmc.net/develop/items/item-models>
- Fabric Docs — Item Appearance / tint sources: <https://docs.fabricmc.net/develop/items/item-appearance>
- Fabric Docs — Loom, plugin IDs: <https://docs.fabricmc.net/develop/loom/>
- Minecraft Wiki — `pack.mcmeta`: <https://minecraft.wiki/w/Pack.mcmeta>
- Minecraft Wiki — Pack format: <https://minecraft.wiki/w/Pack_format>
- Minecraft Wiki — Data values: <https://minecraft.wiki/w/Data_values>
- Modrinth Minotaur: <https://github.com/modrinth/minotaur>
- mod-publish-plugin: <https://github.com/modmuss50/mod-publish-plugin>
- CurseGradle (archived): <https://github.com/matthewprenger/CurseGradle>
- Blockbench: <https://www.blockbench.net/>
- Animated Java: <https://github.com/Animated-Java/animated-java>
- KubeJS: <https://github.com/kube-mods/kubejs>
- CraftTweaker: <https://github.com/CraftTweaker/CraftTweaker>
- GroovyScript: <https://github.com/CleanroomMC/GroovyScript>
- NBT Studio: <https://github.com/tryashtar/nbt-studio>
- Amulet: <https://www.amuletmc.com/>
- WorldEdit: <https://github.com/EngineHub/WorldEdit>
- Misode: <https://misode.github.io/>
- mcasset.cloud: <https://mcasset.cloud/>
- minecraft-schemas: <https://github.com/misode/minecraft-schemas>
- GitHub Actions — `setup-java`: <https://github.com/actions/setup-java>
- Gradle Actions: <https://github.com/gradle/actions>
- JitPack docs: <https://docs.jitpack.io/>
