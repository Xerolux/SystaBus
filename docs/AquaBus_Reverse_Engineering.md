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

- Bedeutung von `frame[22..23]` (bleibt unter Status 3/PSO 100% konstant `00 00`, Auslöser unbekannt).
- Bedeutung von `frame[32..37]` (Byte32 `0x29` ist vermutlich statische Geräte-/Firmware-Kennung, Bytes33-37 bleiben auch unter Solarbetrieb `00 00`).
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
| 22-23 | 2 | 00 00 | Zähler/Reserve oder Fehlzirkulation | analyse |
| 24-27 | 4 | 00 00 00 03 | Tagesenergie in kWh | bestätigt |
| 28-31 | 4 | 00 00 1F 4D | Gesamtenergie in kWh | bestätigt |
| 32 | 1 | 29 | Firmware-/Hardware-Version | analyse |
| 33 | 1 | 00 | Firmware Sub-Version / Variante | analyse |
| 34 | 1 | 00 / 80 | Tastenflags / Bedienfeld-Status | analyse |
| 35 | 1 | 00 | Diagnose-Status 1 | analyse |
| 36 | 1 | 00 / 11 | Diagnose-Status 2 / Fehlerspeicher | analyse |
| 37 | 1 | 00 | Fehlerspeicher oder Reserve | analyse |
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

## 16. Telemetrie-Analyse: Bytes 22-23 und 32-37

Analyse von 35 aufeinanderfolgenden FC24-Frames (2026-06-25, Status 8, PSO 0%):

### Byte22–23: Reserve / dynamischer Zähler

**Befund**: Über alle 35 Frames konsistent `00 00`.

**Interpretation**:
- Derzeit keine Aktivität unter Status 8 (Kollektortemperatur zu niedrig)
- Wahrscheinlich: Reserve oder Zähler für andere Betriebsmodi
- Hypothese: Aktiviert unter Status 3 (solare Wärme einspeisen) oder bei Fehlerfall

**Mögliche Bedeutungen**:
1. Fehlzirkulationserkennung (Status inaktiv)
2. Schaltzähler der Heizleitung / ULV
3. Reserve für zukünftige Erweiterungen
4. Status-Reserve für andere Firmware-Versionen

**Aktivierungsbedingung nicht bekannt.** Langzeit-Messung unter Status 3 empfohlen.

### Bytes 32–37: Geräte-ID und Status

**Befund**: Über alle 35 Frames konsistent `29 00 00 00 00 00`.

#### Byte 32: Firmware-/Hardware-Version

- Dezimal: **41** (konstant)
- Hexadezimal: **0x29** (konstant)
- Konsistent über alle Messungen → **Geräte-Identifikation**

**Hypothese**: 
```
Byte 32 = Firmware-Hauptversion (0x29 ≈ v41 oder v3.x)
Byte 33 = Firmware-Subversion oder Variante
```

Vergleich mit bekannter Firmware "V3.00 060613" → Byte32 könnte Komponenten-Version sein.

#### Bytes 33–37: Status und Diagnose

In der Messserie alle `00 00 00 00 00` → Status inaktiv unter Status 8.

**Hypothese**:
```
Byte 33: Sub-Version oder Konfigurationsmerkmale
Byte 34: Tastenflags / Bedienfeld-Status (00 = alle aus)
Byte 35: Diagnose-Status 1
Byte 36: Diagnose-Status 2 / Fehlerspeicher Bit
Byte 37: Fehlerspeicher oder Reserve
```

**Erwartet unter anderen Bedingungen**:
- Status 3 (solare Wärme einspeisen) → Byte34-37 ändern sich
- PSO > 0% → Byte35 oder Byte36 aktiviert
- ULV aktiv → Byte36 oder Byte37 setzen sich
- Fehler / Störcode != 0 → Byte36-37 ändern sich

### Messung unter Status 3, PSO 100% (2026-06-27)

Live-Messung über ca. 90 Sekunden mit aktivem Solarbetrieb (Status 3, PSO 100 %, TSA steigt von 64.0 °C auf 64.5 °C, TSE von 54.6 °C auf 55.6 °C):

```text
FC 24 0B 01 02 80 02 22 02 74 FE E0 64 00 03 00 00 00 13 23 27 06 00 00 00 00 00 09 00 00 1F D1 29 00 00 00 00 00 EE
...
FC 24 0B 01 02 85 02 2C 02 75 FE E0 64 00 03 00 00 00 13 24 27 06 00 00 00 00 00 09 00 00 1F D1 29 00 00 00 00 00 DD
```

**Ergebnis**: Byte22-23 bleibt durchgehend `00 00`, Bytes32-37 bleiben durchgehend `29 00 00 00 00 00`. Auch bei aktivem Solarbetrieb mit voller Pumpenleistung (PSO 100 %) und kontinuierlich steigenden Temperaturen ändern sich diese Bytes nicht.

**Schlussfolgerung**: Die Hypothese "Aktivierung unter Status 3" ist widerlegt. Byte22-23 und Bytes32-37 hängen nicht direkt an Status 3 oder PSO. Bytes32-37 sind damit eher als statische Geräte-/Firmware-Kennung zu verstehen (Byte32 `0x29` konstant über alle bisherigen Messungen, unabhängig vom Betriebszustand). Für Byte22-23 bleiben als Auslöser nur noch ULV-Aktivierung oder ein realer Störfall (Störcode != 0) übrig.

### Langzeitmessung (2026-06-27 bis 2026-06-28, ca. 19 Stunden)

Auswertung von 2480 erfassten `raw_frame_solar_ulv_stoerung`-Frames (Home-Assistant-Historie, Capture-Bedingung weiterhin `(status_solar == 3 && pso > 0) || ulv != 0 || stoercode != 0`) über einen Zeitraum von ca. 19 Stunden, mit stark wechselnden TSA/TSE/PSO-Werten (PSO u. a. zwischen 0x32 und 0x85 beobachtet):

- Byte22-23: über alle 2480 Frames durchgehend `00 00`
- Bytes32-37: über alle 2480 Frames durchgehend `29 00 00 00 00 00`

**Schlussfolgerung**: Auch über einen Langzeitraum mit vielfach wechselnden Betriebszuständen (Solarbetrieb mit unterschiedlicher Pumpenleistung) bleiben beide Bytebereiche absolut konstant. Damit ist endgültig bestätigt, dass Byte22-23 und Bytes32-37 nicht an PSO, TSA/TSE oder den normalen Solarbetrieb (Status 3) gekoppelt sind. In diesem Beobachtungszeitraum trat weder eine aktive ULV-Schaltung noch ein Störfall (Störcode != 0) auf – die einzigen verbleibenden Hypothesen (ULV-Aktivierung, realer Störfall) konnten somit noch nicht verifiziert werden, da die entsprechenden Trigger-Bedingungen während der Messung nicht eintraten.

### Empfohlene Dekodierungs-Schritte

1. ~~Messungen unter Status 3 durchführen~~ – erledigt, keine Änderung beobachtet (auch über 19h Langzeitmessung bestätigt)
2. **ULV-Verhalten** beobachten, falls Umlenkventil aktiv
3. **Fehlerfall-Dokumentation** sammeln (Störcode != 0)
4. **Hardware-Vergleich** durchführen (mehrere Geräte mit unterschiedlicher Firmware)
5. **Bit-Level-Analyse** bei erweiterten Datenmengen

### C++ Pseudo-Code für zukünftige Verarbeitung

```cpp
// Geräte-Identifikation
uint8_t fw_version = frame[32];      // z.B. 0x29
uint8_t fw_revision = frame[33];     // z.B. 0x00

// Status und Diagnose (dynamisch)
uint8_t control_bits = frame[34];    // Tastenflags / Bedienfeld
uint8_t diag_status_1 = frame[35];   // Diagnose 1
uint8_t diag_status_2 = frame[36];   // Diagnose 2 / Fehler
uint8_t diag_status_3 = frame[37];   // Reserve / Fehler

// Reserve / Zähler
uint16_t counter_or_reserve = ((uint16_t)frame[22] << 8) | frame[23];
```

---

## 17. Offene Punkte

Aktuell offen:

1. ~~Byte22–23 unter Status 3~~: widerlegt, bleibt unter Status 3/PSO 100% konstant `00 00` (siehe Messung 2026-06-27)
2. ~~Bytes32–37 unter Status 3~~: widerlegt, bleiben unter Status 3/PSO 100% konstant `29 00 00 00 00 00`
3. **ULV-Verhalten**: Auswirkung auf Byte22-23 oder Bytes34-37, noch nicht getestet (ULV bisher immer `0`)
4. **Fehlerspeicher**: Bytes36-37 unter Störcode != 0, noch nicht getestet
5. Ob Volumenstrom irgendwo im Busframe übertragen wird.
6. Sicherer minimaler SET-Befehl statt großem Parameterblock.
7. ESP32-S3 Pinbelegung auf Adapter final testen.
8. Langzeittest ohne Raw-Frame und mit ESP32-S3.
9. Optional: eigener ESPHome-External-Component Decoder statt `uart.debug`-Lambda.

---

## 18. Haftungsausschluss

Diese Dokumentation ist ein Reverse-Engineering-Arbeitsstand.

Schreibtelegramme können Reglerparameter verändern. Nutzung auf eigene Gefahr. Vor Einsatz an produktiven Heizungs- oder Solaranlagen Sicherung, Dokumentation und Rückstellmöglichkeit prüfen.
