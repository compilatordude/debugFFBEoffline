# Exvius Offline — findings compendium

Everything discovered while modding **Exvius Offline v0.1.25 / v0.1.30** on a HONOR phone and a
MuMu x86 emulator: file formats, point/trust/EXP mechanics, the summon-rate data, the crash we
chased, and the exact addresses involved.

Companion documents: `TUTORIAL.md` (step-by-step Windows commands), `ITEMS-NOTES.md`,
`EXP-UNITS.md`, `POTS.md`, `SUMMON-RATES.md`, `AWAKENING-COMPARISON.md`,
`AWAKENING-MATERIALS-ANOMALIES.md`, `FUNDAMENTAL-FORCES-AWAKENING.md`.

---

## 1. Environment

| | |
|---|---|
| Game | Exvius Offline (fan project of Final Fantasy Brave Exvius, Global data) |
| Upstream | `github.com/Elvlin/exvius-offline` |
| Package | `com.exvius.offline.test` |
| Launcher activity | `com.exvius.local.DataDownloadActivity` |
| Game activity | `com.square_enix.android_googleplay.FFBEWW.Lapis` |
| Phone | HONOR MTN-NX1, Android 16 (SDK 36), 1200×2640 |
| Emulator | MuMu Player 15 (Android 15), x86_64 with `libhoudini` ARM translation |
| Manifest | `https://raw.githubusercontent.com/Elvlin/exvius-offline/main/data/latest.json` |
| Manifest contents | revision 9, version `2026.11.04.9`, 19 packs, 13.64 GB |

### Versions and hashes

| Version | versionCode | versionName | Size | SHA-256 |
|---|---|---|---|---|
| v0.1.25 | 9065 | `9.0.0-offline.25` | 47,911,174 B | `3625b1e6c423d1c4dfdb54172d8f512444314bd2dd9065d67e32adb45061b531` |
| v0.1.30 | 9070 | `9.0.0-offline.30` | 101.4 MB | `98875e7b8ebcd04794b36392f8afa71c207bebeba2f90d939ee9572f034e95e2` |

Our re-signed builds use key `release.jks`, alias `exvius`, password `exvius123`; certificate
SHA-256 `ccf89ccfc6e6066accfa0e39b7e1a5c6b15488ceab24c45f61ff1eaf5eebf45a`.

---

## 2. The APK patch set

Four changes, all that separate a stock APK from our build:

| # | Where | Change |
|---|---|---|
| 1 | `AndroidManifest.xml` | `android:debuggable="false"` → `"true"` |
| 2 | `AndroidManifest.xml` | add `android:usesCleartextTraffic="true"` to `<application>` |
| 3 | `smali_classes6/com/exvius/local/DataUpdates.smali` | `LATEST` field + two `const-string` (3 occurrences): `https://raw.githubusercontent.com/Elvlin/exvius-offline/main/data/latest.json` → `http://192.168.0.11:8080/latest.json` |
| 4 | same file | `ASSETS` prefix field + two `const-string` (3 occurrences): `https://github.com/Elvlin/exvius-offline/releases/download/` → `http://192.168.0.11:8080/` |

Why #4 matters: the client validates every pack URL with
`url.startsWith("https://github.com/Elvlin/exvius-offline/releases/download/")` before accepting a
manifest; anything else throws **“Invalid data pack information”**. Patching that constant to the LAN
prefix is what makes local hosting legal for the client.

`debuggable=true` is what enables `run-as` — the whole save-editing path depends on it.

Build: `apktool 3.0.3` (Java 25), `zipalign -p 4`, `apksigner` (build-tools 36.0.0).
Install over an existing build with `adb install -r` — data survives when the signing key matches.

---

## 3. Save file anatomy

```
/data/user/0/com.exvius.offline.test/files/exvius/state/campaign.json
```

- Plain JSON, **no checksum, no signature, no encryption**.
- Owner `u0_a315`, mode `600`, SELinux `u:object_r:app_data_file`.
- `"version": 2` (unchanged by the 0.1.25 → 0.1.30 update).
- Game snapshots to `files/exvius/state/backups/`.
- Sizes seen: 6,606 B (fresh new game), 504,679 B (mid-game), 708,273 B (late game, 1,203 units).

Top-level keys: `version, meta, identity, rank, currencies, profile, emblem_event, counts, orbs,
roster, vision_cards, party, inventory, mailbox, login_bonus, catchup, anniversary, progress, beasts,
beast_decks, gacha, records, notices, quest_progress, daily_quest, exvius_repeat,
exvius_resource_recovery, bundle_purchases, exvius_unit_points, summon_archive, exvius_craft,
exvius_vision_card, event_archive, exvius_trophy`.

### Editing mechanics

- Pull: `run-as … base64 files/exvius/state/campaign.json` (base64 avoids binary corruption on stdout).
- Push: `adb push` to `/data/local/tmp/`, `chmod 644`, then
  `run-as … cp /data/local/tmp/x files/exvius/state/campaign.json`.
  (`run-as` cannot write to `/data/local/tmp`, but it can read world-readable files there.)
- Always `am force-stop` first, relaunch after.
- The game rewrites the file in its own pretty-printed format on the next save; compact JSON is fine.

---

## 4. Currencies

```json
"currencies": { "lapis_free": …, "lapis_paid": …, "gil": …, "friend_point": … }
```

- `lapis_free` is the gameplay bucket, `lapis_paid` the store bucket.
- **Lapis display truncates to 7 digits**: with `999,999,999` stored the HUD shows `9,999,999`
  (`999999999 mod 10^7`). Spending still uses the true value. Practical readable maximum ≈ 9,999,999.
- Gil renders 10 digits fine (`1,024,801,085`).
- No cap enforced on load; arbitrary values accepted.

---

## 5. Items

```json
"inventory": {
  "items":     [ { "item_type": 20, "item_id": 106305900, "count": 10000 } ],
  "carry_items": [],
  "equipment": [ { "item_id": 302000100, "count": 2 } ]
}
```

- `F_ITEM_MST.ndjson`: **4,729 rows**; `F_TEXT_ITEM_NAME.ndjson`: **3,435 English names**, keyed
  `MST_ITEM_NAME_<item_id>`.
- Stack cap column: `9J6uRe8f` (999 for most, 99999 for currency-like items).
- `item_type` observed: `20` normal stackable, `22` vision-card-ish, `23` important/ticket
  (e.g. `300` = Trust Coin).
- **Master cap is not enforced on load** — 5,000 and 99,999 were accepted for pearls.

### Useful ids

| Category | ids |
|---|---|
| Awakening crystals | `290050100` Esper Shard, `290050200` Esper Cryst, `290050300` Sacred, `290050400` Holy, `290050500` Divine |
| Pearls | `1209002041` Azure, `293000100` Transcension, `293000200` Ascension |
| Unit fragments | `310001240` Fundamental Force, `310000890` Frostblade Fryevia, `310001230` Eithne |
| Metal units | `900020101` Minituar, `900020201` Cactuar, `900020301` Gigantuar, `900020401` King Minituar, `306000804` Liquid Metal Slime, `902000505` Eraser Gigantuar |
| Trust moogles | see §7 |

---

## 6. Roster / units

```json
{
  "instance_id": "1794275752126", "master_id": "401016407",
  "level": 102, "exp": 263317, "total_exp": 4962804, "lb_exp": 0,
  "nv_ex_level": 3, "guest": 0, "storage_state": 0, "favourite": 0,
  "granted_rarity": "7", "first_get_unit_id": null, "lb_level": 1,
  "equipment": {"1":"303001300"}, "materia": ["1500000265","0","0","0"],
  "vision_card": "0-1794292829641",
  "affinity_points": ["1000","1000"], "affinity_info": ["",""],
  "shift_equipment": {"1":"0"}, "shift_materia": ["0","0","0","0"],
  "skin_id": 0, "stat_pots": [0,0,0,0,0,0], "stat_doors": [0,0,0,0,0,0],
  "affinity_reward_claimed": ["1:22:1500000265:1","0:21:1100000828:1"]
}
```

- `instance_id` is epoch-milliseconds, numeric string, unique. New units: max + 1.
- `counts.unit` must equal `roster.length`.
- `granted_rarity` may be `null` or a number/string — the client tolerates both.
- **Auto-exchange on load**: material units are converted into the point bank and removed.
  Rule (`exvius_actions_shops.py::convert_material_units`):
  - `useType` = `2odVXyT8` must be `1`;
  - role `9` in `27GkYdEu` → credits `unit_mix_exp()` to EXP points;
  - roles `12`/`13` → credits `Y2X8d7R7`-based stat/LB points (only on explicit exchange request);
  - **skipped when `favourite` is set** or the unit is deployed in a party.
  Set `"favourite": 1` on an added material unit to make it persist.

---

## 7. Trust Moogles — the per-instance payload

Trust value is **not** in `F_UNIT_MST`; it lives on the roster entry:

```json
"affinity_points": ["", "1000"],
"affinity_info":   ["", "70:0:0:1000:0:0:0"]
```

From the mod's own `player_state.py::grant_unit`:

```python
moogle = {'904000101': ['10',  ''],    # 1%
          '904000104': ['50',  ''],    # 5%
          '904000105': ['100', ''],    # 10%
          '906000104': ['', '500'],    # 50%  (channel 1)
          '906000105': ['', '1000']}   # 100% (channel 1)
```

- Scale is 1000 = 100%.
- `904…` family → **channel 0**; `906…` family → **channel 1**.
- `affinity_info` must be the wildcard `70:0:0:1000:0:0:0` on any channel whose points > 0,
  otherwise the client ignores the instance payload.
- Empty `affinity_points`/`affinity_info` = inert moogle (this was the initial mistake).
- Unit type flag `a9qg8vnP` = **8** for moogles; max level 1.

Master ids: `904000101/103/104/105` Trust Moogle (1★/3★/4★/5★), `906000103` Super (unit-specific),
`906000104` Super, `906000105` Super (ALL), `904000115` Prism Moogle.

---

## 8. EXP units

Yield formula (`player_state.py::unit_mix_exp`, mirrors `GameUtils::getMixExp`):

```
EXP = (isExpUpUnit ? total_exp : total_exp / 2.5) + F_UNIT_MST.N85uJT1d + rarityBonus
rarityBonus = 100/200/500/1000/1500/3000/5000 for rarity 1..7
```

| Unit | master_id | rarity | N85uJT1d | + bonus | EXP |
|---|---|---|---|---|---|
| Metal Minituar | `900020101` | 1 | 4,900 | 100 | 5,000 |
| Metal Cactuar | `900020201` | 2 | 9,800 | 200 | 10,000 |
| Metal Gigantuar | `900020301` | 3 | 29,500 | 500 | 30,000 |
| King Metal Minituar | `900020401` | 4 | 99,000 | 1,000 | 100,000 |
| Liquid Metal Slime | `306000804` | 4 | 2,999,000 | 1,000 | 3,000,000 |
| Eraser Gigantuar | `902000505` | 5 | 0 | 1,500 | 1,500 |

Our edit set the four metals to **393,000,000 each** (`N85uJT1d` = 392,999,900 / 800 / 500 / 000),
leaving Liquid Metal Slime and Eraser Gigantuar untouched.

Master table path (device):

```
/data/user/0/com.exvius.offline.test/files/exvius/data/static/mst_decoded/F_UNIT_MST.ndjson
```

Original 4,585,646 B / 3,313 rows; edited 4,588,977 B. Backup `F_UNIT_MST.ndjson.bak`.

---

## 9. Pots and the unit point bank

Pots are material units (`2odVXyT8 = 1`, role `12`, max level 1) exchanged into the point bank.

### Burst Pots (LB) — value in `1Rf2VDoI`

| Unit | master_id | LB points |
|---|---|---|
| Mini Burst Pot | `905000102` | 100 |
| Burst Pot | `905000103` | 3,000 |
| King Burst Pot | `905000104` | 10,000 |

### Stat Pots — value in `Y2X8d5R7` (HP/MP/ATK/DEF/INT/MND)

| Family | Stat | Mini | Pot | King |
|---|---|---|---|---|
| `900000101/201/301` | HP | 10 | 30 | 50 |
| `900001101/201/301` | MP | 5 | 10 | 15 |
| `900002101/201/301` | ATK | 1 | 5 | 10 |
| `900003101/201/301` | DEF | 1 | 5 | 10 |
| `900004101/201/301` | INT | 1 | 5 | 10 |
| `900005101/201/301` | MND | 1 | 5 | 10 |

### Bank tokens (`exvius_unit_points.banks["0"]`)

`XeRVw3s2` EXP · `jWS57HG3` LB · `LD4UJaf0` HP · `ui34MYgo` MP · `5sNq90o6` ATK · `0xgsDY9E` DEF ·
`VSL4Wj9z` INT · `y7Z5AmhC` MND · `h5s68EtJ`/`eu2DXK4I`/`8RUQtk4i`/`8cj19yhb`/`rjK5oB1H`/`16NCtrGw`
MAX_LIMIT_EX stats.

### Capacity tests

| Written | Stored | UI |
|---|---|---|
| 2,147,483,647 | same | — |
| 3,000,000,000 | same | — |
| 1,000,000,000,000 | same | `9,999,999,999 +` |
| 1,000,000,000,000,000 | same | — |
| 9,223,372,036,854,775,807 (int64 max) | same | `9,999,999,999 +` |

No clamping — 64-bit storage. Display caps: **EXP/LB `9,999,999,999 +`**, **stat materials `9,999 +`**.
All 13 pot tokens were set to **9,000,000,000,000,000,000** (headroom 2.23×10¹⁷).

⚠ At int64 max any further credit wraps negative; keep headroom.

---

## 10. Awakening

### Ordinary class-up — `F_UNIT_CLASS_UP_MST.ndjson`, 1,728 rows

Most rows use the crystal ladder (`290050100`…`290060400`). **Exactly 20 rows** use Azure Pearl and
they are all `…07 → …27` at 3,000,000 gil + **300× Azure Pearl** (`1209002041`):

```
100043107 Lasswell -Grim Beast-      100043607 Dark Fina -Grim Beast-
100044807 Crimson Knight Rain         100044907 Azure Knight Lasswell
100045007 Tyvas                      100045107 Rhus
207002307 Tifa (Avalanche)           216000107/207/307/407 Clive, Jill, Cidolfus, Benedikta
331001507/707 Maria, Fye             339000907 Stell
401016107 Blood Mage Zyrus           401016207 Winged Officer Celeste
401016407 Fundamental Forces         401016507 Eithne
401016607 Imperial Shield Wilhelm    401016707 Blade Breaker Esther
```

### NV EX — `F_NV_EX_CLASS_UP_MST.ndjson`, 1,528 rows / 510 units

| EX | gil | fragments | pearl |
|---|---|---|---|
| 1 | 250,000 | 50 | 1 |
| 2 | 500,000 | 100 | 2 |
| 3 | 750,000 | 200 | 2 |

Pearl split: 482 units use **Transcension** (`293000100`), 102 use **Ascension** (`293000200`) and also
need 1 Transcension at EX+3, 28 units need no pearl at EX+3. One dummy row uses unit id `1`.

### Worked examples

| | Fundamental Forces | Frostblade Fryevia | Eithne |
|---|---|---|---|
| master_id | `401016407` → `401016427` | `401013007` → `401013017` | `401016507` → `401016527` |
| internal | 8452 (8453/8454/8455) | 8375 → 8376 | 8456 → 8458 |
| max level | 120 → 130 | 120 | 120 → 130 |
| EXP pattern | 35 → 40 | 35 | 35 → 40 |
| class-up | 3M gil + 300 Azure | none | 3M gil + 300 Azure |
| fragment | `310001240` | `310000890` | `310001230` |
| stat block EX1 | 651,45,**117**,29,21,27 | 650,38,74,26,**77**,30 | 696,60,17,30,**117**,36 |

### EXP to max

| Pattern | cap | total EXP | milestones |
|---|---|---|---|
| 35 | 120 | **70,063,057** | L100 3,648,203 · L120 70,063,057 |
| 40 | 130 | **392,088,803** | L120 70,063,057 · L130 392,088,803 |

The 120→130 stretch alone is 322,025,746 EXP. `F_UNIT_EXP_PATTERN_MST` stores **per-level steps**
(`93fnYUJG`), summed 2→N; max level is `F_UNIT_MST.aU3o2D4t`.

---

## 11. Summon rates

Files live in the APK at `assets/rules/` (not on the data partition):

| File | Banners | Shape |
|---|---|---|
| `gacha_community.json` | 369 | per-unit pools with explicit rates |
| `gacha_catalogue.json` | 273 | campaign → shipped banner id (no pools for many) |
| `gacha_named.json` | 90 | named banners |
| `gacha_standard.json` | 8 | FREE-tab, tier rates |
| `gacha_authored.json` | 9 | authored, tier rates |
| `gacha_raid.json` | 71 | **new in 0.1.30**, no rate entries |

820 banners total, 31,637 pool entries. Two entry shapes:

```json
{"unit_id": "100002103", "rarity": 3, "rate": 0.026444444444444448}
{"rarity": 7, "rate": 0.03, "units": [...], "featured": [...], "featured_share": 0.3333}
```

- Rarity values present: 1, 2, 3, 4, 5, **7** (NV).
- **NV exists only as tier entries** — 227 of them (community 123, named 77, catalogue 26, standard 1).
- NV tier rate median **3%**; `featured_share` values 1/3 (82), 0.8 (29), 2/3 (45), 0.5 (26), 1.0 (11),
  0.2 (8), 0.0 (26).
- **No NV+ (EX-form) ids in any pool** — NV+ is produced by awakening only.

### Our edit (v0.1.30)

1. Every pool rescaled so rarity ≥5 totals **30%** (rest scaled to 70%).
2. `featured_share` set to **0.40** on every rarity ≥5 tier that lists featured units.

Results: 714 banners scaled, 21 already ≥30% left alone, 1 with no 5★; 588 featured tiers updated.

Per-banner effect depends on tier layout — `9200008` (Hero Resurrected Taivas) has NV 10% + 5★ 20%,
so its featured unit shows **4%**, not 12%; single-NV-tier banners (e.g. `9100009`) show **12%**.

v0.1.30 vs 0.1.25: `gacha_community` pools identical (0 changes), `gacha_named` 6 banners changed,
`gacha_raid` new.

---

## 12. Crashes

### 12.1 World-map crash (upstream issue #31)

Signature on ARM (phone/emulator tombstones):

```
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0000000000000058
pc 0x2b19f7c  libgame.so   cocos2d::Texture2D::setAliasTexParameters()+36
```

| item | value |
|---|---|
| Function symbol | `_ZN7cocos2d9Texture2D21setAliasTexParametersEv` |
| Function address | `0x2b19f58` (size 192 bytes) |
| Crash pc | `0x2b19f7c` (+36); reported as `+32` in one tombstone |
| Caller frame #01 | `0x3bf2cb4`, module base `0x399c000` |
| Fault | `this` pointer was `0x8`, `ldrb w8,[x0,#0x50]` → 0x58 |
| BuildId | `d810bec04d71db5834ceb6f8b172b3b99368ec1a` |

Tombstones captured: `tombstone_00` (2026-11-09 21:08:17, pid 1874, GLThread 41),
`tombstone_01` (21:57:09, pid 2926, GLThread 32), `tombstone_02` (22:42:59, patched build),
`tombstone_05` (reproduced deliberately). All `fault addr 0x58` on stock builds.

On MuMu the same crash surfaces inside `libhoudini.so`
(BuildId `9534e5d5bc1633fd1d9ebfcb40d3a1acec53a177`) at pcs `0x10f30d`, `0x1d5b7b`, `0x0f1a7e`,
`0x31c645` — no ARM frames, because the fault happens in translated code.

### 12.2 The 8-byte patch we tried

At file offset **`0x2b19f78`** (`libgame.so`, vaddr == file offset for the text segment):

```
original : 08 40 41 39  a8 03 00 34     ldrb w8,[x0,#0x50] ; cbz w8, +0x74
patched  : 1f 04 00 f1  a3 03 00 54     cmp  x0,#0x1000    ; b.lo 0x2b19ff0
```

Effect: the crash moved from fault `0x58` to fault **`0x38`** (tombstone_02) — the null texture is
dereferenced again later in scene setup, so guarding the one call is not enough. Reverted; restored
`libgame.so` SHA-256 `313b9d3b5c9ea19ac1c4e8b8ec591fe5a49d565b6f849509c7c5d7eed0becdb3`.

### 12.3 What the crash actually is

The world-map scene applies texture parameters to a `Texture2D*` that resolved to nothing. On the
**phone (ARM) the world map opened fine** when tested directly via adb, so the emulator reproduces it
reliably (Houdini) while the phone did not in our test. Issue #31 remains open upstream; v0.1.30's
notes do not mention it.

### 12.4 Other app log findings

```
I/ExviusTest: RULES_ASSET_MISSING rules/save_profile.json
I/ExviusTest: SAVE_LOADED /data/data/com.exvius.offline.test/files/exvius/state/campaign.json
I/ExviusTest: state: live slot='campaign' -- rank 16, 908 unit(s), ...
I/ExviusTest: unit-exchange: credited points or gil for material(s): 1
I/ExviusTest: OFFLINE_NETWORK_SOCKET_BLOCKED
```

`unit-exchange` is the auto-conversion of material units (§6). `OFFLINE_NETWORK_SOCKET_BLOCKED`
confirms the client is intentionally offline-only.

---

## 13. Limits, risks and gotchas

| Topic | Behaviour |
|---|---|
| Save integrity | No checksum — edits load directly |
| 64-bit fields | int64 max accepted; additions overflow to negative at the ceiling |
| Display truncation | lapis 7 digits; EXP/LB 10 digits `+`; stat material counters 4 digits `+` |
| Data re-extraction | Master-table edits (`F_UNIT_MST`) persist across app restarts but will be lost on reinstall / “repair game assets” |
| APK updates | Same key → data preserved; stock APK → signature mismatch, requires uninstall (wipes save + 14 GB) |
| LAN pack hosting | Requires patched `ASSETS` prefix + cleartext; IP `192.168.0.11` is hard-coded in the build |
| apktool rebuild | Dex and resources are recompiled; not byte-identical to stock even with no functional change |
| Auto-exchange | Material units vanish unless `favourite: 1` |
| `counts.unit` | Must match roster length after edits |

---

## 14. What v0.1.30 fixed (official notes)

Awakening crashes (incl. Elena NV), prism/Omniprism use in awakening, batch Trust Moogle fusion,
summon pools for individual banners, older translated JP summons, raid summon & Multi-Summon,
Dread Frost destination, King Mog's Exchange + Home crashes, expedition limits/daily resets, esper
board unlocking/resets, quest-progress on leaving towns, Magic Key + Festival Robe+/Cap+ recipes,
Chamber of Supercite rewards, mailbox pagination, Vortex categories, season filtering/ordering,
Exchange Shop filtering, 222 missing English names.

Still open: world map crash (#31), summon crash (#39), Ordoll Port crash (#36).

---

## 15. Artifacts and backups

| Item | Location |
|---|---|
| APKs (stock, debuggable, LAN, LAN+rates) | `ExviusOffline\` and `…\apk_patch\` |
| Signing key | `ExviusOffline\release.jks` |
| Save backups | `campaign.before_*.json` on device + archive |
| Original gacha JSONs | `D:\ExviusOfflineServer\gacha_backup_0130` |
| Original unit table | device `…/mst_decoded/F_UNIT_MST.ndjson.bak` |
| Rate diff CSV | `ExviusOffline\gacha_rates_diff.csv` (820 rows) |
| LAN manifest | `ExviusOffline\latest.lan.json` |
