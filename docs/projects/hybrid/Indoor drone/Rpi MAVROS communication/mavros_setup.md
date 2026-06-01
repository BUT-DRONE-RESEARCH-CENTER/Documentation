# Připojení a konfigurace ArduPilotu (NxtPX4v2) v ROS 2 Humble (Docker)

Tato dokumentace popisuje postup pro propojení autonomní jednotky **NxtPX4v2** s firmwarem **ArduCopter (V4.8.0-dev)** a minipočítače **Raspberry Pi 5** běžícího na **Raspberry Pi OS (Debian Bookworm)**. Komunikace je zajištěna pomocí rozhraní **MAVROS** v izolovaném **Docker** kontejneru.

---

## 1. Hardwarové zapojení a parametry

* **Port na dronu:** TELEM 2 (UART 4)
* **Piny na Raspberry Pi 5:** GPIO 14 (TX) a GPIO 15 (RX)
* **Systémový port (Host OS):** `/dev/ttyAMA0` (UART0)
* **Výchozí přenosová rychlost (Baudrate):** `460800`

---

## 2. Konfigurace hostitelského systému (Raspberry Pi OS)

Před předáním portu do Dockeru je nutné nakonfigurovat hardwarový UART0 port.

### Krok 2.1: Aktivace portu v config.txt
1. Otevřete konfigurační soubor:
```bash
sudo nano /boot/firmware/config.txt
```

2. Na konec souboru přidejte následující řádky:
```text
dtoverlay=uart1-pi5
dtparam=uart0=on
```

3. Uložte změny (`Ctrl + O`, `Enter`) a zavřete editor (`Ctrl + X`).

### Krok 2.2: Restart systému

Pro uplatnění změn restartujte Raspberry Pi:

```bash
sudo reboot
```

---

## 3. Spuštění Docker kontejneru

Při spouštění ROS 2 kontejneru je nutné explicitně namapovat zařízení `/dev/ttyAMA0`:

```bash
sudo docker run -it \
  --net=host \
  --ipc=host \
  --privileged \
  --device=/dev/ttyAMA0 \
  --name ros2_ardupilot \
  ros:humble-ros-base
```

---

## 4. Instalace a nastavení MAVROS (Uvnitř kontejneru)

Po vstupu do kontejneru nainstalujte balíček MAVROS a stáhněte povinné geografické datové sady (GeographicLib), bez kterých se uzel nespustí.

```bash
# Aktualizace a instalace MAVROS
apt update && apt install -y ros-humble-mavros ros-humble-mavros-extras wget

# Stažení GeographicLib databází
sudo /opt/ros/humble/lib/mavros/install_geographiclib_datasets.sh
```

---

## 5. Spuštění komunikace a aktivace datových proudů

### Krok 5.1: Spuštění MAVROS uzlu (Terminál 1)

Spusťte komunikační most mezi MAVLink protokolem a ROS 2 tématy:

```bash
source /opt/ros/humble/setup.bash
ros2 run mavros mavros_node --ros-args -p fcu_url:=/dev/ttyAMA0:460800
```

*(Úspěšné spojení potvrdí řádek: `CON: Got HEARTBEAT, connected. FCU: ArduPilot`). Tento terminál nechte běžet.*

### Krok 5.2: Aktivace streamování IMU dat (Terminál 2)

ArduPilot ve výchozím stavu na telemetrických portech šetří šířku pásma a data ze senzorů neposílá automaticky. Otevřete nové okno terminálu, připojte se do kontejneru (`sudo docker exec -it ros2_ardupilot bash`) a zavolejte ROS 2 službu pro vynucení streamu:

```bash
source /opt/ros/humble/setup.bash
ros2 service call /mavros/set_stream_rate mavros_msgs/srv/StreamRate "{stream_id: 0, message_rate: 50, on_off: true}"
```

---

## 6. Ověření publikovaných témat (Topics)

Nyní jsou data z autopilota plně dostupná v ROS 2 ekosystému. Můžete je vyčítat pomocí následujících příkazů:

* **Surová IMU data (Akcelerometru + Gyroskop):**
```bash
ros2 topic echo /mavros/imu/data_raw
```


* **Filtrovaná data (Orientace dronu v prostoru):**
```bash
ros2 topic echo /mavros/imu/data
```


* **Letový stav a režim (Armed/Disarmed, Mode):**
```bash
ros2 topic echo /mavros/state
```