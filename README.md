# Home Assistant Stromzaehler

Dokumentation und Beispielkonfigurationen fuer zwei Tasmota/ESP32-Zaehleradapter im Hausnetz:

- Hauswirtschaftsraum: Netzbezug und Netzeinspeisung am Itron Zweirichtungszaehler mit bitShake SmartMeterReader-UART.
- PV-Anlage: PV-Erzeugung der 8-kW-Anlage mit ESP32-ETH01 / WT32-ETH01.

Ziel ist die kabelgebundene Integration beider Zaehler in Home Assistant Energy unter:

```text
http://<HOME_ASSISTANT_IP>:8123/energy/overview
```

## Bestand

| Aufgabe | IP | Hardware | Firmware | Messwerte |
| --- | --- | --- | --- | --- |
| Hauswirtschaftsraum Netzzaehler | `<HWR_TASMOTA_IP>` | aktuell Tasmota/WLAN, Umbau auf ESP32-ETH01 geplant | Tasmota | Bezug, aktuelle Leistung, Einspeisung |
| PV-Erzeugung | `<PV_TASMOTA_IP>` | WT32-ETH01 / ESP32-ETH01 | Tasmota | PV-Bezug/Zaehlerstand, PV-Leistung, PV-Gesamterzeugung |

Der bitShake-UART-Lesekopf arbeitet mit 3 bis 5 V und liefert die SML-Daten seriell an den ESP32.

Beispielhafte Tasmota-MQTT-Topics:

- Hauswirtschaftsraum: `<HWR_TASMOTA_TOPIC>`
- PV-Anlage: `<PV_TASMOTA_TOPIC>`

Beim Umbau des Hauswirtschaftsraum-Adapters auf ein neues ESP32-ETH01-Board sollte der Topic bewusst gesetzt werden, z. B. sprechend auf `stromzaehler_hwr`. Die Home-Assistant-YAML muss zum Topic passen.

## Relevante Referenzen

- ESPHome kann SML direkt ueber UART lesen; zum Finden der OBIS-Codes empfiehlt die Doku Debug-Logs der SML-Komponente. Siehe [ESPHome SML](https://esphome.io/components/sml/).
- ESPHome unterstuetzt kabelgebundenes Ethernet fuer ESP32-Boards. Siehe [ESPHome Ethernet](https://esphome.io/components/ethernet/).
- bitShake beschreibt die Home-Assistant-Anbindung per MQTT/Tasmota fuer den SmartMeterReader. Siehe [bitShake Home Assistant Support](https://www.bitshake.de/support/home-assistant/).
- Vergleichbare Projekte nutzen ESPHome oder Tasmota, UART/SML und Home Assistant, z. B. [Afraskai/ESPHome_smartmeter](https://github.com/Afraskai/ESPHome_smartmeter) und das Home-Assistant-Community-Beispiel [PoESP32 SML Meter](https://community.home-assistant.io/t/integrate-sml-meter-using-poesp32-replaces-volkszahler-rpi/928723).
- Home Assistant Energy erwartet langfristige Energie-Sensoren mit passenden Einheiten und `state_class: total_increasing`. Siehe [Home Assistant Electricity Grid](https://www.home-assistant.io/docs/energy/electricity-grid/).

## Verdrahtung ESP32-ETH01 mit bitShake UART

Empfohlene Verdrahtung fuer Tasmota-SML am ESP32-ETH01:

| bitShake SmartMeterReader-UART | ESP32-ETH01 / WT32-ETH01 |
| --- | --- |
| `VCC` / `3-5V` | `3V3` oder stabile `5V`, passend zum Board-Aufbau |
| `GND` | `GND` |
| `TX` vom Lesekopf | `GPIO5` / ESP32 RX fuer Tasmota `+1,5,s,0,9600,...` |
| `RX` vom Lesekopf | nicht benoetigt, offen lassen |

Hinweise:

- Der Lesekopf sendet Daten, deshalb wird sein `TX` mit dem RX-GPIO des ESP32 verbunden.
- Beim bestehenden PV-Adapter wird GPIO `5` mit 9600 Baud genutzt.
- ESP32-ETH01 braucht eine stabile Versorgung. Wenn das Ethernet-Board ueber 5 V versorgt wird, darf die UART-Logik trotzdem nur 3,3-V-kompatibel am ESP32-Pin anliegen. Der bitShake-UART-Leser ist laut Aufdruck fuer 3 bis 5 V geeignet.
- Der Lesekopf muss magnetisch exakt auf der optischen Schnittstelle des Zaehler sitzen. Bei falscher Orientierung kommen keine oder unvollstaendige SML-Telegramme.

## OBIS-Codes

### Hauswirtschaftsraum / Netzzaehler

Aus dem bestehenden Tasmota-Script:

| Wert | OBIS | Tasmota-Feld | Einheit | Bedeutung |
| --- | --- | --- | --- | --- |
| Netzbezug Zaehlerstand | `1-0:1.8.0*255` | `HWR_Pow.Total_in` | kWh | Energie aus dem Netz |
| aktuelle Leistung | `1-0:16.7.0*255` | `HWR_Pow.Power_curr` | W | Momentanleistung |
| Netzeinspeisung Zaehlerstand | `1-0:2.8.0*255` | `HWR_Pow.Total_out` | kWh | Energie zum Versorger |

Live-Beispiel am 2026-10-10:

```json
{
  "HWR_Pow": {
    "Total_in": 46317.3878,
    "Power_curr": 653,
    "Total_out": 38914.2942
  }
}
```

### PV-Anlage

Aus dem bestehenden ESP32-ETH01-Script:

| Wert | OBIS | Tasmota-Feld | Einheit | Bedeutung |
| --- | --- | --- | --- | --- |
| PV-Bezugszaehler | `1-0:1.8.0*255` | `PV.PVB` | kWh | Nebenwert im PV-Zaehler |
| PV-Leistung | `1-0:16.7.0*255` | `PV.PVL` | W | aktuelle PV-Leistung |
| PV-Gesamterzeugung | `1-0:2.8.0*255` | `PV.PVG` | kWh | Erzeugte PV-Energie |

Live-Beispiel am 2026-10-10:

```json
{
  "PV": {
    "PVB": 278.7192,
    "PVL": 3,
    "PVG": 69187.8713
  }
}
```

## Umsetzung mit Tasmota

1. ESP32-ETH01 mit Tasmota32 flashen, idealerweise mit Scripting/SML-Unterstuetzung.
2. Ethernet konfigurieren, z. B. per Weboberflaeche oder Tasmota-Konsole.
3. Lesekopf anschliessen: `TX` des bitShake-UART-Lesers an GPIO5, `GND` gemeinsam, Versorgung anschliessen.
4. Script unter `Tools -> Edit Script` eintragen.
5. `Script enable` aktivieren und speichern.
6. In der Konsole `Sensor53 r` ausfuehren oder Neustart durchfuehren.
7. Unter `http://<ip>/cm?cmnd=Status%2010` pruefen, ob die Werte erscheinen.
8. MQTT auf `<MQTT_HOST>:1883` konfigurieren.
9. Home Assistant MQTT-Sensoren anlegen.
10. In Home Assistant Energy Grid/PV-Sensoren auswaehlen.

## Home Assistant Energy Zuordnung

In Home Assistant unter `Einstellungen -> Dashboards -> Energie`:

- Stromnetz Verbrauch aus Netz: `sensor.hwr_netzbezug_total`
- Rueckgabe ans Netz: `sensor.hwr_netzeinspeisung_total`
- Solarproduktion: `sensor.pv_erzeugung_total`

Momentanleistungen wie `Power_curr` oder `PVL` sind hilfreich fuer Dashboards, aber fuer Energy werden die kumulativen kWh-Zaehlerstaende benoetigt.

## Dateien

- [`tasmota/pv-esp32-eth01.script`](tasmota/pv-esp32-eth01.script): bereinigtes Script fuer den bestehenden PV-Adapter.
- [`tasmota/hwr-stromzaehler-esp32-eth01.script`](tasmota/hwr-stromzaehler-esp32-eth01.script): Script fuer den Netzzaehler nach Umbau auf ESP32-ETH01.
- [`homeassistant/mqtt-sensors.yaml`](homeassistant/mqtt-sensors.yaml): MQTT-Sensoren fuer Home Assistant.
- [`docs/umbau-esp32-eth01.md`](docs/umbau-esp32-eth01.md): detaillierte Umbauanleitung.

## Sicherheit

Dieses Repository ist oeffentlich. API-Keys, MQTT-Passwoerter und pvoutput.org-Keys gehoeren nicht ins Repo. In den Beispiel-Scripts sind deshalb Platzhalter eingetragen.
