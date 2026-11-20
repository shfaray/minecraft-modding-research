# 26.1–26.3: деобфускация, Java 25 и календарные версии

[← К оглавлению](00-index.md)

> [!NOTE]
> **TL;DR.** С 26.1 Minecraft поставляется **деобфусцированным**: официальные имена Mojang — это имена в рантайме, параметры и локальные переменные видны, `Yarn`/`intermediary`/`refmap` больше не нужны, а `officialMojangMappings()` в Loom теряет смысл. Одновременно игра перешла на Java 25, версии — на схему `YY.D.H`, а `ItemStack` (и `FluidStack`/`ItemResource` в NeoForge) нельзя создать до загрузки реестров (отсюда `ItemStackTemplate`/`FluidStackTemplate`). Для миксинов это главное упрощение со времён появления Mixin — и одновременно повод переписать половину таргетов, потому что `GuiGraphics` стал `GuiGraphicsExtractor`.

## Содержание

- [Календарное версионирование](#календарное-версионирование)
- [Удаление обфускации в 26.1](#удаление-обфускации-в-261)
  - [Что именно изменилось](#что-именно-изменилось)
  - [Чего деобфускация не даёт](#чего-деобфускация-не-даёт)
  - [Почему `officialMojangMappings()` падает](#почему-officialmojangmappings-падает)
- [Java 25 и требования к IDE](#java-25-и-требования-к-ide)
- [Тулинг: Loom, Gradle, NeoForge, MDG, NeoGradle](#тулинг-loom-gradle-neoforge-mdg-neogradle)
- [`ItemStackTemplate`, `FluidStackTemplate`, `ItemResource`](#itemstacktemplate-fluidstacktemplate-itemresource)
- [Loot type unrolling](#loot-type-unrolling)
- [`GuiGraphics` → `GuiGraphicsExtractor`](#guigraphics--guigraphicsextractor)
- [`ChunkPos`, `MutableQuad` и прочее 26.1](#chunkpos-mutablequad-и-прочее-261)
- [26.2: Vulkan, `BindGroupLayout`, конец `ChatFormatting`](#262-vulkan-bindgrouplayout-конец-chatformatting)
- [26.3: Renderpearl, SDL, OIT](#263-renderpearl-sdl-oit)
- [Что это значит для миксинов](#что-это-значит-для-миксинов)
  - [`remap = false` и не-MC цели](#remap--false-и-не-mc-цели)
  - [`@Local` по имени](#local-по-имени)
  - [Баг с именами параметров до Loader 0.18.4](#баг-с-именами-параметров-до-loader-0184)
  - [Почему refmap теперь warning](#почему-refmap-теперь-warning)
  - [Как читать таргеты без mappings](#как-читать-таргеты-без-mappings)
  - [`compatibilityLevel`, конфиг, AccessWidener](#compatibilitylevel-конфиг-accesswidener)
- [Таблица «до 26.1 / после 26.1»](#таблица-до-261--после-261)
- [Чек-лист порта 1.21.11 → 26.1](#чек-лист-порта-12111--261)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Календарное версионирование

2 декабря 2025 Mojang объявила новую схему: `YY.D.H`, где `YY` — две цифры года, `D` — номер дропа в году, `H` — номер хотфикса. Первый релиз дропа идёт без `H`. Последняя версия старой схемы — **1.21.11**.

С 16 декабря 2025 снапшоты тоже переименованы: вместо `25w41a` — `26.1 Snapshot 1`.

| Версия | Название | Дата | Data version | Resource pack | Data pack | Java | Protocol |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 26.1 | Tiny Takeover | 2026-03-24 | 4786 | 84.0 | 101.1 | 25 | 775 |
| 26.2 | Chaos Cubed | 2026-06-16 | 4903 | 88.0 | 107.1 | 25 | 776 |
| 26.3 | Wilderness Bound | 2026-09-15 | 5023 | 97.1 | 121.0 | 25 | 777 |
| 26.4 | Deep Freeze | Q4 2026, на 2026-11-21 ещё не вышел | — | — | — | 25 | — |

Что это значит на практике:

- **Версия игры и версия лоадера синхронизированы.** NeoForge `26.1.0.1-beta` — это Minecraft `26.1.0` (дроп 26.1, хотфикс 0) + сборка NeoForge `1`, суффикс `-beta`. Четыре компонента вместо трёх.
- **Хотфиксы — это часть версии.** `26.1.1`, `26.1.2` и т.д. — отдельные релизы; мод, собранный под `26.1.0`, обычно работает, но API мог поменяться.
- **Снапшоты стали предсказуемыми.** `26.3 Snapshot 4` — это четвёртый снапшот 26.3, а не «неделя 4».
- **Даты важнее номеров.** Между 26.1 и 26.2 — меньше трёх месяцев; между 26.2 и 26.3 — три месяца. Планировать порт «раз в год» больше нельзя.

> [!WARNING]
> Не путайте `26.1` и `1.21.1`. Это разные версии; в коде и в логах они выглядят похоже, а API — нет. `1.21.1` — обфусцированная, Java 21, `ResourceLocation`. `26.1` — деобфусцированная, Java 25, `Identifier`.

---

## Удаление обфускации в 26.1

Mojang анонсировала это в октябре 2025; Fabric опубликовала разбор 31 октября 2025; первые деобфусцированные снапшоты пошли после 1.21.11, а первым релизом стал 26.1 (24 марта 2026).

### Что именно изменилось

1. **Официальные имена — в рантайме.** Класс `Creeper` — это `net.minecraft.world.entity.monster.Creeper` (а после 1.21.11-переименований — `...monster.creeper.Creeper`), а не `brc`. Нет промежуточного namespace.
2. **Имена параметров и локальных переменных.** Раньше Mojmap давал имена классов/методов/полей, но не параметров; теперь видны и они. Это важно для `@Local(name = ...)`, `@ModifyVariable`, отладки и чтения стектрейсов.
3. **Yarn и intermediary не нужны.** Yarn остаётся только для старых версий; для 26.x его нет. Intermediary (`class_XXXX`, `method_XXXX`, `field_XXXX`) в рантайме не используется.
4. **Refmap не нужен.** Annotation processor Mixin-а больше не генерирует `refmap.json`; Loom не ремапит миксины для необфусцированных версий.
5. **`officialMojangMappings()` теряет смысл.** Официальные mappings — это теперь не отдельный артефакт, а сами имена. Loom-проект, который просит mappings для 26.x, падает или сообщает, что они недоступны.
6. **Моды не были обфусцированы — но были исключения.** Классы модов всегда компилировались в свои имена. Однако *переопределения методов Minecraft внутри классов мода* (например, `@Override public void tick()`) до 26.1 компилировались в обфусцированные имена и требовали ремапа. После 26.1 это обычные Java-имена.

### Чего деобфускация не даёт

- **Константы остаются встроенными.** `static final int`/`String` могут быть заинлайнены компилятором; в байткоде не будет ссылки на поле. Это не «недостаток деобфускации», а свойство Java.
- **Synthetic/bridge/lambda-имена остаются синтетическими.** `lambda$foo$0`, `access$100`, мостовые методы — это не «настоящие» имена API, и они могут меняться от версии к версии.
- **Локальные переменные могут отсутствовать.** Если класс собран без debug-информации, `LocalVariableTable` пуст. У Mojang она есть, но у сторонних библиотек — не всегда.
- **Имена не равны стабильности.** Mojang может переименовать метод между 26.1 и 26.2; деобфускация убирает *маппинг*, а не *изменения API*.
- **Это не «открытый исходник».** Javadoc-ов и комментариев нет; декомпилятор всё ещё нужен.

### Почему `officialMojangMappings()` падает

В Loom есть два режима:

- `net.fabricmc.fabric-loom-remap` (и старый `fabric-loom`) — для обфусцированных версий: скачивает mappings, ремапит игру и моды.
- `net.fabricmc.fabric-loom` — для необфусцированных: **ничего не ремапит**.

Для 26.x официальных mappings-артефактов не существует, поэтому вызов `officialMojangMappings()` в remap-режиме не может ничего разрешить. Точный текст исключения зависит от версии Loom (`MappingsNotFoundException`, «Could not resolve mappings», «no mappings for version» и т.п.) — проверяйте свой билд. `[unverified]` — конкретный класс исключения не сверялся.

Правильная миграция:

```groovy
// Было (1.21.11)
plugins {
    id 'fabric-loom' version '1.12-SNAPSHOT'
}

dependencies {
    minecraft "com.mojang:minecraft:1.21.11"
    mappings loom.officialMojangMappings()
    modImplementation "net.fabricmc:fabric-loader:0.16.14"
}

// Стало (26.1+)
plugins {
    id 'net.fabricmc.fabric-loom' version '1.15-SNAPSHOT'
}

dependencies {
    minecraft "com.mojang:minecraft:26.1"
    // mappings не нужны
    implementation "net.fabricmc:fabric-loader:0.18.4"
}
```

И в задачах:

```groovy
// Было
remapJar { ... }
modImplementation "..."
modCompileOnly "..."

// Стало
jar { ... }
implementation "..."
compileOnly "..."
```

> [!IMPORTANT]
> `modImplementation`/`remapJar` — это не «устаревшие алиасы». В необфусцированном Loom они не имеют смысла, потому что ремапить нечего. Если оставить их, сборка либо упадёт, либо соберёт jar с неверными зависимостями.

---

## Java 25 и требования к IDE

26.1 требует **Java 25** и для запуска, и для Gradle JVM.

```groovy
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(25)
    }
}
```

Ваниль использует новые возможности языка. Главная — **Flexible Constructor Bodies**: операторы до `super(...)`.

- JEP 447 — preview (Java 22).
- JEP 482/492 — последующие preview.
- **JEP 513** — финализация в Java 25.

Для миксинов это важно косвенно: конструкторы ванильных классов могут содержать код до `super`, а байткод выглядит иначе, чем в Java 21. Если вы инжектитесь в `<init>`, проверяйте байткод, а не декомпилятор.

IDE:

| IDE | Минимум | Комментарий |
| --- | --- | --- |
| IntelliJ IDEA | 2025.2 | Минимум для Java 25 (NeoForge primer) |
| IntelliJ IDEA | 2025.3 | Минимум, чтобы корректно работали миксины (Fabric blog) |
| Eclipse | 2025-12 | Либо 2025-09 + плагин Java 25 Support |
| Gradle | 9.1+ (MDG/NG), 9.2+ (Loom 1.14), 9.4+ (Loom 1.16) | Зависит от плагина |

> [!NOTE]
> «IntelliJ 2025.2» и «2025.3» — не противоречие: 2025.2 достаточно для компиляции Java 25, но для миксинов Fabric рекомендует 2025.3. Если миксины ведут себя странно (не видят параметры, ломается `@Local`), первым делом обновите IDE.

---

## Тулинг: Loom, Gradle, NeoForge, MDG, NeoGradle

### Loom

| Loom | Дата | Ключевое |
| --- | --- | --- |
| 1.12 | 2025-10-29 | Legacy Mixin AP выключен по умолчанию; refmap перестаёт генерироваться |
| 1.13 | 2025-11-04 | `migrateMappings` с поддержкой миксинов (MercuryMixin) |
| 1.14 | 2025-12-04 | **Новые plugin ID**; начальная поддержка необфусцированных версий; Gradle 9.2 |
| 1.15 | 2026-01-22 | `loom.defaultMixinRemapType`; прямой ремап official → named без intermediary; удалён FernFlower |
| 1.16 | 2026-04-20 | Минимум Gradle 9.4; enum extensions; ремап локальных переменных |
| 1.17 | 2026-06-07 | Gradle 9.5; новый RunConfiguration API |
| 1.18 | 2026-09-20 | Gradle task run configs; Java 25 FFM вместо нативных библиотек |
| 1.19 | 2026-10-23 | Поддержка mixin lambda selectors в tiny-remapper |

На 2026-11-21 актуальна **Loom 1.19**. Для 26.1 на момент релиза рекомендовали 1.15 + Gradle 9.4; сейчас разумно брать 1.19 и Gradle 9.5+.

Plugin ID:

```groovy
// 26.x (необфусцированные)
plugins {
    id 'net.fabricmc.fabric-loom' version '1.19-SNAPSHOT'
}

// 1.21.11 и раньше (обфусцированные)
plugins {
    id 'net.fabricmc.fabric-loom-remap' version '1.19-SNAPSHOT'
}

// Старый ID — только для обратной совместимости
plugins {
    id 'fabric-loom' version '...'
}
```

### NeoForge

- Версии — **четырёхкомпонентные**: `26.1.0.1-beta` = MC `26.1.0` + NeoForge release `1` + `-beta`.
- На релизе 26.1 актуальна `26.1.0.1-beta`.
- **MDG 2.0.141**, **NeoGradle 7.1.21** (на март 2026).
- **Gradle 9.1.0+** для MDG/NG.
- **NeoForm** версии `26.1-1` (формат `<mc>-<build>`).
- **Parchment можно удалить**: имена параметров теперь приходят с игрой. Оставить имеет смысл только ради javadoc-ов.

```groovy
// MDG 2.x, ориентировочно
neoForge {
    version = '26.1.0.1-beta'
    // ...
}
```

> [!NOTE]
> Точный DSL MDG 2.x отличается от 1.x (`loadedMods` вместо `mods`, `neoForge.ideSyncTask` и т.д.). См. `01-toolchain/1-build-systems.md`; здесь важен только факт смены версии и требований к Gradle.

---

## `ItemStackTemplate`, `FluidStackTemplate`, `ItemResource`

Главное изменение данных в 26.1: **`ItemStack` нельзя создать до загрузки реестров**.

Причина: компоненты предмета могут ссылаться на datapack-реестры (зачарования, зелья, инструменты, лут-таблицы). Эти реестры загружаются позже статических, а в 26.1 компоненты ещё и инициализируются на reload (`DataComponentInitializers`). Значит, `new ItemStack(Items.DIAMOND_SWORD)` в статическом инициализаторе рецепта — ошибка.

Важно: в ваниле появился только `ItemStackTemplate`. `FluidStackTemplate`, `ItemResource` и `FluidResource` — зеркальные типы **NeoForge**; в ванильном коде их нет.

Решение — `ItemStackTemplate`: неизменяемый «черновик» стака.

```java
public record ItemStackTemplate(
    Holder<Item> item,
    int count,
    DataComponentPatch components
) {
    // ...
}
```

Создание:

```java
ItemStackTemplate apple = new ItemStackTemplate(
    Items.APPLE.builtInRegistryHolder(),
    5,
    DataComponentPatch.builder()
        .set(DataComponents.ITEM_NAME, Component.literal("Apple?"))
        .build()
);

// В стек — только когда реестры загружены
ItemStack stack = apple.create();

// Из непустого стека — в шаблон
ItemStackTemplate template = ItemStackTemplate.fromNonEmptyStack(stack);

// Короткий конструктор без компонентов
ItemStackTemplate simple = new ItemStackTemplate(Items.APPLE);
```

Свойства:

- `ItemStackTemplate` **не может быть пустым**. Если нужен «возможно пустой» шаблон — используйте `@Nullable ItemStackTemplate`.
- `ItemInstance` — общий интерфейс для `ItemStack` и `ItemStackTemplate`: даёт `typeHolder()`, `count()`, `get(DataComponentType)`, `is(...)`.
- `CODEC`, `MAP_CODEC`, `STREAM_CODEC` — для JSON/NBT и сети.
- `create()` и `apply(...)` — превращение в стек (с добавлением компонентов или без).

Где шаблоны заменили стеки:

- рецепты (`ShapedRecipe`, `ShapelessRecipe`, `SmeltingRecipe`, `SmithingTransformRecipe`, `TransmuteRecipe`, …);
- `RecipeBuilder` и `Recipe#getResultItem`;
- advancements (`DisplayInfo#icon`, `Advancement.Builder#display`);
- `HoverEvent.ShowItem`;
- `ItemParticleOption`, `BreakingItemParticle`;
- `BundleContents`;
- `Item#getCraftingRemainder`;
- `UseRemainder`;
- villager trades (см. ниже);
- `SlotDisplay.ItemStackSlotDisplay`.

Проверка «тот же предмет и те же компоненты»:

```java
// 26.1+
boolean same = ItemStack.isSameItemSameComponents(stack, template);
boolean resourceMatches = resource.matches(template);
```

### `ItemResource` (NeoForge)

`net.neoforged.neoforge.transfer.item.ItemResource` — иммутабельная комбинация `Item` + data components **без количества** из транспортного API NeoForge. Он тоже требует загруженных реестров, поэтому в data-файлах вместо него используется `ItemStackTemplate`.

Типичное применение — там, где количество не важно: ингредиенты, фильтры, проверки «этот предмет с такими компонентами». Не пытайтесь создать `ItemResource` в статическом инициализаторе; передавайте его из загруженного контекста. Аналогично устроен `FluidResource` для жидкостей.

### `FluidStackTemplate`

NeoForge зеркалит ванильную модель на жидкости:

- `FluidStack` требует загруженных реестров.
- `FluidResource` — аналог `ItemResource`.
- `FluidStackTemplate` — иммутабельный шаблон (fluid + amount + components).
- У жидкостей появились **дефолтные data components**.

```java
// NeoForge 26.1+
FluidStackTemplate template = new FluidStackTemplate(
    ModFluids.EXAMPLE.get(),
    1000,
    DataComponentPatch.EMPTY
);

FluidStack stack = template.create();
```

Практическое следствие: рецепты и datagen, которые раньше хранили `FluidStack`, должны перейти на `FluidStackTemplate`. Старый код с `new FluidStack(...)` в статическом поле компилируется, но падает при загрузке.

> [!WARNING]
> Если вы видите `NullPointerException`/`IllegalStateException` «registry not loaded» при инициализации рецепта — почти наверняка вы создаёте `ItemStack`/`FluidStack` слишком рано. Замените на шаблон и создавайте стек в `create()`/`assemble()`.

---

## Loot type unrolling

26.1 убрала промежуточные `*Type`-классы в лут-системе. Раньше каждый loot-объект имел «тип» — отдельный зарегистрированный объект, хранящий codec. Теперь реестр хранит **сам `MapCodec`**.

```java
// Было: LootItemFunctionType
public record NoopItemFunction() implements LootItemFunction {
    public static final LootItemFunctionType NOOP = new LootItemFunctionType(...);

    @Override
    public LootItemFunctionType getType() {
        return NOOP;
    }
}

// Стало: MapCodec как реестровый объект
public record NoopItemFunction() implements LootItemFunction {
    public static final NoopItemFunction INSTANCE = new NoopItemFunction();
    public static final MapCodec<NoopItemFunction> MAP_CODEC = MapCodec.unit(INSTANCE);

    @Override
    public MapCodec<NoopItemFunction> codec() {
        return MAP_CODEC;
    }
}

Registry.register(
    BuiltInRegistries.LOOT_FUNCTION_TYPE,
    Identifier.fromNamespaceAndPath("examplemod", "noop"),
    NoopItemFunction.MAP_CODEC
);
```

Что удалено/переименовано:

| Было | Стало |
| --- | --- |
| `LootPoolEntryType` | нет; реестр хранит `MapCodec<LootPoolEntryContainer>` |
| `LootItemFunctionType` | нет; `MapCodec<LootItemFunction>` |
| `LootItemConditionType` | нет; `MapCodec<LootItemCondition>` |
| `LootNumberProviderType` | нет; `MapCodec<NumberProvider>` |
| `LootNbtProviderType` | нет; `MapCodec<NbtProvider>` |
| `LootScoreProviderType` | нет; `MapCodec<ScoreboardNameProvider>` |
| `FloatProviderType` | нет; `MapCodec<FloatProvider>` |
| `IntProviderType` | нет; `MapCodec<IntProvider>` |
| `getType()` | `codec()` |
| `CODEC` (в провайдерах) | `MAP_CODEC` |

Дополнительно:

- `FloatProvider`/`IntProvider` стали интерфейсами; их реализации — record-ы.
- `getMinValue()`/`getMaxValue()` → `min()`/`max()` (float) и `minInclusive()`/`maxInclusive()` (int).
- `LootPoolEntries`, `LootItemFunctions`, `LootItemConditions` и т.п. больше не держат singleton-поля; используйте codec-ы и `bootstrap`.
- `NbtProvider`/`ScoreProvider` теперь реализуют `LootContextUser` (и `Validatable`).

Для модов это означает: любой кастомный loot-функция/условие/провайдер требует переписать регистрацию и убрать `*Type`.

---

## `GuiGraphics` → `GuiGraphicsExtractor`

26.1 переименовала GUI-слой в терминах «extract»: экран больше не рисует, а **извлекает render state** в дерево, которое потом исполняет рендерер.

| 1.21.11 | 26.1 |
| --- | --- |
| `GuiGraphics` | `GuiGraphicsExtractor` |
| `Screen#render` | `Screen#extractRenderState` |
| `Screen#renderBackground` | `Screen#extractBackground` |
| `AbstractContainerScreen#renderBg` | `Screen#extractBackground` |
| `AbstractContainerScreen#renderLabels` | `AbstractContainerScreen#extractLabels` |
| `AbstractWidget#renderWidget` | `AbstractWidget#extractWidgetRenderState` |
| `Renderable#render` | `Renderable#extractRenderState` |
| `GuiGraphics#renderOutline` | `GuiGraphicsExtractor#outline` |
| `GuiGraphics#submitEntityRenderState` | `GuiGraphicsExtractor#entity` |
| `GuiGraphics#hline` | `GuiGraphicsExtractor#horizontalLine` |
| `*String*` методы | `*Text*` |

Правила переименования `GuiGraphics`:

- убираются префиксы `draw*`, `render*`, `submit*`;
- убираются суффиксы `*RenderState`;
- `String` в имени заменяется на `Text`.

Пример:

```java
// 1.21.11
@Override
public void render(GuiGraphics graphics, int mouseX, int mouseY, float partialTick) {
    super.render(graphics, mouseX, mouseY, partialTick);
    graphics.drawString(this.font, this.title, 8, 8, 0xFFFFFF);
    graphics.renderOutline(this.leftPos, this.topPos, this.imageWidth, this.imageHeight, 0xFF000000);
}

// 26.1
@Override
public void extractRenderState(GuiGraphicsExtractor graphics, int mouseX, int mouseY, float partialTick) {
    super.extractRenderState(graphics, mouseX, mouseY, partialTick);
    graphics.text(this.font, this.title, 8, 8, 0xFFFFFF);
    graphics.outline(this.leftPos, this.topPos, this.imageWidth, this.imageHeight, 0xFF000000);
}
```

Важно: **это не механическое переименование**. «Extract» означает, что метод наполняет `GuiRenderState`/дерево, а не рисует в буфер. Если ваш миксин полагался на побочный эффект «после render всё уже на экране», он сломается.

---

## `ChunkPos`, `MutableQuad` и прочее 26.1

### `ChunkPos`

`ChunkPos` стал record-ом; конструкторы заменены на фабрики:

```java
// Было
ChunkPos pos = new ChunkPos(blockPos);
long packed = ChunkPos.asLong(blockPos);
ChunkPos unpacked = new ChunkPos(packed);

// Стало
ChunkPos pos = ChunkPos.containing(blockPos);
long packed = ChunkPos.pack(blockPos);
ChunkPos unpacked = ChunkPos.unpack(packed);
```

Также появились `getRegionX()`/`getRegionZ()` — регион, в котором находится чанк.

### `MutableQuad` (NeoForge)

`MutableQuad` — изменяемая альтернатива `BakedQuad` для построения и правки квадов.

Методы, которые стоит знать:

| Метод | Что делает |
| --- | --- |
| `setCubeFaceFromSpriteCoords` | строит позиции грани по 2D-координатам спрайта |
| `setCubeFace` | строит грань 3D-куба по его размеру |
| `bakeUvsFromPosition` | генерирует UV так, как это делает ваниль для block model |
| `recalculateWinding` | переставляет вершины под ambient occlusion ванили |
| `setSpriteAndMoveUv` | меняет спрайт и пересчитывает UV в атласе |

Типичное применение — динамическая геометрия (block entity, custom renderer), где `BakedQuad` слишком жёсткий.

### Прочее 26.1

- **`DataComponentInitializers`.** Компоненты привязываются к `Holder`-у и инициализируются на reload; `Item.Properties#delayedComponent`/`delayedHolderComponent` откладывают значение до загрузки реестров. `Holder#components()` отдаёт `DataComponentMap`.
- **Villager trades стали datapack-реестром.** `data/<ns>/villager_trade/*.json`, `data/<ns>/trade_set/*.json`, теги `villager_trade`. `VillagerTrades.ItemListing` больше нет; есть `VillagerTrade` + `TradeSet`.
- **Validation overhaul.** `Validatable#validate(ValidationContext)`, `ProblemReporter`, `ValidationContextSource`; `CriterionValidator` удалён.
- **`Level#random` стал `protected`.** Используйте `getRandom()`.
- **`Recipe#assemble` больше не принимает `HolderLookup.Provider`.** Рецепт должен хранить всё нужное внутри.
- **`RegistryDataLoader#load` возвращает `CompletableFuture`** замороженного `RegistryAccess`.
- **`RecipeSerializer`** — просто `MapCodec` + `StreamCodec`; внутренние классы-сериализаторы не нужны.
- **`ChunkSectionLayer`** выставляется автоматически по свойствам спрайта (transluent/cutout/solid); ручная регистрация слоёв блоков не нужна.

---

## 26.2: Vulkan, `BindGroupLayout`, конец `ChatFormatting`

26.2 (Chaos Cubed, 16 июня 2026) — «рендер снова меняется», но уже более механически.

### Vulkan

- Появился экспериментальный Vulkan-бэкенд; в настройках — «Graphics API» (Default / Prefer Vulkan / Prefer OpenGL).
- Для каждого класса `com.mojang.blaze3d.opengl.*` появился параллельный `com.mojang.blaze3d.vulkan.*`.
- Требование: Vulkan 1.2 с dynamic rendering и push descriptors (может измениться).
- OpenGL остаётся fallback-ом.

Для модов, которые делают прямые GL-вызовы, это сигнал: **пора уходить на Blaze3D-абстракцию**. В 26.3 прямые GL-вызовы станут ещё менее переносимыми.

### Графические типы

- `SourceFactor`/`DestFactor` → единый `BlendFactor`; `BlendOp` определяет операцию.
- `TextureFormat` и форматы `VertexFormatElement` → `GpuFormat` (`R8_UNORM`, `RGBA32_SINT`, `D32_FLOAT_S8_UINT`, …).
- `VertexFormat` собирается динамически: `VertexFormat.Builder#addAttribute`, до 16 элементов.
- `RenderPipeline` может иметь до 8 color target-ов и до 16 vertex buffer-ов.
- `BindGroupLayout` заменил плоские списки samplers/uniforms; `RenderPipeline.Builder#withBindGroupLayout`.
- `RenderPass` может рисовать в подобласть текстуры (`RenderArea`).

```java
public static final BindGroupLayout EXAMPLE_LAYOUT = BindGroupLayout.builder()
    .withSampler("Sampler0")
    .withUniform("Globals", UniformType.UNIFORM_BUFFER)
    .build();

public static final RenderPipeline EXAMPLE_PIPELINE = RenderPipeline.builder()
    .withLocation(Identifier.fromNamespaceAndPath("examplemod", "pipeline/example"))
    .withVertexShader(Identifier.fromNamespaceAndPath("examplemod", "example_shader"))
    .withFragmentShader(Identifier.fromNamespaceAndPath("examplemod", "example_shader"))
    .withVertexBinding(0, DefaultVertexFormat.ENTITY)
    .withPrimitiveTopology(PrimitiveTopology.QUADS)
    .withBindGroupLayout(EXAMPLE_LAYOUT)
    .withBindGroupLayout(BindGroupLayouts.MATRICES_PROJECTION)
    .build();
```

### `ChatFormatting` больше нет

`ChatFormatting` «выпотрошен»: цвет и стиль задаются через `Style`/`TextColor`.

```java
// 1.21.11
Component text = Component.literal("Hi").withStyle(ChatFormatting.GOLD);
int rgb = ChatFormatting.GOLD.getColor();

// 26.2+
Component text = Component.literal("Hi")
    .withStyle(style -> style.withColor(TextColor.fromRgb(0xFFAA00)));
TextColor color = TextColor.parseColor("gold").getOrThrow();
```

`ChatFormatting` перестал быть `StringRepresentable`; `CODEC`/`COLOR_CODEC` заменены на `TextColor#CODEC`; `getId`/`getName`/`getByName`/`getById`/`getNames` удалены или заменены.

### `Gui` и `Hud`

`Gui` стал «владельцем» всего GUI-состояния:

```java
// 1.21.11
Minecraft.getInstance().screen
Minecraft.getInstance().gui.setOverlayMessage(...)

// 26.2+
Minecraft.getInstance().gui.screen()
Minecraft.getInstance().gui.hud.setOverlayMessage(...)
```

`Hud` — отдельный класс для in-game HUD.

### Шрифты

`Font#draw*` удалены. Текст готовится вручную:

```java
Font.PreparedText text = font.prepareText("Hello", 0, 0, 0xFF00FF00, false, 0);
text.visit(new Font.GlyphVisitor() {
    @Override
    public void acceptRenderable(TextRenderable renderable) {
        // ...
    }
});
```

Для большинства модов это не нужно: используйте `OrderedSubmitNodeCollector#submitText`.

### Feature rendering

`MultiBufferSource` перестал быть универсальным путём: `FeatureRenderDispatcher` принимает `SubmitNodeStorage`, а моды могут добавлять свои `SubmitNode`/`FeatureRenderPhase`/`FeatureRenderer`.

---

## 26.3: Renderpearl, SDL, OIT

26.3 (Wilderness Bound, 15 сентября 2026) — самый «инфраструктурный» релиз линии.

### Renderpearl

Рендер-API вынесли из Blaze3D в отдельный проект **Renderpearl**:

```text
com.mojang.renderpearl.api.*       — публичные интерфейсы
com.mojang.renderpearl.backend.*   — OpenGL/Vulkan-реализации
com.mojang.renderpearl.frontend.*  — валидация и реализация API
```

Массовые переезды:

- `com.mojang.blaze3d.buffers.GpuBuffer` → `renderpearl.api.buffers.GpuBuffer`;
- `CommandEncoder` → `renderpearl.frontend.FrontendCommandEncoder`;
- `GpuDevice` → `renderpearl.frontend.FrontendGpuDevice`;
- `RenderPass` → `renderpearl.frontend.FrontendRenderPass`;
- `RenderPipeline` → `renderpearl.api.pipeline.RenderPipeline`;
- `VertexFormat`/`VertexFormatElement` → `renderpearl.api.vertex.*`;
- `GpuTexture`/`GpuTextureView`/`GpuSampler` → `renderpearl.api.textures.*`.

Для модов это означает: все импорты `com.mojang.blaze3d.*`, связанные с GPU, надо менять. Прямые GL-вызовы (`GlStateManager`, `GL11`, …) становятся ещё более хрупкими.

### SDL вместо GLFW

LWJGL теперь использует **SDL**, а не GLFW.

- Коды клавиш — scancode-ы SDL (`InputConstants`, `SDLScancode`).
- `KeyEvent#key` — scancode; `keycode` (бывший `scancode`) — SDL keycode.
- `MouseButtonInfo#button` — `SDL_BUTTON_*`.
- Модификаторы — маска `SDL_KMOD_*`.
- Текстовый ввод требует `SDL_StartTextInput`/`SDL_StopTextInput`; в GUI — `TextInputManager#onTextInputFocusChange`.

```java
@Override
public void setFocused(boolean focused) {
    super.setFocused(focused);
    Minecraft.getInstance().onTextInputFocusChange(this, focused);
}
```

Если у вас кастомное текстовое поле и оно перестало принимать ввод в 26.3 — почти наверняка вы не вызываете `onTextInputFocusChange`.

### OIT

Появилась **Order-Independent Transparency** (Wavelet OIT):

- три фазы: depth bounds, transmittance, accumulate;
- шейдерный define `OIT_ALPHA_ONLY`;
- `OitPipelineSet` собирает три `RenderPipeline`;
- `RenderSetup#setOitPipelines`;
- `RenderType#prepare` + `PreparedRenderType#drawFromBufferOit`.

Для модов с прозрачной геометрией это новая головная боль: нужно либо поддерживать OIT, либо явно жить в «classic»-режиме.

### Шейдеры

- `#moj_import` → `#include`;
- `in`/`out` обязаны иметь `layout(location = N)`;
- `GlslCompiler` определяет `RENDERPEARL_*`-макросы;
- большинство ванильных шейдеров требуют `GL_ARB_separate_shader_objects`.

### Предметы и фазы

- `ItemQuads` — квады предмета, разделённые на solid/translucent.
- `RenderSystem#outputColorTextureOverride`/`outputDepthTextureOverride` удалены; output задаётся через `RenderPass`.
- `PoseStack#mulPose` → `rotate`.
- `Camera#extractRenderState` принимает `DeltaTracker`.
- Палитровая метадата для текстур (`palette` в `.mcmeta`), `PalettedTextureManager`.

---

## Что это значит для миксинов

Это самая важная часть файла. До 26.1 миксин-разработчик жил в мире «имя в исходнике → refmap → intermediary/SRG → рантайм». С 26.1 осталось только «имя в исходнике = имя в рантайме».

### `remap = false` и не-MC цели

На обфусцированных версиях миксин по умолчанию ремапится: имена из аннотаций переводятся в runtime-namespace. На необфусцированных ремапить нечего, поэтому для не-MC целей `remap = false` — ожидаемое поведение (и его можно зафиксировать через `loom.defaultMixinRemapType`).

```java
// 26.1+: цель — ванильный класс, имена совпадают
@Mixin(ItemStack.class)
public abstract class ItemStackMixin {
    @Inject(method = "getDamageValue", at = @At("HEAD"), cancellable = true)
    private void example$damage(CallbackInfoReturnable<Integer> cir) { ... }
}

// 26.1+: цель — класс другого мода (не обфусцирован)
@Mixin(value = OtherModClass.class, remap = false)
public abstract class OtherModMixin { ... }

// 26.1+: цель — ванильный класс, но мод компилировался с remap для совместимости
@Mixin(value = ItemStack.class, remap = false)
public abstract class ExplicitNoRemapMixin { ... }
```

Правила:

- Для **ванильных** целей в 26.x ремап не нужен. Не ставьте `remap = true` «на всякий случай»: в необфусцированной сборке ему нечего ремапить, и поведение зависит от версии Loom (ошибка или предупреждение).
- Для **модовых** целей — `remap = false`.
- Для **переопределений MC-методов внутри модовых классов** до 26.1 требовался `remap = true`; в 26.x это обычные имена.
- Loom 1.15 добавил `loom.defaultMixinRemapType` — можно задать поведение по умолчанию для необфусцированных сборок.

```groovy
// build.gradle
loom {
    defaultMixinRemapType = "false" // или "true", если нужен ремап
}
```

> [!NOTE]
> Точное значение свойства (`"false"`/`"none"`/enum) зависит от версии Loom. Сверяйтесь с релиз-нотами 1.15 и документацией Loom. `[unverified]` — строковый формат не проверялся.

### `@Local` по имени

До 26.1 `@Local(name = "...")` не работал: в обфусцированном байткоде локальные переменные назывались `arg0`, `lvt_1_1_` и т.п. Приходилось использовать тип + `ordinal` или индекс.

С 26.1 имя локальной переменной — настоящее:

```java
@Mixin(ItemStack.class)
public abstract class ExampleMixin {
    @Inject(method = "someMethod", at = @At("HEAD"))
    private void example$capture(@Local(name = "stack") ItemStack stack) {
        // ...
    }
}
```

Для параметров:

```java
@Inject(method = "someMethod", at = @At("HEAD"))
private void example$captureArg(@Local(argsOnly = true, name = "level") Level level) { ... }
```

Что важно:

- Имя должно совпадать с `LocalVariableTable` **точно**, включая регистр.
- Если переменная переименована в новой версии — миксин сломается. Это всё ещё «хрупкая» привязка, просто теперь читаемая.
- Для параметров до Fabric Loader 0.18.4 был баг (см. ниже).

### Баг с именами параметров до Loader 0.18.4

Fabric Mixin долгое время сохранял имена параметров как `arg0`, `arg1`, … вместо настоящих. Исправлено в версии Mixin, поставляемой с **Fabric Loader 0.18.4**.

Симптом:

```text
@Local(name = "stack") did not match any local variable
```

при том, что в исходниках параметр действительно называется `stack`.

Лечение:

- обновите Fabric Loader до **0.18.4+**;
- если обновиться нельзя — используйте `argsOnly = true` + `ordinal`/тип;
- на NeoForge аналогичная проблема зависит от версии Mixin, поставляемой лоадером; проверяйте `-Dmixin.debug.verbose=true`.

> [!IMPORTANT]
> Это не «баг 26.1», а баг тулинга, который стал заметен именно после деобфускации: раньше имён параметров всё равно не было, и все привыкли к `arg0`.

### Почему refmap теперь warning

Refmap — это JSON, который annotation processor генерировал, чтобы «мягкие» имена в аннотациях (`method = "foo"`, `@At(target = "...")`) превратить в runtime-имена.

Что изменилось:

- Loom 1.12 (октябрь 2025) **выключил legacy Mixin AP по умолчанию**.
- Loom 1.14+ для необфусцированных версий вообще не ремапит.
- В 26.x имена в аннотациях уже совпадают с рантаймом.

Следствие: `refmap` в `*.mixins.json` больше не нужен. Если он остался, Loom/Mixin обычно игнорирует его и может выдать warning. Точное поведение зависит от версии Loom; в некоторых сборках warning выглядит как «refmap not found» или «refmap is not required for unobfuscated versions». `[community]`

Правильная миграция:

```json
// 1.21.11
{
  "required": true,
  "package": "com.example.mixin",
  "compatibilityLevel": "JAVA_21",
  "refmap": "examplemod.refmap.json",
  "mixins": ["ItemStackMixin"],
  "client": ["GuiMixin"],
  "injectors": { "defaultRequire": 1 }
}

// 26.1+
{
  "required": true,
  "package": "com.example.mixin",
  "compatibilityLevel": "JAVA_25",
  "mixins": ["ItemStackMixin"],
  "client": ["GuiMixin"],
  "injectors": { "defaultRequire": 1 }
}
```

Также:

- `useLegacyMixinAp = true` больше не нужен; если он включён, AP может пытаться генерировать refmap и мешать.
- `mixin { useLegacyMixinAp = false }` — безопасный дефолт для 26.x (и так дефолт с Loom 1.12).
- Если вы публикуете мод для 1.21.11 и 26.x из одной кодовой базы, держите два mixin-конфига: старый с `refmap`, новый без.

### Как читать таргеты без mappings

До 26.1 маршрут был: Linkie / mappings.dev / Yarn javadoc → имя → refmap. Теперь:

1. **Официальный jar.** Скачайте client.jar из piston-meta, распакуйте, читайте имена напрямую. `javap -p -c -l` покажет методы, дескрипторы и `LocalVariableTable`.
2. **mcsrc.dev.** Онлайн-декомпилятор от Fabric; умеет работать с официальными именами и помогает строить миксины/AW. Указан в блоге Fabric 26.1.
3. **IDE.** Loom/MDG подключают источники; навигация по ванильному коду работает как по обычной библиотеке.
4. **`javap -v`.** Для проверки дескриптора, `ACC_*`-флагов и наличия synthetic-методов.
5. **Логи Mixin.** `-Dmixin.debug.export=true` выгружает пропатченные классы в `.mixin.out/class`; `-Dmixin.debug.verbose=true` печатает, что именно искал Mixin.

Что больше не нужно:

- Linkie для 26.x (он привязан к mappings старых версий);
- mappings.dev для 26.x (он показывает 1.21.11 и раньше);
- refmap;
- `yarn`/`mojmap`-словари.

### `compatibilityLevel`, конфиг, AccessWidener

- `compatibilityLevel` должен соответствовать Java: `JAVA_25` для 26.x. Если оставить `JAVA_21`, Mixin откажется загружаться или будет вести себя непредсказуемо.
- `minVersion` Mixin-а должен поддерживать Java 25; обновите Mixin/Loader вместе.
- **AccessWidener** работает: он не связан с обфускацией. Формат `accessible`/`extendable`/`mutable` тот же.
- **ClassTweaker** (Fabric) работает; Loom 1.16 добавил enum extensions (нужен Loader 0.19.0+).
- **AccessTransformer** (NeoForge) работает; синтаксис тот же.
- **Interface injection** (NeoForge) работает; Loom 1.18 добавил задачу валидации injected interfaces.

---

## Таблица «до 26.1 / после 26.1»

### Имена и сборка

| До 26.1 (1.21.11 и раньше) | После 26.1 |
| --- | --- |
| Yarn / Mojmap / SRG / intermediary | Официальные имена Mojang в рантайме |
| `class_XXXX`, `method_XXXX`, `field_XXXX` | `Creeper`, `tick`, `health` |
| `officialMojangMappings()` | Не нужен; для 26.x не существует |
| `refmap.json` + Mixin AP | Не нужен; AP выключен |
| `fabric-loom` / `fabric-loom-remap` | `net.fabricmc.fabric-loom` (для 26.x) |
| `modImplementation` / `modCompileOnly` | `implementation` / `compileOnly` |
| `remapJar` | `jar` |
| `remap = true` по умолчанию | `remap = false` для не-MC целей |
| `@Local` по типу/ordinal/индексу | `@Local(name = "...")` |
| Java 21 | Java 25 |
| Gradle 8.x/9.0 | Gradle 9.2+ (Loom 1.14), 9.4+ (Loom 1.16) |
| NeoForge `21.1.x` | NeoForge `26.1.0.1-beta` (4 компонента) |
| NeoForm `1.21.11-...` | NeoForm `26.1-1` |
| Parchment для имён параметров | Не нужен (имена приходят с игрой) |

### Данные

| До 26.1 | После 26.1 |
| --- | --- |
| `new ItemStack(...)` где угодно | Только после загрузки реестров |
| `ItemStack` в рецептах/advancement-ах | `ItemStackTemplate` |
| `FluidStack` в рецептах | `FluidStackTemplate` (NeoForge) |
| `ItemResource`/`FluidResource` (NeoForge) без реестров | Требуют загруженных реестров |
| `ItemStack#getTag()` (уже нет с 1.20.5) | `DataComponents` + `CustomData` |
| Компоненты на предмете | Компоненты на `Holder` + `DataComponentInitializers` |
| `LootItemFunctionType` | `MapCodec<LootItemFunction>` |
| `getType()` | `codec()` |
| `LootPoolEntryType` | `MapCodec<LootPoolEntryContainer>` |
| `RecipeSerializer#fromNetwork` | `StreamCodec` |
| `Recipe#assemble(..., HolderLookup.Provider)` | `Recipe#assemble(...)` без provider |
| Villager trades в коде | `data/<ns>/villager_trade`, `trade_set` |

### Рендер и GUI

| До 26.1 | После 26.1 |
| --- | --- |
| `GuiGraphics` | `GuiGraphicsExtractor` |
| `Screen#render` | `Screen#extractRenderState` |
| `Screen#renderBackground` | `Screen#extractBackground` |
| `AbstractContainerScreen#renderBg` | `Screen#extractBackground` |
| `AbstractContainerScreen#renderLabels` | `AbstractContainerScreen#extractLabels` |
| `Renderable#render` | `Renderable#extractRenderState` |
| `AbstractWidget#renderWidget` | `AbstractWidget#extractWidgetRenderState` |
| `GuiGraphics#renderOutline` | `GuiGraphicsExtractor#outline` |
| `GuiGraphics#submitEntityRenderState` | `GuiGraphicsExtractor#entity` |
| `GuiGraphics#hline` | `GuiGraphicsExtractor#horizontalLine` |
| `*String*` | `*Text*` |
| `new ChunkPos(BlockPos)` | `ChunkPos.containing(BlockPos)` |
| `ChunkPos.asLong(...)` | `ChunkPos.pack(...)` |
| `new ChunkPos(long)` | `ChunkPos.unpack(long)` |
| `ResourceLocation` (1.21.11) | `Identifier` |
| `ChatFormatting` (26.2) | `Style` / `TextColor` |
| `com.mojang.blaze3d.*` (26.3) | `com.mojang.renderpearl.*` |
| GLFW (26.3) | SDL |
| `RenderSystem#output*Override` (26.3) | `RenderPass` |
| `PoseStack#mulPose` (26.3) | `PoseStack#rotate` |

---

## Чек-лист порта 1.21.11 → 26.1

1. **Java.** Поставьте JDK 25, `toolchain = 25`, `compatibilityLevel = JAVA_25`, обновите IDE.
2. **Loom.** Перейдите на `net.fabricmc.fabric-loom`; уберите `officialMojangMappings()`; `modImplementation` → `implementation`; `remapJar` → `jar`.
3. **Mixin-конфиг.** Уберите `refmap`; проверьте `compatibilityLevel`; обновите Mixin/Loader (Fabric Loader 0.18.4+).
4. **Имена.** `ResourceLocation` → `Identifier`; проверьте пакеты entity/model (переименования 1.21.11).
5. **GUI.** `GuiGraphics` → `GuiGraphicsExtractor`; `render` → `extractRenderState`; `renderBackground` → `extractBackground`; `renderLabels` → `extractLabels`.
6. **Данные.** `ItemStack` в рецептах/advancement-ах → `ItemStackTemplate`; `FluidStack` → `FluidStackTemplate`; проверьте, что ничего не создаётся до загрузки реестров.
7. **Loot.** Уберите `*Type`; `getType()` → `codec()`; зарегистрируйте `MapCodec`.
8. **Villager trades.** Перенесите в datapack (`villager_trade`, `trade_set`).
9. **Миксины.** Для модовых целей — `remap = false`; для локальных переменных — `@Local(name = "...")`; проверьте, что имена параметров не `arg0`.
10. **Тесты.** `-Dmixin.debug.export=true`, `-Dmixin.debug.verbose=true`; прогоните клиент и сервер; проверьте GUI и рендер предметов.

---

## Проверено на

- Дата проверки: **2026-11-21**.
- Версии: Minecraft **26.1** (2026-03-24), **26.2** (2026-06-16), **26.3** (2026-09-15); **26.4** на дату проверки ещё не вышел (план — Q4 2026).
- Тулинг: NeoForge **26.1.0.1-beta**, MDG **2.0.141**, NeoGradle **7.1.21**, NeoForm **26.1-1**, Loom **1.14–1.19**, Fabric Loader **0.18.4**.
- Утверждения про «refmap warning» и точный текст исключения `officialMojangMappings()` помечены `[community]`/`[unverified]`: поведение зависит от версии Loom.
- Даты, data version, pack format и Java-версии — Minecraft Wiki; версии тулинга — релиз-ноты Loom и новости NeoForged.

## Источники

- Minecraft.net, *Removing obfuscation in Java Edition* (анонс, октябрь 2025) — <https://www.minecraft.net/en-us/article/removing-obfuscation-in-java-edition>
- Minecraft.net, *Minecraft's new version numbering system* (2 декабря 2025) — <https://www.minecraft.net/en-us/article/minecraft-new-version-numbering-system>
- Minecraft Wiki, *Java Edition 26.1* — <https://minecraft.wiki/w/Java_Edition_26.1>
- Minecraft Wiki, *Java Edition 26.2* — <https://minecraft.wiki/w/Java_Edition_26.2>
- Minecraft Wiki, *Java Edition 26.3* — <https://minecraft.wiki/w/Java_Edition_26.3>
- Minecraft Wiki, *Version formats* (схема `YY.D.H`, переименование снапшотов) — <https://minecraft.wiki/w/Version_formats>
- NeoForged, *Minecraft 1.21.11 → 26.1 Mod Migration Primer* — <https://docs.neoforged.net/primer/docs/26.1/>
- NeoForged, *Minecraft 26.1.x → 26.2 Mod Migration Primer* — <https://docs.neoforged.net/primer/docs/26.2/>
- NeoForged, *Minecraft 26.2 → 26.3 Mod Migration Primer* — <https://docs.neoforged.net/primer/docs/26.3/>
- NeoForged news, *NeoForge for Minecraft 26.1* (4-компонентные версии, `ItemStackTemplate`, `FluidStackTemplate`, `MutableQuad`, `ChunkPos`, `GuiGraphicsExtractor`) — <https://neoforged.net/news/26.1release/>
- Fabric, *Fabric for Minecraft 26.1* (Loom 1.15, Gradle 9.4, Loader 0.18.4, IntelliJ 2025.3, `ItemStackTemplate`, `FluidModel`) — <https://fabricmc.net/2026/03/14/261.html>
- Fabric, *Removing Obfuscation from Fabric* (31 октября 2025) — <https://fabricmc.net/2025/10/31/obfuscation.html>
- Fabric Wiki (draft), *Mixins and Obfuscation* (`remap = false`, `@Local(name = ...)`, баг параметров до Loader 0.18.4) — <https://wiki.fabricmc.net/drafts:mixin_obfuscation>
- Fabric Loom releases: 1.14 (plugin IDs, Gradle 9.2), 1.15 (`defaultMixinRemapType`), 1.16 (Gradle 9.4), 1.18, 1.19 — <https://github.com/FabricMC/fabric-loom/releases>
- OpenJDK, *JEP 513: Flexible Constructor Bodies* — <https://openjdk.org/jeps/513>
- OpenJDK, *JEP 447: Statements before super(...)* — <https://openjdk.org/jeps/447>
