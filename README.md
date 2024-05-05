# Wie man ein RaspberryPi mit docker und node-red einrichtet

SD Karte mit `Raspberry Imager` flashen, hier direkt ein WLAN angeben, welches man bei der weiteren Einrichtung verwenden will.
(Später kann man dann noch das WLAN einrichten, welches dann Vorort existiert).
Nach der Installation per SSH auf das PI verbinden

## Docker installieren: 
```
sudo curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

In der Console prüfen ob es funktioniert hat:
```
docker
```

Pi in docker Gruppe hinzufügen, dass man Container starten kann als pi user:
```
sudo usermod -aG docker pi
```
TODO: Wieso muss man dann bei den docker Befehlen im Anschluss noch "sudo" davor schreiben?

## node-red als docker Container starten
https://nodered.org/docs/getting-started/docker
```
sudo docker run -d --restart unless-stopped -p 1880:1880 -p 502:502 -v node_red_data:/data --name mynodered nodered/node-red
```
```
-d = detached (keine logs on der aktuellen shell)
-it = attached (man sieht alles in der aktuellen shell, kann diese aber dann auch nicht mehr verwenden
-p 1880:1880 (Port von Node-Red mappen und von außen zugänglich machen)
-p 502:502 (Port des ModbusTCP Server mappen und von außen zugänglich machen)
```

Kontrolle ob der Container läuft:
```
docker ps
docker volume ls
```
Im Browser die IP mit Port 1880 an surfen (zB 192.168.178.40:1800) -> node-red ist erreichbar

Zum Beenden des Containers:
```
sudo docker rm -f mynodered
```

## docker Einstellungen
Damit beim Start (boot) des Pi auch docker automatisch gestartet wird:
```
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```
(Der node-red Container wird immer gestartet wenn docker gestartet wird, wenn "--restart unless-stopped" oben mitgegeben wurde)

## WLAN
Damit das RaspberryPi sich direkt mit einem WLAN verbindet, welches an dem Ort wo man es verwenden will, muss man die Verbindung vorher [mit dem NetworkManager einrichten](https://raspberrytips.com/raspberry-pi-wifi-setup/#set-up-your-wifi-on-raspberry-pi-os-lite) einrichten:
```
sudo nmtui
```
(Manuell müsste es auch gehen, dann muss man eine "\<Wifi-Name\>.nmconnection" Datei anlegen, siehe https://forums.raspberrypi.com/viewtopic.php?t=360175)

# Mittels node-red einen Balkonwechselrichter als externen PV-Erzeuger registrieren
Damit die PV Erzeugung im Fornius SolarWeb neben einem Fornius Wechselrichter auch einen weiteren Balkonwechselrichter berücksichtigt, muss man diese dem Fronius Wechselrichter als externen Energiezähler zugänglich machen.

Dazu muss man einen ModbusTCP Server anlegen der einen Energiezähler (GEN24) so simuliert, dass der Fronius Wechselrichter sich aus dem Register des Modbus Servers die Werte holen kann.
In die Register des Modbus Server muss man also die benötigten Werte (`AC Power value (Total) [W]`) aus dem externen PV-Erzeuger (zB Balkonkraftwerk) in die dafür vorgesehen Register schreiben.
Die Register Mappings können [hier von Fornius](https://www.fronius.com/QR-link/0006) runtergeladen werden, oder siehe im [Anhang](Meter_Register_Map_Float_v1.0.xlsx).
(Beachten ob im Fronius Wechselrichter "Float" oder "Int+SF" eingestellt ist!)

## Dependecies in node-red installieren
- [node-red-contrib-shelly](https://flows.nodered.org/node/node-red-contrib-shelly) zum Auslesen der PV Erzeugung von einem Shelly
- [node-red-contrib-modbus](https://flows.nodered.org/node/node-red-contrib-modbus) zum Schreiben der Daten mittels Modbus in den Fronius Wechselrichter
- [node-red-contrib-buffer-parser](https://flows.nodered.org/node/node-red-contrib-buffer-parser) zum einfachen Konvertieren von Registern zu lesbaren Werten und zurück (optional)

## Node-Red Flow
In node-red den [Flow](flow.json) (angelehnt an diesen [flow](https://discourse.nodered.org/t/simulate-a-modbus-tcp-server-and-feed-registers/78763)) importieren.
In dem Flow werden nach einem boot/deployment einmalig die benötigten Daten, dass der Fronius Wechselrichter den Modbus Server als GEN24 Energiezähler erkennt in den Modbus Server geschrieben.
Außerdem wird jede Sekunde beim Shelly die Erzeugungsdaten abgefragt und diese (muss negativ sein, da Erzeugung) in den Modbus Server geschrieben.
(Es stehen weitere lesende Operationen im Flow bereit, diese werden aber für den Workflow nicht benötigt und sie nur zur Kontrolle da.)

### Nach dem Import des flows muss noch folgendes gemacht werden:
- Shelly IP Adresse im Node anpassen
- Im Web Interface des Fronius Wechselrichters den Modbus Server als Energiezähler hinzufügen:
-- **RaspberryPi IP Adresse** mit **Port 502** und **UnitID 3**
![Energiezaehler im Fornius](https://raw.githubusercontent.com/MatthiasOs/raspberry-pi-docker-node-red/main/Energiezaehler.jpg)

## Ergebnis
Im SolarWeb kann man dann im Vergleich zur Erzeugung im Fornius Wechselrichter sehen, dass das Balkonkraftwerk korrekt eingebunden wurde:
![Vergleich Fronius Wechselrichter und SolarWeb](https://raw.githubusercontent.com/MatthiasOs/raspberry-pi-docker-node-red/main/Vergleich.jpg)
Der Unterschied ist genau die Erzeugung des Balkonwechselrichters.
