# Setup the Internet connection through ethernet for Jetson (Windows → Jetson Orin NX 16GB + Waveshare Jetson IO base)

## Prerequisites
1. Windows PC
2. Jetson Orin NX (Jetpack 6.1.)
3. Jetson carrier that supports Ethernet

## Procedure 

1. Connect the PC and Jetson with Ethernet cable
2. In your PC go to **Control Panels** and select **Network and Internet → Network and Sharing Center**
3. At the left panel select **Change adapter settings** (new pop-up window will appear)
4. Select your source of internet (Wifi, Ethernet etc.), right click and select **Properties**
5. At the new pop-up window, switch to **Sharing** at a top
6. In the **Internet Connection Sharing** section turn on the **Allow other network users to connect through this computer's internet connection**
7. In column **Home networking connection** select the connection with the Jetson (probably Ethernet or Ethernet 1 or something like that)

After a minute or so, the Jetson will be able to connect to the internet without any other packages.


# Setup the wifi (iwlwifi package) for Jetpack 6.1. (Jetson Orin NX 16GB + Waveshare Jetson IO base)

>Fully documented and solved issue on Nvidia forum:
https://forums.developer.nvidia.com/t/jetpack-6-wifi-slow-startup-with-backport-iwlwifi-dkms/297967

### 1. Go to website with backports: https://backports.docs.kernel.org/releases.html

### 2. Download required backport
For Jetpack 6.1. it is this one:

```bash
backports-5.15.153-1.tar.xz
SHA256: eaa24df968c79385c57707068a209fb1ea43271b573f24885805ae96a58ee3a8
```

- if you dont know the correct backport, it corresponds with your tegra -> you can found out in terminal:

    ```bash
    uname -r
    ```

### 3. After download go to *Download* folder in terminal from home
```bash
cd /Downloads
```

### 4. Tar and go to the file:
```bash
tar Jxfv backports-5.15.153-1.tar.xz
cd backports-5.15.153-1
```

### 5. make the config:
```bash
make defconfig-iwlwifi
make -j8
```

### 6. Install it and reboot
```bash
sudo make install
sudo reboot
```

