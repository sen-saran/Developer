---
# 🐳 Docker Installation Guide
---
## Ubuntu 24.04 LTS
### Step 1: อัปเดตระบบให้พร้อม
```bash
# ก่อนจะเริ่ม เราควรทำให้แน่ใจว่าดัชนีแพ็กเกจในเครื่องเป็นปัจจุบันที่สุด
sudo apt update && sudo apt upgrade -y
```

### Step 2: ติดตั้ง Dependencies ที่จำเป็น
```bash
# ติดตั้งเครื่องมือเพื่อให้ Ubuntu สามารถดึงข้อมูลผ่าน HTTPS ได้อย่างปลอดภัย
sudo apt install ca-certificates curl gnupg lsb-release -y
```
### Step 3: เพิ่ม Docker’s Official GPG Key
```bash
# ขั้นตอนนี้เป็นการยืนยันตัวตนว่าซอฟต์แวร์ที่เราจะโหลดมาจาก Docker โดยตรง ไม่ได้ถูกปลอมแปลง
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Step 4: ตั้งค่า Repository
```bash
# เราจะบอกให้ Ubuntu รู้ว่าต้องไปโหลด Docker จากที่ไหน (Official Repo)
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 5: ติดตั้ง Docker Engine
```bash
# หลังจากตั้งค่าแหล่งที่มาแล้ว ก็สั่งติดตั้งได้เลยครับ (ชุดนี้รวม Docker Compose มาให้ด้วย)
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```
---
# 🛠 การตรวจสอบและตั้งค่าหลังติดตั้ง
---
```bash
# ตรวจสอบสถานะ
sudo systemctl status docker
sudo systemctl enable docker
sudo systemctl start docker
# ทดสอบด้วย Hello World
sudo docker run hello-world
# รัน Docker โดยไม่ต้องพิมพ์ sudo (Optional แต่แนะนำ)
sudo usermod -aG docker $USER
newgrp docker
groups $USER
docker --version
docker info
# Container Management
docker ps
docker ps -a
docker start container_name
docker stop container_name
docker rm container_name
# Image Management
docker images
docker pull image_name
docker rmi image_name
docker build -t image_name .
```
