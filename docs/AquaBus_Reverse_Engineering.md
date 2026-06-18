# AquaBus Reverse Engineering

> Community-Dokumentation für den Paradigma SystaSolar Aqua / SystaBus.
>
> Dieses Dokument fasst die bisherigen Erkenntnisse aus Messungen, Display-Vergleichen,
> ESPHome-Logs und praktischen Tests zusammen. Es ist bewusst als Arbeitsstand
> geschrieben und darf/soll erweitert werden.

## 1. Ziel

Ziel ist das Lesen und teilweise Schreiben von Daten auf dem Solar-Bus einer
Paradigma SystaSolar Aqua Steuerung mit einem ESP-basierten Adapter und ESPHome.

Aktueller Stand:

- FC24-Monitortelegramm wird stabil gelesen.
- Temperaturen, Pumpenansteuerung, Status, Störung, Tagesertrag und Gesamtertrag sind dekodiert.
- Auto/Aus/Hand-Schreibtelegramme sind bekannt, aber mit Vorsicht zu verwenden.
- ESP8266 funktioniert, zeigt aber unter ESPHome/WLAN/UART Last Watchdog-Resets.
- ESP32-S3 Mini wird als stabilere Zielplattform empfohlen.

---

## 2. Hardware

### 2.1 Getestete Steuerung

- Paradigma SystaSolar Aqua
- Display-Firmware im Test: `V3.00 060613`
- Bus-Anschluss am Regler: `BUS + / -`

### 2.2 Getesteter Adapter

Im Projekt wurde ein kleiner Bus-Leser verwendet. Zur Vermeidung von Namens-/Markenverwechslungen
wird er hier neutral **AquaBus Adapter** genannt.

Typischer Aufbau:

- Bus-Klemme `BUS+ / BUS-`
- Optokoppler/Transistor-Stufe zur Pegelaufbereitung
- ESP8266 D1 Mini kompatibles Steckmodul
- Datenleitung auf GPIO13
- Versorgung über USB oder Adapterplatine

### 2.3 Getesteter ESP

Verwendet wurde ein D1-Mini-kompatibles ESP8266-Board:

- ESP8266EX
- CH340C USB-Seriell
- USB-C
- D1-Mini-Pinout

### 2.4 Empfohlener ESP für Weiterentwicklung

Empfohlen: **LOLIN ESP32-S3 Mini**

Gründe:

- deutlich mehr RAM
- stabilere UART-Verarbeitung
- echte ESP32-Plattform statt ESP8266-SoftwareSerial
- ESPHome gut unterstützt
- gleiche grobe D1-Mini-Bauform
- GPIO13 vorhanden

Nicht blind 1:1 einstecken, vorher Pinout prüfen. Mechanisch sieht es passend aus,
elektrisch muss die Datenleitung des Adapters auf den passenden ESP32-S3-GPIO gelegt werden.

---

## 3. Anschluss / Pinbelegung

### 3.1 Reglerseite

Am SystaSolar Aqua Regler:

| Reglerklemme | Bedeutung |
|---|---|
| BUS + | Bus Plus |
| BUS - | Bus Minus |

### 3.2 Adapterseite

Beobachtete/abgeleitete Adapter-Pins:

| Adapter-Beschriftung | Bedeutung |
|---|---|
| BUS + | an Regler BUS + |
| BUS - | an Regler BUS - |
| DATA | aufbereitete Bus-Daten |
| 3.3V | Logikversorgung |
| GND | Masse |
| 5V | Versorgung |
| GPIO13 | Datenleitung zum ESP |

### 3.3 ESPHome UART, ESP8266

Aktuell genutzt:

```yaml
uart:
  id: uart_bus
  tx_pin: GPIO1
  rx_pin: GPIO13
  baud_rate: 9600
  data_bits: 8
  parity: NONE
  stop_bits: 1
```

Hinweis: Auf ESP8266 ist GPIO13 hier über ESPHome als SoftwareSerial im Einsatz.
Das ist wahrscheinlich die Ursache für Watchdog-Resets bei hoher Last.

---

## 4. UART / Busparameter

Bisher funktionierende Parameter:

| Parameter | Wert |
|---|---|
| Baudrate | 9600 |
| Datenbits | 8 |
| Parität | NONE |
| Stopbits | 1 |
| Richtung | Lesen RX; Schreiben über UART möglich |
| Frame-Erkennung | über Startbytes |

---

## 5. Checksumme

Für die beobachteten Telegramme gilt:

```cpp
uint8_t sum = 0;
for (size_t i = 0; i < frame.size() - 1; i++) {
  sum += frame[i];
}
bool ok = frame.back() == (uint8_t)(0 - sum);
```

Also:

```text
Checksumme = 0x00 - Summe(alle Bytes außer Checksumme)
```

---

## 6. Bekannte Telegrammtypen

### 6.1 FC24 Monitorframe

Dies ist aktuell das wichtigste Telegramm.

Beispiel:

```text
FC 24 0B 01 01 F7 01 C0 02 21 FE E0 00 00 06 00 00 00 17 06 17 06 00 00 00 00 00 0A 00 00 1F 49 29 00 80 00 11 00 AE
```

Interpretation:

```text
TSA    = 50.3 °C
TSE    = 44.8 °C
TWU    = 54.5 °C
TW2    = -28.8 °C / Sensor nicht vorhanden
PSO    = 0 %
ULV    = 0
Status = 6
Störung= 0
Zeit   = 17:06 17.06
Tag    = 10 kWh
Gesamt = 8009 kWh
```

Gesamtlänge im ESPHome-Log: 39 Bytes inklusive Checksumme.

---

## 7. FC24 Offsets

Offset ist nullbasiert, also `frame[0] = 0xFC`.

| Offset | Länge | Beispiel | Bedeutung | Status |
|---:|---:|---|---|---|
| 0 | 1 | FC | Start / Telegrammtyp | bestätigt |
| 1 | 1 | 24 | Länge / Frame-Kennung | bestätigt |
| 2 | 1 | 0B | Kommando | bestätigt |
| 3 | 1 | 01 | Subkommando | bestätigt |
| 4-5 | 2 | 01 F7 | TSA Kollektor, signed int16 / 10 | bestätigt |
| 6-7 | 2 | 01 C0 | TSE Eintritt/Rücklauf, signed int16 / 10 | bestätigt |
| 8-9 | 2 | 02 21 | TWU Speicher unten, signed int16 / 10 | bestätigt |
| 10-11 | 2 | FE E0 | TW2, signed int16 / 10 | bestätigt, bei Anlage nicht vorhanden |
| 12 | 1 | 00 / 32 / 64 | PSO Solarpumpe in % | bestätigt |
| 13 | 1 | 00 | ULV Umlenkventil | plausibel |
| 14 | 1 | 03 / 06 / 08 | Status Solar | bestätigt |
| 15 | 1 | 00 | Störcode | bestätigt |
| 16 | 1 | 00 | Frostschutz | plausibel |
| 17 | 1 | 00 | Ctr / Counter / Diagnose | unklar |
| 18 | 1 | 17 | Stunde, BCD/hex-dezimal | bestätigt |
| 19 | 1 | 06 | Minute, BCD/hex-dezimal | bestätigt |
| 20 | 1 | 17 | Tag, BCD/hex-dezimal | bestätigt |
| 21 | 1 | 06 | Monat, BCD/hex-dezimal | bestätigt |
| 22-23 | 2 | 00 00 | Zähler/Reserve, evtl. Fehlzirkulation | unklar |
| 24-27 | 4 | 00 00 00 0A | Tagesenergie in kWh | bestätigt |
| 28-31 | 4 | 00 00 1F 49 | Gesamtenergie in kWh | bestätigt |
| 32 | 1 | 29 | unbekannt | offen |
| 33 | 1 | 00 | unbekannt / Jahr? | offen |
| 34 | 1 | 80 | Taste/Flags? | offen |
| 35 | 1 | 00 | Diagnose Korr? | offen |
| 36 | 1 | 11 | Diagnose Merkmale 1? | offen |
| 37 | 1 | 00 | Diagnose Merkmale 2? | offen |
| 38 | 1 | AE | Checksumme | bestätigt |

---

## 8. Bestätigte Display-Vergleiche

### 8.1 Temperaturen

Display zeigte unter anderem:

```text
TSA 51,3 °C
TSE 46,5 °C
TWU 57,2 °C
```

Die Buswerte passten zu:

```text
Kollektor / TSA
Eintritt / TSE
Speicher Unten / TWU
```

### 8.2 Erträge

Display zeigte:

```text
Solargewinn Tag: 10 kWh
Solargewinn Gesamt: 8009 kWh
```

Buswerte:

```text
Tagesenergie = 10
Gesamtenergie = 8009
```

### 8.3 Pumpe

Display:

```text
Pumpe Solar PSO 0 %
```

Bus:

```text
frame[12] = 0x00
```

Display Handbetrieb:

```text
Pumpe Solar PSO 100 %
```

Bus:

```text
frame[12] = 0x64
```

Damit ist `frame[12] = PSO in Prozent` bestätigt.

---

## 9. Statuscodes

Status kommt aus `frame[14]`.

| Code | Bedeutung | Status |
|---:|---|---|
| 0 | Maximale Speichertemperatur erreicht | aus Doku/Mapping |
| 1 | Stillstand, Dampf im Kollektor | aus Doku/Mapping |
| 2 | Frostschutzfunktion aktiv | aus Doku/Mapping |
| 3 | Solare Wärme einspeisen | beobachtet/plausibel |
| 4 | Anschiebefunktion aktiv | aus Doku/Mapping |
| 5 | Einschaltverzögerung | aus Doku/Mapping |
| 6 | Betriebsart Hand/Test/Aus | bestätigt |
| 7 | Störabschaltung | aus Doku/Mapping |
| 8 | Kollektortemperatur zu niedrig / abgeschaltet | bestätigt |

### Beobachtungen

#### Status 8

Situation:

```text
TSA < TWU
PSO = 0 %
Reglerstatus Display = Abgeschaltet
```

Interpretation:

```text
Status 8 = Kollektortemperatur zu niedrig / keine Einspeisung
```

#### Status 6

Getestete Betriebsarten:

- Aus
- Test
- Hand

Alle ergaben:

```text
Status = 6
```

Daher kann der FC24-Monitorframe diese drei Betriebsarten nicht sauber unterscheiden.
Er zeigt nur: Regelbetrieb verlassen / Hand-Test-Aus-Gruppe.

---

## 10. PSO / Pumpenansteuerung

`frame[12]`

| Wert hex | Wert dezimal | Bedeutung |
|---|---:|---|
| 00 | 0 | Pumpe aus |
| 32 | 50 | Pumpe 50 % |
| 64 | 100 | Pumpe 100 % |

Die Pumpe ist PWM- oder frequenzgesteuert. PSO ist der Stellwert in Prozent.

---

## 11. Solarleistung

Der FC24-Frame enthält nach aktuellem Stand **keine direkte Leistung in Watt**.

Eine Näherung kann berechnet werden über:

```text
P [W] = Volumenstrom [l/min] × ΔT [K] × 69,78
```

mit:

```text
ΔT = TSA - TSE
```

Bei am Regler eingestelltem Volumenstrom:

```text
Volumenstrom = 2,5 l/min
```

ergibt sich:

```text
P [W] = 2,5 × (TSA - TSE) × 69,78
```

Wichtig:

- Nur sinnvoll, wenn PSO > 0.
- Bei PSO = 0 sollte Leistung = 0 W gesetzt werden.
- Der Volumenstrom ist ein Reglerparameter und nicht sicher im FC24-Frame enthalten.

---

## 12. Bekannte Reglerparameter vom Display

Aus dem Menü abgelesen:

| Parameter | Wert |
|---|---|
| Speicher maximal | 90 °C |
| Speicher OPTIMA / EXPRESSO | Nein |
| Min. Drehzahl PSO | 50 % |
| Schaltdifferenz | 5 K |
| Vorlauf aussetzen | 2 m |
| Zwischenspeichersystem | Nein |
| Volumenstrom | 2,5 l/min |
| Kollektorkaskade | Nein |
| Akustische Alarmierung | Aus |
| Sprache | Deutsch |
| Version | V3.00 060613 |

---

## 13. Schreibtelegramme / SET-Befehle

Es wurden drei lange SET-Sequenzen gefunden.

> Achtung: Diese Sequenzen scheinen nicht nur eine einzelne Betriebsart zu setzen,
> sondern möglicherweise einen kompletten Parameterblock zu übertragen.
> Vor produktiver Nutzung unbedingt am eigenen Regler testen und verstehen.

### 13.1 Solar Auto

```text
0A 53 1D 0B 11 53 45 54 00 01 00 00 00 00 0F 5B 46 40 00 36 B0 A0 00 03 02 A0 02 62 00 02 BC 01 F4 00 32 00 32 00 30 0B 00 00 00 00 00 00 08 2D 0D 81 00 00 00 01 00 00 00 00 00 00 07 C1 C1 00 00 02 00 00 00 00 00 16 00 10 20 06 37 0F 00 00 CB
```

Auffällig:

```text
Modebyte vermutlich = 00
Checksumme = CB
```

### 13.2 Solar Aus

```text
0A 53 1D 0B 11 53 45 54 01 01 00 00 00 00 0F 5B 46 40 00 36 B0 A0 00 03 02 A0 02 62 00 02 BC 01 F4 00 32 00 32 00 30 0B 00 00 00 00 00 00 08 2D 0D 81 00 00 00 01 00 00 00 00 00 00 07 C1 C1 00 00 02 00 00 00 00 00 16 00 10 20 06 37 0F 00 00 CA
```

Auffällig:

```text
Modebyte vermutlich = 01
Checksumme = CA
```

### 13.3 Solar Hand

```text
0A 53 1D 0B 11 53 45 54 03 01 00 00 00 00 0F 5B 46 40 00 36 B0 A0 00 03 02 A0 02 62 00 02 BC 01 F4 00 32 00 32 00 30 0B 00 00 00 00 00 00 08 2D 0D 81 00 00 00 01 00 00 00 00 00 00 07 C1 C1 00 00 02 00 00 00 00 00 16 00 10 20 06 37 0F 00 00 C8
```

Auffällig:

```text
Modebyte vermutlich = 03
Checksumme = C8
```

### 13.4 Unterschied der SET-Telegramme

| Betriebsart | Byte an Position 8 | Checksumme |
|---|---:|---:|
| Auto | 00 | CB |
| Aus | 01 | CA |
| Hand | 03 | C8 |

---

## 14. ESPHome Beispiel: Lesen FC24

Kernlogik:

```cpp
if (!(bytes[0] == 0xFC && bytes[1] == 0x24 && bytes[2] == 0x0B && bytes[3] == 0x01)) {
  return;
}

float tsa = read_i16(4) / 10.0;
float tse = read_i16(6) / 10.0;
float twu = read_i16(8) / 10.0;
float tw2 = read_i16(10) / 10.0;

uint8_t pso = bytes[12];
uint8_t ulv = bytes[13];
uint8_t status_solar = bytes[14];
uint8_t stoercode = bytes[15];

uint32_t tagesenergie = read_u32(24);
uint32_t gesamtenergie = read_u32(28);
```

---

## 15. ESPHome Beispiel: berechnete Leistung

```cpp
float delta_t = tsa - tse;
float solar_leistung = 0.0;

if (pso > 0 && delta_t > 0.0) {
  solar_leistung = 2.5 * delta_t * 69.78;
}
```

---

## 16. ESPHome Beispiel: Schreibbutton

Beispiel für Solar Auto:

```yaml
button:
  - platform: template
    name: "Solar Auto setzen"
    icon: "mdi:white-balance-sunny"
    on_press:
      - uart.write:
          id: uart_bus
          data: [0x0A, 0x53, 0x1D, 0x0B, 0x11, 0x53, 0x45, 0x54, 0x00, 0x01, 0x00]
```

Hinweis: Im produktiven Code muss das komplette Telegramm inklusive Checksumme gesendet werden.
Die Kurzform oben zeigt nur das Prinzip.

---

## 17. Home Assistant Entities

Bekannte/gewünschte Entities:

| Name | Typ | Einheit |
|---|---|---|
| Kollektor Temp | Sensor | °C |
| Eintritt Temp | Sensor | °C |
| Speicher Unten | Sensor | °C |
| Speicher 2 Temp | Sensor | °C |
| Solarpumpe PSO | Sensor | % |
| Umlenkventil ULV | Sensor | - |
| Status Solar Code | Sensor | - |
| Stoercode Solar | Sensor | - |
| Frostschutz | Sensor | - |
| Ctr | Sensor | - |
| Tagesertrag | Sensor | kWh |
| Gesamtertrag | Sensor | kWh |
| Solar Leistung | Sensor | W |
| Status Solar | Text Sensor | Text |
| Regler Zeit | Text Sensor | Text |
| SystaBus Raw | Text Sensor | Hex |

### Dashboard-Fehler

Wenn im Dashboard steht:

```text
Entity not available: sensor.solar_15d031_solar_leistung
```

dann fehlt der Sensor `Solar Leistung` in ESPHome oder die Entity-ID hat sich geändert.

---

## 18. ESP8266 Crash-Analyse

Beobachteter Fehler:

```text
Hardware WDT - Level1Int
wDev_ProcessFiq
ESP8266SoftwareSerial::gpio_intr
```

Interpretation:

- Watchdog Reset
- Interruptlast im ESP8266
- SoftwareSerial auf GPIO13
- WLAN/API gleichzeitig aktiv
- zu viel Logging oder zu häufiges Publishen kann das verschlimmern

### Gegenmaßnahmen

- `logger.level: WARN`
- `baud_rate: 0`
- Raw-Frame nicht dauerhaft publishen
- Sensorwerte nur bei Änderung publishen
- Frostschutz/Ctr ebenfalls nur bei Änderung publishen
- ESP32-S3 verwenden

---

## 19. Empfohlene ESPHome-Stabilisierung

```yaml
logger:
  level: WARN
  baud_rate: 0

api:
  reboot_timeout: 0s

wifi:
  reboot_timeout: 5min
```

In der Lambda:

- letzte Werte merken
- nur bei Änderung veröffentlichen
- Raw-Frame optional deaktivieren
- keine großen `ESP_LOGI` Ausgaben im Dauerbetrieb

---

## 20. Offene Punkte

Noch nicht vollständig geklärt:

- Bedeutung von Offset 22-23
- Bedeutung von Offset 32-37
- ob Volumenstrom irgendwo im Bus übertragen wird
- ob Betriebsart Auto/Aus/Hand separat in einem anderen Frame sichtbar ist
- vollständige Display-Text-Telegramme
- sicherer minimaler SET-Befehl statt großem Parameterblock
- saubere ESP32-S3 Pinbelegung für den Adapter

---

## 21. To-do für das GitHub-Projekt

- [ ] ESPHome Beispiel für ESP8266 hinzufügen
- [ ] ESPHome Beispiel für ESP32-S3 hinzufügen
- [ ] Adapter-Pinout grafisch dokumentieren
- [ ] FC24 Decoder als Tabelle pflegen
- [ ] SET-Kommandos nur mit Warnhinweis dokumentieren
- [ ] Home Assistant Dashboard Beispiel hinzufügen
- [ ] Testlogs sammeln: Auto lädt / Auto steht / Hand / Aus / Test
- [ ] Prüfen, ob ULV jemals 1 wird
- [ ] Prüfen, ob Status 3 bei Automatikladung immer gilt

---

## 22. Haftungsausschluss

Diese Dokumentation ist ein Reverse-Engineering-Arbeitsstand.

Schreibtelegramme können Reglerparameter verändern. Nutzung auf eigene Gefahr.
Vor Einsatz an produktiven Heizungs-/Solaranlagen Sicherung, Dokumentation und Rückstellmöglichkeit prüfen.
