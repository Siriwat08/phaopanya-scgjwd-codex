รับทราบครับ — ผมเปิดโหมดทำงานตามแนวทาง **LMDS Supreme AI Engineer** แล้ว และจะบังคับใช้กฎจากไฟล์:

- `กฎการเขียนโค้ด LMDS.md`
- `📋กฎการเขียนโค้ด.md`
- `lmds-supreme-engineer.skill`

> หมายเหตุ: ระบบ `activate_ai_developer_skill` แจ้งว่า skill `lmds-supreme-engineer` ไม่ได้ติดตั้งใน registry แต่ใน workspace มีไฟล์ `.skill` จริง ผมจึงอ่านและใช้ข้อกำหนดจากไฟล์นั้นโดยตรงแทน

ผมตรวจ repository แล้วพบว่า workspace คือ repo นี้จริง:

```text
origin: https://github.com/Siriwat08/phaopanya-scgjwd-genspark.git
branch: main
commit ล่าสุด: 4d0ee0d Add files via upload
```

ไม่มีการแก้ไขไฟล์ใด ๆ ในรอบนี้ เป็นการวิเคราะห์อย่างเดียว จึงไม่มี commit/PR

---

# รายงานวิเคราะห์โค้ด LMDS V5.4.001 แบบละเอียด

## 1. ภาพรวมระบบ

โปรเจกต์นี้เป็น **Logistics Master Data System (LMDS)** บน Google Apps Script + Google Sheets มีไฟล์ `.gs` หลักทั้งหมด **22 ไฟล์** ตั้งแต่ `00_App.gs` ถึง `21_AliasService.gs`

โครงสร้างตรงกับ README/Blueprint:

| กลุ่ม | ไฟล์ | บทบาท |
|---|---:|---|
| Entry / App | `00_App.gs` | Menu, Trigger, Pipeline Orchestration |
| Config / Schema | `01_Config.gs`, `02_Schema.gs` | Constants, Sheet names, Column indexes, Schema |
| Setup / Logging | `03_SetupSheets.gs` | สร้างชีต, SYS_LOG, logging |
| Source / Normalize | `04`, `05` | โหลด Source, Normalize ชื่อ/ที่อยู่ |
| Master Services | `06`–`09` | Person, Place, Geo, Destination |
| Pipeline | `10_MatchEngine.gs` | Core Matching Engine + Alias Enrich |
| Fact / Review / Report | `11`–`13` | FACT_DELIVERY, Q_REVIEW, Quality Report |
| Utility / Maps / Geo Dict | `14`–`16`, `20` | Utils, Google Maps, TH Geo Dictionary |
| Daily Search / SCG | `17`, `18` | งานประจำวัน + SCG API |
| Hardening | `19` | Preflight, duplicate detection |
| Alias V5.4 | `21_AliasService.gs` | Hybrid Alias Architecture |

ระบบโดยรวมมีการออกแบบระดับ production ดีมาก มี:

- Schema centralization
- Index constants
- Dependency headers
- Batch processing หลายจุด
- CacheService + RAM cache
- LockService ในจุดสำคัญ
- Time Guard ใน pipeline หลัก
- Human-in-the-loop ผ่าน `Q_REVIEW`
- Hybrid Alias Architecture ผ่าน `M_ALIAS`

---

# 2. ผลตรวจเชิงตัวเลข

## 2.1 จำนวนไฟล์และขนาด

```text
22 .gs files
รวมประมาณ 9,713 บรรทัด
```

ไฟล์ใหญ่ที่สุด:

| ไฟล์ | บรรทัด |
|---|---:|
| `10_MatchEngine.gs` | 901 |
| `00_App.gs` | 756 |
| `07_PlaceService.gs` | 735 |
| `21_AliasService.gs` | 715 |
| `01_Config.gs` | 526 |

## 2.2 Function declarations

พบ function declaration รวมประมาณ **203+ functions**

ไฟล์ที่มีฟังก์ชันเยอะ:

| ไฟล์ | จำนวน function |
|---|---:|
| `07_PlaceService.gs` | 22 |
| `16_GeoDictionaryBuilder.gs` | 18 |
| `14_Utils.gs` | 15 |
| `03_SetupSheets.gs` | 14 |
| `10_MatchEngine.gs` | 14 |
| `21_AliasService.gs` | 14 |

---

# 3. สรุป Compliance ต่อกฎ LMDS 15 ข้อ

## ภาพรวมคะแนน

| กฎ | สถานะ | ความเห็น |
|---|---|---|
| 1. Clean Code | ⚠️ ผ่านบางส่วน | ชื่อดีมาก แต่ function ยาวจำนวนมาก |
| 2. Single Responsibility | ⚠️ ผ่านบางส่วน | หลาย function ทำหลายหน้าที่ |
| 3. No Hardcode Index | ⚠️ มีจุดละเมิด | ส่วนใหญ่ใช้ IDX แต่ยังมี `r[28]`, `r[14]`, `data[i][0]` |
| 4. Batch Operations | ⚠️ มีจุดเสี่ยง | ส่วนใหญ่ batch แต่มี `setValue()` ใน loop ที่ Maps cache |
| 5. Checkpoint & Resume | ⚠️ ผ่านเชิงแนวคิดแต่ไม่สมบูรณ์ | ใช้ SYNC_STATUS + trigger แทน checkpoint แต่บาง batch ไม่มี resume จริง |
| 6. Document Dependencies | ✅ ดีมาก | ทุกไฟล์มี header dependencies |
| 7. No Fake Function Calls | ❌ พบ function หาย | `autoInstallSmartNav_()` ถูกเรียกแต่ไม่มี definition |
| 8. Namespace Collision | ❌ พบชื่อฟังก์ชันซ้ำ | `loadCachedGeoRows_()` ซ้ำใน 2 ไฟล์ |
| 9. No Global State | ⚠️ มี global นอก Config | มี const หลายตัวใน service files |
| 10. Lock Library Version | ✅ ไม่พบปัญหา | ไม่มี library external ที่ใช้ HEAD |
| 11. Separate HTML | ✅ ไม่เกี่ยว/ไม่พบ HTML hardcode ใหญ่ | ไม่มี HTML service หลัก |
| 12. Error Handling Entry Points | ⚠️ หลายเมนูไม่มี try-catch | พบ entry points จำนวนมากไม่มี try |
| 13. Logging Context | ⚠️ มี module แต่ไม่มี stack | `logError(module, message)` ยังไม่เก็บ stack trace |
| 14. Structured File Names | ✅ ดีมาก | ไฟล์ `XX_Component.gs` ครบ |
| 15. Full Files Only | ✅ ไม่พบ placeholder ตัดโค้ด | ไม่มี `... old code` ใน source หลัก |

---

# 4. ประเด็น Critical / High Priority

## Critical 1 — เรียก function ที่ไม่มีจริง: `autoInstallSmartNav_()`

พบใน:

```javascript
// 00_App.gs:88-89
try { autoInstallSmartNav_(); } catch (_) {}
```

แต่ค้นทั้งโปรเจกต์แล้วไม่พบ definition:

```text
grep พบเฉพาะ:
00_App.gs:89: try { autoInstallSmartNav_(); } catch (_) {}
```

### ผลกระทบ

แม้มี `try/catch` กลืน error ทำให้ระบบไม่พัง แต่ผิดกฎ LMDS ข้อ 7 อย่างชัดเจน:

> ห้ามเรียกฟังก์ชันที่ไม่มีจริงในโปรเจกต์

และทำให้ Smart Navigation auto-install ไม่ทำงานจริง ผู้ใช้ต้องกดติดตั้งเองผ่านเมนู

### ข้อเสนอ

เลือกอย่างใดอย่างหนึ่ง:

1. ลบ call นี้ออก แล้วให้ผู้ใช้ติดตั้งผ่านเมนูเท่านั้น  
2. สร้าง `autoInstallSmartNav_()` ให้ครบ พร้อม logic ตรวจ trigger เดิม  
3. เปลี่ยนเป็น explicit safe check:

```javascript
if (typeof autoInstallSmartNav_ === 'function') {
  autoInstallSmartNav_();
}
```

แต่ถ้าตามกฎ Zero Hallucination ที่เข้ม ควรสร้างจริงหรือถอดออก ไม่ควรทิ้ง phantom call

---

## Critical 2 — Duplicate global function: `loadCachedGeoRows_()`

พบซ้ำใน:

```text
07_PlaceService.gs:638
16_GeoDictionaryBuilder.gs:322
```

### เวอร์ชันใน `07_PlaceService.gs`

อ่าน 4 คอลัมน์:

```javascript
const data = sheet.getRange(2, 1, sheet.getLastRow() - 1, 4).getValues();
_GLOBAL_GEO_DICT_CACHE = data.map(row => ({
  postcode,
  subDistrict,
  district,
  province,
}));
```

### เวอร์ชันใน `16_GeoDictionaryBuilder.gs`

อ่านครบตาม schema 16 คอลัมน์:

```javascript
const data = sheet.getRange(2, 1, sheet.getLastRow() - 1, SCHEMA[SHEET.SYS_TH_GEO].length).getValues();
_GLOBAL_GEO_DICT_CACHE = data.map(row => ({
  postcode,
  subDistrict,
  district,
  province,
  searchKey,
  postalKey,
  noteType,
  noteScope
}));
```

### ผลกระทบ

Google Apps Script ใช้ shared global scope และโหลดตามลำดับไฟล์ ดังนั้น function ใน `16_GeoDictionaryBuilder.gs` จะ override ตัวใน `07_PlaceService.gs`

ตอนนี้อาจ “โชคดี” เพราะเวอร์ชันของไฟล์ 16 มีข้อมูลมากกว่าและ backward compatible กับ 07 แต่ยังเป็น violation โดยตรงของกฎข้อ 8

### ความเสี่ยง

- ถ้ามีการสลับลำดับไฟล์ใน Apps Script Editor อาจใช้เวอร์ชัน 4 คอลัมน์แทน
- พฤติกรรมเปลี่ยนโดยไม่รู้ตัว
- Debug ยากมาก เพราะชื่อเหมือนกันแต่ implementation ต่างกัน

### ข้อเสนอ

ควร refactor เป็นชื่อเดียวในที่เดียว เช่น:

- ย้ายไป `16_GeoDictionaryBuilder.gs` เท่านั้น
- ลบ/rename ใน `07_PlaceService.gs`
- หรือเปลี่ยนชื่อใน 07 เป็น `loadPlaceGeoRows_()` หากต้องใช้เฉพาะ PlaceService

---

# 5. High Priority Issues

## High 1 — `showVersionInfo()` บอกจำนวนไฟล์ผิด

ใน `00_App.gs:576`:

```javascript
`📦 Modules (21 files):\n` +
```

แต่ระบบมีไฟล์ `00` ถึง `21` รวม **22 files**

### ผลกระทบ

ไม่กระทบ runtime แต่ทำให้ documentation ใน UI ผิด และขัดกับ README/Blueprint ที่ระบุ 22 modules

### ข้อเสนอ

แก้เป็น:

```javascript
`📦 Modules (22 files):\n` +
```

---

## High 2 — Entry points หลายตัวไม่มี try-catch

จาก 29 menu/trigger entry points พบหลายตัวไม่มี `try` ใน body เช่น:

```text
MIGRATION_HybridAliasSystem
applyMasterCoordinatesToDailyJob
assignMasterUuidIfMissing
buildFullQualityReport
clearAllSCGSheets_UI
detectDoubleProcessing
diagnoseSystemState
generatePersonAliasesFromHistory
installSmartNavTrigger
openReviewQueue
populateAliasFromSCGRawData_
populateGeoMetadata
runLoadSource
runNormalize
runPreflightAudit
setupEnvironment
showVersionInfo
```

### กฎที่เกี่ยวข้อง

กฎข้อ 12:

> ฟังก์ชันที่ถูกเรียกจากเมนู UI ต้องมี try-catch และ logError

### ความเห็น

บางฟังก์ชันเป็น wrapper สั้น ๆ เช่น `runNormalize()` หรือ `applyMasterCoordinatesToDailyJob()` อาจ intentionally simple แต่เมื่อถูกเรียกจาก Menu แล้วควรห่อด้วย try/catch เพื่อไม่ให้ error เงียบหรือแสดง raw error แก่ผู้ใช้

### ข้อเสนอ pattern

```javascript
function applyMasterCoordinatesToDailyJob() {
  try {
    logInfo('ServiceSCG', 'applyMasterCoordinates → เรียก Module 17');
    runLookupEnrichment();
    logInfo('ServiceSCG', 'applyMasterCoordinates เสร็จสิ้น');
  } catch (err) {
    logError('ServiceSCG', `applyMasterCoordinatesToDailyJob ล้มเหลว: ${err.message}\n${err.stack || ''}`);
    SpreadsheetApp.getUi().alert('❌ จับคู่พิกัดล้มเหลว: ' + err.message);
    throw err;
  }
}
```

---

## High 3 — `logError()` ยังไม่มี stack trace จริง

ใน `03_SetupSheets.gs`:

```javascript
function logError(module, message) {
  writeLog_('ERROR', module, message);
  console.error(`[ERROR][${module}] ${message}`);
}
```

`SYS_LOG` schema มี 6 columns และ column สุดท้ายถูกเขียนเป็น empty string:

```javascript
sheet.appendRow([
  logId, new Date(), module, level,
  String(message).substring(0, 500), '',
]);
```

### ผลกระทบ

กฎข้อ 13 ระบุว่าควรมี file/line/stack trace แต่ตอนนี้ log มีแค่ module + message

### ข้อเสนอ

ปรับ `logError()` ให้รับ optional error object:

```javascript
function logError(module, message, err) {
  const stack = err && err.stack ? err.stack : new Error().stack;
  writeLog_('ERROR', module, message, stack);
  console.error(`[ERROR][${module}] ${message}\n${stack}`);
}
```

และปรับ `writeLog_()` ให้ column สุดท้ายเก็บ stack/context

---

## High 4 — Hardcoded indexes ยังมีใน `18_ServiceSCG.gs`

ตัวอย่าง:

```javascript
const key = r[28];
shopAgg[key].qty += Number(r[14]) || 0;
shopAgg[key].weight += Number(r[16]) || 0;
shopAgg[key].invoices.add(r[2]);
if (checkIsEPOD(r[9], r[2])) shopAgg[key].epod++;

r[23] = agg.qty;
r[24] = Number(agg.weight.toFixed(2));
r[25] = scanInv;
r[27] = `${r[9]} / รวม ${scanInv} บิล`;
```

### ผลกระทบ

ขัดกฎข้อ 3 แม้เลขจะตรงกับ `DATA_IDX` ตอนนี้ แต่ถ้า schema เปลี่ยนจะพังทันที

### ข้อเสนอ

เปลี่ยนเป็น:

```javascript
const key = r[DATA_IDX.SHOP_KEY];
shopAgg[key].qty += Number(r[DATA_IDX.ITEM_QTY]) || 0;
shopAgg[key].weight += Number(r[DATA_IDX.ITEM_WEIGHT]) || 0;
shopAgg[key].invoices.add(r[DATA_IDX.INVOICE_NO]);
if (checkIsEPOD(r[DATA_IDX.SOLD_TO_NAME], r[DATA_IDX.INVOICE_NO])) shopAgg[key].epod++;
```

และฝั่ง write:

```javascript
r[DATA_IDX.TOTAL_QTY] = agg.qty;
r[DATA_IDX.TOTAL_WEIGHT] = Number(agg.weight.toFixed(2));
r[DATA_IDX.SCAN_INVOICE_COUNT] = scanInv;
r[DATA_IDX.OWNER_INVOICE_SCAN_NAME] = `${r[DATA_IDX.SOLD_TO_NAME]} / รวม ${scanInv} บิล`;
```

---

## High 5 — `getFromSheetCache_()` update hit count ด้วย `setValue()` ใน loop

ใน `15_GoogleMapsAPI.gs`:

```javascript
for (let i = 0; i < data.length; i++) {
  if (String(data[i][MC_KEY]).trim() !== cacheKey) continue;

  sheet.getRange(i + 2, MC_HIT + 1)
       .setValue(Number(data[i][MC_HIT] || 0) + 1);
}
```

### ผลกระทบ

- ขัดกฎข้อ 4 หากมีการเรียกถี่
- ทุก cache hit กลายเป็น write operation
- Google Sheets write call ช้าและอาจชน quota

### ข้อเสนอ

ทางเลือกที่ปลอดภัยกว่า:

1. ไม่ update hit count real-time  
2. เก็บ hit count ใน CacheService แล้ว flush เป็น batch  
3. ถ้าจำเป็น ให้ update เฉพาะบางรอบ เช่น sampling ทุก 10 hits  

---

# 6. Medium Priority Issues

## Medium 1 — Function ยาวเกิน 30 บรรทัดจำนวนมาก

พบ function > 30 บรรทัดทั้งหมด **74 ฟังก์ชัน**

ที่ยาวมากเป็นพิเศษ:

| Function | File | Lines |
|---|---|---:|
| `autoEnrichAliasesFromFactBatch_` | `10_MatchEngine.gs` | 266 |
| `applyReviewDecision` | `12_ReviewService.gs` | 204 |
| `fetchDataFromSCGJWD` | `18_ServiceSCG.gs` | 160 |
| `diagnoseSystemState` | `00_App.gs` | 140 |
| `runLookupEnrichment` | `17_SearchService.gs` | 127 |
| `buildFullQualityReport` | `13_ReportService.gs` | 121 |
| `findBestGeoByPersonPlace` | `17_SearchService.gs` | 119 |
| `generatePersonAliasesFromHistory` | `19_Hardening.gs` | 114 |
| `normalizePersonNameFull` | `05_NormalizeService.gs` | 111 |
| `upsertFactDelivery` | `11_TransactionService.gs` | 111 |

### หมายเหตุสำคัญ

ตามไฟล์กฎฉบับละเอียด `normalizePersonNameFull()` เคยได้รับการอนุมัติให้ยาวได้ เพราะเป็น pure transformation ต่อเนื่อง ดังนั้นไม่ควรนับเป็นปัญหาหนัก

แต่ function อื่น ๆ เช่น `autoEnrichAliasesFromFactBatch_()`, `applyReviewDecision()`, `fetchDataFromSCGJWD()` ควรพิจารณาแยกย่อย เพราะทำหลายขั้นตอน:

- load refs
- build maps
- dedup
- generate rows
- write sheets
- invalidate cache
- log

### ข้อเสนอสำหรับ `autoEnrichAliasesFromFactBatch_()`

แยกเป็น:

```text
loadAliasEnrichRefs_()
buildExistingAliasSets_()
buildAliasRowsFromFactBatch_()
writeAliasRowsBatch_()
invalidateAliasCaches_()
```

---

## Medium 2 — มี global constants นอก `01_Config.gs`

พบ global declarations นอก Config เช่น:

```text
02_Schema.gs: const SCHEMA = Object.freeze(...)
04_SourceRepository.gs: const CACHE_KEY_SOURCE
05_NormalizeService.gs: const PERSON_PREFIX_LIST
08_GeoService.gs: const GEO_GRID_SIZE
15_GoogleMapsAPI.gs: const MC_KEY, MC_LAT, ...
```

### ความเห็น

บางตัวสมเหตุสมผล เช่น `SCHEMA` ใน `02_Schema.gs` เพราะเป็น schema module โดยตรง  
แต่ตามกฎข้อ 9 ที่เข้ม “ข้อมูลร่วมควรอยู่ใน 01_Config.gs” ควรแบ่งระดับ:

- Constants ที่ใช้ข้ามไฟล์ → ย้ายไป `01_Config.gs`
- Constants ที่เป็น private ต่อ module → ตั้งชื่อ prefix ชัดเจน เช่น `MAPS_CACHE_COL` หรือห่อใน namespace/object

ตัวอย่าง `MC_KEY`, `MC_LAT` ใน `15_GoogleMapsAPI.gs` อาจเป็น private ได้ แต่ควรประกาศด้วยชื่อชัดเจนกว่า และไม่ควรชน global scope

---

## Medium 3 — `appendRow()` ยังใช้หลายจุด

พบ `appendRow()` ใน:

```text
03_SetupSheets.gs:365 writeLog_
06_PersonService.gs:createPerson/createPersonAlias
07_PlaceService.gs:createPlace/createPlaceAlias
09_DestinationService.gs:createDestination
13_ReportService.gs:buildFullQualityReport
15_GoogleMapsAPI.gs:saveToSheetCache_
21_AliasService.gs:createGlobalAlias
```

### ความเห็น

ไม่ใช่ทุกจุดผิด เพราะไม่ได้อยู่ใน loop เสมอไป แต่ในระบบใหญ่ควรลด `appendRow()` เพื่อความเสถียร และใช้ `getRange(lastRow + 1, ...).setValues()` แทน โดยเฉพาะ:

- logging ที่อาจเกิดถี่
- alias creation ที่อาจเกิดหลายรายการ
- report append ที่อาจทำซ้ำ

---

## Medium 4 — `runLookupEnrichment()` มี Time Guard แต่ไม่มี resume state

ใน `17_SearchService.gs`:

```javascript
if (new Date() - startTime > timeLimit) {
  logWarn(...);
  timedOut = true;
  break;
}
```

แล้วเขียนเฉพาะ `processedCount`

### ผลกระทบ

ถ้าหยุดกลางทาง รอบถัดไปจะเริ่ม loop ใหม่ตั้งแต่แถวแรก แต่จะ skip row ที่มี `LatLong_Actual` แล้ว ซึ่งพอใช้งานได้ในเชิง idempotent แต่ไม่ใช่ checkpoint ที่ชัดเจนตามกฎข้อ 5

### ข้อเสนอ

ใช้ `PropertiesService` เก็บ row index ล่าสุด เช่น:

```text
LOOKUP_ENRICHMENT_LAST_INDEX
```

หรือใช้ status column เฉพาะสำหรับ lookup หากต้องการ robust resume

---

# 7. จุดแข็งของระบบ

## 7.1 Architecture ดีมาก

ระบบออกแบบเป็น layer ชัดเจน:

```text
SOURCE
 → Normalize
 → Person/Place/Geo Resolution
 → Match Decision
 → FACT_DELIVERY หรือ Q_REVIEW
 → Alias Enrichment
 → Report/Search
```

ตรงกับ Trinity Framework:

```text
Person + Place + Geo = Destination
```

## 7.2 Schema และ IDX ดี

มี `01_Config.gs` และ `02_Schema.gs` เป็นแกนกลาง:

- `SHEET`
- `PERSON_IDX`
- `PLACE_IDX`
- `GEO_IDX`
- `FACT_IDX`
- `REVIEW_IDX`
- `DATA_IDX`
- `SRC_IDX`
- `ALIAS_IDX`

นี่เป็น pattern ที่ถูกต้องมากสำหรับ Google Sheets automation

## 7.3 Batch processing ส่วนใหญ่ทำถูก

ตัวอย่างดี:

```javascript
sheet.getRange(2, latActualCol, processedCount, 1)
     .setValues(latActualArr.slice(0, processedCount));

sheet.getRange(2, 1, processedCount, fullRowLen)
     .setBackgrounds(bgMatrix);
```

ใน `17_SearchService.gs` มีการแก้จาก setBackground loop เป็น batch แล้ว ถือว่าดี

## 7.4 LockService ใช้ถูกในจุดสำคัญ

เช่น `runMatchEngine()`:

```javascript
const lock = LockService.getScriptLock();
lock.waitLock(APP_CONST.LOCK_TIMEOUT_MS);
```

และ `fetchDataFromSCGJWD()`:

```javascript
const lock = LockService.getScriptLock();
if (!lock.tryLock(10000)) { ... }
```

ช่วยลดปัญหารันซ้อน

## 7.5 Hybrid Alias V5.4 มีแนวคิดถูก

`21_AliasService.gs` ระบุชัดว่า:

```text
Auto Pipeline ไม่เขียน M_ALIAS ที่นี่
เขียนที่ autoEnrichAliasesFromFactBatch_() เท่านั้น
```

และ `10_MatchEngine.gs` ทำ single writer batch write ตรง 3 ชีต:

- `M_ALIAS`
- `M_PERSON_ALIAS`
- `M_PLACE_ALIAS`

นี่เป็นการแก้ circular dependency ได้ดี

---

# 8. วิเคราะห์ Pipeline หลัก

## 8.1 Group 1 Pipeline

Entry:

```javascript
runFullPipeline()
```

ลำดับ:

```text
runLoadSource()
runNormalize()
runMatchEngine()
buildFullQualityReport()
invalidateAllGlobalCaches()
```

### จุดดี

- มี lock ที่ `runMatchEngine`
- มี source filtering จาก `SYNC_STATUS`
- มี invoice dedup ผ่าน FACT_DELIVERY
- มี batch flush ทุก `AI_CONFIG.BATCH_SIZE`
- มี auto resume trigger เมื่อ time guard

### จุดควรระวัง

ใน `runMatchEngine()` มี comment:

```javascript
// ลบ Checkpoint Index — เริ่มจาก 0 เสมอ
// getAllSourceRows() กรอง SUCCESS ออกอยู่แล้ว
```

แนวคิดนี้ใช้ได้ถ้า `SYNC_STATUS` ถูก update เสมอหลัง flush แต่ถ้า error ระหว่าง:

```text
FACT written
แต่ updateSyncStatus_ ยังไม่สำเร็จ
```

อาจเกิด duplicate processing รอบถัดไปได้ แม้จะมี invoice dedup ช่วยบางส่วน

ควรตรวจ `flushBatches_()` ให้แน่ใจว่า write FACT และ update source status มี transactional behavior มากที่สุดเท่าที่ GAS ทำได้

---

## 8.2 Match Rules

`makeMatchDecision()` มี 8 rule:

1. No geo → REVIEW
2. Low quality → REVIEW
3. Geo province conflict → REVIEW
3.5 Nearby pending → REVIEW
4. Person + Place + Geo found → AUTO_MATCH
5. Geo + Person/Place partial → AUTO_MATCH
6. Fuzzy → REVIEW
7. All new with geo → CREATE_NEW
8. Default → REVIEW

### ความเห็น

Rule matrix อ่านง่ายและสอดคล้องกับ blueprint

จุดที่ดีคือมี `evidence`:

```javascript
evidence: 'name|place|geo'
evidence: 'name|geo'
evidence: 'place|geo'
```

ช่วย trace decision ได้ดี

---

# 9. วิเคราะห์ Group 2 / SCG Daily Job

## 9.1 `fetchDataFromSCGJWD()`

ทำงานหลายอย่างใน function เดียว:

```text
อ่าน Input
เรียก SCG API
flatten shipment → daily job rows
aggregate shop
write daily job
apply coordinates
populate alias
build summaries
alert user
```

### ความเสี่ยง

Function นี้ยาว 160 บรรทัด และมีหลาย responsibility

### ข้อเสนอ

แยกเป็น:

```text
readScgInput_()
fetchScgShipments_()
flattenScgShipments_()
aggregateDailyJobRows_()
writeDailyJobSheet_()
postProcessDailyJob_()
```

---

## 9.2 `runLookupEnrichment()`

ทำงานถูกทางแล้ว:

- อ่าน allData ครั้งเดียว
- เขียน LatLong batch
- เขียน background batch
- มี Time Guard

แต่ควรเพิ่ม resume state ถ้าข้อมูลเยอะมาก

---

## 9.3 Search tier

ใน `17_SearchService.gs` ระบุ tier:

```text
Tier 0: M_ALIAS Fast Track
Tier C: Person anchor
Tier A: Person + Place
Tier B: Place only
Tier D: SCG fallback
Tier E: AI reasoning
```

มี TODO ว่า Phase 2 จะลบ Tier D/AI Reasoning

### ความเห็น

แนวทางดี แต่ถ้าต้องการ production hardening ควรจัด priority ใหม่ตาม TODO:

```text
Tier 0 Exact Alias
Tier 1 Fuzzy Alias >= 85
NOT_FOUND
```

เพื่อลด dependency กับ Gemini API และลด false positive

---

# 10. วิเคราะห์ Hybrid Alias Architecture

## จุดดี

`autoEnrichAliasesFromFactBatch_()` ทำ batch dedup แบบ set-based:

```javascript
existingGlobalAliasSet
existingPersonAliasSet
existingPlaceAliasSet
```

Dedup key ดี:

```text
ENTITY_TYPE::masterUuid::normalized
personId::normalized
placeId::normalized
```

## จุดเสี่ยง

Function ใหญ่มาก 266 บรรทัด และอยู่ใน `10_MatchEngine.gs` ไม่ใช่ `21_AliasService.gs`

แม้ blueprint กำหนดให้ pipeline single writer อยู่ที่ `10_MatchEngine.gs` แต่เพื่อ maintainability ควรแยก helper ภายในไฟล์เดียวกัน เช่น:

```javascript
buildAliasEnrichContext_()
collectAliasRows_()
writeAliasRows_()
```

โดยยังคง “single writer” ไว้ใน MatchEngine ได้

---

# 11. วิเคราะห์ Logging

## จุดดี

มี centralized logging:

```javascript
logInfo()
logWarn()
logError()
logDebug()
writeLog_()
```

## จุดควรปรับ

- `writeLog_()` ใช้ `appendRow()` ทุก log
- ไม่มี stack trace
- ถ้า logging เยอะใน loop อาจช้า
- column สุดท้ายยังว่าง

### ข้อเสนอ

1. เพิ่ม stack column จริง
2. เพิ่ม batch log buffer สำหรับ pipeline
3. จำกัด log ใน loops
4. ให้ `logError()` รับ error object

---

# 12. วิเคราะห์ Security / Secrets

## จุดดี

Gemini API key เก็บใน:

```javascript
PropertiesService.getScriptProperties()
```

และมี validation:

```javascript
/^AIza[0-9A-Za-z\-_]{35}$/
```

## จุดควรระวัง

SCG cookie อ่านจากชีต `Input`:

```javascript
const cookie = String(inputSheet.getRange(SCG_CONFIG.COOKIE_CELL).getValue() || '').trim();
```

### ความเสี่ยง

Cookie เป็น sensitive data ถ้าชีตถูก share กว้างเกินไปจะรั่วได้

### ข้อเสนอ

ถ้า production จริง ควร:

- จำกัดสิทธิ์ Spreadsheet
- ล้าง cookie หลังใช้งานถ้าไม่จำเป็น
- หรือเก็บ cookie ใน UserProperties แทน sheet ถ้าต้องการ privacy ต่อผู้ใช้

---

# 13. Top 10 งานแก้ไขที่แนะนำตามลำดับ

## P0 — ต้องแก้ก่อน

1. แก้/ลบ `autoInstallSmartNav_()` phantom call  
2. แก้ duplicate `loadCachedGeoRows_()`  
3. เพิ่ม try-catch ให้ menu entry points สำคัญ  
4. เพิ่ม stack trace ให้ `logError()`  

## P1 — ควรแก้รอบถัดไป

5. แก้ hardcoded indexes ใน `18_ServiceSCG.gs`  
6. แก้ `setValue()` ใน loop ของ `15_GoogleMapsAPI.gs`  
7. แก้ `showVersionInfo()` จาก 21 files เป็น 22 files  
8. Refactor `autoEnrichAliasesFromFactBatch_()` ให้เล็กลง  

## P2 — ปรับคุณภาพระยะกลาง

9. ทำ resume state ให้ `runLookupEnrichment()`  
10. ลด global constants นอก `01_Config.gs` หรือห่อ namespace  

---

# 14. คำตัดสินโดยรวม

## สถานะระบบ

```text
Architecture: ดีมาก
Schema discipline: ดีมาก
Batch discipline: ดีแต่ยังมีจุดหลุด
Runtime risk: ปานกลาง
Maintainability risk: ปานกลางถึงสูง เพราะ function ยาวหลายจุด
LMDS rule compliance: ผ่านประมาณ 70–75%
Production readiness: ใช้ได้ แต่ควรแก้ P0 ก่อน
```

## จุดที่น่าชื่นชม

- โครงสร้าง 22-module ชัดเจน
- Header documentation ดีมาก
- มี blueprint/README ที่ละเอียด
- ใช้ IDX constants เป็นหลัก
- Hybrid Alias ออกแบบดี
- Pipeline มี Lock + Time Guard
- มี Human Review Queue
- มี Hardening module

## จุดที่ควรแก้ทันที

ประเด็นที่ “เสี่ยงจริง” ไม่ใช่แค่ style คือ:

1. `autoInstallSmartNav_()` ไม่มีจริง  
2. `loadCachedGeoRows_()` ซ้ำข้ามไฟล์  
3. หลาย menu entry points ไม่มี try-catch  
4. `logError()` ไม่มี stack trace  
5. hardcoded index ใน SCG aggregation  

---

# 15. สรุปสั้นสำหรับผู้บริหาร/เจ้าของโปรเจกต์

โค้ดชุดนี้เป็นระบบ LMDS ที่ออกแบบมาดีมากในเชิงสถาปัตยกรรม และมีความเป็น production-grade สูงกว่าทั่วไปของ Google Apps Script อย่างชัดเจน โดยเฉพาะ schema/index discipline, batch write, cache, match engine และ hybrid alias

แต่ยังมีจุดละเมิดกฎ LMDS ที่ควรแก้ก่อนใช้งานยาว ๆ ได้แก่ function phantom, function name collision, error handling ไม่ครบ, logging ยังไม่มี stack และ hardcoded index บางจุด

ถ้าแก้ P0/P1 ตามรายการด้านบน ระบบจะเสถียรและ maintain ง่ายขึ้นมาก โดยไม่ต้องเปลี่ยนสถาปัตยกรรมใหญ่ครับ