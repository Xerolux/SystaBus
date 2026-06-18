# Monitoring und offene Punkte

## Neu in den ESPHome-Beispielen

Die Beispielkonfigurationen für ESP8266 und ESP32-S3 enthalten jetzt zusätzliche Monitoring- und Diagnosefunktionen.

### SystaBus Parse Errors

Sensor:

```yaml
sensor:
  - platform: template
    name: "SystaBus Parse Errors"
```

Gezählt werden aktuell:

- zu kurze `FC 24` Frames
- FC24 Frames mit ungültiger Checksumme

Damit lässt sich in Home Assistant prüfen, ob die Buskommunikation stabil läuft.

### Last Successful Update

Textsensor:

```yaml
text_sensor:
  - platform: template
    name: "Last Successful Update"
```

Der Sensor wird bei jedem erfolgreich dekodierten FC24-Frame aktualisiert. Wenn Home-Assistant-Zeit verfügbar ist, wird diese verwendet. Sonst wird die Reglerzeit angezeigt.

### Solar Aktiv

Binary Sensor:

```yaml
binary_sensor:
  - platform: template
    name: "Solar Aktiv"
```

Aktiv wenn:

```text
status_solar == 3 && pso > 0
```

Damit können Automationen sehr einfach auf laufende Solareinspeisung reagieren.

### Temperatur-Range-Validierung

Temperaturwerte werden nur veröffentlicht, wenn sie grob plausibel sind:

```text
-40 °C < Temperatur < 250 °C
```

TW2 wird zusätzlich nur veröffentlicht, wenn der Sensor wirklich vorhanden wirkt:

```text
-20 °C < TW2 < 160 °C
```

### Störtext

Zusätzlich zum numerischen Störcode gibt es einen Textsensor:

```yaml
text_sensor:
  - platform: template
    name: "Stoerung Solar Text"
```

Bisher bestätigt:

| Code | Text |
|---:|---|
| 0 | Keine Stoerung |

Weitere Störcodes müssen erst real beobachtet werden.

---

## Weiterhin offen

### FC24 Bytes 22..23

Noch unbekannt. Bisher meist `00 00` beobachtet.

Mögliche Bedeutungen:

- Reserve
- Zähler
- Fehlzirkulation
- Diagnosewert

### FC24 Bytes 32..37

Noch nicht final dekodiert.

Beobachtet wurden Werte wie:

```text
29 00 80 00 11 00
29 00 00 00 00 00
```

Mögliche Bedeutungen:

- Flags
- Tastenzustand
- Diagnosemerkmale
- Firmware-/Modulzustand

### Volumenstrom

Der Volumenstrom wurde am Reglerdisplay als Parameter `2,5 l/min` abgelesen. Im FC24-Frame wurde bisher kein eindeutig dynamischer Volumenstromwert gefunden.

Die Solarleistung wird deshalb aktuell berechnet:

```text
P [W] = Volumenstrom [l/min] × (TSA - TSE) × 69,78
```

### Minimaler SET-Befehl

Bekannt sind lange SET-Telegramme für:

- Auto
- Aus
- Hand

Noch offen ist, ob ein kürzerer, sicherer Befehl existiert oder ob der Regler immer einen größeren Parameterblock erwartet.

### ESP32-S3 Pinout

Die ESP32-S3 Beispielkonfiguration nutzt aktuell:

```yaml
rx_pin: GPIO16
tx_pin: GPIO17
```

Falls der Adapter die Datenleitung mechanisch auf D1-Mini-GPIO13 führt, muss `rx_pin` auf `GPIO13` geändert oder die Leitung umverdrahtet werden.

### ULV Verhalten

`frame[13]` ist als ULV/Umlenkventil plausibel, wurde bisher aber nur mit Wert `0` gesehen. Ein real aktives Umlenkventil muss noch getestet werden.

### Langzeittest

Noch ausstehend:

- ESP8266 ohne Raw-Frame im Dauerbetrieb
- ESP32-S3 mit Hardware-UART im Dauerbetrieb
- Anzahl Parse Errors pro Tag
- Verhalten bei WLAN-Reconnects
