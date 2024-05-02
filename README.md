# Wie man ein RaspberryPi mit docker und nore-red einrichtet

SD Karte mit raspberry imager flashen, hier kann man direkt ein WLAN mit angeben (wpa_supplicant Datei funktioniert nicht mehr, keine Alternative?)

## Docker installieren: 
```
sudo curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

In der console prüfen ob es funktioniert hat:
```
docker
```

Pi in docker Gruppe hinzufügen, dass man Container starten kann als pi user:
```
sudo usermod -aG docker pi
```
TODO: Wieso muss man dann bei den docker Befehlen im Anschluss noch "sudo" davor schreiben?

## nore-red Container starten
https://nodered.org/docs/getting-started/docker
```
sudo docker run -d --restart unless-stopped -p 1880:1880 -v node_red_data:/data --name mynodered nodered/node-red
```
```
-d = detached (keine logs on der aktuellen shell)
-it = attached (man sieht alles in der aktuellen shell, kann diese aber nicht mehr verwenden
```

Kontrolle ob der Container läuft:
```
docker ps
docker volume ls
```
Im Browser die IP+port 1880 an surfen (zB 192.168.178.40:1800) -> node-red ist erreichbar

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
(Der node-red Container wird immer gestartet wenn docker gestartet wird, wenn "--restart unless-stopped" mitgegeben wurde)

## WLAN
Damit das RaspberryPi sich direkt mit einem WLAN verbindet, welches an dem Ort vorhanden ist, wo man es verwenden will, muss man die Verbindung vorher [mit dem NetworkManager einrichten](https://raspberrytips.com/raspberry-pi-wifi-setup/#set-up-your-wifi-on-raspberry-pi-os-lite):
```
sudo nmtui
```
(Manuell müsste es auch gehen, dann muss man eine "\<Wifi-Name\>.nmconnection" Datei anlegen, siehe https://forums.raspberrypi.com/viewtopic.php?t=360175)

# Mittels node-red einen Balkonwechselrichter als externen PV-Erzeuger einrichten
Damit die PV Erzeugung im Fornius Portal auch einen weiteren Balkonwechselrichter berücksichtigt, muss man diese manuell dem Fronius Wechselrichter mitteilen.

Es gibt dafür zwei Wege, entweder mit OpenDTU einen GEN24 Energiezähler im Netzwerk simulieren, dann holt sich Fronius die Daten selbst ab (https://github.com/AloisKlingler/OpenDTU-FroniusSM-MB/), oder die Werte in die korrekten Modbus Register des Fronius schreiben.
Mit node-red kommt nur der letzte Weg in Frage. Es gibt auch eine Fornius API, diese unterstützt aber aktuell nur HTTP GET (lesen) und kein POST (schreiben).

Es wird also in node-red ein flow benötigt, der die PV Erzeugungsdaten der Balkonwechselrichter (AC Power value (Total) [W]", "Total Watt Hours Exportet [Wh]" und "Total Watt Hours Imported [Wh]") zuerst ausliest, die Daten in die korrekte Form bringt und dann per Modbus in die entsprechenden Register des Fronius Wechselrichter schreibt.

## Dependecies in node-red installieren
- [node-red-contrib-shelly](https://flows.nodered.org/node/node-red-contrib-shelly) zum Auslesen der PV Erzeugung von einem Shelly
- [node-red-contrib-modbus](https://flows.nodered.org/node/node-red-contrib-modbus) zum Schreiben der Daten mittels Modbus in den Fronius Wechselrichter

## Flow importieren (WORK IN PROGRESS)
In node-red den [flow.json](flow.json) ([Quelle](https://discourse.nodered.org/t/simulate-a-modbus-tcp-server-and-feed-registers/78763)) importieren
- IP Adresse von Shelly anpassen 
- IP Adresse von Fronius Wechselrichter anpassen
- ???
