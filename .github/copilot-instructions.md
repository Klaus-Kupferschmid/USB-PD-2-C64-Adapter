# USB-PD-2-C64-Adapter - Copilot Instructions

## Projektübersicht
KiCad-PCB-Adapter für ein modernes USB-C-PD-Netzteil zur Stromversorgung eines Commodore C64.

**Anschlüsse:**
- Eingang: USB-C Stecker (USB-PD, 20V)
- Ausgang: C64-Netzteilstecker (DIN 7-polig)

**Erforderliche Ausgangsspannungen (gemäß Original-Netzteil):**
- **5 VDC** mit **5 A** (Logik, SID, VIC-II)
- **9 VAC** mit **2 A** (CIA Timer, User Port)

## Systemarchitektur

### USB-PD Eingangsstufe
- **CH224K** USB PD Power Receiving Chip fordert 20V vom USB-C Netzteil an

### 5 VDC / 5 A Zweig (Buck Converter)
```
USB-C 20V ──► TVS ──► Input Caps ──► RT6318 Buck ──► Inductor ──► +5.1V Rail
                                                                      │
    ┌─────── Lf (1.5µH) ───┬─── Rsense (0.03Ω) ───┬─────────────────► OUT
    │                      │                       │
   Cout                   Cf (220µF)            Crowbar
    │                      │                       │
   GND                    GND                     GND

OUT ──┬──────────────────┬────────────────────────► Last (C64)
      │                  │
     SCR                 │
      │                  │
     GND                 │

Crowbar-Trigger:
OUT ──── R (1k) ────┬──── Gate SCR
                    │
                  ZD 5.6V
                    │
                   GND
```

### 9 VAC / 2 A Zweig (Inverter)
- Referenz-Design: [wagiminator ATtiny412-USB-PD-Inverter](https://github.com/wagiminator/ATtiny412-USB-PD-Inverter)
- **Kritische Anforderungen:**
  - Sauberer Sinus bei 50 Hz oder 60 Hz
  - Belastbarkeit mindestens 2 A kontinuierlich
  - THD (Total Harmonic Distortion) prüfen

## Projektstruktur
```
KiCad/
├── USB-PD-2-C64-Adapter.kicad_pro    # KiCad 10.0 Projekt
├── USB-PD-2-C64-Adapter.kicad_sch    # Hauptschaltplan
├── USB-PD-2-C64-Adapter.kicad_pcb    # PCB Layout
├── USB-PD-2-C64-Adapter.kicad_sym    # Projektsymbole (Custom Library)
├── Production/                        # Fertigungsdaten
│   └── USB-PD-2-C64-Adapter_BOM.csv  # JLCPCB-konforme BOM
└── Datasheet/                         # Datenblätter
```

## Stückliste (BOM) - 5V Zweig

| Nr. | Komponente | Wert/Typ | Package | LCSC-Nr. | Menge |
|-----|------------|----------|---------|----------|-------|
| 1 | RT6318CGQUF | Buck Converter | QFN | C6288609 | 1 |
| 2 | Induktor | 1µH 10A | - | C15949 | 1 |
| 3 | Input MLCC | 10µF 25V X7R | 0805 | C440198 | 1 |
| 4 | Bulk Input Cap | 47µF 25V | - | C178882 | 1 |
| 5 | Output MLCC | 22µF 10V X7R | 0805 | C45783 | 4 |
| 6 | Output Polymer | 150µF 10V | - | C12530 | 1 |
| 7 | Post LC Filter Induktor | 1.5µH | - | C15949 | 1 |
| 8 | Post Filter Cap | 220µF | - | C12530 | 1 |
| 9 | Shunt Resistor | 0.03Ω | 1206 | C25110 | 1 |
| 10 | SCR | BT151 equiv. | - | C2563 | 1 |
| 11 | Z-Diode | 5.6V BZT52C5V6 | - | C31673 | 1 |
| 12 | Gate Resistor | 1kΩ | 0805 | C17513 | 1 |
| 13 | TVS Diode | SMBJ24A | SMB | C13982 | 1 |

## Design-Vorgaben

### Elektrische Spezifikationen
- **5V Zweig:**
  - Ausgangsspannung: 5.0V ±3% (4.85V - 5.15V)
  - Ausgangsstrom: 5A kontinuierlich
  - Eingangsspannung: 20V von USB-PD
  - Ripple: <50mV pp

- **9V AC Zweig:**
  - Ausgangsspannung: 9V RMS
  - Frequenz: 50Hz oder 60Hz (konfiguierbar)
  - Ausgangsstrom: 2A kontinuierlich
  - Wellenform: Sinus (THD zu verifizieren)

### Schutzfunktionen
- **Crowbar-Schaltung:** SCR löst bei >5.6V aus (schützt C64 vor Überspannung)
- **TVS-Diode:** SMBJ24A am Eingang gegen Transienten
- **Post-LC-Filter:** Zusätzliche Glättung der Ausgangsspannung

## Projekt-Konventionen

### KiCad Symbole
- Alle Custom-Symbole in `USB-PD-2-C64-Adapter.kicad_sym`
- **Pflichtfeld:** `LCSC` mit C-Nummer (z.B. `C6288609`)
- Bei jeder Änderung: BOM synchronisieren

### Komponenten-Referenzen
- R = Widerstand
- C = Kondensator  
- L = Induktor
- U = IC
- D = Diode
- Q = Thyristor/SCR

### BOM-Verwaltung
- Format: CSV für JLCPCB
- Pfad: `Production/USB-PD-2-C64-Adapter_BOM.csv`
- **Bei jeder Schaltplan-Änderung BOM aktualisieren!**

## Offene Fragen / TODOs

### Design-Review erforderlich
- [ ] **5V Zweig:** Ist das Buck-Converter-Design mit Crowbar ausreichend für 5A?
- [ ] **9V AC Zweig:** Kann ATtiny412-Inverter-Design 2A sauber liefern?
- [ ] **Sinusqualität:** THD-Messung des 9V AC Ausgangs
- [ ] **Thermik:** Verlustleistungsberechnung bei Volllast

### Fertigung
- [ ] PCB Review vor Bestellung
- [ ] Thermische Analyse (Buck Converter, Inverter)
- [ ] EMV-Überlegungen

### Mechanik
- [ ] Gehäuse-Design (3D-Druck)
- [ ] Kabelführung USB-C ↔ DIN-7
