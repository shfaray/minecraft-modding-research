# Кулинарная книга инъекций

[← К оглавлению](README.md)

> [!NOTE]
> 18 рецептов, каждый — в трёх эпохах: **1.20.1**, **1.21.x** и **26.x**. Это не учебник по Mixin: теорию выбора инъектора и якоря см. в [`02-mixin-playbook.md`](02-mixin-playbook.md). Здесь — готовые швы, сигнатуры и то, что ломается при портировании.
>
> Главный вывод: меняются не аннотации Mixin, а **имена, дескрипторы и владельцы методов**. `@Inject` / `@WrapOperation` / `@ModifyVariable` остаются теми же; переписывать приходится target и сигнатуру handler'а.

## Содержание

1. [Изменить входящий урон сущности](#1-изменить-входящий-урон-сущности)
2. [Добавить строку в тултип предмета](#2-добавить-строку-в-тултип-предмета)
3. [Отрисовать оверлей в GUI (HUD)](#3-отрисовать-оверлей-в-gui-hud)
4. [Отменить взаимодействие с блоком (right-click)](#4-отменить-взаимодействие-с-блоком-right-click)
5. [Добавить логику на тик блока (BlockEntity)](#5-добавить-логику-на-тик-блока-blockentity)
6. [Изменить размер стака / лимит предмета](#6-изменить-размер-стака--лимит-предмета)
7. [Запретить предмету попадать в слот](#7-запретить-предмету-попадать-в-слот)
8. [Добавить цель ИИ мобу (goal)](#8-добавить-цель-ии-мобу-goal)
9. [Перехватить / изменить исходящий custom payload](#9-перехватить--изменить-исходящий-custom-payload)
10. [Изменить результат крафта](#10-изменить-результат-крафта)
11. [Проиграть звук при событии](#11-проиграть-звук-при-событии)
12. [Изменить опыт с блока / моба](#12-изменить-опыт-с-блока--моба)
13. [Изменить поведение жидкости](#13-изменить-поведение-жидкости)
14. [Добавить эффект при использовании предмета](#14-добавить-эффект-при-использовании-предмета)
15. [Отменить ванильный вызов: `@WrapOperation` вместо `@Redirect`](#15-отменить-ванильный-вызов-wrapoperation-вместо-redirect)
16. [Изменить текст / локализацию на лету](#16-изменить-текст--локализацию-на-лету)
17. [Добавить кнопку в экран (Screen)](#17-добавить-кнопку-в-экран-screen)
18. [Изменить спавн моба](#18-изменить-спавн-моба)

## Три эпохи в двух словах

| | 1.20.1 | 1.21.x | 26.x |
| --- | --- | --- | --- |
| Java | 17 | 21 | 25 |
| Имена в рантайме | Mojmap (Forge) / intermediary + Yarn (Fabric) | Mojmap (NeoForge) / intermediary (Fabric) | **Mojang-имена без ремапа** |
| `compatibilityLevel` | `JAVA_17` | `JAVA_21` | `JAVA_25` |
| Идентификатор | `ResourceLocation` | `ResourceLocation` | `Identifier` (см. врезку ниже) |
| Предметные данные | NBT | Data Components (с 1.20.5) | Data Components + `ItemStackTemplate` |
| GUI | `GuiGraphics`, `render` | `GuiGraphics`, `render` | `GuiGraphicsExtractor`, `extractRenderState` |
| Результат взаимодействия | `InteractionResult` (enum) | `InteractionResult` (sealed, с 1.21.2) | `InteractionResult` (sealed) |
| Рецепты | `Recipe<C extends Container>` | `Recipe<T extends RecipeInput>` | `Recipe`, результат — `ItemStackTemplate` |

> [!IMPORTANT]
> В 26.2 и 26.3 часть имён из 26.1 уже сдвинулась: HUD вынесен в `Hud`, `Gui#extractRenderState` перестал принимать `GuiGraphicsExtractor`, а сигнатуры `extractRenderState` у сущностей и блок-сущностей менялись ещё несколько раз. Код для «26.x» ниже — **базовая линия 26.1**; отличия 26.2/26.3 отмечены в каждом рецепте. Это честнее, чем делать вид, что 26.x монолитен.

> [!NOTE]
> **Про `Identifier`.** В примерах официального primer'а для 26.1–26.3 идентификатор называется `Identifier.fromNamespaceAndPath(...)`, а `ResourceLocation` в тексте 26.2/26.3 не встречается вовсе. В этом документе для 26.x используется `Identifier`. Если в ваших mappings класс всё ещё называется `ResourceLocation` — это тот же тип, ориентируйтесь на свой IDE. `[unverified]` для точного имени в вашей сборке.

### Соглашения

- Код — Mojang mappings, если не сказано иное. На Fabric 1.20.1/1.21.x в исходниках будут Yarn-имена; миксин компилируется в named и ремапится Loom'ом.
- Пакеты: геймплей — `net.minecraft.world.*`, клиент — `net.minecraft.client.*`, сеть — `net.minecraft.network.protocol.*`. В `@Mixin` указан простой класс; полное имя при необходимости подскажет IDE.
- Дескрипторы даны в формате `Lowner;name(args)ret;` и только там, где без них якорь неоднозначен.
- `[unverified]` — имя или сигнатура не подтверждены первоисточником для этой конкретной версии; код с такой пометкой **иллюстративен** и требует проверки в IDE/`javap`.
- «Иллюстративно» — код показывает форму решения, а не гарантированный рабочий патч.
- `@Unique` на служебных полях/методах миксина опущен для краткости, но в реальном моде обязателен.
- `refmap`/`remap`: на 1.20.1 Fabric его генерирует Loom, на Forge/NeoForge он не нужен; с 26.1 ремапа нет вовсе, поэтому `remap = false` для модовых целей больше не требуется.
- Все примеры предполагают, что миксин уже подключён в `*.mixins.json`; конфиг и `refmap` здесь не рассматриваются.
- `@Expression` и `@Local` — из MixinExtras; на 1.20.1 Forge библиотеку нужно подключать отдельно, в NeoForge она встроена. `@Local` по имени локальной переменной доступен только с 26.1, до этого — по типу/ordinal/индексу.

Неочевидные импорты, которые встречаются ниже:

```java
import com.llamalad7.mixinextras.sugar.Local;
import com.llamalad7.mixinextras.injector.ModifyReturnValue;
import com.llamalad7.mixinextras.injector.wrapoperation.Operation;
import com.llamalad7.mixinextras.injector.wrapoperation.WrapOperation;
import com.llamalad7.mixinextras.expression.Definition;
import com.llamalad7.mixinextras.expression.Expression;
```

---

## Рецепты

### 1. Изменить входящий урон сущности

**Где искать:** `LivingEntity#hurt` (1.20.1) и `LivingEntity#hurtServer` (1.21.2+). Шов — **до** расчёта брони, сопротивления и эффектов; если нужно менять уже итоговый урон, смотрите `actuallyHurt` / `applyDamage`. Не путать с `Entity#hurt`: у `Entity` это диспетчер, у `LivingEntity` — основная логика.

**1.20.1**

```java
@Mixin(LivingEntity.class)
public abstract class IncomingDamageMixin {

    @ModifyVariable(
        method = "hurt(Lnet/minecraft/world/damagesource/DamageSource;F)F",
        at = @At("HEAD"),
        argsOnly = true
    )
    private float cookbook$scaleDamage(float amount,
                                       @Local(argsOnly = true) DamageSource source) {
        if (source.is(DamageTypeTags.IS_FIRE)) {
            return amount * 0.5F;
        }
        return amount;
    }
}
```

`@Local(argsOnly = true)` — сахар MixinExtras: он достаёт соседний аргумент метода, не заставляя угадывать индекс локальной переменной. Без MixinExtras пришлось бы писать `@ModifyVariable(..., index = 2)`.

**1.21.x**

```java
@Mixin(LivingEntity.class)
public abstract class IncomingDamageMixin {

    @ModifyVariable(
        method = "hurtServer(Lnet/minecraft/server/level/ServerLevel;Lnet/minecraft/world/damagesource/DamageSource;F)F",
        at = @At("HEAD"),
        argsOnly = true
    )
    private float cookbook$scaleDamage(float amount,
                                       @Local(argsOnly = true) ServerLevel level,
                                       @Local(argsOnly = true) DamageSource source) {
        if (source.is(DamageTypeTags.IS_FIRE)) {
            return amount * 0.5F;
        }
        return amount;
    }
}
```

С 1.21.2 `hurt` помечен deprecated и разложен на `hurtServer(ServerLevel, DamageSource, float)` и `hurtClient(DamageSource)`. На 1.21–1.21.1 ещё работает старый `hurt`. На 1.21.11 `hurtClient` принимает **только** `DamageSource` — величина урона там уже не передаётся.

**26.x** — без изменений относительно 1.21.x: `hurtServer(ServerLevel, DamageSource, float)` сохранился, `hurt` остался deprecated-обёрткой. Меняются только окружающие имена (`Identifier` вместо `ResourceLocation`), которых в этом рецепте нет. Поэтому код идентичен; отличие будет в конфиге миксина (`JAVA_25`) и в том, что `remap = false` больше не нужен.

> [!WARNING]
> `hurtServer` выполняется только на сервере. Если вы **отменяете** урон, клиент об этом не узнает и может показать анимацию/звук получения урона. Для полноценной отмены нужен свой пакет или синхронизация; для изменения величины достаточно серверной стороны.

**Что изменилось:**

- 1.21.2: `hurt` → `hurtServer` / `hurtClient`; сигнатура серверного метода получила `ServerLevel`.
- Дескриптор якоря изменился: `(DamageSource;F)F` → `(ServerLevel;DamageSource;F)F`.
- 26.x: сигнатура та же, но имена — рантайм-имена Mojang, `refmap` не нужен.
- `@Local(argsOnly = true)` работает во всех трёх эпохах, если MixinExtras подключён.

**Подводные камни:**

- `@ModifyVariable` без `index` выбирает переменную **по типу handler'а**. Если в метод добавится второй `float`, якорь станет неоднозначным — фиксируйте `index` или используйте `@Local`.
- Урон здесь ещё «сырой»: броня, enchantments и resistance применяются позже. Умножение на 0.5 не означает «половина итогового урона».
- Не вызывайте `setHealth` из этого инъектора — вы обойдёте death-логику, advancement'ы и `LivingEntity#die`.
- На 1.20.1 `DamageSource#is(TagKey<DamageType>)` есть; на 26.x теги типов урона те же, но реестры datapack-driven — проверяйте, что тег загружен.

---

### 2. Добавить строку в тултип предмета

**Где искать:** центральный шов — `ItemStack#getTooltipLines(...)`, а не `Item#appendHoverText`. `appendHoverText` переопределяют десятки предметов, поэтому миксин в `Item` поймает только базовую реализацию. `getTooltipLines` собирает список один раз и возвращает его клиенту.

**1.20.1**

```java
@Mixin(ItemStack.class)
public abstract class TooltipMixin {

    @Inject(method = "getTooltipLines", at = @At("RETURN"))
    private void cookbook$appendLine(Player player, TooltipFlag flag,
                                     CallbackInfoReturnable<List<Component>> cir) {
        if (this.is(MyItems.MYSTERY_INGOT)) {
            cir.getReturnValue().add(
                Component.literal("Проверено поваром")
                    .withStyle(ChatFormatting.GRAY, ChatFormatting.ITALIC)
            );
        }
    }
}
```

**1.21.x**

```java
@Mixin(ItemStack.class)
public abstract class TooltipMixin {

    @Inject(method = "getTooltipLines", at = @At("RETURN"))
    private void cookbook$appendLine(Item.TooltipContext context, Player player,
                                     TooltipFlag flag,
                                     CallbackInfoReturnable<List<Component>> cir) {
        if (this.is(MyItems.MYSTERY_INGOT)) {
            cir.getReturnValue().add(
                Component.literal("Проверено поваром")
                    .withStyle(ChatFormatting.GRAY, ChatFormatting.ITALIC)
            );
        }
    }
}
```

В 1.20.5 `Level` в тултипе заменили на `Item.TooltipContext` (реестры + tick rate + map data), а в 1.21.5 `appendHoverText` переделали на `(ItemStack, Item.TooltipContext, TooltipDisplay, Consumer<Component>, TooltipFlag)` и пометили deprecated. `getTooltipLines` при этом остался удобной точкой «после всех строк».

**26.x** — без изменений относительно 1.21.x: сигнатура `getTooltipLines(Item.TooltipContext, Player, TooltipFlag)` сохранилась, `TooltipDisplay` по-прежнему фильтрует скрытые компоненты до возврата списка. В 26.3 у `TooltipProvider#addToTooltip` появился overload с `TooltipProvider.Getter`, но на этот шов он не влияет.

**Что изменилось:**

- 1.20.5: `Player, TooltipFlag` → `Item.TooltipContext, Player, TooltipFlag`.
- 1.21.5: `appendHoverText` получил `TooltipDisplay` и `Consumer<Component>`; для data-driven предметов правильнее регистрировать `TooltipProvider`-компонент, а не миксин.
- 26.x: сигнатуры тултипа стабильны; `ItemStackTemplate` используется для иконок/рецептов, но не для тултипа.

**Подводные камни:**

- `getTooltipLines` вызывается и на сервере (например, для `HoverEvent.ShowItem`), поэтому не лезьте в `Minecraft.getInstance()`.
- Список может быть неизменяемым в некоторых версиях/путях; безопаснее `@ModifyReturnValue` и вернуть новый `ArrayList`.
- Если другой мод тоже добавляет строку, порядок зависит от priority миксина. Не полагайтесь на «последнюю строку».
- Данные предмета: в 1.20.1 — NBT (`stack.getTag()`/`getOrCreateTag()`), в 1.20.5+ — компоненты (`stack.get(DataComponents.X)`). Не переносите NBT-код в 1.21.x/26.x без переписывания.
- Не добавляйте строки для пустого стака: проверка `isEmpty()` до `is(...)`.

---

### 3. Отрисовать оверлей в GUI (HUD)

**Где искать:** 1.20.1/1.21.x — `Gui#render`; 26.1 — `Gui#extractRenderState`; 26.2+ — `Hud#extractRenderState`. Рисовать в `Screen#render` нельзя: HUD живёт вне экранов и должен переживать открытие инвентаря.

**1.20.1**

```java
@Mixin(Gui.class)
public abstract class HudOverlayMixin {

    @Inject(method = "render", at = @At("TAIL"))
    private void cookbook$drawBadge(GuiGraphics graphics, float partialTick,
                                    CallbackInfo ci) {
        Minecraft minecraft = Minecraft.getInstance();
        if (minecraft.options.hideGui || minecraft.player == null) {
            return;
        }
        int x = 6;
        int y = 6;
        graphics.fill(x, y, x + 70, y + 16, 0x90000000);
        graphics.drawString(minecraft.font, "Cookbook", x + 4, y + 4, 0xFFFFFF);
    }
}
```

Дескриптор: `render(Lnet/minecraft/client/gui/GuiGraphics;F)V`.

**1.21.x**

```java
@Mixin(Gui.class)
public abstract class HudOverlayMixin {

    @Inject(method = "render", at = @At("TAIL"))
    private void cookbook$drawBadge(GuiGraphics graphics, DeltaTracker deltaTracker,
                                    CallbackInfo ci) {
        Minecraft minecraft = Minecraft.getInstance();
        if (minecraft.options.hideGui || minecraft.player == null) {
            return;
        }
        int x = 6;
        int y = 6;
        graphics.fill(x, y, x + 70, y + 16, 0x90000000);
        graphics.drawString(minecraft.font, "Cookbook", x + 4, y + 4, 0xFFFFFF);
    }
}
```

В 1.21 `Gui#render` принимает `DeltaTracker` вместо `float partialTick` (это изменение 1.21, а не 1.20.5). Дескриптор: `(Lnet/minecraft/client/gui/GuiGraphics;Lnet/minecraft/client/DeltaTracker;)V`.

**26.x (26.1 baseline)**

```java
// [unverified] 26.1 baseline: точный порядок параметров extractRenderState
@Mixin(Gui.class)
public abstract class HudOverlayMixin {

    @Inject(method = "extractRenderState", at = @At("TAIL"))
    private void cookbook$drawBadge(GuiGraphicsExtractor graphics,
                                    DeltaTracker deltaTracker,
                                    CallbackInfo ci) {
        Minecraft minecraft = Minecraft.getInstance();
        if (minecraft.options.hideGui || minecraft.player == null) {
            return;
        }
        int x = 6;
        int y = 6;
        graphics.fill(x, y, x + 70, y + 16, 0x90000000);
        graphics.text(minecraft.font, "Cookbook", x + 4, y + 4, 0xFFFFFF);
    }
}
```

`[unverified]` — точный порядок параметров `Gui#extractRenderState` в 26.1. В 26.2 `extractRenderState` перестал принимать `GuiGraphicsExtractor` и стал принимать два boolean'а (рендерить ли уровень и загружены ли ресурсы), а HUD переехал в `Hud`. Для 26.2+ цель — `Hud#extractRenderState`, и рисовать нужно не напрямую, а через `GuiRenderState`/`GuiGraphicsExtractor` в соответствующей фазе. Это уже не «переименование», а другой контракт.

**Что изменилось:**

- 1.21: `float partialTick` → `DeltaTracker`.
- 26.1: `GuiGraphics` → `GuiGraphicsExtractor`; `render` → `extractRenderState`; `drawString` → `text`, `draw*`/`render*`/`submit*` префиксы убраны.
- 26.2+: HUD вынесен в `Hud`; `Gui` больше не рисует HUD напрямую.
- 26.3: `GuiGraphicsExtractor#textWithWordWrap` возвращает следующий свободный `y`, `tooltip` получил overload с доп. отступом.

> [!WARNING]
> В 26.x рисование в `extractRenderState` — это **подготовка render state**, а не немедленный draw call. Если вы вызываете `graphics.fill` в неправильной фазе, на экране ничего не появится, хотя миксин применился. Проверяйте через `-Dmixin.debug.export=true`, что метод действительно пропатчен, и через `GuiRenderState`, что ваш элемент попал в дерево.

**Подводные камни:**

- `hideGui` (F1) нужно проверять вручную; иначе оверлей останется поверх скрытого HUD.
- Не создавайте `new Component`/`String` каждый кадр — кэшируйте.
- `GuiGraphics#fill` принимает ARGB; альфа `0x90` — полупрозрачный фон.
- На 26.x `GuiGraphicsExtractor#fill` не переименован, но `drawString` → `text`; старый код не скомпилируется.
- Не миксиньте `Gui#render` на 26.2+, если цель — HUD: метод может быть пустым/делегирующим.

---

### 4. Отменить взаимодействие с блоком (right-click)

**Где искать:** 1.20.1 — `BlockBehaviour#use(BlockState, Level, BlockPos, Player, InteractionHand, BlockHitResult)`. 1.21.2+ — `BlockBehaviour#useItemOn(ItemStack, BlockState, Level, BlockPos, Player, InteractionHand, BlockHitResult)` плюс отдельный `useWithoutItem(...)` для пустой руки. Возврат `InteractionResult.FAIL` останавливает дальнейшую обработку.

**1.20.1**

```java
@Mixin(BlockBehaviour.class)
public abstract class BlockUseCancelMixin {

    @Inject(
        method = "use(Lnet/minecraft/world/level/block/state/BlockState;Lnet/minecraft/world/level/Level;Lnet/minecraft/core/BlockPos;Lnet/minecraft/world/entity/player/Player;Lnet/minecraft/world/InteractionHand;Lnet/minecraft/world/phys/BlockHitResult;)Lnet/minecraft/world/InteractionResult;",
        at = @At("HEAD"),
        cancellable = true
    )
    private void cookbook$cancelUse(BlockState state, Level level, BlockPos pos,
                                    Player player, InteractionHand hand,
                                    BlockHitResult hit,
                                    CallbackInfoReturnable<InteractionResult> cir) {
        if (state.is(Blocks.ENCHANTING_TABLE) && !player.isCreative()) {
            cir.setReturnValue(InteractionResult.FAIL);
        }
    }
}
```

**1.21.x**

```java
@Mixin(BlockBehaviour.class)
public abstract class BlockUseCancelMixin {

    @Inject(
        method = "useItemOn(Lnet/minecraft/world/item/ItemStack;Lnet/minecraft/world/level/block/state/BlockState;Lnet/minecraft/world/level/Level;Lnet/minecraft/core/BlockPos;Lnet/minecraft/world/entity/player/Player;Lnet/minecraft/world/InteractionHand;Lnet/minecraft/world/phys/BlockHitResult;)Lnet/minecraft/world/InteractionResult;",
        at = @At("HEAD"),
        cancellable = true
    )
    private void cookbook$cancelUse(ItemStack stack, BlockState state, Level level,
                                    BlockPos pos, Player player, InteractionHand hand,
                                    BlockHitResult hit,
                                    CallbackInfoReturnable<InteractionResult> cir) {
        if (state.is(Blocks.ENCHANTING_TABLE) && !player.isCreative()) {
            cir.setReturnValue(InteractionResult.FAIL);
        }
    }
}
```

`ItemStack` идёт **первым** аргументом — это главное изменение 1.21.2. Если нужна реакция на пустую руку, цель — `useWithoutItem(BlockState, Level, BlockPos, Player, BlockHitResult)`.

**26.x** — без изменений относительно 1.21.x: `useItemOn(...)` и `useWithoutItem(...)` сохранились, `InteractionResult` остался sealed-интерфейсом с константами `FAIL`, `PASS`, `SUCCESS`, `CONSUME`, `TRY_WITH_EMPTY_HAND`. Отличие только в конфиге (`JAVA_25`) и в том, что `BlockBehaviour` — рантайм-имя без ремапа.

**Что изменилось:**

- 1.21.2: `use` разложен на `useItemOn(ItemStack, ...)` и `useWithoutItem(...)`; добавлен `InteractionResult.TRY_WITH_EMPTY_HAND`.
- 1.21.2: `InteractionResult` из enum стал sealed-интерфейсом; `InteractionResultHolder` удалён.
- 26.x: сигнатуры те же, но `Identifier`/`ItemStackTemplate` и прочие 26.x-имена вокруг.
- На 1.20.1 `BlockBehaviour#use` — единственная точка; на 1.21.2+ легко пропустить `useWithoutItem`, если игрок кликает пустой рукой.

> [!WARNING]
> `useItemOn` вызывается и на клиенте, и на сервере. Если отменять только на сервере, клиент проиграет анимацию/звук и может «застрять» в предсказании. Возвращайте `FAIL` с обеих сторон или синхронизируйте состояние. И не отменяйте взаимодействие для Creative, если ванильный блок должен оставаться доступным.

**Подводные камни:**

- `InteractionResult.FAIL` не всегда означает «ничего не произошло»: для предметов он может заблокировать дальнейшие обработчики, но не откатить уже проигранную клиентскую анимацию.
- `PASS` — «я не обработал, иди дальше»; `FAIL` — «не обрабатывай дальше». Перепутать легко.
- На 1.21.2+ порядок такой: сначала `useItemOn`, затем, если результат `TRY_WITH_EMPTY_HAND`, — `useWithoutItem`. Отменяя только первый, вы не отмените взаимодействие пустой рукой.
- Блоки с собственным `useItemOn` (сундуки, верстаки) могут обрабатываться в override — миксин в `BlockBehaviour` поймает только базовую реализацию; для конкретного блока лучше миксинить его класс.

---

### 5. Добавить логику на тик блока (BlockEntity)

**Где искать:** у `BlockEntity` **нет** метода `tick()` — это частая ошибка. Тик приходит через `BlockEntityTicker`, который возвращает `Block#getTicker`. Для ванильного блока нужно миксинить его статический `serverTick` (например, `AbstractFurnaceBlockEntity#serverTick`); для своего — просто зарегистрировать `BlockEntityTicker` без миксина.

**1.20.1**

```java
@Mixin(AbstractFurnaceBlockEntity.class)
public abstract class FurnaceTickMixin {

    @Inject(method = "serverTick", at = @At("TAIL"))
    private static void cookbook$afterTick(Level level, BlockPos pos, BlockState state,
                                           AbstractFurnaceBlockEntity furnace,
                                           CallbackInfo ci) {
        if (level instanceof ServerLevel server && server.getGameTime() % 20 == 0) {
            server.sendParticles(ParticleTypes.SMOKE,
                pos.getX() + 0.5, pos.getY() + 1.0, pos.getZ() + 0.5,
                1, 0.1, 0.1, 0.1, 0.01);
        }
    }
}
```

Дескриптор: `serverTick(Lnet/minecraft/world/level/Level;Lnet/minecraft/core/BlockPos;Lnet/minecraft/world/level/block/state/BlockState;Lnet/minecraft/world/level/block/entity/AbstractFurnaceBlockEntity;)V`.

**1.21.x**

```java
@Mixin(AbstractFurnaceBlockEntity.class)
public abstract class FurnaceTickMixin {

    @Inject(method = "serverTick", at = @At("TAIL"))
    private static void cookbook$afterTick(ServerLevel level, BlockPos pos, BlockState state,
                                           AbstractFurnaceBlockEntity furnace,
                                           CallbackInfo ci) {
        if (level.getGameTime() % 20 == 0) {
            level.sendParticles(ParticleTypes.SMOKE,
                pos.getX() + 0.5, pos.getY() + 1.0, pos.getZ() + 0.5,
                1, 0.1, 0.1, 0.1, 0.01);
        }
    }
}
```

С 1.21.2 `serverTick` принимает `ServerLevel` вместо `Level`. Это ломает дескриптор, но не логику: на 1.20.1 параметр был `Level`, поэтому приходилось проверять `instanceof ServerLevel`; в 1.21.2+ контракт уже серверный.

**26.x** — без изменений относительно 1.21.x: `serverTick(ServerLevel, BlockPos, BlockState, AbstractFurnaceBlockEntity)` сохранился. В 26.2 появился `BlockEntityTicker#andThen` — для **своего** BlockEntity это лучший способ, миксин не нужен вовсе.

**Что изменилось:**

- 1.21.2: `Level` → `ServerLevel` в `serverTick`; дескриптор изменился.
- 26.2: `BlockEntityTicker#andThen` позволяет цеплять тикеры без миксина.
- 26.x: имена — рантайм-имена; `BlockEntityTicker` остался функциональным интерфейсом.
- Для своего BE `BlockEntity#tick()` так и не появился: используйте `BlockEntityTicker`.

**Подводные камни:**

- Handler для static-метода должен быть `static`, иначе Mixin не соберёт инъекцию.
- `@At("TAIL")` выполняется после ванильного тика; если нужен «до», ставьте `HEAD`.
- Не вызывайте `setChanged()` каждый тик — это лишние пакеты и нагрузка на чанк.
- `sendParticles` на каждом тике — дорого; проверяйте `gameTime % N`, но помните, что `gameTime` общий для мира.
- Миксин в `AbstractFurnaceBlockEntity#serverTick` поймает и печь, и доменную печь, и коптильню: они используют один статический метод. Если нужна только печь — проверяйте `furnace.getType()`.

---

### 6. Изменить размер стака / лимит предмета

**Где искать:** 1.20.1 — `Item#getMaxStackSize()` (поле `maxStackSize`, задаётся `Item.Properties#stacksTo`). 1.20.5+ — `ItemStack#getMaxStackSize()` и компонент `DataComponents.MAX_STACK_SIZE`. 26.1 — `ItemInstance#getMaxStackSize` (общий контракт для `ItemStack` и `ItemStackTemplate`).

**1.20.1**

```java
@Mixin(Item.class)
public abstract class StackSizeMixin {

    @Inject(method = "getMaxStackSize", at = @At("HEAD"), cancellable = true)
    private void cookbook$override(CallbackInfoReturnable<Integer> cir) {
        if ((Object) this == MyItems.GIANT_APPLE) {
            cir.setReturnValue(64);
        }
    }
}
```

`ItemStack#getMaxStackSize()` делегирует в `Item#getMaxStackSize()`, поэтому этого достаточно для любого стака.

**1.21.x**

```java
@Mixin(ItemStack.class)
public abstract class StackSizeMixin {

    @ModifyReturnValue(method = "getMaxStackSize", at = @At("RETURN"))
    private int cookbook$override(int original) {
        return this.is(MyItems.GIANT_APPLE) ? 64 : original;
    }
}
```

С 1.20.5 лимит хранится в компоненте `minecraft:max_stack_size`; `Item.Properties#stacksTo(64)` — просто удобный способ его выставить. `@ModifyReturnValue` лучше `@Inject(RETURN) + setReturnValue`, потому что цепляется с другими модами. Если миксин не нужен, достаточно `new Item.Properties().component(DataComponents.MAX_STACK_SIZE, 64)`.

**26.x**

```java
// [unverified] 26.1: метод переехал в ItemInstance, но ItemStack может иметь override
@Mixin(ItemInstance.class)
public abstract class StackSizeMixin {

    @ModifyReturnValue(method = "getMaxStackSize", at = @At("RETURN"))
    private int cookbook$override(int original) {
        if (this instanceof ItemStack stack && stack.is(MyItems.GIANT_APPLE)) {
            return 64;
        }
        return original;
    }
}
```

`[unverified]` — в 26.1 `getMaxStackSize` переехал в `ItemInstance`, но `ItemStack` может сохранять override. Если миксин в интерфейс не применяется, цельтесь в `ItemStack#getMaxStackSize` и проверяйте через `-Dmixin.debug.verbose=true`, какой класс реально патчится.

**Что изменилось:**

- 1.20.5: поле `maxStackSize` → data component `MAX_STACK_SIZE`.
- 1.20.5+: меняется `ItemStack`, а не `Item`; `Item#getMaxStackSize` больше не единственный источник.
- 26.1: `ItemStack` реализует `ItemInstance`; общий метод живёт в `ItemInstance`, чтобы работать и для `ItemStackTemplate`.
- 26.x: `ItemStackTemplate` тоже отвечает на `getMaxStackSize`, поэтому миксин в `ItemStack` может не покрыть шаблоны.

> [!WARNING]
> Компонент `MAX_STACK_SIZE` имеет codec-диапазон `1..99`. Вернуть из `getMaxStackSize` 128 технически можно, но такой стак не сохранится/не синхронизируется корректно. Если нужен «стак больше 99», это отдельная задача: свой компонент, свой слот и свой GUI, а не один `@ModifyReturnValue`.

**Подводные камни:**

- `ItemStack#setCount` не проверяет лимит; за вас это делает `Inventory`/`Container`. Миксин лимита не защищает от `/give` с NBT.
- На 1.21.x `ItemStack#getMaxStackSize` вызывается очень часто (comparator, hopper, GUI); не делайте в нём тяжёлых проверок.
- На 26.x `ItemInstance` — интерфейс; mixin в интерфейс с `@ModifyReturnValue` работает, но не гарантирует перехват override'ов.
- Клиент и сервер должны видеть одинаковый лимит, иначе предметы будут «прыгать» при перемещении.

---

### 7. Запретить предмету попадать в слот

**Где искать:** `Slot#mayPlace(ItemStack)` — основная проверка для ручного перемещения и `quickMoveStack`. Для автоматизации (воронки, трубы) нужен ещё `Container#canPlaceItem(int, ItemStack)`: `Slot#mayPlace` там не вызывается.

**1.20.1**

```java
@Mixin(Slot.class)
public abstract class SlotFilterMixin {

    @Inject(method = "mayPlace", at = @At("HEAD"), cancellable = true)
    private void cookbook$filter(ItemStack stack,
                                 CallbackInfoReturnable<Boolean> cir) {
        if (stack.is(Items.NETHER_STAR) && this.container instanceof Inventory) {
            cir.setReturnValue(false);
        }
    }
}
```

`Slot#container` — `public final Container`, поэтому доступен без `@Shadow`.

**1.21.x** — без изменений: `mayPlace(ItemStack)` и поле `container` те же. Единственное отличие вокруг — в 1.21.2 клики меню стали `ContainerInput` вместо `ClickType`, но сама проверка слота не изменилась.

**26.x** — без изменений: `Slot#mayPlace(ItemStack)` сохранился; в 26.3 удалили `BrewingStandMenu.PotionSlot#mayPlaceItem`, но это другой метод. Если в вашей сборке `container` стал приватным, добавьте `@Shadow @Final Container container;` или используйте геттер.

**Что изменилось:**

- Практически ничего: `mayPlace` — один из самых стабильных швов.
- 1.21.2: `AbstractContainerMenu#clicked` перешёл на `ContainerInput`; это влияет на обработку клика, но не на `mayPlace`.
- 26.3: `BrewingStandMenu.PotionSlot#mayPlaceItem` удалён — не путать с `Slot#mayPlace`.
- 26.x: имена — рантайм-имена; `@Shadow` на поле всё ещё работает.

> [!WARNING]
> `Slot#mayPlace` не защищает от воронок, дозаторов и других автоматических путей. Если предмет не должен попадать в слот вообще, добавьте миксин в `Container#canPlaceItem` конкретного контейнера или в `WorldlyContainer#canPlaceItemThroughFace`.

**Подводные камни:**

- `quickMoveStack` вызывает `moveItemStackTo`, а тот — `mayPlace`; но некоторые меню делают собственные проверки и могут обойти ваш фильтр.
- Если сервер запрещает, а клиент разрешает, предмет будет «мигать» и возвращаться. Проверка должна быть детерминированной с обеих сторон.
- `this.container instanceof Inventory` — грубая проверка; лучше сравнивать `this.container` с конкретным инвентарём или проверять `this.getClass()`.
- Creative-игрок может класть предметы через `/give` и drag-события, которые не всегда идут через `mayPlace`.

---

### 8. Добавить цель ИИ мобу (goal)

**Где искать:** `Mob#registerGoals()` — protected-метод, вызывается из конструктора. `goalSelector` и `targetSelector` — protected-поля `Mob`. Цели добавляются в `registerGoals`; для brain-based мобов (Warden, Piglin) goals не используются.

**1.20.1**

```java
@Mixin(Zombie.class)
public abstract class ZombieGoalMixin {

    @Inject(method = "registerGoals", at = @At("TAIL"))
    private void cookbook$addGoals(CallbackInfo ci) {
        this.goalSelector.addGoal(2, new MeleeAttackGoal(this, 1.0D, false));
        this.goalSelector.addGoal(7, new WaterAvoidingRandomStrollGoal(this, 1.0D));
    }
}
```

**1.21.x** — без изменений: `registerGoals()`, `goalSelector`, `MeleeAttackGoal`, `WaterAvoidingRandomStrollGoal` сохранились. AI goals — один из самых стабильных API в игре.

**26.x** — без изменений: `Mob#registerGoals()` и `GoalSelector` остались. В 26.1 менялись activities/brains и sound variants, но классические goals не трогали.

**Что изменилось:**

- Ничего в самом goal-API за три эпохи.
- 26.1: изменения в `Activities`/`Brains` и `EntitySoundVariant` не затрагивают `goalSelector`.
- 26.x: имена — рантайм-имена; `registerGoals` может быть `protected` и переопределён в подклассе.
- Для brain-based мобов цель надо добавлять в `Brain`/`Activity`, а не в `goalSelector`.

**Подводные камни:**

- `registerGoals` вызывается **из конструктора** до того, как поля подкласса инициализированы. Не обращайтесь к своим полям — они будут `null`.
- Приоритет: меньшее число = выше приоритет. `addGoal(0, ...)` может полностью вытеснить ванильные цели.
- `targetSelector` и `goalSelector` — разные селекторы; атакующая цель идёт в `targetSelector`, поведение — в `goalSelector`.
- Миксин в `Zombie` не поймает `Drowned`/`Husk`, если они не вызывают `super.registerGoals()`; проверяйте иерархию.
- `MeleeAttackGoal(this, ...)` требует `PathfinderMob`; для не-мобов компилятор не даст собрать.

---

### 9. Перехватить / изменить исходящий custom payload

**Где искать:** 1.20.1 — `ClientboundCustomPayloadPacket(ResourceLocation, FriendlyByteBuf)` с `getIdentifier()`/`getData()`. 1.20.2+ — `ClientboundCustomPayloadPacket(CustomPacketPayload)`; в 1.21.11 это record с `payload()`. Правильный шов — конструктор пакета, а не `Connection#send`: меньше риска рекурсии и гонок.

**1.20.1**

```java
@Mixin(ClientboundCustomPayloadPacket.class)
public abstract class OutgoingPayloadMixin {

    @ModifyVariable(method = "<init>", at = @At("HEAD"), argsOnly = true)
    private static FriendlyByteBuf cookbook$rewrite(
            FriendlyByteBuf data,
            @Local(argsOnly = true) ResourceLocation channel) {
        if (!channel.equals(new ResourceLocation("cookbook", "stats"))) {
            return data;
        }
        FriendlyByteBuf out = new FriendlyByteBuf(Unpooled.buffer());
        out.writeBytes(data.copy());
        out.writeVarInt(1); // добавляем поле в конец
        return out;
    }
}
```

`ClientboundCustomPayloadPacket` в 1.20.1 хранит `ResourceLocation` + `FriendlyByteBuf`; «типизированного» payload ещё нет. Handler для конструктора — `static`, чтобы не зависеть от неинициализированного `this`. Схема **иллюстративна**: проверьте, что `@ModifyVariable` в `<init>` с `@Local` применяется в вашей версии Mixin.

**1.21.x**

```java
@Mixin(ClientboundCustomPayloadPacket.class)
public abstract class OutgoingPayloadMixin {

    @ModifyVariable(method = "<init>", at = @At("HEAD"), argsOnly = true)
    private static CustomPacketPayload cookbook$rewrite(CustomPacketPayload payload) {
        if (payload instanceof StatsPayload stats) {
            return new StatsPayload(stats.value() + 1);
        }
        return payload;
    }
}
```

Тип payload объявляется один раз:

```java
public record StatsPayload(int value) implements CustomPacketPayload {

    public static final Type<StatsPayload> TYPE =
        new Type<>(ResourceLocation.fromNamespaceAndPath("cookbook", "stats"));

    public static final StreamCodec<FriendlyByteBuf, StatsPayload> STREAM_CODEC =
        StreamCodec.composite(
            ByteBufCodecs.VAR_INT, StatsPayload::value,
            StatsPayload::new
        );

    @Override
    public Type<? extends CustomPacketPayload> type() {
        return TYPE;
    }
}
```

**26.x**

```java
public record StatsPayload(int value) implements CustomPacketPayload {

    public static final Type<StatsPayload> TYPE =
        new Type<>(Identifier.fromNamespaceAndPath("cookbook", "stats"));

    public static final StreamCodec<FriendlyByteBuf, StatsPayload> STREAM_CODEC =
        StreamCodec.composite(
            ByteBufCodecs.VAR_INT, StatsPayload::value,
            StatsPayload::new
        );

    @Override
    public Type<? extends CustomPacketPayload> type() {
        return TYPE;
    }
}
```

Сам миксин в `ClientboundCustomPayloadPacket` — без изменений относительно 1.21.x: это по-прежнему record с конструктором `(CustomPacketPayload)` и методом `payload()`. В 26.x меняется только `ResourceLocation` → `Identifier`; в 26.3 всё больше пакетов становятся record'ами (`ServerboundUseItemOnPacket` и др.), что на этот шов не влияет.

**Что изменилось:**

- 1.20.2: введён `CustomPacketPayload` с `Type<T>` и `StreamCodec`; raw `ResourceLocation + FriendlyByteBuf` ушёл.
- 1.20.5: `StreamCodec` стал стандартом для payload'ов; `FriendlyByteBuf` остался транспортом.
- 1.21.11: `ClientboundCustomPayloadPacket` — record с `payload()`; `GAMEPLAY_STREAM_CODEC` и `CONFIG_STREAM_CODEC` разделены.
- 26.x: `ResourceLocation` → `Identifier`; `ByteBufCodecs.VAR_INT` и `StreamCodec.composite` сохранились.

> [!WARNING]
> Не перехватывайте `Connection#send`, если можно изменить пакет в конструкторе. `send` — горячий путь netty event loop; вызов `send` из него рекурсивно может привести к дэдлоку или StackOverflow. Если всё же перехватываете — ставьте guard (ThreadLocal/маркер в payload) и никогда не отправляйте пакет синхронно из того же потока.

**Подводные камни:**

- `ClientboundCustomPayloadPacket` используется и в gameplay, и в configuration phase. Не предполагайте, что реестры уже загружены.
- `MAX_PAYLOAD_SIZE` — 1 MiB; добавление байтов может превысить лимит и привести к disconnect.
- `@ModifyVariable` на `<init>` сработает для **всех** экземпляров; проверяйте `channel`/`payload.type()`, иначе испортите чужие пакеты.
- На 1.21.x `Type` должен быть зарегистрирован на обеих сторонах; иначе `payload()` не сериализуется.
- На 26.x `Identifier` может называться `ResourceLocation` в вашей IDE — проверяйте импорт, а не только текст.

---

### 10. Изменить результат крафта

**Где искать:** быстрый и стабильный шов — `ResultSlot#onTake(Player, ItemStack)`: это стак, который забирают из слота результата. Если нужно изменить **превью** и сам рецепт, цельтесь в `CraftingMenu#slotChangedCraftingGrid` (1.20.1/1.21.x) или в `Recipe#assemble` (26.x).

**1.20.1**

```java
@Mixin(ResultSlot.class)
public abstract class CraftResultMixin {

    @ModifyVariable(method = "onTake", at = @At("HEAD"), argsOnly = true)
    private ItemStack cookbook$boost(ItemStack stack) {
        if (stack.is(Items.IRON_PICKAXE)) {
            ItemStack copy = stack.copy();
            copy.grow(1);
            return copy;
        }
        return stack;
    }
}
```

Дескриптор: `onTake(Lnet/minecraft/world/entity/player/Player;Lnet/minecraft/world/item/ItemStack;)V`.

**1.21.x** — без изменений: `ResultSlot#onTake(Player, ItemStack)` сохранился, `@ModifyVariable(argsOnly = true)` работает так же. Меняется только окружение рецептов: с 1.21.2 `Recipe<T extends RecipeInput>`, `getResultItem` заменён на `display()`, а `getRemainingItems` переехал в `CraftingRecipe`.

**26.x** — без изменений для `ResultSlot#onTake`: это по-прежнему `ItemStack`. Но **определение** рецепта изменилось: `Recipe#assemble(RecipeInput)` больше не принимает `HolderLookup.Provider`, а результат хранится как `ItemStackTemplate`. Если нужно менять именно рецепт, а не взятие из слота:

```java
// [unverified] 26.x: проверить, объявлен ли assemble в ShapedRecipe
@Mixin(ShapedRecipe.class)
public abstract class RecipeResultMixin {

    @ModifyReturnValue(method = "assemble", at = @At("RETURN"))
    private ItemStack cookbook$assemble(ItemStack original, RecipeInput input) {
        if (original.is(Items.IRON_PICKAXE)) {
            ItemStack copy = original.copy();
            copy.grow(1);
            return copy;
        }
        return original;
    }
}
```

`[unverified]` — что `assemble` объявлен именно в `ShapedRecipe`, а не в общем `Recipe`/`CraftingRecipe`. Проверьте `javap` или исходник для своей 26.x-сборки.

**Что изменилось:**

- 1.21.2: `Recipe<C extends Container>` → `Recipe<T extends RecipeInput>`; `getResultItem` → `display()`; `getRemainingItems` → `CraftingRecipe`.
- 26.1: `Recipe#assemble` без `HolderLookup.Provider`; результат рецепта — `ItemStackTemplate`.
- 26.x: `ItemStackTemplate#create`/`fromNonEmptyStack` — способ превратить шаблон в стак.
- `ResultSlot#onTake` при этом стабилен: это самый дешёвый способ изменить выдачу, но не превью.

> [!WARNING]
> `onTake` вызывается после того, как предмет уже удалён из слота. Если вы возвращаете стак с большим count, вы фактически дублируете предметы. На сервере это эксплойт; обязательно ограничивайте количество и проверяйте, что игрок не в Creative, если это не задумано.

**Подводные камни:**

- `@ModifyVariable(argsOnly = true)` на `ItemStack` выбирает аргумент по типу; в `onTake` он один, но в других методах может быть несколько.
- Изменение только `onTake` не меняет то, что показывает recipe book и JEI/REI.
- На 1.21.2+ `display()` возвращает `RecipeDisplay`, а не `ItemStack`; для превью нужно менять `SlotDisplay`/`RecipeDisplay`, а не `getResultItem`.
- На 26.x `ItemStackTemplate` иммутабелен: `copy()` уже не тот паттерн, используйте `create()`/`apply(...)`.

---

### 11. Проиграть звук при событии

**Где искать:** `Level#playSound(Player, double x, double y, double z, SoundEvent, SoundSource, float volume, float pitch)`. Событие — любое: `Player#attack`, `Block#playerDestroy`, `LivingEntity#hurtServer`. Звук должен играть **на сервере**, иначе клиенты не увидят его синхронно.

**1.20.1**

```java
@Mixin(Player.class)
public abstract class AttackSoundMixin {

    @Inject(method = "attack", at = @At("TAIL"))
    private void cookbook$ding(Entity target, CallbackInfo ci) {
        if (this.level().isClientSide) {
            return;
        }
        this.level().playSound(
            null,
            this.getX(), this.getY(), this.getZ(),
            SoundEvents.EXPERIENCE_ORB_PICKUP,
            SoundSource.PLAYERS,
            0.6F, 1.4F
        );
    }
}
```

`SoundEvents.EXPERIENCE_ORB_PICKUP` — обычный `SoundEvent` (не `Holder`), поэтому `.value()` не нужен. Дескриптор: `attack(Lnet/minecraft/world/entity/Entity;)V`. Поле `Entity#level` в 1.20.1 уже приватное, поэтому используется аксессор `level()` — он есть во всех трёх эпохах.

**1.21.x** — без изменений: `this.level().playSound(...)` и `SoundEvents.EXPERIENCE_ORB_PICKUP` те же. В 1.21.5 часть `SoundEvents`-констант стала `Holder<SoundEvent>` (ITEM_BREAK, SHIELD_BLOCK, SHIELD_BREAK, WOLF_ARMOR_BREAK), но `EXPERIENCE_ORB_PICKUP` остался обычным `SoundEvent`.

**26.x** — без изменений относительно 1.21.x: `this.level().playSound(...)` и `SoundEvents.EXPERIENCE_ORB_PICKUP` работают. В 26.3 у `Level#playSound` появился overload с `Holder<SoundEvent>`, если ваша константа holder-wrapped. В 26.1 добавились registry'и sound variants для сущностей, но это не про `Level#playSound`.

**Что изменилось:**

- В этом шве — ничего: `Entity#level()` и `Level#playSound` не менялись с 1.20.1.
- 1.21.5: часть `SoundEvents` стала `Holder<SoundEvent>`; проверяйте тип константы, а не версию.
- 26.3: overload `playSound(..., Holder<SoundEvent>, ...)`.
- 26.x: звуки сущностей могут быть data-driven (sound variants), но произвольный `playSound` — тот же.

**Подводные камни:**

- Если вызвать звук и на клиенте, и на сервере, он проиграется дважды. `Level#playSound` на сервере рассылает пакет; на клиенте — играет локально.
- `null` в первом аргументе — «всем игрокам»; `player` — всем, кроме него.
- `SoundSource.PLAYERS` влияет на ползунок «Игроки» в настройках; для UI-звука используйте `MASTER`/`UI`.
- Не играйте звук каждый тик — это быстро превращается в шумовую атаку.
- `this.getX()` в `Player#attack` — позиция атакующего, а не цели; если нужен звук у цели, берите `target.getX()`.

---

### 12. Изменить опыт с блока / моба

**Где искать:** блок — `Block#popExperience(ServerLevel, BlockPos, int)`; моб — `LivingEntity#getExperienceReward()` (1.20.1) и `LivingEntity#getBaseExperienceReward(ServerLevel)` (1.21+). В 1.21 `getExperienceReward(ServerLevel, Entity)` стал `final`-обёрткой, которая учитывает убийцу и множители; переопределять нужно `getBaseExperienceReward`.

**1.20.1**

```java
@Mixin(Block.class)
public abstract class BlockXpMixin {

    @ModifyVariable(method = "popExperience", at = @At("HEAD"), argsOnly = true)
    private int cookbook$doubleBlockXp(int amount) {
        return amount * 2;
    }
}

@Mixin(LivingEntity.class)
public abstract class MobXpMixin {

    @Inject(method = "getExperienceReward", at = @At("HEAD"), cancellable = true)
    private void cookbook$mobXp(CallbackInfoReturnable<Integer> cir) {
        if ((Object) this instanceof Zombie) {
            cir.setReturnValue(10);
        }
    }
}
```

**1.21.x**

```java
@Mixin(Block.class)
public abstract class BlockXpMixin {

    @ModifyVariable(method = "popExperience", at = @At("HEAD"), argsOnly = true)
    private int cookbook$doubleBlockXp(int amount) {
        return amount * 2;
    }
}

@Mixin(LivingEntity.class)
public abstract class MobXpMixin {

    @ModifyReturnValue(method = "getBaseExperienceReward", at = @At("RETURN"))
    private int cookbook$mobXp(int original, ServerLevel level) {
        return (Object) this instanceof Zombie ? 10 : original;
    }
}
```

`getExperienceReward(ServerLevel, Entity)` в 1.21 — `final`; если попытаться его миксинить, получите ошибку или неожиданное поведение. `getBaseExperienceReward(ServerLevel)` — правильная точка.

**26.x** — без изменений относительно 1.21.x: `Block#popExperience(ServerLevel, BlockPos, int)` и `LivingEntity#getBaseExperienceReward(ServerLevel)` сохранились. В 26.1 менялись реестры и компоненты, но не эта пара методов.

**Что изменилось:**

- 1.21: `getExperienceReward()` → `getBaseExperienceReward(ServerLevel)`; `getExperienceReward(ServerLevel, Entity)` стал `final`.
- Сигнатура блочного опыта не изменилась: `popExperience(ServerLevel, BlockPos, int)`.
- 26.x: те же сигнатуры, другие имена вокруг.
- 26.3: `Level#playSound` получил Holder-overload; на опыт не влияет.

**Подводные камни:**

- `getBaseExperienceReward` может вызываться несколько раз (например, при проверке и при фактической смерти); не начисляйте опыт из него напрямую.
- `popExperience` вызывается на сервере; на клиенте его нет, поэтому `@ModifyVariable` безопасен.
- Отрицательный или огромный `amount` ломает `ExperienceOrb`; ограничивайте разумным диапазоном.
- `skipDropExperience`/`isAlwaysExperienceDropper` могут обходить вашу логику; проверяйте флаги.
- В 1.21+ `getExperienceReward` учитывает `Entity#getKillCredit`; если вы заменяете базу, множители всё равно применятся сверху.

---

### 13. Изменить поведение жидкости

**Где искать:** 1.20.1 — `FlowingFluid#canSpreadTo(BlockGetter, BlockPos, BlockState, Direction, BlockPos, BlockState, FluidState, Fluid)`. 1.21.2+ — тот же метод переименован в `canPassThrough(BlockGetter, Fluid, BlockPos, BlockState, Direction, BlockPos, BlockState, FluidState)`. Планирование распространения — `spread`/`spreadToSides` (в 1.21.2+ принимают `ServerLevel` и `BlockState`).

**1.20.1**

```java
@Mixin(FlowingFluid.class)
public abstract class FluidSpreadMixin {

    @Inject(method = "canSpreadTo", at = @At("HEAD"), cancellable = true)
    private void cookbook$blockSpread(BlockGetter level,
                                      BlockPos fromPos, BlockState fromState,
                                      Direction direction,
                                      BlockPos toPos, BlockState toState,
                                      FluidState fluidState, Fluid fluid,
                                      CallbackInfoReturnable<Boolean> cir) {
        if (direction == Direction.DOWN && toState.is(Blocks.MAGMA_BLOCK)) {
            cir.setReturnValue(false);
        }
    }
}
```

Дескриптор: `(Lnet/minecraft/world/level/BlockGetter;Lnet/minecraft/core/BlockPos;Lnet/minecraft/world/level/block/state/BlockState;Lnet/minecraft/core/Direction;Lnet/minecraft/core/BlockPos;Lnet/minecraft/world/level/block/state/BlockState;Lnet/minecraft/world/level/material/FluidState;Lnet/minecraft/world/level/material/Fluid;)Z`.

**1.21.x**

```java
@Mixin(FlowingFluid.class)
public abstract class FluidSpreadMixin {

    @Inject(method = "canPassThrough", at = @At("HEAD"), cancellable = true)
    private void cookbook$blockSpread(BlockGetter level, Fluid fluid,
                                      BlockPos fromPos, BlockState fromState,
                                      Direction direction,
                                      BlockPos toPos, BlockState toState,
                                      FluidState fluidState,
                                      CallbackInfoReturnable<Boolean> cir) {
        if (direction == Direction.DOWN && toState.is(Blocks.MAGMA_BLOCK)) {
            cir.setReturnValue(false);
        }
    }
}
```

В 1.21.2 `canSpreadTo` переименован в `canPassThrough`, а `Fluid` переехал с последнего места на второе. Это ломает и имя, и порядок аргументов handler'а.

**26.x** — без изменений относительно 1.21.x для распространения: `canPassThrough` и `FlowingFluid#spread` сохранились. В 26.1 «Fluid Logic Reorganization» вынес движение сущностей в жидкости в `EntityFluidInteraction` (`updateInWaterStateAndDoFluidPushing` → `updateFluidInteraction`), но это не распространение жидкости. В 26.3 изменения только в рендере (`FluidModel`, `FluidStateModelSet`).

**Что изменилось:**

- 1.21.2: `canSpreadTo` → `canPassThrough`; `Fluid` — второй аргумент.
- 1.21.2: `spread`/`getNewLiquid`/`canConvertToSource` принимают `ServerLevel`; `spread` — ещё и `BlockState`.
- 26.1: движение сущностей в жидкости вынесено в `EntityFluidInteraction`.
- 26.3: `FluidModel`/`FluidStateModelSet` — только рендер.

> [!WARNING]
> `canPassThrough`/`canSpreadTo` вызывается на каждом шаге распространения и на каждом тике жидкости. Любая аллокация или запрос к реестру здесь превращается в лаг-спайк на больших фермах. Кэшируйте результат, если проверка сложная.

**Подводные камни:**

- `FlowingFluid#spreadTo` в 1.20.1 — `abstract`; инъекция в абстрактный метод не сработает. Цельтесь в конкретный `canSpreadTo`/`canPassThrough` или в `WaterFluid`/`LavaFluid`.
- `Direction.DOWN` — не единственный путь; жидкость может растекаться по бокам и вверх.
- `toState` может быть `FluidState`-содержащим блоком (waterlogged); проверяйте `fluidState` и `toState`.
- На клиенте жидкость предсказывается; серверный запрет может вызвать рассинхрон.
- В 26.x `FluidState#getAmount()` может быть удобнее, чем `FlowingFluid#getAmount(FluidState)`.

---

### 14. Добавить эффект при использовании предмета

**Где искать:** `ItemStack#use(Level, Player, InteractionHand)` — единая точка, которая вызывает `Item#use` и возвращает результат. Миксин в `Item` поймает только базовую реализацию; `ItemStack#use` — общий вход.

**1.20.1**

```java
@Mixin(ItemStack.class)
public abstract class UseEffectMixin {

    @Inject(method = "use", at = @At("RETURN"))
    private void cookbook$applyEffect(Level level, Player player,
                                      InteractionHand hand,
                                      CallbackInfoReturnable<InteractionResultHolder<ItemStack>> cir) {
        if (!level.isClientSide && cir.getReturnValue().getResult().consumesAction()) {
            player.addEffect(new MobEffectInstance(MobEffects.GLOWING, 200));
        }
    }
}
```

**1.21.x**

```java
@Mixin(ItemStack.class)
public abstract class UseEffectMixin {

    @Inject(method = "use", at = @At("RETURN"))
    private void cookbook$applyEffect(Level level, Player player,
                                      InteractionHand hand,
                                      CallbackInfoReturnable<InteractionResult> cir) {
        if (!level.isClientSide && cir.getReturnValue().consumesAction()) {
            player.addEffect(new MobEffectInstance(MobEffects.GLOWING, 200));
        }
    }
}
```

С 1.21.2 `InteractionResultHolder` удалён, `Item#use`/`ItemStack#use` возвращают `InteractionResult`. `MobEffects.GLOWING` с 1.20.5 — `Holder<MobEffect>`, а `MobEffectInstance` принимает `Holder<MobEffect>`; текст конструкции не изменился, но типы другие.

**26.x** — без изменений относительно 1.21.x: `ItemStack#use(Level, Player, InteractionHand)` возвращает `InteractionResult`, `InteractionResult` остался sealed-интерфейсом. В 26.3 `GameTestHelper#useItemOnBlock` — удобный способ протестировать этот путь без клиента.

**Что изменилось:**

- 1.21.2: `InteractionResultHolder<ItemStack>` → `InteractionResult`; `Item#use` и `ItemStack#use` поменяли возвращаемый тип.
- 1.20.5: `MobEffects` и `MobEffectInstance` перешли на `Holder<MobEffect>`.
- 1.21.2: `InteractionResult` стал sealed; `consumesAction()` остался.
- 26.x: те же сигнатуры; `ItemStackTemplate` для «шаблонных» стаков, но `use` — про живой `ItemStack`.

**Подводные камни:**

- `ItemStack#use` не вызывается для еды/зелий в момент завершения — там работает `finishUsingItem`. Если эффект нужен после еды, добавьте второй миксин.
- `consumesAction()` истинно для `SUCCESS`/`CONSUME`; `PASS`/`FAIL` — нет. Проверяйте именно результат, а не «предмет не воздух».
- `player.addEffect` на клиенте создаст «фантомный» эффект, который исчезнет; всегда проверяйте `!level.isClientSide`.
- Эффект без cooldown — эксплойт: игрок может кликать быстрее, чем длится эффект.
- На 1.21.2+ `InteractionResult.Success#heldItemTransformedTo` может заменить предмет; не игнорируйте `ItemContext`.

---

### 15. Отменить ванильный вызов: `@WrapOperation` вместо `@Redirect`

**Где искать:** `Player#attack(Entity)` вызывает `Entity#hurt(DamageSource, float)`. Это классический пример: `@Redirect` заменяет вызов и конфликтует с другими модами, `@WrapOperation` оборачивает и позволяет вызвать оригинал. В 1.21.2+ `hurt` deprecated, а реальная работа уходит в `hurtServer`; если `attack` вызывает `hurt`, дескриптор не меняется.

**1.20.1 — как не надо (`@Redirect`)**

```java
// Плохо: два мода с @Redirect на один вызов несовместимы.
@Redirect(
    method = "attack",
    at = @At(value = "INVOKE",
        target = "Lnet/minecraft/world/entity/Entity;hurt(Lnet/minecraft/world/damagesource/DamageSource;F)Z")
)
private boolean cookbook$redirectHurt(Entity target, DamageSource source, float amount) {
    return false; // полностью заменяет вызов
}
```

**1.20.1 — как надо (`@WrapOperation`)**

```java
@Mixin(Player.class)
public abstract class AttackWrapMixin {

    @WrapOperation(
        method = "attack",
        at = @At(value = "INVOKE",
            target = "Lnet/minecraft/world/entity/Entity;hurt(Lnet/minecraft/world/damagesource/DamageSource;F)Z")
    )
    private boolean cookbook$guardHurt(Entity target, DamageSource source,
                                       float amount, Operation<Boolean> original) {
        if (target instanceof Player victim && PvpZone.isProtected(victim)) {
            return false;
        }
        return original.call(target, source, amount);
    }
}
```

**1.21.x** — дескриптор `Entity;hurt(DamageSource;F)Z` тот же, `@WrapOperation` тот же; `hurt` помечен deprecated, но ещё существует. `[unverified]` — что `Player#attack` в вашей 1.21.x-сборке вызывает именно `hurt`, а не `hurtServer`/`hurtOrSimulate`. Проверьте `javap -c` или `-Dmixin.debug.verbose=true`: если якорь не находится, Mixin напишет, какие вызовы есть в методе.

**26.x**

```java
// [unverified] 26.x: проверить, что Player#attack вызывает hurtServer напрямую
@Mixin(Player.class)
public abstract class AttackWrapMixin {

    @WrapOperation(
        method = "attack",
        at = @At(value = "INVOKE",
            target = "Lnet/minecraft/world/entity/Entity;hurtServer(Lnet/minecraft/server/level/ServerLevel;Lnet/minecraft/world/damagesource/DamageSource;F)Z")
    )
    private boolean cookbook$guardHurt(Entity target, ServerLevel level,
                                       DamageSource source, float amount,
                                       Operation<Boolean> original) {
        if (target instanceof Player victim && PvpZone.isProtected(victim)) {
            return false;
        }
        return original.call(target, level, source, amount);
    }
}
```

`[unverified]` — что вызов в `attack` идёт напрямую в `hurtServer`. Если нет, используйте `hurt` с дескриптором из 1.21.x: deprecated-обёртка в 26.x ещё присутствует.

Тот же шов можно описать структурно, без `ordinal` и `shift`:

```java
@Definition(
    id = "hurt",
    method = "Lnet/minecraft/world/entity/Entity;hurt(Lnet/minecraft/world/damagesource/DamageSource;F)Z"
)
@Expression("?.hurt(?, ?)")
@WrapOperation(method = "attack", at = @At("MIXINEXTRAS:EXPRESSION"))
private boolean cookbook$guardHurt(Entity target, DamageSource source,
                                   float amount, Operation<Boolean> original) {
    if (target instanceof Player victim && PvpZone.isProtected(victim)) {
        return false;
    }
    return original.call(target, source, amount);
}
```

Для 26.x в `@Definition` подставляется `hurtServer` с `ServerLevel` в дескрипторе, а в выражении — `?.hurtServer(?, ?, ?)`. `@Expression` не отменяет дескриптор, но убирает зависимость от позиции вызова и делает намерение читаемым. Это особенно полезно, когда в методе несколько похожих вызовов.

**Что изменилось:**

- 1.21.2: `hurt` deprecated; появился `hurtServer(ServerLevel, DamageSource, float)`.
- 26.x: `hurtServer` — основной путь; дескриптор включает `ServerLevel`.
- `@WrapOperation` доступен через MixinExtras: в NeoForge он встроен, на 1.20.1 Forge его нужно подключать отдельно (jar-in-jar).
- `@Redirect` не цепляется с другими модами; `@WrapOperation` цепляется, и каждый мод получает `Operation`, представляющую предыдущую обёртку.

**Подводные камни:**

- `@WrapOperation` должен найти **ровно один** вызов; если в методе два `hurt`, добавьте `ordinal` или более точный `@At`.
- `Player#attack` вызывается на обеих сторонах. Если блокировать `hurt` только на клиенте, предсказание разойдётся с сервером; если только на сервере — клиент проиграет анимацию. Для PvP-защиты решение должно быть одинаковым на обеих сторонах.
- `original.call(...)` должен получить все аргументы в правильном порядке; забытый `ServerLevel` в 26.x — типичная ошибка.
- Возврат `false` не отменяет knockback, sweep attack и другие эффекты, которые `attack` делает **вне** `hurt`. Если нужна полная отмена, ставьте `@Inject(HEAD, cancellable=true)` в `attack`.
- `@Redirect` всё ещё встречается в старых гайдах; не копируйте его без необходимости.
- На 1.20.1 Forge без MixinExtras `@WrapOperation` не скомпилируется; проверяйте зависимости.

---

### 16. Изменить текст / локализацию на лету

**Где искать:** клиентский `Language#getOrDefault(String key, String fallback)`; его вызывает `TranslatableContents#visit` при отрисовке. `Language#getInstance()` пересоздаётся при reload ресурсов, поэтому миксин должен быть в client-конфиге.

**1.20.1**

```java
@Mixin(Language.class)
public abstract class TranslationMixin {

    @ModifyReturnValue(
        method = "getOrDefault(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;",
        at = @At("RETURN")
    )
    private String cookbook$override(String original, String key, String fallback) {
        if (key.equals("item.cookbook.mystery_ingot")) {
            return "Таинственный ингредиент";
        }
        return original;
    }
}
```

**1.21.x** — без изменений: `Language#getOrDefault(String, String)` и `Language#getInstance()` сохранились. Единственное отличие — вокруг: в 1.21.2 `SoundEvent` стал record, а resource pack формат поднялся, но API `Language` не трогали.

**26.x** — без изменений: `Language#getOrDefault` остался. В 26.x изменились формат ресурспака и способ загрузки языков, но точка перевода — тот же метод. `[unverified]` — если в вашей сборке `Language` переехал в другой пакет, компилятор покажет это сразу.

**Что изменилось:**

- В этом шве — ничего за три эпохи.
- 26.x: resource pack format и `Identifier` для путей, но ключи переводов — те же строки.
- 1.21.2: `SoundEvent` record и прочие изменения не касаются `Language`.
- 26.3: `TooltipProvider.Getter` и тултипы — не про локализацию.

> [!WARNING]
> `Language` — **клиентский** класс. На сервере его нет; серверные сообщения нужно локализовать через `Component.translatable` и отправлять ключ, а не готовую строку. Если вы подменяете перевод глобально, это затронет все предметы/блоки с тем же ключом, включая чужие моды.

**Подводные камни:**

- `getOrDefault(String)` (один аргумент) — другой overload; если нужен и он, добавьте второй миксин.
- Не кэшируйте `Language` в статическом поле: после reload это другой экземпляр.
- Возврат `null` сломает рендер; всегда возвращайте исходную строку, если не уверены.
- Хардкод русского текста ломает другие языки; лучше подменять ключ на другой ключ или использовать конфиг.
- На 26.x ключи могут быть `Identifier`-based в datapack-переводах; проверяйте формат.

---

### 17. Добавить кнопку в экран (Screen)

**Где искать:** `Screen#init()` — вызывается при открытии и при resize; `addRenderableWidget` добавляет виджет в текущий экран. Для примера миксиним `InventoryScreen`, а не `Screen`, иначе кнопка появится везде.

**1.20.1**

```java
@Mixin(InventoryScreen.class)
public abstract class InventoryButtonMixin {

    @Inject(method = "init", at = @At("TAIL"))
    private void cookbook$addButton(CallbackInfo ci) {
        this.addRenderableWidget(
            Button.builder(Component.literal("Cookbook"), button ->
                    Minecraft.getInstance().setScreen(new CookbookScreen())
                )
                .bounds(this.width - 110, 6, 100, 20)
                .build()
        );
    }
}
```

**1.21.x** — без изменений: `Screen#init()`, `addRenderableWidget`, `Button.builder(...).bounds(...).build()` сохранились. В 1.21.2 `AbstractContainerScreen#render` стал вызывать `renderTooltip` сам, но на добавление кнопки это не влияет.

**26.x** — без изменений для `init`/`addRenderableWidget`: жизненный цикл экрана не менялся. Меняется **рендер**: `AbstractWidget#renderWidget` → `extractWidgetRenderState`, `Screen#render` → `extractRenderState`, `GuiGraphics` → `GuiGraphicsExtractor`. Если вы переопределяете рендер кнопки, код придётся переписать; если только добавляете стандартную `Button` — нет.

**Что изменилось:**

- 26.1: `render*`/`draw*` методы GUI → `extract*`/`extract*RenderState`; `GuiGraphics` → `GuiGraphicsExtractor`.
- 26.1: `AbstractContainerScreen` принимает ширину/высоту фона в конструкторе; `imageWidth`/`imageHeight` стали final.
- 26.1: `AbstractContainerScreen#render` сам вызывает `renderTooltip`; не переопределяйте его без нужды.
- 1.21.2: `AbstractContainerMenu#clicked` принимает `ContainerInput` вместо `ClickType`.

**Подводные камни:**

- `init` вызывается при каждом resize; `addRenderableWidget` каждый раз создаёт новый виджет, но старые очищаются — дубликатов не будет, если не хранить их в статике.
- `this.width`/`this.height` — размеры экрана, а не окна; на больших GUI они уже учитывают scale.
- `Minecraft.getInstance().setScreen(...)` — только на клиенте; миксин должен быть в client-конфиге.
- На 26.x не вызывайте `graphics.drawString` — метода больше нет; для кастомной кнопки используйте `extractWidgetRenderState` и `GuiGraphicsExtractor#text`.
- Кнопка может перекрыть ванильные элементы; проверяйте `this.width - 110` и не забывайте про `leftPos`/`topPos`.

---

### 18. Изменить спавн моба

**Где искать:** `NaturalSpawner#isValidSpawnPostitionForType(ServerLevel, MobCategory, StructureManager, ChunkGenerator, MobSpawnSettings.SpawnerData, BlockPos.MutableBlockPos, double)` — проверка «можно ли здесь заспавнить этот тип». Обратите внимание на опечатку `Postition` в ванильном имени: она дожила до 1.21.11. Альтернатива — `SpawnPlacements#checkSpawnRules` / `Mob#checkSpawnRules`.

**1.20.1**

```java
@Mixin(NaturalSpawner.class)
public abstract class SpawnControlMixin {

    @Inject(method = "isValidSpawnPostitionForType", at = @At("HEAD"), cancellable = true)
    private static void cookbook$blockSpawn(ServerLevel level, MobCategory category,
                                            StructureManager structures,
                                            ChunkGenerator generator,
                                            MobSpawnSettings.SpawnerData data,
                                            BlockPos.MutableBlockPos pos,
                                            double distance,
                                            CallbackInfoReturnable<Boolean> cir) {
        if (data.type == EntityType.CREEPER && level.dimension() == Level.OVERWORLD) {
            cir.setReturnValue(false);
        }
    }
}
```

**1.21.x** — без изменений: имя с опечаткой `isValidSpawnPostitionForType` и сигнатура сохранились (проверено по 1.21.11). В 1.21.2 `NaturalSpawner#spawnForChunk` разбили на `getFilteredSpawningCategories` и `spawnForChunk`, но проверка позиции не изменилась.

**26.x** — `[unverified]` — возможно, опечатку наконец исправили на `isValidSpawnPositionForType`. В 26.3 `NaturalSpawner#createState` принимает `ServerLevel`, а `$SpawnPredicate#test` — `ServerLevel`; это соседние методы, не сама проверка. Если якорь не находится, ищите оба варианта имени:

```java
// [unverified] 26.x: опечатка Postition могла быть исправлена
@Mixin(NaturalSpawner.class)
public abstract class SpawnControlMixin {

    @Inject(method = "isValidSpawnPositionForType", at = @At("HEAD"), cancellable = true)
    private static void cookbook$blockSpawn(ServerLevel level, MobCategory category,
                                            StructureManager structures,
                                            ChunkGenerator generator,
                                            MobSpawnSettings.SpawnerData data,
                                            BlockPos.MutableBlockPos pos,
                                            double distance,
                                            CallbackInfoReturnable<Boolean> cir) {
        // ...
    }
}
```

**Что изменилось:**

- 1.21.2: `NaturalSpawner#spawnForChunk` разделён; `getFilteredSpawningCategories` стал отдельным шагом.
- 26.3: `createState` принимает `ServerLevel`; `$SpawnPredicate#test` — `ServerLevel`.
- 26.x: возможен фикс опечатки `Postition` → `Position`; проверяйте фактическое имя.
- Сама сигнатура проверки (ServerLevel, MobCategory, StructureManager, ChunkGenerator, SpawnerData, MutableBlockPos, double) стабильна с 1.20.1.

> [!WARNING]
> `isValidSpawnPostitionForType` вызывается на **каждую** попытку спавна, для каждого типа и каждой категории. Тяжёлая логика здесь убивает TPS на прогруженных чанках. Кэшируйте по `(dimension, biome, entityType)` и не сканируйте мир.

**Подводные камни:**

- `data.type` — `EntityType<?>`; сравнивать с `EntityType.CREEPER` можно, но для модовых типов используйте `EntityTypeTags`.
- `distance` — квадрат расстояния до игрока; не путайте с блоками.
- `MobSpawnSettings.SpawnerData` может быть `null` в некоторых путях; проверяйте.
- Отмена только `isValidSpawnPostitionForType` не отменяет спавн через `/summon`, спавн-яйца, структуры и `SpawnPlacements`.
- На 26.x `NaturalSpawner` может быть переписан под data-driven spawn; проверяйте primer для своей версии.

---

## Сводка: что меняется чаще всего

| Шов | 1.20.1 | 1.21.x | 26.x | Риск при портировании |
| --- | --- | --- | --- | --- |
| Урон | `hurt(DamageSource, float)` | `hurtServer(ServerLevel, DamageSource, float)` | то же | средний |
| Тултип | `getTooltipLines(Player, TooltipFlag)` | `getTooltipLines(TooltipContext, Player, TooltipFlag)` | то же | низкий |
| HUD | `Gui#render(GuiGraphics, float)` | `(GuiGraphics, DeltaTracker)` | `extractRenderState(GuiGraphicsExtractor, DeltaTracker)`; 26.2+ `Hud` | **высокий** |
| Блок | `BlockBehaviour#use(...)` | `useItemOn(ItemStack, ...)` | то же | средний |
| BlockEntity | `serverTick(Level, ...)` | `serverTick(ServerLevel, ...)` | то же | низкий |
| Стак | `Item#getMaxStackSize` | `ItemStack#getMaxStackSize` | `ItemInstance#getMaxStackSize` | средний |
| Слот | `Slot#mayPlace` | то же | то же | низкий |
| Goal | `Mob#registerGoals` | то же | то же | низкий |
| Payload | raw `ResourceLocation + FriendlyByteBuf` | `CustomPacketPayload` | `Identifier` вместо `ResourceLocation` | **высокий** |
| Крафт | `ResultSlot#onTake` | то же | то же (рецепт — `ItemStackTemplate`) | низкий |
| Звук | `level().playSound` | то же | то же | низкий |
| Опыт | `getExperienceReward()` | `getBaseExperienceReward(ServerLevel)` | то же | низкий |
| Жидкость | `canSpreadTo` | `canPassThrough` | то же | средний |
| Использование | `InteractionResultHolder` | `InteractionResult` | то же | средний |
| `@WrapOperation` | `hurt` | `hurt` (deprecated) | `hurtServer` | средний |
| Локализация | `Language#getOrDefault` | то же | то же | низкий |
| Screen | `init` + `addRenderableWidget` | то же | то же | низкий |
| Спавн | `isValidSpawnPostitionForType` | то же | возможно, `...Position...` | низкий |

## Проверено на

- Дата: **2026-11-21**.
- Ориентир по 26.x: **NeoForge 26.3.0.23-beta** (как в README); базовая линия кода — 26.1, потому что 26.2/26.3 меняют часть GUI-сигнатур.
- 1.20.1: сигнатуры сверены с mappings.dev (Mojmap, Forge-эпоха) и ванильными классами; поведение проверялось по декомпиляции в общих чертах, не построчно.
- 1.21.x: базовая линия — **1.21.11** (последняя версия старой схемы), с отличиями 1.21.2/1.21.5 по NeoForge primer'ам.
- 26.x: базовая линия — **26.1**, с отличиями 26.2/26.3 по NeoForge primer'ам. Текущая версия 26.3 требует отдельной проверки для GUI/HUD.
- Код **не запускался в игре** в рамках этого документа: это рецепты и карта изменений, а не протестированный мод. Перед использованием проверяйте `-Dmixin.debug.export=true` и `-Dmixin.debug.verbose=true`.
- `[unverified]`-пункты: точная сигнатура `Gui#extractRenderState` в 26.1; миксин в `ItemInstance` для 26.x; имя `isValidSpawnPositionForType` в 26.x; вызов `hurtServer` из `Player#attack` в 26.x; принадлежность `assemble` к `ShapedRecipe` в 26.x.
- 26.x-имена `Identifier`/`ItemStackTemplate`/`GuiGraphicsExtractor` взяты из NeoForge primer'ов 26.1–26.3; в вашей сборке IDE может показывать старые имена из-за кэша mappings.

## Источники

- Fabric Wiki — Mixin tutorial: <https://wiki.fabricmc.net/tutorial:mixin:injection>
- SpongePowered Mixin — javadoc и wiki: <https://github.com/SpongePowered/Mixin/wiki>
- MixinExtras wiki — `@WrapOperation`, `@ModifyReturnValue`, `@Local`, `@Expression`: <https://github.com/LlamaLad7/MixinExtras/wiki>
- NeoForge Documentation — Primers: <https://docs.neoforged.net/primer/docs/>
  - 1.20.5: <https://docs.neoforged.net/primer/docs/1.20.5/>
  - 1.21: <https://docs.neoforged.net/primer/docs/1.21/>
  - 1.21.2: <https://docs.neoforged.net/primer/docs/1.21.2/>
  - 1.21.5: <https://docs.neoforged.net/primer/docs/1.21.5/>
  - 26.1: <https://docs.neoforged.net/primer/docs/26.1/>
  - 26.2: <https://docs.neoforged.net/primer/docs/26.2/>
  - 26.3: <https://docs.neoforged.net/primer/docs/26.3/>
- mappings.dev — сверка сигнатур 1.20.1 и 1.21.11: <https://mappings.dev/>
- NeoForge Docs — события, реестры, data components: <https://docs.neoforged.net/>

---

[← К оглавлению](README.md)
