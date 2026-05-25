# LMDS — Bug Hunt (Critical & High Priority)

อ้างอิงจากโค้ดจริงเท่านั้น (ตรวจด้วย `rg -n`)

## 🔴 BUG-01: Hardcoded Array Indexes ใน 18_ServiceSCG.gs
- **ไฟล์:** `18_ServiceSCG.gs:162-176`
- **โค้ด:**
```javascript
const key = r[28];
shopAgg[key].qty += Number(r[14]) || 0;
shopAgg[key].weight += Number(r[16]) || 0;
shopAgg[key].invoices.add(r[2]);
if (checkIsEPOD(r[9], r[2])) shopAgg[key].epod++;
...
r[23] = agg.qty;
r[24] = Number(agg.weight.toFixed(2));
r[25] = scanInv;
r[27] = `${r[9]} / รวม ${scanInv} บิล`;
```
- **ผลกระทบ:** เสี่ยงพังทันทีเมื่อ schema ขยับคอลัมน์ และขัดกฎ No Hardcode Index.
- **ข้อเสนอแก้ไข:** ย้ายเป็น constant index (`DATA_IDX.*`) จาก `02_Schema.gs` แล้วแทนทุก `r[n]`.

## 🔴 BUG-02: Single Writer Pattern Violation (M_ALIAS เขียนนอก autoEnrich)
- **ไฟล์:** `21_AliasService.gs:502,523,654,701,711` และ `19_Hardening.gs:419`
- **โค้ด:**
```javascript
var result = createGlobalAlias(masterUuid, aliasName, 'PERSON', matchScore, 'V52_LEGACY_MIGRATION');
var result = createGlobalAlias(masterUuid, aliasName, 'PLACE', matchScore, 'V52_LEGACY_MIGRATION');
var result = createGlobalAlias(matchedUuid, rawName, matchedType, 90, 'SCG_RAW_IMPORT');
var result = createGlobalAlias(masterUuid, info.rawName, 'PERSON', 95, 'FACT_DELIVERY_IMPORT');
const result = createGlobalAlias(...); // in generatePersonAliasesFromHistory
```
- **ผลกระทบ:** ถ้าตีความตามกฎ “ห้ามเขียน M_ALIAS นอก `autoEnrichAliasesFromFactBatch_`” จะถือว่า breach ของ write-path เดียว.
- **ข้อเสนอแก้ไข:**
  1) กำหนดนโยบายให้ชัดว่า admin/migration อนุญาตหรือไม่
  2) ถ้าไม่อนุญาต ให้เปลี่ยนทุกเส้นทางไป buffer แล้วเขียนผ่าน writer เดียว
  3) ถ้าอนุญาตเฉพาะ admin ให้ lock+label source ให้ชัด และกันรันซ้อน.

## 🔴 BUG-03: Missing Time Guard ในงานยาวนอก MIGRATION
- **ไฟล์:** `21_AliasService.gs:575-717` (`populateAliasFromSCGRawData_`, `populateAliasFromFactDelivery_`)
- **โค้ด:**
```javascript
for (var normKey in nameCount) { ... createGlobalAlias(...) }
for (var normKey in nameMap)  { ... createGlobalAlias(...) }
```
- **ผลกระทบ:** วนข้อมูลจำนวนมากได้โดยไม่มี time checkpoint/resume ในฟังก์ชันเอง เสี่ยงชนเพดานเวลา GAS.
- **ข้อเสนอแก้ไข:** เพิ่ม `ensureTime_`/checkpoint ทุก N records และ resume token ด้วย PropertiesService.

## 🟠 BUG-04: Row-by-Row Write Operations
- **ไฟล์:** `15_GoogleMapsAPI.gs:282-287`, `09_DestinationService.gs:190-198`, `12_ReviewService.gs:343-386`
- **โค้ด:**
```javascript
for (let i = 0; i < data.length; i++) {
  sheet.getRange(i + 2, 8).setValue(Number(data[i][MC_HIT] || 0) + 1);
}

sheet.getRange(targetRow, lastSeenCol).setValue(now);
sheet.getRange(targetRow, usageCountCol).setValue(curr + 1);

sheet.getRange(targetRow, REVIEW_IDX.STATUS + 1).setValue('Done');
sheet.getRange(targetRow, REVIEW_IDX.REVIEWER + 1).setValue(reviewer);
```
- **ผลกระทบ:** API calls สูง, ช้า, quota เสี่ยง, timeout เสี่ยง.
- **ข้อเสนอแก้ไข:** อ่าน-เขียนเป็นช่วง (range) ต่อแถว/ต่อชุด หรือสะสม array แล้ว `setValues()` ทีเดียว.

## 🟠 BUG-05: Entry Points Without Try-Catch
- **ไฟล์:** `00_App.gs:209` (`installSmartNavTrigger`), `21_AliasService.gs:397` (`assignMasterUuidIfMissing`), `21_AliasService.gs:575` (`populateAliasFromSCGRawData_`)
- **โค้ด:**
```javascript
function installSmartNavTrigger() {
  const triggers = ScriptApp.getProjectTriggers();
  ...
}

function assignMasterUuidIfMissing() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  ...
}

function populateAliasFromSCGRawData_() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  ...
}
```
- **ผลกระทบ:** เมนู/แอดมิน action ล้มเหลวแล้วสะท้อนผู้ใช้ไม่สม่ำเสมอ, debug ยาก.
- **ข้อเสนอแก้ไข:** ครอบ top-level ด้วย try/catch + `logError` + `ui.alert`.

## 🟠 BUG-06: Hardcoded Index สายใน Google Maps Cache
- **ไฟล์:** `15_GoogleMapsAPI.gs:62-64,320-321`
- **โค้ด:**
```javascript
// [0] cache_key ... [9] district
result.province || '',  // col [8]
result.district || '',  // col [9]
```
- **ผลกระทบ:** ผูกกับตำแหน่งคอลัมน์คงที่โดยไม่ใช้ constant map.
- **ข้อเสนอแก้ไข:** สร้าง `MAP_CACHE_IDX` constants แล้ว refactor ให้ใช้ symbolic index.

---

## Pre-Deploy Status (จากการ grep รอบนี้)
- **Critical ผ่าน:** 0/5
- **Critical ไม่ผ่าน:** 3/5 (BUG-01, BUG-02, BUG-03)
- **High ไม่ผ่าน:** 3/2+ (BUG-04, BUG-05, BUG-06)

> สรุป: **ยังไม่พร้อม Deploy** ต้องแก้ Critical ก่อน.
