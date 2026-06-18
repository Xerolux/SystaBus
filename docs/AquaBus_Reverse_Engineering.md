# AquaBus / SystaSolar Aqua Reverse Engineering

Community-Dokumentation für Paradigma SystaSolar Aqua am SystaBus. Arbeitsstand aus praktischen Messungen, Display-Vergleichen, ESPHome-Logs und SET-Telegrammen.

## Kurzstatus

### Bestätigt

- `FC 24 0B 01` ist der zentrale Monitorframe der getesteten SystaSolar Aqua.
- Temperaturen sind dekodiert: TSA/Kollektor, TSE/Eintritt, TWU/Speicher unten, TW2/Speicher 2 beziehungsweise fehlender Sensor.
- `frame[12]` ist PSO, also Pumpenansteuerung in Prozent.
- `frame[14]` ist Status Solar.
- Status `3` bedeutet: solare Wärme einspeisen.
- Status `6` bedeutet: Betriebsart Hand/Test/Aus. Aus, Test und Hand wurden praktisch getestet und liefern alle Status 6.
- Status `8` bedeutet: Kollektortemperatur zu niedrig / abgeschaltet.
- `frame[24..27]` ist Tagesertrag in kWh.
- `frame[28..31]` ist Gesamtertrag in kWh.
- Checksumme ist `0x00 - Summe(alle Bytes ohne Checksumme)`.
- Eine aktuelle Solarleistung wird im FC24-Frame nicht direkt übertragen und wird aus Volumenstrom und ΔT berechnet.

### Offen

- Bedeutung von `frame[22..23]`.
- Bedeutung von `frame[32..37]`.
- Ob der Volumenstrom irgendwo im Bus übertragen wird oder nur als Reglerparameter existiert.
- Minimaler sicherer SET-Befehl statt großem Parameterblock.
- Saubere ESP32-S3 Pinbelegung auf der vorhandenen Adapterplatine final testen.
- Verhalten von ULV, falls ein Umlenkventil aktiv ist.

---

## 1. Getestete Hardware

### Steuerung

- Paradigma SystaSolar Aqua
- Display-Firmware im Test: `V3.00 060613`
- Bus-Anschluss am Regler: `BUS + / BUS -`

### Adapter

Der verwendete Adapter wird hier neutral **AquaBus Adapter** genannt.

Beobachteter Aufbau:

- Bus-Klemme `BUS+ / BUS-`
- D1-Mini-kompatibler Steckplatz
- Datenleitung auf GPIO13
- Versorgung über USB/Adapterplatine

### ESP8266

Getestet:

- ESP8266EX D1 Mini kompatibles Board
- CH340C USB-Seriell
- USB-C
- D1-Mini-Pinout

### ESP32 Empfehlung

Empfohlen für Weiterentwicklung:

- LOLIN ESP32-S3 Mini

Gründe:

- mehr RAM
- bessere UART-Verarbeitung
- ESPHome-Unterstützung
- weniger Watchdog-Probleme als ESP8266 + SoftwareSerial
- D1-Mini-ähnliche Bauform

Wichtig: Nicht blind 1:1 stecken. Pinout prüfen, insbesondere die Bus-Datenleitung.

---

## 2. Busparameter

| Parameter | Wert |
|---|---|
| Baudrate | 9600 |
| Datenbits | 8 |
| Parität | NONE |
| Stopbits | 1 |
| Richtung | RX zum Lesen, TX für SET-Kommandos |
| Frame-Erkennung | Startbytes / Header |

ESPHome UART auf ESP8266 im Test:

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

---

## 3. Checksumme

Checksumme über kompletten Frame außer letztem Byte:

```cpp
uint8_t sum = 0;
for (size_t i = 0; i < frame.size() - 1; i++) {
  sum += frame[i];
}
bool ok = frame.back() == (uint8_t)(0 - sum);
```

Kurz:

```text
checksum = 0x00 - sum(frame_without_checksum)
```

---

## 4. FC24 Monitorframe

### Beispiel: Hand/Aus/Test-Gruppe

```text
FC 24 0B 01 01 F7 01 C0 02 21 FE E0 00 00 06 00 00 00 17 06 17 06 00 00 00 00 00 0A 00 00 1F 49 29 00 80 00 11 00 AE
```

Dekodiert:

```text
TSA     50.3 °C
TSE     44.8 °C
TWU     54.5 °C
TW2    -28.8 °C / Sensor fehlt
PSO      0 %
ULV      0
Status   6
Störung  0
Zeit    17:06 17.06
Tag     10 kWh
Gesamt  8009 kWh
```

### Beispiel: Automatik, solare Wärme einspeisen, PSO 50 %

```text
FC 24 0B 01 02 04 01 C8 01 D3 FE E0 32 00 03 00 00 00 07 28 18 06 00 00 00 00 00 00 00 00 1F 49 29 00 00 00 00 00 40
```

Dekodiert:

```text
TSA     51.6 °C
TSE     45.6 °C
TWU     46.7 °C
TW2    -28.8 °C / Sensor fehlt
PSO     50 %
ULV      0
Status   3 = Solare Wärme einspeisen
Störung  0
Zeit    07:28 18.06
Tag      0 kWh
Gesamt  8009 kWh
```

### Beispiel: Automatik, solare Wärme einspeisen, PSO 100 %

```text
FC 24 0B 01 02 8A 02 31 02 40 FE E0 64 00 03 00 00 00 10 05 18 06 00 00 00 00 00 03 00 00 1F 4C 29 00 00 00 00 00 C4
```

Dekodiert:

```text
TSA     65.0 °C
TSE     56.1 °C
TWU     57.6 °C
TW2    -28.8 °C / Sensor fehlt
PSO    100 %
ULV      0
Status   3 = Solare Wärme einspeisen
Störung  0
Zeit    10:05 18.06
Tag      3 kWh
Gesamt  8012 kWh
```

Späterer Frame:

```text
... 00 00 00 03 00 00 1F 4D ...
```

zeigt:

```text
Tagesertrag = 3 kWh
Gesamtertrag = 8013 kWh
```

Damit sind Tages- und Gesamtertrag bestätigt.

---

## 5. FC24 Offsets

Offset ist nullbasiert, also `frame[0] = 0xFC`.

| Offset | Länge | Beispiel | Bedeutung | Status |
|---:|---:|---|---|---|
| 0 | 1 | FC | Start / Telegrammtyp | bestätigt |
| 1 | 1 | 24 | Länge / Kennung | bestätigt |
| 2 | 1 | 0B | Kommando | bestätigt |
| 3 | 1 | 01 | Subkommando | bestätigt |
| 4-5 | 2 | 02 8A | TSA Kollektor, signed int16 / 10 | bestätigt |
| 6-7 | 2 | 02 31 | TSE Eintritt/Rücklauf, signed int16 / 10 | bestätigt |
| 8-9 | 2 | 02 40 | TWU Speicher unten, signed int16 / 10 | bestätigt |
| 10-11 | 2 | FE E0 | TW2, signed int16 / 10 | bestätigt, bei Testanlage nicht vorhanden |
| 12 | 1 | 00 / 32 / 64 | PSO Solarpumpe in % | bestätigt |
| 13 | 1 | 00 | ULV Umlenkventil | plausibel |
| 14 | 1 | 03 / 06 / 08 | Status Solar | bestätigt |
| 15 | 1 | 00 | Störcode | bestätigt |
| 16 | 1 | 00 | Frostschutz | plausibel |
| 17 | 1 | 00 | Ctr / Counter / Diagnose | offen |
| 18 | 1 | 10 | Stunde, BCD/hex-dezimal | bestätigt |
| 19 | 1 | 05 | Minute, BCD/hex-dezimal | bestätigt |
| 20 | 1 | 18 | Tag, BCD/hex-dezimal | bestätigt |
| 21 | 1 | 06 | Monat, BCD/hex-dezimal | bestätigt |
| 22-23 | 2 | 00 00 | Zähler/Reserve, evtl. Fehlzirkulation | offen |
| 24-27 | 4 | 00 00 00 03 | Tagesenergie in kWh | bestätigt |
| 28-31 | 4 | 00 00 1F 4D | Gesamtenergie in kWh | bestätigt |
| 32 | 1 | 29 | unbekannt | offen |
| 33 | 1 | 00 | unbekannt / Jahr? | offen |
| 34 | 1 | 00 / 80 | Taste/Flags? | offen |
| 35 | 1 | 00 | Diagnose Korr? | offen |
| 36 | 1 | 00 / 11 | Diagnose Merkmale 1? | offen |
| 37 | 1 | 00 | Diagnose Merkmale 2? | offen |
| 38 | 1 | C4 | Checksumme | bestätigt |

---

## 6. Temperaturen

Temperaturen sind Big Endian signed int16, Faktor 0,1.

```cpp
float tsa = read_i16(4) / 10.0;
float tse = read_i16(6) / 10.0;
float twu = read_i16(8) / 10.0;
float tw2 = read_i16(10) / 10.0;
```

Beispiel:

```text
02 8A = 650 = 65.0 °C
FE E0 = -288 = -28.8 °C
```

`FE E0` wird bei der Testanlage als nicht vorhandener TW2-Sensor interpretiert.

---

## 7. Pumpenansteuerung PSO

`frame[12]`

| Hex | Dezimal | Bedeutung |
|---|---:|---|
| 00 | 0 | Pumpe aus |
| 32 | 50 | Pumpe 50 % |
| 64 | 100 | Pumpe 100 % |

Bestätigt durch Displayvergleich:

- Pumpe Solar 0 % -> `frame[12] = 00`
- Hand Pumpe 100 % -> `frame[12] = 64`
- Automatikladung mit Mindestdrehzahl 50 % -> `frame[12] = 32`

Die Pumpe ist PWM- oder frequenzgesteuert. PSO ist der Stellwert in Prozent.

---

## 8. Statuscodes

`frame[14]`

| Code | Bedeutung | Status |
|---:|---|---|
| 0 | Maximale Speichertemperatur erreicht | Mapping |
| 1 | Stillstand, Dampf im Kollektor | Mapping |
| 2 | Frostschutzfunktion aktiv | Mapping |
| 3 | Solare Wärme einspeisen | bestätigt |
| 4 | Anschiebefunktion aktiv | Mapping |
| 5 | Einschaltverzögerung | Mapping |
| 6 | Betriebsart Hand/Test/Aus | bestätigt |
| 7 | Störabschaltung | Mapping |
| 8 | Kollektortemperatur zu niedrig / abgeschaltet | bestätigt |

### Beobachtungen

Status 8:

```text
TSA < TWU
PSO = 0 %
Display: Abgeschaltet
```

Status 6:

```text
Betriebsart Aus  -> Status 6
Betriebsart Test -> Status 6
Betriebsart Hand -> Status 6
```

Status 3:

```text
Auto, Pumpe läuft, solare Wärme wird eingespeist -> Status 3
```

---

## 9. Erträge

### Tagesertrag

`frame[24..27]`, Big Endian uint32, Einheit kWh.

```cpp
uint32_t tagesenergie = read_u32(24);
```

Beispiel:

```text
00 00 00 03 = 3 kWh
```

### Gesamtertrag

`frame[28..31]`, Big Endian uint32, Einheit kWh.

```cpp
uint32_t gesamtenergie = read_u32(28);
```

Beispiele:

```text
00 00 1F 49 = 8009 kWh
00 00 1F 4C = 8012 kWh
00 00 1F 4D = 8013 kWh
```

---

## 10. Solarleistung

Der FC24-Frame enthält nach aktuellem Stand **keinen direkten Leistungswert in Watt**.

Berechnung:

```text
P [W] = Volumenstrom [l/min] × ΔT [K] × 69,78
```

mit:

```text
ΔT = TSA - TSE
```

Bei gemessenem/eingestelltem Volumenstrom:

```text
Volumenstrom = 2,5 l/min
```

ergibt sich:

```text
P [W] = 2,5 × (TSA - TSE) × 69,78
```

Nur berechnen, wenn:

```text
PSO > 0
ΔT > 0
```

Sonst:

```text
P = 0 W
```

Beobachtete berechnete Werte:

- ca. 1047 W bei PSO 50 %
- ca. 1553 W bei PSO 100 %

---

## 11. Display-Parameter

Aus dem Regler-Menü abgelesen:

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

## 12. SET-Kommandos / Schreibbefehle

Es wurden SET-Sequenzen für Auto, Aus und Hand gefunden.

> Warnung: Die Sequenzen scheinen einen größeren Parameterblock zu übertragen. Nicht unkritisch produktiv verwenden, bevor klar ist, welche Parameter mitgeschrieben werden.

### 12.1 Gemeinsamer Header

```text
0A 53 1D 0B 11 53 45 54
```

`53 45 54` entspricht ASCII `SET`.

### 12.2 Solar Auto

```text
0A 53 1D 0B 11 53 45 54 00 01 00 00 00 00 0F 5B 46 40 00 36 B0 A0 00 03 02 A0 02 62 00 02 BC 01 F4 00 32 00 32 00 30 0B 00 00 00 00 00 00 08 2D 0D 81 00 00 00 01 00 00 00 00 00 00 07 C1 C1 00 00 02 00 00 00 00 00 16 00 10 20 06 37 0F 00 00 CB
```

### 12.3 Solar Aus

```text
0A 53 1D 0B 11 53 45 54 01 01 00 00 00 00 0F 5B 46 40 00 36 B0 A0 00 03 02 A0 02 62 00 02 BC 01 F4 00 32 00 32 00 30 0B 00 00 00 00 00 00 08 2D 0D 81 00 00 00 01 00 00 00 00 00 00 07 C1 C1 00 00 02 00 00 00 00 00 16 00 10 20 06 37 0F 00 00 CA
```

### 12.4 Solar Hand

```text
0A 53 1D 0B 11 53 45 54 03 01 00 00 00 00 0F 5B 46 40 00 36 B0 A0 00 03 02 A0 02 62 00 02 BC 01 F4 00 32 00 32 00 30 0B 00 00 00 00 00 00 08 2D 0D 81 00 00 00 01 00 00 00 00 00 00 07 C1 C1 00 00 02 00 00 00 00 00 16 00 10 20 06 37 0F 00 00 C8
```

### 12.5 Unterschiede

| Betriebsart | Byte nach `SET` | Checksumme |
|---|---:|---:|
| Auto | 00 | CB |
| Aus | 01 | CA |
| Hand | 03 | C8 |

---

## 13. ESPHome Decoder-Kern

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

## 14. Home Assistant Entities

Empfohlene Entities:

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

---

## 15. ESP8266 Watchdog / Crash

Beobachtet:

```text
Hardware WDT - Level1Int
wDev_ProcessFiq
ESP8266SoftwareSerial::gpio_intr
```

Interpretation:

- ESP8266 wird im Zusammenspiel von WLAN, API und SoftwareSerial-Interrupts überlastet.
- Der Decoder selbst ist nicht offensichtlich falsch.
- Weniger Logging reduziert Last, behebt aber nicht zwingend die Ursache.

Gegenmaßnahmen:

- `logger.level: WARN`
- `baud_rate: 0`
- Sensorwerte nur bei Änderung publishen
- Raw-Frame optional deaktivieren
- ESP32-S3 mit Hardware-UART verwenden

---

## 16. Offene Punkte

Aktuell offen:

1. Bedeutung von `frame[22..23]`.
2. Bedeutung von `frame[32..37]`.
3. Ob Volumenstrom irgendwo im Busframe übertragen wird.
4. Sicherer minimaler SET-Befehl statt großem Parameterblock.
5. ESP32-S3 Pinbelegung auf Adapter final testen.
6. Verhalten von ULV mit real aktivem Umlenkventil.
7. Langzeittest ohne Raw-Frame und mit ESP32-S3.
8. Optional: eigener ESPHome-External-Component Decoder statt `uart.debug`-Lambda.

---

## 17. Haftungsausschluss

Diese Dokumentation ist ein Reverse-Engineering-Arbeitsstand.

Schreibtelegramme können Reglerparameter verändern. Nutzung auf eigene Gefahr. Vor Einsatz an produktiven Heizungs- oder Solaranlagen Sicherung, Dokumentation und Rückstellmöglichkeit prüfen.
