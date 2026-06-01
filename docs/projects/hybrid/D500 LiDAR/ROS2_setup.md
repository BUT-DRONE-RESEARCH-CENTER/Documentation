
# Dokumentace: Zprovoznění LiDARu D500 (STL-19P) na Raspberry Pi 5 s ROS 2

Tato dokumentace popisuje kompletní postup konfigurace minipočítače **Raspberry Pi 5 (4GB RAM)** s operačním systémem **Raspbian (Debian Bookworm)** pro provoz 360° laserového skeneru **D500 LiDAR Kit (STL-19P)** od DFRobot. 

Vzhledem k tomu, že na pinech GPIO 14 a 15 je současně provozován autopilot Ardupilot a hostitelský systém nepodporuje přímou instalaci předkompilovaných balíčků ROS 2, je řešení realizováno izolovaně pomocí **Dockeru** s distribucí **ROS 2 Humble**.

---

## 1. Hardwarové zapojení

* **LiDAR D500:** Zapojen na piny **GPIO 0 (TXD0)** a **GPIO 1 (RXD0)**. V systému RPi 5 se toto rozhraní mapuje jako hardwarový sériový port **UART1** (`/dev/ttyAMA1`).
* **Ardupilot FC:** Zapojen na piny **GPIO 14 (TX)** a **GPIO 15 (RX)**. V systému se mapuje jako **UART0** (`/dev/ttyAMA0`).

---

## 2. Konfigurace hostitelského systému (Raspberry Pi OS)

### Krok 2.1: Aktivace sériových portů v config.txt
Na systému Debian Bookworm je nutné upravit konfigurační soubor v adresáři `/boot/firmware/`.

1. Otevřete konfigurační soubor:
```bash
sudo nano /boot/firmware/config.txt
```

2. Na konec souboru vložte následující řádky pro aktivaci obou portů zároveň:
```text
dtoverlay=uart1-pi5
```


3. Uložte změny (`Ctrl + O`, `Enter`) a zavřete editor (`Ctrl + X`).

### Krok 2.2: Restartování systému

Pro uplatnění změn v Device Tree restartujte Raspberry Pi:

```bash
sudo reboot
```

### Krok 2.3: Ověření dostupnosti portů

Po restartu ověřte, zda systém oba porty správně aktivoval:

```bash
ls -l /dev/ttyAMA*
```

Ve výpisu musíte vidět `/dev/ttyAMA0` (Ardupilot) i `/dev/ttyAMA1` (LiDAR).

---

## 3. Instalace a nastavení Dockeru

### Krok 3.1: Instalace Docker Engine

Nainstalujte Docker pomocí oficiálního automatického skriptu:

```bash
curl -fsSL [https://get.docker.com](https://get.docker.com) -o get-docker.sh && sudo sh get-docker.sh
```

### Krok 3.2: Nastavení uživatelských oprávnění

Přidejte aktuálního uživatele do skupiny `docker`, abyste nemuseli spouštět kontejnery pod `sudo` (což by později blokovalo předávání grafického rozhraní do X11):

```bash
sudo usermod -aG docker $USER && newgrp docker
```

---

## 4. Příprava a spuštění ROS 2 v Dockeru

### Krok 4.1: Povolení X11 grafiky pro Docker

Povolte lokálnímu kontejneru přístup k displeji hostitelského systému (nutné pro vykreslení okna RViz2):

```bash
xhost +local:root
```

### Krok 4.2: Spuštění ROS 2 kontejneru (ARM64)

Spusťte kontejner s oficiálním obrazem ROS 2 Humble pro architekturu ARM64. Příkaz předá přístup k displeji a nasdílí sériový port LiDARu (`/dev/ttyAMA1`):

```bash
docker run -it \
  --net=host \
  --ipc=host \
  --privileged \
  --env="DISPLAY=$DISPLAY" \
  --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
  --device=/dev/ttyAMA1 \
  --name ros2_lidar \
  ros:humble-ros-base
```

*Poznámka: Nyní se nacházíte uvnitř terminálu Docker kontejneru (`root@raspberrypi:/#`).*

---

## 5. Instalace ovladače a vizualizace (Uvnitř kontejneru)

### Krok 5.1: Instalace RViz2 a vývojových nástrojů

Aktualizujte repozitáře uvnitř kontejneru a nainstalujte grafický vizualizátor společně s kompilátorem `colcon` a nástrojem `git`:

```bash
apt update && apt install -y ros-humble-rviz2 git python3-colcon-common-extensions
```

### Krok 5.2: Stažení zdrojových kódů ovladače

Vytvořte ROS 2 pracovní prostor (workspace) a stáhněte oficiální repozitář pro sérii skenerů LDROBOT/STL (kam spadá i STL-19P):

```bash
mkdir -p /ros2_ws/src && cd /ros2_ws/src
git clone [https://github.com/ldrobotSensorTeam/ldlidar_stl_ros2.git](https://github.com/ldrobotSensorTeam/ldlidar_stl_ros2.git)
```

### Krok 5.3: Změna konfigurace na správný sériový port

Ovladač má v launch souborech standardně nastaven port `/dev/ttyUSB0`. Pomocí příkazu `sed` jej hromadně přepíšeme na náš port na GPIO pinech (`/dev/ttyAMA1`):

```bash
sed -i 's/\/dev\/ttyUSB0/\/dev\/ttyAMA1/g' /ros2_ws/src/ldlidar_stl_ros2/launch/*.py
```

### Krok 5.4: Kompilace projektu

Přejděte do kořene pracovního prostoru a zkompilujte balíček:

```bash
cd /ros2_ws && colcon build
```

---

## 6. Spuštění LiDARu a RViz2

Před spuštěním je nutné pokaždé načíst proměnné prostředí ROS 2 a zkompilovaného prostoru. Poté spusťte launch soubor určený pro zobrazení dat (viewer):

```bash
source /opt/ros/humble/setup.bash
source /ros2_ws/install/setup.bash
ros2 launch ldlidar_stl_ros2 viewer_ld19.launch.py
```

**Výsledek:** LiDAR se automaticky roztočí a na ploše operačního systému se otevře grafické okno **RViz2**, které začne v reálném čase vykreslovat body (`LaserScan`) z LiDARu.

---

## Tipy pro opakované spuštění po restartu RPi

Pokud minipočítač restartujete, vytvořený Docker kontejner zůstane uložený v paměti. Nemusíte vše instalovat znovu. Stačí v terminálu hostitelského systému spustit:

```bash
xhost +local:root
docker start -ai ros2_lidar
```

Tím se opět ocitnete uvnitř kontejneru a pak už jen stačí spustit příkazy z **Kroku 6**.
