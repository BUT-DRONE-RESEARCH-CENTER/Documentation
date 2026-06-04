# Kompletní průvodce instalací a spuštěním OpenVINS v Dockeru
**(Pro Ubuntu 22.04 s podporou NVIDIA GPU)**

Tento návod popisuje kompletní postup od čisté instalace operačního systému, přes přípravu nástrojů, až po spuštění vizuálně-inerciální odometrie (OpenVINS) na reálném datasetu (EuRoC MAV) pomocí kontejnerizace.

## Fáze 0: Příprava systému (Předpoklady)

Tyto kroky se provádějí přímo na hostitelském operačním systému (Ubuntu 22.04 LTS).

**1. Instalace ROS 2 Humble (Volitelné pro hostitele, doporučené pro vývoj):**
Pokud chcete pracovat s ROS 2 nástroji i mimo Docker, nainstalujte ROS 2 Humble podle oficiální dokumentace:
[Ubuntu Install Debs - ROS 2 Humble](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html)

**2. Instalace Dockeru:**
Postup se liší podle distribuce, tento návod vychází z Linuxu. Na Ubuntu lze Docker nainstalovat následovně:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

**3. Instalace NVIDIA Container Toolkitu (Kritické):**
Abyste mohli v Dockeru využívat hardwarovou akceleraci (GPU) a průchod grafiky, nainstalujte NVIDIA Container Toolkit podle oficiální dokumentace:
[NVIDIA Container Toolkit Installation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
*(Poznámka: Bez tohoto kroku bude kontejner při pokusu o spuštění grafiky nebo výpočtů padat. Aktuální postup instalace je vhodné ověřit přímo v dokumentaci, protože starší varianta s `apt-key` je deprecated.)*

**4. Povolení vykreslování grafiky (X11 Forwarding):**
Aby Docker mohl otevírat grafická okna (jako je RViz) na vaší obrazovce, musíte povolit přístup k X serveru:
```bash
xhost +
```
*(Tento příkaz je nutné spustit na hostiteli po každém restartu počítače před prvním spuštěním kontejneru).*

**5. Rychlé ověření Dockeru a GUI (Volitelné):**
Pokud si chcete ověřit, že Docker i předání GUI fungují správně, můžete zkusit jednoduchý testovací kontejner:
```bash
docker run -it --net=host --gpus all \
    --env="NVIDIA_DRIVER_CAPABILITIES=all" \
    --env="DISPLAY" \
    --env="QT_X11_NO_MITSHM=1" \
    --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
    osrf/ros:noetic-desktop-full \
    bash -it -c "roslaunch gazebo_ros empty_world.launch"
```
Pokud chcete jen skočit do shellu v kontejneru a zkusit například `rviz`, použijte:
```bash
docker run -it --net=host --gpus all \
    --env="NVIDIA_DRIVER_CAPABILITIES=all" \
    --env="DISPLAY" \
    --env="QT_X11_NO_MITSHM=1" \
    --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
    osrf/ros:noetic-desktop-full \
    bash
```
V otevřeném shellu pak lze spustit `rviz`.

---

## Fáze 1: Tvorba složek a práva (Directory Binding)

Abychom mohli sdílet kód a data mezi vaším počítačem a Dockerem, vytvoříme fyzické složky a nastavíme práva.

**1. Vytvoření pracovních složek:**
Nejprve je nutné vytvořit fyzické složky na vašem počítači (Host), které budou sdíleny (bind mount) do Docker kontejneru.
Otevřete terminál a spusťte:
```bash
mkdir -p ~/workspace/catkin_ws_ov/src
mkdir -p ~/datasets
```


**2. Oprávnění pro Docker (Důležité):**
Abyste nemuseli spouštět Docker pod právy `sudo` (což by rozbilo fungování aliasů), přidejte svého uživatele do skupiny `docker`:
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
```
*Poznámka: Pro aplikování těchto změn je **nutné se odhlásit a znovu přihlásit** do Ubuntu (případně restartovat PC).*

**3. Povolení vykreslování grafiky (X11 Forwarding):**
Aby Docker mohl otevírat grafická okna (jako je RViz) na vaší obrazovce, musíte povolit přístup k X serveru:
```bash
xhost +
```
*(Tento příkaz je nutné spustit na hostiteli po každém restartu počítače před prvním spuštěním kontejneru).*

---

## Fáze 2: Nastavení Directory Binding (Alias)

Vytvoříme alias, který jedním příkazem spustí Docker, propíše do něj grafickou kartu a propojí vytvořené složky.

**1. Úprava souboru `.bashrc`:**
```bash
nano ~/.bashrc
```

**2. Vložení aliasu:**
Na úplný konec souboru vložte následující řádky:
```bash
# OpenVINS Docker nastavení
export DOCKER_CATKINWS=$HOME/workspace/catkin_ws_ov
export DOCKER_DATASETS=$HOME/datasets

alias ov_docker="docker run -it --net=host --gpus all \
    --env=\"NVIDIA_DRIVER_CAPABILITIES=all\" --env=\"DISPLAY\" \
    --env=\"QT_X11_NO_MITSHM=1\" --volume=\"/tmp/.X11-unix:/tmp/.X11-unix:rw\" \
    --mount type=bind,source=\$DOCKER_CATKINWS,target=/catkin_ws \
    --mount type=bind,source=\$DOCKER_DATASETS,target=/datasets"
```
Uložte (`Ctrl+O`, `Enter`) a zavřete (`Ctrl+X`).

**3. Načtení změn:**
```bash
source ~/.bashrc
```

---

## Fáze 3: Stažení kódu a dat

**1. Klonování repozitáře OpenVINS:**
Zdrojové kódy musíte stáhnout do složky, kterou jsme pro ně připravili:
```bash
cd ~/workspace/catkin_ws_ov/src
git clone [https://github.com/rpng/open_vins.git](https://github.com/rpng/open_vins.git)
```

**2. Stažení testovacího datasetu (EuRoC MAV):**
Stáhněte si `.bag` soubor přes prohlížeč (např. z ETHZ Research Collection) a umístěte jej do složky `~/datasets`.
Pokud je stažený soubor v archivu `.zip`, nezapomeňte jej rozbalit:
```bash
cd ~/datasets
unzip NAZEV_STAZENEHO_ARCHIVU.zip
```
Výsledkem musí být soubor s koncovkou `.bag` přímo ve složce `~/datasets`.

---

## Fáze 4: Sestavení Docker obrazu (Build)

Nyní vytvoříme Docker obraz obsahující veškeré závislosti (ROS Noetic, OpenCV, Eigen atd.).

**1. Přesun do složky s Dockerfiles:**
```bash
cd ~/workspace/catkin_ws_ov/src/open_vins/docker
```

**2. Spuštění buildu:**
```bash
docker build -t ov_ros1_20_04 -f Dockerfile_ros1_20_04 .
```
*(Pozor na tečku na konci příkazu. Tento proces může trvat 10-20 minut).*

Pokud se build rozbije a chcete obraz odstranit a zkusit to znovu, použijte:
```bash
docker image list
docker image rm ov_ros1_20_04 --force
```

---

## Fáze 5: Kompilace OpenVINS (Uvnitř Dockeru)

Nyní vstoupíme do Dockeru a zkompilujeme samotný kód OpenVINS.

**1. Vstup do kontejneru pomocí aliasu:**
```bash
ov_docker ov_ros1_20_04 bash
```

**2. Kompilace balíčků:**
Uvnitř Dockeru se přepněte do workspace a spusťte build:
```bash
cd /catkin_ws
catkin build
source devel/setup.bash
```
Na konci byste měli vidět hlášku `Summary: All 6 packages succeeded!`.

---

## Fáze 6: Spuštění OpenVINS na reálných datech

Pro spuštění celého systému budete potřebovat **4 různé terminály**. V každém z nich se nejprve musíte přepnout do Dockeru.

Otevřete si 4 nová okna terminálu a v **každém z nich** nejprve zadejte:
```bash
ov_docker ov_ros1_20_04 bash
source /catkin_ws/devel/setup.bash
```

Následně rozdělte práci mezi terminály takto:

**Terminál 1 (ROS Master):**
```bash
roscore
```

**Terminál 2 (Vizualizace RViz):**
```bash
rosrun rviz rviz -d /catkin_ws/src/open_vins/ov_msckf/launch/display.rviz
```

**Terminál 3 (Spuštění algoritmu):**
Předpokládáme využití EuRoC datasetu, proto načítáme příslušnou konfiguraci:
```bash
roslaunch ov_msckf subscribe.launch config:=euroc_mav
```

**Terminál 4 (Přehrávání datasetu):**
*(Zkontrolujte název souboru přes `ls /datasets` a nahraďte jím zástupný text)*
```bash
rosbag play /datasets/NAZEV_VASEHO_SOUBORU.bag
```

**Výsledek:** Jakmile spustíte poslední příkaz, data začnou proudit do algoritmu a v okně RViz uvidíte v reálném čase odhadovanou trajektorii pohybu, detekované vizuální prvky (features) a odhadovanou mapu okolí.

---

### Řešení nejčastějších problémů

*   **Chyba "permission denied... docker.sock":** Zapomněli jste přidat uživatele do skupiny `docker` (Fáze 1, Krok 2) nebo jste se po tomto kroku neodhlásili a nepřihlásili.
*   **Chyba "exit code -6" u uzlu `ov_msckf` (Pád při startu):** Docker nemá oprávnění otevřít grafické okno na vašem hostiteli. Spusťte na hostiteli (nikoliv v Dockeru) příkaz `xhost +`.
*   **Chyba "Error opening file: /datasets/vase_data.bag":** Kontrolujte přesný název a umístění staženého `.bag` souboru. Soubor musí ležet fyzicky ve složce `~/datasets` na hostitelském systému.







******************
real data 

rosbag play --clock /datasets/dataset.bag

roslaunch ov_msckf my_drone_subscribe.launch
