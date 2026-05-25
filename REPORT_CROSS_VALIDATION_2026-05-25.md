# LMDS V5.4.001 — Cross-Validation รายงานคำแนะนำ AI 6 ท่าน

วันที่ตรวจ: 2026-05-25 (UTC)

## วิธีตรวจ (ตามข้อบังคับ)
- ตรวจจากโค้ดจริงเท่านั้น (`*.gs`) และไฟล์คำแนะนำทั้ง 6 ไฟล์
- ใช้ `rg -n` เพื่อหา evidence ทุกข้อกล่าวอ้าง
- ทุกข้อสรุประบุพิกัด `file:line` และ snippet

## ขอบเขตที่ตรวจ
- คำแนะนำ: `คำแนะนำclaude.md`, `คำแนะนำgemini.md`, `คำแนะนำgenspark.md`, `คำแนะนำminimax.md`, `คำแนะนำzai.md`, `คำแนะนำdeepseek.md`
- โค้ดจริง LMDS: `00_App.gs` ถึง `21_AliasService.gs`
- กฎมาตรฐาน: `กฎการเขียนโค้ด LMDS.md`, `📋กฎการเขียนโค้ด.md`

---

## A) ผล Cross-Validation (ประเด็น Critical/High)

### 1) Single Writer Pattern ถูกละเมิดจริง (CRITICAL)
**Evidence 1 (ประกาศนโยบาย):**
- `21_AliasService.gs:9` ระบุว่า auto pipeline ไม่ควรเขียน M_ALIAS ที่นี่

```javascript
* ⚠️ Auto Pipeline ไม่เขียน M_ALIAS ที่นี่ — เขียนที่ autoEnrichAliasesFromFactBatch_() เท่านั้น
```

**Evidence 2 (การเรียกจริง):**
- `18_ServiceSCG.gs:204` เรียก `populateAliasFromSCGRawData_()` จาก flow ดึง SCG

```javascript
populateAliasFromSCGRawData_();
```

**Evidence 3 (ฟังก์ชันที่ถูกเรียกเขียน M_ALIAS):**
- `21_AliasService.gs:559` มีฟังก์ชัน `populateAliasFromSCGRawData_()` และภายในเรียก `createGlobalAlias(...)` หลายจุด

**สรุป:** ข้อท้วงติงจาก Claude/Genspark/Zai ว่าเสี่ยงผิด Single Writer “มีมูล” และต้องแก้ก่อน deploy

---

### 2) Duplicate Function Name: `loadCachedGeoRows_` ซ้ำจริง (CRITICAL)
**Evidence:**
- `16_GeoDictionaryBuilder.gs:322` → `function loadCachedGeoRows_()`
- `07_PlaceService.gs:638` → `function loadCachedGeoRows_()`

**สรุป:** เป็น global-scope collision ตามกฎข้อ 8 (Namespace Collision)

---

### 3) Migration ไม่มี Time Guard จริง (CRITICAL)
**Evidence:**
- `21_AliasService.gs:451` ฟังก์ชัน `MIGRATION_HybridAliasSystem()`
- ภายในมีหลาย loop ขั้นใหญ่ (`Step 2/3/4/5`) แต่ไม่มีรูปแบบ time guard/checkpoint ชัดเจนในฟังก์ชันเดียวกัน

**สรุป:** คำเตือนเรื่อง timeout จาก Claude/Zai/Minimax “ถูกต้อง”

---

### 4) Row-by-row Spreadsheet write ใน update stats จริง (HIGH)
**Evidence:**
- `06_PersonService.gs:313` (`updatePersonStats`)
- `07_PlaceService.gs:595` (`updatePlaceStats`)
- `08_GeoService.gs:279` (`updateGeoStats`)

ทั้งสามฟังก์ชันมี pattern `getRange(...).setValue(...)` และ/หรือ read-modify-write แบบรายแถว

**สรุป:** เป็น performance risk ตามกฎ Batch Operations

---

### 5) Phantom function: `autoInstallSmartNav_` “ไม่พบ definition” (HIGH)
**Evidence:**
- `00_App.gs:79` มี `try { autoInstallSmartNav_(); } catch (_) {}`
- จากการค้น `rg -n "function autoInstallSmartNav_" *.gs` ไม่พบ definition

**สรุป:** คำเตือนกลุ่มที่บอกว่าอาจเป็น fake call มีมูลในประเด็นนี้

---

## B) คัดกรองความถูกต้องของ AI ทั้ง 6 ท่าน (เฉพาะที่ยืนยันด้วยโค้ดจริง)

### Claude
- **ถูก:** ชี้ Single Writer violation, duplicate `loadCachedGeoRows_`, migration ไม่มี time guard, row-by-row stats
- **ต้องระวัง:** ข้อกล่าวอ้างบางส่วนที่อิงไฟล์ docs เก่า/นอกโค้ดจริง ต้องไม่ใช้ตัดสิน production โดยตรง
- **Verdict:** แม่นยำสูงสุดในกลุ่มที่ตรวจ

### Genspark
- **ถูก:** โครงสร้างระบบ, หลายประเด็น performance/compliance
- **ผิด/เกินจริง:** บางสรุป “ผ่านบางส่วน” ต้องลงหลักฐานรายบรรทัดเพิ่ม
- **Verdict:** ใช้เป็นฐานได้ แต่ต้อง line-by-line verify

### Zai
- **ถูก:** หลาย critical ซ้ำกับ Claude (โดยเฉพาะ duplicate function + migration risk)
- **ต้องระวัง:** มีบางข้อเป็นการตีความเชิงแนวคิดมากกว่าหลักฐานตรง
- **Verdict:** คุณภาพดี รองจาก Claude

### Minimax
- **ปัญหา:** พบ claim ที่ PASS ทั้งที่ยังมีจุดเสี่ยงจริงในโค้ด
- **Verdict:** ใช้ได้บางส่วน แต่ไม่พอสำหรับ production sign-off

### Gemini
- **ปัญหา:** มีลักษณะสรุประดับ architecture สูง แต่ evidence แบบ grep/line ยังไม่พอ
- **Verdict:** ใช้เป็นภาพรวม ไม่ควรใช้เป็นตัวตัดสิน bug list

### DeepSeek
- **ปัญหา:** เน้นแผน implement ใหม่มากกว่า audit ความจริงของโค้ดปัจจุบัน
- **Verdict:** เหมาะใช้เป็นแรงบันดาลใจ refactor ไม่ใช่ผล audit ที่ final

---

## C) Implementation Plan (ทำได้ทันที)

### Phase 0 — Hotfix ก่อน Deploy
1. **ปิดจุดเขียน M_ALIAS นอก Single Writer**
   - เอา call `populateAliasFromSCGRawData_()` ออกจาก flow อัตโนมัติใน `18_ServiceSCG.gs`
   - คงไว้เป็นเมนู admin/manual เท่านั้น
2. **แก้ global collision**
   - เปลี่ยนชื่อ `loadCachedGeoRows_` ในไฟล์ใดไฟล์หนึ่ง เช่น `loadCachedGeoRowsForPlace_`
   - ปรับ call sites ให้ครบ
3. **เพิ่ม time guard ให้ migration**
   - แทรก guard + checkpoint ทุก N records ใน `MIGRATION_HybridAliasSystem()`
4. **หยุด row-by-row stats write**
   - สะสม delta ใน memory แล้ว batch update ท้ายรอบ

### Phase 1 — High Priority Hardening
5. **จัดการ phantom function**
   - นิยาม `autoInstallSmartNav_` จริง หรือเอา call ออกให้ชัดเจน
6. **Entry point try/catch audit**
   - ฟังก์ชันที่ผูกเมนู/trigger ต้องมี try/catch + logError
7. **Performance lint checks**
   - เพิ่มสคริปต์ตรวจ `setValue(` ใน loop และ duplicate function names ก่อน merge

### Phase 2 — Quality Gate ก่อน Production
8. **Pre-deploy checklist อัตโนมัติ**
   - ตรวจ Phantom, Duplicate names, Hardcode indexes, Time Guard, Batch writes
9. **Regression tests (manual+scripted)**
   - ทดสอบ Group 1 pipeline + Group 2 search flow หลัง refactor

---

## D) สรุปสำหรับ Developer
- ถ้าต้อง “เลือกคำแนะนำที่เชื่อถือที่สุดตอนนี้”: **Claude + Zai** (แต่ต้องยึดโค้ดจริงเป็นหลักทุกบรรทัด)
- ประเด็นที่ควรแก้ทันที: **Single Writer violation, duplicate function names, migration time guard, row-by-row writes, phantom function call**
- เอกสารฉบับนี้พร้อมใช้เป็น task list สำหรับ implementation sprint ถัดไป
