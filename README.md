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

## nore-red Container starten
https://nodered.org/docs/getting-started/docker
```
sudo docker run -d --restart unless-stopped -p 1880:1880 -v node_red_data:/data --name mynodered nodered/node-red
```
-d = detached (keine logs on der aktuellen shell)
-it = attached (man sieht alles in der aktuellen shell, kann diese aber nicht mehr verwenden 

Kontrolle ob der Container läuft:
```
docker ps
docker volume ls
```
Im Browser die IP+port 1880 an surfen -> nore-red ist erreichbar

Zum Beenden des Containers:
```
sudo docker rm -f mynodered
```

## docker Einstellungen
Damit beim Start des Pi docker gestartet wird:
```
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```
(Der Container wird immer gestartet wenn docker gestartet wird, wenn "--restart unless-stopped" mitgegeben wurde)

## WLAN
Damit das RaspberryPi sich direkt mit einem WLAN verbindet, welches an dem Ort vorhanden ist, wo man das RaspberryPi verwenden will, muss man es vorher mit dem NetworkManager einrichten:
https://raspberrytips.com/raspberry-pi-wifi-setup/#set-up-your-wifi-on-raspberry-pi-os-lite
```
sudo nmtui
```
Dann manuell das WLAN der Remote Location eingeben.
Manuell müsste es auch gehen, dann muss man eine "<Wifi-Name>.nmconnection" Datei anlegen, siehe https://forums.raspberrypi.com/viewtopic.php?t=360175

## nore-red flow importieren
Flow von https://discourse.nodered.org/t/simulate-a-modbus-tcp-server-and-feed-registers/78763 kopieren und anpassen.
- IP Adresse von Shelly anpassen 
- IP Adresse von Fronius Wechselrichter anpassen
- ???
