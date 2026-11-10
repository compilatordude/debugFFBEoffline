# Exvius Offline — data-editing tutorial (Windows)

Step-by-step commands to do everything described in `COMPENDIUM.md`: read and rewrite the save,
add units/items, boost EXP units, max the point bank, and rebuild the APK with edited summon rates.

Written for **Windows PowerShell** (stock `powershell.exe` is fine; no PowerShell 7 needed).
Phone must be connected with USB debugging authorised.

---

## 0. Prerequisites

| Tool | Where to get it | Example path used here |
|---|---|---|
| `adb` | Android platform-tools, or MuMu's copy | `C:\Android\platform-tools\adb.exe` |
| Python 3 | python.org | `python` on PATH |
| Java (JDK/JRE 17+) | Android Studio's JBR works | `…\Android Studio\jbr\bin\java.exe` |
| `apktool.jar` | github.com/iBotPeaches/Apktool releases | `C:\tools\apktool.jar` |
| `zipalign.exe`, `apksigner.bat` | Android SDK `build-tools\<ver>\` | `…\Android\Sdk\build-tools\36.0.0\` |
| Re-signed/debuggable APK | see §7 | `exvius-offline-v0.1.30-lan.apk` |

The whole save-editing path **requires the debuggable build** — `run-as` refuses on the stock APK.

Start every session with:

```powershell
$ADB = "C:\Android\platform-tools\adb.exe"
$PKG = "com.exvius.offline.test"
$STATE = "files/exvius/state/campaign.json"

& $ADB devices
# expect: <serial>   device     (not "unauthorized")

& $ADB shell "run-as $PKG id"
# expect: uid=10xxx(u0_aXXX) ... context=u:r:runas_app:s0
```

If `run-as` says *package not debuggable*, you are on the stock APK — install your patched build
first (§7).

---

## 1. Helper functions (paste once per PowerShell window)

```powershell
function Get-Save {
  param([string]$Out = "$PWD\campaign.json")
  & $ADB shell "am force-stop $PKG"
  Start-Sleep -Milliseconds 800
  $b64 = (& $ADB shell "run-as $PKG base64 $STATE") -join ''
  if ($b64.Length -lt 100) { throw "pull failed: $b64" }
  [IO.File]::WriteAllBytes($Out, [Convert]::FromBase64String($b64.Trim()))
  "pulled $($Out) ($((Get-Item $Out).Length) bytes)"
}

function Set-Save {
  param([string]$In = "$PWD\campaign_new.json")
  & $ADB shell "am force-stop $PKG"
  & $ADB push $In /data/local/tmp/campaign_new.json | Out-Null
  & $ADB shell "chmod 644 /data/local/tmp/campaign_new.json"
  & $ADB shell "run-as $PKG cp /data/local/tmp/campaign_new.json $STATE"
  & $ADB shell "am start -n $PKG/com.exvius.local.DataDownloadActivity" | Out-Null
  "pushed and relaunched"
}

function Save-Json {
  param($Object, [string]$Out = "$PWD\campaign_new.json")
  $json = $Object | ConvertTo-Json -Depth 20 -Compress
  [IO.File]::WriteAllText($Out, $json, (New-Object Text.UTF8Encoding($false)))
  "wrote $Out ($((Get-Item $Out).Length) bytes)"
}
```

`run-as` **cannot write** to `/data/local/tmp`, which is why pushes go through `adb push` + `cp`.
Pulls go through `base64` because binary stdout gets mangled.

---

## 2. Always back up first

```powershell
Get-Save "$PWD\campaign.before_edit.json"
# optional on-device copy the app keeps next to the live save:
& $ADB shell "run-as $PKG cp $STATE files/exvius/state/campaign.before_edit.json"
```

---

## 3. Read/verify fields

```powershell
$sv = Get-Content .\campaign.json -Raw | ConvertFrom-Json

$sv.currencies | ConvertTo-Json -Compress
$sv.roster.Count
$sv.inventory.items.Count
$sv.roster | Where-Object { $_.master_id -eq '401016427' } |
    Select-Object master_id, level, nv_ex_level, lb_level | Format-Table
```

---

## 4. Example edits

### 4.1 Currencies

```powershell
$sv = Get-Content .\campaign.json -Raw | ConvertFrom-Json
$sv.currencies.lapis_free = 9999999     # display truncates above 9,999,999
$sv.currencies.lapis_paid = 9999999
$sv.currencies.gil        = 999999999
Save-Json $sv
Set-Save
```

### 4.2 Items — top up / add

```powershell
$sv = Get-Content .\campaign.json -Raw | ConvertFrom-Json
$items = @($sv.inventory.items)

function Set-ItemCount($items, $id, $count) {
  $e = $items | Where-Object { [string]$_.item_id -eq "$id" } | Select-Object -First 1
  if ($e) { $e.count = $count } else {
    $items += [pscustomobject]@{ item_type = 20; item_id = [int64]$id; count = $count }
  }
}

# all named materials below 900 -> 900, skipping coins/tickets
$names = @{}
foreach ($l in Get-Content .\item_names.ndjson) {
  if ($l.Trim()) { $o = $l | ConvertFrom-Json; $names[($o.key -replace '^MST_ITEM_NAME_','')] = $o.v[0] }
}
foreach ($it in $items) {
  $n = $names[[string]$it.item_id]
  if ($n -and $n -notmatch '(?i)coin|ticket' -and [int64]$it.count -lt 900) { $it.count = 900 }
}
Set-ItemCount $items 290050300 999     # Sacred Crystal
Set-ItemCount $items 1209002041 999    # Azure Pearl
$sv.inventory.items = $items
Save-Json $sv; Set-Save
```

### 4.3 Units — add a 100% Super Trust Moogle

```powershell
$sv = Get-Content .\campaign.json -Raw | ConvertFrom-Json
$maxId = [int64](($sv.roster | Measure-Object instance_id -Maximum).Maximum)

$u = $sv.roster[0] | ConvertTo-Json -Depth 6 | ConvertFrom-Json   # clone any entry as a shape template
$u.instance_id = [string]($maxId + 1)
$u.master_id   = '906000105'              # Super Trust Moogle (ALL)
$u.level = 1; $u.exp = 0; $u.total_exp = 0; $u.lb_exp = 0; $u.nv_ex_level = 0; $u.lb_level = 0
$u.guest = 0; $u.storage_state = 0; $u.favourite = 0
$u.granted_rarity = $null; $u.first_get_unit_id = $null
$u.equipment = [pscustomobject]@{}; $u.materia = @('0','0','0','0'); $u.vision_card = ''
$u.affinity_points = @('','1000')                   # channel 1 for 906… family
$u.affinity_info   = @('','70:0:0:1000:0:0:0')

$sv.roster = @($sv.roster) + $u
$sv.counts.unit = $sv.roster.Count
Save-Json $sv; Set-Save
```

Trust values (1000 = 100%): `904000101` `["10",""]` 1% · `904000104` `["50",""]` 5% ·
`904000105` `["100",""]` 10% · `906000104` `["","500"]` 50% · `906000105` `["","1000"]` 100%.
`904…` uses channel 0, `906…` channel 1.

### 4.4 Material unit that must not be auto-converted

Metal Minituars/pots get converted to points on load. Add `"favourite": 1`:

```powershell
$u.favourite = 1
```

### 4.5 Point bank

```powershell
$sv = Get-Content .\campaign.json -Raw | ConvertFrom-Json
$bank = $sv.exvius_unit_points.banks.'0'
foreach ($t in 'XeRVw3s2','jWS57HG3','LD4UJaf0','ui34MYgo','5sNq90o6','0xgsDY9E','VSL4Wj9z','y7Z5AmhC',
               'h5s68EtJ','eu2DXK4I','8RUQtk4i','8cj19yhb','rjK5oB1H','16NCtrGw') {
  $bank | Add-Member -NotePropertyName $t -NotePropertyValue ([int64]9000000000000000000) -Force
}
Save-Json $sv; Set-Save
```

---

## 5. Master-data edit (EXP units)

Yield = `N85uJT1d` + rarity bonus (100/200/500/1000/1500/3000/5000 for rarity 1–7).

```powershell
$MST = "files/exvius/data/static/mst_decoded/F_UNIT_MST.ndjson"
& $ADB shell "am force-stop $PKG"
& $ADB shell "run-as $PKG cp $MST $MST.bak"          # backup on device

$b64 = (& $ADB shell "run-as $PKG base64 $MST") -join ''
[IO.File]::WriteAllBytes("$PWD\F_UNIT_MST.ndjson", [Convert]::FromBase64String($b64.Trim()))
```

Edit with Python (target total 393,000,000 each, accounting for the rarity bonus):

```python
# boost_exp.py
import json
TARGETS = {'900020101': 392_999_900, '900020201': 392_999_800,
           '900020301': 392_999_500, '900020401': 392_999_000}
out = []
for line in open('F_UNIT_MST.ndjson', encoding='utf-8'):
    if not line.strip():
        out.append(line); continue
    o = json.loads(line)
    if str(o.get('3HriTp6B')) in TARGETS:
        o['N85uJT1d'] = str(TARGETS[str(o['3HriTp6B'])])
    out.append(json.dumps(o, ensure_ascii=False, separators=(',', ':')) + '\n')
open('F_UNIT_MST_boosted.ndjson', 'w', encoding='utf-8').write(''.join(out))
```

```powershell
python boost_exp.py
& $ADB push .\F_UNIT_MST_boosted.ndjson /data/local/tmp/unit.ndjson | Out-Null
& $ADB shell "chmod 644 /data/local/tmp/unit.ndjson"
& $ADB shell "run-as $PKG cp /data/local/tmp/unit.ndjson $MST"
& $ADB shell "am start -n $PKG/com.exvius.local.DataDownloadActivity"
```

Verify persist after launch:

```powershell
& $ADB shell "run-as $PKG grep -c 392999900 $MST"     # expect 1
```

Rollback: `run-as $PKG cp $MST.bak $MST`.
This edit is lost on reinstall / “repair game assets”.

---

## 6. Summon-rate edit (requires APK rebuild)

```powershell
$JAVA = "C:\Program Files\Android\Android Studio\jbr\bin\java.exe"
$SDK  = "C:\Users\<you>\AppData\Local\Android\Sdk\build-tools\36.0.0"
$env:JAVA_HOME = "C:\Program Files\Android\Android Studio\jbr"
$env:PATH = "$env:JAVA_HOME\bin;$env:PATH"

# 1) decode the stock APK
& $JAVA -Xmx4g -jar C:\tools\apktool.jar d .\exvius-offline-v0.1.30.apk -o .\decoded -f
```

Rate files are `.\decoded\assets\rules\gacha_*.json`. Script that sets 5★+ to 30% and
`featured_share` to 0.40:

```python
# set_rates.py  (run inside the folder holding gacha_*.json)
import json, os
TARGET5, SHARE = 0.30, 0.40
def scale(entries, getr, getp, setr):
    s5 = sum(getp(e) for e in entries if getr(e) >= 5)
    total = sum(getp(e) for e in entries)
    if s5 <= 0 or s5 >= TARGET5 or total - s5 <= 0: return False
    f5, fr = TARGET5 / s5, (1 - TARGET5) / (total - s5)
    for e in entries: setr(e, getp(e) * (f5 if getr(e) >= 5 else fr))
    return True
for fn in os.listdir('.'):
    if not (fn.startswith('gacha') and fn.endswith('.json')): continue
    d = json.load(open(fn, encoding='utf-8')); bs = d.get('banners') if isinstance(d, dict) else d
    if not isinstance(bs, list): continue
    for b in bs:
        pool = b.get('pool')
        if isinstance(pool, list):
            scale(pool, lambda e: int(e.get('rarity',0)), lambda e: float(e.get('rate',0)),
                  lambda e,v: e.__setitem__('rate', v))
        elif isinstance(pool, dict):
            ts = pool.get('tiers', [])
            scale(ts, lambda t: int(t.get('rarity',0)), lambda t: float(t.get('rate',0)),
                  lambda t,v: t.__setitem__('rate', v))
            for t in ts:
                if int(t.get('rarity',0)) >= 5 and t.get('featured'):
                    t['featured_share'] = SHARE
    json.dump(d, open(fn,'w',encoding='utf-8'), ensure_ascii=False, separators=(',',':'))
```

```powershell
cd .\decoded\assets\rules ; python set_rates.py ; cd ..\..\..

# 2) rebuild, align, sign
& $JAVA -Xmx4g -jar C:\tools\apktool.jar b .\decoded -o .\rates-unsigned.apk
& "$SDK\zipalign.exe" -p -f 4 .\rates-unsigned.apk .\rates-aligned.apk
& "$SDK\apksigner.bat" sign --ks .\release.jks --ks-pass pass:exvius123 --key-pass pass:exvius123 `
    --out .\exvius-offline-v0.1.30-rates.apk .\rates-aligned.apk

# 3) install as update (same key keeps data + 14 GB)
& $ADB install -r .\exvius-offline-v0.1.30-rates.apk
```

---

## 7. Making a debuggable build from the stock APK

1. `apktool d exvius-offline-v0.1.30.apk -o decoded -f`
2. `decoded\AndroidManifest.xml`:
   - `android:debuggable="false"` → `"true"`
   - add `android:usesCleartextTraffic="true"` to `<application …>`
3. Optional LAN pack hosting — `decoded\smali_classes6\com\exvius\local\DataUpdates.smali`:
   - `https://raw.githubusercontent.com/Elvlin/exvius-offline/main/data/latest.json` → `http://<your-pc-ip>:8080/latest.json`
   - `https://github.com/Elvlin/exvius-offline/releases/download/` → `http://<your-pc-ip>:8080/`
     (**required**, or the client throws “Invalid data pack information”)
4. Rebuild + align + sign (as §6 step 2).
5. Create the key once:

```powershell
& "$env:JAVA_HOME\bin\keytool.exe" -genkeypair -keystore release.jks -alias exvius `
  -keyalg RSA -keysize 2048 -validity 10000 -storepass exvius123 -keypass exvius123 `
  -dname "CN=Exvius Offline Mod"
```

6. `adb install -r` — **never `adb uninstall`** unless you accept losing the save and the 14 GB.

---

## 8. Rollback / restore

```powershell
# save
Set-Save -In .\campaign.before_edit.json

# master table
& $ADB shell "run-as $PKG cp files/exvius/data/static/mst_decoded/F_UNIT_MST.ndjson.bak files/exvius/data/static/mst_decoded/F_UNIT_MST.ndjson"

# APK (unmodified rates, same patches)
& $ADB install -r .\exvius-offline-v0.1.30-lan.apk
```

---

## 9. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `run-as: package not debuggable` | stock APK — install your patched build |
| Pushed save ignored, units vanish | material units auto-exchanged → set `favourite: 1` |
| “Invalid data pack information” | pack-URL prefix not patched to your host |
| Counts show lower than written | display truncation (lapis 7 digits, EXP 10, materials 4) |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | different signing key — re-sign with your key, or uninstall (wipes data) |
| Save reverts after reinstall | expected — uninstall deletes `/data/user/0/<pkg>` |
| Master edits revert | app re-extracted data (reinstall/repair); re-apply |
| Game shows “Point conversion has been completed” | normal notice after auto-exchange |
| Bank value wrapped negative | exceeded int64 through repeated additions — restore a backup |

---

## 10. Safety checklist

1. Back up `campaign.json` before every edit.
2. `am force-stop` before pulling/pushing.
3. Keep at least one known-good APK and `release.jks` off-device.
4. Don’t fill int64 fields to the brim if the game can still add to them.
5. Re-check `counts.unit` after any roster change.
