# คู่มือ IBM i Agent Bridge v0.2.2

IBM i Agent Bridge เพิ่มเครื่องมือใน GitHub Copilot Chat สำหรับค้นตำแหน่ง source member ค้นข้อความอ้างอิง อ่าน source และเสนอการแก้ไข โดยใช้ connection ที่เปิดอยู่ใน Code for IBM i ควรเริ่มจาก DEV และใช้โมเดลที่องค์กรอนุมัติสำหรับ source ที่ส่งให้ Copilot

## ติดตั้งและเชื่อมต่อ

1. ใช้ VS Code Desktop 1.105 ขึ้นไป พร้อม **Code for IBM i** และ **GitHub Copilot Chat**
2. ดาวน์โหลด [ibmi-agent-bridge-0.2.2.vsix](./ibmi-agent-bridge-0.2.2.vsix) จาก GitHub โดยกด **Download raw file**
3. กด `Ctrl+Shift+P` เลือก **Extensions: Install from VSIX…** แล้วเลือกไฟล์ที่ดาวน์โหลด
4. เปิด Code for IBM i และเชื่อมต่อ DEV จากนั้นเปิด Copilot Chat ใน Agent session ที่เรียกใช้ extension tools ได้ เปิดเครื่องมือของ Bridge ใน Tool Picker หากยังไม่ปรากฏ

## ตั้งค่า library

เปิด **Preferences: Open User Settings (JSON)** แล้วเพิ่มคีย์นี้ใน settings เดิม เปลี่ยน `MY-IBM-I-DEV` เป็น **ชื่อ connection ที่แสดงใน Code for IBM i** และใส่ library ที่ต้องการค้นจริง

```json
{
  "ibmiAgentBridge.extraSearchLibraries": {
    "MY-IBM-I-DEV": ["DEVLIB", "COMMONSRC"]
  }
}
```

| เครื่องมือ | ใช้เมื่อ | ขอบเขต |
|---|---|---|
| `#ibmiFindMembers` | รู้ชื่อ member และต้องการหา path | `*LIBL` ของ SQL job, Current Library และ User Library List ของ connection รวมถึง library ที่ตั้งเพิ่ม |
| `#ibmiFindReferences` | ต้องการหาว่า member ไหนมีข้อความที่ระบุ | เฉพาะ library ใน `extraSearchLibraries` ของ connection ปัจจุบัน ค้นด้วย `FNDSTRPDM` |
| `#ibmiReadMember` | รู้ path และต้องการอ่าน source ที่บันทึกแล้ว | ระบุ library, source file และ member ตามสิทธิ์ของ IBM i profile |
| `#ibmiReadActive` | ต้องการอ่าน source ใน editor ที่เปิดอยู่ | รวมการแก้ไขที่ยังไม่ได้บันทึก |
| `#ibmiProposePatch` | ต้องการดู diff ก่อนแก้ source | ใช้ snapshot ของ member ที่เปิดอยู่ และต้องตั้งสิทธิ์แก้ไขสำหรับ DEV |

ทั้งสองคำสั่งค้นหาจะเลือกเฉพาะ source physical file ที่ profile มองเห็นใน catalog และอยู่ในรายการนี้

```text
QCBLLESRC  QCLLESRC   QCLSRC     QCMDSRC    QDDSSRC
QLBLSRC    QMENUSRC   QPNLSRC    QQMFORMSRC QQMQRYSRC
QRPGLESRC  QRPGSRC    QSQLSRC    QTEMPSRC   QTXTSRC
```

`QTEMPSRC` คือชื่อ source file หาก library มีไฟล์ชื่อนี้และอยู่ในขอบเขตจึงจะถูกค้น

## ทดลองค้นโปรแกรมที่เรียกโปรแกรมอื่น

เลือกชื่อโปรแกรมจริงที่รู้ว่ามี caller ใน DEV ตัวอย่างนี้ใช้ `ORD001`

> ใช้ #ibmiFindMembers หา member ชื่อ ORD001 แล้วแสดง path ของแต่ละตัวที่พบ

> ใช้ #ibmiFindReferences ค้น ORD001 ให้ครบทุกหน้าจน nextCursor เป็น null สรุป library/source file/member ที่พบ และแยกไฟล์ที่รายงานหาย ถูกตัด หรือค้นไม่สำเร็จ

`#ibmiFindReferences` ให้ IBM i ค้นข้อความจาก source ที่บันทึกแล้วด้วย PDM โดยค้นแบบไม่แยกตัวพิมพ์เล็ก–ใหญ่และจับข้อความบางส่วน เช่น `ORD001` อาจตรงกับ `ORD001A` หรืออยู่ใน comment ได้ เมื่อพบ candidate ให้ใช้ `#ibmiReadMember` ตรวจคำสั่งเรียกจริง เช่น `CALL PGM(ORD001)` ชื่อโปรแกรมที่ประกอบขึ้นตอนรันอาจค้นไม่เจอด้วยวิธีนี้

### อ่านผลจาก PDM

Bridge ประมวลผลครั้งละไม่เกิน 4 source files และแสดงสูงสุด 100 บรรทัดของรายงานต่อไฟล์ ให้ Copilot ใช้ `nextCursor` เดิมกับชื่อค้นเดิมเพื่อไปหน้าถัดไป

| สถานะ | วิธีตีความ |
|---|---|
| `report` | อ่านรายงานได้ ตรวจรายชื่อ member ในรายงาน |
| `no-report` | ไม่พบ spool ที่ Bridge ใช้อ่านรายงาน ผลของไฟล์นี้ยังยืนยันไม่ได้ |
| `truncated` | รายงานเกินขนาด preview ใช้ `retainedSpool` เปิดรายงานเต็มใน `WRKSPLF` |
| `error` | คำสั่งหรือการอ่านรายงานล้มเหลว ดูรายละเอียดใน `error` และชื่อ `spoolName` |

Bridge สร้าง spool แบบ held และพยายามลบเฉพาะ spool ของตัวเองหลังอ่านรายงานครบ หากเห็น `retainedSpool` หรือ `cleanupError` ให้ใช้ข้อมูลนั้นหา spool ที่ค้างอยู่

## ทดลองเสนอแก้ไข source บน DEV

เพิ่มค่าต่อไปนี้ใน User Settings เดิมเมื่อจะใช้การเสนอแก้ไข เปลี่ยนชื่อ connection และ library ให้ตรงกับ DEV ของคุณ

```json
{
  "ibmiAgentBridge.allowedWriteConnection": "MY-IBM-I-DEV",
  "ibmiAgentBridge.allowedWriteLibraries": ["DEVLIB"],
  "files.autoSave": "off"
}
```

1. เปิด member สำหรับทดสอบใน Object Browser แล้วบันทึกการแก้ไขที่ค้างอยู่
2. ให้ Copilot ใช้ `#ibmiReadActive` แล้วใช้ `#ibmiProposePatch` เสนอแก้ comment หนึ่งบรรทัด ตรวจ diff และตำแหน่งคอลัมน์ของ fixed-format source
3. เลือก **IBM i Agent Bridge: Apply Reviewed Patch** และยืนยัน จากนั้นกด `Ctrl+S` เพื่อเขียนลง IBM i หรือเลือก **Discard Patch** หากไม่ต้องการใช้ diff
4. เปิด member ใหม่เพื่อยืนยันผล แล้ว compile ผ่าน workflow DEV ของทีมตามปกติ

## แก้ปัญหาเบื้องต้น

| อาการ | สิ่งที่ควรตรวจ |
|---|---|
| Code for IBM i Connect แล้ว timeout | ตรวจ VPN และ host/port ที่ตั้งไว้ ใช้ `Test-NetConnection -ComputerName HOST -Port 22` โดยเปลี่ยนเป็นพอร์ตจริง และเทียบ `RemoteAddress` กับ IP ที่ทีมยืนยัน |
| Copilot หาเครื่องมือ Bridge ไม่เจอ | ตรวจว่าติดตั้ง v0.2.2 แล้วและเปิดเครื่องมือใน Tool Picker ของ Agent session |
| ค้น references แล้วไม่เห็น library ที่ต้องการ | ตรวจชื่อ connection ใน User Settings และดู `scope.libraries` ในผลลัพธ์ เพิ่ม library นั้นใน `extraSearchLibraries` แม้จะอยู่ใน `*LIBL` แล้ว |
| ค้น references ไม่ครบหรือช้า | ลองหนึ่งหน้าแล้วดู `reports` กับ `nextCursor` เพื่อแยกปัญหาการค้นแต่ละไฟล์ออกจากจำนวนหน้าทั้งหมด |
| รายงาน PDM ผิดพลาด | ตรวจ Output ของ Code for IBM i และสิทธิ์ของ profile ต่อ source และ spool ฝั่งระบบต้องมี PDM, `QSYS2.QCMDEXC`, `QSYS2.SYSTABLES`, `QSYS2.SPOOLED_FILE_INFO` และ `SYSTOOLS.SPOOLED_FILE_DATA` |

ถ้าต้องการแยกว่า PDM ค้นไม่เจอหรือ Bridge อ่านรายงานไม่ได้ ให้ทดสอบคำสั่งเดียวกันกับ source file ที่รู้ว่ามี caller บน DEV คำสั่งนี้สร้างรายงาน PDM ใน spool

```cl
FNDSTRPDM STRING('ORD001') FILE(DEVLIB/QCLSRC) MBR(*ALL) OPTION(*NONE) CASE(*IGNORE) PRTMBRLIST(*YES) PRTRCDS(*NONE)
```
