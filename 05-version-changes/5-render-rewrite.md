# 1.21.x: как переписывали рендер (до 1.21.11)

[← К оглавлению](00-index.md)

> [!NOTE]
> **TL;DR.** 1.21.x — это не один рефакторинг, а шесть последовательных: `HolderSet` и `EntityRenderState` (1.21.2), `ItemModel` (1.21.4), `RenderPipeline`/`GpuDevice` (1.21.5), `GuiRenderState` (1.21.6), feature-submission (1.21.9) и «rename shuffle» + `GpuSampler` (1.21.11). Каждый ломает свой слой модов: модели предметов, GUI, шейдеры, рендер сущностей. К 1.21.11 от старого `RenderType`/`MultiBufferSource`-мира почти ничего не осталось — и именно поэтому 26.1 смог убрать обфускацию без ещё одного большого перелома.

## Содержание

- [Хронология 1.21.x](#хронология-121x)
- [Три поправки к популярным тезисам](#три-поправки-к-популярным-тезисам)
- [1.21 / 1.21.1: стабилизация данных](#121--1211-стабилизация-данных)
  - [`ResourceLocation` становится приватным](#resourcelocation-становится-приватным)
  - [Рецепты и `RecipeInput`](#рецепты-и-recipeinput)
  - [`ItemStack` и codec-и](#itemstack-и-codec-и)
  - [Про `@Nullable`](#про-nullable)
- [1.21.2 / 1.21.3: `HolderSet`, render states и шейдеры](#1212--1213-holderset-render-states-и-шейдеры)
  - [`HolderSet` вместо `TagKey`](#holderset-вместо-tagkey)
  - [GUI: `RenderType` как параметр](#gui-rendertype-как-параметр)
  - [Шейдеры: JSON переписан](#шейдеры-json-переписан)
  - [`EntityRenderState`](#entityrenderstate)
  - [`UseAnim` → `ItemUseAnimation`](#useanim--itemuseanimation)
- [1.21.4: `ItemModel`](#1214-itemmodel)
  - [Формат client item: `items/*.json`](#формат-client-item-itemsjson)
  - [Шесть типов моделей](#шесть-типов-моделей)
  - [Что стало с `models/item`](#что-стало-с-modelsitem)
  - [Код рендера предметов](#код-рендера-предметов)
  - [Влияние на ресурспаки](#влияние-на-ресурспаки)
- [1.21.5: `RenderPipeline` и конец JSON-шейдеров](#1215-renderpipeline-и-конец-json-шейдеров)
  - [`GpuDevice`, `CommandEncoder`, `GpuBuffer`](#gpudevice-commandencoder-gpubuffer)
  - [Что стало с `RenderType`](#что-стало-с-rendertype)
  - [GUI, тултипы, инвентарь](#gui-тултипы-инвентарь)
- [1.21.6: `GuiRenderState` и два прохода GUI](#1216-guirenderstate-и-два-прохода-gui)
- [1.21.7 / 1.21.8: стабилизация](#1217--1218-стабилизация)
- [1.21.9: feature submission](#1219-feature-submission)
- [1.21.10: мелочи](#12110-мелочи)
- [1.21.11: последняя обфусцированная](#12111-последняя-обфусцированная)
- [Таблица «что ломалось у модов»](#таблица-что-ломалось-у-модов)
- [Sodium, Iris и чужие рендер-моды](#sodium-iris-и-чужие-рендер-моды)
- [Влияние на миксины GUI и рендера](#влияние-на-миксины-gui-и-рендера)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Хронология 1.21.x

| Версия | Дата | Ключевое изменение |
| --- | --- | --- |
| 1.21 | 2024-06-13 | Датapack-реестр зачарований, `RecipeInput`, приватный `ResourceLocation` |
| 1.21.1 | 2024-08-08 | Хотфикс; API-поверхность 1.21 |
| 1.21.2 | 2024-10-22 | `HolderSet`, GUI render types, новые шейдеры, `EntityRenderState` |
| 1.21.3 | 2024-10-23 | Хотфикс 1.21.2 |
| 1.21.4 | 2024-12-03 | **`ItemModel`**: `assets/<ns>/items/*.json` |
| 1.21.5 | 2025-03-25 | **`RenderPipeline`**, `GpuDevice`, `CommandEncoder`; удалены JSON-шейдеры |
| 1.21.6 | 2025-06-17 | **`GuiRenderState`/`GuiRenderer`**: prepare + render |
| 1.21.7 | 2025-06-30 | Хотфикс |
| 1.21.8 | 2025-07-17 | Хотфикс |
| 1.21.9 | 2025-09-30 | **Feature submission**: `SubmitNodeCollector`, `submit` вместо `render` |
| 1.21.10 | 2025-10-07 | Мелкие изменения |
| 1.21.11 | 2025-12-09 | **Rename shuffle**, `GpuSampler`, `RenderSetup`; последняя обфусцированная версия |

Даты — по Minecraft Wiki. Обратите внимание: 1.21.2 и 1.21.3 вышли с разницей в день; 1.21.7/1.21.8 — тоже хотфиксы. Для порта это означает, что «1.21.3» и «1.21.2» — одна API-линия, а «1.21.8» и «1.21.9» — разные.

---

## Три поправки к популярным тезисам

Прежде чем идти дальше, уберём три неточности, которые кочуют по чатам.

1. **Java 21 пришла не в 1.21, а в 1.20.5.** 1.21 лишь *требует* её. Если мод уже собирался под 1.20.5/1.20.6, менять `toolchain` на 21 при переходе на 1.21 не нужно. Java 25 появится только в 26.1.
2. **`GpuBuffer`/`CommandEncoder` — это 1.21.5, а не 1.21.6.** В 1.21.5 удалили JSON-шейдеры и ввели `RenderPipeline` + `GpuDevice` + `CommandEncoder`. В 1.21.6 переписали *GUI*: `GuiRenderState`/`GuiRenderer`. В 1.21.9 переписали *сущности/блок-энтити/партиклы*: `SubmitNodeCollector`.
3. **`DataComponents` не «появились» в 1.21.** Они появились в 1.20.5. В 1.21.x API только устоялся и оброс новыми компонентами (`ENCHANTABLE`, `REPAIRABLE`, `TOOLTIP_DISPLAY`, `TOOL`, `WEAPON`, `BLOCKS_ATTACKS`).

> [!IMPORTANT]
> Дальше в тексте «1.21.x» означает «1.21–1.21.11», а не «1.21 и всё». Если вы портируете с 1.20.1, сначала прочитайте `4-data-components.md`: без Data Components порт на 1.21 невозможен.

---

## 1.21 / 1.21.1: стабилизация данных

### `ResourceLocation` становится приватным

В 1.21 конструкторы `ResourceLocation` стали приватными. Это ломает почти каждый мод, который создавал ID «руками».

```java
// 1.20.x
new ResourceLocation("examplemod", "example_item");
new ResourceLocation("examplemod:example_item");
new ResourceLocation("minecraft", "stone");

// 1.21+
ResourceLocation.fromNamespaceAndPath("examplemod", "example_item");
ResourceLocation.parse("examplemod:example_item");
ResourceLocation.withDefaultNamespace("stone");
```

Дополнительно:

- `ResourceLocation#of` → `bySeparator`.
- `isValidResourceLocation` удалён.
- В 1.21.11 `ResourceLocation` вообще переименуют в `Identifier` (см. ниже), так что код, который вы напишете сейчас, проживёт недолго.

### Рецепты и `RecipeInput`

1.21 заменила `Container` в рецептах на `RecipeInput`:

```java
// 1.20.x
public class ExampleRecipe implements Recipe<CraftingContainer> {
    @Override
    public boolean matches(CraftingContainer container, Level level) { ... }
}

// 1.21+
public class ExampleRecipe implements Recipe<CraftingInput> {
    @Override
    public boolean matches(CraftingInput input, Level level) {
        return input.getItem(0).is(Items.DIAMOND);
    }
}
```

Что появилось:

- `RecipeInput` — минимальный view: `getItem(int)`, `size()`, `isEmpty()`.
- `CraftingInput` — сетка с шириной/высотой; `CraftingInput#of(...)`.
- `SingleRecipeInput` — один предмет.
- `RecipeHolder`, `RecipeManager`, `RecipeBookMenu` перешли на `RecipeInput`.

Практическое следствие: любой миксин в `RecipeManager#getRecipeFor(Container, ...)` перестаёт применяться — дескриптор другой.

### `ItemStack` и codec-и

К 1.21 `ItemStack` окончательно стал codec-сериализуемым объектом:

```java
// JSON/NBT
ItemStack stack = ItemStack.CODEC.parse(ops, json).getOrThrow();
JsonElement json = ItemStack.CODEC.encodeStart(ops, stack).getOrThrow();

// Сеть
ItemStack.STREAM_CODEC.encode(buf, stack);
ItemStack received = ItemStack.STREAM_CODEC.decode(buf);
```

Точные имена полей (`CODEC`, `OPTIONAL_CODEC`, `STREAM_CODEC`, `OPTIONAL_STREAM_CODEC`, `LIST_STREAM_CODEC`) стоит сверять с исходниками своей версии: Mojang добавляла их постепенно. Но принцип один: **не пишите ручную сериализацию стака**, используйте codec.

Новые компоненты 1.21.x, которые стоит знать:

| Компонент | Где появился | Зачем |
| --- | --- | --- |
| `ENCHANTABLE` | 1.21.2 | заменяет `Item#getEnchantmentValue` |
| `REPAIRABLE` | 1.21.2 | заменяет `Item#isValidRepairItem` |
| `GLIDER` | 1.21.2 | любой предмет может планировать как элитры |
| `TOOL` | 1.21.2 | материал инструмента + правила добычи |
| `TOOLTIP_DISPLAY` | 1.21.5 | скрытие частей тултипа |
| `WEAPON` | 1.21.5 | урон и отключение блока щитом |
| `BLOCKS_ATTACKS` | 1.21.5 | параметры блокирования урона |
| `BREAK_SOUND` | 1.21.5 | звук поломки предмета |

### Про `@Nullable`

В 1.21.x Mojang заметно плотнее расставила nullability-аннотации в исходниках. Для миксинов это **ничего не меняет**: `@Nullable` — это `SOURCE`/`CLASS`-аннотация, она не входит в дескриптор метода и не влияет на выбор таргета. Меняется только качество подсказок в IDE и поведение статических анализаторов.

Практический совет: не пытайтесь «поймать» аннотацию в `@At`/`@Inject` — Mixin работает с байткодом, а не с аннотациями исходников. Если нужна проверка на null, делайте её сами.

---

## 1.21.2 / 1.21.3: `HolderSet`, render states и шейдеры

1.21.2 — самый «широкий» релиз линии: он затронул данные, GUI, шейдеры и рендер сущностей одновременно.

### `HolderSet` вместо `TagKey`

Методы, принимавшие `TagKey<T>` или сырой реестровый объект, начали принимать `HolderSet<T>`.

```java
// 1.20.x
public void setItems(TagKey<Item> tag) { ... }

// 1.21.2+
public void setItems(HolderSet<Item> items) { ... }
```

Два вида `HolderSet`:

- **named** — обёртка над тегом (`#minecraft:planks`);
- **direct** — встроенный список (`HolderSet.direct(...)`).

В JSON это выглядит так:

```json
{ "items": "minecraft:apple" }
{ "items": ["minecraft:apple", "minecraft:stick"] }
{ "items": "#minecraft:planks" }
```

Получить `HolderSet` в коде:

```java
HolderSet<Item> direct = HolderSet.direct(BuiltInRegistries.ITEM.wrapAsHolder(Items.APPLE));
HolderSet<Item> named = BuiltInRegistries.ITEM.getOrThrow(ItemTags.PLANKS);
```

Для datapack-реестров — `HolderGetter#getOrThrow`.

### GUI: `RenderType` как параметр

`GuiGraphics`-методы перестали быть «самодостаточными»: теперь им передают функцию `ResourceLocation → RenderType`, а `blit` требует размер PNG.

```java
// 1.21.1
graphics.blit(TEXTURE, x, y, 0, 0, w, h);

// 1.21.2+
graphics.blit(
    RenderType::guiTextured,
    TEXTURE,
    x, y, 0, 0, w, h,
    256, 256 // размер исходной текстуры
);
```

Что удалено/изменено:

- `GuiGraphics#drawManaged` удалён.
- `GuiGraphics#setColor` удалён — цвет передаётся в `blit`/`blitSprite`.
- `bufferSource()` → `drawSpecial(...)`.
- `PlayerFaceRenderer`-методы принимают дополнительный `int color`.
- `ScreenRectangle`/scissor-логика постепенно переезжает в render state.

### Шейдеры: JSON переписан

1.21.2 переписала формат шейдеров:

- пост-эффекты переехали из `shaders/post` в `post_effect`;
- у каждого шейдера появился JSON-манифест в `assets/<ns>/shaders/*.json`;
- `DiffuseSampler` → `InSampler`/`MainSampler`/`MainDepthSampler`;
- `ShaderInstance` → `CompiledShaderProgram`, `ShaderProgram` — идентификатор;
- `GameRenderer#get*Shader` → `CoreShaders#*`.

Для модов, которые правили core-шейдеры ресурспаком, это означало: старые JSON больше не загружаются. Для модов, которые инжектили шейдеры кодом, — смена типов и регистрации (`CoreShaders#PROGRAMS`).

### `EntityRenderState`

Рендер сущностей разделили на «собрать состояние» и «нарисовать»:

```java
public class MyEntityRenderState extends LivingEntityRenderState {
    public boolean glowing;
}

public class MyEntityRenderer extends AgeableMobRenderer<MyEntity, MyEntityRenderState, MyEntityModel> {
    @Override
    public MyEntityRenderState createRenderState() {
        return new MyEntityRenderState();
    }

    @Override
    public void extractRenderState(MyEntity entity, MyEntityRenderState state, float partialTick) {
        super.extractRenderState(entity, state, partialTick);
        state.glowing = entity.isGlowing();
    }
}
```

Что поменялось:

- `EntityRenderer` получил второй generic — `EntityRenderState`.
- `render` больше не принимает сущность; он принимает state.
- `getTextureLocation` переехал в `LivingEntityRenderer` и принимает state.
- `RenderLayer` generic — тоже state.
- `AgeableMobRenderer` выбирает взрослую/детскую модель по `isBaby`.
- `BabyModelTransform` заменил `AgeableListModel`/`ColorableAgeableListModel`.

Это ещё не submission 1.21.9, но уже подготовка: рендерер больше не читает сущность напрямую.

### `UseAnim` → `ItemUseAnimation`

Да, переименование произошло именно в 1.21.2.

```java
// 1.21.1
stack.getUseAnimation() == UseAnim.EAT;

// 1.21.2+
stack.getUseAnimation() == ItemUseAnimation.EAT;
```

Ломается всё, что импортировало `net.minecraft.world.item.UseAnim`, включая `@Redirect` на `ItemStack#getUseAnimation`.

### Прочие изменения 1.21.2

- `Item.Properties#overrideDescription`/`overrideModel` + компоненты `ITEM_NAME`/`ITEM_MODEL`.
- `ToolMaterial` заменил `Tier`.
- `ArmorMaterial` стал рекордом, а не реестровым объектом; броня описывается компонентами + `EquipmentModel`.
- `Inventory`: `findSlotMatchingUnusedItem` → `findSlotMatchingCraftingIngredient`, `swapPaint` → `setSelectedHotbarSlot`, `StackedContents` → `StackedItemContents` (полная переделка слотов — в 1.21.5).
- `MobEffect`/`MobEffectInstance`-хуки (`applyEffectTick`, `onMobHurt`, `onMobRemoved`) принимают `ServerLevel`; сами эффекты всё ещё `Holder<MobEffect>` (переход был в 1.20.5).

---

## 1.21.4: `ItemModel`

1.21.4 — релиз, который сломал ресурспаки предметов сильнее всего со времён 1.14.

### Формат client item: `items/*.json`

Появился новый тип ассетов: **client item**. Для каждого предмета теперь нужен файл `assets/<namespace>/items/<path>.json`:

```json
{
  "model": {
    "type": "minecraft:model",
    "model": "examplemod:item/example_item"
  }
}
```

`model` указывает на `assets/examplemod/models/item/example_item.json`. То есть старый файл модели **не исчез** — исчезла *прямая* связь «item ID → models/item/<id>.json».

Если файла `items/<id>.json` нет, предмет отрендерится как missing model (или не отрендерится вовсе), даже если `models/item/<id>.json` на месте.

### Шесть типов моделей

| Тип | Назначение |
| --- | --- |
| `minecraft:model` | обычная модель + список tint-источников |
| `minecraft:range_dispatch` | выбор модели по числовому свойству (damage, count, cooldown, time…) |
| `minecraft:select` | выбор по enum/строке (display context, block state, trim material…) |
| `minecraft:condition` | выбор по boolean (damaged, using item, keybind…) |
| `minecraft:composite` | наложение нескольких моделей |
| `minecraft:special` | динамический рендер (сундук, баннер, голова, щит…) |

Пример `range_dispatch` для damage:

```json
{
  "model": {
    "type": "minecraft:range_dispatch",
    "property": "minecraft:damage",
    "scale": 1,
    "fallback": { "type": "minecraft:model", "model": "examplemod:item/example_sword" },
    "entries": [
      { "threshold": 0.5, "model": { "type": "minecraft:model", "model": "examplemod:item/example_sword_damaged" } }
    ]
  }
}
```

Пример `condition` для «в руке»:

```json
{
  "model": {
    "type": "minecraft:condition",
    "property": "minecraft:is_using_item",
    "on_true": { "type": "minecraft:model", "model": "examplemod:item/bow_pulling" },
    "on_false": { "type": "minecraft:model", "model": "examplemod:item/bow" }
  }
}
```

Старые `overrides` в `models/item/*.json` больше не работают: их заменил `range_dispatch`/`select`/`condition`. `ItemOverride` и `BakedOverrides` удалены.

### Что стало с `models/item`

- Файлы `models/item/*.json` остались и всё ещё описывают геометрию/текстуры.
- Но выбирает модель теперь `items/*.json`.
- `ItemModelGenerator` (генерация модели из текстуры) стал `UnbakedModel`.
- `BlockEntityWithoutLevelRenderer` удалён; его место заняли `SpecialModelRenderer`/`NoDataSpecialModelRenderer`.
- `ItemColor`/`ItemColors` удалены; tint задаётся `ItemTintSource` в `items/*.json`.

### Код рендера предметов

Старый путь:

```java
// 1.21.3
BakedModel model = Minecraft.getInstance().getItemRenderer()
    .getModel(stack, level, entity, seed);
itemRenderer.render(stack, ItemDisplayContext.GUI, false, pose, buffer, light, overlay, model);
```

Новый путь:

```java
// 1.21.4+
ItemStackRenderState state = new ItemStackRenderState();
Minecraft.getInstance().getItemModelResolver().updateForTopItem(
    state,
    stack,
    ItemDisplayContext.GUI,
    false,
    level,
    entity,
    0
);
state.render(pose, buffer, light, overlay);
```

Ключевые классы:

- `ItemModelResolver` — обновляет state;
- `ItemStackRenderState` — «отрендеренный» state (слои, foil, transform, bounds);
- `ItemModel` — бейкед-модель, которая умеет `update(...)`, но не рисует;
- `SpecialModelRenderer` — динамика (сундуки, знамёна, головы).

### Влияние на ресурспаки

1. Добавьте `assets/<ns>/items/<id>.json` для каждого предмета.
2. Перенесите `overrides` в `range_dispatch`/`select`/`condition`.
3. Проверьте `custom_model_data`: теперь это `CustomModelDataProperty` внутри `select`.
4. Если предмет использовал `BlockEntityWithoutLevelRenderer`, перепишите его как `SpecialModelRenderer` и зарегистрируйте `MapCodec`.

---

## 1.21.5: `RenderPipeline` и конец JSON-шейдеров

Это самый глубокий рендер-рефакторинг линии. Он не «улучшил» старый API, а заменил его.

### `GpuDevice`, `CommandEncoder`, `GpuBuffer`

Появился слой абстракции над OpenGL:

```java
GpuDevice device = RenderSystem.getDevice();

// Создание текстуры (в 1.21.5 — TextureFormat; в 26.2 станет GpuFormat)
GpuTexture texture = device.createTexture(
    "examplemod:target",
    GpuTexture.USAGE_COLOR_ATTACHMENT | GpuTexture.USAGE_TEXTURE_BINDING,
    TextureFormat.RGBA8,
    width, height, 1, 1
);

// Рендер-пасс
try (RenderPass pass = device.createCommandEncoder()
        .createRenderPass(texture, OptionalInt.of(0xFFFFFFFF))) {
    pass.setPipeline(pipeline);
    pass.setVertexBuffer(0, vertexBuffer);
    pass.draw(vertexCount, 1, 0, 0);
}
```

Компоненты:

- `GpuDevice` — «драйвер»: создаёт буферы, текстуры и энкодеры (сэмплеры как отдельный объект появятся в 1.21.11).
- `CommandEncoder` — записывает команды: `createRenderPass`, `clearColorTexture`, `writeToTexture`, `copyTextureToTexture`.
- `GpuBuffer` — вершинный/индексный/uniform-буфер.
- `RenderPass` — state внутри пасса: pipeline, сэмплеры, scissor, vertex/index buffers, `draw*`.
- `RenderPipeline` — всё, что раньше лежало в JSON шейдера: шейдеры, формат вершин, blend, depth, samplers, uniforms.

```java
public static final RenderPipeline EXAMPLE = RenderPipeline.builder()
    .withLocation(ResourceLocation.fromNamespaceAndPath("examplemod", "pipeline/example"))
    .withVertexShader(ResourceLocation.fromNamespaceAndPath("examplemod", "core/example"))
    .withFragmentShader(ResourceLocation.fromNamespaceAndPath("examplemod", "core/example"))
    .withVertexFormat(DefaultVertexFormat.POSITION_TEX_COLOR, VertexFormat.Mode.QUADS)
    .withSampler("Sampler0")
    .withUniform("ModelViewMat", UniformType.MATRIX4X4)
    .build();
```

> [!WARNING]
> В 1.21.5 `RenderPipeline.Builder` использует `withVertexFormat`, `withSampler`, `withUniform`. В 26.2 это переедет на `withVertexBinding` и `withBindGroupLayout`. Не копируйте примеры 1.21.5 в 26.x без правок.

### Что стало с `RenderType`

`RenderType` остался, но его шейдерная часть теперь ссылается на `RenderPipeline`:

- `MultiBufferSource` работает с `ByteBufferBuilder`;
- JSON-шейдеры (`assets/<ns>/shaders/core/*.json`) удалены; вместо них — код.

Полное разделение `RenderType` на `RenderSetup` + `RenderType` произойдёт только в 1.21.11 (см. ниже). В 1.21.5 `RenderType` ещё хранит настройки, но уже не может «сам» загрузить шейдер из JSON.

Для модов это означает: **нельзя** больше «подменить core-шейдер» ресурспаком. Нужно либо собирать свой `RenderPipeline`, либо инжектить в существующий.

### GUI, тултипы, инвентарь

1.21.5 также затронул данные и GUI:

- GUI всё ещё на `GuiGraphics`; основные изменения — тултипы (`TooltipDisplay`/`TooltipProvider`).
- `appendHoverText(ItemStack, Item.TooltipContext, TooltipDisplay, Consumer<Component>, TooltipFlag)` — новая сигнатура; метод помечен deprecated в пользу `TooltipProvider`.
- `TOOLTIP_DISPLAY` — компонент, управляющий скрытием строк.
- `TooltipProvider#addToTooltip` принимает `DataComponentGetter`.
- `ArmorItem`, `SwordItem`, `DiggerItem`, `PickaxeItem`, `AnimalArmorItem` **удалены**; предметы описываются компонентами.
- `EquipmentSlot.SADDLE`, `EntityEquipment`, `SimpleEquipmentLayer`.
- `Inventory#armor`/`offhand` → `EQUIPMENT_SLOT_MAPPING`; `selected` приватный; `getSelected` → `getSelectedItem`.
- `Item#inventoryTick(ItemStack, Level, Entity, int, boolean)` → `inventoryTick(ItemStack, ServerLevel, Entity, EquipmentSlot)`.
- `BlockEntity#preRemoveSideEffects` + `BlockBehaviour#affectNeighborsAfterRemoval` вместо `onRemove`.
- `MouseHandler` в 1.21.5 **не** переписан: добавились только debug-хелперы (`getScaledXPos`, `getScaledYPos`, `drawDebugMouseInfo`). Существенная переделка ввода будет в 26.3 (SDL).

---

## 1.21.6: `GuiRenderState` и два прохода GUI

1.21.6 разделил отрисовку GUI на **prepare** и **render**.

- **Prepare.** `Screen#render`, `Gui#render`, `AbstractContainerScreen#renderBg` больше не рисуют, а *складывают* элементы в `GuiRenderState`.
- **Render.** `GuiRenderer` читает `GuiRenderState`, сортирует элементы и рисует.

Сортировка:

1. **Strata + linear tree.** `GuiGraphics#nextStratum()` создаёт новый слой; внутри слоя элементы упорядочены по `ScreenArea`.
2. **Comparator:** scissor → порядок `RenderPipeline` → `hashCode` `TextureSetup`.

Элементы:

- `BlitRenderState` — текстура;
- `ColoredRectangleRenderState` — прямоугольник;
- `GlyphRenderState`/`GlyphEffectRenderState` — текст;
- `GuiItemRenderState` — предмет (перед рендером превращается в `BlitRenderState`);
- `GuiTextRenderState` — текст (превращается в glyph-state);
- `PictureInPictureRenderState` — динамика (сундук, книга, entity в инвентаре).

Практические следствия:

```java
// 1.21.5
graphics.renderTooltip(font, component, mouseX, mouseY);

// 1.21.6+
graphics.setTooltipForNextFrame(font, component, mouseX, mouseY);
// ...и позже, когда тултип действительно нужен:
graphics.renderDeferredTooltip();
```

- `drawString` теперь требует ненулевой alpha в ARGB; иначе текст не отрисуется.
- `GuiGraphics#renderItem(ItemStack, int, int, int, int)` удалён.
- `drawSpecial` удалён; вместо него — `submit*RenderState`.
- `LayeredDraw` удалён.
- `RenderType.gui*` удалены; GUI рисуется через `RenderPipelines`.
- Тултипы больше не «рисуются сразу»: они складываются в state и рендерятся на отдельной страте.

Это критично для миксинов: инъекция в `GuiGraphics#renderTooltip` может не сработать, потому что метод переименован/разделён.

---

## 1.21.7 / 1.21.8: стабилизация

1.21.7 и 1.21.8 — хотфиксы. Крупных API-изменений нет; в основном исправления рендера и крашей. Для порта их можно считать «1.21.6 + багфиксы», но не наоборот: мод, собранный под 1.21.8, может использовать методы, которых не было в 1.21.6.

---

## 1.21.9: feature submission

1.21.9 заменила «рендер сразу в буфер» на «отправить описание → отрисовать позже». Это относится к сущностям, блок-энтити, партиклам, имени имён, хитбоксам и т.д.

```java
// 1.21.8
@Override
public void render(MyEntity entity, float yaw, float partialTick, PoseStack pose, MultiBufferSource buffer, int light) {
    // рисуем в buffer
}

// 1.21.9+
@Override
public void submit(MyEntityRenderState state, PoseStack pose, SubmitNodeCollector collector, CameraRenderState camera) {
    collector.submitModel(
        model, state, pose, RenderType.solid(), light, overlay, -1, null, state.outlineColor, null
    );
}
```

Ключевые типы:

- `SubmitNodeCollector` / `OrderedSubmitNodeCollector` — сбор описаний;
- `FeatureRenderDispatcher` — исполнение;
- `SubmitNode`, `SubmitNodeStorage`, `SubmitNodeCollection`;
- `BlockEntityRenderState` — теперь и у блок-энтити;
- `ParticlesRenderState` + `ParticleFeatureRenderer`;
- `SingleQuadParticle` (слияние `SingleQuadParticle` и `TextureSheetParticle`);
- `ArmorModelSet` и `retainExactParts` для брони.

Что ломается:

- `EntityRenderer#render` → `submit`;
- `BlockEntityRenderer#render` → `submit` + `extractRenderState`;
- `RenderLayer#render` → `submit`;
- `EntityRenderer#renderNameTag` → `submitNameTag` (через collector);
- `MultiBufferSource` больше не главный путь для сущностей;
- `ParticleEngine#render` → `ParticleEngine#extract` + `ParticlesRenderState#submit`.

Для модов на Sodium/Iris это означало ещё один слой, который нужно учитывать: теперь мало «обернуть `render`», нужно работать с `SubmitNodeCollector` и feature renderer-ами.

---

## 1.21.10: мелочи

- `HangingEntity#getPopBox` — бокс, в котором висящая сущность «отваливается».
- `BlockBehaviour#entityInside` принимает `boolean` (пересекается/внутри).
- В остальном — багфиксы.

---

## 1.21.11: последняя обфусцированная

1.21.11 — «rename shuffle» и последняя версия перед деобфускацией.

### Переименования

- `ResourceLocation` → **`Identifier`** (во всех сигнатурах, полях, методах).
- `net.minecraft.advancements.critereon` → `criterion`.
- `Util`, `FileUtil`, `BlockUtil` → `net.minecraft.util.*`.
- `net.minecraft.world.entity.*` разложен по подпакетам (`animal.cow`, `monster.zombie`, `projectile.arrow`, …).
- `net.minecraft.client.model.*` — аналогично (`animal.cat`, `object.boat`, `monster.creeper`, …).
- `ResourceLocationArgument` → `IdentifierArgument`; `readResourceLocation` → `readIdentifier`; `ResourceKey#location` → `identifier`.

### Рендер

- `GpuTexture` больше не хранит address/filter mode: появился **`GpuSampler`**.
- `RenderPass#bindTexture` принимает `GpuSampler`.
- `RenderSystem#getSamplerCache()` — кеш всех 32 комбинаций.
- `RenderStateShard` → **`RenderSetup`**; `RenderType` разделён на `RenderSetup` + `RenderType`.
- Все стандартные типы переехали в `RenderTypes` (`RenderType.solid()` → `RenderTypes.solid()`).
- `RenderPipelines.SOLID` → `SOLID_BLOCK`/`SOLID_TERRAIN`; `CUTOUT` → `CUTOUT_BLOCK`/`CUTOUT_TERRAIN`; `TRANSLUCENT` → `TRANSLUCENT_TERRAIN`.
- Появился отдельный атлас предметов `minecraft:items` (`AtlasIds.ITEMS`).
- `mipmap_strategy` в `.mcmeta` (`mean`, `cutout`, `strict_cutout`, `dark_cutout`).
- `Gizmos` — новый API для debug-примитивов (точки, линии, прямоугольники, текст).

### Права

Целочисленный permission level заменён на `Permission`/`PermissionSet`/`PermissionCheck`. Для команд и мод-команд это отдельная миграция.

> [!IMPORTANT]
> 1.21.11 — **последняя обфусцированная версия**. После неё `Yarn`/`intermediary`/`refmap` перестают быть частью уравнения. Следующий файл (`6-unobfuscated.md`) начинается ровно здесь.

---

## Таблица «что ломалось у модов»

| Версия | Что ломалось | Симптом |
| --- | --- | --- |
| 1.21 | `new ResourceLocation(...)`, `Recipe<Container>`, импорт `UseAnim` (частично) | ошибка компиляции |
| 1.21.2 | `GuiGraphics#blit` без `RenderType`; `EntityRenderer#render`; `UseAnim`; shader JSON | компиляция + чёрные/невидимые текстуры |
| 1.21.4 | `models/item` без `items/*.json`; `overrides`; `ItemRenderer#getModel`; `ItemColor`; `BEwLR` | missing model, предмет без текстуры |
| 1.21.5 | JSON core-шейдеры; `RenderType.create`; `ArmorItem`/`SwordItem`; `appendHoverText`; `Inventory#armor` | краш `NoSuchMethodError`, пустые тултипы |
| 1.21.6 | `GuiGraphics#renderTooltip`; `Screen#renderBackground`; `RenderType.gui*`; `LayeredDraw` | GUI без фона/тултипов, миксин не применяется |
| 1.21.9 | `EntityRenderer#render` → `submit`; `BlockEntityRenderer#render`; `ParticleEngine#render` | сущности/партиклы не рисуются |
| 1.21.11 | импорты `ResourceLocation`; `RenderType.solid()`; `GpuTexture` sampler; пакеты entity/model | массовые ошибки компиляции |

---

## Sodium, Iris и чужие рендер-моды

> [!NOTE]
> Ниже — анализ влияния, а не описание внутренностей Sodium/Iris. Конкретные версии и патчи этих модов проверяйте в их трекерах: они обновляются независимо от Minecraft. Помечено `[community]`.

- **1.21.2 (render types + shader JSON).** Любой мод, который подменял `RenderType` или core-шейдер, должен был перейти на новые `RenderType`-фабрики и `CoreShaders`. Для shader-паков это означало переписать пост-эффекты.
- **1.21.4 (`ItemModel`).** Рендер предметов перестал быть «получи BakedModel и нарисуй». Моды, которые кешировали `BakedModel` по `Item`, должны перейти на `ItemStackRenderState`/`ItemModelResolver`.
- **1.21.5 (`RenderPipeline`).** Самый болезненный пункт: JSON-шейдеров больше нет. Всё, что инжектило шейдеры «по имени», должно собирать `RenderPipeline` и работать с `RenderPass`. Sodium/Iris вынуждены переписывать интеграцию; старые shader-паки несовместимы.
- **1.21.6 (`GuiRenderState`).** Моды, которые рисовали GUI «поверх» через `RenderType.gui*`, ломаются: GUI теперь батчится и сортируется `GuiRenderer`-ом.
- **1.21.9 (feature submission).** Обёртки над `EntityRenderer#render`/`BlockEntityRenderer#render` перестают работать: методы заменены на `submit`.
- **1.21.11 (`GpuSampler`, `RenderSetup`).** Ещё один слой переименований; моды, которые лезли в `RenderStateShard`, должны перейти на `RenderSetup`.

Вывод: к 26.1 Sodium/Iris и подобные моды уже живут в мире `RenderPipeline`/`RenderPass`/`SubmitNodeCollector`. Поэтому 26.1 для них — в основном переименования и `Identifier`, а не новый рендер-рефакторинг. Основная боль 26.x — Vulkan (26.2) и Renderpearl (26.3).

---

## Влияние на миксины GUI и рендера

### 1. Дескрипторы GUI меняются каждый релиз

```java
// 1.21.1
@Inject(method = "render", desc = "(Lnet/minecraft/client/gui/GuiGraphics;IIF)V", at = @At("HEAD"))

// 1.21.6 — метод стал частью prepare-фазы, а тултипы — отдельными state
@Inject(method = "setTooltipForNextFrame", desc = "(Lnet/minecraft/client/gui/Font;Lnet/minecraft/network/chat/Component;II)V", at = @At("HEAD"))
```

Правило: **не хардкодьте дескриптор, если можно использовать имя + `@At` с `target`**. Но если имён несколько, дескриптор обязателен — тогда сверяйтесь с `javap`.

### 2. `@Redirect` на `GuiGraphics#blit` почти всегда ломается

`blit` менял сигнатуру в 1.21.2 (добавился `RenderType`), в 1.21.6 (state), в 26.1 (`GuiGraphicsExtractor`). Вместо `@Redirect` лучше:

- `@WrapOperation` на конкретный вызов (цепляется с другими модами);
- `@ModifyArg` на индекс аргумента;
- `@Inject` в метод экрана, а не в `GuiGraphics`.

### 3. `RenderType` больше не «шард»

Если миксин лез в `RenderStateShard`/`RenderType$CompositeState`, в 1.21.11+ его нужно переписать на `RenderSetup`. В 26.x `RenderType` окончательно станет «именованным `RenderSetup`».

### 4. `EntityRenderer#render` → `submit`

```java
// 1.21.8
@Inject(method = "render", at = @At("HEAD"))
private void example$beforeRender(MyEntity entity, ..., CallbackInfo ci) { ... }

// 1.21.9+
@Inject(method = "submit", at = @At("HEAD"))
private void example$beforeSubmit(MyEntityRenderState state, PoseStack pose, SubmitNodeCollector collector, CameraRenderState camera, CallbackInfo ci) { ... }
```

Если мод «дорисовывал» что-то после сущности, теперь это делается через `collector.submit*` в `submit`, а не через `MultiBufferSource` в `render`.

### 5. Шейдерные миксины

До 1.21.5 можно было инжектить в `ShaderInstance`/`Program`. После — только в `RenderPipeline`/`CompiledShaderProgram`/`RenderPass`. При этом `RenderPipeline` часто `static final`, и «подменить» его миксином сложнее: обычно нужен свой pipeline и `RenderType`.

### 6. `@Local` и locals

До 26.1 локальные переменные в обфусцированном коде не имеют стабильных имён. `@Local(name = "...")` не работает; приходится использовать тип + ordinal или `@WrapOperation`. Это ещё одна причина, по которой 26.1 упрощает миксины (см. следующий файл).

---

## Проверено на

- Дата проверки: **2026-11-21**.
- Версии: Minecraft **1.21 / 1.21.1 / 1.21.2 / 1.21.3 / 1.21.4 / 1.21.5 / 1.21.6 / 1.21.7 / 1.21.8 / 1.21.9 / 1.21.10 / 1.21.11**.
- Источники уровня API: NeoForge-праймеры `1.20.6 → 1.21`, `1.21.1 → 1.21.2`, `1.21.2/3 → 1.21.4`, `1.21.4 → 1.21.5`, `1.21.5 → 1.21.6`, `1.21.8 → 1.21.9`, `1.21.9 → 1.21.10`, `1.21.10 → 1.21.11`; Fabric blog 26.1.
- Утверждения про Sodium/Iris помечены `[community]`: их внутренности не проверялись по исходникам, только по влиянию публичного API.
- Даты релизов — Minecraft Wiki.

## Источники

- NeoForged, *Minecraft 1.20.6 → 1.21* — <https://docs.neoforged.net/primer/docs/1.21/>
- NeoForged, *Minecraft 1.21.1 → 1.21.2* — <https://docs.neoforged.net/primer/docs/1.21.2/>
- NeoForged, *Minecraft 1.21.2/3 → 1.21.4* — <https://docs.neoforged.net/primer/docs/1.21.4/>
- NeoForged, *Minecraft 1.21.4 → 1.21.5* — <https://docs.neoforged.net/primer/docs/1.21.5/>
- NeoForged, *Minecraft 1.21.5 → 1.21.6* — <https://docs.neoforged.net/primer/docs/1.21.6/>
- NeoForged, *Minecraft 1.21.8 → 1.21.9* — <https://docs.neoforged.net/primer/docs/1.21.9/>
- NeoForged, *Minecraft 1.21.9 → 1.21.10* — <https://docs.neoforged.net/primer/docs/1.21.10/>
- NeoForged, *Minecraft 1.21.10 → 1.21.11* — <https://docs.neoforged.net/primer/docs/1.21.11/>
- Minecraft Wiki, *Java Edition 1.21.4* (client items) — <https://minecraft.wiki/w/Java_Edition_1.21.4>
- Fabric, *Fabric for Minecraft 26.1* (Loom 1.15, Loader 0.18.4, `ItemStackTemplate`) — <https://fabricmc.net/2026/03/14/261.html>
- Fabric Wiki (draft), *Mixins and Obfuscation* — <https://wiki.fabricmc.net/drafts:mixin_obfuscation>
