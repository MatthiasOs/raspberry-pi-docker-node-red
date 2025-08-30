# Übersicht der Anwendungen: ![Folie1.JPG](Folie1.JPG) 


# Wie man ein RaspberryPi mit docker und node-red einrichtet

SD Karte mit `Raspberry Imager` flashen (SSH aktivieren), hier direkt ein WLAN angeben, welches man bei der weiteren Einrichtung verwenden will.
(Später kann man dann noch das WLAN einrichten, welches dann Vorort existiert).
Nach der Installation per SSH auf das PI verbinden

## Docker installieren: 
```
sudo curl -sSL https://get.docker.com | sh
sudo apt install -y docker-compose
```

In der Console prüfen ob es funktioniert hat:
```
docker
```

User in docker Gruppe hinzufügen, dass man Container als aktuller User starten:
```
sudo usermod -aG docker $USER
```
Anschließend Gruppenzugehörigkeit neu laden:
```
newgrp docker
```

## node-red als [docker Container starten](https://nodered.org/docs/getting-started/docker)
Ordner für Node-RED anlegen
```
sudo mkdir -p ~/nodered
cd ~/nodered
```
docker-compose.yaml anlegen `sudo nano docker-compose.yaml` (Nicht benötigte Port Mappings löschen):
```
services:
  nodered:
    image: nodered/node-red:latest-minimal
    container_name: nodered
    restart: unless-stopped
    ports:
      - "1881:1880" # Für Node-Red GUI
      - "1883:1883" # Für MQTT
      - "502:502" # Für Modbus
    volumes:
      - ./data:/data
    environment:
      - TZ=Europe/Berlin
```
Zum Speichern: STRG+X -> Y -> Enter

Optional: `docker-compose.yaml` validieren
```
docker-compose config
```
Anschließend Container starten:
```
docker-compose up -d
```
Kontrolle ob der Container läuft:
```
docker ps
docker volume ls
```
Im Browser die IP mit Port 1880 an surfen (zB http://192.168.178.40:1800) -> node-red ist erreichbar

Zum Beenden des Containers:
```
docker-compose down
```

## docker Einstellungen
Damit beim Start (boot) des Pi auch docker automatisch gestartet wird:
```
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```
## WLAN
Damit das RaspberryPi sich direkt mit dem WLAN am Ort wo man es verwenden will verbindet, muss man vorher die Verbindung [mit dem NetworkManager einrichten](https://raspberrytips.com/raspberry-pi-wifi-setup/#set-up-your-wifi-on-raspberry-pi-os-lite) einrichten:
```
sudo nmtui
```
(Manuell müsste es auch gehen, dann muss man eine "\<Wifi-Name\>.nmconnection" Datei anlegen, siehe https://forums.raspberrypi.com/viewtopic.php?t=360175)

# Dependecies in node-red Palette installieren
- [node-red-contrib-shelly](https://flows.nodered.org/node/node-red-contrib-shelly) zum Auslesen der PV Erzeugung von einem Shelly (nur benötigt für Variante 1)
- [node-red-contrib-modbus](https://flows.nodered.org/node/node-red-contrib-modbus) zum Schreiben der Daten mittels Modbus in den Fronius Wechselrichter
- [node-red-contrib-buffer-parser](https://flows.nodered.org/node/node-red-contrib-buffer-parser) zum einfachen Konvertieren von Registern zu lesbaren Werten und zurück (optional)

# Externer Zähler im Fronius Wechselrichter
Damit im Fronius SolarWeb neben dem Fronius Wechselrichter auch ein externer Erzeuger (zB Balkonkraftwerk) oder Verbraucher (zB Ladepunkt) berücksichtigt wird, muss man diese dem Fronius Wechselrichter als externen Energiezähler (GEN24) zugänglich machen.

Dazu wird ein ModbusTCP Server angelegt der den Energiezähler so simuliert, dass der Fronius Wechselrichter sich aus dem Register dieses Modbus Servers die Werte holen kann.
In die Register des Modbus Server muss man neben Standardwerte zur Identifikation als Energiezähler noch den benötigten `AC Power value (Total) [W]` des Erzeugers/Verbrauchers in die dafür vorgesehen Register schreiben.
Die Register Mappings können [hier von Fronius](https://www.fronius.com/QR-link/0006) runtergeladen werden, oder siehe im [Anhang](Meter_Register_Map_Float_v1.0.xlsx).
(Beachten ob im Fronius Wechselrichter "Float" oder "Int+SF" eingestellt ist!)

# Variante 1: Einen Balkonwechselrichter als externen PV-Erzeuger registrieren
In node-red den [Flow](shelly_pv_erzeuger_flow.json) (angelehnt an diesen [flow aus dem node-red Forum](https://discourse.nodered.org/t/simulate-a-modbus-tcp-server-and-feed-registers/78763)) importieren.
Nach einem boot/deployment werden automatisch die einmalig benötigten Daten in den Modbus Server geschrieben.
Außerdem wird jede Sekunde beim Shelly die Erzeugungsdaten abgefragt und diese (muss negativ sein, da Erzeugung) in den Modbus Server geschrieben.
(Es stehen weitere lesende Operationen im flow bereit, diese werden aber für den Anwendungsfall nicht benötigt und sie nur zur Kontrolle da.)

## Nach dem Import des flows muss noch folgendes gemacht werden:
- Shelly IP Adresse im Node anpassen

# Variante 2: Einen Hardy Barth Ladepunkt als externen Verbraucher registrieren
In node-red den [Flow](hardy_barth_verbraucher_flow.json) (angelehnt an diesen [flow aus dem node-red Forum](https://discourse.nodered.org/t/simulate-a-modbus-tcp-server-and-feed-registers/78763)) importieren.
Nach einem boot/deployment werden automatisch die einmalig benötigten Daten in den Modbus Server geschrieben.
Außerdem wird jede Sekunde ein http request an die Ladesäule gesendet und die Verbrauchsdaten abgefragt und diese (muss positiv sein, da Verbrauch) in den Modbus Server geschrieben.
(Es stehen weitere lesende Operationen im flow bereit, diese werden aber für den Anwendungsfall nicht benötigt und sie nur zur Kontrolle da.)

## Nach dem Import des flows muss noch folgendes gemacht werden:
- http request Adresse im Node überprüfen, müsste aber generisch für Anwendungsfälle mit nur einem Ladepunkt so passen
- wenn die IP Adresse `http://ecb1.local` nicht aufgelöst werden kann, muss die tatsächliche IP der Ladestation eingetragen werden

# Anzeige im SolarWeb
Im Web Interface des Fronius Wechselrichters muss man nun noch den Modbus Server als Energiezähler hinzufügen:
- **RaspberryPi IP Adresse** mit **Port 502** und **UnitID 3**
  
![Energiezaehler im Fronius](Energiezaehler.jpg)

Anschließend kann man den Unterschied zwischen SolarWeb und dem Fronius Wechselrichter sehen.
In dem Fall, wurde ein PV-Erzeuger (Variante 1) eingebunden:
![Vergleich Fronius Wechselrichter und SolarWeb](Vergleich.jpg)

# WORK IN PROGRESS: Nice to have
## Reboot/Shutdown über node-red
Damit man das Raspberry Pi über node-red neustarten und ausschalten kann, muss man den [Flow](shutdown_reboot_flow.json) importieren.
Der User, der den Docker Container startet hat aber leider nur die Rechte einen Reboot zu machen. 
Für Shutdown braucht speziell die Erlaubnis den shutdown Befehl auszuführen.
-> visudo!???
