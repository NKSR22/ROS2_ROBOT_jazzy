# pi5_ros_robot

คู่มือนี้ออกแบบมาสำหรับการเริ่มต้นระบบหุ่นยนต์ ROS 2 Jazzy ที่แยกงานเป็น 2 ฝั่ง:

- `pc/` สำหรับงานประมวลผลหนัก เช่น Nav2, SLAM, Gazebo และ RViz
- `rpi5/` สำหรับอ่าน LiDAR, กล้อง และเชื่อม ESP32 ผ่าน micro-ROS Agent

แนวทางนี้เน้นให้ ROS 2 รันใน Docker เป็นหลัก เพื่อให้เครื่อง host ติดตั้งน้อยที่สุดและย้ายไปใช้งานบนเครื่องอื่นได้ง่าย

## สถาปัตยกรรมระบบ

```text
PC (Ubuntu Desktop + Docker)
  |- Nav2
  |- SLAM Toolbox
  |- Gazebo / RViz
  `- subscribe topics จาก Raspberry Pi 5

Raspberry Pi 5 (Raspberry Pi OS Lite 64-bit + Docker)
  |- LiDAR node
  |- Camera node
  `- micro-ROS Agent for ESP32
```

ทั้งสองฝั่งใช้:

- ROS 2 Jazzy
- `rmw_cyclonedds_cpp`
- `ROS_DOMAIN_ID` เดียวกัน
- `network_mode: host` เพื่อให้ ROS 2 discovery ทำงานง่ายขึ้น

## โครงสร้างโฟลเดอร์

```text
.
|-- .gitignore
|-- README.md
|-- pc/
|   |-- Dockerfile
|   |-- docker-compose.yml
|   `-- src/
`-- rpi5/
    |-- Dockerfile
    |-- docker-compose.yml
    `-- src/
```

## แนวทาง OS ที่แนะนำ

อ้างอิงตามสถานะปัจจุบัน ณ วันที่ 13 มีนาคม 2026:

- PC: `Ubuntu Desktop 24.04 LTS` 64-bit
- Raspberry Pi 5: `Raspberry Pi OS Lite` 64-bit รุ่นล่าสุดที่มีใน Raspberry Pi Imager

เหตุผล:

- ROS 2 Jazzy รองรับ Ubuntu 24.04 อย่างเป็นทางการ
- ฝั่ง PC ต้องใช้ Linux เพื่อให้ `network_mode: host` และ X11 GUI forwarding ทำงานตรงไปตรงมาที่สุด
- ฝั่ง Pi ใช้งาน Docker เป็นหลัก จึงเลือก Raspberry Pi OS Lite 64-bit เพื่อให้เบาและเข้าถึงฮาร์ดแวร์สะดวก

ถ้าต้องการให้ทั้งสองฝั่งใช้ Ubuntu เหมือนกันทั้งหมด ก็สามารถใช้ `Ubuntu Server 24.04 LTS` บน Pi 5 ได้เช่นกัน

## สิ่งที่ต้องเตรียม

- PC 1 เครื่อง
- Raspberry Pi 5
- microSD หรือ SSD สำหรับ Pi
- LiDAR ที่รองรับ ROS 2 package `sllidar_ros2`
- กล้องที่มองเห็นเป็น `/dev/video*`
- ESP32 ที่แฟลช micro-ROS firmware แล้ว
- สาย USB สำหรับ LiDAR และ ESP32
- PC และ Pi อยู่ในวง LAN เดียวกัน

## ส่วนที่ 1: ติดตั้งระบบปฏิบัติการบน PC

### 1.1 ติดตั้ง Ubuntu Desktop 24.04 LTS

1. ดาวน์โหลด Ubuntu Desktop 24.04 LTS จากหน้า official
2. สร้าง USB boot
3. ติดตั้งแบบปกติ
4. เชื่อมต่ออินเทอร์เน็ต
5. login เข้าระบบ

### 1.2 อัปเดตแพ็กเกจพื้นฐาน

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y git curl ca-certificates x11-xserver-utils mesa-utils
sudo reboot
```

### 1.3 ทดสอบหลังติดตั้ง Ubuntu

```bash
uname -a
lsb_release -a
echo $XDG_SESSION_TYPE
```

สิ่งที่ควรเห็น:

- Ubuntu 24.04
- มี desktop session
- ถ้าใช้ X11 จะใช้งาน `xhost` ได้ตรงไปตรงมา

ถ้าเครื่องใช้ Wayland อยู่ ยังใช้งานได้ในหลายกรณี แต่ถ้าจะเปิด GUI จาก container แล้วมีปัญหา ให้สลับไปใช้ Xorg ที่หน้า login

## ส่วนที่ 2: ติดตั้งระบบปฏิบัติการบน Raspberry Pi 5

### 2.1 Flash Raspberry Pi OS Lite 64-bit

1. ติดตั้ง Raspberry Pi Imager บน PC
2. เลือก `Raspberry Pi 5`
3. เลือก `Raspberry Pi OS Lite (64-bit)` รุ่นล่าสุดในโปรแกรม
4. ตั้งค่าในเมนู advanced options ก่อน write

ค่าที่แนะนำ:

- hostname: `pi5-robot`
- เปิด `SSH`
- ตั้ง username/password
- ตั้งค่า Wi-Fi ถ้าไม่ได้ใช้สาย LAN
- ตั้ง locale และ timezone ให้ถูกต้อง

### 2.2 บูต Pi ครั้งแรก

เสียบ storage, เปิดเครื่อง, แล้วหา IP ของ Pi จาก router หรือใช้:

```bash
ping pi5-robot.local
ssh <username>@pi5-robot.local
```

### 2.3 อัปเดตระบบบน Pi

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

### 2.4 ทดสอบพื้นฐานบน Pi

หลัง reboot:

```bash
ssh <username>@pi5-robot.local
uname -a
cat /etc/os-release
ip a
```

สิ่งที่ควรตรวจ:

- เป็นระบบ 64-bit
- Pi ออกเน็ตได้
- PC สามารถ `ping` ถึง Pi ได้

## ส่วนที่ 3: ติดตั้ง Docker บนทั้งสองอุปกรณ์

คู่มือนี้ใช้ Docker Engine + Docker Compose plugin

### 3.1 ติดตั้ง Docker บน Ubuntu PC

รันตามนี้บน PC:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3.2 ติดตั้ง Docker บน Raspberry Pi OS

รันตามนี้บน Pi:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3.3 ให้ user ใช้ Docker โดยไม่ต้องพิมพ์ sudo

ทำทั้งบน PC และ Pi:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

หรือ logout/login ใหม่หนึ่งรอบ

### 3.4 ทดสอบ Docker

```bash
docker --version
docker compose version
docker run hello-world
```

ถ้าคำสั่งสุดท้ายรันผ่าน แปลว่า Docker พร้อมใช้งาน

## ส่วนที่ 4: ดาวน์โหลดโปรเจ็กต์

บนทั้ง PC และ Pi ให้ clone รีโปเดียวกัน:

```bash
git clone https://github.com/NKSR22/ROS2_ROBOT_jazzy.git
cd ROS2_ROBOT_jazzy
```

ถ้าคุณกำลังใช้งานรีโปนี้ในชื่อโฟลเดอร์อื่นก็ไม่มีปัญหา ขอแค่มีโครงสร้าง `pc/` และ `rpi5/` ตามไฟล์ในโปรเจ็กต์

## ส่วนที่ 5: ตรวจสอบอุปกรณ์ที่ต่อเข้ากับ Raspberry Pi 5

ต่อ LiDAR, กล้อง และ ESP32 เข้ากับ Pi แล้วตรวจสอบ:

```bash
ls /dev/ttyUSB*
ls /dev/ttyACM*
ls /dev/video*
```

ตรวจสอบรายละเอียดเพิ่ม:

```bash
dmesg | tail -n 50
```

สิ่งที่ต้องรู้ก่อนแก้ compose:

- LiDAR มักเป็น `/dev/ttyUSB0` หรือ `/dev/ttyUSB1`
- ESP32 อาจเป็น `/dev/ttyUSB0` หรือ `/dev/ttyACM0`
- กล้องมักเป็น `/dev/video0`

ถ้าชื่อ device ไม่ตรง ให้แก้ [rpi5/docker-compose.yml](/c:/DEV/pi5_ros_robot/rpi5/docker-compose.yml)

## ส่วนที่ 6: ตั้งค่าเครือข่ายและ ROS_DOMAIN_ID

ค่าเริ่มต้นของโปรเจ็กต์นี้คือ:

```bash
ROS_DOMAIN_ID=30
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

ทั้ง PC และ Pi ต้องใช้ค่า `ROS_DOMAIN_ID` เดียวกัน

ทดสอบ network ระหว่างสองฝั่ง:

บน PC:

```bash
ping <ip-of-pi>
```

บน Pi:

```bash
ping <ip-of-pc>
```

## ส่วนที่ 7: การ build และรันฝั่ง PC

### 7.1 อนุญาต X11 ให้ container เปิด GUI

บน PC:

```bash
xhost +local:root
```

ทดสอบว่ามีค่า `DISPLAY`:

```bash
echo $DISPLAY
```

ถ้าผลลัพธ์ว่าง ให้ login ใหม่ใน desktop session ก่อน หรือ export ค่าเองตามระบบของคุณ

### 7.2 Build image ฝั่ง PC

```bash
cd pc
docker compose build
```

### 7.3 รัน container ฝั่ง PC

```bash
docker compose up -d
```

### 7.4 ตรวจสอบสถานะ

```bash
docker compose ps
docker logs ros2_jazzy_pc
```

### 7.5 เข้าไปใน container ฝั่ง PC

```bash
docker exec -it ros2_jazzy_pc bash
source /opt/ros/jazzy/setup.bash
printenv | grep ROS
```

### 7.6 ทดสอบ GUI ใน container

เข้า container แล้วลอง:

```bash
rviz2
```

ถ้า RViz เปิดขึ้น แปลว่า X11 forwarding พร้อมใช้งาน

## ส่วนที่ 8: การ build และรันฝั่ง Raspberry Pi 5

### 8.1 Build image ฝั่ง sensors

```bash
cd rpi5
docker compose build
```

### 8.2 รันทุก services บน Pi

```bash
docker compose up -d
```

### 8.3 ตรวจสอบสถานะ

```bash
docker compose ps
docker logs rpi_sensors
docker logs rpi_microros
```

### 8.4 เข้า container ฝั่ง sensors

```bash
docker exec -it rpi_sensors bash
source /opt/ros/jazzy/setup.bash
printenv | grep ROS
ls /dev/ttyUSB*
ls /dev/video*
```

### 8.5 ยืนยันว่า micro-ROS Agent image ใช้งานได้

image `microros/micro-ros-agent:jazzy` รองรับทั้ง `amd64` และ `arm64` แล้ว จึงเหมาะกับ Pi 5 และ PC Linux

ถ้าต้องการทดสอบแยก:

```bash
docker manifest inspect microros/micro-ros-agent:jazzy
```

## ส่วนที่ 9: การรันโหนด ROS 2 ภายใน container

หมายเหตุสำคัญ:

- ตอนนี้ service `pc_simulation` และ `sensors_node` ถูกตั้งให้เปิด `bash`
- แปลว่า container จะพร้อมใช้งาน แต่ยังไม่ได้ launch sensor หรือ Nav2 อัตโนมัติ
- ข้อดีคือ debug ง่าย และเหมาะกับช่วงพัฒนา

### 9.1 รัน LiDAR node บน Pi

เข้า container:

```bash
docker exec -it rpi_sensors bash
source /opt/ros/jazzy/setup.bash
```

จากนั้นรัน package ของ LiDAR ตามรุ่นที่ใช้งานจริง โดยตรวจสอบ launch files หรือ executables ก่อน:

```bash
ros2 pkg executables sllidar_ros2
ros2 pkg prefix sllidar_ros2
```

ถ้าคุณมี launch file ของรุ่น LiDAR อยู่แล้ว จึงค่อยสั่ง launch ด้วย serial port ที่ตรงกับเครื่องจริง

### 9.2 รันกล้องบน Pi

ใน `rpi_sensors` container:

```bash
source /opt/ros/jazzy/setup.bash
ros2 run v4l2_camera v4l2_camera_node --ros-args -p video_device:=/dev/video0
```

ถ้ากล้องอยู่ device อื่น ให้เปลี่ยนเป็นค่าจริง

### 9.3 micro-ROS Agent บน Pi

service นี้จะรันอัตโนมัติจาก `docker-compose.yml`

ตรวจสอบ log:

```bash
docker logs -f rpi_microros
```

### 9.4 ตรวจ topics จากฝั่ง PC

เข้า container ฝั่ง PC:

```bash
docker exec -it ros2_jazzy_pc bash
source /opt/ros/jazzy/setup.bash
ros2 topic list
```

ถ้าทุกอย่างถูกต้อง ควรเริ่มเห็น topics เช่น:

- `/scan`
- `/camera/image_raw`
- `/odom`

### 9.5 ทดสอบ subscribe จากฝั่ง PC

```bash
ros2 topic echo /scan
ros2 topic echo /odom
```

## ส่วนที่ 10: คำสั่ง Docker ที่ต้องใช้บ่อย

### 10.1 เริ่มรัน

ฝั่ง PC:

```bash
cd pc
docker compose up -d
```

ฝั่ง Pi:

```bash
cd rpi5
docker compose up -d
```

### 10.2 หยุดแบบยังไม่ลบ container

```bash
docker compose stop
```

### 10.3 เริ่มใหม่หลัง stop

```bash
docker compose start
```

### 10.4 รีสตาร์ต service

```bash
docker compose restart
```

หรือระบุ service:

```bash
docker compose restart sensors_node
docker compose restart microros_agent
```

### 10.5 ปิดและลบ container/network ของ compose ชุดนั้น

```bash
docker compose down
```

### 10.6 ดู log แบบต่อเนื่อง

```bash
docker compose logs -f
```

หรือ:

```bash
docker logs -f ros2_jazzy_pc
docker logs -f rpi_sensors
docker logs -f rpi_microros
```

### 10.7 เข้า shell ของ container

```bash
docker exec -it ros2_jazzy_pc bash
docker exec -it rpi_sensors bash
```

### 10.8 build ใหม่เมื่อแก้ Dockerfile

```bash
docker compose build --no-cache
docker compose up -d
```

## ส่วนที่ 11: ลำดับการทดสอบแบบแนะนำ

### ขั้นที่ 1: ทดสอบ host network

- PC ping ถึง Pi
- Pi ping ถึง PC

### ขั้นที่ 2: ทดสอบ Docker

- `docker run hello-world` ผ่านทั้งสองเครื่อง
- `docker compose version` ใช้งานได้

### ขั้นที่ 3: ทดสอบ device บน Pi

- เห็น `/dev/ttyUSB*` หรือ `/dev/ttyACM*`
- เห็น `/dev/video*`

### ขั้นที่ 4: ทดสอบ compose

- `docker compose build` ผ่าน
- `docker compose up -d` ผ่าน
- `docker compose ps` แสดงสถานะ `Up`

### ขั้นที่ 5: ทดสอบ micro-ROS Agent

- `docker logs rpi_microros` ไม่มี error เรื่อง serial device not found

### ขั้นที่ 6: ทดสอบ ROS 2 discovery

- บน PC เห็น topic จาก Pi ด้วย `ros2 topic list`

### ขั้นที่ 7: ทดสอบ GUI

- `rviz2` เปิดจาก container บน PC ได้

## ส่วนที่ 12: ปัญหาที่พบบ่อย

### 12.1 เปิด RViz ไม่ขึ้น

ตรวจ:

```bash
echo $DISPLAY
xhost
```

แนวทางแก้:

- รัน `xhost +local:root`
- ใช้ Ubuntu Xorg ถ้า Wayland มีปัญหา
- เช็กว่า mount `/tmp/.X11-unix` ถูกต้อง

### 12.2 บน PC มองไม่เห็น topic จาก Pi

ตรวจ:

```bash
printenv | grep ROS
ping <peer-ip>
```

เช็กว่า:

- ทั้งสองฝั่งใช้ `ROS_DOMAIN_ID` เดียวกัน
- ทั้งสองฝั่งใช้ `rmw_cyclonedds_cpp`
- network อยู่ subnet เดียวกัน
- ไม่มี firewall block multicast

### 12.3 Pi หา serial device ไม่เจอ

ตรวจ:

```bash
ls /dev/ttyUSB*
ls /dev/ttyACM*
dmesg | tail -n 50
```

แล้วแก้ mapping ใน [rpi5/docker-compose.yml](/c:/DEV/pi5_ros_robot/rpi5/docker-compose.yml)

### 12.4 กล้องไม่ออกเป็น `/dev/video0`

ตรวจ:

```bash
ls /dev/video*
v4l2-ctl --list-devices
```

บางกล้องต้องลง driver เพิ่ม หรืออาจได้ชื่อเป็น `/dev/video1`

### 12.5 Container รันแล้วเด้งออก

ตรวจ log:

```bash
docker compose logs
docker logs ros2_jazzy_pc
docker logs rpi_sensors
docker logs rpi_microros
```

## ส่วนที่ 13: ขั้นตอนปิดระบบอย่างปลอดภัย

ถ้าจะหยุดชั่วคราว:

```bash
docker compose stop
```

ถ้าจะปิดงานของรอบนี้ทั้งหมด:

```bash
docker compose down
```

แนะนำให้ปิดโหนดที่ใช้งาน sensor อยู่ก่อน แล้วค่อย `down`

## ส่วนที่ 14: ขั้นตอนถัดไปที่ควรทำในโปรเจ็กต์นี้

เมื่อระบบพื้นฐานเชื่อมกันได้แล้ว แนะนำให้เพิ่มไฟล์เหล่านี้ต่อ:

- launch file สำหรับ LiDAR
- launch file สำหรับกล้อง
- launch file รวมของฝั่ง Pi
- launch file สำหรับ SLAM และ Nav2 ฝั่ง PC
- `.env` สำหรับกำหนด `ROS_DOMAIN_ID`
- script สำหรับ start/stop แบบคำสั่งเดียว

## อ้างอิงทางการ

- ROS 2 Jazzy documentation: https://docs.ros.org/en/jazzy/index.html
- Ubuntu download: https://ubuntu.com/download/desktop
- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Docker Engine on Debian: https://docs.docker.com/engine/install/debian/
- Docker post-install: https://docs.docker.com/engine/install/linux-postinstall/
- Raspberry Pi Imager / getting started: https://www.raspberrypi.com/software/
