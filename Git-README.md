# Git for Developers 🚀

ไฟล์นี้รวบรวมคำสั่งพื้นฐานและการตั้งค่าที่จำเป็นสำหรับการใช้งาน **Git Version Control** ในการพัฒนาซอฟต์แวร์

---

## 1. Git First-Time Setup (การตั้งค่าครั้งแรก)
การกำหนดตัวตนผู้ใช้งาน เพื่อให้ระบบบันทึกว่าใครเป็นคนแก้ไข Code

### กำหนดข้อมูลผู้ใช้
```bash
git config --global user.name "Your Name"
git config --global user.email yourname@example.com
```

### ตั้งค่าชื่อ Branch หลักเป็น "main"
```bash
git config --global init.defaultBranch main
```

### ตรวจสอบการตั้งค่า
```bash
git config --list                 # ดูการตั้งค่าทั้งหมด
git config --global --list        # ดูเฉพาะการตั้งค่าระดับ Global
git config --list --show-origin   # ดูที่มาของไฟล์การตั้งค่า (System/Global/Local)
```

# 2. Git Workflow (วงจรการทำงาน)
* Working Directory: ไฟล์ที่เรากำลังแก้ไข (สถานะ U - Untracked หรือ Modified)
* Staging Area: ไฟล์ที่เตรียมจะ Commit (สถานะ A - Added)
* Local Repository: ไฟล์ที่ถูกบันทึกประวัติลงฐานข้อมูล Git แล้ว

### คำสั่งใช้งานพื้นฐาน
```bash
git status                     # ตรวจสอบสถานะไฟล์ปัจจุบัน
git add <file_name>            # เพิ่มไฟล์เฉพาะเจาะจงเข้า Staging Area
git add .                      # เพิ่มไฟล์ทั้งหมดที่แก้ไข/สร้างใหม่เข้า Staging Area
git commit -m "หัวข้อ: รายละเอียด"  # บันทึกการเปลี่ยนแปลงพร้อมคำอธิบาย
```

# 3. History & Comparison (ตรวจสอบประวัติและความต่าง)
### ดูประวัติการแก้ไข
```bash
git log                        # ดูประวัติการ Commit ทั้งหมดแบบละเอียด
git log --oneline              # ดูประวัติแบบย่อบรรทัดเดียว (สะดวกในการหา Commit ID)
git log -n 5                   # ดูประวัติย้อนหลังแค่ 5 รายการล่าสุด
```
### เช็คความแตกต่าง (Diff)
```bash
git diff                       # ดูความต่างของไฟล์ที่ยังไม่ได้ add
git diff --staged              # ดูความต่างของไฟล์ที่ add เข้า staging แล้ว
git diff <commit_id1> <commit_id2> # เทียบความต่างระหว่าง 2 Commit
```

# 4. Undo & Recovery (การย้อนกลับเมื่อเกิดข้อผิดพลาด)
### ย้อนกลับไปดูเวอร์ชั่นเก่า (Temporary)
```bash
git checkout <commit_id_7_digits> # ย้อนไปดู Code ในอดีต (สถานะ Read-only)
git checkout main                # กลับมาที่เวอร์ชันล่าสุดใน Branch หลัก
```
### การยกเลิกการเปลี่ยนแปลง (Restore/Reset)
```bash
git restore <file>             # ยกเลิกการแก้ไขไฟล์ที่ยังไม่ถูก add (ย้อนกลับไปสภาพล่าสุดที่ commit)
git reset HEAD~1               # ยกเลิก Commit ล่าสุด (ไฟล์ที่แก้ยังอยู่แต่ถูกดึงออกจาก staging)
git reset --hard HEAD~1        # ย้อนกลับ 1 Commit และลบสิ่งที่แก้ไขทิ้งทั้งหมด (ระวัง: ข้อมูลจะหายถาวร!)
```


# 5. Working with Remote (การเชื่อมต่อกับ Server)
### เชื่อมต่อและส่ง Code ขึ้น Server
```bash
git remote add origin <url>    # เชื่อม Local Repo เข้ากับ Remote Server
git remote -v                  # ตรวจสอบ URL ของ Remote ที่เชื่อมต่ออยู่
git push -u origin main        # ส่ง Code ขึ้น Server (ใช้ -u เฉพาะครั้งแรกเพื่อจำค่า)
git pull origin main           # ดึง Code ล่าสุดจาก Server ลงมาอัปเดตที่เครื่องเรา
```

# 6. Git Branching (การแยกสายพัฒนา)
### ส่วนที่เพิ่มเติมเพื่อการทำงานเป็นทีม
```bash
git branch <branch_name>       # สร้าง Branch ใหม่
git checkout <branch_name>     # สลับไปยัง Branch ที่ต้องการ
git checkout -b <branch_name>  # สร้าง Branch ใหม่และสลับไปใช้งานทันที
git merge <branch_name>        # รวม Branch ที่ระบุ เข้ากับ Branch ปัจจุบันที่เราอยู่
git branch -d <branch_name>    # ลบ Branch ที่ไม่ใช้แล้ว (หลังจาก merge แล้ว)
```
