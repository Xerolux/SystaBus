# SystaBus Messphase 2: Status 3 Datenerfassung

## Überblick

Nach der erfolgreichen Telemetrie-Analyse von Status 8 (abgeschaltet) mit 35 Frames vom 2026-06-25 steht die **Messphase 2** an:

**Ziel**: Daten unter **Status 3 (Solare Wärme einspeisen)** mit **PSO > 0%** sammeln, um die unbekannten Bytes22-23 und Bytes32-37 endgültig zu dekodieren.

---

## Neue Konfigurationen

### v2 YAML-Dateien (Fixes & neue Sensoren)

**Verfügbar**:
- `esp8266-systasolar-aqua-v2.yaml` — für ESP8266/D1 Mini
- `esp32-s3-systasolar-aqua-v2.yaml` — für ESP32-S3 (Hardware-UART empfohlen)

**Neue Features**:

#### 1. 255-Zeichen-Limit Fix ✅

**Problem (alte Version)**:
```
raw_frame_hex: "FC 24 0B 01 ... 38 Byte Hex ..." (lange State Attribute)
→ Home Assistant: "State is longer than 255 characters, falling back to unknown"
```

**Lösung (v2)**:
```yaml
substitutions:
  raw_frame_logging: "false"  # Default: deaktiviert
```

- Raw-Frame wird **nur bei Bedarf aktiviert** (für Debugging)
- Default: `raw_frame_logging: "false"` → State-String bleibt < 255 Zeichen
- Wenn aktiviert: Nur mit Status-Änderungen publishen (nicht kontinuierlich)

#### 2. Neue Sensoren für Messphase 2

```yaml
sensor:
  # Byte22-23: Reserve / dynamischer Zähler
  - platform: template
    name: "Byte22-23 (Counter)"
    id: byte22_23_sensor
    
  # Bytes32-37: Firmware-ID und Status-Bits
  - platform: template
    name: "Byte34 (Control Bits)"
    id: byte34_control_sensor
  - platform: template
    name: "Byte35 (Diag 1)"
    id: byte35_diag1_sensor
  - platform: template
    name: "Byte36 (Diag 2)"
    id: byte36_diag2_sensor
  - platform: template
    name: "Byte37 (Diag 3)"
    id: byte37_diag3_sensor
```

**Monitoring in Home Assistant**:
```
Byte22-23 (Counter):   00  → erwartet Änderung unter Status 3
Byte34 (Control):      00  → erwartet Änderung bei PSO > 0%
Byte35 (Diag 1):       00  → erwartet Änderung bei Wärmeeinspeising
Byte36 (Diag 2):       00  → Fehler-Bits?
Byte37 (Diag 3):       00  → Reserve oder Fehlerspeicher
```

---

## Migrations-Anleitung

### Von v1 zu v2 wechseln

1. **Backup erstellen** (optional):
   ```bash
   cp esp32-s3-systasolar-aqua.yaml esp32-s3-systasolar-aqua-backup.yaml
   ```

2. **v2 Datei verwenden**:
   ```bash
   # In ESPHome Web UI:
   # 1. Datei editieren → esp32-s3-systasolar-aqua-v2.yaml
   # 2. Or: Inhalte aus v2 in v1 kopieren
   ```

3. **Flash / OTA Update**:
   ```bash
   esphome run esp32-s3-systasolar-aqua-v2.yaml
   ```

4. **Home Assistant Entities neuladen**:
   - Settings → Devices & Services → ESPHome → Neustart
   - Neue Sensoren erscheinen in Home Assistant

---

## Datenerfassungs-Checkliste

### Voraussetzungen ✓

- [ ] ESP8266 oder ESP32-S3 mit SystaBus-Adapter verbunden
- [ ] WLAN und API konfiguriert
- [ ] ESPHome läuft mit v2 Konfiguration
- [ ] Home Assistant verbunden
- [ ] Sensoren sichtbar: `Byte22-23`, `Byte34–37`

### Messperioden

#### A) Normale Abkühlung (Status 8)
**Bedingung**: Status = 8, PSO = 0%  
**Erwartung**: 
- Byte22-23 = 0x0000 (wie in Phase 1)
- Byte34-37 = 0x00 (inaktiv)
- Baseline zur Validierung

**Dauer**: 10-15 Minuten (z.B. morgens/abends)

#### B) Warmstart → Solaraktiv (Status 3)
**Bedingung**: 
- PSO von 0% → > 0% (z.B. 50% oder 100%)
- Status ändert sich von 8 → 3
- Kollektor-Aufheizung (TSA steigt)

**Erwartung**:
- **Byte22-23**: Erneuerung / Inkrementierung?
- **Byte34**: Control-Bits aktiv (bei PSO > 0%)?
- **Byte35-37**: Änderungen bei Wärmeeinspeising?

**Dauer**: 20-30 Minuten (Transition dokumentieren!)

#### C) Solaraktiv Plateau (Status 3, PSO 50-100%)
**Bedingung**:
- Status = 3 (stabil)
- PSO 50% oder 100% (konstant)
- TSA > 60°C, delta_t > 5K

**Erwartung**:
- Byte22-23: Stabil oder zählend?
- Byte34-37: Stabil oder sich ändernde Bits?

**Dauer**: 30-60 Minuten (je länger, desto besser für Muster)

#### D) Auskühlung (Status 3 → 8)
**Bedingung**:
- PSO 100% → 0%
- Status 3 → 8
- TSA sinkt wieder

**Erwartung**:
- Byte22-23: Rückkehr zu 0x0000?
- Byte34-37: Rückkehr zu 0x00?

**Dauer**: 10-20 Minuten

---

## Datenerfassung in Home Assistant

### Automatisierung zum Loggen

**Empfohlen**: Erstelle eine Automatisierung, die Byte-Änderungen protokolliert:

```yaml
# configuration.yaml
automation:
  - alias: "SystaBus Byte22-23 Änderung"
    trigger:
      entity_id: sensor.byte22_23_counter
      platform: state
    action:
      - service: logger.write
        data:
          message: "Byte22-23 geändert → {{ trigger.to_state.state }}"
          level: WARNING

  - alias: "SystaBus Byte34-37 Änderungen"
    trigger:
      entity_id: 
        - sensor.byte34_control_bits
        - sensor.byte35_diag_1
        - sensor.byte36_diag_2
        - sensor.byte37_diag_3
      platform: state
    action:
      - service: logger.write
        data:
          message: |
            Status: {{ states('sensor.status_solar_code') }}
            PSO: {{ states('sensor.solarpumpe_pso') }}%
            Byte34: {{ states('sensor.byte34_control_bits') }}
            Byte35: {{ states('sensor.byte35_diag_1') }}
            Byte36: {{ states('sensor.byte36_diag_2') }}
            Byte37: {{ states('sensor.byte37_diag_3') }}
          level: INFO
```

### Daten exportieren

**Home Assistant UI**:
1. Gehe zu: **Developer Tools** → **States**
2. Suche nach `sensor.byte*`
3. Kopiere Historie für Analyse

**Oder: InfluxDB**:
```yaml
influxdb:
  database: homeassistant
```

---

## Erwartete Ergebnisse

### Hypothese 1: Byte22-23 sind inaktiv
```
Status 8 (Phase 1):  Byte22-23 = 0x0000 ✓ (bestätigt)
Status 3 (Phase 2):  Byte22-23 = 0x0000 (gleichbleibend)
→ Reserve / Padding
```

### Hypothese 2: Byte22-23 sind dynamischer Zähler
```
Status 8:  Byte22-23 = 0x0000 ✓
Status 3:  Byte22-23 = 0x0001 (inkrementiert)
Status 3:  Byte22-23 = 0x0002, 0x0003, ... (weiteres Inkrementieren)
→ Betriebsstunden-Zähler oder Schaltzyklus-Zähler
```

### Hypothese 3: Bytes32-37 sind Status-Bits
```
Status 8 (Phase 1):     Byte34-37 = 00 00 00 00 ✓
Status 3, PSO 0%:       Byte34-37 = ?? ?? ?? ??
Status 3, PSO 50%:      Byte34-37 = ?? ?? ?? ?? (Bit-Pattern?)
Status 3, PSO 100%:     Byte34-37 = ?? ?? ?? ?? (andere Bits?)
→ Tastenflags / Betriebsart-Bits
```

---

## Troubleshooting

### Problem: Byte-Sensoren zeigen "unknown"

**Ursache**: Rahmen wird nicht korrekt geparst  
**Lösung**:
1. Prüfe UART-Verbindung
2. Aktiviere Logging: `logger: level: DEBUG`
3. Prüfe auf Parse-Fehler im Logs

### Problem: 255-Zeichen-Fehler tritt immer noch auf

**Ursache**: `raw_frame_logging: "true"` ist noch aktiv  
**Lösung**:
```yaml
substitutions:
  raw_frame_logging: "false"  # Ändern auf false
```

### Problem: Status 3 wird nicht erreicht

**Ursache**: Zu wenig Sonneneinstrahlung oder Speicher zu heiß  
**Lösung**:
- Messungen nur mittags bei klarem Himmel durchführen
- TWU muss unter Max-Temperatur (z.B. 90°C) sein
- TSA sollte > 60°C sein für Einspeisung

---

## Analyse nach der Messphase

### Nächste Schritte:

1. **Datensätze sammeln** (Status 3 vollständige Zyklen)
2. **Bytes visualisieren** (Home Assistant History Stats)
3. **Patterns erkennen** (zeitliche Korrelationen)
4. **Hypothesen validieren** (Byte-Bedeutung prüfen)
5. **Dokumentation aktualisieren** (neue Erkenntnisse)

### Ergebnis-Format:

```markdown
# Phase 2 Ergebnisse (Datum)

## Status 3 Messungen

### Byte22-23
- Min: 0x0000, Max: 0x???? (Bereich)
- Trend: konstant / inkrementierend / zyklisch
- Korrelation: Status / PSO / Zeit

### Bytes32-37
Byte34: 0x00 → 0x?? Änderung erkannt
Byte35: 0x00 → 0x?? Änderung erkannt
Byte36: 0x00 → 0x?? Änderung erkannt
Byte37: 0x00 → 0x?? Änderung erkannt
```

---

## Zeitplan

| Phase | Bedingung | Ziel | Dauer |
|-------|-----------|------|-------|
| 1 ✅ | Status 8 | Baseline | 35 Frames |
| 2 → | Status 3 | Byte-Dekodierung | 100+ Frames |
| 3 | Fehlerfall | Error-Bits | 5-10 Frames |
| 4 | ULV aktiv | Umlenkventil | ggf. später |

---

## Kontakt & Support

Bei Fragen oder Problemen:
- Dokumentation: `/docs/AquaBus_Reverse_Engineering.md`
- Telemetrie-Analyse: `/docs/Telemetry_Analysis_20260625.md`
- ESPHome Config: `/examples/esphome/`

---

**Messphase 2 gestartet**: 2026-06-25  
**Ziel**: Dekodierung Bytes 22-23, 32-37  
**Status**: Bereit für Status 3 Erfassung 🚀
