Vzhledem k tomu, že tvůj kontejner existuje, ale aktuálně neběží, připravil jsem ti kompletní postup krok za krokem do textových polí (bloků kódu) pro snadné kopírování. Nezapomeň nahradit `<nazev_kontejneru>` a `<nazev_souboru.bag>` svými skutečnými názvy.

### 1. Spuštění kontejneru a základní kontrola

Nejprve musíš existující kontejner nastartovat, vstoupit do něj a vypsat si základní informace o bagu.

```bash
# 1. Spuštění zastaveného kontejneru
docker start <nazev_kontejneru>

# 2. Vstup do kontejneru (Terminál 1)
docker exec -it <nazev_kontejneru> bash

# 3. Přechod do složky a výpis informací
cd /data
rosbag info <nazev_souboru.bag>

```

### 2. Spuštění přehrávání bagu

Ve stejném terminálu (Terminál 1), kde jsi provedl předchozí kroky, nyní spusť přehrávání. Tento terminál necháš běžet.

```bash
rosbag play <nazev_souboru.bag>

```

### 3. Hloubková kontrola dat (v novém terminálu)

Zatímco se bag přehrává v prvním terminálu, otevři si na svém PC **nové okno terminálu**, připoj se do stejného kontejneru a proveď testy.

```bash
# 1. Vstup do běžícího kontejneru z nového okna (Terminál 2)
docker exec -it <nazev_kontejneru> bash

# --- TEST 1: Ztracené snímky (Frame drops) ---
# Sleduj, zda jdou čísla plynule po sobě (1, 2, 3...) bez skoků
rostopic echo /usb_cam/image_raw/header/seq | head -n 100

# --- TEST 2: Časový rozptyl kamery (Jitter) ---
# Sleduj frekvenci (ideálně ~30 Hz) a 'std dev' (ideálně pod 0.005)
# Pro ukončení testu stiskni Ctrl+C
rostopic hz /usb_cam/image_raw

# --- TEST 3: Kontrola frekvence IMU ---
# Sleduj průměrnou frekvenci (minimum je 100 Hz, ideál 200-400 Hz)
# Pro ukončení testu stiskni Ctrl+C
rostopic hz /mavros/imu/data

```