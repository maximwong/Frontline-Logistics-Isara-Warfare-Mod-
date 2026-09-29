# Frontline Vehicles — 开发手册 / Development Guide

适用版本 **v5.68.2** · 游戏 `FrontlineLogistics_IsaraWarfare 0.8.706`（Steam 试玩版，Unity 2021.3.20f1c1 / Mono）
最后更新：见文末变更记录

> 本文写给**接手这个 MOD 的人或 LLM**。读完应当能独立完成：
> 新增载具、改贴图、改文案、加刷新规则、改配置、构建、校验、打包发布。
> 文中所有"游戏内部事实"都经过 IL / 资产实测，附证据位置，**不要凭直觉推翻它们**。

---

## 0. 三十秒速览

| 项 | 值 |
|---|---|
| 这是什么 | 《前线后勤：伊萨拉战争》的载具 MOD（BepInEx 插件），当前 **43 台载具** + 5 类建筑刷新规则 + 随车物资系统 |
| 插件 DLL | `FrontlineVehicles.dll`（**不是** `Uaz469.dll` —— 那是它前身"超级机械化 MOD"的名字） |
| 插件 GUID | `com.mod.uaz469`（**故意保持与旧版相同**，让用户已有的 `.cfg` 设置能继承） |
| 配置 | `BepInEx\config\com.mod.uaz469.cfg` |
| 源码 | `mod\deploy\plugin-src\Uaz469Plugin.cs`（**单文件**，约 4600 行） |
| 编译器 | `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe`（**只支持 C# 5**） |
| 构建 | `mod\deploy\plugin-src\build.ps1` |
| 游戏目录 | `D:\SteamLibrary\steamapps\common\FrontlineLogistics_IsarianWarfare_Demo\FrontlineLogistics_IsaraWarfare` |

**铁律**：
1. 只支持 **C# 5**（不能用 `?.`、字符串插值、`nameof`、表达式体成员、`async`…）。
2. **绝不修改任何游戏原版文件**（当前 172 个原版文件零改动，所有文本改写都在内存里）。
3. **不要改配置默认值** —— 默认值是给新用户的保守值，调参一律走 `.cfg`（用户明确要求过）。
4. 每次构建后**必须跑完整校验矩阵**（第 8 节），一条都不能省。

---

## 1. 目录结构

```
mod\
├─ deploy\
│  ├─ plugin-src\
│  │  ├─ Uaz469Plugin.cs        ← 全部源码（单文件）
│  │  └─ build.ps1              ← 构建脚本（内嵌 24 张贴图）
│  ├─ sprites\                  ← 24 张已处理好的图集（build.ps1 从这里读）
│  └─ loader\BepInEx-5.4.23.3-x64\   ← BepInEx 发行文件（构建输出落这里）
├─ src\                         ← 未处理的原画（美术给的 PNG）
├─ tools\                       ← 全部 Python/PowerShell 工具（第 7 节）
│  └─ pylibs\                   ← Pillow 等依赖（脚本会 sys.path.insert 进来）
├─ spec\                        ← 设计与给美术的 prompt（第 11 节）
├─ out\                         ← 工具产出的报表/JSON（可随时重建）
├─ release\                     ← **发布模板**（README/MANUAL/CHANGELOG/install/uninstall）
├─ dist\                        ← 打包产物（目录 + zip）
└─ backup\                      ← 归档（如已删除的油罐车贴图）
```

**git 无关**：本仓库不是 git 仓库，靠 `backup\` 与版本号回溯。

---

## 2. 构建

```powershell
powershell -ExecutionPolicy Bypass -File 'D:\Codex Work\mod\deploy\plugin-src\build.ps1'
```

输出：`deploy\loader\BepInEx-5.4.23.3-x64\BepInEx\plugins\FrontlineVehicles.dll`
（**不是**游戏目录 —— 部署是另一步，见 2.3）

### 2.1 build.ps1 做了什么

1. **编码守卫**：校验源码是合法 UTF-8、中文常量完好、**每行 ASCII `"` 数量为偶数**。
2. 用 `csc.exe /codepage:65001` 编译 `Uaz469Plugin.cs`。
3. 把 `$sheets` 数组里的 **24 张 PNG 作为嵌入资源**编进 DLL（`/resource:`）。
4. 报告 DLL 大小。

### 2.2 ⚠️ 引号守卫（最常踩）

脚本逐行统计 ASCII 引号，**奇数即 BUILD FAILED**。两个真实踩坑：

* **注释里出现单个 ASCII `"`** → 失败。**注释里请用「」或中文引号**。
* **用 `\"` 把字符串字面量拆到两行** → 失败。宁可写长行，也别拆。

（中文引号 `“”` 不会被计入，所以中文注释里用中文引号是安全的。）

### 2.3 部署

```powershell
# 先确认游戏没在运行！
Get-Process -Name 'FrontlineLogistics*'
Copy-Item <构建产物> '<游戏目录>\BepInEx\plugins\FrontlineVehicles.dll' -Force
```

**游戏运行时复制会失败**：`The requested operation cannot be performed on a file with a user-mapped section open`。
必须先关游戏。

`BepInEx\plugins\` 里**只应有这一个 DLL**；`*.old-*` / `*.locked_*` 这类惰性备份 BepInEx 不加载，可以留着。

---

## 3. 源码结构（Uaz469Plugin.cs）

单文件，按以下顺序组织（改代码前先看这里定位）：

| 区域 | 作用 |
|---|---|
| `Plugin : BaseUnityPlugin` | BepInEx 入口，`[BepInPlugin(GUID, "Frontline Vehicles", "x.y.z")]` |
| `VehicleSpec` + `Specs[]` | **载具规格表**（每台一条）；`Specs = new VehicleSpec[43]` |
| `Initialize()` | 构建所有 spec → `BuildLightVariants()` / `BuildHeavyVariants()` → 绑定全部配置 |
| 部件改名系统 | `PartText` / `PartBase` / `ClonePartWithText` / `MirrorConfigTables` / `MirrorTextTables` |
| `InjectAll()` | 注入入口：`SanitizeText()` → 逐台 `InjectOne()` |
| `VehiclePool` + `Pools[]` | 4 个载具池 + `LogPools()` 自检 |
| 刷新规则 | 见第 5 节（5 条规则 + 共用生成器） |
| 随车物资 | `CargoEntry` / `CargoForSpec` / `GiveCargo`（第 6 节） |
| 型号生成器 | `MakeLightVariant()` / `BuildLightVariants()` / `BuildHeavyVariants()`（第 4.3 节） |
| `VehicleDriver` | 运行时 MonoBehaviour：按键、HUD、排队处理、`InjectAll()` 触发 |

### 3.1 `VehicleSpec` 字段

```
Key, SrcChassisKey, SrcArmamentId, ChassisId, ChassisKey, ArmamentId, ArmamentInt,
DisplayName, ChassisName, BaseName, Price, Passengers, Cargo,
ExtraDefaultParts, InvisibleExtraParts, Desc, PartTexts,
SheetResource, CellW, CellH, Row, Col, PivX, PivY, TestTile, ResolvedTile
```

**ID 规则（务必遵守）**：

* 底盘 ID 从 **10346** 起顺序分配；已占用：`10346–10379`、`10381–10392`。
* **`10380` 永久禁用** —— 它是 `ZIL-4310POL` 的**幽灵部件 ID**（`ChassisId + 10 = 10370 + 10`），
  再用会导致字典键碰撞。
* **幽灵部件 ID = `ChassisId + 10 + i`**（用于"看不见的额外部件"）。
* **克隆件 ID = `ChassisId * 100 + (库存件 ID % 100)`**。
* 装备 ID = `ChassisId * 100000`（字符串形式，如 `1034600000`）。
* 载具源（`SrcChassisKey` / `SrcArmamentId`）只有三族：
  * **皮卡族** `"10050"` / `"1005000000"`
  * **AA-320 族** `"10305"` / `"1030500000"`
  * **面包车族** `"10348"` / `"1034800000"`

### 3.2 注入链（`InjectOne`）

1. 注册装备数据 `ArmamentDataDic` + 镜像 `ConfigsTable` 里**所有按部件 ID 索引的表**
   （`ArmamentPartDataDic`、`ArmamentFuelTankDataDic`、`ArmamentEngineDataDic`、
   `ArmamentTurretPartDataDic`、`ArmamentChassisDataDic`）、`ItemDataDic`、`MirrorItemSets`、
   以及 `GameTextData` 里所有 `Dictionary<int,string>`。
2. `ClonePartWithText()` 克隆部件并写入改名文案。
3. 贴图从嵌入资源取出、按 `Row/Col/PivX/PivY` 切格填入 `ChassisData._8DirectionalSprites`。

**部件改名为什么必须克隆**：`PartComposition.partName` 虽然写入了但**从不被读取**（IL 交叉引用已证），
界面显示的是 `PartData.partName` —— 而 `PartData` 是按部件 ID 共享的，
所以不克隆就无法做到"每台车一套部件名"。

---

## 4. 新增一台载具（标准流程）

### 4.1 清单

1. 把原画（**PNG，带真实 alpha**）放进 `mod\src\`。
2. 跑图集流水线（第 9 节）→ 输出到 `mod\deploy\sprites\`。
3. 在 `Uaz469Plugin.cs` 里加一个 `VehicleSpec` 块，填 `Key` / ID / 名称 / 简介 / 部件文案 / 贴图名 / 格尺寸 / pivot。
4. `Specs[n] = x;` 并**把 `Specs = new VehicleSpec[N]` 的 N 加一**。
5. 在 `build.ps1` 的 `$sheets` 数组里加一行（`"名字_sheet.png"`）。
6. （可选）加进某个 `VehiclePool`。
7. 构建 → 跑校验矩阵（第 8 节）→ 重新 `extract_vehicle_table.py` → 部署。

### 4.2 最省事的做法：抄一台同族的

同族载具的 `SrcChassisKey` / `SrcArmamentId` / `CellW` / `CellH` / pivot 通常一致。
**pivot 不要凭感觉填** —— 用 `measure_pivots.py` 量，或用 `audit_pivots.py` 验。

### 4.3 型号变体（不要复制粘贴 12 个 spec 块）

B1000 / W50 / ZIL-4310CIV 的"服装/家电/家具/农业/工业/运煤"型号是**程序化生成**的：

```csharp
VehicleSpec b = SpecByKey("B1000BLUE");
MakeLightVariant(b, LightTypeSuffix[i], LightTypeNameCn[i],
                 B1000VariantChassis[i], B1000VariantSheets[i], 25 + i);
```

`MakeLightVariant` 会**照抄基础 spec 的全部外观字段**（贴图/格尺寸/pivot/简介/部件文案），
只改 `Key`、名称、底盘与装备 ID。**这样不用新美术，也绝不会抄错 pivot。**

⚠️ **代价**：这些变体对"解析字面 spec 块"的工具有盲区 —— 已通过 `tools/vehicle_specs.py`
统一解析器解决（它**同时理解字面块与生成器**）。加新型号时**必须确认解析器也认**
（跑 `python tools\vehicle_specs.py`，总数应与运行时一致）。

⚠️ **血泪教训**：加型号后一定要跑校验 —— 曾两次因为"池子里的 Key 与生成的 Key 不一致"
（`B1000-CLOTH` vs 实际生成的 `B1000BLUE-CLOTH`）导致**型号静默刷不出来**，
肉眼完全看不出，是校验器抓到的。

---

## 5. 刷新规则（五条）与生成器

### 5.1 五种规则

| 规则 | 字段前缀 | 触发建筑（可配置） | 行为 |
|---|---|---|---|
| 医院救护车 | `[Hospital Ambulance]` | `HospitalBuildings`（默认 50085,50086） | 全图 1-2 台，车位优先 |
| 火车站重型民用 | `[Train Station Truck]` | `StationBuildings`（50070） | 全图 1-2 台 |
| 车辆刷新 → 轻型民用 | `[Vehicle Refresh]` | `RefreshBuildings`（13 类城镇建筑+街边停车场） | **每车位 20%**，池内 40/30/20/10 |
| 事件点 | `[Crash Site]` | `CrashBuildings`（50040） | **每个事件点** 3-7 台事故车（耐久 0-30%）+ 0-3 台应急救援 |
| 城市道路 → 重型应急 | `[City Road]` | `CityRoadBuildings`（同上 13 类） | **全图 1-2-3 台**（一次掷配额） |

### 5.2 架构：**排队 → 延迟处理**

```
BuildingInfo.AfterStart  (Harmony 后缀 BuildingSpawnPostfix)
        ↓  只做轻量判断，把建筑塞进对应队列
   PendingSpawns / PendingRefresh / PendingCrash / PendingCityRoad
        ↓  每帧由 VehicleDriver.Update 调 ProcessPendingSpawns()
   真正生成（此时装备表已注入、地图已就绪）
```

**为什么必须这样**：`AfterStart` 跑在 Unity `Start` 阶段，**可能早于本 MOD 注入装备表**，
当场生成会拿到空数据。队列非空就每帧重试，直到条件满足。

⚠️ **早退守卫要检查所有队列**：
```csharp
if (PendingSpawns.Count == 0 && PendingRefresh.Count == 0 &&
    PendingCrash.Count == 0 && PendingCityRoad.Count == 0) return;
```
（曾经只检查 `PendingSpawns`，导致"地图上没有医院/火车站时，城市道路规则永不执行"。）

### 5.3 生成一台车的正确姿势

```csharp
// 1) 根锚点 GameObject（必须是根对象，否则 DataManager.InitializeUnit() 会在
//    child.GetComponent<BaseUnitAi>() 上 NRE）
GameObject anchor = new GameObject("spawn");
anchor.transform.position = worldPos;
// 2) 4 参重载 —— 绝不要用 6 参（见 5.4）
BaseArmamentInfo info = dm.SpwanArms(armamentId, TypeManager.CampType.BCamp, anchor.transform, null);
// 3) 挂到单位容器下，销毁锚点
info.transform.SetParent(dm.unitContainer, true);
Destroy(anchor);
// 4) 显式登记地块（原版那段被门控掉了）
info.baseArmamentAi.SetUnitOnTile(tile);
```

### 5.4 ⚠️ 本作最关键的三个"陷阱"

**① `DataManager.FullReleaseMark()` 硬编码返回 false**

```
DataManager::FullReleaseMark() -> System.Boolean
     0 ldc.i4.0
     1 ret
```

**6 参 `SpwanArms` 的整个后处理块第一步就是它**：

```
190 call  DataManager::FullReleaseMark
195 brfalse IL_013d          ← 为 false 直接跳过整块
```

后果：**原版建筑刷车的那条路在本作里是死的** —— 不会随机耐久、不会 `SetUnitOnTile`、
不会随机涂装、不重算初始价值。**所以：**

* **永远不要用 6 参 `SpwanArms` 生成载具**；
* 耐久随机、燃料、朝向、地块登记**全部要自己来**。

原版"车祸残骸（0-50% 耐久）"因此在本作里根本不生效 —— 这就是我们要自建全套的原因。

**② `partList` 在 `SpwanArms` 返回时还是空的**

`partList` 由 `BaseArmamentInfo.Initialization()` 在 **Start 阶段**构建。
所以生成后立刻改部件耐久**什么也不会发生**（曾导致"刷出来的车部件全是新的"）。

正确做法：**协程等 `armInitialized == true` 再动手**（不要用 `partList.Count > 0` 判断 ——
部件是逐个加进列表的，可能只装了一半）：

```csharp
for (int i = 0; i < 300 && info != null && !info.armInitialized; i++) yield return null;
```

`DeferredSetup()` 就是这个统一收口，它负责：**耐久随机 → 燃料 → 朝向 → 装货**。

**③ 朝向由公有属性控制**

```
BaseUnitAi::set_UnitDirection @IL_0059  →  BaseArmamentAi::SetChassisSprite(InGameDirection)
```

停着的车没人下移动命令，朝向永远是默认 `Up(0)` —— 所以**所有车都朝一个方向**。
随机朝向要显式设（且必须在 `SetUnitOnTile` **之后**，因为它自己也会调 `SetChassisSprite`）：

```csharp
info.baseArmamentAi.UnitDirection = (lgBase.DirectionDetector.InGameDirection)Random.Range(0, 8);
```

枚举：`Up=0, TopRight=1, Right=2, BottomRight=3, Down=4, BottomLeft=5, Left=6, TopLeft=7, None=8`。

### 5.5 放置：找空闲地块

```csharp
FindFreeTileNear(building, usedSet, roadOnly)
```

* `BuildingInfo.tileList` 是**私有字段，本程序集读不到** → 以 `buildingOnTile.gridLocation` 为中心**逐圈向外扫**（半径 1→6）。
* 判据：`t.BuildingOnTile == null`（**没有建筑就说明在建筑外面**）、`TileHasRoom(t)`、按需 `tilePathDataGroupDic.Count > 0`（≈道路）。
* **绝不要把车放在建筑自己的地块上** —— 曾经这样做，结果车停在屋顶上（用户实测）。
* **同格最多 3 台**：`TileStack` 计数 + `unitsOnTileList.Count`；`MarkTile()` 返回"本次是该格第几台"，据此加 `StackOffset()` 让贴图不重合。
* 找不到位置时**打警告并停止**，不要回退到建筑地块。

### 5.6 全图配额 vs 按建筑计数（**极易搞错**）

用户要的"N 台"**几乎总是指整张地图**。按建筑计数会导致"上百辆"（已被用户实测抓到两次）。

```csharp
// 全图只掷一次配额，各建筑依次消耗，用完即止
int left = Random.Range(min, max + 1);
for (...) { if (left <= 0) break; left -= SpawnOne(..., left); }
```

* **只有事件点规则**是**按点计数**（用户明确要求）。
* 其余规则全部是全图配额。

### 5.7 存档记账（**硬性要求 —— 缺了必出 BUG**）

游戏用 `MapManager.mapArmsSpawnActivedList`（`Dictionary<Vector3Int, string>`）记录"这栋楼 / 这个车位刷过了没"，
**这个字典会进存档**（`WorldLowDynamicData` → `PerformSaveGameAsync` → `TilemapGenerator.GenerateMapDetail`）。

> ⚠️ **任何生成规则写完都必须写它。** 缺了就会：
> 每次读档 → 建筑 `AfterStart` 重跑 → 规则再执行一遍 → **又刷一批** ✗

**v5.67.0 修过这个 BUG**：城市道路 / 车祸点 / 车辆刷新点三条规则当初漏了记账，
表现为"**每次加载存档，重型应急载具越来越多**"（用户实测）。

键位号段分配（避免规则之间互相覆盖）：

| 规则 | 键 | 语义 |
|---|---|---|
| 医院 / 火车站 | `(x, y, 500+i)` / `(x, y, 590+n)` | 车位 / 外圈兜底位 |
| 车祸点 | `(x, y, 700)` | 该事件点已刷 |
| **城市道路** | **`(-1, -1, 800)`（哨兵键）** | **本存档该规则已执行** → 全图配额变成"一存档一次" |
| 车辆刷新点 | `(x, y, 900+i)` | 该车位已刷 |

**四条要点**：

1. **`TileStack` / `GlobalSpawnTiles` 是内存静态集合，不进存档** —— 只能防同一局内重复，
   **拦不住读档重刷**，不能替代 `mapArmsSpawnActivedList`。
2. **全图配额类规则必须用哨兵键** —— 否则配额每次读档重掷，照样累积（城市道路就是这么修的）。
3. `(-1, -1, …)` 作哨兵是安全的：地图上不存在负坐标，绝不会与真实地块撞键。
4. **修复不能回收存档里已经多出来的车**，只保证以后不再增加。

---

## 6. 随车物资系统

`CargoEntry { Label; Min; Max; int[] ItemIds; }` —— 一组**同类物品**共用一份数量。

```csharp
GiveCargo(info, tag, table, mult)   // 每项：先掷总数 Min..Max*mult，再逐件随机分配给组内物品
```

* **多物品类语义**：先掷总数、再逐件随机分配（用户选定）——如"5-20 家具"分给木桌/木椅/木床。
* **倍数 `mult` 只作用于上限**（下限不变）：W50 **×2**、ZIL-4310CIV **×5**，
  由 `CargoForSpec(key, out mult)` 按 Key 前缀判断。
* **必须按实际装载量回报**：`GiveItem()` 会受载重夹紧，只信它返回的值（不要信掷出的数）。

### 6.1 物品 ID（实测，别猜）

| 名称 | ID | 备注 |
|---|---|---|
| 5.45mm子弹(散装) | **10001** | 散装，需压制才能用 |
| 止血包 / 血浆 / **抗感染药** / 吗啡 / 麻醉剂 / 手术用具 | 10007 / 10008 / **10009** / 10010 / 10011 / 10033 | 10009 名字是占位符 `To be iterated`，**用户已要求从刷新清单移除** |
| 发动机/轻武器/重武器/油箱系统/轮式机构 通用零件 | 10053–10057 | |
| 机械零件 / 机械润滑油 | 10058 / 10059 | |
| **烟雾弹** | **10060** | |
| 废金属(金属废料) | 10071 | |
| 芯片 / 布料 | 10077 / 10079 | |
| 车床 / 大型家电 / 小型家电 | 10082 / 10083 / 10084 | |
| 黄瓜罐头 / 肉罐头 | 10087 / 10088 | |
| 化肥 / 蓄电池 | 10093 / 10134 | |
| 快速修理包 | 10118 | |
| 铁镐 / 铲子 | 10147 / 10149 | |
| 煤块 | 10154 | |
| 黄油 | 10212 | |
| 小麦面粉 / 黑麦面粉 | 10214 / 10215 | |
| 履带式机构通用零件 | 10190 | |
| 破旧的木桌/木椅/木床（"家具"） | 10029 / 10030 / 10078 | 表里**没有**单一"家具"物品 |
| 汽油 / 柴油 / 航空煤油（货物形态） | 10044 / 10045 / 10326 | |

**全表没有任何物品叫"工具"** —— 用户指定用**通用零件家族**（10053-10058、10190）代替。
全表唯一含"工具"的是 `70016 修复手术(缺乏工具)`，那是**动作提示**不是物品。

---

## 7. 工具链

### 7.1 核心解析器（**所有校验工具的基础**）

`tools\vehicle_specs.py` —— 统一解析 `Uaz469Plugin.cs`，**同时理解字面 spec 块与型号生成器**，
输出与实际运行时一致的完整载具表。改动载具后先跑它自检：

```powershell
python tools\vehicle_specs.py      # 应打印 43 台（base + variants）
```

### 7.2 校验工具

| 工具 | 验什么 |
|---|---|
| `verify_dll.py` | DLL 里每张图集与 `deploy\sprites\` **逐字节一致**；版本串；`unexplained == 0` |
| `extract_vehicle_table.py` | 生成 `out\vehicle-table.json`；报总数/重复 ID/重复槽位/唯一贴图数 |
| `verify_part_text.py` | 派生部件 ID 撞号、与库存 ID 冲突、改名文案对应的部件是否真的在车上 |
| `verify_pools.py` | 池里 Key 是否存在（**拼错会静默少车**）、是否重复归池、谁没归池 |
| `check_grid.py` | 八方向是否 1:1 落在规范格位 |
| `audit_pivots.py` | pivot 是否合理 |
| `check_dll_text.py` / `find_strings.py` | DLL 里某些字符串是否存在 |
| `il-query.ps1` | IL 查询：`-Fields` `-Methods` `-Xref`（方法**和字段**）`-Call` `-New` `-Find` `-Str` |
| `il-dump.ps1` | 反编译指定方法/类型的 IL |

### 7.3 图集工具

见第 9 节。

### 7.4 打包工具

见第 10 节。

---

## 8. 校验矩阵（每次构建后**必须**全跑）

```powershell
cd 'D:\Codex Work\mod\tools'
python extract_vehicle_table.py     # 总数 / 重复 ID / 重复槽位 / 唯一贴图
python verify_part_text.py          # 派生部件 ID 0 撞号
python verify_pools.py              # 池 0 问题
python verify_dll.py                # RESULT: PASS
python check_grid.py                # PASS
python audit_pivots.py              # 0 issues
```

**每一条都要看输出，不能只看退出码。**

### 8.1 已知盲区与历史教训

* 早期 `verify_dll.py` 的期望清单来自 `out\vehicle-table.json` —— 加了车但忘了重新生成，
  会出现 `expected 23 / embedded 24 / unexplained 1` 却**仍然 PASS**。
  现在 `unexplained` 按哈希算，并要求为 0。
* 早期校验工具**只认字面 spec 块**，18 个型号变体对它们完全隐形 → 已用 `vehicle_specs.py` 解决。
  **加新型号后必须确认解析器认得。**
* `lost_content.py` 给过误导数字（1652 vs 实际 37）；**信 `cell_counts.py` 的逐格像素数**。
* `count_specks.py` 对白色车身误报；**用 `count_islands.py`**。

### 8.2 控制台编码陷阱

Python 输出经 PowerShell 管道会被 **GBK 二次编码**（中文全变乱码）。
**正确做法：让 Python 自己用 `io.open(..., encoding='utf-8')` 写文件，再用
`Get-Content -Encoding UTF8` 读。**

---

## 9. 贴图流水线（顺序不可换）

```powershell
cd 'D:\Codex Work\mod\tools'
python prepare_sheet.py   <原图>                        # 抠底
python refix_sheet.py     <上一步输出> --perm standard   # 重排成引擎字段顺序
python fringe_flood.py    <上一步输出>                   # 清描边外残留
python measure_pivots.py  <上一步输出>                   # 量 pivot（**必须最后**）
```

⚠️ `prepare_sheet.py` 会改变包围盒，所以 **pivot 必须最后量**。

### 9.1 顺序与朝向依据

用户给的图集是**旧绘制顺序** → 用 `--perm standard`，但**必须验证**：
`refix_sheet.py` 自带守卫（`CLASS_OF_DIR`，按**方向名**而非字段名），
再独立复核一遍（包围盒分类比较 + 目视条带）。

字段顺序：`Up, TopRight, Right, BottomRight, Down, BottomLeft, Left, TopLeft`。
字段下标 `k` 服务世界朝向 `45°×(k+1)`（0°=北，顺时针）。

规范格位：
```
GridRow = { 0,0,1,2,2,2,1,0 }
GridCol = { 1,2,2,2,1,0,0,0 }
```
`(0,0)=N, (0,1)=NE, (0,2)=E, (1,0)=NW, (1,2)=SE, (2,0)=W, (2,1)=SW, (2,2)=S`
（窄视图在 (0,0)/(2,2)，侧面在 (0,2)/(2,0)，斜向在其余位置。）

### 9.2 验收（**不能只看包围盒**）

```powershell
python cell_counts.py   <图集>    # 逐格像素数
python count_islands.py <图集>    # 碎块数
python dark_preview.py  <图集>    # 出深色预览图目视
```

### 9.3 生成器怪癖（会稳定复现）

* **ZIL-4310 派生的图集**：两个纯侧视图在格内**比其他六个高 6-7 px**（内容 y=6..25 或 5..23，其余 y=1..30）。
  → 这类图集**必须逐方向 pivot**，例如 `ZIL-4310CIV: PivX {22,21.5,21.5,22,21.5,21.5,22,22}` / `PivY {1,8,2,1,2,8,1,1}`。
* 无损白底 PNG 是**完全不透明**、含 4000–5700 个噪声色；`--white 238` 安全（内部不吃掉亮像素），
  之后 `fringe_flood.py` 会再清 0–40 px。
* **一定要 PNG 带真实 alpha**。聊天软件转 JPEG 会把透明区变白底噪声，抠底时连主体边缘一起吃掉 —— 这个坑踩过很多次。

---

## 10. 发布

### 10.1 流程

1. 更新 `mod\release\` 里的模板：`README.md`（三语新增内容介绍）、`MANUAL.md`（三语说明书）、
   `CHANGELOG.md`、`install.ps1`、`uninstall.ps1`。
2. 打包：
   ```powershell
   cd tools
   python make_release.py FrontlineVehicles-v5.63.1
   ```
   产出 `dist\FrontlineVehicles-v5.63.1\` 与 `dist\FrontlineVehicles-v5.63.1.zip`。

### 10.2 ⚠️ `make_release.py` 的坑

* 它会 **`shutil.rmtree(dist\<PKG>)` 重建目录** —— **不要在 `dist\` 里写文档**，
  写了会被删掉（我犯过这个错，两份文档没进包）。
* 文档只从 `release\` 复制，且**只认一个固定清单**：
  ```python
  for name in ('README.md', 'MANUAL.md', 'CHANGELOG.md', 'install.ps1', 'uninstall.ps1'):
  ```
  **新增文档类型必须同时改这一行。**
* 它有 `FORBIDDEN = ('Uaz469.dll',)` 检查 —— 包内出现旧插件名直接中止。
* zip 用正斜杠路径（`.NET ZipFile.CreateFromDirectory` 写反斜杠，非标准）。

### 10.3 兼容性保证（用户明确要求"不和已安装的 MOD 冲突"）

* 包内**只有 BepInEx 与自己的插件**，不含任何游戏原版文件。
* 插件名 `FrontlineVehicles.dll` ≠ 旧版 `Uaz469.dll`；安装脚本**会删除旧 DLL**。
* **两者不能同时安装** —— 共用配置 GUID `com.mod.uaz469`（故意，为了继承设置）。
  这一点必须在三语文档里写明。
* ⚠️ **存档里若有这些载具，删除插件会导致该存档无法载入** —— 卸载前先在游戏内处理掉。

### 10.4 发布前检查

```powershell
# 包内 DLL 是否与线上一致
(Get-FileHash dist\<PKG>\BepInEx\plugins\FrontlineVehicles.dll).Hash -eq (Get-FileHash <游戏目录>\...\FrontlineVehicles.dll).Hash
# 包内是否有旧插件名
Get-ChildItem dist\<PKG> -Recurse -Filter 'Uaz469.dll'
```

---

## 11. 文档规范

| 文件 | 内容 |
|---|---|
| `spec\sheet-pipeline.md` | 图集处理流水线与验收标准（最详细） |
| `spec\vehicle-pools.md` | 4 个载具池定义 |
| `spec\vanilla-vehicle-spawn-rules.md` | 原版刷新机制（含 IL 证据） |
| `spec\newgame-spawn-rules.md` | 本 MOD 的刷新规则设计 |
| `spec\vehicle-text-overrides.md` | 文案改写记录 |
| `spec\*-art-prompt.md` | **给生图模型的 prompt**（每台车一份） |
| `spec\<车>.spec.md` | 单车设计规格 |
| `spec\valena-art-prompt.md` | 角色 prompt（人物而非载具，尺寸=立起的面包车 96×132） |

### 11.1 绘图 prompt 的统一结构

1. 用途说明（中英）
2. **硬性规格表**（网格 3×3 / 透明背景 / 等距俯视 2:1 / 描边 #1E2019 / 2-3 阶平涂 / 主体占格宽 60-75%）
3. **九宫格位置表**（含"朝北 = 背对观察者"这类方向说明 + **中格必须透明**）
4. **英文主 prompt**（可直接复制）
5. **负面 prompt**
6. **后处理流水线** + 交付要求（PNG 真 alpha）
7. 给生图模型的救急提示（九宫格崩坏 → 分格生成再拼）

### 11.2 文案规则（游戏文本）

* 本地化是 `FrontlineLogistics_Data\StreamingAssets` 下的**纯 CSV**：
  `ObjectIdDic.csv` / `DescriptionDic.csv` / `ContentIdDic.csv`，
  列：`ID, TAG, DESCRIPTION, English, SimplifiedChinese, TraditionalChinese, Japanese, Russian, ...`
* 文本必须写进 `GameTextData.DescriptionDic`（会**追加**在自动生成的数值行之后）；
  `TotalDescriptionDic` 会**替换整个面板** —— 别用错。
* 游戏程序集里**没有任何 `\n` → 换行的转换**（已用 `-Str '\n'` 验证 0 命中）→ **简介写成单段**。
* **世界观**：文本里的"苏联"必须显示为"伊萨拉" —— `SanitizeText()` 在 `InjectAll()` 开头
  统一处理所有 `GameTextData` 字符串字典（苏联→伊萨拉、蘇聯→伊薩拉、俄罗斯/俄羅斯→伊萨拉/伊薩拉）。
* 发动机改名**只授权给**：W50 → 东德 IFA 4 VD 14,5/12-1 SRW（**Nordhausen 厂**，不是 Schönebeck）；
  丰田陆巡 → 丰田 2H（3980cc，12 气门 OHV 间接喷射，1989 年末前使用）。
  **功率数值一律不改。**
* 多色载具（LADA×3、B1000×3、丰田×2）的**名称里不带颜色**（用户要求），
  但 `DisplayName`/`ChassisName`/`BaseName` 三个属性要一起改。

---

## 12. 游戏内部知识（速查）

### 12.1 命名空间陷阱

* **需要 `lgBase.` 前缀**：`BuildingInfo`、`BaseUnitInfo`、`MapManager`、`BaseArmamentInfo`、`BaseUnitAi`、`RangeFinder`、`FOB`、`BaseArea`、`OverlayTile`、`ArmamentPartInfo`、`DirectionDetector`
* **全局（不加前缀）**：`ChassisData`、`PartData`、`ItemStockData`、`ArmamentData`、`ConfigsTable`、`DataManager`、`GameTextData`、`TypeManager`、`ParkingSpot`、`VehicleInternalSpaceUI`
* `TypeManager` 是全局：`TypeManager.CampType`、`TypeManager.ArmamentPartType.Chassis`

### 12.2 `OverlayTile` 常用字段

```
gridLocation (Vector2Int)      tileWorldPosition (Vector3)
_buildingOnTile → BuildingOnTile (BuildingInfo)   ← 有建筑就不能放车
unitsOnTileList (List<BaseUnitInfo>)              ← 已站单位
tilePathDataGroupDic (Dictionary<int,TilePathData>) ← 非空 ≈ 可通行/道路
areaOnTile / theFob / inFobRange / medicalInfo / mineData
```
⚠️ **`BuildingInfo.tileList` 是私有字段，外部读不到。**

### 12.3 燃料

* `BaseArmamentInfo.currentFuel`（公有读写 float）、`MaxFuelCapacity`（公有只读，
  装配时按油箱部件算出，**`armInitialized` 之前是 0**）。
* 燃料物品：汽油 `10044`、柴油 `10045`、航空煤油 `10326`；
  `ItemType.Fuel = 1024`、`SubItemType.FuelOil = 512`；
  `TypeManager.FuelType { Petrol=0, Diesel=1, AviationKerosene=2 }`。
* 大宗液体存在基地层：`FOB.fuelTankDic` / `DataManager.FuelTankDic`（`Dictionary<FuelType,float>`）。
* ⚠️ `BaseUnitInfo.GetFuel(variable, itemID)` **从不减少物品数量**（会产生幽灵数量），
  且只在 `itemID == EngineData.fuelTypeID` 时加油 —— 历史上油罐车 BUG 的根因之一。

### 12.4 载重与货物

* `BaseUnitInfo.ItemTransferAndUpdateFreeItemDic(itemID, variable, showTips, pickEmplacedWeapon, itemDatas)`
  → 写入"随车自由物资"。三条无操作路径都返回 **`null`**（= 拒绝），
  **返回空 `List<>` 是错的**（调用方会理解为"全部消耗"）。
* `BaseUnitAi.AdjustLoadAmount<T>(target, itemID, inputAmount) -> int` 是**真正的"能装多少"闸门**。
  ⚠️ **Harmony 无法给开放泛型定义打补丁** —— 必须构造封闭实例：
  `open.MakeGenericMethod(typeof(lgBase.BaseUnitInfo))`。

### 12.5 原版刷新机制（供对照）

* `ConfigsTable.CollectionConfigDic` 是 `Dictionary<String, List<CollectionData>>`，按 `BuildingTypeID.ToString()` 索引。
* `BuildingInfo::AfterStart` **按车位**掷：
  `weight > 0` 的条目（listA）每车位**加权抽一条**再掷 `dropProbability`；
  `weight == 0` 的（listB）**各自独立**掷。
* 阵营硬编码 `CampType.BCamp`；需要 `CheckTileClear(0)` 通过；跳过内部车位；主基地跳过。
* 原版**只**给两类建筑挂了载具：`50040 事件点`（破损皮卡 10% + ISZ-695N 公交车 1%）、
  以及"车辆刷新"那 13 类（破损皮卡 10%）。

### 12.6 建筑 ID（实测）

| ID | 名称 | 用途 |
|---|---|---|
| **50040** | **事件点 (Event Location)** | 原版残骸刷在这里。**注意：不是"车祸点"** |
| 50041 | 事件点 | **`DESCRIPTION` 列写着「未启用」，任何刷新表都不引用 → 不要用** |
| 90001 | **车祸现场 (Car Accident Scene)** | 带简介的场景对象，**未确认是否为 `BuildingInfo`** |
| 50100 | 坠机现场 (Crash Site) | |
| 50045 / 50055 | 空投物资 / 清理废墟 | |
| 50085 / 50086 | 城镇诊所 / 传染病门诊 | 医院规则锚点 |
| 50070 | 城镇火车站主站房 | 火车站规则锚点 |
| 50089 / 50092 | 街边停车场 | 属"车辆刷新"那批 |
| 车辆刷新 13 类 | 50006 木制村屋 / 50017 老式砖砌公寓楼 / 50018 砖砌车库 / 50019 砖砌厂房 / 50021 村镇商店 / 50024 大型棚屋 / 50029 砖砌住宅 / 50030 村镇学校 / 50034 小型砖砌库房 / 50035 砖砌工厂 / 50036 木制仓库 / 50089 街边停车场 / 50092 街边停车场 | |

---

## 13. 已知限制与待办

* **F5 循环有 43 台**，转一圈要按 43 次。
* **事件点规则按点计数**（用户明确要求，不加全图配额）。
* **程序化生成的 18 个型号**：靠 `vehicle_specs.py` 覆盖校验，**新增型号时必须确认解析器认**。
* **发布包版本落后**是常见状态 —— 改完代码记得重打包（第 10 节）。
* 待确认：`90001 车祸现场` 是否可作为 `AfterStart` 触发点（若不是 `BuildingInfo`，需换挂载思路）。
* 插件目录里可能残留 `*.old-*` / `*.locked_*` 惰性备份（BepInEx 不加载，可清理）。

---

## 14. 给接手者（人或 LLM）的操作约定

1. **改代码前先跑一遍校验矩阵**，拿到基线。
2. **改完立刻构建 + 全跑校验矩阵**，任何一条不绿都不要部署。
3. **PowerShell 陷阱**：
   * 单引号字符串里的 `` `r`n `` **不会展开** —— 会作为字面量写进源码（编译报 `CS1056 意外的字符`）。
     需要真实换行时用双引号，或直接用 `write` 工具写文件。
   * **正则匹配要先确认括号数量** —— 例如 `DeferredSetup(..., medical));` 是**两个**右括号，
     只写一个会静默不匹配（改完必须验证替换是否真的生效，用 `Select-String` 读**磁盘**内容）。
   * `Select-String -Path` 读的是**磁盘**，若与 `WriteAllText` 同处一个脚本，注意执行顺序 ——
     打印可能仍是旧内容。
4. **改配置默认值前先问** —— 用户明确要求"不要改默认区间"，调参走 `.cfg`。
5. **`BepInEx` 只在退出时写配置** —— 游戏运行中改 `.cfg` 会被退出时覆盖回去。
6. **部署前必须确认游戏已关闭**。
7. **用中文汇报，承认错误要直接** —— 用户风格简短、重视实机验证，每次错误都要明确说明。
8. **不要凭直觉编造游戏内部事实** —— 用 `il-query.ps1` / CSV 表 / 资产提取去查证，并把证据写进注释。

---

## 15. 数据资产提取（原版数据 → JSON）

改数值、查机制、做对比表都靠这一套。**UnityPy 已随 `tools/pylibs` 提供**，脚本里
`sys.path.insert(0, r'D:\Codex Work\mod\tools\pylibs')` 即可。

### 15.1 基本流程

```
① 在 globalgamemanagers.assets 里遍历 MonoScript，用 m_ClassName 找到目标类 → 拿到它的 path_id
   （MonoScript 的 read_typetree() **是可靠的**，与 MonoBehaviour 不同）
② 遍历 Data 目录下所有 *.assets，对每个 MonoBehaviour 取 get_raw_data()，
   读偏移 20 处的 int32（= MonoScript 的 path_id）匹配 → 命中即目标实例
③ 手工偏移解析字段（见 15.3），或先试 read_typetree()
```

### 15.2 scriptID 对照表（实测）

| path_id | 类名 | 内容 | 产出 |
|---|---|---|---|
| **838** | `CollectionConfigs` | 建筑 / 小队刷新表（27 条） | `out/collectionconfigs.json` |
| **509** | `WeaponConfigs` | **武器表**：8 个资产共 **71 条** | `out/weapons_full.json` |
| **709** | `FoodItemConfigs` | **物品 / 弹药**：3 个资产共 **103 条** | `out/items.json` |
| **315** | `ArmamentItemConfigs` | **武器物品**：94 + 11 = **105 条** | `out/armament_items.json` |
| **1374** | `BattleMemberConfigs` | **士兵编制**：17 子资产共 **58 条** | `out/members.json` |

**`WeaponConfigs` 的 8 个资产**（都叫"XX Config"，按用途分）：

| 资产名 | 条目 | 用途 |
|---|---|---|
| `GunsConfig` | 10 | **步兵枪械**（步枪 / 机枪 / 狙击 / 喷火） |
| `ExplosiveIndividualWeaponsConfig` | 11 | **单兵爆炸武器**（手榴弹 / 火箭筒 / 榴弹发射器） |
| `EmplacedWeaponConfig` | 10 | **架设武器**（通用机枪 / 高射机枪 / 反坦克导弹架） |
| `ArmamentsConfig` | 17 | **车载 / 火炮** |
| `AirStrike` | 14 | **航空打击** |
| `InteralDataWeapon` | 8 | **内部数据**（多弹种基型、烟幕、炮火支援） |
| `MeleeCombatConfig` | 1 | **近战**（只有「搏斗」一条） |
| `MineWeaponConfig` | 0 | 地雷（空） |

**`BattleMemberConfigs` 的 17 个子资产**（按兵种分）：`Rifleman(4)` `Grenadier(4)` `Machinegunner(4)`
`Sniper(4)` `Assault(2)` `Scout(3)` `CombatEngineer(5)` `SpecialForcesOperator(4)` `WeaponOperator(4)`
`SecondLineTroops(9)` `NonCombatant(7)` `Animal(8)` + 若干空资产（合计 58 条；camp=1/2 各一份镜像）

### 15.3 MonoBehaviour 头与序列化规则（**必须记住**）

```
偏移  0–15   m_GameObject PPtr
偏移 20      scriptID (int32) = MonoScript 的 path_id   ← 用它匹配
偏移 28      m_Name 长度 (int32)
偏移 32      m_Name 内容（UTF-8），之后 4 字节对齐
偏移 ?       序列化字段开始（声明顺序）
```

| 类型 | 字节 |
|---|---|
| `string` | int32 长度 + UTF-8 字节，**尾部 4 字节对齐** |
| `int` / `enum` / `float` / `bool` | **各 4 字节**（bool 占 4 字节！不是 1） |
| `PPtr<T>`（资源引用） | int32 fileID + int64 pathID = **12 字节** |
| `T[]` / `List<T>` | int32 数量 + 数量×元素，尾部 4 字节对齐 |

**嵌套结构（实测）**：

```csharp
FireNum            { int fireNumMax; int fireNumMin; }                    // 8 字节
DamageSourceData   { int weight; DamageType damageType; }                 // 8 字节
EquipmentData      { string itemName; int itemID; int itemNum; int useOrder;
                     List<WeaponType> replaceableTypes; }                  // List<enum> = int32 数量 + int32×n
```

### 15.4 扫描法解析（**处理"未知嵌套结构"的关键技巧**）

`WeaponData` 有 39 个字段，其中 `damageSources` / `fireRateArray` / `fireNumArray` 是嵌套数组，
**布局未知时无法算下一条的起点**。我在这上面失败过两次（只解 15 个字段就去读下一条 ✗）。

**正确做法**：

1. **只解析已在高置信度下验证过的前 N 个字段**（`WeaponData` 是前 15 个）；
2. 然后**从当前偏移向前逐字节扫描**，寻找下一条的起点，判据全部满足才算命中：
   * `rd_str()` 成功（长度合理 + 全可见字符）
   * 紧随其后的 int32 是**合理范围内的 `weaponID`**（`0 < id < 200000`）
   * `weaponType` 落在枚举范围内（`0..17`）
   * 名字长度 ≥ 2
3. 命中则继续，直到**解析数量 == 资产头里声明的条目数**（自校验 ✓）。

这个方法的**判据是"解析数量与声明数量一致"** —— 数量对得上就说明整条链没错位 ✓

### 15.5 五个陷阱（都是实际踩过的）

1. **`read_typetree()` 对 MonoBehaviour 经常失败**（Mono 资产的类型树不内嵌）——
   我第一次静默 `except: continue`，导致"未找到资产"这个**错误结论**。
   **要么打印异常，要么直接走手工偏移解析。**
2. **空字符串是合法的** —— `FoodItemConfigs` 的每条 `m_Name` 都是空字符串，
   我最初把 `len <= 0` 当非法并 `raise`，**整表 0 条** ✗。`rd_str` 必须允许 `n == 0`。
3. **整数别按 float 读** —— `damageSources` 的 `weight`（int）我按 float 解，
   得到 `1.4e-43` 这种畸形值。**按声明的类型读**。
4. **不要假设资产在哪个文件里** —— 遍历 `Data` 目录下**全部** `.assets`；
   资产也可能在 bundle 里（那时需要换方法）。
5. **GBK 控制台会毁掉中文** —— Python 的 `print` 经 PowerShell 管道会二次编码乱码。
   **让 Python 自己用 `io.open(..., encoding='utf-8')` 写文件，再 `Get-Content -Encoding UTF8` 读。**

### 15.6 `GameGlobalVariables` —— 所有数值常数的所在地

**v0.8.706 的全局常量全部写在 `GameGlobalVariables::.cctor`（静态构造函数）里的字面量**，
例如：

```
3314 ldc.r4  30
     stsfld GameGlobalVariables::<SecondsForEat>k__BackingField
```

**⚠️ 教训：`il-query.ps1 -Xref` 不索引 `stsfld` / `stfld` 写入。**
我曾因 `-Xref Calorie_Sufficient` 返回 0 命中，就错误地下结论"数值在数据资产里、需要提取资源" ✗。
**遇到"xref 0 命中"时，要反编译 `.cctor` 或初始化方法，而不是断定代码里没有。**

---

## 16. 原版战斗数据速查

### 16.1 武器类型 `TypeManager.WeaponType`（19 项，实测）

| 值 | 名 | 值 | 名 |
|---|---|---|---|
| 0 | Normal | 10 | Mine |
| 1 | **Grenade**（投掷手榴弹，只有 2 条） | 11 | **MG** |
| 2 | **ArtilleryShell** | 12 | **AssaultRifle** |
| 3 | **AP** | 13 | **SniperRifle** |
| 4 | **Belt**（弹链） | 14 | **Flame** |
| 5 | **HEAT** | 15 | Hypocenter |
| 6 | MISS | 16 | **Thermobaric** |
| 7 | None | 17 | **cqbWeapon** |
| 8 | **MeleeCombat**（近战，只有「搏斗」） | | |
| 9 | **RPG** | | |

### 16.2 `WeaponData` 字段（**声明顺序，39 个**）

```
1  weaponName(String)        2  weaponID(Int32)          3  manufacturer(enum)
4  weaponType(enum)          5  isVehicleWeapon(Bool)    6  isMultiAmmoWeapon(Bool)
7  parentWeapon(Int32)       8  mountCount(Int32)        9  mountWeapon(Int32)
10 isGuideWeapon(Bool)       11 bulletID(Int32)          12 extraCost(Int32)
13 isLock(Bool)              14 isPersonalWeapon(Bool)   15 bulletName(String)
16 horizontalSpeed(Float)    17 verticalSpeed(Float)     18 weaponAccuracy(Float)
19 staticAccuracy(Float)     20 objectDamage(Float)      21 objectDamageMin(Float)
22 sceneDamage(Float)        23 attackCount(Int32)       24 damageSources(DamageSourceData[])
25 fireRate(Float)           26 fireRateReadOnly(Float)  27 fireRateArray(Single[])
28 fireNumArray(FireNum[])   29 maxMagazineCapacity(Int32) 30 reloadTime(Float)
31 isAutoReload(Bool)        32 penetrationEffect(Float) 33 weaponRangeMax(Int32)
34 weaponRangeMin(Int32)     35 aimingTime(Float)        36 reactionTime(Float)
37 explosionRange(Float)     38 shockedBuffHit(Int32)    39 deviationRange(Int32)
```

**易混淆字段的正确解读**：

| 字段 | 解读 |
|---|---|
| **`weaponAccuracy` vs `staticAccuracy`** | 前者是基础/移动精度（实际启用）；**`staticAccuracy` 全武器为 0**，并非"静止精度绝对值"，语义未确证 |
| **`fireRate` vs `fireRateReadOnly`** | `fireRate = 0` 表示"由 `fireRateArray` 决定"（突击步枪）；狙击枪两者相同；**联装武器的 `fireRateReadOnly` = 单管值 × 管数** |
| **`fireRateArray` / `fireNumArray`** | 多档射击模式。如突击步枪 `[150, 600]` + `[(6,1), (6,2)]` = 单发 150 / 连发 600，每次点射 6 发 |
| **`damageSources`** | `[(weight, damageType)]` 列表 —— 伤害构成。手榴弹 `[(100,1)]`；QQB-9 温压 `[(80,1),(100,16)]` |
| **`sceneDamage`** | **对建筑/工事的破坏力** —— 区分武器用途的核心指标：火箭炮 800 / 航空火箭 600 / 125mm 炮 300 / 高射炮 40 |
| **`deviationRange`** | 散布。**步兵枪械全为 0**，只有火炮/火箭/榴弹非 0 |
| **`shockedBuffHit`** | **压制值**（见 16.5） |

### 16.3 多弹种机制：`isMultiAmmoWeapon` + `parentWeapon`

* **`isMultiAmmoWeapon` 只被界面读**：`WeaponInfoCellUI::Start`、`UnlockEntryCell::Unlock` ✗ 战斗代码不读它
* **`parentWeapon` 被 `WeaponInfoCellUI::OnDropdownWeaponValueChanged` 读** —— 是**武器信息面板里的下拉框**
* 所以多弹种 = **同一件武器拆成多条 `WeaponData`，子条目用 `parentWeapon` 指向基型**，
  基型放在 `InteralDataWeapon` 且 `bulletID = 0`：

| 基型 | `multi` | 子条目 | 弹种 |
|---|---|---|---|
| **10073 BRA-7单兵火箭筒** | True | 10155 / 10156 / 10157 | HE / HEAT / RPO |
| **10280 BRA-29重型单兵火箭筒** | True | 10281 / 10282 | HEAT / RPO |
| 10085 QQB-9（物品 `10285`） | — | 10286 / 10287 | HEAT / HE |

**全 71 条武器里只有这 2 条基型是 `multi = True`** ✓ —— **连车载火炮都不启用**：
`10111 125mm滑膛炮` 与 `10112 125mm穿甲弹`、`10334/10335 57mm穿甲/高爆`都是**独立条目**，
是另一种"换弹种"做法 ✓

⚠️ **不要用 6 参 `SpwanArms`**（见 §5.4）—— 与多弹种无关但常被混淆。

### 16.4 装填与换装：`EquipmentData.replaceableTypes`

```csharp
// MemberData.weaponList / .equipment 都是 EquipmentData[]
EquipmentData { string itemName; int itemID; int itemNum; int useOrder;
                List<WeaponType> replaceableTypes; }   // ← 该槽允许被哪些武器类型替换
```

**`replaceableTypes` 为空 = 该槽不可换装** ✓。实测：

| 士兵 | 槽 | 武器 | `replaceableTypes` | 含义 |
|---|---|---|---|---|
| 20002 步枪手 | 主武器 | 10019 | **（空）** | 不可换 |
| 20001 机枪手 | 主武器 | 10175 | （空） | 不可换 |
| 20004 狙击手 | 主武器 | 10021 | （空） | 不可换 |
| 20038 **特战队士官** | 主武器 | 10269 | **[11]** | 可换机枪 |
| 20039 **特战队军官** | 主武器 | 10269 | **[11, 13]** | 可换机枪/狙击枪 |
| **20038/20039** | **副武器** | **10074 CB-25** | **[9] = RPG** | **该槽可在 CB-25 与火箭筒之间切换** |
| 20040 喷火兵 | 主武器 | 10251 | [16] = Thermobaric | 可换温压 |
| 20016 掷弹兵 | 副武器 | 10156 / 10155 | [16, 2, 9] | 可换温压/炮弹/火箭 |

**关键实例：`10074 CB-25枪挂式榴弹发射器` 没有物品条目**（`ArmamentItemConfig` 105 条里没有它 ✗）
→ **不可采购**，只能通过**编制**出现在 **`20038 特战队士官` / `20039 特战队军官`** 的副武器槽（各 5 发）✓
这是"枪挂式"的正确读法：**依附在消音突击步枪（10269）下的附加火力，非独立装备**。

**弹药量写在 `itemNum` 里**（步枪手 180 发、机枪手 300 发、狙击手 100 发）✓

### 16.5 压制机制

* 武器侧：**`shockedBuffHit`** —— 爆炸类武器（手榴弹除外）**全部为 5**，手榴弹 0
* 单位侧：**`BaseUnitAi.unitIsSuppressed`**（`get_unitIsSuppressed`）
* 效果：**打断行为**。已确证的例子 —— 被压制时**不能去食堂吃饭**
  （`SafeToDiningInBase` 的门禁之一）

### 16.6 营养与食堂机制

**三层结构**：配给标准（`RationData`） → 食堂区域（`Area 60005`） → 单位就餐状态

**① 营养状态**：`BaseUnitInfo.nutritionData` 是 `Vector3`
= **(x 热量, y 蛋白质, z 维生素)** —— 由 `GetCalorieStatusLevel(x)` / `GetProteinStatusLevel(y)` /
`GetVitaminStatusLevel(z)` 分别判级 ✓

**② 四级界线**（数值来自 `GameGlobalVariables::.cctor`）：

| 营养轴 | Deficient | Low | Sufficient | 等级 |
|---|---|---|---|---|
| **热量** | **2400** | **3200** | **4000** | ≥4000→4 · ≥3200→3 · ≥2400→2 · <2400→1 |
| **蛋白质** | **60** | **100** | **120** | 同上结构 |
| **维生素** | **25** | **40** | **80** | 同上结构 |

（每轴另有 `_Max` 变体，在 `Sufficient` 之前赋值，供界面封顶用）

**③ 吃饭周期：`SecondsForEat = 30`** —— 注意名字写 "Seconds" 但传给了
`DateTime::AddMinutes`，**实际按分钟** ✓ 且 `GameTimeToRealTimeScale = 1.0`（游戏时间 = 现实时间）

**④ 触发条件（不是"饿了才吃"，是定时 + 门禁）**

```
每 30 分钟 → SafeToDiningInBase()  门禁：
    步兵（isArmament == false）· GameObject 激活 · 未移动 · 未战斗 · 未被压制
        ↓ 全通过
DiningInIdelState 协程 → nextEat = Now + 30 分钟（期间移动/战斗/被压制则中断）
        ↓
DiningAtBase() —— 必须站在 inFobRange != null 的地块上（否则 false）
                 → 从 FOB.SupplyStockDic 筛选食材 → 进食 → AddIntake() 累加 nutritionData
```

**⑤ 数据表**：`ConfigsTable.FoodDataDic`（食物）· `AmmoDataDic`（`FoodData`，弹药复用该结构）·
`FoodItemConfigs`（资产）· `FoodNutritionManager`（**配餐自定义界面**，不是吃饭逻辑）
营养换算基准：主食**每 100g**、肉/蔬菜**每 50g**、油脂**每 10g**、零食/饮料**每 50g** ✓

### 16.7 数值实例（可直接对照）

| 武器 | ID | 伤害 | 场景 | 穿深 | 射程 | 精度 | 弹匣 | 装填 | 弹速 |
|---|---|---|---|---|---|---|---|---|---|
| 5.45mm74型突击步枪 | 10019 | 3 | 3 | 1 | **0–12** | 60 | 30 | 4 | 16.46 |
| 7.62mm79型狙击步枪 | 10021 | 5 | 5 | 1 | **0–24** | 90 | 10 | 3 | 25.71 |
| **7.62mm79型（抑制器）** | 10270 | **3** | 3 | 1 | **0–8** | 90 | 10 | 3 | **18.29** |
| 12.7mmMQ2反器材 | 10273 | 12 | 20 | 2 | **0–36** | 100 | 10 | 3 | 35.27 |

**抑制器版的设计规律**：**只砍伤害 / 射程 / 弹速，精度、射速、瞄准、弹匣、装填全不变** ✓

物品编码：武器物品 `itemType = 4096`（2¹²），`subItemType` 为 **16384**（2¹⁴）或 **16777216**（2²⁴）两个子类旗标 ✓

---

## 变更记录

| 版本 | 摘要 |
|---|---|
| v5.68.2 | 瓦莲娜隐藏（`Specs[]` 回到 43）；安装包重打 |
| v5.68.1 | `Hidden` 字段（预留，未启用） |
| v5.68.0 | 瓦莲娜重置为 44×64 卡车两倍尺寸（`measure_pivots` 实测 pivot） |
| v5.67.0 | **修复读档重复刷新**（三条规则补 `mapArmsSpawnActivedList` 记账） |
| v5.66.2 | 瓦莲娜 pivot 按 `audit_pivots` 实测值修正 |
| v5.66.0 | 瓦莲娜改为**载具**（弃用步兵班路径）；F6 单位循环停用 |
| v5.63.1 | 多色载具名称去掉颜色后缀（21 处） |
| v5.63.0 | 随机朝向（`UnitDirection`）+ 同格叠放偏移（`StackOffset`） |
| v5.62.0 | 所有刷新载具开局燃料 0-30.0 L（`SetSpawnFuel`，按容量夹紧） |
| v5.61.1 | 城市道路改为**全图**配额（2-3 台）；修 `ProcessPendingSpawns` 早退守卫 |
| v5.61.0 | 城市道路 → 重型应急载具（2-3 台，耐久 0-70%，200-1000 发 5.45 散装） |
| v5.60.0 | 事件点规则（3-7 台事故车 0-30% + 0-3 台应急救援）；同格最多 3 台 |
| v5.59.0 | `vehicle_specs.py` 统一解析器；修池 Key 不匹配与 `Pools[0]` 分叉两个 bug |
| v5.58.0 | ZIL-4310 民用 6 型号（物资上限 ×5） |
| v5.57.0 | B1000 / W50 各 6 型号 + 随车物资体系（`CargoEntry`） |
| v5.56.0 | 车辆刷新点 → 民用轻型载具池（每车位 20%） |
| v5.55.0 | 部件耐久随机化改为等 `armInitialized` 之后执行（修"部件全新"BUG） |
| v5.54.0 | 医院/火车站改为**全图**配额（修"4 卡车 + 4 救护车"） |
| v5.53.0 | 生成位置改为建筑外圈空闲地块（修"车停在屋顶上"） |
| v5.51.0 | 火车站规则；医疗清单移除抗感染药 10009 |
| v5.48.0 | 医院救护车规则 |
| v5.47.0 | 4 个载具池 |
