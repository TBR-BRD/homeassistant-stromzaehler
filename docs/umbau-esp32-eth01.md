# Umbau auf ESP32-ETH01

Diese Anleitung beschreibt den Umbau des bestehenden WLAN-Stromzaehleradapters im Hauswirtschaftsraum auf einen kabelgebundenen ESP32-ETH01 sowie die Integration des bereits vorhandenen PV-ESP32-ETH01.

## Zielbild

```text
Itron Zweirichtungszaehler
  -> bitShake UART-Lesekopf
  -> ESP32-ETH01 mit Tasmota SML
  -> Ethernet
  -> MQTT Broker in Home Assistant
  -> Home Assistant Energy

PV-Zaehler
  -> UART/SML-Lesekopf
  -> ESP32-ETH01 mit Tasmota SML
  -> Ethernet
  -> MQTT Broker in Home Assistant
  -> Home Assistant Energy
```

## Hardware

- ESP32-ETH01 oder WT32-ETH01
- bitShake SmartMeterReader-UART, 3 bis 5 V
- stabile 5-V-Versorgung fuer das Ethernet-Board
- kurze Dupont- oder Schraubklemmenleitung fuer `VCC`, `GND`, `TX`
- Ethernet-Anschluss im Hauswirtschaftsraum

## Anschluss

1. ESP32-ETH01 spannungsfrei machen.
2. bitShake `GND` mit ESP32 `GND` verbinden.
3. bitShake `VCC` mit der passenden Versorgung verbinden.
4. bitShake `TX` mit ESP32 `GPIO5` verbinden.
5. bitShake `RX` nicht anschliessen.
6. Ethernet einstecken.
7. Lesekopf auf die optische Schnittstelle des Zaehler setzen.

Warum `TX -> GPIO5`: Im Tasmota-Script ist die SML-Schnittstelle als `+1,5,s,0,9600,...` definiert. Die `5` steht fuer GPIO5 als Empfangspin.

## Tasmota vorbereiten

1. Tasmota32 mit SML/Scripting-Unterstuetzung flashen.
2. Weboberflaeche oeffnen.
3. MQTT konfigurieren:
   - Host: `<MQTT_HOST>`
   - Port: `1883`
   - User: Home-Assistant-MQTT-User
   - Passwort: nicht ins Repo schreiben
   - Topic fuer HWR z. B. auf `stromzaehler_hwr` setzen oder die Home-Assistant-YAML entsprechend anpassen
4. TelePeriod auf 20 Sekunden setzen:

```text
TelePeriod 20
```

5. Script aktivieren und speichern.
6. Neustarten.

## Testbefehle

Hauswirtschaftsraum:

```bash
curl http://<HWR_TASMOTA_IP>/cm?cmnd=Status%2010
```

PV:

```bash
curl http://<PV_TASMOTA_IP>/cm?cmnd=Status%2010
```

Erwartete Werte:

- `HWR_Pow.Total_in`
- `HWR_Pow.Power_curr`
- `HWR_Pow.Total_out`
- `PV.PVL`
- `PV.PVG`

## Home Assistant

1. MQTT-Integration aktivieren.
2. Datei `homeassistant/mqtt-sensors.yaml` in die Home-Assistant-Konfiguration uebernehmen.
3. In `configuration.yaml` einbinden:

```yaml
mqtt: !include homeassistant/mqtt-sensors.yaml
```

Falls bereits ein `mqtt:`-Block existiert, nur den `sensor:`-Teil uebernehmen.

4. Home Assistant neu starten.
5. Unter `Entwicklerwerkzeuge -> Zustaende` pruefen:
   - `sensor.hwr_netzbezug_total`
   - `sensor.hwr_netzeinspeisung_total`
   - `sensor.pv_erzeugung_total`
6. Energy-Dashboard konfigurieren:
   - Netzverbrauch: `sensor.hwr_netzbezug_total`
   - Rueckspeisung: `sensor.hwr_netzeinspeisung_total`
   - Solarproduktion: `sensor.pv_erzeugung_total`

## Fehlersuche

Keine Werte:

- Lesekopf drehen oder genauer aufsetzen.
- GND pruefen.
- TX des Lesekopfs muss an GPIO5.
- Script muss aktiviert sein.
- `Sensor53 r` in Tasmota-Konsole ausfuehren.

Werte kommen in Tasmota, aber nicht in Home Assistant:

- MQTT-Verbindung in Tasmota-Status pruefen.
- Topic pruefen.
- MQTT Explorer oder Home-Assistant-MQTT-Integration verwenden.
- Home-Assistant-Sensor-Topics anpassen.

Energy akzeptiert Sensor nicht:

- Einheit muss `kWh` sein.
- `device_class: energy`
- `state_class: total_increasing`
- Sensor muss numerische Werte liefern.
