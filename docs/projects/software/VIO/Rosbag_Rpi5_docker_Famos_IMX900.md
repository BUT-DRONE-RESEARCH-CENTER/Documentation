# Kompletní průvodce: Framos IMX900 + ArduPilot (MAVROS) v Dockeru pro OpenVINS

Tento dokument popisuje kompletní postup zprovoznění průmyslové MIPI kamery Framos (IMX900) a letové jednotky ArduPilot na Raspberry Pi 5. Cílem je záznam synchronizovaných dat (obraz + IMU) do souboru `rosbag` pro vizuálně-inerciální odometrii (OpenVINS) uvnitř ROS 1 Noetic Docker kontejneru.

## Fáze 1: Hostitelský systém (Raspberry Pi OS)

Systém musí běžet na čisté instalaci **Raspberry Pi OS (64-bit)** (verze Bookworm). Všechny příkazy v této fázi se zadávají přímo na hostitelském systému, nikoliv v Dockeru.

### 1. Instalace závislostí pro kompilaci ovladačů

```bash
sudo apt update
sudo apt install -y git build-essential dkms linux-headers-$(uname -r)
```

### 2. Stažení a kompilace ovladačů Framos

1. Naklonujte zdrojové kódy Framos Raspberry Pi driverů do domovského adresáře na Raspberry Pi:

```bash
cd ~
git clone https://github.com/framosimaging/framos-rpi-drivers
cd ~/framos-rpi-drivers
```

2. Přepněte se na správnou větev `framos-rpi-drivers` podle cílové verze Raspberry Pi OS:

```bash
git checkout framos_20250916
```

3. Zkompilujte kernel moduly a device tree overlaye:

```bash
cd ~/framos-rpi-drivers/build
source target_build.sh
build_all
```

4. Nainstalujte kernel moduly a device tree overlaye:

```bash
install_all
```

### 3. Konfigurace `/boot/firmware/config.txt`

Otevřete konfiguraci:

```bash
sudo nano /boot/firmware/config.txt
```

Proveďte tyto změny:

1. Odkomentujte nebo přidejte `dtparam=i2c_arm=on`.
2. Nastavte `camera_auto_detect=0`.
3. Na konec souboru přidejte overlay podle zapojeného portu, zde pro CAM0:

```text
dtoverlay=fr_imx900,cam0
```

Uložte změny pomocí `Ctrl+O`, potvrďte `Enter` a zavřete `Ctrl+X`.

### 4. Bezpečný restart a připojení kamery

> ⚠️ **Kritické varování:** Raspberry Pi 5 posílá 3.3 V do MIPI portů i při softwarovém vypnutí. Připojení kamery pod proudem ji může trvale zničit.

1. Vypněte RPi: `sudo poweroff`
2. Fyzicky odpojte USB-C napájecí kabel.
3. Připojte MIPI kabel kamery.
4. Znovu zapojte napájení a zařízení spusťte.

Po startu ověřte detekci senzoru:

```bash
sudo dmesg | grep -i "imx900"
# Očekávaný výstup: fr_imx900 4-001a: Detected imx900 sensor
```

### 5. Stažení a instalace Framos libcamera

1. Naklonujte repozitář Framos do domovského adresáře na Raspberry Pi:

```bash
cd ~
git clone https://github.com/framosimaging/framos-libcamera.git
```

2. Přepněte se na správnou větev `framos-libcamera` podle cílové verze Raspberry Pi OS.


- Příklad: pro Raspberry Pi OS (64-bit) 2025-11-24 použijte větev `framos_v0.5.2+rpt20250903`.

```bash
cd ~/framos-libcamera
git checkout framos_v0.5.2+rpt20250903
```

- Správnou větev najdete v dokumentaci [Supported Raspberry Pi OS versions and Framos libcamera branch compatibility](https://github.com/framosimaging/framos-libcamera#supported-raspberry-pi-os-versions-and-framos-libcamera-branch-compatibility).

3. Spusťte automatický instalační skript. Ten nainstaluje `libcamera`, `rpicam-apps` a další potřebné knihovny:

```bash
sudo ./install_libcamera.sh
```

4. Ověřte náhled videa z připojené kamery:

```bash
rpicam-hello --timeout 0
```

---

## Fáze 2: Nastavení mostu pro kameru (V4L2 Loopback)

Protože Framos používá upravený `libcamera` stack, který není kompatibilní s výchozím GStreamerem v ROS 1, vytvoříme virtuální kameru a obraz do ní přesměrujeme pomocí nativních nástrojů.

### 1. Instalace nástrojů

```bash
sudo apt update
sudo apt install -y v4l2loopback-dkms ffmpeg
```

### 2a. Vytvoření virtuální kamery

Před každým nahráváním, případně automaticky po startu, spusťte na hostitelském RPi tyto dva příkazy:

```bash
# Vytvoření virtuálního zařízení /dev/video10
sudo modprobe v4l2loopback video_nr=10
```

### 2b. spuštění streamu

**Při každém restartu systému je nutné obnovit běh streamu.**

```bash
# Přesměrování obrazu do virtuální kamery; příkaz nechte běžet na pozadí
rpicam-vid -t 0 --width 1280 --height 720 --framerate 30 --codec yuv420 -o - | ffmpeg -loglevel warning -f rawvideo -pix_fmt yuv420p -s 1280x720 -r 30 -i - -f v4l2 -pix_fmt yuyv422 /dev/video10
```

---

## Fáze 3: Nastavení ArduPilotu (přes Mission Planner)

Aby ArduPilot posílal IMU data dostatečně rychle pro VIO, nastavte v parametrech letové jednotky pro sériový port, přes který je Pi připojeno, například `SR0` pro USB:

- `SR0_RAW_SENS` = 50 nebo vyšší, minimum 50 Hz
- `SR0_EXTRA1` = 50
- `SERIAL0_PROTOCOL` = 2, tedy MAVLink 2

---

## Fáze 4: Instalace a nastavení ROS Dockeru

### 1. Instalace Dockeru na hostitelský systém

```bash
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Vytvoření sdílené složky a spuštění kontejneru

```bash
mkdir -p ~/ros_data

docker run -it --name ros_vins_record \
--privileged \
--net=host \
-v /dev:/dev \
-v ~/ros_data:/data \
ros:noetic-ros-base \
bash
```

### 3. Instalace ROS balíčků uvnitř kontejneru

Příkazy spouštějte v terminálu Dockeru, například `root@raspberrypi:/#`:

```bash
apt update
apt install -y wget curl ros-noetic-usb-cam ros-noetic-mavros ros-noetic-mavros-extras

# Instalace GeographicLib pro MAVROS
wget https://raw.githubusercontent.com/mavlink/mavros/master/mavros/scripts/install_geographiclib_datasets.sh
chmod +x install_geographiclib_datasets.sh
./install_geographiclib_datasets.sh

# Nastavení prostředí
echo "source /opt/ros/noetic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## Fáze 5: Spouštění uzlů a nahrávání dat (rosbag)

Pro nahrávání potřebujete otevřít více terminálů uvnitř běžícího kontejneru. Připojíte se z hostitelského systému pomocí `docker exec -it ros_vins_record bash`.

### Terminál 1: MAVROS (připojení ArduPilotu)

```bash
# Změňte ttyACM0 na správný port, například ttyUSB0
roslaunch mavros apm.launch fcu_url:=/dev/ttyACM0:115200

# Pokud data IMU netečou automaticky, vynuťte je tímto příkazem v jiném okně:
rosservice call /mavros/set_stream_rate 0 50 1
```

### Terminál 2: Kamera (usb_cam)

Ignorujte případná žlutá varování `[WARN]` o autofokusu a white balance, jde o virtuální kameru.

```bash
rosrun usb_cam usb_cam_node _video_device:=/dev/video10 _pixel_format:=yuyv _image_width:=1280 _image_height:=720
```

### Terminál 3 (mimo docker): Ověření funkčnosti kamery

Ověříme správnou funkčnost kamery pomocí nástroje ROS `image_saver`.

1. Nainstaluj balíček pro práci s obrázky (v Terminálu 3):

```bash
apt update
apt install -y ros-noetic-image-view
```

2. Přesuň se do sdílené složky a spusť ukládání:

```bash
cd /data
rosrun image_view image_saver image:=/usb_cam/image_raw
```

Uzel se spustí a začne okamžitě sekat fotky (uvidíš hlášky, že ukládá `left0000.jpg`, `left0001.jpg` atd.). **Po zhruba 2 vteřinách ho zastav pomocí `Ctrl+C`.

3. Kontrola na hostitelském systému:
Přepni se z Dockeru zpět do normálního terminálu Raspberry Pi (přímo na tvém SSD) a zadej:

```bash
ls -lh ~/ros_data
```

Pokud tam uvidíš několik `.jpg` souborů, most funguje dokonale! Můžeš si ten obrázek přesunout do počítače a prohlédnout, jestli je obraz ostrý a má správné barvy.

### Terminál 4: Ověření toku IMU

Pro potvrzení, že IMU data (akcelerometr a gyroskop) skutečně proudí do ROSu, použijte příkaz:

```bash
rostopic echo /mavros/imu/data_raw
```

Pokud je vše v pořádku, terminál začne okamžitě vypisovat číselné hodnoty. Sledujte zejména sekci `linear_acceleration`, kde by u osy `z` měla být hodnota přibližně **9.8** (zemská gravitace), pokud dron leží v klidu.

### Terminál 4: Nahrávání rosbagu pro OpenVINS

Před nahráváním se vždy přesvědčte, že jste ve sdílené složce `/data`, jinak se soubor ztratí uvnitř kontejneru.

```bash
cd /data
rosbag record -O openvins_dataset.bag /usb_cam/image_raw /mavros/imu/data_raw
```

**Postup kalibračního pohybu:**

1. Po zahájení nahrávání vezměte drona do ruky.
2. Plynule s ním pohybujte po místnosti po dobu 1 až 2 minut.
3. Excitujte IMU ve všech osách, tedy dopředu a dozadu, nahoru a dolů, do stran, a přidávejte rotace.
4. Kamerou mířte na texturované povrchy, ne na čisté zdi.
5. Ukončete nahrávání pomocí `Ctrl+C`.

Výsledný dataset `openvins_dataset.bag` bude bezpečně uložen na hostitelském disku ve složce `~/ros_data`.