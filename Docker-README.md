# Docker for Developers 🚀


    Created: สร้าง Container จาก Image แล้วแต่ยังไม่ได้สั่ง Run

    Running: Container กำลังทำงาน (Primary Process กำลังทำงาน)

    Paused: หยุดการทำงานชั่วคราว (Freeze สถานะไว้ใน Memory)

    Stopped: หยุดการทำงาน (Process จบลง แต่ข้อมูลใน Container ยังอยู่)

    Deleted: ลบ Container ออกจากระบบ

---

## 1. Docker Lifecycle (วงจรชีวิต Container)
การจัดการสถานะของ Container ตั้งแต่สร้างจนถึงลบทิ้ง

### คำสั่งจัดการ Container
```bash
docker run -d --name my-app <image>  # สร้างและรัน Container ในพื้นหลัง (Background)
docker ps                            # ดู Container ที่กำลังรันอยู่
docker ps -a                         # ดู Container ทั้งหมด (รวมที่หยุดรันแล้ว)
docker stop <container_id>           # สั่งหยุดการทำงาน
docker start <container_id>          # สั่งให้กลับมาทำงานใหม่
docker rm -f <container_id>          # ลบ Container ทิ้งทันที
```

## 2. Docker Images (การจัดการภาพจำลอง)
Image คือพิมพ์เขียว (Template) ที่ใช้สร้าง Container

### คำสั่งจัดการ Container
```bash
docker images                        # ดูรายการ Image ในเครื่อง
docker pull <image_name>             # ดาวน์โหลด Image จาก Docker Hub
docker rmi <image_id>                # ลบ Image ออกจากเครื่อง
docker build -t my-image:v1 .        # สร้าง Image จาก Dockerfile ใน Folder ปัจจุบัน
```


## 3. Docker Volumes & Networking (เบื้องต้น)
ใช้สำหรับการเก็บข้อมูลถาวรและการเชื่อมต่อ

### คำสั่งจัดการ Container
```bash
# การใช้ Volume เพื่อไม่ให้ข้อมูลหายเมื่อลบ Container
docker run -d -v my_data:/app/data <image>

# การ Mapping Port (เครื่องเรา:ใน Container)
docker run -d -p 8080:80 <image>
```


## 4. Docker Compose (จัดการหลาย Container พร้อมกัน)
เครื่องมือที่ใช้รันโปรเจกต์ที่มีหลาย Service (เช่น Web + Database)

### คำสั่งจัดการ Container
```bash
docker-compose up -d                 # สั่งรันทุก Service ตามที่เขียนในไฟล์ yaml
docker-compose down                  # สั่งหยุดและลบ Container/Network ในโปรเจกต์
docker-compose logs -f               # ดู Log ของทุก Service แบบ Real-time
```
