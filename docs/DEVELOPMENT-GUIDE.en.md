# Frontline Vehicles — Development Guide

_English translation of DEVELOPMENT-GUIDE.md. The Chinese original is authoritative; if the two disagree, the Chinese version wins._

Applies to version **v5.68.2** · game `FrontlineLogistics_IsaraWarfare 0.8.706` (Steam demo, Unity 2021.3.20f1c1 / Mono)
Last updated: see the changelog at the end of this document

> This document is written for **whoever takes over this MOD — a human or an LLM**. After reading it you should be able to complete the following unaided:
> adding vehicles, changing sprites, changing text, adding spawn rules, changing configuration, building, verifying, packaging and releasing.
> Every "game internal fact" in this document has been verified against IL / assets, with the location of the evidence given — **do not overturn them on intuition**.

---

## 0. Thirty-second overview

| Item | Value |
|---|---|
| What this is | The vehicle MOD (BepInEx plugin) for *Frontline Logistics: Isarian Warfare* (《前线后勤：伊萨拉战争》), currently **43 vehicles** + 5 categories of building spawn rules + an onboard cargo system |
| Plugin DLL | `FrontlineVehicles.dll` (**not** `Uaz469.dll` — that was the name of its predecessor, the "super mechanised MOD") |
| Plugin GUID | `com.mod.uaz469` (**deliberately kept identical to the old version** so that users' existing `.cfg` settings can be inherited) |
| Configuration | `BepInEx\config\com.mod.uaz469.cfg` |
| Source | `mod\deploy\plugin-src\Uaz469Plugin.cs` (**a single file**, about 4600 lines) |
| Compiler | `C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe` (**only supports C# 5**) |
| Build | `mod\deploy\plugin-src\build.ps1` |
| Game directory | `D:\SteamLibrary\steamapps\common\FrontlineLogistics_IsarianWarfare_Demo\FrontlineLogistics_IsaraWarfare` |

**Iron rules**:
1. Only **C# 5** is supported (no `?.`, string interpolation, `nameof`, expression-bodied members, `async`…).
2. **Never modify any vanilla game file** (currently 172 vanilla files, zero changes; all text rewriting happens in memory).
3. **Do not change the configuration defaults** — the defaults are conservative values meant for new users; all tuning goes through the `.cfg` (the user explicitly asked for this).
4. After every build you **must run the full verification matrix** (section 8) — not one item may be skipped.

---

## 1. Directory structure

```
mod\
├─ deploy\
│  ├─ plugin-src\
│  │  ├─ Uaz469Plugin.cs        ← all source code (single file)
│  │  └─ build.ps1              ← build script (embeds 24 sprites)
│  ├─ sprites\                  ← 24 processed sprite sheets (build.ps1 reads from here)
│  └─ loader\BepInEx-5.4.23.3-x64\   ← BepInEx distribution files (build output lands here)
├─ src\                         ← unprocessed source art (PNGs supplied by the artist)
├─ tools\                       ← all Python/PowerShell tools (section 7)
│  └─ pylibs\                   ← dependencies such as Pillow (scripts sys.path.insert these)
├─ spec\                        ← design docs and prompts for the artist (section 11)
├─ out\                         ← reports/JSON produced by the tools (can be rebuilt at any time)
├─ release\                     ← **release templates** (README/MANUAL/CHANGELOG/install/uninstall)
├─ dist\                        ← packaging output (directory + zip)
└─ backup\                      ← archive (e.g. the deleted tanker truck sprites)
```

**Not a git repository**: this repo is not a git repo; history is reconstructed from `backup\` and version numbers.

---

## 2. Build

```powershell
powershell -ExecutionPolicy Bypass -File 'D:\Codex Work\mod\deploy\plugin-src\build.ps1'
```

Output: `deploy\loader\BepInEx-5.4.23.3-x64\BepInEx\plugins\FrontlineVehicles.dll`
(**not** the game directory — deployment is a separate step, see 2.3)

### 2.1 What build.ps1 does

1. **Encoding guard**: verifies that the source is valid UTF-8, that the Chinese constants are intact, and that **the number of ASCII `"` on every line is even**.
2. Compiles `Uaz469Plugin.cs` with `csc.exe /codepage:65001`.
3. Bakes the **24 PNGs** in the `$sheets` array into the DLL **as embedded resources** (`/resource:`).
4. Reports the DLL size.

### 2.2 ⚠️ The quote guard (the most commonly stepped-on trap)

The script counts ASCII quotes line by line; **an odd number means BUILD FAILED**. Two real pitfalls:

* **A single ASCII `"` inside a comment** → failure. **Use 「」 or Chinese quotation marks in comments.**
* **Splitting a string literal across two lines with `\"`** → failure. Better to write one long line than to split it.

(Chinese quotes `“”` are not counted, so using Chinese quotation marks inside Chinese comments is safe.)

### 2.3 Deployment

```powershell
# Make sure the game is not running first!
Get-Process -Name 'FrontlineLogistics*'
Copy-Item <build output> '<game directory>\BepInEx\plugins\FrontlineVehicles.dll' -Force
```

**Copying while the game is running fails**: `The requested operation cannot be performed on a file with a user-mapped section open`.
The game must be closed first.

There should be **only this one DLL** in `BepInEx\plugins\`; lazy backups such as `*.old-*` / `*.locked_*` are not loaded by BepInEx and may be left in place.

---

## 3. Source structure (Uaz469Plugin.cs)

A single file, organised in the following order (look here to locate things before changing code):

| Region | Purpose |
|---|---|
| `Plugin : BaseUnityPlugin` | BepInEx entry point, `[BepInPlugin(GUID, "Frontline Vehicles", "x.y.z")]` |
| `VehicleSpec` + `Specs[]` | **the vehicle spec table** (one entry per vehicle); `Specs = new VehicleSpec[43]` |
| `Initialize()` | builds every spec → `BuildLightVariants()` / `BuildHeavyVariants()` → binds all configuration |
| Part renaming system | `PartText` / `PartBase` / `ClonePartWithText` / `MirrorConfigTables` / `MirrorTextTables` |
| `InjectAll()` | injection entry point: `SanitizeText()` → `InjectOne()` per vehicle |
| `VehiclePool` + `Pools[]` | 4 vehicle pools + `LogPools()` self-check |
| Spawn rules | see section 5 (5 rules + a shared spawner) |
| Onboard cargo | `CargoEntry` / `CargoForSpec` / `GiveCargo` (section 6) |
| Variant generators | `MakeLightVariant()` / `BuildLightVariants()` / `BuildHeavyVariants()` (section 4.3) |
| `VehicleDriver` | runtime MonoBehaviour: key input, HUD, queue processing, `InjectAll()` triggering |

### 3.1 `VehicleSpec` fields

```
Key, SrcChassisKey, SrcArmamentId, ChassisId, ChassisKey, ArmamentId, ArmamentInt,
DisplayName, ChassisName, BaseName, Price, Passengers, Cargo,
ExtraDefaultParts, InvisibleExtraParts, Desc, PartTexts,
SheetResource, CellW, CellH, Row, Col, PivX, PivY, TestTile, ResolvedTile
```

**ID rules (must be obeyed)**:

* Chassis IDs are allocated sequentially starting from **10346**; already taken: `10346–10379`, `10381–10392`.
* **`10380` is permanently disabled** — it is a **ghost part ID** of `ZIL-4310POL` (`ChassisId + 10 = 10370 + 10`),
  and using it again causes dictionary key collisions.
* **Ghost part ID = `ChassisId + 10 + i`** (used for "invisible extra parts").
* **Clone part ID = `ChassisId * 100 + (stock part ID % 100)`**.
* Armament ID = `ChassisId * 100000` (string form, e.g. `1034600000`).
* There are only three vehicle source families (`SrcChassisKey` / `SrcArmamentId`):
  * **pickup family (皮卡族)** `"10050"` / `"1005000000"`
  * **AA-320 family (AA-320 族)** `"10305"` / `"1030500000"`
  * **van family (面包车族)** `"10348"` / `"1034800000"`

### 3.2 Injection chain (`InjectOne`)

1. Register the armament data in `ArmamentDataDic` + mirror **every table in `ConfigsTable` that is indexed by part ID**
   (`ArmamentPartDataDic`, `ArmamentFuelTankDataDic`, `ArmamentEngineDataDic`,
   `ArmamentTurretPartDataDic`, `ArmamentChassisDataDic`), `ItemDataDic`, `MirrorItemSets`,
   and every `Dictionary<int,string>` in `GameTextData`.
2. `ClonePartWithText()` clones the part and writes the renaming text into it.
3. Sprites are taken out of the embedded resources, sliced into cells by `Row/Col/PivX/PivY` and written into `ChassisData._8DirectionalSprites`.

**Why part renaming requires cloning**: although `PartComposition.partName` is written, it **is never read** (proven by IL cross-references);
what the UI displays is `PartData.partName` — and `PartData` is shared per part ID,
so without cloning it is impossible to achieve "one set of part names per vehicle".

---

## 4. Adding a vehicle (standard procedure)

### 4.1 Checklist

1. Put the source art (**PNG with a real alpha channel**) into `mod\src\`.
2. Run the sheet pipeline (section 9) → output goes to `mod\deploy\sprites\`.
3. Add a `VehicleSpec` block in `Uaz469Plugin.cs`, filling in `Key` / IDs / names / description / part texts / sprite name / cell size / pivot.
4. `Specs[n] = x;` and **increment the N in `Specs = new VehicleSpec[N]`**.
5. Add a line to the `$sheets` array in `build.ps1` (`"name_sheet.png"`).
6. (Optional) add it to a `VehiclePool`.
7. Build → run the verification matrix (section 8) → re-run `extract_vehicle_table.py` → deploy.

### 4.2 The cheapest approach: copy one from the same family

Vehicles in the same family usually share the same `SrcChassisKey` / `SrcArmamentId` / `CellW` / `CellH` / pivot.
**Do not fill in the pivot by feel** — measure it with `measure_pivots.py`, or verify it with `audit_pivots.py`.

### 4.3 Model variants (do not copy-paste 12 spec blocks)

The "clothing / home appliance / furniture / agricultural / industrial / coal hauling" variants of B1000 / W50 / ZIL-4310CIV are **generated programmatically**:

```csharp
VehicleSpec b = SpecByKey("B1000BLUE");
MakeLightVariant(b, LightTypeSuffix[i], LightTypeNameCn[i],
                 B1000VariantChassis[i], B1000VariantSheets[i], 25 + i);
```

`MakeLightVariant` **copies every appearance field of the base spec verbatim** (sprite / cell size / pivot / description / part texts),
and only changes the `Key`, the names, and the chassis and armament IDs. **This way no new art is needed, and the pivot can never be copied wrong.**

⚠️ **The cost**: these variants are a blind spot for tools that "parse literal spec blocks" — solved with the unified parser `tools/vehicle_specs.py`
(it **understands both literal blocks and generators**). When adding a new variant you **must confirm that the parser recognises it too**
(run `python tools\vehicle_specs.py`; the total should match the runtime).

⚠️ **Lesson paid for in blood**: always run verification after adding a variant — twice, an inconsistency between "the Key in the pool and the generated Key"
(`B1000-CLOTH` vs. the actually generated `B1000BLUE-CLOTH`) caused a variant to **silently never spawn**,
completely invisible to the naked eye; the verifier caught it.

---

## 5. Spawn rules (five of them) and the spawner

### 5.1 The five rules

| Rule | Field prefix | Trigger buildings (configurable) | Behaviour |
|---|---|---|---|
| Hospital ambulance | `[Hospital Ambulance]` | `HospitalBuildings` (default 50085,50086) | 1-2 per map, parking spots first |
| Train station heavy civilian | `[Train Station Truck]` | `StationBuildings` (50070) | 1-2 per map |
| Vehicle refresh → light civilian | `[Vehicle Refresh]` | `RefreshBuildings` (13 categories of town buildings + roadside parking lots) | **20% per parking spot**, within the pool 40/30/20/10 |
| Event location | `[Crash Site]` | `CrashBuildings` (50040) | **per event location** 3-7 wrecked vehicles (durability 0-30%) + 0-3 emergency rescue |
| City road → heavy emergency | `[City Road]` | `CityRoadBuildings` (the same 13 categories) | **1-2-3 per map** (the quota is rolled once) |

### 5.2 Architecture: **queue → deferred processing**

```
BuildingInfo.AfterStart  (Harmony postfix BuildingSpawnPostfix)
        ↓  only does lightweight checks, pushes the building into the corresponding queue
   PendingSpawns / PendingRefresh / PendingCrash / PendingCityRoad
        ↓  every frame VehicleDriver.Update calls ProcessPendingSpawns()
   actual spawning (by then the armament tables are injected and the map is ready)
```

**Why this is mandatory**: `AfterStart` runs during the Unity `Start` phase and **may run before this MOD injects the armament tables**;
spawning on the spot would get empty data. As long as a queue is non-empty it retries every frame until the conditions are met.

⚠️ **The early-return guard must check every queue**:
```csharp
if (PendingSpawns.Count == 0 && PendingRefresh.Count == 0 &&
    PendingCrash.Count == 0 && PendingCityRoad.Count == 0) return;
```
(It once checked only `PendingSpawns`, which meant "when there is no hospital/train station on the map, the city road rule never runs".)

### 5.3 The correct way to spawn one vehicle

```csharp
// 1) Root anchor GameObject (it must be a root object, otherwise DataManager.InitializeUnit() will
//    NRE on child.GetComponent<BaseUnitAi>())
GameObject anchor = new GameObject("spawn");
anchor.transform.position = worldPos;
// 2) The 4-parameter overload — never use the 6-parameter one (see 5.4)
BaseArmamentInfo info = dm.SpwanArms(armamentId, TypeManager.CampType.BCamp, anchor.transform, null);
// 3) Parent it under the unit container, destroy the anchor
info.transform.SetParent(dm.unitContainer, true);
Destroy(anchor);
// 4) Register the tile explicitly (that part of the vanilla code is gated off)
info.baseArmamentAi.SetUnitOnTile(tile);
```

### 5.4 ⚠️ The three most critical "traps" in this game

**① `DataManager.FullReleaseMark()` is hard-coded to return false**

```
DataManager::FullReleaseMark() -> System.Boolean
     0 ldc.i4.0
     1 ret
```

**The very first step of the entire post-processing block of the 6-parameter `SpwanArms` is it**:

```
190 call  DataManager::FullReleaseMark
195 brfalse IL_013d          ← if false, skip the whole block
```

Consequence: **the vanilla path that spawns vehicles at buildings is dead in this game** — no random durability, no `SetUnitOnTile`,
no random paint scheme, no recomputation of the initial value. **Therefore:**

* **Never spawn vehicles with the 6-parameter `SpwanArms`**;
* random durability, fuel, facing and tile registration **must all be done by yourself**.

The vanilla "car crash wreck (0-50% durability)" therefore does not take effect at all in this game — which is exactly why we built the whole set ourselves.

**② `partList` is still empty when `SpwanArms` returns**

`partList` is built by `BaseArmamentInfo.Initialization()` during the **Start phase**.
So changing part durability immediately after spawning **does nothing at all** (this once caused "every spawned vehicle had brand-new parts").

The correct approach: **use a coroutine to wait until `armInitialized == true` before touching anything** (do not test `partList.Count > 0` —
parts are added to the list one by one, so only half may be installed):

```csharp
for (int i = 0; i < 300 && info != null && !info.armInitialized; i++) yield return null;
```

`DeferredSetup()` is that single consolidation point; it handles: **durability randomisation → fuel → facing → loading cargo**.

**③ Facing is controlled by a public property**

```
BaseUnitAi::set_UnitDirection @IL_0059  →  BaseArmamentAi::SetChassisSprite(InGameDirection)
```

A parked vehicle receives no movement order, so its facing is always the default `Up(0)` — which is why **all vehicles face the same direction**.
Random facing must be set explicitly (and it must come **after** `SetUnitOnTile`, because that method calls `SetChassisSprite` itself):

```csharp
info.baseArmamentAi.UnitDirection = (lgBase.DirectionDetector.InGameDirection)Random.Range(0, 8);
```

Enum: `Up=0, TopRight=1, Right=2, BottomRight=3, Down=4, BottomLeft=5, Left=6, TopLeft=7, None=8`.

### 5.5 Placement: finding a free tile

```csharp
FindFreeTileNear(building, usedSet, roadOnly)
```

* `BuildingInfo.tileList` is a **private field that this assembly cannot read** → scan **ring by ring outwards** (radius 1→6) centred on `buildingOnTile.gridLocation`.
* Criteria: `t.BuildingOnTile == null` (**no building means we are outside any building**), `TileHasRoom(t)`, and optionally `tilePathDataGroupDic.Count > 0` (≈ road).
* **Never place a vehicle on the building's own tile** — this was once done, and the vehicles ended up parked on the roof (confirmed by the user in play).
* **At most 3 per cell**: `TileStack` counting + `unitsOnTileList.Count`; `MarkTile()` returns "which vehicle on this cell this is", and `StackOffset()` is added accordingly so that sprites do not overlap.
* When no position can be found, **log a warning and stop**; do not fall back to the building tile.

### 5.6 Map-wide quota vs per-building counting (**extremely easy to get wrong**)

When the user asks for "N vehicles" they **almost always mean the whole map**. Counting per building leads to "hundreds of vehicles" (the user has caught this twice in play).

```csharp
// roll the quota once for the whole map; buildings consume it in turn and it stops when exhausted
int left = Random.Range(min, max + 1);
for (...) { if (left <= 0) break; left -= SpawnOne(..., left); }
```

* **Only the event location rule** counts **per location** (explicitly requested by the user).
* All the other rules use a map-wide quota.

### 5.7 Save-game bookkeeping (**mandatory — omitting it guarantees a BUG**)

The game uses `MapManager.mapArmsSpawnActivedList` (`Dictionary<Vector3Int, string>`) to record "has this building / this parking spot spawned already",
and **this dictionary goes into the save file** (`WorldLowDynamicData` → `PerformSaveGameAsync` → `TilemapGenerator.GenerateMapDetail`).

> ⚠️ **Every spawn rule must write it once implemented.** Omitting it means:
> every save load → the building's `AfterStart` runs again → the rule executes again → **another batch spawns** ✗

**v5.67.0 fixed this BUG**: the city road / crash site / vehicle refresh rules initially forgot the bookkeeping,
which showed up as "**every time you load a save, the heavy emergency vehicles keep multiplying**" (confirmed by the user in play).

Key number-range allocation (so that rules do not overwrite each other):

| Rule | Key | Semantics |
|---|---|---|
| Hospital / train station | `(x, y, 500+i)` / `(x, y, 590+n)` | parking spot / fallback position on the outer ring |
| Crash site | `(x, y, 700)` | this event location has already spawned |
| **City road** | **`(-1, -1, 800)` (sentinel key)** | **this rule has already run in this save** → the map-wide quota becomes "once per save" |
| Vehicle refresh point | `(x, y, 900+i)` | this parking spot has already spawned |

**Four key points**:

1. **`TileStack` / `GlobalSpawnTiles` are in-memory static collections and do not go into the save file** — they can only prevent duplication within one session,
   they **cannot stop a save load from spawning again**, and they are no substitute for `mapArmsSpawnActivedList`.
2. **Map-wide quota rules must use a sentinel key** — otherwise the quota is re-rolled on every save load and still accumulates (this is exactly how the city road rule was fixed).
3. Using `(-1, -1, …)` as a sentinel is safe: negative coordinates do not exist on the map, so it can never collide with a real tile.
4. **The fix cannot reclaim vehicles that are already duplicated in a save file**; it only guarantees that no more will be added.

---

## 6. Onboard cargo system

`CargoEntry { Label; Min; Max; int[] ItemIds; }` — a group of **items of the same kind** shares one quantity.

```csharp
GiveCargo(info, tag, table, mult)   // per entry: first roll the total Min..Max*mult, then distribute it randomly item by item within the group
```

* **Multi-item semantics**: roll the total first, then distribute it randomly item by item (chosen by the user) — e.g. "5-20 furniture" distributed among wooden tables / wooden chairs / wooden beds.
* **The multiplier `mult` only applies to the upper bound** (the lower bound is unchanged): W50 **×2**, ZIL-4310CIV **×5**,
  determined by `CargoForSpec(key, out mult)` from the Key prefix.
* **You must report back the actual loaded amount**: `GiveItem()` is clamped by the load capacity, so trust only the value it returns (do not trust the rolled number).

### 6.1 Item IDs (measured, do not guess)

| Name | ID | Remark |
|---|---|---|
| 5.45mm ammunition (loose) | **10001** | loose (bulk); must be pressed/belted before use |
| Hemostatic pack / plasma / **anti-infection drug** / morphine / anaesthetic / surgical instruments | 10007 / 10008 / **10009** / 10010 / 10011 / 10033 | 10009's name is the placeholder `To be iterated`, and **the user has asked for it to be removed from the spawn list** |
| Engine / light weapon / heavy weapon / fuel tank system / wheeled mechanism generic parts | 10053–10057 | |
| Mechanical parts / mechanical lubricant | 10058 / 10059 | |
| **Smoke grenade** | **10060** | |
| Scrap metal (metal waste) | 10071 | |
| Chip / cloth | 10077 / 10079 | |
| Lathe / large home appliance / small home appliance | 10082 / 10083 / 10084 | |
| Canned cucumber / canned meat | 10087 / 10088 | |
| Fertiliser / storage battery | 10093 / 10134 | |
| Quick repair kit | 10118 | |
| Iron pickaxe / shovel | 10147 / 10149 | |
| Coal lump | 10154 | |
| Butter | 10212 | |
| Wheat flour / rye flour | 10214 / 10215 | |
| Tracked mechanism generic parts | 10190 | |
| Worn wooden table / wooden chair / wooden bed ("furniture") | 10029 / 10030 / 10078 | there is **no** single "furniture" item in the table |
| Petrol / diesel / aviation kerosene (in cargo form) | 10044 / 10045 / 10326 | |

**No item in the whole table is called "tool"** — the user specified that the **generic parts family** (10053-10058, 10190) be used instead.
The only entry in the whole table that contains "tool" is `70016 修复手术(缺乏工具)` (repair surgery (lacking tools)), which is an **action prompt**, not an item.

---

## 7. Toolchain

### 7.1 Core parser (**the foundation of every verification tool**)

`tools\vehicle_specs.py` — a unified parser for `Uaz469Plugin.cs` that **understands both literal spec blocks and variant generators**,
and outputs a complete vehicle table matching the actual runtime. After changing vehicles, run it first as a self-check:

```powershell
python tools\vehicle_specs.py      # should print 43 (base + variants)
```

### 7.2 Verification tools

| Tool | What it verifies |
|---|---|
| `verify_dll.py` | every sprite sheet inside the DLL is **byte-for-byte identical** to `deploy\sprites\`; the version string; `unexplained == 0` |
| `extract_vehicle_table.py` | generates `out\vehicle-table.json`; reports the total / duplicate IDs / duplicate slots / number of unique sprites |
| `verify_part_text.py` | derived part ID collisions, conflicts with stock IDs, whether the part a renaming text refers to is really on the vehicle |
| `verify_pools.py` | whether the Keys in the pools exist (**a typo silently loses a vehicle**), whether any vehicle is in two pools, who is in no pool |
| `check_grid.py` | whether the eight directions land 1:1 on the canonical cell positions |
| `audit_pivots.py` | whether the pivot is reasonable |
| `check_dll_text.py` / `find_strings.py` | whether certain strings exist in the DLL |
| `il-query.ps1` | IL queries: `-Fields` `-Methods` `-Xref` (methods **and fields**) `-Call` `-New` `-Find` `-Str` |
| `il-dump.ps1` | decompiles the IL of the specified method/type |

### 7.3 Sheet tools

See section 9.

### 7.4 Packaging tools

See section 10.

---

## 8. Verification matrix (**must** be run in full after every build)

```powershell
cd 'D:\Codex Work\mod\tools'
python extract_vehicle_table.py     # total / duplicate IDs / duplicate slots / unique sprites
python verify_part_text.py          # 0 derived part ID collisions
python verify_pools.py              # 0 pool problems
python verify_dll.py                # RESULT: PASS
python check_grid.py                # PASS
python audit_pivots.py              # 0 issues
```

**You must look at the output of every single one of them; do not just look at the exit code.**

### 8.1 Known blind spots and lessons from history

* The expected list in the early `verify_dll.py` came from `out\vehicle-table.json` — if you added a vehicle but forgot to regenerate it,
  you would get `expected 23 / embedded 24 / unexplained 1` and it would **still PASS**.
  Now `unexplained` is computed by hash and is required to be 0.
* The early verification tools **only recognised literal spec blocks**, so the 18 model variants were completely invisible to them → solved with `vehicle_specs.py`.
  **After adding a new variant you must confirm that the parser recognises it.**
* `lost_content.py` gave misleading numbers (1652 vs. the actual 37); **trust the per-cell pixel counts from `cell_counts.py`**.
* `count_specks.py` false-positives on white vehicle bodies; **use `count_islands.py`**.

### 8.2 Console encoding trap

Python output piped through PowerShell gets **double-encoded as GBK** (all Chinese turns into mojibake).
**The correct approach: have Python write the file itself with `io.open(..., encoding='utf-8')`, then read it with `Get-Content -Encoding UTF8`.**

---

## 9. Sprite sheet pipeline (the order cannot be changed)

```powershell
cd 'D:\Codex Work\mod\tools'
python prepare_sheet.py   <source image>                        # background removal
python refix_sheet.py     <previous step's output> --perm standard   # rearrange into the engine's field order
python fringe_flood.py    <previous step's output>                   # clean leftover specks outside the outline
python measure_pivots.py  <previous step's output>                   # measure pivots (**must be last**)
```

⚠️ `prepare_sheet.py` changes the bounding box, so **the pivot must be measured last**.

### 9.1 Order and facing evidence

The sprite sheets the user supplied use the **old drawing order** → use `--perm standard`, but **you must verify it**:
`refix_sheet.py` has its own guard (`CLASS_OF_DIR`, keyed by **direction name** rather than field name),
and you should independently re-check it once more (bounding-box classification comparison + visual strip inspection).

Field order: `Up, TopRight, Right, BottomRight, Down, BottomLeft, Left, TopLeft`.
Field index `k` serves the world facing `45°×(k+1)` (0° = north, clockwise).

Canonical cell positions:
```
GridRow = { 0,0,1,2,2,2,1,0 }
GridCol = { 1,2,2,2,1,0,0,0 }
```
`(0,0)=N, (0,1)=NE, (0,2)=E, (1,0)=NW, (1,2)=SE, (2,0)=W, (2,1)=SW, (2,2)=S`
(narrow views are at (0,0)/(2,2), side views at (0,2)/(2,0), diagonals in the remaining positions.)

### 9.2 Acceptance (**do not look only at the bounding box**)

```powershell
python cell_counts.py   <sheet>    # per-cell pixel count
python count_islands.py <sheet>    # number of fragments
python dark_preview.py  <sheet>    # produce a dark preview image for visual inspection
```

### 9.3 Generator quirks (they reproduce reliably)

* **Sheets derived from ZIL-4310**: the two pure side views sit **6-7 px higher inside their cells than the other six** (content y=6..25 or 5..23, the rest y=1..30).
  → such sheets **need per-direction pivots**, for example `ZIL-4310CIV: PivX {22,21.5,21.5,22,21.5,21.5,22,22}` / `PivY {1,8,2,1,2,8,1,1}`.
* Lossless PNGs on a white background are **fully opaque** and contain 4000–5700 noise colours; `--white 238` is safe (it does not eat bright pixels in the interior),
  and `fringe_flood.py` then cleans a further 0–40 px.
* **Always use PNGs with a real alpha channel.** Chat apps convert to JPEG, which turns the transparent area into white-background noise, and background removal then eats into the subject's edges together with it — this trap has been stepped on many times.

---

## 10. Release

### 10.1 Process

1. Update the templates in `mod\release\`: `README.md` (trilingual introduction of the new content), `MANUAL.md` (trilingual manual),
   `CHANGELOG.md`, `install.ps1`, `uninstall.ps1`.
2. Package:
   ```powershell
   cd tools
   python make_release.py FrontlineVehicles-v5.63.1
   ```
   Produces `dist\FrontlineVehicles-v5.63.1\` and `dist\FrontlineVehicles-v5.63.1.zip`.

### 10.2 ⚠️ Pitfalls of `make_release.py`

* It **does `shutil.rmtree(dist\<PKG>)` and rebuilds the directory** — **do not write documents into `dist\`**,
  they will be deleted (I made this mistake; two documents never made it into the package).
* Documents are only copied from `release\`, and it **only recognises one fixed list**:
  ```python
  for name in ('README.md', 'MANUAL.md', 'CHANGELOG.md', 'install.ps1', 'uninstall.ps1'):
  ```
  **Adding a new document type means changing this line as well.**
* It has a `FORBIDDEN = ('Uaz469.dll',)` check — if the old plugin name appears anywhere in the package it aborts immediately.
* The zip uses forward-slash paths (`.NET ZipFile.CreateFromDirectory` writes backslashes, which is non-standard).

### 10.3 Compatibility guarantees (the user explicitly asked for "no conflict with installed MODs")

* The package contains **only BepInEx and its own plugin**, and no vanilla game file at all.
* The plugin name `FrontlineVehicles.dll` ≠ the old `Uaz469.dll`; the install script **deletes the old DLL**.
* **The two cannot be installed at the same time** — they share the configuration GUID `com.mod.uaz469` (deliberate, in order to inherit settings).
  This point must be stated in all three language versions of the documentation.
* ⚠️ **If a save file contains these vehicles, deleting the plugin will make that save unloadable** — deal with them in-game before uninstalling.

### 10.4 Pre-release checks

```powershell
# is the DLL in the package identical to the one online
(Get-FileHash dist\<PKG>\BepInEx\plugins\FrontlineVehicles.dll).Hash -eq (Get-FileHash <game directory>\...\FrontlineVehicles.dll).Hash
# does the package contain the old plugin name
Get-ChildItem dist\<PKG> -Recurse -Filter 'Uaz469.dll'
```

---

## 11. Documentation conventions

| File | Content |
|---|---|
| `spec\sheet-pipeline.md` | sprite sheet processing pipeline and acceptance criteria (the most detailed) |
| `spec\vehicle-pools.md` | definition of the 4 vehicle pools |
| `spec\vanilla-vehicle-spawn-rules.md` | vanilla spawn mechanics (with IL evidence) |
| `spec\newgame-spawn-rules.md` | the design of this MOD's spawn rules |
| `spec\vehicle-text-overrides.md` | record of text overrides |
| `spec\*-art-prompt.md` | **prompts for the image-generation model** (one per vehicle) |
| `spec\<vehicle>.spec.md` | design spec for a single vehicle |
| `spec\valena-art-prompt.md` | character prompt (a person rather than a vehicle; size = a standing van, 96×132) |

### 11.1 Uniform structure of an art prompt

1. Purpose statement (Chinese and English)
2. **Hard specification table** (3×3 grid / transparent background / isometric top-down 2:1 / outline #1E2019 / 2-3 flat shading steps / subject occupying 60-75% of the cell width)
3. **Nine-cell position table** (including direction notes such as "facing north = back to the viewer" + **the centre cell must be transparent**)
4. **Main English prompt** (ready to copy)
5. **Negative prompt**
6. **Post-processing pipeline** + delivery requirements (PNG with real alpha)
7. Emergency tips for the image model (nine-grid breakdown → generate cell by cell and stitch)

### 11.2 Text rules (game text)

* Localisation is **plain CSV** under `FrontlineLogistics_Data\StreamingAssets`:
  `ObjectIdDic.csv` / `DescriptionDic.csv` / `ContentIdDic.csv`,
  with columns `ID, TAG, DESCRIPTION, English, SimplifiedChinese, TraditionalChinese, Japanese, Russian, ...`
* Text must be written into `GameTextData.DescriptionDic` (it is **appended** after the auto-generated numeric lines);
  `TotalDescriptionDic` **replaces the entire panel** — do not use the wrong one.
* There is **no `\n` → newline conversion anywhere in the game assembly** (verified with `-Str '\n'`, 0 hits) → **write the description as a single paragraph**.
* **World view**: "Soviet" in the text must be displayed as "Isara" — `SanitizeText()` at the start of `InjectAll()`
  handles all `GameTextData` string dictionaries uniformly (苏联→伊萨拉, 蘇聯→伊薩拉, 俄罗斯/俄羅斯→伊萨拉/伊薩拉, i.e. Soviet → Isara in both simplified and traditional forms).
* Engine renaming is **only authorised for**: W50 → East German IFA 4 VD 14,5/12-1 SRW (**Nordhausen plant**, not Schönebeck);
  Toyota Land Cruiser → Toyota 2H (3980cc, 12-valve OHV indirect injection, used until late 1989).
  **Power figures are never changed.**
* The **names of the multi-colour vehicles** (LADA×3, B1000×3, Toyota×2) **do not include the colour** (user request),
  but the three properties `DisplayName`/`ChassisName`/`BaseName` must be changed together.

---

## 12. Game internals (quick reference)

### 12.1 Namespace traps

* **Requires the `lgBase.` prefix**: `BuildingInfo`, `BaseUnitInfo`, `MapManager`, `BaseArmamentInfo`, `BaseUnitAi`, `RangeFinder`, `FOB`, `BaseArea`, `OverlayTile`, `ArmamentPartInfo`, `DirectionDetector`
* **Global (no prefix)**: `ChassisData`, `PartData`, `ItemStockData`, `ArmamentData`, `ConfigsTable`, `DataManager`, `GameTextData`, `TypeManager`, `ParkingSpot`, `VehicleInternalSpaceUI`
* `TypeManager` is global: `TypeManager.CampType`, `TypeManager.ArmamentPartType.Chassis`

### 12.2 Common `OverlayTile` fields

```
gridLocation (Vector2Int)      tileWorldPosition (Vector3)
_buildingOnTile → BuildingOnTile (BuildingInfo)   ← if there is a building you cannot place a vehicle
unitsOnTileList (List<BaseUnitInfo>)              ← units already standing there
tilePathDataGroupDic (Dictionary<int,TilePathData>) ← non-empty ≈ passable / road
areaOnTile / theFob / inFobRange / medicalInfo / mineData
```
⚠️ **`BuildingInfo.tileList` is a private field and cannot be read from outside.**

### 12.3 Fuel

* `BaseArmamentInfo.currentFuel` (public read/write float), `MaxFuelCapacity` (public read-only,
  computed from the fuel tank part during assembly, **0 before `armInitialized`**).
* Fuel items: petrol `10044`, diesel `10045`, aviation kerosene `10326`;
  `ItemType.Fuel = 1024`, `SubItemType.FuelOil = 512`;
  `TypeManager.FuelType { Petrol=0, Diesel=1, AviationKerosene=2 }`.
* Bulk liquids are stored at the base level: `FOB.fuelTankDic` / `DataManager.FuelTankDic` (`Dictionary<FuelType,float>`).
* ⚠️ `BaseUnitInfo.GetFuel(variable, itemID)` **never decreases the item count** (it produces ghost quantities),
  and it only refuels when `itemID == EngineData.fuelTypeID` — one of the root causes of the historical tanker truck BUG.

### 12.4 Load and cargo

* `BaseUnitInfo.ItemTransferAndUpdateFreeItemDic(itemID, variable, showTips, pickEmplacedWeapon, itemDatas)`
  → writes "free cargo carried by the vehicle". All three no-op paths return **`null`** (= refusal),
  and **returning an empty `List<>` is wrong** (the caller interprets it as "everything consumed").
* `BaseUnitAi.AdjustLoadAmount<T>(target, itemID, inputAmount) -> int` is **the real gate for "how much can be loaded"**.
  ⚠️ **Harmony cannot patch open generic definitions** — you must construct a closed instance:
  `open.MakeGenericMethod(typeof(lgBase.BaseUnitInfo))`.

### 12.5 Vanilla spawn mechanics (for comparison)

* `ConfigsTable.CollectionConfigDic` is a `Dictionary<String, List<CollectionData>>`, indexed by `BuildingTypeID.ToString()`.
* `BuildingInfo::AfterStart` rolls **per parking spot**:
  entries with `weight > 0` (listA) get **one weighted draw per parking spot** and then roll `dropProbability`;
  entries with `weight == 0` (listB) are rolled **independently**.
* The camp is hard-coded to `CampType.BCamp`; `CheckTileClear(0)` must pass; interior parking spots are skipped; the main base is skipped.
* Vanilla attaches vehicles to **only** two kinds of buildings: `50040 event location` (damaged pickup 10% + ISZ-695N bus 1%),
  and the 13 "vehicle refresh" categories (damaged pickup 10%).

### 12.6 Building IDs (measured)

| ID | Name | Use |
|---|---|---|
| **50040** | **Event Location (事件点)** | vanilla wrecks spawn here. **Note: this is not a "car crash site" (车祸点)** |
| 50041 | Event location (事件点) | the `DESCRIPTION` column says 「未启用」 (not enabled), and no spawn table references it → **do not use** |
| 90001 | **Car Accident Scene (车祸现场)** | a scene object with a description, **not confirmed to be a `BuildingInfo`** |
| 50100 | Crash Site (坠机现场) | |
| 50045 / 50055 | Airdropped supplies / clearing rubble (空投物资 / 清理废墟) | |
| 50085 / 50086 | Town clinic / infectious disease clinic (城镇诊所 / 传染病门诊) | hospital rule anchors |
| 50070 | Town train station main building (城镇火车站主站房) | train station rule anchor |
| 50089 / 50092 | Roadside parking lot (街边停车场) | part of the "vehicle refresh" group |
| Vehicle refresh 13 categories | 50006 wooden village house (木制村屋) / 50017 old brick apartment block (老式砖砌公寓楼) / 50018 brick garage (砖砌车库) / 50019 brick factory building (砖砌厂房) / 50021 village shop (村镇商店) / 50024 large shed (大型棚屋) / 50029 brick residence (砖砌住宅) / 50030 village school (村镇学校) / 50034 small brick warehouse (小型砖砌库房) / 50035 brick factory (砖砌工厂) / 50036 wooden warehouse (木制仓库) / 50089 roadside parking lot (街边停车场) / 50092 roadside parking lot (街边停车场) | |

---

## 13. Known limitations and TODOs

* **The F5 cycle has 43 vehicles**, so going all the way round takes 43 presses.
* **The event location rule counts per location** (explicitly requested by the user; no map-wide quota is added).
* **The 18 programmatically generated variants**: coverage is verified by `vehicle_specs.py`, and **when adding a new variant you must confirm that the parser recognises it**.
* **The release package version lagging behind** is a common state — remember to re-package after changing code (section 10).
* To be confirmed: whether `90001 car accident scene` can serve as an `AfterStart` trigger point (if it is not a `BuildingInfo`, a different attachment approach is needed).
* Lazy backups such as `*.old-*` / `*.locked_*` may be left in the plugin directory (BepInEx does not load them; they can be cleaned up).

---

## 14. Working conventions for whoever takes over (human or LLM)

1. **Run the verification matrix once before changing code** to get a baseline.
2. **Build immediately after the change + run the full verification matrix**; do not deploy if any single item is not green.
3. **PowerShell traps**:
   * `` `r`n `` inside a single-quoted string **does not expand** — it is written into the source as a literal (the compiler reports `CS1056 unexpected character`).
     Use double quotes when you need a real newline, or simply write the file with the `write` tool.
   * **Check the number of parentheses before matching with a regex** — for example `DeferredSetup(..., medical));` has **two** closing parentheses,
     and writing only one silently fails to match (after a change you must verify that the replacement really took effect, using `Select-String` to read what is on **disk**).
   * `Select-String -Path` reads from **disk**; if it is in the same script as `WriteAllText`, watch the order of execution —
     what it prints may still be the old content.
4. **Ask before changing configuration defaults** — the user explicitly requested "do not change the default ranges"; all tuning goes through the `.cfg`.
5. **`BepInEx` only writes the configuration on exit** — editing the `.cfg` while the game is running gets overwritten back on exit.
6. **You must confirm the game is closed before deploying.**
7. **Report in Chinese, and admit mistakes directly** — the user's style is brief and values real in-game verification; every mistake must be stated clearly.
8. **Do not invent game internal facts from intuition** — verify them with `il-query.ps1` / the CSV tables / asset extraction, and write the evidence into the comments.

---

## 15. Data asset extraction (vanilla data → JSON)

Changing values, investigating mechanics and building comparison tables all rely on this toolset. **UnityPy is already provided with `tools/pylibs`**, so
`sys.path.insert(0, r'D:\Codex Work\mod\tools\pylibs')` in a script is enough.

### 15.1 Basic procedure

```
① In globalgamemanagers.assets, walk the MonoScripts and use m_ClassName to find the target class → get its path_id
   (MonoScript's read_typetree() **is reliable**, unlike MonoBehaviour's)
② Walk every *.assets under the Data directory, take get_raw_data() for each MonoBehaviour,
   read the int32 at offset 20 (= the MonoScript's path_id) and match → a hit is the target instance
③ Parse the fields by manual offsets (see 15.3), or try read_typetree() first
```

### 15.2 scriptID reference table (measured)

| path_id | Class name | Content | Output |
|---|---|---|---|
| **838** | `CollectionConfigs` | building / squad spawn tables (27 entries) | `out/collectionconfigs.json` |
| **509** | `WeaponConfigs` | **weapon table**: 8 assets, **71 entries** in total | `out/weapons_full.json` |
| **709** | `FoodItemConfigs` | **items / ammunition**: 3 assets, **103 entries** in total | `out/items.json` |
| **315** | `ArmamentItemConfigs` | **weapon items**: 94 + 11 = **105 entries** | `out/armament_items.json` |
| **1374** | `BattleMemberConfigs` | **soldier roster**: 17 sub-assets, **58 entries** in total | `out/members.json` |

**The 8 assets of `WeaponConfigs`** (all named "XX Config", grouped by purpose):

| Asset name | Entries | Purpose |
|---|---|---|
| `GunsConfig` | 10 | **infantry firearms** (rifle / machine gun / sniper / flamethrower) |
| `ExplosiveIndividualWeaponsConfig` | 11 | **individual explosive weapons** (hand grenade / rocket launcher / grenade launcher) |
| `EmplacedWeaponConfig` | 10 | **emplaced weapons** (general-purpose machine gun / AA machine gun / anti-tank missile mount) |
| `ArmamentsConfig` | 17 | **vehicle-mounted / artillery** |
| `AirStrike` | 14 | **air strikes** |
| `InteralDataWeapon` | 8 | **internal data** (multi-ammo base types, smoke screen, artillery support) |
| `MeleeCombatConfig` | 1 | **melee** (only the single entry "brawling" (搏斗)) |
| `MineWeaponConfig` | 0 | mines (empty) |

**The 17 sub-assets of `BattleMemberConfigs`** (grouped by branch): `Rifleman(4)` `Grenadier(4)` `Machinegunner(4)`
`Sniper(4)` `Assault(2)` `Scout(3)` `CombatEngineer(5)` `SpecialForcesOperator(4)` `WeaponOperator(4)`
`SecondLineTroops(9)` `NonCombatant(7)` `Animal(8)` + several empty assets (58 entries in total; camp=1/2 each has a mirror copy)

### 15.3 MonoBehaviour header and serialisation rules (**must be remembered**)

```
offset  0–15   m_GameObject PPtr
offset 20      scriptID (int32) = the MonoScript's path_id   ← use this to match
offset 28      m_Name length (int32)
offset 32      m_Name content (UTF-8), then 4-byte alignment
offset ?       serialised fields begin (declaration order)
```

| Type | Bytes |
|---|---|
| `string` | int32 length + UTF-8 bytes, **4-byte aligned at the end** |
| `int` / `enum` / `float` / `bool` | **4 bytes each** (bool takes 4 bytes! not 1) |
| `PPtr<T>` (asset reference) | int32 fileID + int64 pathID = **12 bytes** |
| `T[]` / `List<T>` | int32 count + count×element, 4-byte aligned at the end |

**Nested structures (measured)**:

```csharp
FireNum            { int fireNumMax; int fireNumMin; }                    // 8 bytes
DamageSourceData   { int weight; DamageType damageType; }                 // 8 bytes
EquipmentData      { string itemName; int itemID; int itemNum; int useOrder;
                     List<WeaponType> replaceableTypes; }                  // List<enum> = int32 count + int32×n
```

### 15.4 Scan-based parsing (**the key technique for "unknown nested structures"**)

`WeaponData` has 39 fields, of which `damageSources` / `fireRateArray` / `fireNumArray` are nested arrays,
and **when the layout is unknown you cannot compute the start of the next entry**. I failed twice on this (parsed only 15 fields and then went on to read the next entry ✗).

**The correct approach**:

1. **Parse only the first N fields that have been verified with high confidence** (`WeaponData`: the first 15);
2. Then **scan forward byte by byte from the current offset** looking for the start of the next entry; a hit requires every criterion to hold:
   * `rd_str()` succeeds (reasonable length + all printable characters)
   * the int32 immediately after it is a **`weaponID` within a plausible range** (`0 < id < 200000`)
   * `weaponType` falls inside the enum range (`0..17`)
   * name length ≥ 2
3. On a hit, continue until the **parsed count == the entry count declared in the asset header** (self-check ✓).

The **criterion of this method is "parsed count matches declared count"** — if the counts agree, the whole chain is correctly aligned ✓

### 15.5 Five traps (all actually stepped on)

1. **`read_typetree()` often fails for MonoBehaviour** (the type tree is not embedded in Mono assets) —
   the first time, I silently did `except: continue`, which led to the **wrong conclusion** "asset not found".
   **Either print the exception, or go straight to manual offset parsing.**
2. **An empty string is legal** — every `m_Name` in `FoodItemConfigs` is an empty string,
   and I initially treated `len <= 0` as invalid and `raise`d, resulting in **0 entries for the whole table** ✗. `rd_str` must allow `n == 0`.
3. **Do not read integers as floats** — I parsed the `weight` (int) of `damageSources` as a float,
   and got malformed values like `1.4e-43`. **Read according to the declared type.**
4. **Do not assume which file an asset is in** — walk **all** `.assets` under the `Data` directory;
   an asset may also be inside a bundle (in which case a different method is needed).
5. **A GBK console destroys Chinese** — Python's `print` gets double-encoded into mojibake through a PowerShell pipe.
   **Have Python write the file itself with `io.open(..., encoding='utf-8')`, then read it with `Get-Content -Encoding UTF8`.**

### 15.6 `GameGlobalVariables` — where all numeric constants live

**Every global constant in v0.8.706 is written as a literal inside `GameGlobalVariables::.cctor` (the static constructor)**,
for example:

```
3314 ldc.r4  30
     stsfld GameGlobalVariables::<SecondsForEat>k__BackingField
```

**⚠️ Lesson: `il-query.ps1 -Xref` does not index `stsfld` / `stfld` writes.**
I once got 0 hits from `-Xref Calorie_Sufficient` and wrongly concluded "the value is in a data asset, so assets need to be extracted" ✗.
**When you get "0 xref hits", decompile `.cctor` or the initialisation method rather than concluding that the code does not contain it.**

---

## 16. Vanilla combat data quick reference

### 16.1 Weapon types `TypeManager.WeaponType` (19 entries, measured)

| Value | Name | Value | Name |
|---|---|---|---|
| 0 | Normal | 10 | Mine |
| 1 | **Grenade** (thrown hand grenade, only 2 entries) | 11 | **MG** |
| 2 | **ArtilleryShell** | 12 | **AssaultRifle** |
| 3 | **AP** | 13 | **SniperRifle** |
| 4 | **Belt** (ammo belt) | 14 | **Flame** |
| 5 | **HEAT** | 15 | Hypocenter |
| 6 | MISS | 16 | **Thermobaric** |
| 7 | None | 17 | **cqbWeapon** |
| 8 | **MeleeCombat** (melee, only "brawling" (搏斗)) | | |
| 9 | **RPG** | | |

### 16.2 `WeaponData` fields (**declaration order, 39 of them**)

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

**Correct reading of easily confused fields**:

| Field | Reading |
|---|---|
| **`weaponAccuracy` vs `staticAccuracy`** | the former is base/moving accuracy (actually in use); **`staticAccuracy` is 0 for every weapon** — it is not "an absolute value for stationary accuracy", and its semantics are unconfirmed |
| **`fireRate` vs `fireRateReadOnly`** | `fireRate = 0` means "determined by `fireRateArray`" (assault rifles); for sniper rifles the two are identical; **for twin-mounted weapons `fireRateReadOnly` = single-barrel value × number of barrels** |
| **`fireRateArray` / `fireNumArray`** | multiple fire modes. For example an assault rifle `[150, 600]` + `[(6,1), (6,2)]` = 150 single shot / 600 burst, 6 rounds per trigger pull |
| **`damageSources`** | a `[(weight, damageType)]` list — the damage composition. Hand grenade `[(100,1)]`; QQB-9 thermobaric `[(80,1),(100,16)]` |
| **`sceneDamage`** | **destructive power against buildings/fortifications** — the core indicator for telling weapon roles apart: rocket artillery 800 / aerial rocket 600 / 125mm gun 300 / AA gun 40 |
| **`deviationRange`** | spread. **0 for every infantry firearm**; only artillery/rockets/grenades are non-zero |
| **`shockedBuffHit`** | **suppression value** (see 16.5) |

### 16.3 Multi-ammo mechanic: `isMultiAmmoWeapon` + `parentWeapon`

* **`isMultiAmmoWeapon` is only read by the UI**: `WeaponInfoCellUI::Start`, `UnlockEntryCell::Unlock` ✗ combat code does not read it
* **`parentWeapon` is read by `WeaponInfoCellUI::OnDropdownWeaponValueChanged`** — it is **the dropdown in the weapon information panel**
* So multi-ammo = **splitting one weapon into several `WeaponData` entries, with the child entries pointing at the base type via `parentWeapon`**,
  where the base type lives in `InteralDataWeapon` and has `bulletID = 0`:

| Base type | `multi` | Child entries | Ammo types |
|---|---|---|---|
| **10073 BRA-7 individual rocket launcher (单兵火箭筒)** | True | 10155 / 10156 / 10157 | HE / HEAT / RPO |
| **10280 BRA-29 heavy individual rocket launcher (重型单兵火箭筒)** | True | 10281 / 10282 | HEAT / RPO |
| 10085 QQB-9 (item `10285`) | — | 10286 / 10287 | HEAT / HE |

**Of all 71 weapons only these 2 base types have `multi = True`** ✓ — **even vehicle-mounted guns do not enable it**:
`10111 125mm滑膛炮` and `10112 125mm穿甲弹`, `10334/10335 57mm穿甲/高爆` are all **independent entries**,
which is another way of doing "switching ammo type" ✓

⚠️ **Do not use the 6-parameter `SpwanArms`** (see §5.4) — unrelated to multi-ammo, but often confused with it.

### 16.4 Reloading and weapon swapping: `EquipmentData.replaceableTypes`

```csharp
// MemberData.weaponList / .equipment are both EquipmentData[]
EquipmentData { string itemName; int itemID; int itemNum; int useOrder;
                List<WeaponType> replaceableTypes; }   // ← which weapon types this slot may be replaced with
```

**An empty `replaceableTypes` = this slot cannot be re-equipped** ✓. Measured:

| Soldier | Slot | Weapon | `replaceableTypes` | Meaning |
|---|---|---|---|---|
| 20002 rifleman (步枪手) | primary weapon | 10019 | **(empty)** | cannot be swapped |
| 20001 machine gunner (机枪手) | primary weapon | 10175 | (empty) | cannot be swapped |
| 20004 sniper (狙击手) | primary weapon | 10021 | (empty) | cannot be swapped |
| 20038 **special forces NCO (特战队士官)** | primary weapon | 10269 | **[11]** | can be swapped for a machine gun |
| 20039 **special forces officer (特战队军官)** | primary weapon | 10269 | **[11, 13]** | can be swapped for a machine gun / sniper rifle |
| **20038/20039** | **secondary weapon** | **10074 CB-25** | **[9] = RPG** | **this slot can switch between the CB-25 and rocket launchers** |
| 20040 flamethrower operator (喷火兵) | primary weapon | 10251 | [16] = Thermobaric | can be swapped for thermobaric |
| 20016 grenadier (掷弹兵) | secondary weapon | 10156 / 10155 | [16, 2, 9] | can be swapped for thermobaric / artillery shell / rocket |

**Key example: `10074 CB-25枪挂式榴弹发射器` has no item entry** (it is not among the 105 `ArmamentItemConfig` entries ✗)
→ **it cannot be purchased**, and can only appear through the **roster** in the secondary weapon slot of **`20038 special forces NCO` / `20039 special forces officer`** (5 rounds each) ✓
This is the correct reading of "under-barrel": **additional firepower attached to the suppressed assault rifle (10269), not standalone equipment**.

**Ammunition counts are written in `itemNum`** (rifleman 180 rounds, machine gunner 300 rounds, sniper 100 rounds) ✓

### 16.5 Suppression mechanic

* Weapon side: **`shockedBuffHit`** — explosive weapons (except hand grenades) are **all 5**, hand grenades 0
* Unit side: **`BaseUnitAi.unitIsSuppressed`** (`get_unitIsSuppressed`)
* Effect: **interrupts behaviour**. A confirmed example — while suppressed a unit **cannot go to the canteen (食堂) to eat**
  (one of the gates in `SafeToDiningInBase`)

### 16.6 Nutrition and canteen mechanics

**Three-layer structure**: ration standard (`RationData`) → canteen area (`Area 60005`) → unit dining state

**① Nutrition state**: `BaseUnitInfo.nutritionData` is a `Vector3`
= **(x calories, y protein, z vitamins)** — graded separately by `GetCalorieStatusLevel(x)` / `GetProteinStatusLevel(y)` /
`GetVitaminStatusLevel(z)` ✓

**② Four-level thresholds** (values from `GameGlobalVariables::.cctor`):

| Nutrition axis | Deficient | Low | Sufficient | Level |
|---|---|---|---|---|
| **Calories** | **2400** | **3200** | **4000** | ≥4000→4 · ≥3200→3 · ≥2400→2 · <2400→1 |
| **Protein** | **60** | **100** | **120** | same structure as above |
| **Vitamins** | **25** | **40** | **80** | same structure as above |

(each axis also has a `_Max` variant, assigned before `Sufficient`, used for capping the UI)

**③ Eating cycle: `SecondsForEat = 30`** — note that the name says "Seconds" but it is passed to
`DateTime::AddMinutes`, so **it is actually in minutes** ✓, and `GameTimeToRealTimeScale = 1.0` (game time = real time)

**④ Trigger conditions (it is not "eat when hungry", it is a timer + gate checks)**

```
every 30 minutes → SafeToDiningInBase() gates:
    infantry (isArmament == false) · GameObject active · not moving · not in combat · not suppressed
        ↓ all pass
DiningInIdelState coroutine → nextEat = Now + 30 minutes (interrupted by movement/combat/suppression during that period)
        ↓
DiningAtBase() —— must be standing on a tile where inFobRange != null (otherwise false)
                 → filter food from FOB.SupplyStockDic → eat → AddIntake() accumulates nutritionData
```

**⑤ Data tables**: `ConfigsTable.FoodDataDic` (food) · `AmmoDataDic` (`FoodData`, ammunition reuses that structure) ·
`FoodItemConfigs` (asset) · `FoodNutritionManager` (**the custom meal-planning UI**, not the eating logic)
Nutrition conversion basis: staples **per 100g**, meat/vegetables **per 50g**, fats and oils **per 10g**, snacks/drinks **per 50g** ✓

### 16.7 Numeric examples (for direct comparison)

| Weapon | ID | Damage | Scene | Penetration | Range | Accuracy | Magazine | Reload | Velocity |
|---|---|---|---|---|---|---|---|---|---|
| 5.45mm Type 74 assault rifle (5.45mm74型突击步枪) | 10019 | 3 | 3 | 1 | **0–12** | 60 | 30 | 4 | 16.46 |
| 7.62mm Type 79 sniper rifle (7.62mm79型狙击步枪) | 10021 | 5 | 5 | 1 | **0–24** | 90 | 10 | 3 | 25.71 |
| **7.62mm Type 79 (suppressor) (抑制器)** | 10270 | **3** | 3 | 1 | **0–8** | 90 | 10 | 3 | **18.29** |
| 12.7mm MQ2 anti-materiel (反器材) | 10273 | 12 | 20 | 2 | **0–36** | 100 | 10 | 3 | 35.27 |

**The design rule of the suppressed version**: **only damage / range / velocity are cut; accuracy, rate of fire, aiming, magazine and reload are all unchanged** ✓

Item encoding: weapon items have `itemType = 4096` (2¹²), and `subItemType` is either **16384** (2¹⁴) or **16777216** (2²⁴), two sub-class flags ✓

---

## Changelog

| Version | Summary |
|---|---|
| v5.68.2 | Valena (瓦莲娜) hidden (`Specs[]` back to 43); installer repackaged |
| v5.68.1 | `Hidden` field (reserved, not enabled) |
| v5.68.0 | Valena reset to twice the size of a 44×64 truck (pivot measured with `measure_pivots`) |
| v5.67.0 | **Fixed duplicate spawning on save load** (bookkeeping via `mapArmsSpawnActivedList` added to three rules) |
| v5.66.2 | Valena pivot corrected to the values measured by `audit_pivots` |
| v5.66.0 | Valena changed into a **vehicle** (abandoned the infantry squad path); F6 unit cycle disabled |
| v5.63.1 | Colour suffix removed from multi-colour vehicle names (21 places) |
| v5.63.0 | Random facing (`UnitDirection`) + same-cell stacking offset (`StackOffset`) |
| v5.62.0 | All spawned vehicles start with 0-30.0 L of fuel (`SetSpawnFuel`, clamped by capacity) |
| v5.61.1 | City road changed to a **map-wide** quota (2-3 vehicles); fixed the `ProcessPendingSpawns` early-return guard |
| v5.61.0 | City road → heavy emergency vehicles (2-3, durability 0-70%, 200-1000 rounds of loose 5.45) |
| v5.60.0 | Event location rule (3-7 wrecked vehicles 0-30% + 0-3 emergency rescue); at most 3 per cell |
| v5.59.0 | `vehicle_specs.py` unified parser; fixed two bugs — pool Key mismatch and the `Pools[0]` fork |
| v5.58.0 | ZIL-4310 civilian 6 variants (cargo cap ×5) |
| v5.57.0 | B1000 / W50 6 variants each + onboard cargo system (`CargoEntry`) |
| v5.56.0 | Vehicle refresh points → civilian light vehicle pool (20% per parking spot) |
| v5.55.0 | Part durability randomisation changed to run after waiting for `armInitialized` (fixed the "brand-new parts" BUG) |
| v5.54.0 | Hospital / train station changed to a **map-wide** quota (fixed "4 trucks + 4 ambulances") |
| v5.53.0 | Spawn position changed to free tiles on the building's outer ring (fixed "vehicles parked on the roof") |
| v5.51.0 | Train station rule; anti-infection drug 10009 removed from the medical list |
| v5.48.0 | Hospital ambulance rule |
| v5.47.0 | 4 vehicle pools |
