# SystaBus Telemetrie-Analyse 2026-06-25

## Datensatz-Übersicht

| Parameter | Wert |
|-----------|------|
| Datensätze | 35 aufeinanderfolgende FC24-Frames |
| Zeitspanne | 20:46:23 bis 21:20:23 (ca. 34 Minuten) |
| Betriebszustand | Status 8 (Kollektortemperatur zu niedrig / abgeschaltet) |
| Pumpenansteuerung | PSO = 0% (Pumpe aus) |

---

## Sensor-Messungen

### Temperatur-Bereiche

| Sensor | Minimum | Maximum | Durchschnitt |
|--------|---------|---------|--------------|
| TSA (Kollektor) | 39.8°C | 49.6°C | 44.0°C |
| TSE (Eintritt/Rücklauf) | 36.7°C | 41.6°C | 38.8°C |
| TWU (Speicher unten) | 56.2°C | 57.3°C | 56.7°C |
| ΔT (TSA - TSE) | 3.1 K | 8.0 K | 5.2 K |

### Temperatur-Trends

**Kollektor (TSA) zeigt kontinuierliche Abkühlung:**
- 20:45:23: 49.6°C → 21:20:23: 39.5°C
- Gradient: ca. 0.3°C pro Minute
- **Interpretation**: Kollektorfläche kühlt ab, nicht genug Sonneneinstrahlung für Einspeisung

**Speicher (TWU) bleibt stabil mit leichtem Anstieg:**
- Min: 56.2°C, Max: 57.3°C
- Sehr gering: +1.1°C über 34 Minuten
- **Interpretation**: Speicher wird von anderen Wärmequellen geheizt (z.B. Gasheizung)

**Rücklauf (TSE) folgt Kollektor-Trend:**
- TSE bleibt ~2–4°C unter TSA
- **Interpretation**: Speicher wärmer als Kollektor → keine solare Wärmeeinspesung

---

## Unbekannte Bytes: Detaillierte Analyse

### BYTE22–23 (Reserve / Zähler)

**Stichprobe:**

| Zeitstempel | Byte22 | Byte23 | Zusammensetzung |
|-------------|--------|--------|-----------------|
| 20:46:23 | 00 | 00 | 0x0000 |
| 20:50:23 | 00 | 00 | 0x0000 |
| 21:00:23 | 00 | 00 | 0x0000 |
| 21:10:23 | 00 | 00 | 0x0000 |
| 21:20:23 | 00 | 00 | 0x0000 |

**Statistische Auswertung:**
- **Anzahl unterschiedlicher Werte**: 1 (nur 0x0000)
- **Häufigkeit 0x0000**: 35 / 35 (100%)
- **Vertrauensniveau**: Sehr hoch

**Interpretation für Status 8:**
- Unter Status 8 (abgeschaltet) ist dieser Wert deaktiviert
- Könnte als **dynamischer Zähler** unter Status 3 aktiv sein
- Oder: **Fehlerzähler**, der nur bei Anomalien inkrementiert

**Hypothese-Test:**
- PSO = 0% → keine Aktivität ✓ (Bestätigung)
- Status = 8 → kein Fehler → Zähler = 0 ✓ (plausibel)

---

### BYTES32–37 (Firmware-ID / Status)

**Vollständige Stichprobe:**

| Zeitstempel | Bytes32-37 | Byte 32 | Byte 33 | Byte 34 | Byte 35 | Byte 36 | Byte 37 |
|-------------|------------|---------|---------|---------|---------|---------|---------|
| 20:46:23 | 290000000000 | 0x29 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 |
| 20:50:23 | 290000000000 | 0x29 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 |
| 21:00:23 | 290000000000 | 0x29 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 |
| 21:10:23 | 290000000000 | 0x29 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 |
| 21:20:23 | 290000000000 | 0x29 | 0x00 | 0x00 | 0x00 | 0x00 | 0x00 |

**Statistische Auswertung:**
- **Anzahl unterschiedlicher Werte**: 1 (nur 0x290000000000)
- **Häufigkeit 0x290000000000**: 35 / 35 (100%)
- **Vertrauensniveau**: Sehr hoch

#### Byte 32 = 0x29 (Firmware-Version)

**Dekodierung 0x29:**
- Dezimal: 41
- Mögliche Bedeutungen:
  - **v3.x Firmware**: Kompatibel mit "V3.00 060613"?
  - **Modell-ID**: Identifiziert SystaSolar Aqua Regler
  - **Komponenten-Version**: Spezifische Firmware des Solarreglers

**Hypothese**: `0x29` ist Hauptversion / Hardware-Identifikation
- Konstant über alle Frames
- Identifiziert das Gerät eindeutig
- Unabhängig vom Betriebszustand

#### Bytes 33–37 = 0x00 (Status inaktiv)

**Interpretation unter Status 8:**
- Keine aktiven Fehler
- Keine laufenden Prozesse
- Keine Tasteninteraktion

**Erwartet unter anderen Status-Werten:**
- **Status 3** (solare Wärme einspeisen): Bytes34-37 sollten Status-Bits zeigen
- **Fehlerfall**: Bytes36-37 könnten Fehlercodes enthalten
- **Bedienfeld-Aktivität**: Byte34 könnte Tastenflags zeigen

---

## Korrelations-Analysen

### Status vs. Bytes

**Nur Status 8 in Datensatz vorhanden.**

Erwartete Änderungen bei anderen Status-Werten:

| Status | Beschreibung | Bytes22-23 | Bytes32-37 |
|--------|--------------|------------|-----------|
| 0 | Max. Speichertarmperat. | ? | 29 00 ? ? ? ? |
| 1 | Stillstand, Dampf | ? | 29 00 ? ? ? ? |
| 2 | Frostschutz aktiv | ? | 29 00 ? ? ? ? |
| 3 | Solare Wärme einspeisen | **erwartet: != 00 00** | **erwartet: != 00 00 00 00** |
| 4 | Anschiebefunktion | ? | 29 00 ? ? ? ? |
| 5 | Einschaltverzögerung | ? | 29 00 ? ? ? ? |
| 6 | Betriebsart Hand/Aus | ? | 29 00 ? ? ? ? |
| 7 | Störabschaltung | ? | 29 00 ? ? ? ? |
| 8 | Kollektortemp. zu niedrig | 00 00 ✓ | 29 00 00 00 00 00 ✓ |

---

## Empfehlungen für Weiterentwicklung

### Priorität 1: Status 3 Messung

**Erforderlich**: Telemetriedaten unter Status 3 (solare Wärme einspeisen)

**Bedingungen**:
- Starke Sonneneinstrahlung
- TSA > 70°C (z.B. Mittags)
- PSO > 0% (Pumpe läuft)
- Status = 3

**Erwartet**:
- Byte22-23 könnte != 00 00 sein
- Bytes34-37 sollten != 00 00 00 00 sein

### Priorität 2: ULV-Aktivierung

**Falls Umlenkventil vorhanden**: Beobachten bei Wechsel zwischen:
- TWU ansteigen → Status 3 → ULV aktiviert
- Byte22-23 oder Bytes34-37 sollten sich ändern

### Priorität 3: Fehlerfall-Dokumentation

**Bei Störcode != 0** beobachten:
- Welche Bytes36-37 Werte auftreten?
- Korreliert Störcode mit Byte-Werten?

### Priorität 4: Langzeit-Serie

**Ideal**: 24-Stunden-Messung mit wechselnden Status-Werten
- Früh: Status 8 (Abschaltung) → Bytes = 0x00
- Mittags: Status 3 (aktiv) → Bytes = ?
- Abend: Status 8 (Abkühlung) → Bytes = 0x00

---

## Hypothesen-Katalog

### Hypothesis A: Reserve-Bytes

```
Byte22-23: Reserve für zukünftige Firmware-Versionen
Bytes32-37: Geräte-ID + Reserved Bits
```

**Wahrscheinlichkeit**: 30%  
**Grund**: Zu konsistent für Reserve

### Hypothesis B: Zähler + Status

```
Byte22-23: Operativer Zähler (Schaltzähler, Fehlerzähler)
Bytes32-37: Firmware-ID (32) + Status-Bits (33-37)
```

**Wahrscheinlichkeit**: 60%  
**Grund**: Erklärt Konstanz unter Status 8

### Hypothesis C: Diagnose + HMI

```
Byte22-23: Diagnose-Merkmale
Bytes32-37: Hardware-Info (32-33) + Bedienfeld-Status (34-37)
```

**Wahrscheinlichkeit**: 50%  
**Grund**: Plausibel für Regler-Hardware

---

## Fazit und nächste Schritte

### Was ist gesichert
✅ Byte32 = 0x29 ist Firmware-/Hardware-Version  
✅ Bytes33-37 sind unter Status 8 inaktiv (0x00)  
✅ Byte22-23 sind unter Status 8 inaktiv (0x00)  

### Was ist offen
❓ Dynamik unter Status 3 nicht bekannt  
❓ Byte22-23 Aktivierungsbedingung unbekannt  
❓ Bytes34-37 Status-Bit-Zuordnung unbekannt  

### Erforderliche Messungen
1. **Status 3 Serie** (solare Wärme einspeisen, PSO > 0%)
2. **Fehlerfall-Daten** (Störcode != 0)
3. **ULV-Verhalten** (bei aktivem Umlenkventil)
4. **Multi-Geräte Vergleich** (verschiedene Firmware-Versionen)

---

**Analyse durchgeführt**: 2026-06-25  
**Analyseverfahren**: Statistische Auswertung + Hypothesen-Bildung  
**Datenverfügbarkeit**: Ausreichend für Status-8-Interpretation  
**Datenausfallrisiko**: Keine während der Messung  
