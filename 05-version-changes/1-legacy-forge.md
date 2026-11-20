# Эпоха 1: legacy Forge (1.7.10–1.12.2)

[← К оглавлению](00-index.md)

> [!NOTE]
> **TL;DR.** Это эпоха MCP-имён, SRG в рантайме, `cpw.mods.fml`, coremod'ов на ASM и `GameRegistry`. Mixin в Forge здесь ещё **нет**: на 1.12.2 его подключали отдельным бутстрапом, а на 1.7.10 его не было вообще. Если вы портируете мод отсюда, вы не «переименовываете классы», а переписываете регистрацию, блокстейты, рендер и рецепты.

## Содержание

- [Контекст эпохи](#контекст-эпохи)
  - [Хронология](#хронология)
  - [Чем эпоха отличалась](#чем-эпоха-отличалась)
  - [Mappings и SRG](#mappings-и-srg)
- [Главные breaking changes](#главные-breaking-changes)
  - [Сводная таблица](#сводная-таблица)
  - [Регистрация: три поколения API](#регистрация-три-поколения-api)
  - [1.8: блоки, состояния и модели](#18-блоки-состояния-и-модели)
  - [1.9: руки и предметы](#19-руки-и-предметы)
  - [1.10–1.12: реестры, сущности, JSON-данные](#110112-реестры-сущности-json-данные)
- [Что это значит для миксинов](#что-это-значит-для-миксинов)
  - [Цели, которые исчезли или переехали](#цели-которые-исчезли-или-переехали)
  - [Доступность членов и `@Shadow`](#доступность-членов-и-shadow)
  - [`@At` и дескрипторы](#at-и-дескрипторы)
  - [Refmap и SRG](#refmap-и-srg)
  - [Пример миксина для 1.12.2](#пример-миксина-для-1122)
- [Как жил моддинг](#как-жил-моддинг)
  - [Coremods и ASM-трансформеры](#coremods-и-asm-трансформеры)
  - [`@Mod`, прокси и жизненный цикл](#mod-прокси-и-жизненный-цикл)
  - [Access Transformers](#access-transformers)
  - [Когда появился Mixin](#когда-появился-mixin)
- [Практический вывод](#практический-вывод)
  - [Что переписывать в первую очередь](#что-переписывать-в-первую-очередь)
  - [Чеклист порта](#чеклист-порта)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Контекст эпохи

### Хронология

Эпоха начинается с **1.7.10 (26 июня 2014)** и заканчивается **1.12.2 (18 сентября 2017)**. Формально. Фактически 1.12.2 оставалась главной моддинговой версией ещё несколько лет: Forge 14.23.5.2864 выходил уже после релиза 1.13, а крупные сборки вроде GTNH живут на 1.7.10 до сих пор.

| Версия | Дата релиза | Java | Лоадеры | Сборка | Mappings |
| --- | --- | --- | --- | --- | --- |
| 1.7.10 | 2014-06-26 | 8 | Forge 10.13.4.1614, LiteLoader | ForgeGradle 1.2; сегодня RFG / Unimined | MCP stable_12, SRG в рантайме |
| 1.8 | 2014-09-02 | 8 | Forge 11.14.4.1577 | ForgeGradle 2.x | MCP, SRG |
| 1.8.9 | 2015-12-09 | 8 | Forge 11.15.1.2318, LiteLoader | ForgeGradle 2.x | MCP, SRG |
| 1.9 | 2016-02-29 | 8 | Forge 12.16.1.1938 | ForgeGradle 2.x | MCP, SRG |
| 1.9.4 | 2016-05-10 | 8 | Forge 12.17.0.2317 | ForgeGradle 2.x | MCP, SRG |
| 1.10.2 | 2016-06-23 | 8 | Forge 12.18.3.2511 | ForgeGradle 2.x | MCP, SRG |
| 1.11.2 | 2016-12-21 | 8 | Forge 13.20.1.2588 | ForgeGradle 2.x | MCP, SRG |
| 1.12 | 2017-06-07 | **Java 8 обязательна** | Forge 14.21.1.2443 | ForgeGradle 2.x | MCP, SRG |
| 1.12.2 | 2017-09-18 | Java 8 | Forge 14.23.5.2864 | ForgeGradle 2.x; сегодня RFG | MCP stable_39, SRG |

> [!IMPORTANT]
> **Java 8 — версия моддинг-стека для всей эпохи** (так зафиксировано в каноне репозитория). При этом 1.12 — первая версия, которая **требует** Java 8: 1.7.10–1.11.2 запускались и на Java 7, хотя моды к 2016 году уже собирали под 8. Для порта это важно: `compatibilityLevel` старых миксинов — `JAVA_8`.

### Чем эпоха отличалась

Три вещи определяют эту эпоху:

1. **MCP-имена.** В исходниках вы видите `World`, `TileEntity`, `EntityPlayer`, `IBlockState`, `EnumHand`. В рантайме — SRG-имена вида `func_180495_p`, `field_70170_p`.
2. **LaunchWrapper и coremods.** Forge грузил моды через `LaunchWrapper`; до 1.13 не было `ModLauncher`. Вмешательство в ваниль делалось через `IFMLLoadingPlugin` + `IClassTransformer` + ASM.
3. **Один большой jar и `mcmod.info`.** Мод — это jar в `mods/`, метаданные — `mcmod.info`, конфиг — `@Mod` + `@SidedProxy`.

Ещё одна особенность: **1.7.10 и 1.8 — это разные API настолько, что это фактически два разных моддинга**. Пакет `cpw.mods.fml` стал `net.minecraftforge.fml`, `ForgeDirection` исчез, метаданные блоков уступили место `IBlockState`. Внутри эпохи тоже было три волны: 1.8 (blockstates и JSON-модели), 1.9 (offhand и `EnumHand`), 1.10–1.12 (реестры, `ItemStack.EMPTY`, JSON-рецепты).

### Mappings и SRG

| Слой | Что это | Пример |
| --- | --- | --- |
| MCP | читаемые имена для разработки | `World`, `getBlockState` |
| SRG | стабильные имена для рантайма | `bwp`, `func_180495_p` |
| Obfuscated | имена Mojang в релизном jar | `bwp`, `a` |

Мод компилируется против MCP, а в production-jar ForgeGradle ремапит ссылки в SRG. Поэтому:

- `@Shadow` и `@At` в исходнике пишутся MCP-именами;
- annotation processor Mixin'а генерирует **refmap**, который переводит MCP → SRG;
- если refmap потерян или устарел, миксин «не находит» метод, хотя в исходнике всё компилируется.

Для 1.12.2 актуальны MCP `stable_39`; для 1.7.10 — `stable_12`. Точные имена каналов зависят от проекта, поэтому в старых сборках их часто фиксируют явно в `build.gradle`.

---

## Главные breaking changes

### Сводная таблица

| # | Было | Стало | Версия | Почему это важно |
| --- | --- | --- | --- | --- |
| 1 | `cpw.mods.fml.*` | `net.minecraftforge.fml.*` | 1.8 | Меняются все импорты `@Mod`, `Side`, `@SideOnly`, событий |
| 2 | `ForgeDirection` | `EnumFacing` | 1.8 | Другая константа (`UNKNOWN` исчез), другой API `getOpposite()` |
| 3 | `World#getBlock(int x, int y, int z)` | `World#getBlockState(BlockPos)` | 1.8 | Координаты ушли в `BlockPos`; блок и метаданные слились в `IBlockState` |
| 4 | `World#setBlock(x, y, z, block, meta)` | `World#setBlockState(BlockPos, IBlockState)` | 1.8 | Метаданные больше не передаются числом |
| 5 | `IIcon` + `registerIcons(IIconRegister)` | JSON-модели + `TextureAtlasSprite` | 1.8 | Иконки в коде заменены `models/block/*.json`, `models/item/*.json` |
| 6 | `GameRegistry.registerBlock(block, "name")` | `setRegistryName(...)` + `GameRegistry.register(...)` | 1.8–1.9 | `setRegistryName` появился в 1.8, обобщённый `register` — в 1.9; реестр требует `ResourceLocation` |
| 7 | `GameRegistry.register(...)` | `RegistryEvent.Register<T>` | 1.10 | `IForgeRegistry` появился в 1.9, событие — в 1.10 |
| 8 | `Item#onItemRightClick(ItemStack, World, EntityPlayer)` | `onItemRightClick(World, EntityPlayer, EnumHand)` | 1.9 | Появилась вторая рука; состав и порядок аргументов изменились |
| 9 | `TileEntity#updateEntity()` | `ITickable#update()` | 1.8 | Тик переехал в интерфейс; в 1.14 интерфейс стал `ITickableTileEntity` |
| 10 | `EntityRegistry.registerModEntity(String, ...)` | `registerModEntity(ResourceLocation, ...)` | 1.11 | Строковое имя заменено на `ResourceLocation` |
| 11 | `null` как пустой стак | `ItemStack.EMPTY` + `isEmpty()` | 1.11 `[unverified]` | К 1.12 это уже норма; проверки `stack == null` перестали быть корректными |
| 12 | Рецепты в коде (`CraftingManager`) | JSON-рецепты + `IRecipe`/`RecipeSorter` | 1.12 | Рецепты и advancements стали данными, а не кодом |
| 13 | `@Mod.EventHandler` + `init` | `@Mod.EventBusSubscriber` для статических обработчиков | 1.10 | Меньше ручной регистрации в `MinecraftForge.EVENT_BUS` |

**Сразу за границей эпохи (1.13), чтобы не потерять при порте:**

| Было | Стало | Версия | Почему важно |
| --- | --- | --- | --- |
| `IChatComponent` | `ITextComponent` | 1.13 | В 1.12 ещё `IChatComponent`; ломается на следующем шаге |
| `GameRegistry.registerTileEntity(Class, String)` | `TileEntityType` + `RegistryEvent` | 1.13 | В 1.12 ещё старый метод; исчезает вместе с `GameRegistry` |

> [!WARNING]
> Границы эпохи не совпадают с границами «одного перехода». 1.13 меняет практически всё, что описано в этой таблице, но относится уже к следующей эпохе. Если вы портируете **с 1.12.2 сразу на 1.16.5 или 1.20.1**, читайте эту таблицу вместе со следующим файлом: часть пунктов (flattening, `BlockState`, `DeferredRegister`) произойдёт «по дороге».

### Регистрация: три поколения API

Это самый показательный пример эволюции.

**1.7.10 — строки и `GameRegistry`:**

```java
// 1.7.10
public static Block myBlock;

@Mod.EventHandler
public void preInit(FMLPreInitializationEvent event) {
    myBlock = new MyBlock();
    GameRegistry.registerBlock(myBlock, "my_block");
    GameRegistry.registerTileEntity(MyTileEntity.class, "my_tile");
}
```

**1.8–1.11 — `setRegistryName` и постепенный отказ от helper'ов.** В 1.8.9 методы `registerBlock`/`registerItem` ещё существуют и всё ещё нужны: объект уже знает своё имя, но регистрируется через helper. С 1.9 появился обобщённый `GameRegistry.register(...)`.

```java
// 1.8.9
myBlock = new MyBlock();
myBlock.setRegistryName(new ResourceLocation("my_mod", "my_block"));
GameRegistry.registerBlock(myBlock);

// 1.9.4+
myBlock.setRegistryName(new ResourceLocation("my_mod", "my_block"));
GameRegistry.register(myBlock);
```

**1.10–1.12 — событие реестра.** С 1.10 (Forge 12.18) правильная точка регистрации — `RegistryEvent.Register<T>`. С 1.11 helper'ы `registerBlock`/`registerItem` удалены, остаётся `GameRegistry.register(entry)` и событие.

```java
// 1.12.2
@Mod.EventBusSubscriber(modid = "mymod")
public class Registration {
    @SubscribeEvent
    public static void registerBlocks(RegistryEvent.Register<Block> event) {
        event.getRegistry().register(new MyBlock().setRegistryName("my_mod", "my_block"));
    }
}
```

| Поколение | Версия | Точка входа | Имя объекта |
| --- | --- | --- | --- |
| helper-API | 1.7.10 | `GameRegistry.registerBlock(block, "name")` | строка |
| `setRegistryName` | 1.8–1.9 | `registerBlock`/`registerItem` (1.8), `register` (1.9) | `ResourceLocation` |
| `RegistryEvent` | 1.10 | `RegistryEvent.Register<Block>` | `ResourceLocation` |
| `DeferredRegister` | 1.14 | `DeferredRegister.create(...)` | `ResourceLocation` |

### 1.8: блоки, состояния и модели

**Метаданные → `IBlockState`.** В 1.7.10 вариант блока — это `int meta`:

```java
// 1.7.10
world.setBlock(x, y, z, Blocks.wool, 14, 3); // красная шерсть
int meta = world.getBlockMetadata(x, y, z);
```

В 1.8 появился `IBlockState`, `BlockStateContainer` и `Property`:

```java
// 1.8+
IBlockState state = Blocks.WOOL.getDefaultState()
        .withProperty(BlockColored.COLOR, EnumDyeColor.RED);
world.setBlockState(pos, state, 3);
```

Это ещё **не flattening**: в 1.8–1.12 разные варианты одного блока всё ещё живут в одном блоке с property/metadata. Flattening с отдельными `red_wool` и `granite` случится в 1.13.

**Иконки → JSON-модели.** В 1.7.10 предмет рисовал себя через `IIcon`:

```java
// 1.7.10
@Override
public void registerIcons(IIconRegister register) {
    this.icon = register.registerIcon("mymod:my_item");
}
```

В 1.8 это заменили модели: `assets/<ns>/models/item/my_item.json`, `assets/<ns>/textures/items/my_item.png`. В коде остаются `ModelResourceLocation`, `ItemModelMesher`, `ModelBakeEvent`. Любой миксин в `ItemRenderer`/`RenderItem` после такого переезда ищет уже другие методы.

**Координаты.** `x, y, z` во всех API заменились на `BlockPos`. Это не косметика: `BlockPos` — immutable-объект, и сигнатуры методов поменялись во всей иерархии `Block`, `World`, `TileEntity`.

### 1.9: руки и предметы

1.9 добавила offhand и `EnumHand`. Сигнатуры стали длиннее:

```java
// 1.8
public ActionResult<ItemStack> onItemRightClick(ItemStack stack, World world, EntityPlayer player)

// 1.9+
public ActionResult<ItemStack> onItemRightClick(World world, EntityPlayer player, EnumHand hand)
```

Следствия:

- любой `@Inject` в `onItemRightClick` для 1.8 не совпадёт с 1.9-дескриптором;
- появились `EnumHand.MAIN_HAND` / `OFF_HAND`, `EntityEquipmentSlot`, `ItemStack#getAttributeModifiers(slot, stack)`;
- в 1.12.2 вы всё ещё видите `EnumHand`; в 1.14 MCP переименует его в `Hand`; в 1.17 Mojmap — в `InteractionHand`.

Также в 1.9 усложнился `Block#onBlockActivated`: к `World`, `BlockPos`, `IBlockState`, `EntityPlayer` добавились `EnumHand` и `EnumFacing`/хит-вектор в зависимости от версии. Для миксинов это означает, что дескриптор `@At` нельзя переносить между 1.8 и 1.9 без проверки.

### 1.10–1.12: реестры, сущности, JSON-данные

**1.9–1.10.** В 1.9 появился `IForgeRegistry` и реестровая переделка; в 1.10 — `RegistryEvent.Register<T>`. Это позволило отказаться от статической инициализации и `GameRegistry.register` в `preInit`.

**1.11.** `EntityRegistry.registerModEntity` перешёл на `ResourceLocation`:

```java
// 1.10 и раньше
EntityRegistry.registerModEntity(MyEntity.class, "my_entity", 0, this, 64, 1, true);

// 1.11+
EntityRegistry.registerModEntity(
        new ResourceLocation("mymod", "my_entity"),
        MyEntity.class, "my_entity", 0, this, 64, 1, true);
```

Там же, по распространённым портам, закрепился `ItemStack.EMPTY` вместо `null` `[unverified]`. В 1.12.2 `isEmpty()` — обычная проверка, а `stack == null` уже считается ошибкой.

**1.12.** Рецепты и advancements стали JSON-данными, появился `RecipeSorter` и реестр `IRecipe`. Это первый шаг к тому, что в 1.13 станет обязательным форматом `data/<ns>/recipes/*.json`.

```json
{
  "type": "crafting_shaped",
  "pattern": ["XXX", "X X", "XXX"],
  "key": { "X": { "item": "minecraft:iron_ingot" } },
  "result": { "item": "mymod:my_item" }
}
```

> [!NOTE]
> В 1.12 рецепты ещё лежат в `assets/`, а в 1.13 переезжают в `data/`. Если вы видите в старом моде `assets/mymod/recipes/`, это 1.12-стиль; для 1.13+ путь нужно менять.

---

## Что это значит для миксинов

### Цели, которые исчезли или переехали

| Цель 1.7.10 | Цель 1.12.2 | Что произошло |
| --- | --- | --- |
| `net.minecraft.world.World` | `net.minecraft.world.World` | Класс тот же, но методы `getBlock`/`setBlock` заменены на `getBlockState`/`setBlockState` |
| `net.minecraft.tileentity.TileEntity#updateEntity` | `ITickable#update` | Тик вынесен в интерфейс в 1.8 |
| `net.minecraft.block.Block#onBlockActivated(int x, int y, int z, ...)` | `onBlockActivated(World, BlockPos, IBlockState, EntityPlayer, EnumHand, ...)` | `BlockPos` в 1.8, `EnumHand` в 1.9 |
| `net.minecraft.item.Item#onItemRightClick(ItemStack, World, EntityPlayer)` | `onItemRightClick(World, EntityPlayer, EnumHand)` | 1.9 |
| `net.minecraft.entity.player.EntityPlayer` | `EntityPlayer` | В 1.16.5 MCP станет `PlayerEntity`, в 1.17 — `Player` |
| `ForgeDirection` | `EnumFacing` | `ForgeDirection` удалён в 1.8 |
| `net.minecraft.client.renderer.texture.IIconRegister` | — | Удалён в 1.8 вместе с `IIcon` |
| `net.minecraft.entity.player.EntityPlayer#getHeldItem()` | `getHeldItem(EnumHand)` | Вторая рука, 1.9 |

Практический вывод: миксин, написанный под 1.7.10, **не компилируется** под 1.12.2 не из-за «обфускации», а из-за реально других сигнатур. Это две разные эпохи API внутри одной эпохи MCP.

### Доступность членов и `@Shadow`

В этой эпохе Mojang ещё не открывала исходники, а MCP давала имена, но не меняла модификаторы доступа. Поэтому:

- `TileEntity#world`, `TileEntity#pos` — `protected`, доступны через `@Shadow` без `@Accessor`;
- `TileEntity#tileEntityInvalid` — `protected boolean`, читается/пишется через `@Shadow`;
- `ItemStack#stackSize` — `private`, нужен `@Accessor` или публичный `getCount()`;
- `Item#maxStackSize` — `private`, нужен `@Accessor` или `getItemStackLimit()`;
- `Entity#world` — `public`, `@Shadow` работает напрямую;
- `World#isRemote` — `public final boolean`; читать можно, писать — только `@Mutable` (и это почти всегда ошибка).

> [!WARNING]
> `@Mutable` на `final`-поле в 1.12.2 — частая причина «миксин применился, но значение не изменилось»: поле могло быть скопировано JIT'ом или перезаписано конструктором. Если нужен предсказуемый доступ, предпочитайте `@Accessor` и вызов метода, а не запись в `final`.

### `@At` и дескрипторы

Дескрипторы в 1.12.2 выглядят так:

```java
@At(
    value = "INVOKE",
    target = "Lnet/minecraft/world/World;getBlockState(Lnet/minecraft/util/math/BlockPos;)Lnet/minecraft/block/state/IBlockState;"
)
```

Типичные ошибки при порте с 1.7.10:

1. **Старый owner.** `Lnet/minecraft/world/World;getBlock(III)Lnet/minecraft/block/Block;` в 1.8+ не существует.
2. **Старый return type.** `setBlockState` возвращает `boolean`, а не `void`; `@At` с неверным return type не матчится.
3. **Интерфейс вместо класса.** В 1.12.2 `IBlockState` — интерфейс; в дескрипторе он указывается как `Lnet/minecraft/block/state/IBlockState;`.
4. **SRG в исходнике.** Если вы копируете `target` из crash-report, там будут `func_...`; в исходнике нужны MCP-имена, иначе refmap не сгенерируется корректно.

### Refmap и SRG

Схема для 1.12.2:

```text
исходник (MCP) --javac--> class-файл с MCP-именами
       |
       +-- Mixin AP --> mixins.mymod.refmap.json (MCP -> SRG)
       |
       +-- ForgeGradle reobf --> production jar (SRG)
```

В `mixins.mymod.json` обязательно должен быть указан `refmap`, иначе в production миксин будет искать `getBlockState` в классе, где метод называется `func_180495_p`.

```json
{
  "required": true,
  "minVersion": "0.7",
  "package": "com.example.mymod.mixin",
  "compatibilityLevel": "JAVA_8",
  "refmap": "mixins.mymod.refmap.json",
  "mixins": ["WorldMixin"],
  "injectors": { "defaultRequire": 1 }
}
```

### Пример миксина для 1.12.2

```java
package com.example.mymod.mixin;

import net.minecraft.block.state.IBlockState;
import net.minecraft.util.math.BlockPos;
import net.minecraft.world.World;
import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.injection.At;
import org.spongepowered.asm.mixin.injection.Inject;
import org.spongepowered.asm.mixin.injection.callback.CallbackInfoReturnable;

@Mixin(World.class)
public class WorldMixin {

    @Inject(
        method = "setBlockState(Lnet/minecraft/util/math/BlockPos;Lnet/minecraft/block/state/IBlockState;I)Z",
        at = @At("HEAD"),
        cancellable = true
    )
    private void mymod$beforeSetBlock(BlockPos pos, IBlockState state, int flags,
                                      CallbackInfoReturnable<Boolean> cir) {
        // ...
    }
}
```

Обратите внимание: в `method` используется **полный дескриптор**, а не только имя. В 1.12.2 это особенно важно: у `World` есть перегрузки `setBlockState` с разным числом аргументов.

---

## Как жил моддинг

### Coremods и ASM-трансформеры

До появления Mixin единственным «официальным» способом изменить ванильный байткод был **coremod**:

```java
// 1.7.10 / 1.12.2
@IFMLLoadingPlugin.MCVersion("1.12.2")
@IFMLLoadingPlugin.TransformerExclusions({"com.example.mymod.asm"})
public class MyLoadingPlugin implements IFMLLoadingPlugin {

    @Override
    public String[] getASMTransformerClass() {
        return new String[]{"com.example.mymod.asm.MyTransformer"};
    }

    @Override
    public String getModContainerClass() { return null; }

    @Override
    public String getSetupClass() { return null; }

    @Override
    public void injectData(Map<String, Object> data) { }

    @Override
    public String getAccessTransformerClass() { return null; }
}
```

Трансформер реализовывал `IClassTransformer` и вручную работал с `ClassNode`/`MethodNode`:

```java
public class MyTransformer implements IClassTransformer {
    @Override
    public byte[] transform(String name, String transformedName, byte[] basicClass) {
        if (!"net.minecraft.world.World".equals(transformedName)) return basicClass;
        ClassReader reader = new ClassReader(basicClass);
        ClassNode node = new ClassNode();
        reader.accept(node, 0);
        // ... правка node ...
        ClassWriter writer = new ClassWriter(ClassWriter.COMPUTE_MAXS);
        node.accept(writer);
        return writer.toByteArray();
    }
}
```

Подключение: `META-INF/MANIFEST.MF` с `FMLCorePlugin: com.example.mymod.MyLoadingPlugin`. Если в этом же jar лежит обычный мод, добавляется `FMLCorePluginContainsFMLMod: true`; иначе трансформер можно держать в отдельном jar.

Почему это было больно:

- трансформер видит **обфусцированные/SRG-имена**, а не MCP;
- нет `@Shadow`/`@Accessor` — доступ к приватным членам только через ASM;
- легко сломать другой мод, который тоже трансформирует тот же класс;
- отладка — только `-Dfml.coreMods.load` и дампы байткода.

Именно из этой боли вырос Mixin: декларативные аннотации, автоматический ремап, кооперативные инъекции.

### `@Mod`, прокси и жизненный цикл

Классический мод 1.12.2:

```java
@Mod(modid = MyMod.MODID, name = "My Mod", version = "1.0.0",
     acceptedMinecraftVersions = "[1.12,1.13)")
public class MyMod {

    public static final String MODID = "mymod";

    @Mod.Instance(MODID)
    public static MyMod instance;

    @SidedProxy(
        clientSide = "com.example.mymod.ClientProxy",
        serverSide = "com.example.mymod.CommonProxy"
    )
    public static CommonProxy proxy;

    @Mod.EventHandler
    public void preInit(FMLPreInitializationEvent event) {
        proxy.preInit(event);
    }

    @Mod.EventHandler
    public void init(FMLInitializationEvent event) {
        proxy.init(event);
        MinecraftForge.EVENT_BUS.register(this);
    }

    @Mod.EventHandler
    public void postInit(FMLPostInitializationEvent event) {
        proxy.postInit(event);
    }

    @Mod.EventHandler
    public void serverStarting(FMLServerStartingEvent event) {
        // ...
    }
}
```

Паттерн `IProxy` — не часть Forge, а соглашение сообщества:

| Класс | Где живёт | Что делает |
| --- | --- | --- |
| `CommonProxy` | обе стороны | регистрация, конфиг, общие события |
| `ClientProxy extends CommonProxy` | только клиент | рендер, клавиши, модели, GUI |
| `ServerProxy extends CommonProxy` | только сервер | серверные команды |

`@SidedProxy` подставлял нужный класс в зависимости от `Side`. В 1.13 аннотацию удалили; на её место пришёл `DistExecutor` (см. следующий файл). Но сам паттерн «common/client» жив до сих пор — просто собирается вручную.

> [!TIP]
> Если вы портируете мод с 1.12.2, не тащите `IProxy` «как есть». В 1.17+ `@SidedProxy` нет; вместо него — `DistExecutor.safeRunWhenOn(Dist.CLIENT, ...)` или `FMLEnvironment.dist`. Прокси-классы можно сохранить, но подключать их придётся явно.

### Access Transformers

AT существовали ещё до 1.8:

- **1.7.10–1.12.2:** coremod возвращает имя класса AT из `IFMLLoadingPlugin#getAccessTransformerClass()`; сам класс работает по тем же правилам, что и современный `accesstransformer.cfg`.
- **1.8+:** появился файл `META-INF/accesstransformer.cfg` и манифест-атрибут `FMLAT`, который указывает на него.

Пример AT для 1.12.2:

```text
# сделать public поле world у TileEntity
public net.minecraft.tileentity.TileEntity field_145850_b # world

# снять final с метода
public-f net.minecraft.world.World func_180495_p(Lnet/minecraft/util/math/BlockPos;)Lnet/minecraft/block/state/IBlockState;
```

Важно: **имена полей и методов — SRG**, а не MCP. Классы — обычные. Это правило не изменится до 26.1, поэтому AT-файлы выглядят «страшно» во всех версиях до деобфускации.

### Когда появился Mixin

| Версия | Mixin в Forge | Как подключали |
| --- | --- | --- |
| 1.7.10 | нет | только coremod/ASM |
| 1.8–1.12.2 | нет в составе Forge | `MixinBootstrap` (1.12.2), `MixinBooter` (1.8–1.12.2), либо shade в свой jar |
| 1.13 | нет | `MixinBootstrap` официально не поддерживает 1.13 из-за версии ModLauncher |
| 1.14 | нет | `MixinBootstrap` частично, только Forge 28.1.45+ |
| 1.15.2 | **да, с 31.2.44+** | встроен в Forge |
| 1.16.1 | **да, с 32.0.72+** | встроен в Forge |

То есть на вопрос «был ли Mixin в поздних сборках Forge 1.12.2» ответ — **нет**, его там не было. Forge 1.12.2-14.23.5.2864 (последняя) всё ещё требует внешний бутстрап. Официальная интеграция появилась только в конце 1.15.2 и в 1.16.1.

> [!IMPORTANT]
> На 1.12.2 Mixin работал через `LaunchWrapper`, а не через `ModLauncher`. Поэтому `MixinBootstrap` — это отдельный загрузчик, который надо ставить в `mods/` рядом с модом. В dev-окружении достаточно зависимости `org.spongepowered:mixin:0.8.5` и `MixinGradle`.

---

## Практический вывод

### Что переписывать в первую очередь

Если вы портируете мод **из этой эпохи** (1.7.10–1.12.2) в современную, порядок такой:

| Приоритет | Что | Почему сначала |
| --- | --- | --- |
| 1 | Регистрация (`GameRegistry` → `DeferredRegister`) | Без неё мод не загрузится; затрагивает все блоки/предметы |
| 2 | Блоки и метаданные (`meta` → blockstates/отдельные блоки) | Flattening 1.13 ломает ID, рецепты, модели, теги |
| 3 | Предметы и урон (`item damage` → отдельные предметы/компоненты) | Ломает рецепты, NBT, креативные вкладки |
| 4 | События и `@Mod`-класс | `@SidedProxy` исчез, `@Mod.EventHandler` заменён на конструктор + `IEventBus` |
| 5 | Рендер (`IIcon`/TESR → JSON-модели, `PoseStack`) | Меняется целиком, а не переименовывается |
| 6 | Рецепты, lang, loot tables, advancements | Переезд `assets/` → `data/`, новые ключи локализации |
| 7 | Миксины/coremods | Старые цели исчезли; лучше писать заново |

### Чеклист порта

- [ ] Заменить `cpw.mods.fml.*` на `net.minecraftforge.fml.*` (если порт на 1.8+).
- [ ] Убрать `GameRegistry.registerBlock/registerItem`; перейти на `setRegistryName` + событие/`DeferredRegister`.
- [ ] Переписать `IBlockState`/`meta` на blockstates; проверить каждый вариант блока.
- [ ] Заменить `IIcon`/`registerIcons` на JSON-модели и `TextureAtlasSprite`.
- [ ] Проверить сигнатуры `onItemRightClick`/`onBlockActivated` (появился `EnumHand`/`InteractionHand`).
- [ ] Заменить `ForgeDirection` на `Direction`/`EnumFacing`.
- [ ] Перевести рецепты и advancements в `data/<ns>/...`.
- [ ] Переписать `@SidedProxy` на `DistExecutor`/`FMLEnvironment.dist`.
- [ ] Переписать coremod-трансформеры в миксины (или в `ILaunchPluginService`, если без Mixin).
- [ ] Сгенерировать новый refmap; проверить, что `@Shadow` и `@At` используют имена целевой версии.
- [ ] Прогнать `-Dmixin.debug.export=true` и убедиться, что миксины применились, а не «тихо пропустились».

> [!WARNING]
> Порт 1.12.2 → 1.13+ — это **не порт, а переписывание**. Официальная документация Forge прямо говорит, что 1.13 из-за затянувшейся разработки почти никто не использовал, и моды прыгали сразу на 1.14. Планируйте не «день на переименования», а недели на блоки, предметы и данные.

---

## Проверено на

- **Дата проверки:** 2026-11-21.
- **Версии, на которые сверялись имена:** Forge 1.12.2-14.23.5.2864, Forge 1.7.10-10.13.4.1614.
- **Что проверено по первоисточникам:** даты релизов, требования Java, версии Forge, наличие/отсутствие Mixin, сигнатуры `GameRegistry`, `EntityRegistry`, `RegistryEvent`, `IFMLLoadingPlugin`, `DistExecutor`/`SidedProxy`.
- **Что помечено `[unverified]`:** точная версия появления `ItemStack.EMPTY`; отдельные даты минорных версий, не влияющие на выводы.
- **Код:** фрагменты иллюстративные и не компилировались; сигнатуры сверялись с mappings и документацией указанных версий.

## Источники

1. Minecraft Wiki — Java Edition 1.7.10: <https://minecraft.wiki/w/Java_Edition_1.7.10>
2. Minecraft Wiki — Java Edition 1.8: <https://minecraft.wiki/w/Java_Edition_1.8>
3. Minecraft Wiki — Java Edition 1.12: <https://minecraft.wiki/w/Java_Edition_1.12> (первая версия, требующая Java 8)
4. Minecraft Wiki — Java Edition 1.12.2: <https://minecraft.wiki/w/Java_Edition_1.12.2>
5. Forge promotions (официальные версии): <https://files.minecraftforge.net/net/minecraftforge/forge/promotions_slim.json>
6. Forge 1.8.9 `GameRegistry`: <https://github.com/MinecraftForge/MinecraftForge/blob/1.8.9/src/main/java/net/minecraftforge/fml/common/registry/GameRegistry.java>
7. Forge 1.10.x `RegistryEvent`: <https://github.com/MinecraftForge/MinecraftForge/blob/1.10.x/src/main/java/net/minecraftforge/event/RegistryEvent.java>
8. Forge 1.11.x `EntityRegistry`: <https://github.com/MinecraftForge/MinecraftForge/blob/1.11.x/src/main/java/net/minecraftforge/fml/common/registry/EntityRegistry.java>
9. Forge 1.12.2 `IFMLLoadingPlugin` (javadoc, community-зеркало): <https://nekoyue.github.io/ForgeJavaDocs-NG/javadoc/1.12.2/net/minecraftforge/fml/relauncher/IFMLLoadingPlugin.html>
10. MixinBootstrap README (Mixin доступен в Forge 1.15.2-31.2.44+ и 1.16.1-32.0.72+): <https://github.com/LXGaming/MixinBootstrap>
11. MixinBooter (Mixin для 1.8–1.12.2): <https://www.curseforge.com/minecraft/mc-mods/mixin-booter>
12. Mixin wiki — Mixins on Minecraft Forge: <https://github.com/SpongePowered/Mixin/wiki/Mixins-on-Minecraft-Forge>
13. Forge docs 1.14.x — Access Transformers (правило SRG-имён): <https://docs.minecraftforge.net/en/1.14.x/advanced/accesstransformers/>
14. Forge docs 1.14.x — Porting from 1.12 to 1.13/1.14: <https://docs.minecraftforge.net/en/1.14.x/legacy/porting1214/>
