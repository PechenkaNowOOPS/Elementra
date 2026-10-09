# Elementra — техническая документация кузнечной системы ForgeSystem

**Редакция:** 1.0 / 8 октября 2026  
**Тип:** интегрированная инженерная спецификация, не свидетельство выпуска  
**Платформа:** Purpur 26.2 / совместимый Paper API, Java 25  
**Клиент:** ванильный Minecraft Java, необязательный серверный resource pack  
**Игровой документ:** [`Elementra — Механика ковки.md`](./forge.md)

> **Разделение фактов и требований.** Сведения о dev-сборках и исходниках ниже являются *снимком, описанным в переданных документах* (в основном 04.10.2026); я не выполнял сборку, runtime-test, осмотр исходников ZIP или проверку JAR при подготовке этого файла. Ключевое слово **MUST** означает нормативное требование целевого проекта. Примеры конфигов и предложенные сигнатуры не следует автоматически считать буквальным содержимым текущего репозитория.

## Содержание

1. [Приоритет редакций и текущий статус](#1-приоритет-редакций-и-текущий-статус)
2. [Архитектура и границы владения](#2-архитектура-и-границы-владения)
3. [Предметные определения и схема Workpiece](#3-предметные-определения-и-схема-workpiece)
4. [Состояния и переходы](#4-конечный-автомат-и-инварианты)
5. [Структура станции и активация](#5-физический-горн-и-активация)
6. [Симуляция нагрева, топлива, Workpiece](#6-тепловые-модели-и-топливо)
7. [Создание заготовок](#7-транзакция-создания-hot_blank)
8. [GUI 54 слота и графический интерфейс](#8-gui-горна-и-ресурспак)
9. [Мини-игра и вычислительные правила](#9-ядро-forgeminigame-v24)
10. [HUD мини-игры, звук, fallback](#10-hud-мини-игры-и-feedback)
11. [Связь Workpiece и активной ForgeSession](#11-интеграция-workpiece-и-forgesession)
12. [Reheat и температура при остановках](#12-остывание-и-повторный-нагрев)
13. [Закалка, рукояти, улучшения](#13-переходы-после-ковки)
14. [События, адаптеры и API](#14-api-и-событийные-контракты)
15. [Атомарность, безопасность, восстановление](#15-безопасность-транзакции-и-recovery)
16. [Конфигурация и безопасная перезагрузка](#16-конфигурация-и-reload)
17. [Команды, разрешения, сборка](#17-команды-permissions-сборка)
18. [Производительность и эксплуатация](#18-производительность-и-эксплуатация)
19. [Матрица испытаний](#19-тесты-и-критерии-приёмки)
20. [План внедрения и технический долг](#20-план-интеграции-и-технический-долг)
21. [Решения, расхождения и открытые параметры](#21-разночтения-и-открытые-вопросы)
22. [Источники, трассировка требований и глоссарий](#22-источники-и-трассировка)

---

## 1. Приоритет редакций и текущий статус

### 1.1. Уровни доказательности

В мастер-документации Elementra определены ступени:

```text
SPEC → SOURCE_READY → COMPILED → STANDALONE_TESTABLE
     → INTEGRATED → RUNTIME_VERIFIED → RELEASE_READY
```

Ступень нельзя повышать автоматически из-за наличия JAR или описания тестов. Соответственно:

| Модуль | Основание текущего статуса | Что известно / что нет |
|---|---|---|
| `ForgeMinigame` | v2.4 / `0.6.0-SNAPSHOT` | Самостоятельная реакционная мини-игра с core, HUD, cooldown/reheat; **не** готовый ForgeSystem |
| `ForgeSystem` | `3.1.0-dev` | По сводке есть физический BLAST_FURNACE-многоблок, GUI, heat/fuel, PDC HOT_BLANK, 9 изделий, RP; **не** интегрированы жизненный цикл и финальный результат |
| `ForgeItems` | SPEC / contract-ready | Definitions, schema, codec/API, lock, models, идентификаторы описаны; production ownership и интеграционные испытания не подтверждены |
| Downstream | SPEC | `UNQUENCHED`, закалка, сборка рукояти, material upgrade составляют обязательные целевые стадии, но не подтверждён готовый runtime |

**Правило приоритета источников в этом файле:** (1) последние статусные данные Elementra v3.1.1 / roadmap 1.1.1; (2) явные production-контракты ForgeSystem Full Mechanics и ForgeItems; (3) GUI & RP ТЗ v2 (03.10.2026) для актуальной slot-map и visual-UX; (4) интеграционный документ для переноса v2.4; (5) ForgeMinigame v2.4 для текущей математики HUD и прототипа; (6) Forge Mechanics 0.2.0-SNAPSHOT для референсных тестовых значений станции. Противоречия не замалчиваются — см. §21.

### 1.2. Основные инварианты

- Сервер — единственный владелец решений о попадании, температуре, состоянии, расходе, выдаче и качестве. Display name, lore, пиксели GUI, `CustomModelData` не являются идентичностью.
- Одна реальная заготовка (`instance_id`) — не более одной активной сессии и одного результата критического перехода.
- Между сессиями состояние живёт в **Workpiece**, а не в `ForgeSession`; GUI/HUD — представления, не транзакционная БД.
- Создание `HOT_BLANK` разрешено только реальному рабочему горну и из `IRON`/`GOLD`; `DIAMOND` и `NETHERITE` — лишь последующий tier готовых изделий.
- Reheat меняет тепловой якорь, **не** поднимает quality и `max_reachable_stage`.
- Первый допустимый ЛКМ по наковальне начинает сессию, но никогда не оценивается как удар.
- `UNQUENCHED` ещё не экипировка; `FINISHED` материализуется в реальный vanilla-equipment ItemStack с кузнечными metadata.

## 2. Архитектура и границы владения

### 2.1. Целевые модули

```text
forge-api/                 Публичные идентификаторы, DTO, результаты и события
forge-core/                Чистая Java: стадии, качество, температура, продукты
forge-paper/               Paper events, станции, GUI, предметы, storage, recovery
forge-integration-tests/   E2E/integration проверка событий и отказов
forge-resourcepack/        GUI, glyph, температурные кадры, Workpiece, молот, HUD

ForgeItems (логический модуль, API и Item Registry)
  → Item Definition / WorkpieceCodec / ItemStack transitions
ForgeSystem (production plugin)
  ├─ ForgeStation (multiblock, heat, fuel, GUI)
  ├─ ForgeSessionManager (runtime sessions + locks)
  ├─ Forging/ForgeMinigame (3 этапа, точная математика)
  ├─ CauldronAdapter (quenching)
  ├─ HandleAssembly (anvil)
  └─ SmithingAdapter (diamond/netherite)
```

ForgeMinigame после слияния **не запускается отдельным игровым production-модулем**, хотя его core, renderer и StageRules сохраняются. `ForgeItems` может иметь отдельную модульную упаковку, но владельцем доменного решения «когда менять предмет» остаётся ForgeSystem.

### 2.2. Владелец состояния

| Домен / данные | Владелец | Запрещённая подмена |
|---|---|---|
| `ForgeStationState` | ForgeStation / ForgeSystem | Title GUI / ambient vanilla-furnace состояние |
| Определение предмета, PDC/версия, codec, проверка подлинности | ForgeItems | Прямой PDC из каждого listener |
| Текущий прогресс **между** ковками | Workpiece (`ItemStack` + persistent snapshot) | `Map<UUID, ForgeSession>` как единственное хранилище |
| Ползунок, зона, таймеры **активной** ковки | Чистый `ForgeSession` | GUI glyph index вместо нормализованного значения |
| Выходной готовый предмет и разрешение перехода | ForgeSystem вызывает ForgeItems transition | Автоматически выполненный `PrepareItemCraft` или display name |
| Распределение CustomModelData/model keys | `ServerPlatform Item Registry` (согласно ForgeItems spec) | Разрозненные magic model ID |

### 2.3. Классы: сохранить и разделить

| v2.4 source class | Production направление |
|---|---|
| `ForgeOutcome`, `ForgeQuality` | Сохранить core; привести quality enum к общему контракту |
| `StageRules`, `sliderPosition()`, `randomizeZone()` | Сохранить чистую математику и конфиг |
| `ForgeSession` | Расширить конструктор от фактической Workpiece/seed, не от команды |
| `ForgeGuiGlyphs`, `ForgeHudRenderer`, `ForgeHudPopup`, `ForgeHudStatus` | Сохранить HUD renderer с адаптацией и проверкой режима RP |
| `ForgeMinigamePlugin` | Разбить монолит на listeners/managers/services и исключить из production flow |

Проектные компоненты GUI: `ForgeStationManager`, `ForgeStationRepository`, `ForgeMultiblockValidator`, `ForgeMenuService`, `ForgeMenuListener`, `ForgeTitleRenderer`, `ForgeResourcePackTracker`, `ForgeAnvilAdapter`, `ForgeCauldronAdapter`, `ForgeScheduler`, `ForgeCommands`. В опубликованных источниках их наличие в JAR не проверялось; это **целевой состав**.

## 3. Предметные определения и схема Workpiece

### 3.1. Item Definitions

| ID | Требование | Роль |
|---|---|---|
| `forgesystem:workpiece` | MUST | Одна физическая деталь в процессе |
| `forgesystem:hammer` | MUST | Рабочий молот для ковки и активации станции |
| `forgesystem:guide` | MAY | Справочник/книга сборки; потеря не блокирует игру |

Внутренний модуль именуется `ForgeItems`, но внешний namespace остаётся `forgesystem` ради совместимости. **Не** вводить отдельные Item Definition для `hot_sword_blank`, `iron_blade`, `pickaxe_head`, `unquenched_chestplate`, `quenched_axe_head`, `handle`, `diamond_blank`, `netherite_blank`: это значения состояния/изделия, vanilla-компоненты или готовый material tier.

### 3.2. Нормативная схема одного экземпляра

```text
item_definition = "forgesystem:workpiece"
item_schema: int                        # обязательная версия формата
instance_id: UUID                       # уникален для каждой детали
material: IRON | GOLD | DIAMOND | NETHERITE
product_type: SWORD | PICKAXE | AXE | SHOVEL | HOE |
              HELMET | CHESTPLATE | LEGGINGS | BOOTS
forge_state: HOT_BLANK | FORGING | UNQUENCHED |
             QUENCHED_PART | FINISHED
quality: NONE | GOOD | EXCELLENT | MASTERWORK
quality_stage: 0..3
current_stage: 1..3
max_reachable_stage: 1..3
blacksmith_uuid: UUID | null
session_id: UUID | null
temperature_value: double
temperature_updated_at_epoch_ms: long
config_revision: id
```

**Уточнение:** `IRON`/`GOLD` — допустимые материалы *создания Workpiece в горне*. `DIAMOND`/`NETHERITE` включены в общую модель предмета для последующих upgrade готового изделия; они **не** являются допустимыми входными слитками станции. `FINISHED` может существовать в доменном snapshot, но игроку выдаётся настоящий ванильный ItemStack. Вариант хранения Forge metadata на нём (`WorkpieceSnapshot` или `FinishedForgeItemSnapshot`) ещё не зафиксирован.

Ключи читаются через один codec, не по имени/lore. Поля `current_stage` и `max_reachable_stage` обязаны сохраняться для продолжения после остывания. `temperature_updated_at_epoch_ms` — **wall-clock** метка, в отличие от `System.nanoTime`, используемого для анимации активной сессии.

### 3.3. Первичное состояние после выдачи

```text
forge_state = HOT_BLANK
quality = NONE
quality_stage = 0
current_stage = 1
max_reachable_stage = 3
blacksmith_uuid = null
session_id = null
instance_id = new UUID
T = min(material.initial-after-forge, station.temperature)
temperature_updated_at_epoch_ms = now
config_revision = snapshot id
```

### 3.4. Молот

Нормативная идентичность: `item_definition = forgesystem:hammer`, `item_schema = 1`. Старый тестовый молот ForgeMinigame — `IRON_AXE`, `unbreakable`, скрытые атрибуты и `<plugin namespace>:forge_hammer=byte(1)` — **legacy prototype**. Переименованный ванильный топор или предмет с тем же model ID не становится молотом. Старый boolean может читаться только в **миграционном адаптере**, не в production-сервисах.

Молот — рабочий инструмент, **не** новый тип боевого оружия. Его рецепт и итоговая прочность не утверждены. Конфиг должен позволять включать/выключать износ; в ForgeItems предложен стартовый вариант **`enabled=false`, `max=0`, `loss-per-forge-hit=1`, `loss-on-forge-activation=0`** (значение потери на удар имеет смысл только если износ включён). Активация станции по умолчанию не изнашивает молот; если удар изнашивает молот, только после реально принятого удара.

### 3.5. Представление и совместимость

Состояния именуются по схеме «Горячая заготовка меча», «Формируемый клинок», «Незакалённая головка кирки», «Закалённый клинок». Lore может показывать материал, температуру, этап, качество и кузнеца, но это **производное отображение**, не источник правды. Работа должна оставаться возможной без RP. Workpiece никогда не стакается ввиду `instance_id`, thermal anchor, stage, quality и session lock. Молот рекомендуется не стакать независимо от режима durability; необязательный guide может стакаться, если не содержит уникальных данных.

### 3.6. Версионирование и миграция

- Каждый специальный предмет содержит `item_schema`.
- Известные старые версии мигрируют **идемпотентно** без изменения Item Definition, создания второго ItemStack или смены Workpiece `instance_id`.
- Неизвестная будущая/повреждённая схема **не интерпретируется наугад**, предмет изымается из опасных производственных операций и остаётся в безопасном состоянии до диагностики.
- Legacy boolean молота после миграции заменяется `forgesystem:hammer` (schema 1), не становится запасным production-критерием.
- Stable model IDs выдаёт общий ServerPlatform registry; логические ключи `forge/workpiece/sword/hot`, `.../forging`, `.../unquenched`, `.../quenched`, `forge/hammer/default`, `forge/guide/default` и аналогичные для остальных изделий.

## 4. Конечный автомат и инварианты

### 4.1. Разрешённый граф

```text
                  reheat / thermal update
                         ↺
HOT_BLANK ────────────────────────────→ FORGING
   ↺                                       ↺
                                           │ successful finish, quality > NONE
                                           ▼
                                      UNQUENCHED
                                     /           \
                     броня, закалка /             \ инструменты, закалка
                                   ▼               ▼
                                FINISHED       QUENCHED_PART
                                                  │ + sticks / anvil
                                                  ▼
                                                FINISHED
                                                   ↺ material upgrade
```

Разрешено: `HOT_BLANK→FORGING`, `FORGING→FORGING` при персистентном прогрессе/reheat, `FORGING→UNQUENCHED`, `UNQUENCHED→FINISHED` **только для брони**, `UNQUENCHED→QUENCHED_PART` **только для пяти инструментов/оружия**, `QUENCHED_PART→FINISHED` после палок, `FINISHED→FINISHED` при допустимом tier-upgrade.

### 4.2. Заведомо запрещено

```text
HOT_BLANK → FINISHED
FORGING → QUENCHED_PART
UNQUENCHED → DIAMOND
QUENCHED_PART → DIAMOND
FINISHED → HOT_BLANK
ARMOR → QUENCHED_PART
QUENCHED_PART/FINISHED → REHEAT
```

`ForgeItems` проверяет допустимость state transition; `ForgeSystem` доказывает выполненные игровые условия. `quality_stage` отражает достигнутый уровень 0..3; `current_stage` — этап текущей ковки, `max_reachable_stage` — необратимый предел после остываний. Не смешивать эти поля.

## 5. Физический горн и активация

### 5.1. Многоблок и координаты

**Core = `BLAST_FURNACE`**. В референсном `multiblocks.yml` шаблон задан при `facing=NORTH`; при иных горизонтальных направлениях применяется поворот относительных смещений. Окончательный внешний вид может меняться через конфиг.

| Локальная позиция `(x,y,z)` при NORTH | Что допустимо |
|---|---|
| `(-1,0,0)` | `WALL_GROUP` |
| `(0,0,0)` | `BLAST_FURNACE` |
| `(+1,0,0)` | `WALL_GROUP` |
| `(-1,0,+1)` | `WALL_GROUP` |
| `(0,0,+1)` | `MAGMA_BLOCK` |
| `(+1,0,+1)` | `WALL_GROUP` |
| `(0,+1,+1)` | `CAMPFIRE` или `SOUL_CAMPFIRE` |
| `(0,+2,+1)` | `WALL_GROUP` |

`WALL_GROUP = {BRICKS, TUFF_BRICKS, POLISHED_BLACKSTONE_BRICKS, DEEPSLATE_BRICKS}`. Материалы в разных позициях могут отличаться. Тестовый многоблок **не объявлен окончательно утверждённой production-формой**.

Алгоритм валидатора: проверить core и его facing, для каждой обязательной локальной позиции повернуть offset и проверить принадлежность блока требуемой группе, не загружать unloaded chunks, кэшировать результат и инвалидировать по событиям изменения соседних блоков (в референсной реализации применяется область ±3 по каждой оси от известного core). Глобальный обход мира и форсированная загрузка чанков запрещены. Если форма повреждена, **все производственные операции отклоняются** даже при открытом GUI; входы безопасно возвращаются при закрытии.

### 5.2. Регистрация и доступ

В старом горне обычный `RIGHT_CLICK_BLOCK` по валидной доменной печи открывал GUI. В отдельной ForgeItems-spec принят новый целевой шаг **активации молотом**:

```text
PlayerInteractEvent (MAIN_HAND / RIGHT_CLICK_BLOCK)
→ ForgeItemsApi.isHammer(mainHand)
→ ForgeActivationGateway.tryActivate(player, clickedBlock, operationId)
→ ForgeStationService: find anchor → validate structure
→ register station atomically → ACTIVATED / ALREADY_ACTIVE / INVALID_STRUCTURE /
                               NOT_ANCHOR / BLOCKED / ERROR
```

`ACTIVATED` создаёт одну persistent-identity станции, `ALREADY_ACTIVE` сохраняет предыдущий station ID; невалидная форма ничего не расходует, повтор одного operation ID не создаёт второй горн. **Согласовать** в окончательной реализации поведение двух путей: когда/зачем регистрация молотом обязательна и когда можно просто открыть GUI ПКМ по core. Принцип «регистрация один раз, после неё доступ к GUI» является совместимой интеграционной трактовкой, а не уже проверенным runtime-поведением.

`ForgeItems` не валидирует структуру, топливо и температуру — эти функции остаются у ForgeStation. PlayerInteractEvent нельзя глобально отменять для каждого ПКМ молотом; только если действие перехвачено системой или необходимо предотвратить ванильный конфликт.

### 5.3. Станционное состояние

```text
station_id
world_uuid
core_x / core_y / core_z
temperature
fuel_seconds
last_update_epoch_ms
config_revision
```

В документе ранней реализации подтверждено лишь **runtime-хранение** температуры/запаса топлива/времени использования; полноценный SQLite repository для станции там не реализован. `ForgeStationRepository` и persistent activation требуют отдельного закрытия и теста. Существующая ранняя реализация чистит полностью остывшие неиспользуемые станции из runtime-кэша после **10 минут** (параметр реализации, не SLA постоянного хранения).

## 6. Тепловые модели и топливо

### 6.1. Станция (отдельный объект)

Референсный пресет `config.yml`:

```yaml
forge-station:
  ambient-temperature: 20.0
  working-temperature: 900.0
  maximum-temperature: 1200.0
  heating-per-second: 25.0
  cooling-per-second: 8.0
  manager-period-ticks: 20
  fuel:
    COAL: 80.0
    CHARCOAL: 80.0
    BLAZE_ROD: 120.0
    COAL_BLOCK: 800.0
```

> **Примечание к YAML:** обёртка `forge-station:` здесь — иллюстрация объединённой схемы; точный путь параметров и формат production-файла должны соответствовать валидатору текущей реализации. Значения взяты из `Forge_Mechanics` и `GUI_RP_TZ_v2`.

Тепловая симуляция по `elapsedSeconds`:

```text
если есть запас fuel_seconds:
  T_station = min(T_max, T_station + heatRate * elapsedSeconds)
иначе:
  T_station = max(T_ambient, T_station - coolRate * elapsedSeconds)
canWork = (T_station >= T_working)
```

Топливо продолжает расходоваться после достижения `T_max`, но T больше не растёт. `COLD`, `HEATING`, `WORKING`, `COOLING` описывают состояние, тогда как **`canWork` описывает допустимость действия**. Во время `COOLING` остаточное тепло может разрешать производство. Рекомендуемый единый `ForgeStationManager` симулирует активные станции каждые **20 тиков**, без task per station.

### 6.2. Пополнение топлива

GUI slot **10** — temporary fuel input, кнопка slot **19**:

```text
ЛКМ       → из входа в запас перевести 1 единицу топлива
Shift+ЛКМ → перевести весь подходящий стак
```

По материалу вычисляется сумма дополнительных секунд, затем *атомарно* обновляется запас станции и вычитается количество из input. Никакой частичный commit, если материал невалиден или состояние станции изменилось.

### 6.3. Металлы и thermal anchor

| Металл | Ingot | `initial-after-forge` | `minimum-working` | `cooling-per-second` | `maximum-safe` |
|---|---|---:|---:|---:|---:|
| IRON | `IRON_INGOT` | 1000.0 | 700.0 | 22.0 | 1200.0 |
| GOLD | `GOLD_INGOT` | 900.0 | 600.0 | 26.0 | 1100.0 |

Тестовые значения. Создание/reheat:

```text
T_new = min(T_initial_after_forge(material), T_station)
```

Вне активной сессии temperature считается **лениво**, при обращении к Workpiece:

```text
elapsedSeconds = (now_epoch_ms - updated_at_epoch_ms) / 1000
T_current = max(0, T_anchor - coolingRate(material) * elapsedSeconds)
```

Выражение `max(0, ...)` согласуется с поведением ForgeMinigame, но нижнюю физическую границу Workpiece и обработку отрицательного elapsed при сбое часов следует утвердить в коде отдельно: исходный Workpiece-описатель показывает линейную формулу без полноценной стратегии перепрыгивания системного времени. **Не сканировать все inventory на каждом тике.** Выход игрока и время вне сервера не останавливают охлаждение. Внутри runtime-session допускается монотонный `System.nanoTime`, но при любом выходе из session актуальное состояние фиксируется в wall-clock anchor.

### 6.4. Границы и разные пороги

Существуют **два разных рабочих порога**: станции (например, 900 °C) и металла в мини-игре (IRON 700 °C, GOLD 600 °C). Система не должна ошибочно требовать T_station≥700 для производства или считать Workpiece уже холодной при 900 °C. Станция нагревается по station rule, предмет охлаждается по material rule. Тепловой резерв на референсных железных значениях `(1000−700)/22 ≈ 13,6 с` — *пример из источника*, не таймер, выдаваемый в качестве гарантии из-за возможного более холодного старта.

## 7. Транзакция создания HOT_BLANK

### 7.1. Рецептуры

| Product | Ingots | Sticks later | Категория |
|---|---:|---:|---|
| `SWORD` | 2 | 1 | инструмент/оружие |
| `PICKAXE` | 3 | 2 | инструмент |
| `AXE` | 3 | 2 | инструмент |
| `SHOVEL` | 1 | 2 | инструмент |
| `HOE` | 2 | 2 | инструмент |
| `HELMET` | 5 | 0 | броня |
| `CHESTPLATE` | 8 | 0 | броня |
| `LEGGINGS` | 7 | 0 | броня |
| `BOOTS` | 4 | 0 | броня |

### 7.2. Сервисная операция

```text
on recipe click (productId, inputStack, stationId, playerId, operationId):
  validate stationId, multiblock, permission, and player menu ownership
  ensure T_station >= T_working
  ensure input material ∈ {IRON_INGOT, GOLD_INGOT}
  ensure products.yml allows chosen product and ingots >= materialCost
  validate result slot/inventory/drop-delivery path
  reject already-completed operationId and conflicting input mutation
  allocate unique instanceId and complete WorkpieceSnapshot
  prepare exact resulting ItemStack via ForgeItems API
  commit input decrement + Workpiece issuance once
  on failure, restore input/delivery consistently (no loss/dupe)
  after successful commit, publish ForgeBlankCreatedEvent
```

Точный механизм «commit/revert» (Paper inventory snapshots, journal, ownership/atomic region) в переданных документах не утверждён. Контракт результата важнее выбранного storage/transaction implementation.

### 7.3. Производственные запреты

- Только две начальные материальные базы: IRON/GOLD.
- Ни слиток, ни Workpiece не выдаются дважды при `DOUBLE_CLICK`, `SHIFT_CLICK`, повторном packet/event и `InventoryCloseEvent`.
- Slоt 14 **не** участвует в первоначальном создании `HOT_BLANK`; он предназначен только для повторного нагрева.
- Новая заготовка содержит уникальный `instance_id`, правильный config snapshot и thermals; output даётся безопасно.
- Клик по испорченной станции не расходует металл и не создаёт результат.

## 8. GUI горна и ресурспак

### 8.1. Нормативная геометрия Inventory

**Размер:** 54 Bukkit-слота, 6 строк по 9. Native-холст интерфейса — **176×222 px**. Для позиции `slot ∈ [0,53]`:

```text
row = floor(slot / 9)
column = slot % 9
x = 8 + column*18
y = 18 + row*18
slot inner area = 16×16 px; stride = 18 px
```

Инвентарь игрока использует строки с `y=140/158/176`, hotbar `y=198`, X — та же сетка с шагом 18 px. Эти координаты — основной контракт совмещения визуальных рамок и фактически кликабельных серверных слотов.

### 8.2. Актуальная карта слотов (GUI/RP ТЗ v2)

| Slot | Роль | Тип / действие |
|---:|---|---|
| **10** | Fuel Input | настоящий input: поддерживаемое топливо |
| **12** | Ingot Input | настоящий input: `IRON_INGOT`/`GOLD_INGOT` |
| **14** | Reheat Input («НАГРЕВ») | настоящий input: `HOT_BLANK` или unlocked `FORGING` |
| **16** | Temperature Fallback | недоступный для ввода status item/tooltip, когда RP нет |
| **19** | Load Fuel | ЛКМ 1 / Shift+ЛКМ stack |
| **20** | Fuel Reserve | индикатор, tooltip/число оставшегося времени |
| **21** | Station Status | индикатор состояния/ошибок |
| **23** | Reheat | кнопка повторного нагрева |
| **25** | Help | подсказка/гайд, информация |
| **28–32** | `SWORD`, `PICKAXE`, `AXE`, `SHOVEL`, `HOE` | 5 рецептурных кнопок |
| **38–41** | `HELMET`, `CHESTPLATE`, `LEGGINGS`, `BOOTS` | 4 рецептурные кнопки |
| **53** | Close | закрытие меню |

**Конфликт поколений:** `ForgeSystem_Forge_Mechanics` содержит раннюю карту брони **34–37**. `ServerMine_ForgeSystem_GUI_RP_TZ_v2`, более поздний и прямо объявленный приоритетным документ GUI, закрепляет **38–41**. Для нового интерфейса использовать только 38–41, legacy карту учитывать при миграции реализации. Внешний RP не должен отображать шестой фиктивный слот инструментов или декоративные рамки под несуществующие input. Служебные элементы могут быть встроенными кнопками/иконками без пустого квадрата.

### 8.3. Сохранность предметов GUI

При `InventoryCloseEvent` остатки из **10/12/14** возвращаются игроку в инвентарь; при отсутствии мест выдаются безопасным drop возле игрока. Внутренние status/buttons не извлекаются. Важен единый владелец временного input: двойной обработчик Close/Click не может вернуть те же ингредиенты дважды. Кнопка закрытия (`slot=53`) запускает тот же корректный lifecycle закрытия, а не удаляет Inventory contents. Повреждение многоблока во время открытого GUI блокирует все производственные операции, но не уничтожает входы.

### 8.4. Визуальный конвейер GUI

GUI/RP ТЗ v2 описывает разделение:

```text
assets/minecraft/font/default.json
assets/servermine/textures/gui/forge/forge_base.png
assets/servermine/textures/gui/forge/forge_labels.png
assets/servermine/textures/gui/forge/heat/forge_heat_00.png ... forge_heat_20.png
assets/servermine/textures/gui/forge/digits/0.png ... 9.png, degree.png, c.png
assets/servermine/font/forge.json
```

- `forge_base.png` — металл корпуса, фон, рамки **только используемых** ячеек, секции инструментов и брони, форма термометра, статус/close; не содержит изменяющихся данных.
- `forge_labels.png` — фиксированные пиксельные надписи «КУЗНЕЧНЫЙ ГОРН», «ТОПЛИВО», «МЕТАЛЛ», «НАГРЕВ», «ТЕМП.», «ИНСТРУМЕНТЫ», «БРОНЯ», «СОСТОЯНИЕ ГОРНА».
- Тепловой overlay — отдельно, прозрачный, в 21 кадре.
- Цифровые glyph — цифры 0…9, знак `°`, `C`, `-`. Без TTF, anti-aliasing, интерполяции и дробных пиксельных координат, чтобы сохранить стиль GUI 1.4.0.

**Нельзя** изменять игровой state посредством title/overlays. Клиентские элементы — отображение серверного snapshot.

### 8.5. 21-кадровая температурная шкала

```text
normalized = clamp((T_station - ambient) / (T_max - ambient), 0, 1)
heatFrame = round(normalized * 20)
workingNormalized = (T_working - ambient) / (T_max - ambient)
```

При 20/900/1200 °C рабочая отметка находится примерно на **74,6%** шкалы (фиксирована для данного конфига), даже когда заполнение меняется. Шкала вертикальная справа, заполняется **снизу вверх**; оранжево-жёлтые уровни внутри металлической рамки. Точное число вроде `824°C` выводится рядом. Gauge и надпись — **не игровые inventory slots**, не должны ловить клики.

### 8.6. Глифы и отрицательные интервалы

Рекомендуемый порядок композиции Inventory title:

```text
[negative spacer] [forge base glyph]
[negative spacer] [forge label glyph]
[negative spacer] [heat glyph] [temperature digit glyphs]
```

Примерное PUA-распределение из ТЗ GUI (не обязательные окончательные ID):

```text
E000       — negative spacer
E001       — forge base
E002       — reset/back spacer
E010       — static labels
E020..E034 — 21 heat frame
E040..E049 — digits 0..9
E04A       — °
E04B       — C
E04C       — -
```

Глиф-коды хранятся централизованно, не размазаны по меню/listeners.

### 8.7. Режимы отображения и fallback

RP tracker различает `ACCEPTED`, `SUCCESSFULLY_LOADED`, `DECLINED`, `FAILED_DOWNLOAD` и выбирает представление в `ForgeMenuRenderer`.

- **Pack loaded:** custom GUI, labels, 21-frame gauge, число °C, item models.
- **Pack absent/declined/failed:** всё тот же 54-slot Inventory с vanilla icons и понятными lore; `slot 16` показывает текущую/рабочую T и состояние, `slot 20` — запас топлива, `slot 21` — статус, девять рецептурных кнопок интерактивны.
- Режим RP не меняет рецепт, валидатор, стоимость, физику или антидюп.

### 8.8. Dirty refresh вместо полного rebuild

```java
record ForgeVisualSnapshot(
    int heatFrame,
    int temperatureCelsius,
    ForgeStationStatus status
) {}
```

- Общая симуляция станции: референсно **20 ticks**.
- Обновление визуального title: референсно **5 ticks**, но только если изменился `heatFrame`, округлённая T или status.
- Рецептурные иконки обновляются только при изменении типа/количества металла, `canWork` или `config_revision`.
- Не пересоздавать 54-слотовый Inventory каждый tick; не перезаписывать lore предметов непрерывно.

### 8.9. Порядок сообщений об ошибках

В панели «СОСТОЯНИЕ ГОРНА» предусмотрены: `ХОЛОДНЫЙ`, `НАГРЕВАЕТСЯ`, `ГОТОВ К РАБОТЕ`, `ОСТЫВАЕТ`, `НЕТ ТОПЛИВА`, `НЕДОСТАТОЧНО МЕТАЛЛА`, `НЕВЕРНАЯ ЗАГОТОВКА`.

Приоритет: **критическая ошибка валидации → ошибка запрошенной операции → общий статус станции**. Сообщение — следствие результата, а не причина допустить действие.

## 9. Ядро ForgeMinigame v2.4

### 9.1. Контракт автономного core

`ru.nyamine.forgeminigame.core.ForgeSession` описан как **чистая Java** без Paper/Bukkit. Базовое runtime-состояние:

```text
stage
quality
temperature
zoneStart
previousZoneCenter
stageStartedNanos
lastTemperatureNanos
pausedCold
finished
maxReachableStage
```

Каждая новая сессия содержит **ровно три `StageRules`**. Она вычисляет slider/zone, принимает `strike(now)` и возвращает `ForgeOutcome`, но не вычитает слитки и не распоряжается ItemStack/GUI. В production не должен использоваться ctor «всегда 1000 °C»; только seed от Workpiece.

### 9.2. Непрерывное время и положение

Внутренний диапазон **`0.0..1.0`**. Параметр `one-way-seconds` — время между противоположными концами пути, обратный путь занимает столько же. Схема ping-pong:

```text
0.0 → 1.0 → 0.0 → 1.0 → ...
```

Из исходного описания: положение определяется от `elapsedSeconds / oneWaySeconds`. Эквивалентная треугольная модель для неотрицательного elapsed:

```text
u = elapsedSeconds / oneWaySeconds
r = u % 2
position = (r <= 1) ? r : (2-r)
```

Эта **производная запись формулы** дана для объяснения, а не как проверенный текст конкретной реализации `sliderPosition()`; конечные поля и edge cases надо тестировать через core. Сервер проверяет **реальное** положение в момент прихода удара.

### 9.3. Генерация hitzone

```text
zoneWidth = StageRules.zoneWidth
zoneStart — случайное значение с учётом edgePadding
zoneEnd = zoneStart + zoneWidth
hit = zoneStart <= sliderPosition(now) <= zoneEnd
```

`edge-padding: 0.03` — 3% с обеих сторон от шкалы; valid `zoneWidth + 2*edgePadding <= 1`. `min-center-shift` пытается сместить центр новой зоны относительно предыдущей; описано **до 16 попыток** генерации. Требование смещения — желаемое, не гарантия бесконечного поиска при невозможном сочетании параметров.

### 9.4. Нормативный Normal difficulty

```yaml
difficulty:
  stages:
    stage-1:
      zone-width: 0.34
      one-way-seconds: 1.80
      min-center-shift: 0.15
      edge-padding: 0.03
    stage-2:
      zone-width: 0.22
      one-way-seconds: 1.30
      min-center-shift: 0.18
      edge-padding: 0.03
    stage-3:
      zone-width: 0.12
      one-way-seconds: 0.95
      min-center-shift: 0.20
      edge-padding: 0.03
```

Поля и значения прямо присутствуют в ForgeMinigame v2.4 и Full Mechanics; сами числа — **настраиваемый стартовый пресет**.

### 9.5. Stage outcomes

| Текущий этап | HIT | MISS | Что происходит при разрешённом next-stage |
|---|---|---|---|
| I | quality=GOOD | stage I reset, новая зона | переход II |
| II | quality=EXCELLENT | `FINISHED` с GOOD | переход III |
| III | `FINISHED` с MASTERWORK | `FINISHED` с EXCELLENT | финал всегда |

Stage I miss **не уничтожает Workpiece** и **не закрывает ковку**; сбрасывает timer и генерирует новую зону. Stage II miss завершает с качеством, ранее полученным на I. Stage III miss завершает с качеством, ранее полученным на II. Успех текущего этапа, после которого `nextStage > maxReachableStage`, **завершает ковку** на качестве текущего этапа.

### 9.6. Качество: миграционный mapping

В автономном v2.4:

```java
NONE(0, "Нет результата")
BASIC(1, "Хорошая ковка")
IMPROVED(2, "Отличная ковка")
MASTERWORK(3, "Мастерская ковка")
```

В ForgeSystem Full Mechanics / ForgeItems production snapshot:

```text
NONE, GOOD, EXCELLENT, MASTERWORK
```

**Явная маппинг-таблица:** `NONE → NONE`, `BASIC → GOOD`, `IMPROVED → EXCELLENT`, `MASTERWORK → MASTERWORK`. Значение `quality_stage` сохраняет число 0..3. Смешение двух enum-рядов без адаптера может нарушить PDC round-trip и отображение результатов; это обязательный пункт unit-тестов.

### 9.7. ForgeOutcome

Результат каждого удара в v2.4:

```text
type
stage
quality
hit
sliderPosition
zoneStart
zoneEnd
message
```

Допустимые типы: `STAGE_SUCCESS`, `STAGE_MISS`, `SESSION_FINISHED`, `PAUSED_COLD`, `IGNORED`. Paper-layer выбирает звук, popup и нужную запись Workpiece **после** доменного outcome; не наоборот.

### 9.8. Типовые difficulty profiles (ориентиры, не балансный релиз)

| Профиль | I зона / one-way | II зона / one-way | III зона / one-way | Iron cooling example |
|---|---|---|---|---|
| Easy | 42% / 2.20s | 30% / 1.70s | 18% / 1.25s | ≈ 16 °C/с |
| Normal | 34% / 1.80s | 22% / 1.30s | 12% / 0.95s | ≈ 22 °C/с |
| Hard | 28% / 1.45s | 17% / 1.00s | 8% / 0.72s | ≈ 30 °C/с |

Для визуального соответствия готовому HUD рекомендуется ограничить рабочие `zone-width` диапазоном **0.04..0.60**. При большем/меньшем числе серверный hitbox может оставаться математически корректным, но RP-картинка будет лишь ближайшим допустимым glyph — риск нечестной визуализации.

### 9.9. Валидация difficulty

**StageRules:**

```text
0 < zoneWidth < 1
oneWaySeconds > 0
0 <= minCenterShift <= 1
0 <= edgePadding < 0.5
zoneWidth + 2*edgePadding <= 1
```

В исходном core некорректные значения приводят к `IllegalArgumentException`. Thermal rules: `initialTemperature > minimumWorkingTemperature`, `coolingPerSecond > 0`. Системные значения должны проверяться при startup/reload **до публикации конфигурации**.

## 10. HUD мини-игры и feedback

### 10.1. Протокол визуального слоя

Источник: `ActionBar` + bitmap font `forgesystem:hud`, из `ForgeMinigame-HUD-v2.4-ZoneVariants.zip`.

Слои HUD в логическом порядке:

```text
BASE → STAGE → ZONE → SLIDER → TEMPERATURE → QUALITY → STATUS → POPUP
```

Negative-space glyph из прототипа: **U+E7F0, advance = −193**. Положение наложений вычисляется renderer, но HUD не влияет на решение `hit`. Элементы: надпись «КОВКА», Stage I/II/III, шкала, зелёная зона, белый ползунок, температурная шкала, три ромба качества, правый статус и popup над полосой.

### 10.2. Quality glyphs, status, popups

```text
◇◇◇ NONE | ◆◇◇ GOOD | ◆◆◇ EXCELLENT | ◆◆◆ MASTERWORK
```

| `ForgeHudStatus` | Визуальная иконка |
|---|---|
| NONE | без специального результата |
| HIT | зелёная галка |
| MISS | красный крест |
| COLD | синяя снежинка |
| LOCKED | янтарный замок |
| DONE | золотая звезда |

Popups (`ForgeHudPopup`): `GOOD` — «ХОРОШАЯ КОВКА», `EXCELLENT` — «ОТЛИЧНАЯ КОВКА», `MASTERWORK` — «МАСТЕРСКАЯ КОВКА», `MISS` — «ПРОМАХ», `COLD` — «МЕТАЛЛ ОСТЫЛ», `LIMITED` — «ЛИМИТ КАЧЕСТВА». Короткая обратная связь не спамит чат; ограничение качества показывается как статус `LOCKED` / popup `LIMITED`, не означает запрет текущего допустимого stage.

### 10.3. Атлас зон и округление

Ресурспак v2.4 содержит:

- **31 положение** ползунка: индексы `0…30`.
- **31 положение** центра зоны.
- **29 ширин** визуальной hitzone: **4, 6, 8, ... 60%**, шаг 2 процентных пункта.
- Общее число zone glyphs: **29×31=899**.
- **11 температурных индексов** по диапазону от `minimumWorkingTemperature` до `initialTemperature`: `0…10`.

Визуальный вариант зоны выбирается по nearest-width (пример: серверная `zone-width=0.173` → RP-ширина 18%). **Сервер не округляет hitbox**; `ZONE` glyph может быть лишь визуальной аппроксимацией. Поэтому диапазон production-width ограничивается рекомендацией 4…60%, а тесты должны проверять не только точный результат, но и приемлемость изображения.

### 10.4. Обновления HUD и popup

Пресеты v2.4:

```yaml
hud:
  resource-pack: true
  update-ticks: 2
  fallback-segments: 31
  quality-popup-ticks: 22
  status-popup-ticks: 14
```

HUD обновляется раз в **2 тика** (порядка 10 раз/с при 20 TPS); clamp для update interval — **1…20 тиков**. Текстовый fallback имеет clamp **15…61 сегмент**, старт **31**. Popup качества — **22 тика (~1.1 сек)**, статуса — **14 тиков (~0.7 сек)**. После `SESSION_FINISHED` финальная плашка остаётся на `max(qualityPopupTicks, statusPopupTicks)+2` тика (**24 тика при defaults**), затем ActionBar очищается однократно.

При охлаждении порядок обратной связи: статус `COLD`, popup «МЕТАЛЛ ОСТЫЛ», звук тушения, короткая выдержка, очистка HUD. В автономном v2.4 runtime-session остаётся в памяти после скрытия HUD, пока не вызвана `/fg reheat`; **в production после сохранения Workpiece runtime-session нужно корректно закрыть**, чтобы не оставлять бесполезный lock.

### 10.5. Звуковая карта v2.4 (прототипные значения)

| Событие | Vanilla sound | Volume | Pitch |
|---|---|---:|---:|
| Старт | `BLOCK_ANVIL_LAND` | 0.35 | 1.35 |
| Успех промежуточного этапа | `BLOCK_ANVIL_LAND` | 0.8 | 1.25 |
| Промах | `BLOCK_ANVIL_LAND` | 0.55 | 0.65 |
| Остывание | `BLOCK_FIRE_EXTINGUISH` | 0.65 | 0.8 |
| Финал успешный | `BLOCK_ANVIL_LAND` | 0.9 | 1.5 |
| Финал после неудачи | `BLOCK_ANVIL_LAND` | 0.9 | 0.7 |

### 10.6. RP pack assets / совместимость

Мини-игра использует:

```text
ForgeMinigame-HUD-v2.4-ZoneVariants.zip
├── pack.mcmeta
├── pack.png
└── assets/forgesystem/
    ├── font/hud.json
    └── textures/font/
        ├── hud_atlas.png
        └── zone_variants.png
```

Установленные в исходном описании bitmap font параметры: `height=48`, `ascent=40`. В исходном пакете заявлены `min_format=[88,0]` и `max_format=[88,0]`; правильность pack format для конкретной целевой клиентской версии следует перепроверить отдельно при runtime-smoke, а не считать обеспеченной самим числом. HUD и GUI могут использовать разные namespace (`forgesystem:hud` для мини-игры, `servermine` для GUI) и должны быть объединены **без glyph collisions**.

Текстовый fallback ActionBar обязан работать без RP; пример: `Этап II | ─────━━◆━━───── | 842°C`. При отсутствии ресурспака нельзя терять возможность попадать в зону, видеть этап, температуру, качество и холод.

## 11. Интеграция Workpiece и ForgeSession

### 11.1. Нормативный вход по наковальне

```text
PlayerInteractEvent:
  action = LEFT_CLICK_BLOCK
  hand = MAIN_HAND
  clickedBlock ∈ {ANVIL, CHIPPED_ANVIL, DAMAGED_ANVIL}
  mainHand = valid forgesystem:hammer
  offHand = valid forgesystem:workpiece
  forge_state ∈ {HOT_BLANK, FORGING (без активного session_id)}
  T_workpiece > minimumWorking(material)
  no active player session
  no other session by instance_id
```

Если хотя бы одно условие не выполнено, нормальное использование наковальни не должно быть перехвачено ошибочно. При принятом ударе Paper-event отменяется для устранения vanilla-конфликта. Первое принятое событие — **только start/resume**, не scored strike. Второй и последующие ЛКМ отправляются в ForgeSession.

### 11.2. Seed новой сессии

Рекомендуемый DTO из документа интеграции:

```java
record ForgeSessionSeed(
    int currentStage,
    ForgeQuality quality,
    int maxReachableStage,
    double currentTemperature,
    double minimumWorkingTemperature,
    double coolingPerSecond,
    StageRules[] rules
) {}
```

Production Core получает seed из **актуального Workpiece**, а не command-default `initial=1000`. Источник правил — immutable config snapshot. Вместе с началом создаётся уникальный `session_id`, устанавливается lock, `blacksmith_uuid` при первом старте получает UUID кузнеца. На повторном запуске `blacksmith_uuid` **сохраняется**; визуальную позицию зоны допустимо сгенерировать заново.

### 11.3. Переходы при начале

| Стартовый `forge_state` | Действия |
|---|---|
| `HOT_BLANK` | `HOT_BLANK→FORGING`; `quality=NONE`, `current_stage=1`, `max_reachable_stage=3`, `blacksmith_uuid` от первого пользователя, новый lock `session_id` |
| `FORGING`, unlocked | Сохранить quality/stage/max и blacksmith; создать новую runtime-session из snapshot; новый `session_id` |
| `FORGING`, active lock | Отказ; нельзя создать вторую сессию |
| `UNQUENCHED`/`QUENCHED_PART`/`FINISHED` | Отказ в мини-игре (нужны иные адаптеры) |

### 11.4. Run-loop и результаты

Текущая мини-игра использует **один scheduler с периодом 1 tick**, проходит только по активным `ForgeSession`, вызывает `session.tick(now)` для охлаждения и читает server-authoritative outcome ударов. HUD публикация — отдельный порог `hud.update-ticks`. Ни один клиентский пакет с уже готовым `HIT` не является достоверным подтверждением. Stage changes/quality фиксируются в persistent Workpiece перед критическим окончанием/снятием lock; промежуточное сохранение должно быть достаточно надёжным, чтобы `logout/disable/crash` не улучшали результат.

### 11.5. Завершение

```text
ForgeSession returns SESSION_FINISHED
→ validate unique operation id and locked instance
→ update snapshot: FORGING → UNQUENCHED
→ persist quality_stage=1|2|3 and quality
→ preserve instance_id, material, product_type, blacksmith_uuid
→ session_id = null
→ commit exactly once / publish ForgeCompletedEvent
→ show final feedback and later clear HUD
```

**Только** `UNQUENCHED` создаётся на этом шаге, не vanilla weapon/armor. Повторный packet, завершение второй раз или потерянный callback не создают второй ItemStack. При одном и том же logical finish возвращается уже известный результат, а не новый.

## 12. Остывание и повторный нагрев

### 12.1. Порог и эффект ограничения

В v2.4 cold-trigger при `T <= minimumWorking`:

```text
T = minimumWorking
pausedCold = true
maxReachableStage = min(maxReachableStage, currentStage)
```

В автономном prototype значение фиксируется на минимальном пороге. **Для постоянной модели Workpiece** требуется сохранить достигнутый предел, stage, quality и thermal anchor и позволить реальной температуре продолжать остывать вне сессии. Следовательно, одноразовая фиксация порога как runtime marker и дальнейшая lazy-temperature вне сессии — **разные обязанности**; production не должен замораживать физическую температуру предмета на пороге навсегда.

Правило качества:

```text
cold on I  ⇒ max_reachable_stage ≤ 1  ⇒ до GOOD
cold on II ⇒ max_reachable_stage ≤ 2  ⇒ до EXCELLENT
cold on III ⇒ max_reachable_stage ≤ 3, MASTERWORK остаётся достижимым
                       после reheat только если не было более раннего лимита
```

Последняя строка следует из формулы `min(max,N)`: охлаждение на III не уменьшает предел ниже III. В документах дан особый текст «чтобы получить MASTERWORK, нужно пройти все три этапа до остывания»; **это потенциальная неоднозначность**. При буквальной формуле cold на III и последующем reheat не запрещает MASTERWORK. До реализации и плейтеста следует явно решить, должна ли III стадия быть исключением или допускается reheat на III. См. §21; в техническом контракте сохранена именно формула, потому что она указана первичным кодовым описанием.

### 12.2. Завершение runtime при COLD

Нормативный поток интеграции:

```text
ForgeSession detects first cold
→ server outcome PAUSED_COLD / visual notification
→ max_reachable_stage=min(max_reachable_stage,current_stage)
→ persist temperature anchor, current_stage, quality, quality_stage,
          max_reachable_stage, blacksmith_uuid
→ clear session_id
→ release player and instance locks
→ stop runtime-session and hide HUD after popup
→ state stays FORGING
```

**Нельзя** продолжать держать занятое `session_id` у охлаждённого предмета бесконечно (в отличие от демонстрационного `/fg reheat`). Состояние `FORGING` остаётся, потому что игрок ещё не закончил изделие. Если деталь уже остыла до начала первой ForgeSession, источники не определяют строго момент применения cap к этапу I; запрещаемый start не должен автоматически выдавать качество или открывать будущие стадии — вопрос отдельного edge-case test/decision.

### 12.3. Reheat через настоящий горн

```text
station valid + T_station>=station.workingTemperature
slot 14 contains valid HOT_BLANK or FORGING with session_id=null
player clicks slot 23
validate operationId and instance lock
T_workpiece = min(material.initial-after-forge, T_station)
updated_at_epoch_ms = now
all non-thermal identity/progress fields unchanged
return same physical instance, exactly once
```

Недопустимо `FORGING` с активной сессией (в том числе offhand не должен перемещаться в GUI под session lock), `UNQUENCHED`, `QUENCHED_PART`, `FINISHED`, некорректный/поддельный предмет.

### 12.4. Прототипное `reheatedCopy()`

В v2.4 `/fg reheat` создаёт новую runtime-копию `session.reheatedCopy(now)`: сохраняет **stage, quality, maxReachableStage**, сбрасывает timer, paused/finished, центр зоны и position, восстанавливает тестовое initial T. После merge такое поведение остаётся только **алгоритмической подсказкой для создания нового ForgeSession из повторно нагретого Workpiece**; сама команда не заменяет реальный горн, а runtime-copy не считается persistent state.

## 13. Переходы после ковки

### 13.1. Закалка у котла

Нормативно:

```text
player holds valid UNQUENCHED Workpiece
→ RIGHT_CLICK_BLOCK on WATER_CAULDRON with level > 0
→ validate protected item, ownership, operation uniqueness
→ consume exactly 1 cauldron water level
→ produce result exactly once
→ sound of quench / hissing / steam feedback
```

Пустой котёл: **нет изменения Workpiece и воды**. Обычный котёл, не подходящий по состоянию, не запускает чужой/ванильный конфликт. Из исходных материалов не следует точная API-сигнатура/какой hand требуется для quench, кроме «держит UNQUENCHED и делает ПКМ», поэтому не надо придумывать новый обязательный mainhand-only contract без теста.

### 13.2. Закалка брони vs деталей

```text
ARMOR: UNQUENCHED → FINISHED → IRON_* / GOLDEN_* armor
TOOL/WEAPON: UNQUENCHED → QUENCHED_PART (технический клинок/головка)
```

Workpiece-брони до `FINISHED` должен иметь безопасный carrier (например `PAPER+PDC+model`), **не** ванильную носимую броню. `QUENCHED_PART` не допускается для armor. Полученная броня наследует `quality`, `blacksmith_uuid` и положенные Forge metadata.

### 13.3. Рукояти

Для пяти продуктов `SWORD` (1 палка), `PICKAXE` (2), `AXE` (2), `SHOVEL` (2), `HOE` (2):

```text
valid QUENCHED_PART + sufficient vanilla STICK in inventory
+ deliberate RIGHT_CLICK_BLOCK on anvil
→ validate + deduct exact stick cost atomically
→ QUENCHED_PART → FINISHED
→ materialize IRON_* / GOLDEN_* equipment ItemStack
→ keep quality/blacksmith metadata
```

Без сознательного действия на наковальне **нельзя** автоматически собирать предмет при появлении палки в инвентаре. Ванильная наковальня при обычном использовании остаётся функциональной.

### 13.4. Diamond / Netherite upgrade

Разрешено только:

```text
FINISHED IRON + diamonds (products.yml cost) → FINISHED DIAMOND
FINISHED DIAMOND + standard netherite-upgrade components → FINISHED NETHERITE
```

Место действия: `SMITHING_TABLE`. Другие входы (`UNQUENCHED`, `QUENCHED_PART`) отклоняются. Обычные рецепты алмазной экипировки должны быть закрыты, иначе iron→diamond chain обходится. Для netherite сохраняется близкое к vanilla поведение и предусмотренные vanilla-компоненты. В документе приведён **частный пример** «IRON_SWORD GOOD + 2 diamonds → DIAMOND_SWORD GOOD», но полной таблицы diamond upgrade costs нет.

На каждом upgrade:

- сохранить `instance_id`/историю и авторство по согласованной finished-item схеме;
- перенести `quality` и служебную Forge metadata;
- если бонус зависит от материала, пересчитать tier-based величины вместо слепого переноса;
- обеспечить ровно один результат даже при `PrepareSmithingEvent`, кликах и внешних плагинах.

### 13.5. Бонусы качества (пока интерфейс для баланса)

Заложить поддержку бонусов долговечности, меньшего износа, затрат ремонта, небольших особых характеристик, подписи кузнеца. **Числа и формулы не определены источниками**. Нельзя вводить сверхсильный Masterwork-iron, который автоматически превосходит хороший netherite tier, без отдельного согласования. Первое назначение `blacksmith_uuid` — при начале ForgeSession; торговля и upgrades его не меняют.


## 14. API и событийные контракты

### 14.1. Граница ForgeItems API

Контракт ниже воспроизводит назначение операций из `ForgeItems_Module_Documentation`; имена и формы DTO должны быть синхронизированы с фактическим кодом при интеграции (это **спецификация**, а не подтверждённый опубликованный бинарный API).

```java
public interface ForgeItemsApi {
    boolean isWorkpiece(ItemStack item);
    Optional<WorkpieceSnapshot> readWorkpiece(ItemStack item);
    ItemStack createWorkpiece(CreateWorkpieceRequest request);
    TransitionResult transition(ItemStack item, WorkpieceTransition transition);
    ItemStack updateThermalAnchor(ItemStack item,
                                  double temperature,
                                  long updatedAtEpochMs);
    ItemStack setSessionLock(ItemStack item, UUID sessionId);
    ItemStack clearSessionLock(ItemStack item);
    boolean isHammer(ItemStack item);
    ItemStack createHammer();
    ForgeItemInspection inspect(ItemStack item);
}

public interface WorkpieceCodec {
    Optional<WorkpieceSnapshot> read(ItemStack item);
    ItemStack write(ItemStack original, WorkpieceSnapshot snapshot);
    boolean isWorkpiece(ItemStack item);
}
```

Требования к `ForgeItemsApi`:

- `readWorkpiece` — read-only: ни автоматического повышения schema, ни ремонта повреждённого ItemStack без отдельной управляемой операции.
- `transition` проверяет переход конечного автомата и возвращает типизированную ошибку без частичного изменения предмета.
- `updateThermalAnchor` обновляет только разрешённые тепловые поля; не меняет quality, достигнутые стадии, историю.
- `setSessionLock`/`clearSessionLock` сохраняют lock metadata в самом объекте, но серверный runtime-регистр всё равно обязателен для эксклюзивности.
- Внешний код не пишет PDC напрямую: записи идут через codec/Item API, иначе нет верификации schema и инвариантов.
- `inspect` — безопасный способ администрирования и диагностики, при необходимости редактирует только «видимое представление» inspection DTO, не предмет.

### 14.2. Активация физического горна

```java
public interface ForgeActivationGateway {
    ForgeActivationResult tryActivate(
        Player player,
        Block clickedBlock,
        UUID operationId
    );
}
```

Результаты из контракта: `ACTIVATED`, `ALREADY_ACTIVE`, `INVALID_STRUCTURE`, `NOT_ANCHOR`, `BLOCKED`, `ERROR`. Проверку предмета-молота обеспечивает ForgeItems/ForgeSystem совместно, проверка схемы многоблока принадлежит ForgeSystem. Повтор того же `operationId` не создаёт второй ForgeStation и не расходует молот повторно. Клик по произвольному блоку не должен активировать станцию.

### 14.3. ForgeSessionManager и ForgeStationService

Контракты из интеграционного проекта, нормализованные по назначению:

```java
public interface ForgeSessionManager {
    StartResult startOrResume(
        PlayerId player,
        WorkpieceSnapshot workpiece,
        ForgeContext context
    );
    StrikeResult strike(PlayerId player, Instant eventMoment);
    Optional<ForgeSessionView> activeSession(PlayerId player);
    void pauseCold(PlayerId player);
    void abortAndPersist(PlayerId player, AbortReason reason);
}

public interface ForgeStationService {
    CreateWorkpieceResult createWorkpiece(
        PlayerId player,
        ForgeStationId station,
        ForgeMaterial material,
        ForgeProductType product
    );
    ReheatResult reheatWorkpiece(
        PlayerId player,
        ForgeStationId station,
        WorkpieceSnapshot workpiece
    );
}
```

**Важное инженерное уточнение:** формулировка источника `Instant monotonicNow` не делает `Instant` монотонными часами. Внутри мини-игры движок использует монотонный elapsed clock/ticks; для сохранения item `updated_at_epoch_ms` используется wall clock. В реальном Java API лучше передавать монотонную длительность или абстракцию clock, а не выдавать календарный `Instant` за `nanoTime`.

`startOrResume` обязан сделать три проверки: physical Workpiece с PDC (не lore), отсутствует чужая active session для instance, положение/температура позволяют начало или продолжение. Он не может повторно начислить попадание с первого ЛКМ. `abortAndPersist` MUST выполнить unlock в try/finally и сохранить прогресс, если это возможно без создания дополнительных копий.

### 14.4. События домена (предполагаемый production contract)

В GUI/RP ТЗ именуются события:

| Событие | Условие публикации | Основной payload (целевой) |
|---|---|---|
| `ForgeBlankCreatedEvent` | успешная транзакция слитки→`HOT_BLANK` | `station_id`, `instance_id`, `product`, `material`, player |
| `ForgeWorkpieceReheatedEvent` | успешный повторный нагрев настоящей заготовки | `instance_id`, `old/new temperature`, station |
| `ForgeStageCompletedEvent` | сервер авторитетно закрыл стадию I/II/III | `instance_id`, stage, strike outcome, stage result |
| `ForgeCompletedEvent` | успешная Stage III и выпуск `UNQUENCHED` | `instance_id`, quality, blacksmith |
| `ForgeItemQuenchedEvent` | закончен переход у водяного котла | old/new state, cauldron, player |
| `ForgeItemFinishedEvent` | броня готова после quench либо part собрана с рукоятью | final equipment, quality |
| `ForgeItemUpgradedEvent` | smithing upgrade завершён | previous/new material tier, quality |

События — **проектные точки расширения**; таблица payload — предложенная разработческая форма, поскольку источники не задают исчерпывающие конструкторы. Публиковать post-commit, чтобы слушатели не увидели phantom result. Для предусловий лучше отдельные cancellable before-event либо прямой транзакционный validator, а не отмена после списания ресурсов.

### 14.5. Paper listeners и запреты на обработку

Минимальное распределение входов:

| Minecraft/Paper событие | Доменный обработчик | Ограничение |
|---|---|---|
| `PlayerInteractEvent` | activation / GUI / anvil striking / quench / handle assembly | проверять action, hand, block, hotbar, priority/cancellation; не запускать двойное событие от offhand |
| `InventoryClickEvent`, `InventoryDragEvent` | ForgeMenuListener, lock guard | только выделенные входы, защитить render-slots, shift-click и number-key swap |
| `InventoryCloseEvent` | ForgeMenuService | закрытие окна не равно abort Workpiece и не теряет входные itemstack |
| `PlayerDropItemEvent` | ItemProtection + session guard | не выпускать `FORGING` и locked Workpiece |
| `PrepareItemCraftEvent`, `CraftItemEvent` | recipe guards | не превращать Workpiece в ванильный предмет по совпадению carrier material |
| smithing prepare/result events | SmithingAdapter | preview без списания, commit с проверкой; не дублировать vanilla upgrade |
| grindstone, furnace, smoker, blast-furnace, stonecutter | ItemProtection | запрет обходных рецептов, уничтожения carrier, несанкционированной переработки |
| death/quit/disable, предметы других плагинов | recovery/lock guard | очистить runtime, без «выдачи из воздуха» при reconnect |

Любая операция, меняющая `ItemStack`, завершается только на основном серверном потоке, если конкретный API явно не разрешает иное. Синхронность Bukkit/Paper calls — инженерное требование, не доказательство наличия такого кода в dev JAR.

## 15. Безопасность, транзакции и recovery

### 15.1. Уникальность и две области блокировки

Идентичность — `instance_id` UUID внутри валидного Workpiece. Активный сеанс — отдельный `session_id` UUID. Нужны независимые индексы:

```text
player_uuid -> session_id
workpiece_instance_id -> session_id
session_id -> {player, instance, authoritative slot, startedAt, state}
```

Уникальность должен проверять реестр, а не только bool `locked` из PDC. Иначе две копии одного предмета с одинаковым `instance_id` и две одновременно инициированные операции могут обойти локальную проверку. Если встретились два разных ItemStack с одним `instance_id`, до расследования **оба не должны завершить цикл независимо**. Никакое имя/lore/CustomModelData без валидного schema namespace и полей не делает предмет настоящим.

### 15.2. Атомарность ресурсоёмких операций

Рекомендуемый порядок commit:

1. Захватить логическую блокировку операции (`operationId`, `instanceId`, station).
2. Повторно подтвердить все предусловия: предмет физически находится в ожидаемом слоте; схема/станция валидна; ресурсов достаточно; output slot доступен; форма рецепта совпадает.
3. Рассчитать результирующий `ItemStack` и delta ресурсов **до** мутации.
4. Commit на main thread как единую серверную транзакцию. При исключении — отмена/компенсация и фиксирование ошибки.
5. Обновить station cache, GUI, PDC snapshot, журнал операции и только затем отправить post-commit domain event.
6. Снять временную блокировку либо продлить lease у реально активного ForgeSession.

Критические операции: `createWorkpiece`, `reheatWorkpiece`, переход stage, `UNQUENCHED` creation, quench с понижением уровня котла, сборка рукояти со списанием палок, smithing с расходом материалов. Для `Prepare...` событий запрещено делать commit, они показывают только preview. `operationId` нужен также для повторов после exception/retry.

### 15.3. Защита carrier, equipment и интерфейсов

- `HOT_BLANK`, `FORGING`, `UNQUENCHED`, `QUENCHED_PART` — технические предметы с явным запретом vanilla использования, крафта, ремонта, переплавки, экипировки, шлифовки и smithing вне контролируемых переходов.
- Workpiece, находящийся в активной сессии, нельзя переложить через shift-click, swap hotbar, drag, drop, offhand swap, creative middle-click clone, контейнерные хопперы и сторонние переносы (адаптеры для внешних inventory/shops требуют отдельной интеграции).
- Для брони Workpiece carrier не должен вести себя как готовая броня. При `FINISHED` item преобразуется в ванильную броню или инструмент с metadata, чтобы он совместимо носился/использовался без подмены vanilla attribute behavior.
- Стоит рассматривать кастомные предметы без metadata / с неверной версией как `INVALID_FORGE_ITEM`, а не «обычный металл», если их модель/маркер указывает на ForgeSystem.

### 15.4. Выход игрока, смерть, chunk unload, plugin disable

| Сценарий | Сохранение и дальнейшее действие |
|---|---|
| Player quit во время Stage I/II/III | сериализовать Workpiece progress, остановить тикер, снять session lock или оставить recoverable marker на короткий transactional период |
| Смерть игрока | не переносить копию Workpiece дважды в drops и inventory; одна каноническая копия сохраняет последние закоммиченные данные |
| Уничтожение/выгрузка горна | прервать операции станции и сохранить переносимые Workpieces; больше не греть их как работающий горн |
| Server restart или `/stop` | flush станции и item transitions, снять ticks/sessions, восстановление происходит из PDC и station repo |
| Неожиданный crash | загрузить last committed snapshot, сравнить operation ledger с фактически существующими itemstack, не выдавать новый предмет автоматически |
| Item с заблокированным `session_id`, но без живой сессии | проверяемый orphan lock recovery, логирование, снятие lock только после восстановления прогресса и отсутствия конкурирующего owner |
| Невалидная schema, отсутствующее обязательное поле | карантин/inspect; не выполнять переходы, не удалять доказательство ошибки |

Recovery не может базироваться только на runtime `Map`: он исчезает при restart. Для физического горна нужны persistent `ForgeStationRepository` и обратимое восстановление ItemSnapshot из PDC; **конкретный формат БД/файлов для station repo источники не фиксируют**. SQLite, YAML или другие backend — варианты архитектурного выбора, а не установленный стандарт.

### 15.5. Минимальные записи журнала

Для расследования дюпов: timestamp, UUID игрока, `instance_id`, `operation_id`, тип перехода, state before/after, reason failure, station key, целочисленное списание ресурсов и итог commit/rollback. Достаточна ограниченная ротация; никогда не печатать содержимое личного инвентаря каждого игрока при обычном log level. Журнал не заменяет проверку PDC.

## 16. Конфигурация и reload

### 16.1. Предлагаемое разбиение

- `config.yml`: флаги модулей, лимиты active sessions, поведение logout/death, глобальная сложность, storage и diagnostics.
- `materials.yml`: IRON/GOLD, ambient/working/critical/maximum temperature, heat/cooling coefficients, fuel values.
- `products.yml`: 5 инструментов/оружия + 4 класса брони, расход слитков, палок, upgrade materials.
- `multiblocks.yml`: структура `BLAST_FURNACE`-горна и правила facing/anchor/cache.
- `gui.yml`: 54-slot mapping, item templates, availability, scale, fallback.
- `resourcepack.yml`: RP namespaces, fonts, frame count, icon groups, model registry integration, pack state behavior.
- `messages.yml`: локализация/feedback, errors, tooltips, actionbar messages.
- `ForgeMinigame` block в config или выделенный `minigame.yml`: StageRules, zone params, cooldown, HUD timings.
- `ForgeItems` schema/settings: namespace, migrations, lock/codec/registry configuration; не дублировать master ownership.

Это **рекомендуемый способ разложить ранее описанные параметры**. Источники прямо используют некоторые названия (`gui.yml`, `resourcepack.yml`), но не устанавливают все вышеперечисленные файлы как существующие.

### 16.2. Фрагмент строго иллюстративного конфига

```yaml
forge:
  enabled: true
  gui:
    size: 54
    slots:
      fuel: 10
      metal: 12
      heat: 14
      temperature-fallback: 16
      armor: [38, 39, 40, 41]
    scale: auto            # auto | 1 | 2 | 3 | 4
  gameplay:
    source-materials: [IRON, GOLD]
    require-real-station: true
    first-anvil-click-is-only-session-start: true
    quench-water-level-cost: 1
  sessions:
    max-active-per-player: 1
    max-active-per-instance: 1
```

Представленный пример **не копия реального `config.yml`** и не должен напрямую загружаться в старый код, где keys другой структуры. Конкретные значения температур/длительностей, difficulty presets и тепловые формулы подробно перечислены в §6/§9; их надо переносить из источника, не подменяя условными примерами.

### 16.3. Config validator

На запуске и reload проверять минимум:

| Проверка | При ошибке |
|---|---|
| `gui.size == 54`, слоты `0..53` | запретить применение reload с понятным сообщением |
| Не пересекаются input, action, armor slots и декоративные immutable slots | сообщить конфликт и адрес offending keys |
| Зона `width>0`, `0<=edge-padding`, `min-center-shift` помещается в 60-char bar | reject unsafe minigame rules |
| `one-way-seconds>0`, все stage time positive | reject/divide-by-zero prevention |
| Температура/рабочий порог/максимум упорядочены, нагрев и остывание неотрицательны | reject или безопасное disabled station состояние |
| Resource pack temperature frames >= 2; glyph/namespace определены | fallback без повреждения Workpiece |
| Типы `IRON`, `GOLD`, продукты и число рецептов соответствуют API enum | ошибка конфигурации, не silent missing product |
| Расход слитков, палок и upgrade materials — целое допустимое значение | reject negative/zero там, где рецепт требует расход |
| Указанный Bukkit Material существует в целевой версии | отказ от невалидного рецепта, диагностическое сообщение |
| ForgeItems schema/namespace и registry не меняются без миграции | reject incompatible config reload |

### 16.4. Безопасный reload

1. Прочитать конфигурацию во временный immutable snapshot.
2. Провалидировать **все** файлы и совместимость версий.
3. Проверить, какие изменения безопасны для активных Workpiece и ForgeSession.
4. На main thread атомарно переключить новый snapshot для **новых** операций; существующие сеансы заканчивают с сохранённым rules snapshot (или безопасно приостанавливаются по явно объявленному правилу).
5. Обновить GUI/HUD для новых открытий и RP-кеши там, где это не меняет логику.
6. Если validation failed — оставить предыдущие настройки и сообщить оператору, не перезапускать серверный `/reload` без необходимости.

Изменение schema PDC/стратегии качества нельзя выдавать за обычный hot reload. Для миграции требуется отдельный план и backup.

## 17. Команды, permissions, сборка

### 17.1. Production команды (предлагаемый публичный контракт)

| Команда | Назначение | Охрана |
|---|---|---|
| `/forge help` | краткая помощь игроку и администратору | `forgesystem.use` |
| `/forge inspect [player]` | показать параметры Forge-предмета в руке (state, stage, instance, T, lock, schema) | для другого игрока admin permission |
| `/forge reload` | атомарная перечитка config snapshot | `forgesystem.admin.reload` |
| `/forge debug <player>` | безопасный вывод станций/сессий/инвариантов | `forgesystem.admin.debug` |
| `/forge givehammer <player>` | выдать ForgeItems hammer из registry | `forgesystem.admin.give` |
| `/forge giveworkpiece ...` | служебная тестовая выдача с валидным schema | `forgesystem.admin.give` + аудит |
| `/forge abort <player>` | управляемая аварийная остановка/сохранение сессии | `forgesystem.admin.recovery` |

Это **предложение итогового UX/API**, не подтверждение наличия команд в текущем 3.1.0-dev. Команды диагностики **не должны** автоматически улучшать quality, восстанавливать недостающие ресурсы или создавать готовую экипировку без явной тестовой/администраторской операции и журнала.

### 17.2. Тестовый контур v2.4

В отдельном прототипе описаны команды:

```text
/fg start
/fg reheat
/fg hammer
/fg hudtest
/fg status
/fg stop
/fg reload
```

Смысл `fg` — smoke-test mini-game без полноценного горна. В production `/fg start` не должен быть обычным способом обхода структуры, слитков и температуры; допустим только при включённом developer flag и OP/debug permission. В GUI/RP документах есть собственные диагностики GUI (`/fguitest ...`) — они также не являются игровым рецептом.

### 17.3. Сборка, runtime, packaging

| Параметр | Исходные сведения |
|---|---|
| Server target | **Purpur 26.2**, Paper API compatible (release/dev версия должна быть проверена на конкретном JAR) |
| Java | **25** в актуальной статусной/технической документации Elementra |
| ForgeMinigame | `0.6.0-SNAPSHOT`, описан отдельный тестовый JAR и standalone сборка |
| Историческая конфигурация прототипа | `26.2.build.129-stable` API / `ru.nyamine` / Gradle в описании v2.4 — точные coordinates сверяются с репозиторием |
| ForgeSystem | `3.1.0-dev` как состояние сводки, не production-integrated |
| Клиент | ванильный Minecraft Java, без Forge/Fabric client mod; RP опционален и имеет fallback |
| Компоненты | ForgeItems schema / Paper adapter / core rules / GUI-RP / tests |

У интеграционного проекта отмечена неполная поставка Gradle Wrapper. Build CI должен проверять Java toolchain и воспроизводимость сборки, а релизная сборка — наличие актуальных `plugin.yml`/`paper-plugin.yml`, регистраций событий, зависимостей, namespace и реальной совместимости Purpur 26.2. **Наличие названия JAR в документе — не свидетельство его запуска или тестирования на сервере.**

## 18. Производительность и эксплуатация

### 18.1. Раздельные частоты обновления

Рекомендованный scheduler:

| Механизм | Частота/триггер | Почему |
|---|---|---|
| `ForgeSession` slider, strike window | каждый тик (20 TPS — номинал) | minigame timing должен быть предсказуемым |
| HUD reaction bar | каждые 2 тика либо по tick-dirty | не слать чрезмерно длинную пачку packet/title |
| GUI dynamic temperature overlay | около 5 тиков и only if changed | icon/model frame меняется не ежетиково |
| Горн: fuel/temperature | примерно 20 тиков или lazy delta calculation | температура может рассчитываться по времени между update |
| Многоблок | при активации, opening, block event / loading | не сканировать каждую структуру каждый тик |
| Offline Workpiece cooling | lazy read/write anchor | нет необходимости тикать каждый item во всех сундуках |

Это ориентиры производительности, а не жёстко зафиксированные исходниками числа для каждого scheduler. Математика slider должна использовать реальное прошедшее монотонное время, чтобы lag/server TPS не давали бонусов за замирания.

### 18.2. Dirty render

- Обновлять температурный frame только после изменения его index и по valid inventory holder, не безусловно заменять весь GUI.
- Избегать пересоздания всей сетки предметов при каждом temperature update: это может сбрасывать cursor item и конфликты кликов.
- Active GUI связан с `ForgeStationId`, а не с заголовком inventory; заголовок может отрендериться иначе у клиента с RP.
- При отсутствии RP не вычислять дорогой text glyph pipeline, если fallback показан через предмет/text.
- Префиксы HUD / unicode codepoint — централизованный registry; соблюдать лимиты длины actionbar и альтернативные шрифты клиента.
- Кеш структуры станции инвалидируется при изменении любого задействованного блока, без принудительной загрузки chunk для проверки соседних блоков.

### 18.3. Thread-safety и измеримость

Состояние `ItemStack`, открытых inventory, World/Block, события Bukkit/Paper и списание ресурсов обслуживаются main server thread. Из внешних потоков можно формировать только immutable данные, безопасно передаваемые на главный поток. Метрики: active sessions, forge stations, temperature updates, GUI viewers, invalid structures, exceptions per transition, duplicate operation blocks, orphan lock recoveries. Телеметрия не должна раскрывать лишнюю пользовательскую информацию.

## 19. Тесты и критерии приёмки

Ниже **план испытаний**, а не заявление о том, что тесты уже прошли. Каждая строка должна быть автоматизирована там, где возможен pure core test, и проверена на реальном Purpur 26.2 при интеграции.

### 19.1. Pure-core / unit

| ID | Сценарий | Критерий PASS |
|---|---|---|
| CORE-01 | Переходы `HOT_BLANK→FORGING→UNQUENCHED` | Только разрешённые transition, stage monotonic |
| CORE-02 | Первое ЛКМ по наковальне | Создание session без начисления hit/result |
| CORE-03 | Stage I MISS на допустимой попытке | Неулучшенная первая стадия и корректный limiter дальнейшей достижимой стадии |
| CORE-04 | Stage II MISS при успешном I | Masterwork становится недостижимым; не повышать автоматически |
| CORE-05 | Stage III успешен после I+II success | Выбор итогового tier соответствует правилам ForgeMinigame |
| CORE-06 | Stage I/II cooling interruption | limiter/paused/resume соответствует исходной стадии; reheating не поднимает cap |
| CORE-07 | Stage III cooling boundary | Поведение соответствует **явно решённому** открытому вопросу §21, нет скрытого arbitrary grade |
| CORE-08 | `sliderPosition`, `randomizeZone`, difficulty params | 60-символьная шкала, нормализованные границы, center shift, edge padding |
| CORE-09 | Параметры времени/minigame | При lag и jitter нет отрицательной duration, невозможны двойные удары из одного input |
| CORE-10 | Календарное охлаждение | `min-temperature` clamp, `updated_at_epoch_ms`, monotonic-like guards на clock skew |
| CORE-11 | Температурные пороги | Проверка событий около 20, 100, 300, 600, 899, 900, 1000 и 1200 °C, где эти значения нормативны для конкретного контекста |
| CORE-12 | Codec roundtrip и schema migration | `instance_id`, product, quality, stage, max cap, author, anchor и history не пропадают |
| CORE-13 | Неполный, неверный или спуфнутый PDC | Ошибка inspect/validation, без forge success |
| CORE-14 | Finished-material progression | Только IRON→DIAMOND→NETHERITE, gold upgrade согласно явной политике, не через Workpiece |

### 19.2. Структура, горн и материалы

| ID | Сценарий | Критерий PASS |
|---|---|---|
| STA-01 | Проверка всех четырёх facing | Одна и та же каноническая multiblock структура валидна при корректном повороте |
| STA-02 | Отсутствует любой обязательный блок | `INVALID_STRUCTURE`; не создан station, не потрачен молот/предмет |
| STA-03 | Обратный/боковой/не-anchor клик | `NOT_ANCHOR` или эквивалентный отказ, без создания |
| STA-04 | Двойной interact в одну tick (main/offhand) | Один station и один `operationId` commit |
| STA-05 | Разборка работающего горна | Station invalidated, нет последующего бесплатного heat/craft |
| STA-06 | Корректный fuel + IRON/GOLD в слоте | Нагрев и списание соответствуют products/materials config |
| STA-07 | DIAMOND или NETHERITE в metal input | Не рождается HOT_BLANK |
| STA-08 | Все 9 изделий | Строгий расход слитков и верный `product_type` / `ForgeMaterial` |
| STA-09 | Нехватка одного слитка, переполненный output | Никакого частичного списания и дублирования |
| STA-10 | Остывание в контейнере/оффлайн | При lazy read корректная температура без per-item ticks |
| STA-11 | Reheat уже готового FINISHED | Не создаётся новый Workpiece и не очищается quality |

### 19.3. GUI, RP и HUD

| ID | Сценарий | Критерий PASS |
|---|---|---|
| UI-01 | Inventory 6×9 | Ровно 54 слота, все индексы `0..53` |
| UI-02 | Fuel=10, metal=12, heat=14, fallback=16 | Контролы работают в указанных местах |
| UI-03 | Armor 38–41 | Актуальная схема, нет старого пересечения 34–37 |
| UI-04 | `scale=auto,1,2,3,4` | У всех масштабов координаты и индикаторы в пределах контракта, без клика по overlay |
| UI-05 | Отсутствует/отклонён RP | Ванильные inventory/button/temperature/text остаются функциональны |
| UI-06 | RP принят и glyphs загружены | 2+ температурных кадров, dynamic bar, шрифт читабелен |
| UI-07 | Быстрая смена температуры и закрытие окна | Не создаются предметы в декоративных слотах, нет cursor duplication |
| UI-08 | SHIFT, number-key, drag, double-click | Защищённые слоты и fake visual items не перемещаются |
| UI-09 | Slider, zone/feedback в реальном времени | ASCII/RP fallback синхронизирован с core state, zone не вылезает за 60 symbols |
| UI-10 | Первый удар, MISS/GOOD/EXCELLENT feedback | Stage updated once, подсказка не врёт относительно server authoritative grade |

### 19.4. Защита, recovery, интеграция

| ID | Сценарий | Критерий PASS |
|---|---|---|
| INT-01 | Два игрока на одном instance | Эксклюзивная session, вторая попытка отклонена |
| INT-02 | Swap/drop/death/quit во время FORGING | Не теряется и не дублируется Workpiece |
| INT-03 | Crash/restart в момент создания HOT_BLANK | Не списаны ресурсы без предмета и не создан предмет без списания |
| INT-04 | Quench при `WATER_CAULDRON level=1` | Ровно один переход и `level` уменьшается на 1 |
| INT-05 | Quench при пустом котле | Нет изменений |
| INT-06 | Armor quench | Сразу готовая vanilla броня с метаданными, без рукояти |
| INT-07 | Tool quench | Результат `QUENCHED_PART`, а не готовый инструмент |
| INT-08 | Стоимость рукояти: 1/2/2/2/2 sticks | Ровно один controlled transition на наковальне, стоимость списана один раз |
| INT-09 | Smithing IRON→DIAMOND, DIAMOND→NETHERITE | Только FINISHED и по policy стоимости, quality/author сохранены |
| INT-10 | Ванильный обход craft diamond gear | Не создаёт предмет, минуя предписанный progression |
| INT-11 | Ванильная furnace/grindstone/anvil/crafting и сторонние inventory | Технические носители нельзя случайно обойти или переплавить |
| INT-12 | Hot reload при активной ForgeSession | Конфиг атомарен, session snapshot стабилен, предмет не теряется |
| INT-13 | Orphan lock after crash | Управляемое recovery без стороннего доступа/дублирования |
| INT-14 | `/fg` dev only в production | Обычный игрок не может обойти рецепты/этапы |
| INT-15 | Purpur 26.2 + Java 25 smoke | Enabled/disabled без error, GUI, RP fallback, 1 полный игровой цикл до FINISHED и upgrade |

### 19.5. Definition of Done

Готовым нельзя считать только «работает HUD» или «выдаёт HOT_BLANK». Минимальный релизный vertical slice: реальный многоблок → fuel/heat → 9 конфигураций заготовок → Workpiece PDC → 3-этапная мини-игра с ограничителем качества → корректное остужение/reheat → quench → рукоять по необходимости → vanilla finished item → diamond/netherite upgrade → сохранность при logout/death/restart → RP и fallback → replayable tests на указанном runtime. Каждое звено подтверждается логом теста/JAR/версией, а не только документом.

## 20. План интеграции и технический долг

### 20.1. Последовательность внедрения

| Этап | Работа | Артефакт и проверка |
|---|---|---|
| P0 — зафиксировать контракты | Объединить product enum, quality, state schema, event naming, slots, progress/cap, material values | Contract tests + approved open issues |
| P1 — ForgeItems | Реализовать codec, schema migration, item registry/hammer, locks и защиты carrier | Roundtrip, spoof, drop/transfer protection tests |
| P2 — ForgeStation + Workpiece | Перевести `HOT_BLANK` из недолговечного предмета в авторитетный Workpiece; связать реальный горн и temperature | Reheat и item persistence E2E |
| P3 — core mini-game | Перенести StageRules/slider/zone/hit/quality из v2.4, заменить test command на реальную наковальню | CORE-01...CORE-11 |
| P4 — Lifecycle + recovery | Реализовать `FORGING`, паузу/возобновление, pre/post-stage transitions, две блокировки | Logout/death/duplicate tests |
| P5 — Finishing | `UNQUENCHED`, water cauldron, `QUENCHED_PART`, 5 handle recipes | INT-04...INT-08 |
| P6 — Material tiers | Smithing Iron→Diamond→Netherite с quality/blacksmith metadata, закрыть обходные рецепты | INT-09...INT-11 |
| P7 — GUI/RP | Целевая 54-slot карта, bar fonts, frame overlay, resource pack fallback, GUI safe click handling | UI-01...UI-10 |
| P8 — QA/release | Integrations, unit/e2e regression, Java25+Purpur runtime, performance and recovery | Tests + startup logs + changelog |

Порядок некоторых независимых работ может выполняться параллельно; **нельзя** допустить, чтобы визуальная часть скрывала отсутствие завершённого Workpiece state machine.

### 20.2. Замечания к наследуемому прототипу

Перечисленные нюансы отмечены в материалах v2.4 и интеграции как legacy/debt либо должны быть явно проверены перед переносом:

- Монолитный `ForgeMinigamePlugin` с mixed command/listener/runtime/UI обязан разделиться на core/domain и Paper presentation adapters.
- `ForgeSession.reheat()` и `/fg reheat` сейчас оперируют runtime-копией; в production первично изменение *теплового якоря Workpiece*, из которого будет восстановлена новая Session.
- Нельзя переносить `System.nanoTime()` как timestamp предмета в persistent storage; wall clock — `epoch ms`, monotonic — для прошедших длительностей.
- В интеграционной документации упоминается отсутствие полноценного Gradle Wrapper в поставке. CI должен иметь воспроизводимый bootstrap.
- В v2.4 упомянуты несогласованный заголовок конфиг-версии 3/4 и неиспользуемые параметры/методы (например `feedbackTicks`, `visualZoneWidth()`) — проверить, какие из них актуальны фактическому исходному дереву.
- Старый «получить новый slot/technical item при клике» обход нельзя оставлять как источник предметов вне ForgeStation transaction.
- Старый enum качества BASIC/IMPROVED/MASTERWORK и новый BASIC/GOOD/EXCELLENT/MASTERWORK не смешивать без явного качества/истории (см. §21).

## 21. Разночтения и открытые вопросы

Этот раздел — **не список автоматически исправленных фактов**. Он разграничивает конкретные source discrepancies, проектные решения для интеграции и то, что всё ещё требует выбора владельца продукта/разработки.

### 21.1. Принятые правила при объединении

| Вопрос | Известные редакции | Правило единой спецификации |
|---|---|---|
| GUI armor slots | Ранние схемы ставят броню в **34–37**; более новое `GUI_RP_TZ_v2` — **38–41** | В целевой production `38–41`. Старую карту описать только как legacy/migration |
| Категории качества | Прототип v2.4 и Full Mechanics используют не полностью совпадающие имена | В production четыре последовательных категории: `BASIC`, `GOOD`, `EXCELLENT`, `MASTERWORK`. Миграция legacy enum не должна автоматически выдумывать дополнительный успешный stage |
| Команда старта и реальная ковка | Standalone `/fg start` vs физическая наковальня + HOT_BLANK | Только реальная Workpiece и наковальня в production; `fg` dev-only |
| Reheat | Standalone `/fg reheat` делает новую runtime session с тестовой T vs многоблок нагревает PDC Workpiece | Reheat происходит на реальном горне; новая Session **восстанавливается из обновлённого Workpiece** |
| Активация кузницы | Контракт ForgeItems hammer + BLAST_FURNACE, документы различают конкретные UI/interaction шаги | ForgeActivationGateway принимает click по каноническому anchor; строгий способ применения молота закрепить в acceptance tests |
| Прогресс | ForgeSession держит stage/runtime state vs ForgeItems Workpiece между сессиями | После каждого commit authoritative прогресс записывается в Workpiece |
| Статус реализации | Готовый standalone mini-game и dev-горн vs необходимая полная игровая цепочка | Называть текущий уровень отдельно для каждого модуля; **не** маркировать всю ForgeSystem как RELEASE_READY |

### 21.2. Не закрытые решения

| № | Что не определено однозначно | Что требуется до релиза |
|---|---|---|
| O-01 | Граница при остывании на Stage III: formula `max_reachable_stage` может допускать Masterwork после reheat, а отдельная формулировка требует все 3 стадии до cooling для максимума | Продуктовое решение; тест Stage III cold→reheat→perfect hit; синхронизировать core/описание |
| O-02 | Подробные numeric бонусы качества: durability, ремонт, характеристики, wear | Таблица баланса и механика применения, не изобретать значения в реализации |
| O-03 | Полная таблица алмазных upgrade costs для всех 9 изделий | Утвердить `products.yml` и отдельные рецепты; пример sword+2 diamonds не обобщать |
| O-04 | Поведение `GOLDEN_*` при progression to diamond; Full Mechanics явно подчёркивает IRON→DIAMOND | Разрешить или запретить gold upgrade в согласованной policy |
| O-05 | Точные ставки fuel/heat и разная длительность прогрева для dev/current production | Зафиксировать материалы/формулы и интеграционные тесты версии 26.2 |
| O-06 | Схема/формат хранения и journal для station states и atomic transitions | Выбрать persistence backend и recovery journal без потери на crash |
| O-07 | Какие атрибуты получает FINISHED item и как Forge metadata переживает vanilla repair/enchant/smithing | Совместимость и versioned finished-item codec |
| O-08 | Политика устаревших item schema и дубликатов `instance_id` | Миграция, карантин, безопасное восстановление, ограничения admin tooling |
| O-09 | Точные Unicode codepoint, font offsets, spacing у resource pack и общесерверная регистрация CustomModelData | Утвердить pack asset registry и исключить коллизии |
| O-10 | Конкретный статус собранной ForgeSystem на новом исходном дереве, а не по текстовой сводке | Компиляция, smoke-test, сохранённые логи и набор протестированных функций |
| O-11 | Нормативный шаг `epoch_ms` при системном переводе часов назад и отрицательный elapsed | Утвердить clamp/clock-skew policy и тесты offline cooling |
| O-12 | Практические ограничения creative mode, сторонних inventory/trade плагинов и multi-hand | Интеграционная матрица сервера и anti-dupe test suite |

### 21.3. Как фиксировать изменения

Каждое закрытие вопроса `O-XX` должно иметь: новую норму, основание (дизайн/реальный runtime), дату, affected modules (core/items/station/GUI/RP), изменения миграции, regression test ID. Если исходники расходятся с нормой, сохранять оба свидетельства до сознательной миграции.

## 22. Источники и трассировка

### 22.1. Использованные девять первичных документов

| Код | Документ | Назначение |
|---|---|---|
| S1 | `ServerMine_ForgeSystem_Full_Mechanics(1).md` | Полный игровой путь, iron/gold, 9 рецептов, heating, stages, reheat, quench, upgrades |
| S2 | `ServerMine_ForgeMinigame_v2.4_Documentation(1).md` | Реальный прототип reaction core, зоны/скорости/оценка, HUD, команды, status/testing |
| S3 | `ForgeSystem_Forge_Mechanics(1).md` | Ранние референсные механики ForgeStation, hot blank, температуры и testing parameters |
| S4 | `ServerMine_ForgeSystem_Integration_Gorn_v1(2).md` | Целевая интеграция standalone мини-игры с физическим горном, state managers, contracts |
| S5 | `ForgeItems_Module_Documentation(1).md` | Item Registry, Workpiece/PDC codec, hammer, technical carrier, locks, API, safety |
| S6 | `ServerMine_ForgeSystem_GUI_RP_TZ_v2(1).md` | Production GUI 54 slots, slots 38–41, RP HUD/fonts/frames, fallback, rendering, events |
| S7 | `Elementra_Концепция_v3.1.1_2026-10-08(1).md` | Место кузнечной системы в общем игровом проекте Elementra |
| S8 | `Elementra_Техническая_спецификация_v3.1.1_2026-10-08(1).md` | Общепроектные технические границы, компоненты и evidence/status semantics |
| S9 | `Elementra_Статус_и_Roadmap_v1.1.1_2026-10-08(1).md` | Наиболее свежие подтверждённые в документах статусы dev/standalone и roadmap |

Файлы S7/S8 очень объёмны и имеют множество подсистем вне ковки. В эту интеграционную документацию внесены только положения, относящиеся к ForgeSystem/ForgeMinigame/ForgeItems, общему стандарту платформы и статусу работ. Не следует трактовать остальную концепцию Elementra как завершённую разработку ForgeSystem.

### 22.2. Быстрая матрица трассировки

| Требование / поведение | Источник(и) | Где описано здесь |
|---|---|---|
| Статус, матрица этапов SPEC→RELEASE_READY | S7, S8, S9, S2 | §1, §20 |
| 9 изделий, iron/gold, расход, запрет прямого diamond | S1, S4 | §3, §7, §13 |
| Multiblock `BLAST_FURNACE` и ForgeHammer | S1, S3, S5, S6 | §5, §14 |
| ForgeStation thermal and fuel | S1, S3, S6 | §6 |
| Workpiece schema, codec, identity, lock | S5, S4 | §3–4, §11, §14–15 |
| 3-stage timing, score, limiter, cooling/reheat | S1, S2, S4 | §9–12, §21 |
| RP Unicode/fonts, GUI frame/scale/fallback | S6, S2 | §8, §10, §18 |
| Quench in WATER_CAULDRON, handle, upgrade | S1, S5, S4 | §13 |
| Paper events, transaction security | S5, S6, S4 | §14–15 |
| Reload, diagnostics, test plan, migration | S2, S4, S5, S6 плюс инженерное проектирование на их основе | §16–20 |

### 22.3. Глоссарий

| Термин | Значение |
|---|---|
| `ForgeStation` | Валидированная физическая многоблочная кузница с температурой, топливом и GUI |
| `ForgeHammer` | Кастомный предмет для активации/участия в кузнечном взаимодействии; идентичность через ForgeItems |
| `Workpiece` | Persistent технический кузнечный предмет: стадия, металл, продукт, качество, автор, температура, identity |
| `HOT_BLANK` | Изготовленная на горне горячая заготовка, готовая к переносу на наковальню |
| `FORGING` | Заготовка в процессе 3-этапной обработки; переносимый progress и runtime session различаются |
| `ForgeSession` | Необязательное к сериализации runtime состояние одного активного исполнения mini-game на наковальне |
| `max_reachable_stage` | Ограничитель наивысшего результата/стадии после ошибок или охлаждения по StageRules |
| `UNQUENCHED` | Пройдена ковка, но предмет ещё нуждается в закалке; не готовая экипировка |
| `QUENCHED_PART` | Закалённая деталь инструмента/оружия без рукояти |
| `FINISHED` | Материализованный ванильный инструмент/оружие/броня с server-side Forge metadata |
| `thermal anchor` | Температура и время последнего authoritative расчёта, из которых вычисляется lazy cooling |
| `RP` | Серверный resource pack с model/font/glyph/GUI; fallback обязателен |
| `PDC` | Bukkit PersistentDataContainer: server-side идентификация и сохранение Forge metadata на ItemStack |
| `idempotency` | Повтор того же запроса или события не создаёт ещё один предмет/расход/результат |
| `migration` | Контролируемое преобразование старой Item/quality/schema версии в новую с сохранением истории |

---

**Конец технической спецификации v1.0 (08.10.2026).** До утверждения открытых вопросов §21 этот файл следует использовать как *целевой контракт разработки и тестирования*, а не как утверждение, что все его части уже существуют в runtime.