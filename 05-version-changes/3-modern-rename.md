# Эпоха 3: Mojmap и большая переименовка (1.17–1.19.4)

[← К оглавлению](00-index.md)

> [!NOTE]
> **TL;DR.** 1.17 — это переход Forge на официальные mappings Mojang и первая большая волна переименований: `World` → `Level`, `TileEntity` → `BlockEntity`, `MatrixStack` → `PoseStack`, `Hand` → `InteractionHand`. 1.18 добавила Java 17, `saveAdditional` и `Holder`; 1.19.3 — `BuiltInRegistries` и `RegistryDataLoader`; 1.19.4 — `ItemDisplayContext`. NeoForge в этих версиях **не существует**.

## Содержание

- [Контекст эпохи](#контекст-эпохи)
  - [Хронология](#хронология)
  - [Что изменил переход на Mojmap](#что-изменил-переход-на-mojmap)
  - [Не NeoForge](#не-neoforge)
- [Главные breaking changes](#главные-breaking-changes)
  - [Сводная таблица](#сводная-таблица)
  - [1.17: первая волна переименований](#117-первая-волна-переименований)
  - [Ловушка `Container`](#ловушка-container)
  - [1.18: worldgen, `Holder`, `saveAdditional`](#118-worldgen-holder-saveadditional)
  - [1.19.3: `BuiltInRegistries` и реестры данных](#1193-builtinregistries-и-реестры-данных)
  - [1.19.4: `ItemDisplayContext`](#1194-itemdisplaycontext)
- [Что это значит для миксинов](#что-это-значит-для-миксинов)
  - [Цели: MCP 1.16.5 → Mojmap 1.17](#цели-mcp-1165--mojmap-117)
  - [Имена, которые изменились только на Forge](#имена-которые-изменились-только-на-forge)
  - [Доступность и `@Shadow`](#доступность-и-shadow)
  - [`@At` и дескрипторы](#at-и-дескрипторы)
  - [Refmap: Mojmap → SRG](#refmap-mojmap--srg)
  - [Пример миксина для 1.19.4](#пример-миксина-для-1194)
- [Как жил моддинг](#как-жил-моддинг)
  - [Forge 37–45](#forge-3745)
  - [Прокси, `DistExecutor` и `@EventBusSubscriber`](#прокси-distexecutor-и-eventbussubscriber)
  - [Fabric и Quilt](#fabric-и-quilt)
  - [Access Transformers против Access Wideners](#access-transformers-против-access-wideners)
  - [MixinExtras: конец эпохи](#mixinextras-конец-эпохи)
- [Практический вывод](#практический-вывод)
  - [Порядок порта 1.16.5 → 1.19.4](#порядок-порта-1165--1194)
  - [Чеклист](#чеклист)
- [Проверено на](#проверено-на)
- [Источники](#источники)

---

## Контекст эпохи

### Хронология

| Версия | Дата релиза | Java | Лоадеры | Сборка | Mappings |
| --- | --- | --- | --- | --- | --- |
| 1.17 | 2021-06-08 | **Java 16** | Forge 37.x, Fabric | ForgeGradle 5.x, Loom | Mojmap (compile) + SRG (runtime); Yarn на Fabric |
| 1.17.1 | 2021-07-06 | Java 16 | Forge 37.1.1, Fabric | ForgeGradle 5.x, Loom | Mojmap + SRG; Yarn |
| 1.18 | 2021-11-30 | **Java 17** | Forge 38.x, Fabric | ForgeGradle 6.x, Loom | Mojmap (+ Parchment); SRG; Yarn |
| 1.18.1 | 2021-12-10 | Java 17 | Forge 39.1.2, Fabric | ForgeGradle 6.x, Loom | Mojmap (+ Parchment); SRG; Yarn |
| 1.18.2 | 2022-02-28 | Java 17 | Forge 40.3.12, Fabric, Quilt | ForgeGradle 6.x, Loom | Mojmap (+ Parchment); SRG; Yarn |
| 1.19 | 2022-06-07 | Java 17 | Forge 41.1.0, Fabric, Quilt | ForgeGradle 6.x, Loom | Mojmap (+ Parchment); SRG; Yarn |
| 1.19.1 | 2022-07-27 | Java 17 | Forge 42.0.9, Fabric, Quilt | ForgeGradle 6.x, Loom | Mojmap (+ Parchment); SRG; Yarn |
| 1.19.2 | 2022-08-05 | Java 17 | Forge 43.5.2, Fabric, Quilt | ForgeGradle 6.x, Loom | Mojmap (+ Parchment); SRG; Yarn |
| 1.19.3 | 2022-12-07 | Java 17 | Forge 44.1.23, Fabric, Quilt | ForgeGradle 6.x, Loom | Mojmap (+ Parchment); SRG; Yarn |
| 1.19.4 | 2023-03-14 | Java 17 | Forge 45.4.5, Fabric, Quilt | ForgeGradle 6.x, Loom | Mojmap (+ Parchment); SRG; Yarn |

> [!IMPORTANT]
> **1.17 — не просто «новая версия».** Это смена всей системы имён для Forge-модов. До 1.16.5 вы писали `World`, `TileEntity`, `Hand`; с 1.17 пишете `Level`, `BlockEntity`, `InteractionHand`. При этом в рантайме всё ещё SRG, поэтому refmap не исчезает — он просто начинает переводить Mojmap → SRG.

### Что изменил переход на Mojmap

Официальная документация Forge 1.17 формулирует это прямо: «Forge uses Mojang's Official Mappings, or MojMaps, for the foreseeable future». Что это значит на практике:

| Аспект | 1.16.5 (MCP) | 1.17+ (Mojmap) |
| --- | --- | --- |
| Имена классов в IDE | `World`, `TileEntity`, `PlayerEntity` | `Level`, `BlockEntity`, `Player` |
| Имена методов | `getBlockState`, `setBlockState` | те же, но у переименованных владельцев |
| Параметры методов | часто `par1`, `par2` | часто `arg0`, `arg1` (Mojmap не даёт имён) |
| Javadoc | нет | нет |
| Рантайм | SRG (`func_...`) | SRG (`m_..._`) |
| Refmap | MCP → SRG | Mojmap → SRG |
| AT-файлы | SRG | SRG |

Mojmap даёт имена классов/методов/полей, но **не даёт имён параметров**. Поэтому в 1.17–1.19.4 в миксинах вы видите `arg0`, `arg1`, а не осмысленные `world`, `pos`. Именно эту проблему позже закрыл Parchment — слой параметров и javadoc'ов поверх Mojmap.

> [!WARNING]
> Не путайте «Mojmap в исходниках» и «деобфускацию». В 1.17–1.19.4 игра всё ещё обфусцирована в рантайме; Forge ремапит её в SRG. Полная деобфускация с официальными именами в рантайме появится только в 26.1. Поэтому `refmap` в этой эпохе обязателен.

### Не NeoForge

NeoForge — форк Forge, появившийся **после 1.20.1**. В эпоху 1.17–1.19.4 его нет:

- нет `neoforge.mods.toml`;
- нет `RegisterEvent` (в Forge здесь `RegistryEvent`);
- нет `IEventBus`-переименований NeoForge;
- нет 4-компонентных версий.

Если вы видите туториал «NeoForge 1.19.2» — это ошибка. Для 1.19.2 существует только Forge 43.x и Fabric.

---

## Главные breaking changes

### Сводная таблица

| # | Было (1.16.5, MCP) | Стало (1.17+, Mojmap) | Версия | Комментарий |
| --- | --- | --- | --- | --- |
| 1 | `net.minecraft.world.World` | `net.minecraft.world.level.Level` | 1.17 | `ServerWorld`/`ClientWorld` → `ServerLevel`/`ClientLevel`; `World` остаётся только в Yarn |
| 2 | `net.minecraft.tileentity.TileEntity` | `net.minecraft.world.level.block.entity.BlockEntity` | 1.17 | `TileEntityType` → `BlockEntityType`; пакет полностью другой |
| 3 | `com.mojang.blaze3d.matrix.MatrixStack` | `com.mojang.blaze3d.vertex.PoseStack` | 1.17 | Меняется и пакет, и имя |
| 4 | `Hand` | `net.minecraft.world.InteractionHand` | 1.17 | До 1.14 назывался `EnumHand` |
| 5 | `PlayerEntity` | `Player` | 1.17 | `ServerPlayerEntity` → `ServerPlayer`; до 1.15 MCP — `EntityPlayer` |
| 6 | `IInventory` | `Container` | 1.17 | Имя `Container` теперь занято интерфейсом инвентаря |
| 7 | `Container` (базовый класс меню) | `AbstractContainerMenu` | 1.17 | См. ловушку ниже |
| 8 | `ITileEntityProvider` | `EntityBlock` | 1.17 | `Block#newBlockEntity`, `getTicker` |
| 9 | `ITextComponent` | `Component` | 1.17 | `TextComponentString` → `TextComponent` |
| 10 | `Item#onItemRightClick`/`onItemUse` | `Item#use`/`useOn` | 1.17 | `Block#onBlockActivated` → `Block#use` |
| 11 | `INamedContainerProvider` | `MenuProvider` | 1.17 | GUI-меню |
| 12 | `IVertexBuilder` | `VertexConsumer` | 1.17 | Рендер-методы меняют тип |
| 13 | `BlockEntity#save` | `BlockEntity#saveAdditional` | 1.18 | Переопределяется `saveAdditional`; старого `save` в API больше нет |
| 14 | `Registry.BLOCK` (статика) | `BuiltInRegistries.BLOCK` | 1.19.3 | Статические реестры вынесены; там же `RegistryDataLoader` |
| 15 | `ItemTransforms.TransformType` | `ItemDisplayContext` | 1.19.4 | Рендер предметов в руках/GUI |

> [!NOTE]
> Таблица покрывает три независимых перехода: 1.16.5 → 1.17 (переименования), 1.17 → 1.18.2 (`saveAdditional`, `Holder`) и 1.18.2 → 1.19.4 (реестры, `ItemDisplayContext`). Для порта удобнее идти по ним последовательно, а не пытаться сделать всё за один проход.

### 1.17: первая волна переименований

**`World` → `Level`.** Это не просто переименование класса: изменился пакет (`net.minecraft.world` → `net.minecraft.world.level`), а вместе с ним — все импорты и дескрипторы.

> [!NOTE]
> В части старых гайдов и таблиц переименование `World` → `Level` датируют 1.18. Для **Forge/Mojmap** это происходит в 1.17: в 1.17 официальные mappings уже содержат `net.minecraft.world.level.Level`, а старое MCP-имя `net.minecraft.world.World` остаётся только в колонке MCP. Если вы видите 1.18 — уточняйте, о какой ветке mappings идёт речь.

```java
// 1.16.5
import net.minecraft.world.World;
import net.minecraft.world.server.ServerWorld;

public void doThing(World world) {
    if (world instanceof ServerWorld) { /* ... */ }
}

// 1.17+
import net.minecraft.world.level.Level;
import net.minecraft.server.level.ServerLevel;

public void doThing(Level level) {
    if (level instanceof ServerLevel serverLevel) { /* ... */ }
}
```

**`TileEntity` → `BlockEntity`.** Пакет `net.minecraft.tileentity` исчез; появился `net.minecraft.world.level.block.entity`. Заодно `TileEntityType` стал `BlockEntityType`.

```java
// 1.16.5
public class MyTileEntity extends TileEntity {
    public MyTileEntity(TileEntityType<?> type) { super(type); }

    @Override
    public void read(BlockState state, CompoundNBT nbt) { /* ... */ }

    @Override
    public CompoundNBT write(CompoundNBT nbt) { return super.write(nbt); }
}

// 1.17
public class MyBlockEntity extends BlockEntity {
    public MyBlockEntity(BlockEntityType<?> type, BlockPos pos, BlockState state) {
        super(type, pos, state);
    }

    @Override
    public void load(CompoundTag tag) { super.load(tag); }

    @Override
    public CompoundTag save(CompoundTag tag) { return super.save(tag); }
}
```

**`MatrixStack` → `PoseStack`.** Пакет `com.mojang.blaze3d.matrix` → `com.mojang.blaze3d.vertex`. В 1.18+ `save` у `BlockEntity` заменён на `saveAdditional`, но об этом ниже.

```java
// 1.16.5
import com.mojang.blaze3d.matrix.MatrixStack;

// 1.17+
import com.mojang.blaze3d.vertex.PoseStack;
```

**`EntityPlayer` → `Player`.** Заодно `EntityPlayerMP` → `ServerPlayer`, `PlayerEntity` (MCP 1.16.5) → `Player`.

**`EnumHand`/`Hand` → `InteractionHand`.** Цепочка за три эпохи:

| Версия | Имя |
| --- | --- |
| 1.9–1.12.2 | `EnumHand` |
| 1.14–1.16.5 | `Hand` |
| 1.17+ | `InteractionHand` |

**Методы взаимодействия.** В 1.17 сигнатуры стали короче и семантичнее:

```java
// 1.16.5
public ActionResultType onItemUse(ItemUseContext context);
public ActionResult<ItemStack> onItemRightClick(World world, PlayerEntity player, Hand hand);

// 1.17+
public InteractionResult useOn(UseOnContext context);
public InteractionResultHolder<ItemStack> use(Level level, Player player, InteractionHand hand);
```

`ActionResultType` → `InteractionResult`, `ActionResult<ItemStack>` → `InteractionResultHolder<ItemStack>`.

### Ловушка `Container`

Это самый опасный пункт при механическом порте. В 1.16.5:

- `IInventory` — интерфейс инвентаря;
- `Container` — **базовый класс меню** (`ContainerPlayer`, `ContainerChest` и т.д. наследуются от него).

В 1.17:

- `Container` — теперь **интерфейс инвентаря** (бывший `IInventory`);
- `AbstractContainerMenu` — базовый класс меню (бывший `Container`);
- `ContainerPlayer` → `InventoryMenu`, `ContainerChest` → `ChestMenu`.

Если вы портируете миксин и делаете `@Mixin(Container.class)`, рассчитывая на меню, вы попадёте в интерфейс инвентаря. Компилятор, скорее всего, промолчит: `Container` существует, просто это другой тип.

```java
// 1.16.5: цель — базовый класс меню
@Mixin(Container.class)
public class MenuMixin { /* ... */ }

// 1.17: НЕПРАВИЛЬНО — это интерфейс инвентаря
@Mixin(Container.class)
public class MenuMixin { /* ... */ }

// 1.17: ПРАВИЛЬНО — бывший Container
@Mixin(AbstractContainerMenu.class)
public class MenuMixin { /* ... */ }
```

> [!WARNING]
> Правило для 1.17: если в старом коде `Container` наследовался или инстанцировался — это `AbstractContainerMenu`. Если `Container` был типом параметра «инвентарь» — это новый интерфейс `Container`. Проверяйте каждый случай отдельно, а не заменяйте глобально.

### 1.18: worldgen, `Holder`, `saveAdditional`

1.18 — Java 17 и большой рефакторинг worldgen:

- `ChunkGenerator`, `BiomeSource`, `NoiseBasedChunkGenerator` получили новые сигнатуры и codec'и;
- `RegistryAccess` стал обычным способом получить реестры;
- `Holder<T>` появился как ссылка на элемент реестра, который может быть ещё не загружен;
- `BlockEntity#save` заменён на `saveAdditional(CompoundTag)`; в 1.18 старого `save` в `BlockEntity` уже нет.

```java
// 1.17: переопределяем save
@Override
public CompoundTag save(CompoundTag tag) {
    tag.putInt("energy", this.energy);
    return super.save(tag);
}

// 1.18+: переопределяем saveAdditional
@Override
protected void saveAdditional(CompoundTag tag) {
    tag.putInt("energy", this.energy);
    super.saveAdditional(tag);
}
```

Почему это важно для миксинов: в 1.18+ `@Inject` в `save` перестаёт находить цель, потому что такого метода в `BlockEntity` больше нет. Если вы хотели изменить NBT, цельтесь в `saveAdditional`; если хотели перехватить весь процесс — ищите публичный метод, который вызывает `saveAdditional` (например, `saveWithFullMetadata`/`saveWithoutMetadata`), и помните, что их сигнатуры тоже могли измениться.

`Holder` в 1.18 используется ещё умеренно; в 1.19 он становится вездесущим: `Holder<Biome>`, `HolderSet<T>`, `TagKey<T>`, `Registry#getHolder`.

### 1.19.3: `BuiltInRegistries` и реестры данных

До 1.19.2 статические реестры жили в `Registry`:

```java
// 1.19.2
Registry.BLOCK
Registry.ITEM
Registry.ENTITY_TYPE
```

В 1.19.3 их вынесли в отдельный класс:

```java
// 1.19.3+
BuiltInRegistries.BLOCK
BuiltInRegistries.ITEM
BuiltInRegistries.ENTITY_TYPE
```

Одновременно появился `RegistryDataLoader` (`net.minecraft.resources.RegistryDataLoader`) — загрузчик datapack-реестров. Для модов это означало, что кастомные реестры и datapack-записи (biomes, dimensions, damage types и т.п.) грузятся иначе, и миксины в `Registry`/`RegistryAccess` нужно перенацеливать.

Там же, в 1.19.3, креативные вкладки стали реестром:

```java
// 1.19.2
CreativeModeTab.TAB_BUILDING_BLOCKS

// 1.19.3+
CreativeModeTabs.BUILDING_BLOCKS
Registries.CREATIVE_MODE_TAB
```

### 1.19.4: `ItemDisplayContext`

1.19.4 переименовала `ItemTransforms.TransformType` в `ItemDisplayContext`. Это тип, который говорит, где рисуется предмет: в руке, в рамке, в GUI, на земле.

```java
// 1.19.3
import net.minecraft.world.item.ItemTransforms.TransformType;

// 1.19.4+
import net.minecraft.world.item.ItemDisplayContext;
```

Для миксинов в `ItemRenderer`/`BlockEntityWithoutLevelRenderer` это означает, что дескрипторы с `TransformType` перестают совпадать. Мелочь, но она ломает рендер-миксины, которые иначе бы «пережили» 1.19.3.

---

## Что это значит для миксинов

### Цели: MCP 1.16.5 → Mojmap 1.17

| 1.16.5 (MCP) | 1.17+ (Mojmap) | Комментарий |
| --- | --- | --- |
| `net.minecraft.world.World` | `net.minecraft.world.level.Level` | Меняется пакет |
| `net.minecraft.world.server.ServerWorld` | `net.minecraft.server.level.ServerLevel` | — |
| `net.minecraft.client.world.ClientWorld` | `net.minecraft.client.multiplayer.ClientLevel` | — |
| `net.minecraft.tileentity.TileEntity` | `net.minecraft.world.level.block.entity.BlockEntity` | — |
| `net.minecraft.tileentity.TileEntityType` | `net.minecraft.world.level.block.entity.BlockEntityType` | — |
| `net.minecraft.entity.player.PlayerEntity` | `net.minecraft.world.entity.player.Player` | — |
| `net.minecraft.entity.player.ServerPlayerEntity` | `net.minecraft.server.level.ServerPlayer` | — |
| `net.minecraft.entity.LivingEntity` | `net.minecraft.world.entity.LivingEntity` | Имя то же, пакет другой |
| `net.minecraft.util.Hand` | `net.minecraft.world.InteractionHand` | — |
| `net.minecraft.inventory.container.Container` | `net.minecraft.world.inventory.AbstractContainerMenu` | Ловушка! |
| `net.minecraft.inventory.IInventory` | `net.minecraft.world.Container` | Имя переиспользовано |
| `net.minecraft.block.BlockState` | `net.minecraft.world.level.block.state.BlockState` | Пакет |
| `com.mojang.blaze3d.matrix.MatrixStack` | `com.mojang.blaze3d.vertex.PoseStack` | Имя и пакет |
| `net.minecraft.util.text.ITextComponent` | `net.minecraft.network.chat.Component` | — |
| `net.minecraft.client.renderer.ActiveRenderInfo` | `net.minecraft.client.Camera` | — |
| `net.minecraft.client.renderer.entity.EntityRendererManager` | `net.minecraft.client.renderer.entity.EntityRenderDispatcher` | — |

> [!IMPORTANT]
> Это таблица только для **Forge/Mojmap**. На Fabric в 1.17–1.19.4 имена другие: `World` остаётся `World`, `MatrixStack` остаётся `MatrixStack`, `Hand` остаётся `Hand`. Если вы пишете кросс-лоадерный мод, у вас будет два набора целей для одних и тех же миксинов.

### Имена, которые изменились только на Forge

| Понятие | Forge/Mojmap 1.17+ | Fabric/Yarn 1.17+ |
| --- | --- | --- |
| Мир | `Level` | `World` |
| Блок-сущность | `BlockEntity` | `BlockEntity` |
| Стек трансформаций | `PoseStack` | `MatrixStack` |
| Рука | `InteractionHand` | `Hand` |
| Игрок | `Player` | `PlayerEntity` |
| Серверный игрок | `ServerPlayer` | `ServerPlayerEntity` |
| Текст | `Component` | `Text` |
| Базовое меню | `AbstractContainerMenu` | `ScreenHandler` |
| Инвентарь | `Container` | `Inventory` |

Поэтому фраза «в 1.17 `World` переименовали в `Level`» верна только для Forge-ветки. Fabric-моддеры продолжали писать `World` и не заметили этого переименования.

### Доступность и `@Shadow`

В 1.17 Mojmap сохранила модификаторы доступа, но имена полей стали другими. Проверенные примеры:

| Член | Модификатор (1.17) | Как достать |
| --- | --- | --- |
| `BlockEntity#level` | `protected Level` | `@Shadow` |
| `BlockEntity#worldPosition` | `protected final BlockPos` | `@Shadow @Final` (только чтение) |
| `BlockEntity#remove` | `protected boolean` | `@Shadow` |
| `BlockEntity#blockState` | `private BlockState` | `@Accessor` |
| `Entity#level` | `public Level` | `@Shadow` |
| `Level#isClientSide` | `public final boolean` | `@Shadow @Final`; запись — `@Mutable` |
| `ItemStack#count` | `private int` | `@Accessor` |
| `Player#inventory` | `private final Inventory` | `@Accessor` (в 1.16.5 было `public final PlayerInventory`) |

Пример:

```java
@Mixin(BlockEntity.class)
public abstract class BlockEntityMixin {

    @Shadow protected Level level;
    @Shadow @Final protected BlockPos worldPosition;
    @Shadow protected boolean remove;

    @Accessor("blockState")
    public abstract BlockState mymod$getBlockState();

    @Inject(method = "setRemoved", at = @At("HEAD"))
    private void mymod$onRemoved(CallbackInfo ci) {
        if (this.level != null && !this.level.isClientSide) {
            // ...
        }
    }
}
```

> [!WARNING]
> `@Shadow @Final` на `worldPosition` работает только для чтения. Если попытаться присвоить значение, Mixin выдаст ошибку или (при `@Mutable`) изменит final-поле, что почти наверняка сломает инварианты `BlockEntity`. Для позиции используйте `getBlockPos()`.

### `@At` и дескрипторы

**1.16.5:**

```java
@At(value = "INVOKE",
    target = "Lnet/minecraft/world/World;getBlockState(Lnet/minecraft/util/math/BlockPos;)Lnet/minecraft/block/BlockState;")
```

**1.17+:**

```java
@At(value = "INVOKE",
    target = "Lnet/minecraft/world/level/Level;getBlockState(Lnet/minecraft/core/BlockPos;)Lnet/minecraft/world/level/block/state/BlockState;")
```

Изменились **три** части дескриптора: пакет владельца, пакет `BlockPos` и пакет `BlockState`. Именно поэтому механическая замена `World` → `Level` в строке не помогает: нужно переписать весь descriptor.

Аналогично для `setBlock`:

```java
// 1.17+
@Inject(
    method = "setBlock(Lnet/minecraft/core/BlockPos;Lnet/minecraft/world/level/block/state/BlockState;I)Z",
    at = @At("HEAD")
)
```

`Level#setBlock` имеет две перегрузки — с тремя и четырьмя аргументами. Если указать только имя `setBlock`, Mixin может выбрать не ту; всегда указывайте полный дескриптор или используйте `@At` с дополнительными условиями.

### Refmap: Mojmap → SRG

Схема 1.17–1.19.4:

```text
исходник (Mojmap) --javac--> class с Mojmap-именами
       |
       +-- Mixin AP --> mixins.mymod.refmap.json (Mojmap -> SRG)
       |
       +-- ForgeGradle reobf --> production jar (SRG)
```

В рантайме crash-report показывает `m_XXXXX_`, а не `Level#getBlockState`. Это нормально: SRG-имена в 1.17+ имеют вид `m_<номер>_`, а не `func_<номер>_`.

```json
{
  "required": true,
  "minVersion": "0.8",
  "package": "com.example.mymod.mixin",
  "compatibilityLevel": "JAVA_17",
  "refmap": "mixins.mymod.refmap.json",
  "mixins": ["LevelMixin"],
  "injectors": { "defaultRequire": 1 }
}
```

`compatibilityLevel` для 1.17 — `JAVA_16`, для 1.18+ — `JAVA_17`. Если поставить `JAVA_8`, конфиг может не загрузиться или инъекции будут вести себя непредсказуемо: Mixin сверяет версию байткода с `compatibilityLevel`.

### Пример миксина для 1.19.4

```java
package com.example.mymod.mixin;

import net.minecraft.core.BlockPos;
import net.minecraft.world.level.Level;
import net.minecraft.world.level.block.state.BlockState;
import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.injection.At;
import org.spongepowered.asm.mixin.injection.Inject;
import org.spongepowered.asm.mixin.injection.callback.CallbackInfoReturnable;

@Mixin(Level.class)
public class LevelMixin {

    @Inject(
        method = "setBlock(Lnet/minecraft/core/BlockPos;Lnet/minecraft/world/level/block/state/BlockState;I)Z",
        at = @At("HEAD"),
        cancellable = true
    )
    private void mymod$beforeSetBlock(BlockPos pos, BlockState state, int flags,
                                      CallbackInfoReturnable<Boolean> cir) {
        // ...
    }
}
```

Конфиг попадает в рантайм через `MixinConfigs` в манифесте (MixinGradle), как и в 1.15/1.16. В 1.17+ это не изменилось.

---

## Как жил моддинг

### Forge 37–45

| Версия | Forge | Особенности |
| --- | --- | --- |
| 1.17.1 | 37.1.1 | Mojmap, Java 16, первый релиз новой эпохи |
| 1.18.2 | 40.3.12 | Java 17, worldgen, `Holder`, `saveAdditional` |
| 1.19.2 | 43.5.2 | `Holder`/`HolderSet`, чат-подпись, реестры |
| 1.19.3 | 44.1.23 | `BuiltInRegistries`, `CreativeModeTabs`, `RegistryDataLoader` |
| 1.19.4 | 45.4.5 | `ItemDisplayContext`, последняя версия перед 1.20 |

Forge всё ещё использует ModLauncher и SRG в рантайме. Mixin встроен с 1.15.2/1.16.1 и продолжает работать; refmap обязателен.

### Прокси, `DistExecutor` и `@EventBusSubscriber`

`@SidedProxy` исчез ещё в 1.13. В 1.17–1.19.4 стандартный способ:

```java
@Mod("mymod")
public class MyMod {
    public MyMod() {
        IEventBus modBus = FMLJavaModLoadingContext.get().getModEventBus();
        modBus.addListener(this::commonSetup);
        DistExecutor.safeRunWhenOn(Dist.CLIENT, () -> ClientSetup::init);
    }

    private void commonSetup(FMLCommonSetupEvent event) { /* ... */ }
}
```

| Паттерн | 1.17–1.19.4 |
| --- | --- |
| Прокси | `DistExecutor.safeRunWhenOn` / `unsafeRunWhenOn` |
| Сторона | `FMLEnvironment.dist`, `Level#isClientSide` |
| События | `@Mod.EventBusSubscriber` + `@SubscribeEvent` |
| Клиентские классы | `@OnlyIn(Dist.CLIENT)` |
| Регистрация | `DeferredRegister` |

`IProxy` как паттерн всё ещё встречается в портах, но это уже «самодельная» архитектура: Forge не предоставляет для неё аннотации. Если вы пишете новый мод, `DistExecutor` + `@EventBusSubscriber` проще.

### Fabric и Quilt

Fabric в этой эпохе — полноценная альтернатива:

| Аспект | Forge | Fabric |
| --- | --- | --- |
| Mappings | Mojmap + SRG | Yarn + intermediary |
| Метаданные | `mods.toml` | `fabric.mod.json` |
| Точка входа | `@Mod` + bus | `ModInitializer` |
| Регистрация | `DeferredRegister` | `Registry.register` |
| Миксины | встроены | встроены |
| Доступ | AT (SRG) | AW (Yarn) |
| API | большой | тонкий + Fabric API |

Quilt появился в 2021–2022 как форк Fabric. Для 1.18.2/1.19.x он совместим с большинством Fabric-модов, но собственный API (Quilt Standard Libraries) в этой эпохе ещё молод. Если мод пишется на Quilt, миксины всё равно используют Yarn-имена.

```java
// Fabric 1.19.4
public class MyMod implements ModInitializer {
    public static final Item MY_ITEM = new Item(new Item.Settings());

    @Override
    public void onInitialize() {
        Registry.register(Registries.ITEM, new Identifier("mymod", "my_item"), MY_ITEM);
    }
}
```

> [!NOTE]
> Fabric-версия `Registries.ITEM` — это Yarn-имя; на Forge в 1.19.3+ аналогичный класс называется `BuiltInRegistries.ITEM`. Один и тот же концепт, разные имена. Не копируйте код между лоадерами без замены namespace.

### Access Transformers против Access Wideners

**Forge 1.17–1.19.4:**

```text
# META-INF/accesstransformer.cfg
public net.minecraft.world.level.block.entity.BlockEntity f_58857_ # level
```

- имена — **SRG** (`f_58857_`), не Mojmap;
- путь строго `META-INF/accesstransformer.cfg`;
- в dev-окружении путь задаётся в `build.gradle`.

**Fabric:**

```text
# mymod.accesswidener
accessWidener v1 named
accessible field net/minecraft/block/entity/BlockEntity world Lnet/minecraft/world/World;
```

Access Widener использует Yarn-имена и не зависит от SRG. Поэтому в кросс-лоадерном проекте у вас будет два файла доступа с разными именами для одного и того же поля.

### MixinExtras: конец эпохи

MixinExtras (LlamaLad7) появился в 2023 году, к концу этой эпохи. В 1.17–1.19.2 его не было; на 1.19.4 его можно было подключить отдельной зависимостью, но в Forge он не входил.

Это значит:

- в 1.17–1.19.2 доступны только классические инъекторы: `@Inject`, `@Redirect`, `@ModifyArg`, `@ModifyVariable`, `@ModifyConstant`, `@Overwrite`;
- `@WrapOperation`, `@ModifyExpressionValue`, `@ModifyReturnValue`, `@WrapWithCondition` — это уже MixinExtras, их в этой эпохе нет «из коробки»;
- если вы видите туториал с `@WrapOperation` для 1.19.2, скорее всего, автор подключил MixinExtras вручную или перепутал версию.

---

## Практический вывод

### Порядок порта 1.16.5 → 1.19.4

1. **Rename-pass 1.17.** `World` → `Level`, `TileEntity` → `BlockEntity`, `MatrixStack` → `PoseStack`, `Hand` → `InteractionHand`, `PlayerEntity` → `Player`.
2. **Ловушка `Container`.** Разделить «меню» (`AbstractContainerMenu`) и «инвентарь» (`Container`); переименовать `ContainerPlayer` → `InventoryMenu`.
3. **Взаимодействие.** `onItemRightClick`/`onItemUse` → `use`/`useOn`; `onBlockActivated` → `use`; `ActionResultType` → `InteractionResult`.
4. **BlockEntity.** `read`/`write` → `load`/`save` (1.17), затем `save` → `saveAdditional` (1.18).
5. **Меню.** `INamedContainerProvider` → `MenuProvider`.
6. **Реестры 1.19.3.** `Registry.BLOCK` → `BuiltInRegistries.BLOCK`; `CreativeModeTab` → `CreativeModeTabs`.
7. **Рендер 1.19.4.** `TransformType` → `ItemDisplayContext`.
8. **Миксины.** Обновить все дескрипторы (пакеты `core`, `world.level`, `world.level.block.state`), перегенерировать refmap, выставить `compatibilityLevel` (`JAVA_16`/`JAVA_17`).

### Чеклист

- [ ] Заменить `net.minecraft.world.World` на `net.minecraft.world.level.Level` во всех импортах, сигнатурах и `@At`.
- [ ] Заменить `TileEntity`/`TileEntityType` на `BlockEntity`/`BlockEntityType`.
- [ ] Заменить `MatrixStack` на `PoseStack` (и пакет `blaze3d.vertex`).
- [ ] Заменить `Hand`/`EnumHand` на `InteractionHand`.
- [ ] Разобрать каждый `Container`: меню → `AbstractContainerMenu`, инвентарь → `Container`.
- [ ] Перейти с `ITileEntityProvider` на `EntityBlock`.
- [ ] Заменить `ITextComponent` на `Component`, `TextComponentString` на `TextComponent`.
- [ ] Обновить `Item#use`/`useOn` и `Block#use`.
- [ ] В 1.18+ переопределять `saveAdditional`, а не `save`.
- [ ] В 1.19.3+ заменить статические `Registry.X` на `BuiltInRegistries.X`.
- [ ] В 1.19.4+ заменить `TransformType` на `ItemDisplayContext`.
- [ ] Перегенерировать refmap; проверить `compatibilityLevel` и `MixinConfigs`.
- [ ] Если мод кросс-лоадерный — проверить, что Fabric-ветка всё ещё использует `World`/`MatrixStack`/`Hand`.

> [!WARNING]
> Порт 1.16.5 → 1.17 выглядит как «переименовать классы», но на деле это переписывание всех дескрипторов в миксинах. Компилятор проверит только Java-код; строки `method = "..."` и `target = "..."` он не проверяет. После порта обязательно запустите игру с `-Dmixin.debug.export=true` и убедитесь, что каждый миксин применился.

> [!TIP]
> Если вам нужен 1.20.1 — это ещё одна небольшая остановка: `GuiGraphics`, `ItemStack` NBT ещё живы, компонентов нет. Если нужен 1.20.5+ — начинается эпоха Data Components, и половина работы с `ItemStack` переписывается заново. См. следующий файл серии.

---

## Проверено на

- **Дата проверки:** 2026-11-21.
- **Версии:** Forge 1.17.1-37.1.1, 1.18.2-40.3.12, 1.19.2-43.5.2, 1.19.3-44.1.23, 1.19.4-45.4.5.
- **Что проверено:** даты релизов, требования Java (1.17 — Java 16, 1.18+ — Java 17), переход Forge на Mojmap в 1.17, имена `Level`, `BlockEntity`, `PoseStack`, `InteractionHand`, `Component`, `EntityBlock`, `Camera`, `EntityRenderDispatcher`, `VertexConsumer`, `MenuProvider`, `BuiltInRegistries` (1.19.3), `CreativeModeTabs` (1.19.3), `ItemDisplayContext` (1.19.4), `saveAdditional` (1.18+), модификаторы `BlockEntity#level`/`worldPosition`/`remove` и `Level#isClientSide`.
- **Что проверено по коду/докам:** `@SidedProxy` отсутствует с 1.13, `DistExecutor` есть с 1.13; AT-файлы в 1.19.x всё ещё требуют SRG-имён.
- **Что не утверждается:** точная дата первого релиза MixinExtras; версии Quilt для каждого патча 1.19.x (указан только факт существования).
- **Код:** фрагменты иллюстративные и не компилировались; сигнатуры сверялись с mappings и документацией указанных версий.

## Источники

1. Minecraft Wiki — Java Edition 1.17 (Java 16): <https://minecraft.wiki/w/Java_Edition_1.17>
2. Minecraft Wiki — Java Edition 1.17.1: <https://minecraft.wiki/w/Java_Edition_1.17.1>
3. Minecraft Wiki — Java Edition 1.18 (Java 17): <https://minecraft.wiki/w/Java_Edition_1.18>
4. Minecraft Wiki — Java Edition 1.18.2: <https://minecraft.wiki/w/Java_Edition_1.18.2>
5. Minecraft Wiki — Java Edition 1.19: <https://minecraft.wiki/w/Java_Edition_1.19>
6. Minecraft Wiki — Java Edition 1.19.2: <https://minecraft.wiki/w/Java_Edition_1.19.2>
7. Minecraft Wiki — Java Edition 1.19.3: <https://minecraft.wiki/w/Java_Edition_1.19.3>
8. Minecraft Wiki — Java Edition 1.19.4: <https://minecraft.wiki/w/Java_Edition_1.19.4>
9. Forge promotions: <https://files.minecraftforge.net/net/minecraftforge/forge/promotions_slim.json>
10. Forge docs 1.17.x — Getting Started («Forge uses Mojang's Official Mappings»): <https://docs.minecraftforge.net/en/1.17.x/gettingstarted/>
11. Forge docs 1.19.x — Access Transformers (SRG-имена): <https://docs.minecraftforge.net/en/1.19.x/advanced/accesstransformers/>
12. mappings.dev, community-браузер mappings — `Level` (1.17): <https://mappings.dev/1.17/net/minecraft/world/level/Level.html>
13. mappings.dev, community-браузер mappings — `BlockEntity` (1.17): <https://mappings.dev/1.17/net/minecraft/world/level/block/entity/BlockEntity.html>
14. mappings.dev, community-браузер mappings — `PoseStack` (1.17): <https://mappings.dev/1.17/com/mojang/blaze3d/vertex/PoseStack.html>
15. mappings.dev, community-браузер mappings — `InteractionHand` (1.17): <https://mappings.dev/1.17/net/minecraft/world/InteractionHand.html>
16. mappings.dev, community-браузер mappings — `BuiltInRegistries` (1.19.3): <https://mappings.dev/1.19.3/net/minecraft/core/registries/BuiltInRegistries.html>
17. mappings.dev, community-браузер mappings — `ItemDisplayContext` (1.19.4): <https://mappings.dev/1.19.4/net/minecraft/world/item/ItemDisplayContext.html>
18. MixinBootstrap README (Mixin в Forge с 1.15.2-31.2.44+ и 1.16.1-32.0.72+): <https://github.com/LXGaming/MixinBootstrap>
19. Mixin wiki — Mixins on Minecraft Forge: <https://github.com/SpongePowered/Mixin/wiki/Mixins-on-Minecraft-Forge>
20. NeoForge (форк после 1.20.1): <https://neoforged.net/>
21. Fabric Documentation — Yarn, Loom и миксины: <https://docs.fabricmc.net/develop/>
22. mappings.dev, community-браузер mappings — `BlockEntity#saveAdditional` (1.18.2): <https://mappings.dev/1.18.2/net/minecraft/world/level/block/entity/BlockEntity.html>
23. mappings.dev, community-браузер mappings — `Holder` (1.18.2): <https://mappings.dev/1.18.2/net/minecraft/core/Holder.html>
