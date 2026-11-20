# 1.20.x: Data Components и разделение NeoForge/Forge

[← К оглавлению](00-index.md)

> [!NOTE]
> **TL;DR.** 1.20.2 — точка, после которой Forge и NeoForge живут отдельно, а 1.20.5 — второй большой водораздел после 1.17: NBT предмета заменён типизированными **Data Components**, реестры стали datapack-driven, а `ItemStack#getTag()` исчез. Если ваш мод для 1.20.1 хранит данные в `CompoundTag` предмета, при порте на 1.20.5+ переписывать придётся не миксин, а модель данных.

## Содержание

- [Хронология 1.20.x](#хронология-120x)
- [NeoForge/Forge: где именно разошлись](#neoforgeforge-где-именно-разошлись)
- [Data Components: зачем и как](#data-components-зачем-и-как)
  - [Три карты и один патч](#три-карты-и-один-патч)
  - [Что стало с `ItemStack#getTag()`](#что-стало-с-itemstackgettag)
  - [Свои компоненты](#свои-компоненты)
  - [Практические грабли](#практические-грабли)
- [Datapack-driven реестры](#datapack-driven-реестры)
  - [`HolderLookup.Provider`](#holderlookupprovider)
  - [`RegistrySetBuilder`, `RegistryBuilder`, `BootstrapContext`](#registrysetbuilder-registrybuilder-bootstrapcontext)
  - [Порядок загрузки](#порядок-загрузки)
- [1.20.5: Holder-эффекты и остальные Holder-ы](#1205-holder-эффекты-и-остальные-holder-ы)
- [1.20.5: рецепты на codec-ах](#1205-рецепты-на-codec-ах)
- [1.20.5: `ChatFormatting`, `Component`, `StreamCodec`](#1205-chatformatting-component-streamcodec)
- [1.20: `GuiGraphics` и новые методы `Screen`](#120-guigraphics-и-новые-методы-screen)
- [Таблица «было → стало»](#таблица-было--стало)
- [Влияние на миксины](#влияние-на-миксины)
- [Чек-лист порта 1.20.1 → 1.20.6](#чек-лист-порта-1201--1206)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Хронология 1.20.x

| Версия | Дата | Java | Что важно для моддинга |
| --- | --- | --- | --- |
| 1.20 | 2023-06-07 | 17 | `GuiGraphics` вместо `GuiComponent`; `LootData`; переписана светотехника |
| 1.20.1 | 2023-06-12 | 17 | «Последняя общая» версия Forge/NeoForge; до сих пор самая населённая в экосистеме |
| 1.20.2 | 2023-09-21 | 17 | Точка расхождения Forge и NeoForge; сетевой протокол переписан на `Configuration` phase |
| 1.20.4 | 2023-12-07 | 17 | Стабильная 1.20-ветка; сюда целится большинство «1.20.1-совместимых» сборок |
| 1.20.5 | 2024-04-23 | **21** | **Data Components**; `Holder`-ы; codec-и; Java 21 |
| 1.20.6 | 2024-04-29 | 21 | Хотфикс 1.20.5; API-поверхность та же |

Даты взяты с Minecraft Wiki. Для нас важны две границы: **1.20.2** (лоадеры) и **1.20.5** (данные). Между ними есть 1.20.3/1.20.4 — это одна и та же API-линия; 1.20.4 вышел через три дня после 1.20.3 как хотфикс.

> [!IMPORTANT]
> Не путайте «версию игры» и «версию лоадера». NeoForge для 1.20.1 имеет версии вида `47.1.x`, а для 1.20.2 — `20.2.x`. Это разные линии, и мод, собранный против `47.x`, не загрузится на `20.2.x` без пересборки.

---

## NeoForge/Forge: где именно разошлись

История короткая, но её постоянно путают.

1. После 1.20.1 внутри Forge произошёл конфликт вокруг управления проектом и темпа приёма патчей. Часть мейнтейнеров ушла и основала **NeoForged**.
2. NeoForge сначала выпустил сборки для **1.20.1** (`47.1.x`) — то есть какое-то время существовали *два* загрузчика для одной версии игры.
3. Начиная с **1.20.2** NeoForge стал основной площадкой «фордж-семейства»: новые версии игры получал в первую очередь он, а Forge ещё выпускал отдельные сборки, но де-факто центром разработки и большинством модов остался 1.20.1.
4. Практическое следствие: «мод для 1.20.1» — это чаще всего Forge; «мод для 1.20.2+» — почти всегда NeoForge.

Что это значит для кода:

- **Пакеты.** NeoForge переименовал `net.minecraftforge.*` в `net.neoforged.neoforge.*` (и `net.neoforged.bus.api.*` для шины событий). Старые импорты не компилируются.
- **События.** Шина событий переехала, часть событий разделена на game/server/client-шины; `@SubscribeEvent` остался, но регистрация другая.
- **Реестры.** `DeferredRegister` остался, но ключи реестров переехали в `NeoForgeRegistries.Keys`; `RegistryObject` заменён на `DeferredHolder` (в 1.20.2+).
- **Access Transformers.** Формат AT остался, но конфигурация в Gradle другая (MDG/NG вместо FG).
- **Coremod-ы.** NeoForge с самого начала делал ставку на Mixin вместо `IClassTransformer`; для 1.20.1-Forge-модов с coremod-ами это отдельная миграция.

> [!NOTE]
> Для порта 1.20.1 → 1.20.2+ одного переименования импортов недостаточно: меняется инициализация мода, регистрация событий и способ подключения миксинов. Поэтому в литературе 1.20.1 и 1.20.2+ часто описывают как «разные платформы», хотя игра одна.

### Сеть в 1.20.2

1.20.2 добавила фазу **Configuration** в протокол: клиент и сервер обмениваются `Configuration`-пакетами до входа в мир. Для модов это означало:

- `NetworkRegistry`/`SimpleChannel`-подобные API пришлось переписать;
- пакеты, которые раньше регистрировались «на старте», теперь регистрируются в configuration-фазе;
- `FriendlyByteBuf`-сериализация постепенно уступает место `StreamCodec` (окончательно — в 1.20.5).

Если вы видите краш «payload not registered» на 1.20.2+, причина обычно в том, что канал не зарегистрирован в нужной фазе, а не в самом пакете.

---

## Data Components: зачем и как

До 1.20.5 предмет хранил произвольный `CompoundTag`:

```java
// 1.20.4 и раньше
ItemStack stack = new ItemStack(Items.DIAMOND_SWORD);
CompoundTag tag = stack.getOrCreateTag();
tag.putInt("Mana", 42);
tag.putString("Owner", "Steve");
```

Проблемы этого подхода:

- **Нет типов.** `getInt("Mana")` вернёт `0`, если ключа нет, и никто не подскажет, что вы опечатались в имени.
- **Нет валидации.** В тег можно положить что угодно, включая `null`-подобные структуры.
- **Нет сети.** Синхронизация тега — ручная; легко получить рассинхрон клиента и сервера.
- **Нет предсказуемости.** Два мода могут писать в один и тот же ключ.

Data Components решают это: каждый компонент — типизированная пара «ключ → значение», с `Codec` для диска и `StreamCodec` для сети.

### Три карты и один патч

В 1.20.5 появились четыре сущности, которые надо различать:

| Тип | Роль | Аналогия |
| --- | --- | --- |
| `DataComponentType<T>` | Ключ + тип значения | колонка в таблице |
| `DataComponentMap` | Неизменяемая карта «тип → значение» | снапшот строки |
| `PatchedDataComponentMap` | Изменяемая карта с патчами | рабочая копия строки |
| `DataComponentPatch` | Набор изменений относительно дефолта | diff |

`ItemStack` реализует `DataComponentHolder`; сам холдер — read-only, а мутирующие методы (`set`, `update`, `remove`) живут на `ItemStack`.

```java
// 1.20.5+
ItemStack stack = new ItemStack(Items.DIAMOND_SWORD);

// Записать компонент
stack.set(DataComponents.CUSTOM_NAME, Component.literal("Excalibur"));

// Прочитать с дефолтом
int damage = stack.getOrDefault(DataComponents.DAMAGE, 0);
int maxDamage = stack.getOrDefault(DataComponents.MAX_DAMAGE, 0);

// Обновить существующее значение
stack.update(
    DataComponents.CUSTOM_NAME,
    Component.empty(),
    name -> name.copy().withStyle(ChatFormatting.AQUA)
);

// Удалить
stack.remove(DataComponents.CUSTOM_NAME);
```

Дефолтные значения компонентов задаются на предмете:

```java
public static final Item MANA_CRYSTAL = new Item(
    new Item.Properties()
        .stacksTo(16)
        .component(DataComponents.RARITY, Rarity.RARE)
        .component(DataComponents.ENCHANTMENT_GLINT_OVERRIDE, true)
);
```

`Item.Properties#component` — это то, что раньше делали через `Item#verifyTagAfterLoad` и ручные проверки. Значения компонентов должны быть **иммутабельными**: `hashCode`/`equals` считаются по ключам и значениям.

### Что стало с `ItemStack#getTag()`

Метод **удалён**. Не помечен `@Deprecated`, не «спрятан» — его просто нет. Вместе с ним ушли:

- `getOrCreateTag()`
- `setTag(CompoundTag)`
- `hasTag()`
- `getTagElement(String)`
- `removeTagKey(String)`
- `tagMatches(...)`

Всё, что не удалось превратить в типизированный компонент, переехало в компонент `minecraft:custom_data` (`DataComponents.CUSTOM_DATA`, тип `CustomData`).

```java
// 1.20.4
CompoundTag tag = stack.getTag();
if (tag != null && tag.contains("Mana")) {
    int mana = tag.getInt("Mana");
}

// 1.20.5+: читаем
CustomData custom = stack.get(DataComponents.CUSTOM_DATA);
if (custom != null) {
    CompoundTag tag = custom.copyTag();
    if (tag.contains("Mana")) {
        int mana = tag.getInt("Mana");
    }
}

// 1.20.5+: пишем
CustomData.update(DataComponents.CUSTOM_DATA, stack, tag -> tag.putInt("Mana", 5));
```

Важные детали:

- `CustomData.EMPTY` — пустой маркер; `stack.getOrDefault(DataComponents.CUSTOM_DATA, CustomData.EMPTY)` безопасен.
- `copyTag()` возвращает **копию**; менять её и ожидать, что стак обновится, нельзя.
- `CustomData#getUnsafe()` отдаёт внутренний тег; использовать только для чтения и только если понимаете последствия.
- `hasTag()` заменяется на `stack.has(DataComponents.CUSTOM_DATA)` (если интересует именно custom data) или на `!stack.getComponents().isEmpty()` (если интересует «есть хоть что-то»).

> [!WARNING]
> `!stack.getComponents().isEmpty()` почти всегда `true`: у любого предмета есть дефолтные компоненты (`max_stack_size`, `rarity` и т.д.). Если нужен именно «пользовательский тег», проверяйте `CUSTOM_DATA`.

### Свои компоненты

Компонент регистрируется как обычный объект `BuiltInRegistries.DATA_COMPONENT_TYPE`.

```java
public record Mana(int current, int max) {
    public static final Codec<Mana> CODEC = RecordCodecBuilder.create(instance -> instance.group(
        Codec.INT.fieldOf("current").forGetter(Mana::current),
        Codec.INT.fieldOf("max").forGetter(Mana::max)
    ).apply(instance, Mana::new));

    public static final StreamCodec<ByteBuf, Mana> STREAM_CODEC = StreamCodec.composite(
        ByteBufCodecs.VAR_INT, Mana::current,
        ByteBufCodecs.VAR_INT, Mana::max,
        Mana::new
    );
}

public static final DataComponentType<Mana> MANA = DataComponentType.<Mana>builder()
    // пишется в NBT/JSON предмета
    .persistent(Mana.CODEC)
    // синхронизируется на клиент
    .networkSynchronized(Mana.STREAM_CODEC)
    // кешировать закодированное значение: полезно, если компонент часто пишется
    .cacheEncoding()
    .build();

// Регистрация (vanilla-стиль; на NeoForge используйте DeferredRegister)
Registry.register(
    BuiltInRegistries.DATA_COMPONENT_TYPE,
    new ResourceLocation("examplemod", "mana"),
    MANA
);
```

Правила:

- `persistent` и `networkSynchronized` независимы. Компонент только с `persistent` не дойдёт до клиента; только с `networkSynchronized` не сохранится.
- `cacheEncoding` имеет смысл для компонентов, которые часто сериализуются (например, в лут-таблицах), и не нужен для «редких» данных.
- `DataComponentType` должен быть зарегистрирован **до** того, как предметы начнут его использовать.

Использование:

```java
stack.set(MANA, new Mana(10, 10));
Mana mana = stack.getOrDefault(MANA, new Mana(0, 10));
```

### Практические грабли

1. **Компонент — не тег.** Нельзя «дописать поле» в существующий компонент. `Mana(current, max)` — это новое значение целиком; `stack.set(MANA, mana.withCurrent(5))`.
2. **`ItemStack` сравнивается по компонентам.** Два стака с разными `custom_name` — разные стаки. Это ломает старые хаки, где имя использовалось как «скрытый ID».
3. **Компоненты влияют на крафт.** Ингредиент с компонентами не совпадёт с ингредиентом без них, если рецепт не разрешает различия.
4. **`DataComponentPatch` не равен `DataComponentMap`.** Патч хранит только отличия; `ItemStack#getComponentsPatch()` вернёт именно патч.
5. **Порядок регистрации.** Если компонент используется в `Item.Properties` при создании предмета, он должен быть зарегистрирован раньше. На практике — статическая инициализация в правильном порядке или `DeferredRegister`.

---

## Datapack-driven реестры

В 1.20.5 окончательно оформилась модель, где значительная часть игровых данных — не поля классов, а **элементы реестров, загружаемые из datapack-а**.

К datapack-реестрам относятся, в частности:

- `worldgen/*` — биомы, фичи, структуры, carver-ы, плотности;
- `damage_type`, `enchantment` (с 1.21), `jukebox_song`, `painting_variant`;
- `trim_material`, `trim_pattern`;
- лут-таблицы, предикаты, item modifiers — с 1.20.5 они тоже стали reloadable-реестрами.

Ключевое отличие от статических реестров (`Registries.ITEM`, `Registries.BLOCK`): элементы datapack-реестра нельзя получить по `ResourceLocation` без доступа к загруженным данным. Отсюда `HolderLookup.Provider`.

### `HolderLookup.Provider`

`HolderLookup.Provider` — это «снимок всех реестров», который передают туда, где нужен доступ к datapack-данным.

```java
// На сервере
HolderLookup.Provider registries = server.registryAccess();

// Получить lookup конкретного реестра
HolderLookup.RegistryLookup<DamageType> damageTypes =
    registries.lookupOrThrow(Registries.DAMAGE_TYPE);

// Получить Holder по ключу
Holder<DamageType> fire = damageTypes.getOrThrow(DamageTypes.IN_FIRE);
```

Иерархия (упрощённо):

```text
HolderLookup.Provider
  └── lookupOrThrow(ResourceKey<Registry<T>>) -> HolderLookup.RegistryLookup<T>
        ├── get(ResourceKey<T>)      -> Optional<Holder.Reference<T>>
        ├── getOrThrow(ResourceKey<T>) -> Holder.Reference<T>
        └── listElements()           -> Stream<Holder.Reference<T>>
```

`Holder<T>` — это ссылка на элемент реестра, которая может существовать до инициализации самого элемента. Именно поэтому в 1.20.5 методы начали принимать `Holder<MobEffect>` вместо `MobEffect`.

Практические следствия:

- В datagen используйте `BootstrapContext#lookup` (не `registryAccess`).
- В игровом коде — `Level#registryAccess()` или `MinecraftServer#registryAccess()`.
- Не сохраняйте `HolderLookup.Provider` в статическое поле: после `/reload` он устареет.
- `RegistryOps` нужен, чтобы `Codec` умел разрешать ссылки на реестры при чтении JSON/NBT.

```java
// Чтение с разрешением реестровых ссылок
RegistryOps<JsonElement> ops = registries.createSerializationContext(JsonOps.INSTANCE);
DataResult<MyData> result = MyData.CODEC.parse(ops, json);
```

### `RegistrySetBuilder`, `RegistryBuilder`, `BootstrapContext`

Три класса, которые постоянно путают.

**`RegistrySetBuilder`** — строитель набора datapack-реестров для datagen и для начальной загрузки. В него добавляют реестр и функцию-bootstrap:

```java
RegistrySetBuilder builder = new RegistrySetBuilder()
    .add(Registries.CONFIGURED_FEATURE, ModFeatures::bootstrap)
    .add(Registries.PLACED_FEATURE, ModPlacements::bootstrap);

// В datagen: собрать набор и отдать его генератору
// (точная сигнатура build зависит от версии; см. исходники)
var built = builder.build(registries);
```

**`BootstrapContext<T>`** — контекст, который получает bootstrap-функция. В 1.20.5 опечатка `BootstapContext` была исправлена на `BootstrapContext` — это ломает импорты, но не логику.

```java
public static void bootstrap(BootstrapContext<ConfiguredFeature<?, ?>> context) {
    HolderGetter<ConfiguredFeature<?, ?>> features = context.lookup(Registries.CONFIGURED_FEATURE);
    context.register(
        ResourceKey.create(Registries.CONFIGURED_FEATURE, new ResourceLocation("examplemod", "my_feature")),
        new ConfiguredFeature<>(Feature.NO_OP, NoneFeatureConfiguration.INSTANCE)
    );
}
```

**`RegistryBuilder`** (`net.minecraft.core.RegistryBuilder`) — низкоуровневый строитель самого объекта `Registry`/`RegistryAccess`. Он используется внутри загрузки реестров, а не в обычном мод-коде. Сигнатуры и набор методов менялись между 1.20.5 и 1.21, поэтому примеры из интернета часто не компилируются. `[unverified]` — сверяйтесь с исходниками своей версии.

> [!TIP]
> Если вам нужен «реестр для своих данных», почти всегда правильный выбор — datapack-реестр через `RegistryDataLoader`/`DataPackRegistryEvent` (NeoForge) или `RegistryBuilder` только если вы действительно собираете `RegistryAccess` вручную.

### Порядок загрузки

Упрощённая последовательность на сервере:

1. Загружаются **статические** реестры (`BuiltInRegistries`): предметы, блоки, компоненты, звуки.
2. Загружаются **datapack-реестры** из `data/<ns>/<registry>/...`.
3. Строится `HolderLookup.Provider`.
4. Инициализируются компоненты предметов, зависящие от datapack-данных (в 26.1 это вынесено в `DataComponentInitializers`, но предпосылки появились уже здесь).
5. Только после этого можно безопасно создавать объекты, которым нужны реестровые ссылки.

Именно поэтому в 1.20.5 появились методы вида `RecipeProvider(CompletableFuture<HolderLookup.Provider>)` и `LootTableSubProvider#generate(HolderLookup.Provider, ...)`: datagen тоже должен видеть те же реестры, что и игра.

---

## 1.20.5: Holder-эффекты и остальные Holder-ы

Самое массовое механическое изменение 1.20.5: методы, принимавшие объект реестра, начали принимать `Holder` этого объекта.

| Было (1.20.4) | Стало (1.20.5) |
| --- | --- |
| `MobEffectInstance(MobEffect, int, int)` | `MobEffectInstance(Holder<MobEffect>, int, int)` |
| `MobEffectInstance#getEffect()` → `MobEffect` | → `Holder<MobEffect>` |
| `MobEffect#addAttributeModifier(..., Attribute, ...)` | `Holder<Attribute>` |
| `ArmorItem(..., ArmorMaterial, ...)` | `Holder<ArmorMaterial>` |
| `ChatType.Bound(ChatType, ...)` | `Holder<ChatType>` |
| `Entity#gameEvent(GameEvent, ...)` | `Holder<GameEvent>` |
| `FogRenderer.MobEffectFogFunction#getMobEffect()` | `Holder<MobEffect>` |

Код обычно меняется минимально, потому что константы (`MobEffects.SPEED`, `Attributes.ARMOR`) уже имеют тип `Holder`:

```java
// 1.20.5
MobEffectInstance speed = new MobEffectInstance(MobEffects.SPEED, 200, 1);

// Сравнение
if (instance.is(MobEffects.SPEED)) { ... }        // появился в 1.20.5
Holder<MobEffect> effect = instance.getEffect();
```

Что ломается:

- методы, возвращавшие `MobEffect` и использовавшие его как ключ в `Map<MobEffect, ...>`;
- `@Redirect`/`@ModifyVariable` на методах, где поменялся дескриптор (`Lnet/minecraft/world/effect/MobEffect;` → `Lnet/minecraft/core/Holder;`);
- сериализация эффектов в NBT: теперь пишется ID реестра, а не прямой объект.

---

## 1.20.5: рецепты на codec-ах

До 1.20.5 рецепт имел `RecipeSerializer` с методами `fromJson`/`toJson`/`fromNetwork`/`toNetwork`. В 1.20.5 это заменили на codec-и:

```java
// 1.20.5+
public class ExampleRecipe implements Recipe<CraftingContainer> {
    public static final MapCodec<ExampleRecipe> CODEC = ...;
    public static final StreamCodec<RegistryFriendlyByteBuf, ExampleRecipe> STREAM_CODEC = ...;

    @Override
    public RecipeSerializer<?> getSerializer() {
        return ModRecipes.EXAMPLE_SERIALIZER.get();
    }
}

public static final RecipeSerializer<ExampleRecipe> EXAMPLE_SERIALIZER =
    new RecipeSerializer<>(ExampleRecipe.CODEC, ExampleRecipe.STREAM_CODEC);
```

Дополнительно:

- `Codec#dispatch` теперь принимает `MapCodec`, а не `Codec`.
- `RecipeSerializer#codec()` возвращает `MapCodec`.
- `RecipeSerializer#fromNetwork`/`toNetwork` удалены.
- `RecipeManager` хранит рецепты по `RecipeHolder`/`ResourceKey`.
- В 1.20.5 рецепт всё ещё принимает `Container`; `RecipeInput` появится в 1.21.

> [!NOTE]
> В 1.20.5 рецепты ещё не стали datapack-реестром в полном смысле — это произошло в 26.1 (reloadable registries, см. `6-unobfuscated.md`). Но направление уже видно: JSON → codec → реестр.

---

## 1.20.5: `ChatFormatting`, `Component`, `StreamCodec`

`Component` в 1.20.5 перестал быть «особым случаем» для сети:

```java
// Было
FriendlyByteBuf buf = ...;
Component text = buf.readComponent();

// Стало
Component text = ComponentSerialization.STREAM_CODEC.decode(buf);
ComponentSerialization.STREAM_CODEC.encode(buf, text);
```

Что ещё поменялось:

- `Component#translationArg` для подстановок.
- `Style` стал ещё более центральным: цвет и форматирование задаются через `Style#withColor`/`withBold` и т.д.
- `ChatFormatting` в 1.20.5 **ещё существует** и реализует `StringRepresentable`. Его окончательно разберут только в 26.2 (см. `6-unobfuscated.md`), поэтому не спешите переписывать `ChatFormatting.GOLD` на `TextColor` в 1.20.x-моде.
- `Component#getString()` остаётся, но для пользовательского текста лучше `getString()` не использовать как «сырой текст» — он теряет стили.

Мелочи, которые ломают миксины:

- `Component` стал ещё чаще передаваться как `Component`, а не `BaseComponent`/`MutableComponent` (последние — подтипы, но дескрипторы в миксинах конкретны).
- `ChatType.Bound` теперь `Holder<ChatType>`; методы, строившие `Bound`, поменяли сигнатуры.
- `GuiMessage`/`ChatComponent` получили новые методы (`storeState`/`restoreState`), что важно для миксинов в чат.

---

## 1.20: `GuiGraphics` и новые методы `Screen`

1.20 убрала `GuiComponent` и ввела `GuiGraphics` — контекст отрисовки GUI, который несёт в себе `PoseStack`, буферы, scissor-стек и размеры экрана.

```java
// 1.19.4
@Override
public void render(PoseStack pose, int mouseX, int mouseY, float partialTick) {
    super.render(pose, mouseX, mouseY, partialTick);
    GuiComponent.fill(pose, 0, 0, 10, 10, 0xFFFF0000);
}

// 1.20
@Override
public void render(GuiGraphics graphics, int mouseX, int mouseY, float partialTick) {
    super.render(graphics, mouseX, mouseY, partialTick);
    graphics.fill(0, 0, 10, 10, 0xFFFF0000);
    graphics.drawString(this.font, "Hi", 5, 5, 0xFFFFFF);
}
```

Ключевые изменения:

- `PoseStack` больше не передаётся в GUI-методы; он доступен через `GuiGraphics#pose()`.
- `GuiGraphics#drawString`, `#blit`, `#fill`, `#renderItem`, `#renderTooltip` — основной API.
- Z-координата GUI ограничена `[-10000, 10000]`.
- `GuiGraphics#drawManaged` изолирует draw-call (flush до и после).
- `ItemRenderer#renderGuiItem` переехал в `GuiGraphics#renderItem`.
- `Screen#renderBackground` в 1.20.1 всё ещё один метод; в 1.20.5 его разбили на `renderPanorama`, `renderBlurredBackground`, `renderMenuBackground` и `renderTransparentBackground`, а `renderDirtBackground` переименовали в `renderMenuBackground`.

Для `AbstractContainerScreen`:

```java
// 1.20.x
@Override
protected void renderBg(GuiGraphics graphics, float partialTick, int mouseX, int mouseY) {
    graphics.blit(TEXTURE, this.leftPos, this.topPos, 0, 0, this.imageWidth, this.imageHeight);
}

@Override
protected void renderLabels(GuiGraphics graphics, int mouseX, int mouseY) {
    graphics.drawString(this.font, this.title, this.titleLabelX, this.titleLabelY, 0x404040, false);
}
```

Это ещё не «extract»-архитектура 26.1, но уже шаг к ней: рендер отделён от состояния экрана, а `GuiGraphics` стал единственной точкой входа.

---

## Таблица «было → стало»

### Данные предмета

| 1.20.4 | 1.20.5+ |
| --- | --- |
| `stack.getTag()` | `stack.get(DataComponents.CUSTOM_DATA)` → `CustomData#copyTag()` |
| `stack.getOrCreateTag()` | `CustomData.update(DataComponents.CUSTOM_DATA, stack, tag -> ...)` |
| `stack.setTag(tag)` | `stack.set(DataComponents.CUSTOM_DATA, CustomData.of(tag))` |
| `stack.hasTag()` | `stack.has(DataComponents.CUSTOM_DATA)` |
| `stack.getTagElement("display")` | отдельные компоненты (`CUSTOM_NAME`, `LORE`, …) |
| `stack.getDamageValue()` / `setDamageValue()` | `DataComponents.DAMAGE` |
| `stack.getOrCreateTag().putBoolean("Unbreakable", true)` | `DataComponents.UNBREAKABLE` (`Unit`) |
| `stack.addEnchantment(...)` | `DataComponents.ENCHANTMENTS` (`ItemEnchantments`) |
| `stack.getEnchantmentTags()` | `stack.getOrDefault(DataComponents.ENCHANTMENTS, ItemEnchantments.EMPTY)` |
| `ItemStack#isSame` / `tagMatches` | `ItemStack.isSameItemSameComponents(...)` |
| `ItemStack#sameItem` | `ItemStack#isSameItem` (1.20) |

### Реестры и рецепты

| 1.20.4 | 1.20.5+ |
| --- | --- |
| `RecipeSerializer#fromJson` / `toJson` | `MapCodec` в `RecipeSerializer` |
| `RecipeSerializer#fromNetwork` / `toNetwork` | `StreamCodec<RegistryFriendlyByteBuf, T>` |
| `Codec#dispatch(..., Codec)` | `Codec#dispatch(..., MapCodec)` |
| `LootDataManager` | `MinecraftServer#reloadableRegistries` |
| `BootstapContext` | `BootstrapContext` |
| `ExtraCodecs.strictOptionalField` | `Codec#optionalFieldOf` |
| `ExtraCodecs.validate` | `Codec#validate` |
| `ExtraCodecs.withAlternative` | `Codec#withAlternative` |

### Эффекты, чат, GUI

| 1.20.4 | 1.20.5+ |
| --- | --- |
| `MobEffectInstance(MobEffect, ...)` | `MobEffectInstance(Holder<MobEffect>, ...)` |
| `MobEffectInstance#getEffect()` → `MobEffect` | → `Holder<MobEffect>` |
| `MobEffect#addAttributeModifier(..., Attribute, ...)` | `Holder<Attribute>` |
| `FriendlyByteBuf#readComponent` | `ComponentSerialization.STREAM_CODEC` |
| `Screen#renderDirtBackground` | `Screen#renderMenuBackground` (1.20.5) |
| `Screen#renderBackground` (один метод) | `renderPanorama` + `renderBlurredBackground` + `renderMenuBackground` + `renderTransparentBackground` |
| `GuiComponent.fill(...)` | `GuiGraphics#fill(...)` |
| `ItemRenderer#renderGuiItem` | `GuiGraphics#renderItem` |

---

## Влияние на миксины

### 1. Миксин на `ItemStack#getTag` просто не применится

```java
// Плохо: на 1.20.5+ такого метода нет
@Mixin(ItemStack.class)
public abstract class OldTagMixin {
    @Inject(method = "getTag", at = @At("HEAD"), cancellable = true)
    private void example$oldTag(CallbackInfoReturnable<CompoundTag> cir) {
        // ...
    }
}
```

Симптом: `InvalidInjectionException: Failed to find target method` или `Mixin apply failed`. Лечение — не «поправить дескриптор», а перенести логику на компонент:

```java
@Mixin(ItemStack.class)
public abstract class ManaMixin {
    @Inject(method = "getHoverName", at = @At("HEAD"), cancellable = true)
    private void example$name(CallbackInfoReturnable<Component> cir) {
        // ...
    }
}
```

### 2. `@Shadow` на поле `tag` невозможен

Поля `ItemStack.tag` больше нет. Любой миксин, который делал:

```java
@Shadow @Final private CompoundTag tag;
```

перестаёт компилироваться/применяться. Замена — `@Shadow` на методы работы с компонентами или `@Accessor`:

```java
@Mixin(ItemStack.class)
public interface ItemStackAccessor {
    @Accessor("components")
    PatchedDataComponentMap example$components();
}
```

### 3. Дескрипторы `Holder` ломают `@Redirect`/`@ModifyVariable`

Если инъекция была привязана к типу `MobEffect`, а метод теперь принимает `Holder`, нужно менять и сигнатуру инъекции:

```java
// 1.20.4
@Redirect(method = "addEffect", at = @At(value = "INVOKE", target = "Lnet/minecraft/world/effect/MobEffect;isInstantenous()Z"))
private boolean example$instant(MobEffect effect) { ... }

// 1.20.5
@Redirect(method = "addEffect", at = @At(value = "INVOKE", target = "Lnet/minecraft/core/Holder;is(Ljava/lang/Object;)Z"))
private boolean example$holder(Holder<MobEffect> holder, Object other) { ... }
```

Точный target лучше смотреть в байткоде (`javap -c`), а не угадывать: `Holder` — интерфейс, и вызов может быть `Holder#value()` или `Holder#is`.

### 4. `@Overwrite` на `appendHoverText`

В 1.20.5 сигнатура тултипа изменилась:

```java
// 1.20.4
public void appendHoverText(ItemStack stack, @Nullable Level level, List<Component> tooltip, TooltipFlag flag)

// 1.20.5
public void appendHoverText(ItemStack stack, Item.TooltipContext context, List<Component> tooltip, TooltipFlag flag)
```

`@Overwrite` слой ломается целиком. Правильнее — `@Inject(at = TAIL)` и добавлять строки, а не заменять метод.

### 5. Миксины в GUI

С 1.20 все GUI-методы принимают `GuiGraphics`:

- `@Inject(method = "render", desc = "(Lnet/minecraft/client/gui/GuiGraphics;IIF)V", ...)` — дескриптор другой;
- `PoseStack` больше не параметр, поэтому `@ModifyVariable` на `PoseStack` не сработает;
- `Screen#renderBackground` в 1.20.5 разбит на несколько методов — инъекция в «старый» метод может попасть не туда.

### 6. Миксины в реестры

Если мод миксинился в `Registry`/`MappedRegistry` (например, чтобы «подсунуть» элемент), в 1.20.5+ это особенно хрупко: реестры стали `Holder`-ориентированными, а datapack-реестры перезагружаются. Лучше использовать публичные API лоадера (`DeferredRegister`, `RegistryDataLoader`-хуки), а не патчить внутренности.

---

## Чек-лист порта 1.20.1 → 1.20.6

1. **Лоадер.** Решите, Forge 1.20.1 или NeoForge 1.20.2+. Это разные импорты, события и Gradle-плагины.
2. **Java.** 1.20.5+ требует Java 21. Обновите `toolchain` и `compatibilityLevel` миксинов до `JAVA_21`.
3. **NBT.** Найдите все `getTag`/`getOrCreateTag`/`hasTag` и замените на компоненты. Начните с `CUSTOM_DATA` как «переходника», но не оставляйте его навсегда.
4. **Дефолты.** Перенесите `Item#verifyTagAfterLoad` и ручные проверки в `Item.Properties#component`.
5. **Эффекты.** Проверьте `MobEffectInstance`, `MobEffect`, `Attribute` на `Holder`.
6. **Рецепты.** Перепишите `RecipeSerializer` на `MapCodec` + `StreamCodec`.
7. **GUI.** Проверьте дескрипторы `render(GuiGraphics, ...)`, `renderBg`, `renderLabels`, `renderBackground`.
8. **Реестры.** Замените прямые `Registry#get` на `HolderLookup.Provider`/`HolderGetter` там, где речь о datapack-данных.
9. **Миксины.** Пересоберите таргеты: удалённые методы, изменённые дескрипторы, `Holder`-ы.
10. **Тесты.** Прогоните `-Dmixin.debug.export=true` и убедитесь, что все миксины применились, а не «тихо пропустились».

---

## Проверено на

- Дата проверки: **2026-11-21**.
- Версии, по которым сверялись утверждения: Minecraft **1.20 / 1.20.1 / 1.20.2 / 1.20.4 / 1.20.5 / 1.20.6**, NeoForge-праймеры `1.19.4 → 1.20` и `1.20.4 → 1.20.5`.
- Вживую сборка 1.20.5 в рамках этой проверки не запускалась; код-примеры составлены по праймеру и официальным именам. Всё, что помечено `[unverified]`, требует сверки с исходниками вашей версии.
- `RegistryBuilder` в 1.20.5 описан на уровне роли, без привязки к конкретным сигнатурам: они менялись.

## Источники

- NeoForged, *Minecraft 1.19.4 → 1.20 Mod Migration Primer* — <https://docs.neoforged.net/primer/docs/1.20/>
- NeoForged, *Minecraft 1.20.4 → 1.20.5 Mod Migration Primer* — <https://docs.neoforged.net/primer/docs/1.20.5/>
- NeoForged, *Minecraft 1.20.5 → 1.20.6 Mod Migration Primer* — <https://docs.neoforged.net/primer/docs/1.20.6/>
- Minecraft Wiki, *Java Edition 1.20*, *1.20.2*, *1.20.5* (даты, Java-версии) — <https://minecraft.wiki/w/Java_Edition_1.20>
- Minecraft Wiki, *Data component format* — <https://minecraft.wiki/w/Data_component_format>
- NeoForged news, *NeoForge for Minecraft 26.1* (объяснение `ItemStackTemplate`, четырёхкомпонентные версии, NeoForm `26.1-1`) — <https://neoforged.net/news/26.1release/>
- Fabric, *Fabric for Minecraft 26.1* (историческая ретроспектива по mappings и `ItemStackTemplate`) — <https://fabricmc.net/2026/03/14/261.html>
- Fabric Wiki (draft), *Mixins and Obfuscation* — <https://wiki.fabricmc.net/drafts:mixin_obfuscation>
