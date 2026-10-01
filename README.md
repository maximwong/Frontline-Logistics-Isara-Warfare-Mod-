# Frontline Vehicles v5.70.6 — 新增内容 / What's New / Что нового

> 本包含此前「超级机械化 MOD」的全部内容并大幅扩展。
> Contains everything the earlier "Super Mechanization" MOD had, greatly expanded.
> Включает всё из мода «Сверхмеханизация» и значительно расширяет его.

---

## 本版新增（v5.68.2 → v5.70.6）/ What's new since v5.68.2 / Что нового с v5.68.2

### 中文

| 变更 | 说明 |
|---|---|
| **新增载具：五十铃ELF4冷藏车** | **载具总数 43 台 → 44 台。** 3.5 吨级平头冷藏车，配用**全新绘制**的八方向贴图（格 32×24）。价位、载客、载重与 IFA W50 同族，另加装一台「**制冷机**」部件。 |
| **冷链规则** | 车厢里的鲜食在「**制冷机完好（≥ 70%）+ 引擎开启 + 油箱有油**」时保持冷冻、不会变质；**打坏制冷机（< 70%）/ 熄火 / 断油**立即停止制冷，货物按正常速度腐坏。**卸货后不再受保护**。新刷出的冷藏车制冷机默认完好（想看失效效果，把它打坏即可）。 |
| **制冷耗油** | 制冷机由本车引擎驱动、与原车共用一箱燃油，按**怠速油耗**持续扣油 —— 停着不动也在烧，油烧完就自动停机。 |
| **加入民用轻型载具池** | 每存档 **至少 1 台、至多 3 台**（1 台保底 + 池内抽取）。配额写进存档，**读档不会重刷、也不会重置**。数量可配置：`[Pool] IsuzuMaxPerMatch`，默认 **3**，**0 = 完全不刷**（上限 4）。 |
| **【冷藏中】标记** | 正在冷藏的食物，物品说明里多一行 **【冷藏中】**；货物面板中对应行显示**原版雪花图标**。制冷停止或食物离开冷藏车，标记自动消失。 |
| **默认不自动卸货** | 冷藏车开进基地 / 仓库范围时**不会自动卸货**，冷链不会被打断；可在单位面板用原版「自动卸货」勾选框逐车改回。`[Reefer] DisableAutoUnload` |
| **停车 / 下车不熄火** | 游戏在停车、下车时会自己熄火；本 MOD 的冷藏车会**自动重新点火**，保住制冷；手动关火依然有效。`[Reefer] KeepEngineOnPark`，false = 恢复原版行为。 |
| **HUD 冷藏状态行** | 屏幕左上角显示每台活跃冷藏车的状态：是否冷冻、制冷机耐久、油量、引擎开关。 |
| **保底刷新更可靠** | 保底那台冷藏车的落点改为三级降级：**配置的城镇民用建筑 → 地图上任何民用建筑 → 道路兜底**，所以任何地图都保证「至少 1 台」。同一进程里连开两局时，上一局占用的格子不再影响新局。 |
| **其它修复** | 自动卸货竞态；食物离开冷藏车后【冷藏中】标记残留；车辆与部件文案改写为真实车型（五十铃 ELF 第四代、ISUZU 4JG2 引擎、MSB-5S 变速箱、DENSO 制冷机组）。 |

### English

| Change | Notes |
|---|---|
| **New vehicle — ISUZU ELF4 reefer truck** | **Vehicle count 43 → 44.** 3.5 t cab-over refrigerated box truck with **brand-new** eight-direction artwork (32×24 cells). Price, passengers and cargo are the W50 family values, plus one extra part: the **reefer unit**. |
| **Cold-chain rule** | Food in the box stays frozen while the **reefer unit is intact (≥ 70 %) AND the engine is ON AND there is fuel**. A broken unit (< 70 %), a switched-off engine or an empty tank stops the cooling and the cargo spoils at the normal rate. **Unloading ends the protection.** Freshly spawned reefers keep an intact unit — wreck it if you want to see the failure path. |
| **The fridge burns fuel** | Driven by the truck's own engine from the same tank, at the **idle** rate — it keeps burning while parked, and stops by itself when the tank runs dry. |
| **Joins the civilian light pool** | **1 to 3 per savegame** (one guaranteed + pool draws). The quota is written into the savegame, so **reloading never re-spawns or resets it**. Config: `[Pool] IsuzuMaxPerMatch`, default **3**, **0 = never spawn** (capped at 4). |
| **【冷藏中】 tag** | Refrigerated food carries the tag on its item description, and the matching row in the cargo panel shows the **stock snowflake icon**. The tag disappears when cooling stops or the food leaves the truck. |
| **No auto-unload by default** | A reefer does **not** dump its cargo when it drives into a base / storage area. Flip the stock auto-unload checkbox in the unit panel per truck to change it. `[Reefer] DisableAutoUnload` |
| **Parking / dismount no longer kills the fridge** | The game switches the engine off by itself when the truck parks; our reefers **re-start it**, and a manual shutdown still sticks. `[Reefer] KeepEngineOnPark`, false = stock behaviour. |
| **Reefer HUD line** | Top-left corner shows each active reefer: frozen or not, unit wear, fuel, engine switch. |
| **More reliable guaranteed spawn** | The guaranteed truck's anchor degrades in three steps: **configured town/civilian buildings → any civilian building → road tile**, so "at least 1 per savegame" holds on any map. A second new game in the same process no longer inherits the previous game's occupied tiles. |
| **Other fixes** | Auto-unload race; the 【冷藏中】 tag left behind after food left the truck; text rewritten for the real vehicle (ISUZU ELF 4th gen, ISUZU 4JG2 engine, MSB-5S gearbox, DENSO reefer set). |

### Русский

| Изменение | Описание |
|---|---|
| **Новая машина — рефрижератор ISUZU ELF4** | **Всего машин: 43 → 44.** 3,5-тонный грузовик с изотермическим фургоном и **полностью новой** графикой восьми направлений (ячейки 32×24). Цена, пассажиры и груз — как у семейства W50, плюс отдельная деталь — **холодильная установка**. |
| **Холодовая цепь** | Продукты в фургоне не портятся, пока **установка цела (≥ 70 %), двигатель включён и в баке есть топливо**. Разбитая установка (< 70 %), выключенный двигатель или пустой бак прекращают охлаждение — груз портится с обычной скоростью. **После разгрузки защита прекращается.** У новых рефрижераторов установка целая; разбейте её, если хотите увидеть отказ. |
| **Холодильник расходует топливо** | Он работает от двигателя машины и берёт топливо из общего бака по **расходу на холостом ходу** — расход идёт и на стоянке, а когда бак опустеет, охлаждение прекратится само. |
| **Входит в пул «гражданские лёгкие»** | **От 1 до 3 на сохранение** (одна гарантированная + выпадения из пула). Квота хранится в сохранении: **перезагрузка не создаёт новых и не сбрасывает счётчик**. Настройка: `[Pool] IsuzuMaxPerMatch`, по умолчанию **3**, **0 = не появляется** (максимум 4). |
| **Метка 【冷藏中】** | У охлаждаемых продуктов в описании предмета появляется строка **【冷藏中】**, а в панели груза у соответствующей строки — **штатный значок-снежинка**. Метка снимается, когда охлаждение прекращается или груз покидает машину. |
| **Автовыгрузки по умолчанию нет** | Рефрижератор **не выгружает** груз при въезде на базу / склад. Вернуть можно штатной галочкой «автовыгрузка» в панели юнита для каждой машины. `[Reefer] DisableAutoUnload` |
| **Стоянка и высадка больше не глушат холодильник** | Игра сама глушит двигатель при парковке, наши рефрижераторы **заводят его снова**; ручное выключение по-прежнему работает. `[Reefer] KeepEngineOnPark`, false = как в оригинале. |
| **Строка состояния на HUD** | В левом верхнем углу — состояние каждого активного рефрижератора: охлаждает или нет, износ установки, топливо, двигатель. |
| **Надёжная гарантированная машина** | Точка привязки гарантированной машины деградирует в три шага: **настроенные городские/гражданские здания → любое гражданское здание → клетка дороги**, поэтому «минимум 1 на сохранение» соблюдается на любой карте. Вторая новая игра в том же процессе больше не наследует занятые клетки предыдущей. |
| **Прочие исправления** | Гонка при автовыгрузке; остававшаяся метка 【冷藏中】 после выгрузки; тексты переписаны под реальную машину (ISUZU ELF 4-го поколения, двигатель ISUZU 4JG2, КПП MSB-5S, агрегат DENSO). |

---

## 上一版新增（v5.62 → v5.68.2）/ Previous release / Предыдущий выпуск

| 变更 | 说明 |
|---|---|
| **刷新载具开局燃料 0-30 L** | 所有本 MOD 刷出的载具统一给 0-30.0 升燃料（按油箱容量夹紧）。`[Spawn Rules] SpawnFuelMin/Max` |
| **随机朝向** | 刷出的载具随机取八个朝向之一（此前全部朝默认方向）。 |
| **同格叠放不重合** | 同一格最多 3 台，第 2/3 台贴图自动错位，不会完全重叠。 |
| **多色载具名称去掉颜色** | LADA ×3、B1000 ×3、丰田陆巡 ×2 的名称不再带"白色/红色/蓝色…"，只靠涂装区分。 |
| **读档重复刷新 BUG 修复** | 城市道路 / 车祸点 / 车辆刷新点三条规则此前**没有把"已刷过"写进存档**，导致每次读档都再刷一批。现已在 `mapArmsSpawnActivedList` 中记账（城市道路用哨兵键，全图配额变成"一存档一次"）。 |
| **城市道路改为全图配额** | 此前是"每栋建筑 2-3 台"，会导致整张地图上百台；现为**全图** 2-3 台。 |
| **医疗清单调整** | 救护车随车医疗物资移除"抗感染药"，加入"手术用具"。 |
| **瓦莲娜（《雪松》）** | 已实装为一台独立载具（132×192 图集，格 44×64），但因用户要求**暂时隐藏**（不在 F5 循环中出现）。恢复方式见开发手册。 |
---

## 中文

### 一、载具：4 台 → **44 台**

**基础 26 台**：UAZ-469 吉普、ZIL-4310 卡车、UAZ-452 面包车/医疗车、LADA-2501（白/红/蓝）、虎式装甲车（基础/遥控武器站）、RAF-22031 救护车、ZIL-130 消防车、B1000 货车（蓝/红/黄）、ZIL-49061 搜救车、丰田陆巡（白/黑）、URAL-4320、IFA W50 栏板卡车、UAZ-469 警车、VAZ-2106 警车、UAZ-452 警车、ZIL-4310 内务部运兵车、PAZ-3205 警用巴士、ZIL-4310 民用重型挡板卡车、**五十铃ELF4冷藏车**（带制冷机，全文见上）。

**型号变体 18 台（新增）**：B1000 / W50 / ZIL-4310民用 各分 **服装 / 家电 / 家具 / 农业 / 工业 / 运煤** 六个型号，每型只装本型物资。

### 二、载具池（4 个）

民用轻型 22 台（含 **五十铃ELF4冷藏车**）· 民用重型 7 台 · 应急轻型 5 台（含 **UAZ-452 警用面包车**）· 应急重型 2 台。

### 三、建筑刷新规则（全新）

1. **城镇医院** → 1-2 台民用救护车（全图配额，停车场优先）
2. **城镇火车站** → 1-2 台重型民用载具
3. **车辆刷新点** → 每车位 **20%** 概率；池内 **LADA 40% / B1000 30% / 丰田 20% / W50 10%**
4. **车祸点** → **3-7 台事故车（耐久 0-30%）** + **0-3 台应急救援车**（型号互不重复）
5. **城市道路** → **2-3 台重型应急载具（耐久 0-70%）**
6. **民用建筑（冷藏车保底）** → 每存档**保底 1 台**五十铃冷藏车：先取车辆刷新用的城镇民用建筑，没有则取地图上**任何民用建筑**，再没有则落到**道路**上。只受 `[Pool] IsuzuMaxPerMatch` 控制，**不受** `[Vehicle Refresh] Enabled` 影响。

各规则**同格最多 3 台**，每个车位/建筑只刷一次并**写入存档**。

### 四、随车物资（全新体系）

* **LADA / 丰田**：0-2 黄瓜罐头 + 0-2 肉罐头 + 0-2 黄油 + 0-1 铲子 + 0-1 快速修理包 + 0-3 布料
* **B1000 / W50 / ZIL民用 六型号**：服装 10-50 布料｜家电 5-10 家电 + 0-4 芯片｜家具 5-20 家具｜农业 10-40 面粉 + 0-10 化肥｜工业 0-2 电池 + 0-2 车床 + 10-30 废旧金属 + 5-10 工具｜运煤 10-200 煤炭
  —— **W50 全部上限 ×2，ZIL-4310民用 全部上限 ×5**
* **救护车**：止血包/血浆/吗啡/麻醉剂/手术用具，各 **0-15**
* **消防车**：0-5 铲子 + 0-5 铁镐
* **警车**：10-100 发 5.45mm 散装弹药
* **重型应急（城市道路）**：200-1000 发 5.45mm 散装弹药

### 五、部件与文案

* **每台车拥有自己的部件套装**（克隆件）：底盘、油箱、检修隔舱、货厢等按真实车型命名并各配简介 —— 面包车不再挂着「皮卡货斗」。
* **发动机按真实型号命名**：W50 → 东德 **IFA 4 VD 14,5/12-1 SRW**；陆巡 → **丰田 2H**（**功率未改动**）。
* **冷藏车按真实车型取名**：五十铃**第四代 ELF**（3.5 吨级，1984 年起投产）冷藏车，引擎 **ISUZU 4JG2**、变速箱 **MSB-5S**、制冷机组 **DENSO**（同代车型亦有装用冷王者）。
* **部件耐久随机**：介乎「完全损坏」与「略有磨损」，区间可配置。
* **世界观对齐**：文本中的「苏联」一律显示为「伊萨拉」。

### 六、按键

`F5` 循环生成下一台（44 台）· `Shift+F5` 回退 · `F4` 开局增援 · `F7` / `F10` 调试导出。

---

## English

### 1. Vehicles: 4 → **44**

**26 base**: UAZ-469, ZIL-4310, UAZ-452 van/ambulance, LADA-2501 ×3, Tiger APC (base/RWS), RAF-22031, ZIL-130 fire, B1000 ×3, ZIL-49061, Land Cruiser ×2, URAL-4320, IFA W50, UAZ-469 police, VAZ-2106 police, UAZ-452 police, ZIL-4310 MVD, PAZ-3205 police bus, ZIL-4310 civilian drop-side, **ISUZU ELF4 reefer truck** (with the reefer unit, see above).

**18 new variants**: B1000 / W50 / ZIL-4310 civilian each split into **Clothing / Appliances / Furniture / Agriculture / Industry / Coal**, each carrying only its own cargo.

### 2. Pools (4)

Civilian light 22 (incl. the **ISUZU ELF4 reefer**) · Civilian heavy 7 · Emergency light 5 (now incl. the **UAZ-452 police van**) · Emergency heavy 2.

### 3. Building spawn rules (new)

1. **Town hospital** → 1-2 ambulances (map-wide quota, parking bays first)
2. **Railway station** → 1-2 heavy civilian vehicles
3. **Vehicle-refresh buildings** → **20%** per bay; **LADA 40 / B1000 30 / Toyota 20 / W50 10**
4. **Crash sites** → **3-7 wrecks at 0-30% durability** + **0-3 emergency vehicles** (distinct models)
5. **Town roads** → **2-3 heavy emergency vehicles at 0-70% durability**
6. **Civilian buildings (guaranteed reefer)** → **exactly one** ISUZU ELF4 per savegame: the town/civilian building types the refresh rule uses, else **any civilian building** on the map, else a **road** tile. Controlled only by `[Pool] IsuzuMaxPerMatch`, **not** by `[Vehicle Refresh] Enabled`.

At most **3 vehicles per tile**; each bay/building fires once, recorded in the savegame.

### 4. On-board supplies (new)

* **LADA / Toyota**: 0-2 canned cucumber + 0-2 canned meat + 0-2 butter + 0-1 shovel + 0-1 repair kit + 0-3 cloth
* **B1000 / W50 / ZIL civilian six types**: Clothing 10-50 cloth | Appliances 5-10 + 0-4 chips | Furniture 5-20 | Agriculture 10-40 flour + 0-10 fertiliser | Industry 0-2 batteries + 0-2 lathes + 10-30 scrap + 5-10 tools | Coal 10-200
  — **W50 caps ×2, ZIL-4310 civilian caps ×5**
* **Ambulance**: medical supplies, 0-15 each
* **Fire truck**: 0-5 shovels + 0-5 pickaxes
* **Police**: 10-100 rounds of 5.45 mm loose ammo
* **Heavy emergency (town roads)**: 200-1000 rounds

### 5. Parts and text

* **Every vehicle owns its part set** (cloned parts) named after the real vehicle, each with a description — the van no longer advertises a pickup bed.
* **Engines named after the real units**: W50 → East German **IFA 4 VD 14,5/12-1 SRW**; Land Cruiser → **Toyota 2H**. Power unchanged.
* **Reefer named after the real truck**: ISUZU **4th-generation ELF** (3.5 t, built from 1984) reefer, **ISUZU 4JG2** engine, **MSB-5S** gearbox and a **DENSO** reefer set.
* **Random part durability** between *completely broken* and *slightly worn* (configurable).
* **World alignment**: the Soviet Union reads as **Isara**.

### 6. Keys

`F5` cycle-spawn (44) · `Shift+F5` back · `F4` starting force · `F7` / `F10` debug dumps.

---

## Русский

### 1. Техника: 4 → **44**

**26 базовых**: UAZ-469, ZIL-4310, UAZ-452 (фургон/медицинский), LADA-2501 ×3, «Тигр» (базовый/с ДУ), RAF-22031, пожарный ZIL-130, B1000 ×3, ZIL-49061, Land Cruiser ×2, URAL-4320, IFA W50, полицейские UAZ-469 / VAZ-2106 / UAZ-452, ZIL-4310 МВД, автобус PAZ-3205, гражданский ZIL-4310, **рефрижератор ISUZU ELF4** (с холодильной установкой, см. выше).

**18 новых вариантов**: B1000 / W50 / гражданский ZIL-4310 разделены на **шесть типов — одежда / бытовая техника / мебель / сельское хозяйство / промышленность / уголь**; каждый возит только свой груз.

### 2. Пулы (4)

Гражданские лёгкие 22 (вкл. **рефрижератор ISUZU ELF4**) · Гражданские тяжёлые 7 · Экстренные лёгкие 5 (теперь с **полицейским фургоном UAZ-452**) · Экстренные тяжёлые 2.

### 3. Появление у зданий (новое)

1. **Городская больница** → 1-2 машины скорой помощи
2. **Ж/д станция** → 1-2 тяжёлые гражданские машины
3. **Точки обновления техники** → **20 %** на место; **LADA 40 / B1000 30 / Toyota 20 / W50 10**
4. **Места аварий** → **3-7 разбитых машин (прочность 0-30 %)** + **0-3 машины экстренных служб**
5. **Городские дороги** → **2-3 тяжёлые экстренные машины (прочность 0-70 %)**
6. **Гражданские здания (гарантированный рефрижератор)** → **ровно одна** ISUZU ELF4 на сохранение: сначала городские/гражданские типы зданий из правила обновления, иначе **любое гражданское здание** на карте, иначе клетка **дороги**. Управляется только `[Pool] IsuzuMaxPerMatch`, **не** зависит от `[Vehicle Refresh] Enabled`.

Не более **3 машин на клетке**; срабатывает один раз и пишется в сохранение.

### 4. Груз (новая система)

LADA/Toyota — небольшой набор; у каждого типа B1000/W50/ZIL свой груз (**у W50 лимиты ×2, у ZIL ×5**); скорая — медикаменты по 0-15; пожарная — 0-5 лопат + 0-5 кирок; полиция — 10-100 патронов 5.45 мм; тяжёлые экстренные — 200-1000 патронов.

### 5. Детали и текст

* **У каждой машины свой набор деталей**, названных по реальной машине, с описанием.
* **Двигатели по реальным моделям**: W50 → **IFA 4 VD 14,5/12-1 SRW**; Land Cruiser → **Toyota 2H**. Мощность не менялась.
* **Рефрижератор назван по реальной машине**: ISUZU **ELF 4-го поколения** (3,5 т, с 1984 года), двигатель **ISUZU 4JG2**, КПП **MSB-5S**, агрегат **DENSO**.
* **Случайная прочность деталей** — от «полностью сломана» до «слегка изношена».
* **СССР в текстах отображается как Изара (Isara).**

### 6. Клавиши

`F5` — следующая машина (44) · `Shift+F5` — назад · `F4` — стартовые силы · `F7`/`F10` — дампы.

---

## 兼容性 / Compatibility / Совместимость

* ✅ 只含 BepInEx 与自己的插件，**不修改任何游戏原版文件**。
  Ships BepInEx and its own plugin only — **no game file is modified**. Файлы игры не изменяются.
* ✅ 插件文件名 **`FrontlineVehicles.dll`**，与旧版 `Uaz469.dll` 不是同一个文件；安装脚本会自动删除旧文件。
* ⚠️ **不能与旧版超级机械化同时安装** —— 两者共用配置标识（`com.mod.uaz469`，为了让设置继承）。
  **Do not install both** — they share the config GUID on purpose. Устанавливайте только один.
* ⚠️ 存档里若已有这些载具，**移除插件会使该存档无法载入** —— 卸载前请先在游戏内处理掉它们。

安装步骤见同目录 **`MANUAL.md`**（中英俄）。 See **`MANUAL.md`**. См. **`MANUAL.md`**.

---

## 文档 / Documentation

| 文档 | 语言 |
|---|---|
| [开发手册 Developer Guide (English)](docs/DEVELOPMENT-GUIDE.en.md) | English |
| [开发手册（中文原版）](docs/DEVELOPMENT-GUIDE.cn.md) | 中文 |

开发手册涵盖：构建流程、源码结构、**如何新增载具**、五条刷新规则、
随车物资系统、校验矩阵、贴图流水线、发布流程，以及
**原版游戏内部机制**（数据资产提取、武器/物品/编制表、战斗与营养系统）。

## 下载 / Download

安装包在 **Releases** 里，不在仓库中（避免把二进制塞进版本库）：

* **最新版 Latest — [Frontline Vehicles v5.70.6](https://github.com/maximwong/Frontline-Logistics-Isara-Warfare-Mod-/releases/tag/v5.70.6)**（`FrontlineVehicles-v5.70.6.zip`）
* **全部版本 All releases**（含可回滚的上一版 **v5.68.2**）：
  **https://github.com/maximwong/Frontline-Logistics-Isara-Warfare-Mod-/releases**

## 许可 / License

本项目以 **MIT** 授权，见 [LICENSE](LICENSE)。

⚠️ 安装包内捆绑的第三方组件遵循各自的许可：
**BepInEx** 为 LGPL-2.1，**Harmony / Mono.Cecil / MonoMod** 为 MIT。
本 MOD **不含任何游戏原版文件**。
