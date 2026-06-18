# SystaBus

Reverse-Engineering und ESPHome-Dokumentation für Paradigma SystaSolar Aqua am SystaBus.

## Aktueller Stand

Der zentrale Monitorframe `FC 24 0B 01` ist für die getestete SystaSolar Aqua weitgehend dekodiert.

Bestätigt sind unter anderem:

- TSA / Kollektor
- TSE / Eintritt
- TWU / Speicher unten
- TW2 / Speicher 2 beziehungsweise fehlender Sensor
- PSO / Solarpumpe in Prozent
- ULV / Umlenkventil, bisher immer 0 beobachtet
- Status Solar
- Störcode
- Reglerzeit
- Tagesertrag
- Gesamtertrag
- berechnete Solarleistung über Volumenstrom und ΔT

## Dateien

- [`docs/AquaBus_Reverse_Engineering.md`](docs/AquaBus_Reverse_Engineering.md) – komplette Reverse-Engineering-Dokumentation
- [`examples/esphome/esp8266-systasolar-aqua.yaml`](examples/esphome/esp8266-systasolar-aqua.yaml) – ESPHome-Beispiel für ESP8266/D1 Mini
- [`examples/esphome/esp32-s3-systasolar-aqua.yaml`](examples/esphome/esp32-s3-systasolar-aqua.yaml) – ESPHome-Beispiel für ESP32-S3

## Wichtige Hinweise

Die ESPHome-Beispiele enthalten keine privaten Zugangsdaten. WLAN, API-Key, OTA-Passwort und AP-Passwort müssen über ESPHome-Secrets oder eigene Werte ergänzt werden.

Schreibtelegramme können Reglerparameter verändern. Nutzung auf eigene Gefahr.

## Offene Punkte

- unbekannte FC24-Bytes `22..23` und `32..37`
- Volumenstrom im Bus noch nicht gefunden
- minimaler SET-Befehl noch nicht sicher
- ESP32-S3 Pinout auf Adapter final testen
- ULV-Verhalten noch nicht bestätigt
