# Как называть takeoffs

Правила именования takeoff-items в OurPlanCore. Имена напрямую влияют на:

- **Auto-routing** — куда упадёт новый item: `sqfts` или `walls / {этаж} floor walls`.
- **Sort order** — в каком порядке item встанет внутри папки, в легенде и в Excel export.
- **Folder matching** — нужная папка ищется по **токенам** имени, а не по тому, какая
  папка сейчас выделена.

!!! tip "Главное правило"
    Используй короткие токены из словарей ниже. Программа надёжнее распознаёт
    `ext 2x6 x`, чем `Exterior wall 2x6 staggered`. Длинные читабельные описания
    держи в `Notes`, не в имени.

!!! warning "Auto-routing срабатывает только у нового item — и зависит от типа замера"
    Роутинг выполняется **при создании item** (`New Item`) и **жёстко завязан на тип
    замера**:

    - **`Area`** может уйти в **`sqfts`**;
    - **`Line`** может уйти в **`walls / {этаж} floor walls`**;
    - **`Count` и `Point` не роутятся никогда** — остаются в выбранной папке.

    Ручной перенос item'а всегда побеждает — авто-роут повторно не срабатывает.

## Как разбирается имя { #normalize .kb-section-title .kb-st--cyan }

Имя item'а **не переписывается** — оно приводится к нижнему регистру и режется на
**токены**. Токен — это цельное слово/размер/этаж, а не подстрока.

| Шаг | Правило | Пример |
| --- | --- | --- |
| Регистр | всё → lowercase, обрезаются пробелы по краям | `Ext 2x6 X` → `ext 2x6 x` |
| Токенизация | режется по любым не-буквам/цифрам (пробел, `_`, `-`, `.`, `/`) | `ext_2x6-x` → токены `{ext, 2x6, x}` |
| Размеры | `2x4`/`2x6`/`2x8` — цельный токен | `2x6` не распадается на `2` и `6` |
| Этажи | `1st`…`8th` — цельный токен | `1st`, `2nd`, … |
| Matching | сравнение **по цельным токенам**, не по подстроке | `corners` ≠ `cor` |

**Aliases** — токены-синонимы, которые считаются равными (имя при этом не меняется,
просто оба варианта распознаются):

| Alias | Равно |
| --- | --- |
| `base` | `basement` (для этажа — ещё `bsmnt`) |
| `blcny` | `balcony` |
| `cant` | `cantilevered` |
| `cor` | `corr` |
| `dem` | `demo` |
| `rf` | `roof` |
| `1st … 8th` | `first … eighth`, а также `floor N` / `flr N` / `level N` / `lvl N` |

!!! note "«Roof family» — не alias, а следствие токенов"
    `rf`, `rf x`, `rf mtl x`, `roof` классифицируются и сортируются одинаково просто
    потому, что все содержат токен `rf`/`roof`; лишние `x`, `mtl` игнорируются.
    А вот `overframe x` **не** содержит sqft-токена и авто-роут не пройдёт.

## SQFT items (только `Area`) { #sqft .kb-section-title .kb-st--green }

Если тип замера — `Area` **и** имя содержит один из токенов ниже (или в имени найден
этаж), item уходит в папку **`sqfts`**. Статус: `Auto routed to Takeoffs/sqfts.`

| Группа | Токены |
| --- | --- |
| База / floors | `base`, `basement`, а также этаж `1st`…`8th` (и алиасы `first…eighth`, `floor N`) |
| Обобщённые SF | `sqft`, `sf`, `sft` |
| Outdoor | `deck`, `porch`, `blcny`, `balcony`, `cant`, `cantilevered` |
| Roof / flat | `flat`, `rf`, `roof` |

!!! info "Внутри `sqfts` нет под-папок по этажам"
    Все sqft-item'ы ложатся прямо в `sqfts` (в отличие от walls, где есть floor-папки).

### Sort внутри `sqfts`

```text
base / basement
1st → 2nd → 3rd → 4th → 5th → 6th → 7th → 8th
deck
porch
blcny / balcony
cant / cantilevered
flat
rf / roof
прочие имена без токена — последними, по натуральному имени
```

!!! warning "Floors не объединяй"
    `1st`, `2nd`, `3rd`… остаются **отдельными** item'ами, даже если геометрия похожа.
    Не суммируй totals по нескольким этажам только потому, что имена близкие — это
    правило конкретных клиентов (напр. WM).

## Wall items (только `Line`) { #wall .kb-section-title .kb-st--magenta }

Если тип замера — `Line` **и** имя содержит один из wall-токенов — item
классифицируется как стена и уходит в `walls`:

| Токен | Расшифровка |
| --- | --- |
| `wall` / `walls` | явное слово «стена» |
| `corner` / `corners` | углы (corner posts/studs) |
| `ext` | exterior |
| `cor` / `corr` | corridor |
| `dem` / `demo` | demising |
| `2x4` / `2x6` / `2x8` | по размеру стойки |
| `half` | half-height |

### Auto-route по активному листу

Программа берёт **имя активного листа**, затем поднимается по его папкам вверх и ищет
**этаж**. Item ложится в `walls / {этаж} floor walls`.

Этаж распознаётся по токенам:

```text
base / basement / bsmnt          → basement floor walls
1st … 8th  (и first…eighth,      → «1st floor walls», «2nd floor walls», …
            floor N / flr N /
            level N / lvl N)
```

!!! danger "В детекторе этажа НЕТ `roof` и `parapet`"
    Стена, нарисованная на roof- или parapet-листе (без токена этажа в имени листа
    или его папок), **не** определит этаж и ляжет в корень `walls` со статусом:
    `Auto routed to Takeoffs/walls; no sheet floor was detected.`
    Детект идёт до **8th** этажа включительно.

**Примеры:**

| Активный лист | Новый `Line`-item | Куда попадёт |
| --- | --- | --- |
| `A101 1st` | `ext 2x6 x` | `walls / 1st floor walls` |
| `A102 2nd` | `cor 2x6 x` | `walls / 2nd floor walls` |
| `Floor 3 Plan` | `corr (2) 2x4 x` | `walls / 3rd floor walls` |
| `A100 Basement` | `dem 2x6 x` | `walls / basement floor walls` |
| `Roof Plan` | `ext 2x6 x` | `walls/` (root) + «no sheet floor was detected» |

### Sort внутри floor-папки walls

```text
corner / corners
ext            (внутри — по убыванию последнего числа в имени)
cor / corr
dem / demo
2x8 → 2x6 → 2x4
half
прочие имена без токена — последними, по натуральному имени
```

Сами floor-папки в `walls` сортируются по этажу: `basement → 1st → 2nd → …`.

## Когда auto-routing НЕ срабатывает { .kb-section-title .kb-st--orange }

| Кейс | Поведение |
| --- | --- |
| Тип замера `Count` / `Point` | Не роутится — остаётся в выбранной папке |
| `Area`, но нет sqft-токена и этажа | Остаётся в выбранной папке · `New item: {name}.` |
| `Line`, но нет wall-токена | Остаётся в выбранной папке · `New item: {name}.` |
| `Line` с wall-токеном, но этаж листа не определён | `walls/` (root) · `Auto routed to Takeoffs/walls; no sheet floor was detected.` |
| Имя похоже на токен, но это подстрока | Не роутится: `corners` ≠ `cor` (но `corners` — **свой** wall-токен) |
| User вручную перенёс item | Manual побеждает; auto-route не повторяется |

## Примеры { .kb-section-title .kb-st--amber }

| Имя (тип) | Токены | Куда |
| --- | --- | --- |
| `1st` (Area) | `{1st}` | `sqfts` |
| `Porch` (Area) | `{porch}` | `sqfts` |
| `cant` (Area) | `{cant}` = `cantilevered` | `sqfts` |
| `rf mtl 1` (Area) | `{rf, mtl, 1}` | `sqfts` (по `rf`) |
| `overframe x` (Area) | `{overframe, x}` | выбранная папка (нет sqft-токена) |
| `ext 2x6 x` (Line, лист `A101 1st`) | `{ext, 2x6, x}` | `walls / 1st floor walls` |
| `corners` (Line) | `{corners}` | `walls / {этаж}` (это wall-токен!) |
| `Exterior 2x6 wall` (Line) | `{exterior, 2x6, wall}` | `walls` (по `wall`; `ext`≠`exterior`) |

<!--
СКРИНШОТЫ — переснять из текущей сборки v2.2.5:
Эта страница текстовая, скринов не содержит. Если добавлять — снять окно New Item
с выбором типа замера (Area/Line/Count) и статус-строку авто-роута снизу.
-->

## See also

- [OurPlanCore — программа](ourplanecore.md) — полный гайд по интерфейсу
- [Создание Job и хранение](job-creation-storage.md) — где что лежит на диске
- [Workflow](../start/workflow.md) · [Структура takeoff](../start/takeoff-structure.md)
- [Советы и важные вещи](boss-feedback-rules.md)
