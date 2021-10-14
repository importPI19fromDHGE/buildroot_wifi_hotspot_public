# WLAN-Hotspot mit Raspberry Pi 4 und Buildroot

## Komponenten

- Access Point: ``wpa_supplicant``
- DHCP-Server: ``dnsmasq``

## Installation und Konfiguration in Buildroot

Die [oben genannten Komponenten](#komponenten) Komponenten sind der einfache Part, darüber hinaus gibt es allerdings einiges zu konfigurieren. Für einen funktionierende Hotspot sollte unter ``Target Packages`` folgendes angewählt werden:

- Busybox-Pakete anzeigen: ``Target packages -> BusyBox -> Show packages that are also provided by busybox``
- ``Hardware handling/Firmware/rpi-firmware``
- ``Hardware handling/Firmware/rpi-wifi-firmware``
- ``Networking applications/dnsmasq inkl. dhcp support``
- ``Networking applications/wireless-regdb``
- ``Networking applications/wpa_supplicant`` inkl. ``nl80211`` Treiber und ``AP mode``

Danach muss die Buildroot-Umgebung angepasst werden, damit auch alles starten kann:

- WiFi-Treiber beim Boot laden: ``System configuration -> /dev management -> Dynamic using devtmpfs + mdev``

Abschließend müssen noch ein paar Konfigurationsdateien auf dem Pi abgelegt / angepasst werden. Es wird empfohlen, dies mit einem Overlay-Ordner zu tun.

Netzwerkschnittstellenkonfiguration **``/etc/network/interfaces``**:

```conf
auto eth0 # automatische Verwaltung der Ethernet-Schnittstelle
iface eth0 inet dhcp # Beziehe IP-Adresse von eth0 von einem DHCP-Server eines anderen Routers
    #pre-up /etc/network/nfs_check
    wait-delay 15
auto wlan0 # automatische Verwaltung der WLAN-Schnittstelle
iface wlan0 inet static # feste IP-Adresse für wlan0
    address 10.4.0.1
    netmask 255.255.255.0
    network 10.4.0.0 # Zielnetz für Hotspot
    gateway 10.4.0.1 # Gateway: wir haben keins, also geben wir "uns" an
    pre-up wpa_supplicant -B -Dnl80211 -iwlan0 -c/etc/wpa_supplicant.conf # starte den Hotspot vor Bereitmeldung des Interface
    post-down killall -q wpa_supplicant # beende Hotspot nach Abmeldung des Interface
    wait-delay 15
iface default inet dhcp
```

Konfiguration von DHCP-Server (und auch DNS-Server, falls später gewünscht) in **``/etc/dnsmasq.conf``**:

```conf
interface=wlan0 # Ziel-Interface
no-dhcp-interface=eth0 # kein DHCP-Server für eth0
dhcp-range=10.4.0.2,10.4.0.200,255.255.255.0,1h # IP-Bereich, den der DHCP-Server vergibt
domain=wlan     # WLAN-DNS-Domäne
address=/gw.wlan/10.4.0.1 # Gateway-Adresse für DNS-Domäne
```

- ``package/busybox/mdev.conf`` und ``package/busybox/S10mdev`` in das Overlay nach ``/etc`` respektive ``/etc/init.d`` kopieren
- ``S10mdev`` ausführbar machen

Jetzt *sollte* der Hotspot funktionieren :-)
