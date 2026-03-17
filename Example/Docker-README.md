# Docker for Developers 🚀

คู่มือสรุปคำสั่ง Docker ที่จำเป็นสำหรับการพัฒนาซอฟต์แวร์ ตั้งแต่การจัดการ Container ไปจนถึงการทำ Orchestration เบื้องต้นด้วย Compose

---

## 1. Docker Lifecycle (วงจรชีวิต Container)
การจัดการสถานะของ Container ตั้งแต่เริ่มต้นจนถึงการทำลายทิ้ง



### สถานะที่สำคัญ:
* **Created**: สร้าง Container จาก Image แล้วแต่ยังไม่ได้สั่ง Run
* **Running**: Container กำลังทำงาน (Primary Process กำลังทำงาน)
* **Paused**: หยุดการทำงานชั่วคราว (Freeze สถานะไว้ใน Memory)
* **Stopped**: หยุดการทำงาน (Process จบลง แต่ข้อมูลใน Container ยังอยู่)
* **Deleted**: ลบ Container ออกจากเครื่องอย่างถาวร

### คำสั่งจัดการสถานะ (Container Management)
### คำสั่งจัดการ Container
```bash
docker run -d --name my-app <image>  # สร้างและรัน Container ในพื้นหลัง (Detached Mode)
docker ps                            # ดู Container ที่กำลังรันอยู่ (Running)
docker ps -a                         # ดู Container ทั้งหมด (รวมที่หยุดรันหรือแค่สร้างไว้)
docker stop <container_id>           # สั่งหยุดการทำงาน (Transition to Stopped)
docker start <container_id>          # สั่งให้กลับมาทำงานใหม่ (Transition to Running)
docker rm -f <container_id>          # ลบ Container ทิ้งทันที (Force Remove)
```
## 2. Docker Images (การจัดการภาพจำลอง)
Image คือ "พิมพ์เขียว" (Template) ที่บรรจุ OS, Library และ Code ที่จำเป็นต้องใช้

### คำสั่งจัดการ Image
```bash
docker images                        # ดูรายการ Image ทั้งหมดที่มีในเครื่อง
docker pull <image_name>             # ดาวน์โหลด Image จาก Docker Hub หรือ Registry
docker rmi <image_id>                # ลบ Image ออกจากเครื่อง (Remove Image)
docker build -t my-image:v1 .        # สร้าง Image ใหม่จาก Dockerfile ใน Folder ปัจจุบัน
```


## 3. Docker Volumes & Networking (การเก็บข้อมูลและการเชื่อมต่อ)
ใช้สำหรับการจัดการข้อมูลให้คงอยู่ (Persist) และการเข้าถึงผ่าน Network

### คำสั่งจัดการ Data & Port
```bash
# Volume: ผูก Folder เครื่องเราเข้ากับ Container เพื่อไม่ให้ข้อมูลหาย
docker run -d -v my_data:/app/data <image>

# Port Mapping: (Port เครื่องเรา : Port ใน Container)
docker run -d -p 8080:80 <image>     # เข้าใช้งานผ่าน localhost:8080
```


## 4. Docker Compose (การจัดการหลาย Container)
เครื่องมือสำหรับรันระบบที่มีหลาย Service พร้อมกัน (เช่น Frontend + Backend + DB)

### คำสั่งใช้งาน Docker Compose
```bash
docker-compose up -d                 # สั่งรันทุก Service ตามที่เขียนในไฟล์ docker-compose.yml
docker-compose down                  # สั่งหยุดและลบ Container, Network และ Image (บางส่วน)
docker-compose logs -f               # ติดตาม Log ของทุก Service แบบ Real-time
docker-compose exec <service> bash   # เข้าไปใช้งาน Terminal ภายใน Container ที่ระบุ
```
