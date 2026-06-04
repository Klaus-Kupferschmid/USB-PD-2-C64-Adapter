# USB-PD-2-C64-Adapter - Copilot Instructions

## Projektübersicht
Dies ist ein KiCad Hardware-Projekt für einen USB Power Delivery zu Commodore 64 Netzteil-Adapter.

**Ziel:** Moderne USB-PD Netzteile (z.B. Laptop-Ladegeräte) als sichere 5V DC Stromversorgung für den Commodore 64 nutzen.

## Projektstruktur
```
KiCad/
├── USB-PD-2-C64-Adapter.kicad_sch    # Hauptschaltplan
├── USB-PD-2-C64-Adapter.kicad_pcb    # PCB Layout
├── USB-PD-2-C64-Adapter.kicad_sym    # Projektsymbole
├── RT6318C.kicad_sym                  # Buck Converter Symbol
├── Production/                        # Fertigungsdaten, BOM
│   └── USB-PD-2-C64-Adapter_BOM.txt  # Stückliste (LCSC-kompatibel)
└── Datasheet/                         # Datenblätter der Komponenten
```

## Hauptkomponenten
| Komponente | Funktion | LCSC-Nr. |
|------------|----------|----------|
| RT6318CGQUF | Synchroner Buck Converter | C6288609 |
| Induktor 1uH 10A | Hauptinduktor | C15949 |
| SCR BT151 | Crowbar-Überspannungsschutz | C2563 |
| BZT52C5V6 | Z-Diode 5.6V Referenz | C31673 |
| SMBJ24A | TVS Transientenschutz | C13982 |

## Design-Vorgaben
- **Ausgangsspannung:** 5V DC (±5%)
- **Eingangsspannung:** 9-20V von USB-PD
- **Max. Strom:** ~2A (typischer C64 Verbrauch)
- **Schutzfunktionen:**
  - Crowbar-Schaltung löst bei >5.6V aus
  - TVS-Diode am Eingang gegen Transienten
  - Post-LC-Filter für saubere Ausgangsspannung

## Fertigungshinweise
- BOM ist für JLCPCB Assembly / LCSC optimiert
- Kondensatoren: MLCC X7R + Polymer-Elkos für niedrige ESR
- Shunt-Widerstand 0.03Ω 1206 für Strommessung

## Konventionen für Code-Generierung
- Bei KiCad-Dateien: S-Expression Syntax beachten
- Komponenten-Referenzen: R=Widerstand, C=Kondensator, L=Induktor, U=IC, D=Diode, Q=Thyristor/SCR
- Einheiten: SI-Einheiten (µF, µH, mΩ)

## TODOs
- [ ] Thermische Analyse des Buck Converters
- [ ] PCB Review vor Fertigung
- [ ] Gehäuse-Design (3D-Druck)
