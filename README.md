# Minecraft Modding Research

Практическая база знаний по моддингу **Minecraft Java Edition**: какими инструментами пользуются мододелы, как они находят цель для миксина и как менялся API игры от **1.7.10** до **26.3**.

![Minecraft](https://img.shields.io/badge/Minecraft-1.7.10%20...%2026.3-3c8527)
![Java](https://img.shields.io/badge/Java-8%20...%2025-e76f00)
![Loaders](https://img.shields.io/badge/loaders-Forge%20%7C%20NeoForge%20%7C%20Fabric%20%7C%20Quilt-4b6cb7)
![License](https://img.shields.io/badge/license-MIT-blue)

---

## Зачем это

Большая часть знаний о моддинге живёт в Discord, YouTube и разрозненных issue-тредах. Здесь собрано то, что обычно приходится выяснять самому:

- **Чем работать** — от Gradle-плагинов и декомпиляторов до профилировщиков и CI.
- **Как понять, куда встраивать миксин** — не «вот пример», а маршрут: где смотреть, что читать в байткоде, как выбрать инъектор и как проверить, что он вообще применился.
- **Что сломалось между версиями** — breaking changes по эпохам, с таблицами «было → стало» и последствиями для миксинов.

## Что внутри

| Документ | О чём | Статус |
| --- | --- | --- |
| [`00-glossary.md`](00-glossary.md) | Термины: от `refmap` и `intermediary` до `@WrapOperation` | ✅ |
| [`01-toolchain/1-build-systems.md`](01-toolchain/1-build-systems.md) | Loom, ForgeGradle, ModDevGradle, NeoGradle, RFG, Unimined | ✅ |
| [`01-toolchain/2-ide-and-navigation.md`](01-toolchain/2-ide-and-navigation.md) | IntelliJ, Ravel, MixinMCP, jdx, Linkie, mappings.dev | ✅ |
| [`01-toolchain/3-decompilers-bytecode.md`](01-toolchain/3-decompilers-bytecode.md) | Vineflower, CFR, Recaf, `javap`, ASM | ✅ |
| [`01-toolchain/4-debug-and-diagnostics.md`](01-toolchain/4-debug-and-diagnostics.md) | `-Dmixin.debug.*`, MixinTrace, hotswap, spark, JFR | ✅ |
| [`01-toolchain/5-content-and-pipeline.md`](01-toolchain/5-content-and-pipeline.md) | Datagen, Blockbench, KubeJS, NBT-редакторы, публикация | ✅ |
| [`01-toolchain/6-tool-matrix.md`](01-toolchain/6-tool-matrix.md) | Сводная таблица всех инструментов | ✅ |
| [`02-mixin-playbook.md`](02-mixin-playbook.md) | **Ядро:** как найти цель и выбрать способ инъекции | ✅ |
| [`03-mixin-cookbook.md`](03-mixin-cookbook.md) | Рецепты инъекций: 1.20.1 / 1.21.x / 26.x | ✅ |
| [`04-mixin-debugging.md`](04-mixin-debugging.md) | Почему миксин не применился и что с этим делать | ✅ |
| [`05-version-changes/`](05-version-changes/00-index.md) | Эволюция API по эпохам | ⏳ |
| [`06-loaders-and-mappings.md`](06-loaders-and-mappings.md) | Лоадеры, mappings, совместимость | ✅ |
| [`07-sources.md`](07-sources.md) | Аннотированные первоисточники | ✅ |

## Как читать

**Если вы пишете первый миксин** → [`00-glossary.md`](00-glossary.md) → [`02-mixin-playbook.md`](02-mixin-playbook.md) → [`03-mixin-cookbook.md`](03-mixin-cookbook.md).

**Если портируете мод** → [`05-version-changes/00-index.md`](05-version-changes/00-index.md) → нужная эпоха → [`04-mixin-debugging.md`](04-mixin-debugging.md).

**Если выбираете инструменты** → [`01-toolchain/6-tool-matrix.md`](01-toolchain/6-tool-matrix.md), затем конкретный файл.

## Ключевые выводы

1. **26.1 — водораздел.** Mojang убрал обфускацию: официальные имена доступны в рантайме, Yarn/intermediary/refmap для 26.x больше не нужны, `officialMojangMappings()` в Loom бросает ошибку. См. [`05-version-changes/6-unobfuscated.md`](05-version-changes/6-unobfuscated.md).
2. **Маршрут «найти цель» изменился.** До 26.1 приходилось держать в голове mappings и `remap = false` для модовых целей; с 26.1 имена в миксине — это имена в рантайме. См. [`02-mixin-playbook.md`](02-mixin-playbook.md).
3. **`@Expression` вытесняет `ordinal` и `@Slice`.** Позиционные якоря ломаются от любого рефакторинга; структурные выражения — нет. См. [`02-mixin-playbook.md`](02-mixin-playbook.md).
4. **MixinExtras — дефолт, а не «плюшка».** `@WrapOperation` / `@ModifyExpressionValue` / `@ModifyReturnValue` цепляются друг за другом, `@Redirect` и `@Overwrite` — нет.
5. **1.20.5 (Data Components) — второй водораздел.** `ItemStack` NBT заменён компонентами; в 26.1 добавился `ItemStackTemplate`, потому что `ItemStack` теперь требует загруженных реестров.
6. **26.1 переписал рендер.** `GuiGraphics` → `GuiGraphicsExtractor`, `Screen#render` → `extractRenderState`; миксины в GUI нужно переписывать, а не переименовывать.
7. **Сборка потяжелела.** Java 25, Gradle 9.2+, Loom расколот на `fabric-loom` и `fabric-loom-remap`, NeoForge перешёл на 4-компонентные версии.
8. **26.2–26.3 переписали графический слой.** Экспериментальный Vulkan, вынос рендер-API в **Renderpearl**, переход с GLFW на **SDL3** и OIT. Моды с прямыми OpenGL-вызовами и кастомными шейдерами ломаются сильнее, чем от смены mappings.

## Конвенции страницы

Каждый документ следует одному шаблону:

```text
# Заголовок
> [!NOTE] TL;DR — 2–4 строки, зачем читать

## Содержание
## <разделы>
## Проверено на
## Источники
```

- Даты и версии указываются явно: «проверено 2026-11-21 на NeoForge 26.3.0.23-beta».
- Неподтверждённое помечается: `[draft]`, `[community]`, `[unverified]`.
- Ссылки только на первоисточники: официальные доки, вики, исходники лоадеров.
- Код — с указанием версии и лоадера, потому что один и тот же миксин в 1.20.1 и 26.3 выглядит по-разному.

## Источники и честность

Это исследование, а не истина в последней инстанции. Экосистема 26.x молодая: часть документации ещё в статусе draft, часть практик существует только в community. Всё, что не удалось подтвердить первоисточником, помечено явно — проверяйте на своей версии.

## Лицензия

[MIT](LICENSE)
