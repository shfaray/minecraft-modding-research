# Эпоха 2: flattening и ModLauncher (1.13–1.16.5)

[← К оглавлению](00-index.md)

> [!NOTE]
> **TL;DR.** 1.13 — это flattening: метаданные умерли, блоки и предметы стали отдельными записями реестра, данные переехали в `data/`. 1.14 дала `DeferredRegister`, 1.15 — `MatrixStack`, 1.16.5 — последняя версия на MCP. Mixin появляется в составе Forge только в поздних 1.15.2 и 1.16.1; до этого его подключали сбоку.

## Содержание

- [Контекст эпохи](#контекст-эпохи)
  - [Хронология](#хронология)
  - [Почему 1.13 «проскочили»](#почему-113-проскочили)
  - [Mappings и два ремапа](#mappings-и-два-ремапа)
- [Главные breaking changes](#главные-breaking-changes)
  - [Сводная таблица](#сводная-таблица)
  - [Flattening: что именно сломалось](#flattening-что-именно-сломалось)
  - [Регистрация: `ForgeRegistries` и `DeferredRegister`](#регистрация-forgeregistries-и-deferredregister)
  - [Рендер: `MatrixStack`](#рендер-matrixstack)
  - [1.16: `RegistryKey` и измерения](#116-registrykey-и-измерения)
- [Что это значит для миксинов](#что-это-значит-для-миксинов)
  - [Цели: MCP 1.12.2 → MCP 1.16.5](#цели-mcp-1122--mcp-1165)
  - [Исчезнувшие методы](#исчезнувшие-методы)
  - [Доступность и `@Shadow`](#доступность-и-shadow)
  - [`@At`, дескрипторы и лямбды](#at-дескрипторы-и-лямбды)
  - [Пример миксина для 1.16.5](#пример-миксина-для-1165)
- [Как жил моддинг](#как-жил-моддинг)
  - [ModLauncher и конец coremod'ов](#modlauncher-и-конец-coremodов)
  - [Когда Mixin попал в Forge](#когда-mixin-попал-в-forge)
  - [`@Mod`, `DistExecutor` и смерть `@SidedProxy`](#mod-distexecutor-и-смерть-sidedproxy)
  - [Fabric: другая философия](#fabric-другая-философия)
  - [Access Transformers и Access Wideners](#access-transformers-и-access-wideners)
- [Практический вывод](#практический-вывод)
  - [Порядок порта](#порядок-порта)
  - [Чеклист](#чеклист)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Контекст эпохи

### Хронология

| Версия | Дата релиза | Java | Лоадеры | Сборка | Mappings |
| --- | --- | --- | --- | --- | --- |
| 1.13 | 2018-07-18 | Java 8 | Forge 25.x, Rift | ForgeGradle 3.x | MCP + SRG |
| 1.13.1 | 2018-08-22 | Java 8 | Forge 25.x | ForgeGradle 3.x | MCP + SRG |
| 1.13.2 | 2018-10-22 | Java 8 | Forge 25.0.223, Rift | ForgeGradle 3.x | MCP + SRG |
| 1.14 | 2019-04-23 | Java 8 | Forge 26.x, Fabric | ForgeGradle 3.x, Loom | MCP + SRG; Yarn + intermediary |
| 1.14.4 | 2019-07-19 | Java 8 | Forge 28.2.30, Fabric | ForgeGradle 3.x, Loom | MCP + SRG; Yarn |
| 1.15 | 2019-12-10 | Java 8 | Forge 29.x, Fabric | ForgeGradle 4.x, Loom | MCP + SRG; Yarn |
| 1.15.2 | 2020-01-21 | Java 8 | Forge 31.2.62, Fabric | ForgeGradle 4.x, Loom | MCP + SRG; Yarn |
| 1.16 | 2020-06-23 | Java 8 | Forge 32.x, Fabric | ForgeGradle 4.x, Loom | MCP + SRG; Yarn |
| 1.16.1 | 2020-06-24 | Java 8 | Forge 32.0.108, Fabric | ForgeGradle 4.x, Loom | MCP + SRG; Yarn |
| 1.16.4 | 2020-11-02 | Java 8 | Forge 35.1.37, Fabric | ForgeGradle 4.x, Loom | MCP + SRG; Yarn |
| 1.16.5 | 2021-01-15 | Java 8 | Forge 36.2.42, Fabric | ForgeGradle 4.x, Loom | MCP + SRG; Yarn |

> [!IMPORTANT]
> **1.16.5 — последняя версия, где Forge работал на MCP.** В 1.17 Forge переходит на официальные mappings Mojang. Поэтому 1.16.5 — естественная «база» для сравнения: справа от неё начинается эпоха переименований, слева — flattening.

### Почему 1.13 «проскочили»

Forge для 1.13 разрабатывался долго и вышел с большим опозданием. Пока он дозревал, вышла 1.14, и сообщество прыгнуло сразу на неё. Официальная документация Forge прямо пишет: «Due to major rewrites to Forge happening during the lifetime of 1.13 it never saw much usage and instead most mods ported directly to 1.14 skipping over 1.13».

Это важно для порта:

- если мод существует в версиях 1.12.2 и 1.14.4, скорее всего, он **никогда не собирался под 1.13**;
- учебники по 1.13 часто описывают промежуточный API, которого уже нет в 1.14;
- «порт на 1.13» почти всегда бессмысленен — целевой версией нужно брать 1.14.4 или 1.16.5.

### Mappings и два ремапа

| Слой | Forge | Fabric |
| --- | --- | --- |
| Имена в исходниках | MCP (`stable_39`, snapshot 20210309-1.16.5) | Yarn |
| Имена в рантайме | SRG (`func_...`, `field_...`) | intermediary (`method_...`, `field_...`) |
| Ремап при сборке | ForgeGradle reobf | Loom + TinyRemapper |
| Refmap для миксинов | обязателен | обязателен (до Loom 1.12+) |

В 1.13–1.16.5 ещё нет «официальных Mojang mappings» в Forge. Это значит, что:

- `@Shadow` и `@At` пишутся MCP-именами;
- AT-файлы пишутся SRG-именами;
- crash-report показывает SRG, а не MCP;
- при обновлении mappings (например, `stable_39` → snapshot) часть имён может измениться.

---

## Главные breaking changes

### Сводная таблица

| # | Было | Стало | Версия | Последствие |
| --- | --- | --- | --- | --- |
| 1 | `int metadata` у блока | `BlockState` + `Property` | 1.13 | `getStateFromMeta`/`getMetaFromState` исчезают; состояния — это объекты |
| 2 | `minecraft:wool` + meta 14 | `minecraft:red_wool` | 1.13 | ID предметов/блоков меняются; рецепты и теги ломаются |
| 3 | `GameRegistry.register(...)` | `ForgeRegistries` + `RegistryEvent.Register<T>` | 1.13 | `GameRegistry` теряет методы регистрации, остаётся `findRegistry` |
| 4 | `RegistryEvent.Register` вручную | `DeferredRegister` + `RegistryObject` | 1.14 | Статическая инициализация становится безопасной |
| 5 | `mcmod.info` | `META-INF/mods.toml` | 1.13 | Метаданные мода, зависимости и `modLoader` переезжают в TOML |
| 6 | `IBlockState` | `BlockState` | 1.14 | Меняется тип во всех сигнатурах; `IBlockState` исчезает |
| 7 | `ITickable` | `ITickableTileEntity` | 1.14 | Старое имя `ITickable` занимает рендер текстур |
| 8 | `OreDictionary` | item tags (`data/<ns>/tags/items/`) | 1.13 | Рецепты и совместимость переезжают на теги |
| 9 | `EntityRegistry.registerModEntity` | `ForgeRegistries.ENTITIES` + `EntityType` | 1.13 | Регистрация сущностей — через реестр, а не helper |
| 10 | `GameRegistry.registerTileEntity` | `TileEntityType` + `RegistryEvent` | 1.13 | Появляется фабрика `TileEntityType.Builder` |
| 11 | `EnumActionResult` | `ActionResultType` | 1.13–1.14 | Меняется возвращаемый тип `onBlockActivated`/`onItemUse` |
| 12 | `EnumHand` / `EnumFacing` | `Hand` / `Direction` | 1.14 | MCP подтягивает имена к ваниле; дескрипторы меняются |
| 13 | `TextComponentTranslation` | `TranslationTextComponent` | 1.15 | Переименование класса в MCP |
| 14 | `new Block(...)` + сеттеры | `Block.Properties` / `Item.Properties` | 1.13–1.14 | Конструкторы становятся builder-based |
| 15 | `GlStateManager` push/pop | `MatrixStack` + `RenderSystem` | 1.15 | Рендер-миксины меняют сигнатуры |
| 16 | `RegistryKey` для миров | `RegistryKey` + `DimensionType` registry | 1.16 | Измерения становятся datapack-driven |

> [!NOTE]
> `World` и `TileEntity` в этой эпохе **не переименовываются** — это произойдёт только в 1.17. `IChatComponent` → `ITextComponent` тоже относится к 1.13, но вынесен в следующий файл, чтобы не смешивать эпохи.

> [!WARNING]
> Таблица смешивает **удаления** и **переименования**. `IBlockState` → `BlockState` — это не «новый интерфейс рядом со старым»: в 1.14 старого типа нет, и весь код, включая миксины, нужно менять.

### Flattening: что именно сломалось

Flattening — это не одно изменение, а пакет из пяти:

1. **Блоки.** Числовые метаданные удалены. Варианты, которые раньше были `meta`, стали либо blockstate-свойствами, либо отдельными блоками.
2. **Предметы.** `ItemStack` с damage-значением заменён отдельными предметами. `minecraft:dye` с meta 1 → `minecraft:red_dye`; `minecraft:coal` с meta 1 → `minecraft:charcoal`.
3. **Реестры.** Блоки и предметы — разные реестры. Не у каждого блока есть предмет.
4. **Данные.** Рецепты, loot tables, advancements, теги переехали из `assets/` в `data/`.
5. **Локализация.** Ключи `item.<name>.name` и `tile.<name>.name` заменены на `item.<namespace>.<path>` и `block.<namespace>.<path>`.

**Было (1.12.2):**

```java
// meta = 1 — гранит
IBlockState granite = Blocks.STONE.getStateFromMeta(1);
world.setBlockState(pos, granite);
```

**Стало (1.13+):**

```java
// гранит — отдельный блок
BlockState granite = Blocks.GRANITE.defaultBlockState();
world.setBlockState(pos, granite);
```

**Blockstate-файл (1.13+):**

```json
{
  "variants": {
    "facing=north": { "model": "mymod:block/my_machine" },
    "facing=south": { "model": "mymod:block/my_machine", "y": 180 },
    "facing=west":  { "model": "mymod:block/my_machine", "y": 270 },
    "facing=east":  { "model": "mymod:block/my_machine", "y": 90 }
  }
}
```

**Тег вместо `OreDictionary` (1.13+):**

```json
{
  "replace": false,
  "values": ["minecraft:iron_ingot", "mymod:steel_ingot"]
}
```

Файл лежит в `data/forge/tags/items/ingots/iron.json` (для common-тегов Forge) или в `data/<ns>/tags/items/...` для собственных.

> [!NOTE]
> Flattening больно бьёт не по коду, а по **данным**: старые миры, старые рецепты, старые конфиги и старые NBT-теги. Forge/Mojang добавляют DataFixer'ы для ванильных миров, но модовые ID они не знают. Если мод хранил в NBT `id: "mymod:foo"` с damage, после 1.13 это нужно мигрировать вручную.

### Регистрация: `ForgeRegistries` и `DeferredRegister`

**1.13 — только событие.** `GameRegistry` больше не регистрирует, у него остался `findRegistry`. Появляется `ForgeRegistries`:

```java
// 1.13.2
@Mod.EventBusSubscriber(modid = "mymod", bus = Mod.EventBusSubscriber.Bus.MOD)
public class Registration {
    @SubscribeEvent
    public static void registerBlocks(RegistryEvent.Register<Block> event) {
        event.getRegistry().register(
            new MyBlock(Block.Properties.create(Material.ROCK))
                .setRegistryName("mymod", "my_block")
        );
    }
}
```

> [!IMPORTANT]
> В 1.13+ `RegistryEvent` приходит на **mod event bus**, а не на `MinecraftForge.EVENT_BUS`. Поэтому в `@Mod.EventBusSubscriber` нужен `bus = Bus.MOD`; иначе обработчик просто не вызовется. В 1.12.2 такого разделения не было — событие шло на общий bus.

**1.14 — `DeferredRegister`.** Теперь можно объявлять объекты статически, а регистрация произойдёт в правильный момент:

```java
// 1.14.4+
public class MyMod {
    public static final DeferredRegister<Block> BLOCKS =
        DeferredRegister.create(ForgeRegistries.BLOCKS, "mymod");

    public static final RegistryObject<Block> MY_BLOCK =
        BLOCKS.register("my_block",
            () -> new Block(Block.Properties.of(Material.STONE)));

    public MyMod() {
        BLOCKS.register(FMLJavaModLoadingContext.get().getModEventBus());
    }
}
```

| Версия | Регистрация | Когда | Плюсы | Минусы |
| --- | --- | --- | --- | --- |
| 1.13 | `RegistryEvent.Register<T>` | после конструктора мода | явно, гибко | много boilerplate |
| 1.14+ | `DeferredRegister` | лениво, по событию | статические поля, `RegistryObject` | нужно не забыть `register(bus)` |

> [!TIP]
> `DeferredRegister` не заменяет `RegistryEvent` полностью. Событие всё ещё нужно для вещей, у которых нет удобного `DeferredRegister` (например, кастомные реестры Forge), и для порядка инициализации. Но для блоков, предметов, `TileEntityType`, `EntityType`, `SoundEvent` он стал стандартом.

### Рендер: `MatrixStack`

1.15 принесла `MatrixStack` и `RenderSystem`. Старый код с `GlStateManager.pushMatrix()`/`popMatrix()` и `glTranslatef` перестал быть правильным:

```java
// 1.14: глобальный стек OpenGL
GlStateManager.pushMatrix();
GlStateManager.translatef(x, y, z);
// ... рендер ...
GlStateManager.popMatrix();

// 1.15+: стек передаётся параметром
matrixStack.push();
matrixStack.translate(x, y, z);
// ... рендер ...
matrixStack.pop();
```

Для миксинов это критично: методы рендера получили новый параметр `MatrixStack`, и старый `@At`-дескриптор перестал совпадать. В 1.17 `MatrixStack` переименуют в `PoseStack`, а `push`/`pop` — в `pushPose`/`popPose`, но суть та же.

| Что | 1.14 | 1.15–1.16.5 | 1.17+ |
| --- | --- | --- | --- |
| Трансформации | `GlStateManager` | `MatrixStack` | `PoseStack` |
| Вершинный буфер | `BufferBuilder` | `IVertexBuilder` | `VertexConsumer` |
| Отрисовка | `RenderType`/`GlStateManager` | `RenderType` + `IRenderTypeBuffer` | `RenderType` + `MultiBufferSource` |

### 1.16: `RegistryKey` и измерения

1.16 сделала измерения и worldgen более datapack-driven:

- `RegistryKey<World>` и `RegistryKey<DimensionType>` стали обычным способом сослаться на мир/тип измерения;
- `DimensionType` превратился в реестр;
- `ServerWorld` создаётся через `Dimension`/`WorldGenSettings`, а не только через `WorldProvider`;
- `RegistryAccess` даёт доступ к реестрам из `MinecraftServer`.

Для модов это означало: кастомные измерения перестали быть «наследником `WorldProvider`» и стали набором JSON + код-генератора. Для миксинов — новые цели в `MinecraftServer#createLevels`, `DimensionType` и `RegistryKey`.

---

## Что это значит для миксинов

### Цели: MCP 1.12.2 → MCP 1.16.5

| 1.12.2 | 1.16.5 | Версия перехода |
| --- | --- | --- |
| `net.minecraft.world.World` | `net.minecraft.world.World` | без изменений |
| `net.minecraft.tileentity.TileEntity` | `net.minecraft.tileentity.TileEntity` | без изменений |
| `net.minecraft.block.state.IBlockState` | `net.minecraft.block.BlockState` | 1.14 |
| `net.minecraft.entity.player.EntityPlayer` | `net.minecraft.entity.player.PlayerEntity` | 1.15–1.16 (MCP) |
| `net.minecraft.entity.EntityLivingBase` | `net.minecraft.entity.LivingEntity` | 1.15–1.16 (MCP) |
| `net.minecraft.entity.player.EntityPlayerMP` | `net.minecraft.entity.player.ServerPlayerEntity` | 1.15–1.16 (MCP) |
| `net.minecraft.util.EnumHand` | `net.minecraft.util.Hand` | 1.14 |
| `net.minecraft.util.EnumFacing` | `net.minecraft.util.Direction` | 1.14 |
| `net.minecraft.util.EnumActionResult` | `net.minecraft.util.ActionResultType` | 1.13–1.14 |
| `net.minecraft.util.text.IChatComponent` | `net.minecraft.util.text.ITextComponent` | 1.13 |
| `net.minecraft.util.text.TextComponentTranslation` | `net.minecraft.util.text.TranslationTextComponent` | 1.15 |
| `net.minecraft.tileentity.ITickable` | `net.minecraft.tileentity.ITickableTileEntity` | 1.14 |

> [!IMPORTANT]
> Это **не** тот же самый переход, что 1.17. Здесь меняются MCP-имена внутри MCP-мира; в 1.17 меняется вся система mappings на Mojmap. Не смешивайте таблицы: `PlayerEntity` — это ещё MCP, `Player` — уже Mojmap.

### Исчезнувшие методы

| Метод/класс | Версия | Чем заменить |
| --- | --- | --- |
| `Block#getStateFromMeta(int)` | 1.13 | `Block#getDefaultState()` + `withProperty` |
| `Block#getMetaFromState(IBlockState)` | 1.13 | `BlockState#getValue`/`getProperties` |
| `Block#damageDropped(IBlockState)` | 1.13 | отдельные предметы / loot tables |
| `Item#getSubItems(CreativeTabs, NonNullList)` | 1.13 | `ItemGroup`/`CreativeModeTab` + `ItemStack` |
| `ItemStack#getItemDamage()` | 1.13 | `getDamage()` (позже — компоненты) |
| `OreDictionary` | 1.13 | теги |
| `GameRegistry.register*` | 1.13 | `RegistryEvent`/`DeferredRegister` |
| `EntityRegistry.registerModEntity` | 1.13 | `ForgeRegistries.ENTITIES` |
| `ITileEntityProvider#createNewTileEntity(World, int)` | 1.13/1.14 | `TileEntityType` + `Block#createTileEntity` |
| `GlStateManager#pushMatrix` | 1.15 | `MatrixStack#pushPose` |

Если миксин целился в `getStateFromMeta`, после 1.13 он не просто «не применится» — его цель физически исчезла. Компилятор об этом не скажет, потому что `@Shadow`/`@At` — это строки.

### Доступность и `@Shadow`

В 1.13–1.16.5 модификаторы доступа всё ещё MCP-шные, но состав полей изменился:

- `TileEntity#world` — `protected World`, `@Shadow` работает;
- `TileEntity#pos` — `protected BlockPos`, `@Shadow` работает;
- `TileEntity#tileEntityInvalid` — `protected boolean`, доступен;
- `BlockState#getBlock()` — публичный, `@Shadow` не нужен;
- `BlockState` внутренние поля — `private`, нужен `@Accessor`;
- `ItemStack#count`/`stackSize` — `private`, нужен `@Accessor` или `getCount()`;
- `World#isRemote` — `public final boolean`; запись только через `@Mutable`;
- `PlayerEntity#inventory` — `public final PlayerInventory`; менять поле нельзя, но методы доступны.

> [!WARNING]
> В 1.13+ участились случаи, когда `@Shadow` на поле компилируется, но падает при загрузке: поле переименовали в рамках MCP-снапшота, а refmap остался старым. Всегда перегенерируйте refmap после смены `mappings` в `build.gradle`.

### `@At`, дескрипторы и лямбды

**Дескриптор 1.12.2:**

```java
@At(value = "INVOKE",
    target = "Lnet/minecraft/world/World;getBlockState(Lnet/minecraft/util/math/BlockPos;)Lnet/minecraft/block/state/IBlockState;")
```

**Дескриптор 1.16.5:**

```java
@At(value = "INVOKE",
    target = "Lnet/minecraft/world/World;getBlockState(Lnet/minecraft/util/math/BlockPos;)Lnet/minecraft/block/BlockState;")
```

Разница только в return type (`IBlockState` → `BlockState`), но этого достаточно, чтобы инъекция не нашлась.

В 1.13+ стало больше лямбд и `invokedynamic`: регистрация, datagen, `RegistryObject`, `DeferredRegister` активно используют `Supplier`. Если `@At` целится в вызов внутри лямбды, обычный `INVOKE` может не сработать:

- цель может быть в synthetic-методе `lambda$...$0`;
- `INVOKEDYNAMIC` требует указывать `target` иначе, чем `INVOKE`;
- иногда проще `@Redirect` на `Supplier#get` или `@ModifyVariable`, чем ловить лямбду.

### Пример миксина для 1.16.5

```java
package com.example.mymod.mixin;

import net.minecraft.block.BlockState;
import net.minecraft.util.math.BlockPos;
import net.minecraft.world.World;
import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.injection.At;
import org.spongepowered.asm.mixin.injection.Inject;
import org.spongepowered.asm.mixin.injection.callback.CallbackInfoReturnable;

@Mixin(World.class)
public class WorldMixin {

    @Inject(
        method = "setBlockState(Lnet/minecraft/util/math/BlockPos;Lnet/minecraft/block/BlockState;I)Z",
        at = @At("HEAD"),
        cancellable = true
    )
    private void mymod$beforeSetBlock(BlockPos pos, BlockState state, int flags,
                                      CallbackInfoReturnable<Boolean> cir) {
        // ...
    }
}
```

Конфиг:

```json
{
  "required": true,
  "minVersion": "0.8",
  "package": "com.example.mymod.mixin",
  "compatibilityLevel": "JAVA_8",
  "refmap": "mixins.mymod.refmap.json",
  "mixins": ["WorldMixin"],
  "injectors": { "defaultRequire": 1 }
}
```

Для Forge 1.15.2+/1.16.1+ конфиг попадает в рантайм через манифест `MixinConfigs` (это делает MixinGradle), а не через `mods.toml`.

---

## Как жил моддинг

### ModLauncher и конец coremod'ов

1.13 заменила `LaunchWrapper` на **ModLauncher**. Это изменило способ вмешательства в загрузку:

| Аспект | 1.12.2 | 1.13+ |
| --- | --- | --- |
| Загрузчик | `LaunchWrapper` | `ModLauncher` |
| Точка входа мода | `@Mod` + `IFMLLoadingPlugin` для coremod | `@Mod` + `ILaunchPluginService` для launch-плагинов |
| Трансформер | `IClassTransformer` | `ILaunchPluginService`/`ITransformationService` |
| Mixin | сбоку (Bootstrap) | через ModLauncher, но библиотека появляется только в 1.15.2/1.16.1 |
| Метаданные | `mcmod.info` | `mods.toml` |

Старые coremod'ы в 1.13+ не работают «как есть»: `IClassTransformer` больше не является публичным путём. Для совместимости Forge какое-то время держала переходные классы, но правильный путь — `ILaunchPluginService` или Mixin.

> [!NOTE]
> ModLauncher — отдельная библиотека (cpw/modlauncher), а не часть Forge. Именно она даёт точку расширения, в которую Forge встраивает Mixin. Поэтому «Mixin в Forge» и «ModLauncher» — связанные, но не одинаковые вещи: ModLauncher появился в 1.13, а Mixin в составе Forge — только в 1.15.2/1.16.1.

### Когда Mixin попал в Forge

| Версия | Статус | Детали |
| --- | --- | --- |
| 1.13.2 | нет | `MixinBootstrap` не поддерживает 1.13 из-за версии ModLauncher |
| 1.14.4 | нет | `MixinBootstrap` работает частично, только с Forge 28.1.45+ |
| 1.15.2 | да, 31.2.44+ | Mixin входит в Forge |
| 1.16.1 | да, 32.0.72+ | Mixin входит в Forge |
| 1.16.5 | да | 36.2.x содержит Mixin |

До этого использовали:

- **MixinBootstrap** — отдельный jar в `mods/`, грузит Mixin через `LaunchWrapper`/ModLauncher;
- **MixinBooter** — аналог для 1.8–1.12.2, популярный в legacy-сборках;
- **shade** — встроить Mixin в свой jar (плохо: конфликты версий с другими модами).

С появлением встроенного Mixin `shade` стал противопоказан: Mixin wiki прямо говорит, что это «likely cause issues».

### `@Mod`, `DistExecutor` и смерть `@SidedProxy`

В 1.13 исчезла аннотация `@SidedProxy`. Прокси-паттерн не умер, но подключать его нужно вручную. Появился `DistExecutor`:

```java
// 1.13
public class MyMod {
    public MyMod() {
        DistExecutor.runWhenOn(Dist.CLIENT, () -> () -> ClientSetup.init());
    }
}
```

В 1.15+ появились `safeRunWhenOn` и `unsafeRunWhenOn`; `runWhenOn` остался как «небезопасный» вариант.

| Паттерн | 1.12.2 | 1.13+ |
| --- | --- | --- |
| Прокси | `@SidedProxy(clientSide=..., serverSide=...)` | `DistExecutor` + `FMLEnvironment.dist` |
| Регистрация | `MinecraftForge.EVENT_BUS.register(this)` | `FMLJavaModLoadingContext.get().getModEventBus()` |
| Обработчики | `@Mod.EventHandler` | `@SubscribeEvent` + `@Mod.EventBusSubscriber` |
| Статическая инициализация | не рекомендуется | `DeferredRegister` (1.14+) |

`FMLJavaModLoadingContext` появился в 1.13; в 1.14+ он стал стандартной точкой получения mod event bus. Constructor injection выглядит так:

```java
@Mod("mymod")
public class MyMod {
    public MyMod() {
        IEventBus bus = FMLJavaModLoadingContext.get().getModEventBus();
        bus.addListener(this::setup);
        bus.addListener(this::clientSetup);
        MinecraftForge.EVENT_BUS.register(this);
    }

    private void setup(FMLCommonSetupEvent event) { /* ... */ }
    private void clientSetup(FMLClientSetupEvent event) { /* ... */ }
}
```

> [!TIP]
> Если вы видите в 1.16.5 моде `@SidedProxy`, это либо порт с 1.12.2, либо самодельная аннотация. В самом Forge её нет с 1.13. Правильный путь — `DistExecutor.safeRunWhenOn` (безопасный) или `unsafeRunWhenOn` (если вы уверены, что класс не загрузится на другой стороне).

### Fabric: другая философия

С 1.14 у мододелов появился выбор. Fabric строился вокруг противоположных принципов:

| Аспект | Forge 1.14–1.16.5 | Fabric 1.14–1.16.5 |
| --- | --- | --- |
| Метаданные | `mods.toml` | `fabric.mod.json` |
| Точка входа | `@Mod` + event bus | `ModInitializer#onInitialize` |
| Регистрация | `DeferredRegister`/`RegistryEvent` | `Registry.register(...)` |
| Mappings | MCP + SRG | Yarn + intermediary |
| Mixin | с 1.15.2/1.16.1 | всегда, через Loader |
| Доступ | Access Transformer | Access Widener |
| API | большой, встроенный | тонкий loader + `Fabric API` |

Fabric-мод не использует прокси: стороны различаются через `@Environment` и `FabricLoader.getInstance().getEnvironmentType()`.

```java
// Fabric 1.16.5
public class MyMod implements ModInitializer {
    public static final Item MY_ITEM = new Item(new Item.Settings().group(ItemGroup.MISC));

    @Override
    public void onInitialize() {
        Registry.register(Registry.ITEM, new Identifier("mymod", "my_item"), MY_ITEM);
    }
}
```

Для миксинов это означало: на Fabric refmap тоже нужен, но имена другие (Yarn), а `compatibilityLevel` и структура конфига те же.

### Access Transformers и Access Wideners

**Forge (1.13–1.16.5):**

```text
# src/main/resources/META-INF/accesstransformer.cfg
public net.minecraft.tileentity.TileEntity field_145850_b # world
```

- имена полей/методов — **SRG**;
- файл должен лежать строго по пути `META-INF/accesstransformer.cfg` в jar;
- в dev-окружении путь задаётся в `build.gradle` через `accessTransformer = file(...)`.

**Fabric:**

```text
# src/main/resources/mymod.accesswidener
accessWidener v1 named
accessible field net/minecraft/block/entity/BlockEntity world Lnet/minecraft/world/World;
```

Access Widener работает с Yarn-именами и не зависит от SRG. Это одна из причин, почему на Fabric проще «достать» приватное поле: не нужно держать в голове два namespace.

---

## Практический вывод

### Порядок порта

Если мод портируется **с 1.12.2 на 1.14.4/1.16.5**:

1. **Flattening.** Пройти по всем блокам и предметам, заменить meta на blockstates/отдельные записи. Это самая долгая часть.
2. **Регистрация.** `GameRegistry` → `DeferredRegister`; `EntityRegistry` → `ForgeRegistries.ENTITIES`; `registerTileEntity` → `TileEntityType`.
3. **Данные.** Рецепты, теги, loot tables, advancements — в `data/`; `OreDictionary` → теги.
4. **Конструкторы.** `new Block(...)`/`new Item(...)` → `Properties`; `setUnlocalizedName` → `setTranslationKey` (и далее `Component.translatable` в 1.19).
5. **Рендер.** `GlStateManager` → `MatrixStack`; TESR → `TileEntityRenderer`; модели — JSON.
6. **Прокси.** `@SidedProxy` → `DistExecutor`; `@Mod.EventHandler` → `@SubscribeEvent`.
7. **Миксины.** Переписать цели, обновить дескрипторы, перегенерировать refmap.

Если мод портируется **с 1.16.5 на 1.17+**, не начинайте с этой таблицы: сразу читайте следующий файл. Там другая система имён, и половина описанных здесь переименований уже неактуальна.

### Чеклист

- [ ] Найти все `metadata`/`getStateFromMeta`/`getMetaFromState` и заменить на blockstates.
- [ ] Проверить ID всех предметов: `wool`/`dye`/`coal`/`stone` и т.п. распались на отдельные записи.
- [ ] Перевести `GameRegistry`-регистрацию на `DeferredRegister` или `RegistryEvent`.
- [ ] Заменить `OreDictionary` на теги; проверить рецепты.
- [ ] Переписать `EntityRegistry.registerModEntity` на `EntityType` + `ForgeRegistries`.
- [ ] Заменить `IBlockState` на `BlockState`, `ITickable` на `ITickableTileEntity`.
- [ ] Обновить `EnumHand` → `Hand`, `EnumFacing` → `Direction`, `EnumActionResult` → `ActionResultType`.
- [ ] Перевести `mcmod.info` в `mods.toml`.
- [ ] Убрать `@SidedProxy`, перейти на `DistExecutor`.
- [ ] Переписать рендер на `MatrixStack` (1.15+).
- [ ] Перегенерировать refmap; проверить `@Shadow`/`@At` под новые MCP-имена.
- [ ] Если Mixin нужен на 1.13/1.14 — подключить `MixinBootstrap` и проверить версию Forge.

> [!WARNING]
> Не тащите `assets/`-рецепты в 1.13+. Рецепт, который лежал в `assets/mymod/recipes/foo.json`, после 1.13 должен лежать в `data/mymod/recipes/foo.json` и иметь корректный `type`. Иначе он просто не загрузится, без внятной ошибки.

---

## Проверено на

- **Дата проверки:** 2026-11-21.
- **Версии:** Forge 1.13.2-25.0.223, 1.14.4-28.2.30, 1.15.2-31.2.62, 1.16.5-36.2.42.
- **Что проверено:** даты релизов, версии Forge, наличие `DeferredRegister` (нет в 1.13.x, есть в 1.14.x), `ForgeRegistries` (1.13+), `TileEntityType` (документация 1.13+), `ITickableTileEntity` (1.14+), `@SidedProxy` (удалён в 1.13), `DistExecutor` (1.13+), Mixin в Forge (1.15.2-31.2.44+, 1.16.1-32.0.72+).
- **Что помечено:** границы некоторых MCP-переименований (`PlayerEntity`, `LivingEntity`, `ActionResultType`) указаны диапазоном 1.13–1.16, потому что MCP-снапшоты менялись внутри эпохи.
- **Код:** фрагменты иллюстративные и не компилировались; сигнатуры сверялись с mappings и документацией указанных версий.

## Источники

1. Minecraft Wiki — Java Edition 1.13: <https://minecraft.wiki/w/Java_Edition_1.13>
2. Minecraft Wiki — Java Edition 1.14: <https://minecraft.wiki/w/Java_Edition_1.14>
3. Minecraft Wiki — Java Edition 1.15: <https://minecraft.wiki/w/Java_Edition_1.15>
4. Minecraft Wiki — Java Edition 1.16: <https://minecraft.wiki/w/Java_Edition_1.16>
5. Minecraft Wiki — Java Edition 1.16.5 (последняя с Java 8): <https://minecraft.wiki/w/Java_Edition_1.16.5>
6. Forge promotions: <https://files.minecraftforge.net/net/minecraftforge/forge/promotions_slim.json>
7. Forge docs 1.14.x — Registries (`DeferredRegister`, `RegistryEvent`): <https://docs.minecraftforge.net/en/1.14.x/concepts/registries/>
8. Forge docs 1.14.x — Porting from 1.12 to 1.13/1.14: <https://docs.minecraftforge.net/en/1.14.x/legacy/porting1214/>
9. Forge docs 1.14.x — Access Transformers: <https://docs.minecraftforge.net/en/1.14.x/advanced/accesstransformers/>
10. Forge 1.13.x `GameRegistry` (только `findRegistry`): <https://github.com/MinecraftForge/MinecraftForge/blob/1.13.x/src/main/java/net/minecraftforge/fml/common/registry/GameRegistry.java>
11. Forge 1.13.x `ForgeRegistries`: <https://github.com/MinecraftForge/MinecraftForge/blob/1.13.x/src/main/java/net/minecraftforge/registries/ForgeRegistries.java>
12. Forge 1.14.x `DeferredRegister`: <https://github.com/MinecraftForge/MinecraftForge/blob/1.14.x/src/main/java/net/minecraftforge/registries/DeferredRegister.java>
13. MixinBootstrap README (совместимость 1.13/1.14, версии Forge с Mixin): <https://github.com/LXGaming/MixinBootstrap>
14. Mixin wiki — Mixins on Minecraft Forge: <https://github.com/SpongePowered/Mixin/wiki/Mixins-on-Minecraft-Forge>
15. Forge 1.16.5 javadoc, community-зеркало (`ITickableTileEntity`, `MatrixStack`, `PlayerEntity`): <https://nekoyue.github.io/ForgeJavaDocs-NG/javadoc/1.16.5/>
16. Fabric Documentation — Yarn, Loom и Access Wideners: <https://docs.fabricmc.net/develop/>
