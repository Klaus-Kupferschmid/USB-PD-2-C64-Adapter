# USB-PD-2-C64-Adapter - Copilot Instructions

## Projektübersicht

KiCad-PCB-Adapter für ein modernes USB-C-PD-Netzteil zur Stromversorgung eines Commodore C64.

**Anschlüsse:**

- Eingang: USB-C Stecker (USB-PD, **15V/3A fest vorgesehen**)
- Ausgang: C64-Netzteilstecker (DIN 7-polig)

**Zwingende Energiequelle:** Als einzige Energiequelle des gesamten Adapters darf ein modernes **USB-C-PD-Netzteil** dienen. Externe Zusatznetzteile, ein separater Trafo oder ein netzspannungsbasierter Trafo-Zweig sind keine Option für dieses Projekt.

**C64-DIN-7-Pinbelegung (Außenansicht auf die C64-Netzbuchse, DIN 45329):**

| Pin | Signal | Bemerkung |
| --- | --- | --- |
| 1 | GND | Masse/unbelegt |
| 2 | GND | Masse |
| 3 | GND | Masse/unbelegt |
| 4 | - / +5V DC | unbelegt oder +5V statt Pin 5; Pin 4 und 5 sind laut C64-Schaltplan direkt nach der Buchse verbunden |
| 5 | +5V DC | 5 Volt Gleichspannung |
| 6 | 9V AC 1 | 9 Volt Wechselspannung |
| 7 | 9V AC 2 | 9 Volt Wechselspannung |

Für das Adapterlayout sind Pin 4/5 als möglicher gemeinsamer +5V-Strompfad und Pin 1/2/3 als Massepfad bewusst zu prüfen. Pin 6 und 7 bilden den 9VAC-Ausgang; Vertauschen der beiden AC-Pins ist elektrisch unkritisch, die mechanische Pinzuordnung muss trotzdem exakt geprüft werden.

**Erforderliche Ausgangsspannungen (gemäß Original-Netzteil):**

- **5 VDC** mit **3 A Zielstrom** und thermischer Reserve (Logik, SID, VIC-II)
- **9 VAC** mit **2 A** (CIA Timer, User Port)

## Systemarchitektur

### USB-PD Eingangsstufe

- **CH224K** USB PD Power Receiving Chip fordert **15V-PD** vom USB-C Netzteil an; 15V/3A ist die aktuelle feste Systemvorgabe

### 5 VDC Zweig (Buck Converter)

**Aktualisierte C64-Zielgröße:** Nach Bewertung moderner C64-Netzteilkonzepte ist 5A Dauerstrom für den C64-Ausgang nicht das eigentliche Qualitätsziel. Das Originalnetzteil lag bei ca. 1.5A auf 5V; Erweiterungen wie REU-orientierte Setups wurden historisch eher mit ca. 2.5A versorgt. Für einen hochwertigen C64-Adapter ist daher **3A auf der 5V-Schiene ein sinnvoller Zielwert mit Reserve**, während die DIN-Buchse und die 5V-Pins nicht unnötig mit einem 5A-Dauerlastanspruch belastet werden sollten. Entscheidend sind präzise 5.0V am C64, geringe Restwelligkeit, kontrolliertes Ein-/Ausschalten, Rückstrom-/Überspannungsschutz und stabile Lastsprungantwort.

**Festlegung Eingangsspannung:** Der aktuelle Entwurf ist auf **15V/3A USB-PD als einzige Energiequelle** festgelegt. 12V bleibt für die 9VAC/2A-Sinuserzeugung zu knapp, weil 9VAC RMS ca. 12.7V Spitzenspannung plus Verluste/Regelreserve benötigt. 20V-PD wird für den AP64500-5V-Zweig nicht mehr als Zielbetrieb vorgesehen; dadurch sinken Schaltstress, Verlustleistung und Anforderungen an Eingangsschutz und Kondensatoren. 15V/2A liefert nur 30W und ist für 5V/3A plus 9VAC/2A Worst-Case knapp; **15V/3A mit 45W ist die aktuelle Systemvorgabe.** Ein separater Trafo- oder Zweitnetzteilpfad ist ausdrücklich ausgeschlossen.

**Bewertung nach RT6318C-Datenblatt (2021-12):** Der RT6318CGQUF ist als 5.1V/8A-Synchron-Buck grundsätzlich geeignet, wenn das Layout eng am Referenzdesign bleibt und der Überspannungsschutz separat präzise ausgelegt wird. Kritisch ist die geringe Eingangsspannungsreserve: RT6318C ist für **5.1V bis 23V VIN empfohlen**, bei **27V absolute maximum**. Ein USB-PD-20V-Netzteil liegt damit im Normalbetrieb passend, Transienten müssen aber durch TVS, kurze Leitungen und Eingangskapazität sicher unterhalb der IC-Grenzen bleiben.

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

**RT6318C-Referenzbeschaltung als Mindestbasis:**

- Eingang: mindestens **2x 10µF X5R/X7R Keramik** direkt an VIN/PGND; vorhandener Bulk-Elko/Polymer zusätzlich sinnvoll. Für 20V-Betrieb 35V oder 50V Spannungsfestigkeit bevorzugen, da 25V MLCC mit DC-Bias und Transienten wenig Reserve hat.
- VCC: **1µF** nach AGND/PGND gemäß Datenblatt, **nicht als externe Lastversorgung verwenden**.
- LDO5: **4.7µF** Stützkondensator; nur bis 100mA belastbar, nicht für externe 5V-Lasten nutzen.
- BOOT-LX: **0.1µF Bootstrap-Kondensator plus mindestens 10Ω RBOOT**. RBOOT kann EMI reduzieren, erhöht aber Schaltverluste leicht.
- Leistungspfad: **1µH Induktor** wie im Datenblatt; Sättigungsstrom und thermischer Strom müssen den berechneten Peak-Strom und Fehlerfälle abdecken.
- Ausgang: **4x 22µF MLCC** als lokale Buck-Ausgangskapazität; Bulk-Kapazität zusätzlich möglich, aber Stabilität/Lastsprung prüfen.
- FF/CFF: RT6318C nutzt integrierte Kompensation, der **Feedforward-Kondensator am FF-Pin** verbessert laut Datenblatt Lastsprungverhalten und Phasenreserve. Nicht weglassen, sondern aus Referenzdesign übernehmen und bei Post-LC-Filter verifizieren.
- EN: nicht offen lassen; mit Pull-up/Steuerung eindeutig definieren. PGOOD ist Open-Drain und benötigt Pull-up.

**Wichtige Datenblatt-Grenzen für das Design:**

- RT6318C: 5.1V fixed output, 8A nominaler Ausgangsstrom, 750kHz Switching Frequency.
- Interner Soft-Start ca. 0.6ms; zusätzlicher Einschaltstrom durch große Ausgangs-/Postfilter-Kapazität muss trotzdem geprüft werden.
- Cycle-by-cycle Valley Current Limit ca. 9A, Peak Current Limit ca. 15A; nicht als alleiniger C64-Schutz betrachten.
- OVP/UVP/OTP arbeiten latch-off und werden durch EN-Toggle oder Power-Cycle zurückgesetzt. Die interne OVP schützt nicht zuverlässig gegen alle Buck-Fehlerarten, daher externe Crowbar/eFuse beibehalten.
- Dauerbetrieb thermisch auf **TJ <125°C Zielwert** auslegen; OTP erst bei ca. 150°C ist kein normaler Betriebsbereich.

**Empfohlene Korrektur am bisherigen 5V-Konzept:**

- Crowbar-Trigger nicht nur mit 5.6V-Zener direkt ans SCR-Gate auslegen. Für C64-Schutz besser TL431/LMV431 oder Komparator-Referenz mit Trigger ca. **5.35V bis 5.45V** verwenden und Toleranzen/Temperatur berücksichtigen.
- Vor der Crowbar ein Abschaltelement vorsehen: Schmelzsicherung, Polyfuse, eFuse oder High-Side-Switch. Ohne Abschaltelement erzeugt die Crowbar nur einen harten Kurzschluss.
- Den 0.03Ω-Shunt überdenken: bei 5A fallen 150mV ab und ca. 0.75W Verlustleistung entsteht. Für Messung/Schutz eher 5mΩ bis 10mΩ mit Kelvin-Anschluss oder dedizierten Current-Sense/eFuse-Baustein verwenden.
- Feedback-/Sense-Punkt bewusst festlegen. Ziel ist **5.0V am C64-DIN-Stecker unter Last**, nicht nur am Buck-Ausgang. Kabel-, Shunt- und Filterverluste müssen in die Toleranzrechnung.
- Post-LC-Filter nicht blind hinter den Buck setzen. Lf/Cf kann die Regelschleife und Lastsprungantwort beeinflussen; Dämpfung (ESR oder RC-Dämpfung) und Messung von Ripple, Overshoot und Undershoot sind Pflicht.
- Historische RT6318C-TVS-Auslegung nicht in den aktuellen AP64500-Entwurf übernehmen. Für den aktuellen 15V/AP64500-Zweig ist D1 als SMBJ18A-Startwert festgelegt; aktueller LCSC-Kandidat ist **C353379**. TVS, Sicherung/eFuse und reale Pulsenergie gemeinsam validieren.
- Layout als Teil der Schaltung behandeln: VIN-Caps direkt am IC, kleinste Hot-Loop VIN-HS-FET-LX-PGND, kurze LX-Fläche, stern-/netztopologisch saubere AGND/PGND-Anbindung, Thermal-Pad mit Via-Feld, breite 5V/GND-Kupferflächen, Kelvin-Sense am Shunt.

**Baustein-Alternativen (LCSC-Stand 2026-06-05, vor BOM-Änderung erneut prüfen):**

- **RT6318CGQUF, LCSC C6288609:** 5.1V fixed, 8A, synchron, 5.1V bis 23V VIN, 750kHz, UQFN-12 3x3. Elektrisch gute Stromreserve und wenige externe Bauteile, aber knappe VIN-Reserve für 20V-PD-Transienten. LCSC zeigte ihn als out of stock. Zusätzlich schlecht für Handaufbau/Reparatur: UQFN-12 3x3 mit Thermal-Pad ist ohne Heißluft/Reflow und gute Inspektion schwer zuverlässig selbst zu löten.
- **DIODES AP64501/AP64500-Familie:** 3.8V bis 40V VIN, 5A synchron, SO-8-EP, integrierte 45mΩ/20mΩ MOSFETs, Spread Spectrum/FSS, UVLO, OVP, Cycle-by-cycle Peak Current Limit und Thermal Shutdown. AP64501: feste 570kHz, programmierbarer Soft-Start, externe Kompensation. AP64500: einstellbare 100kHz bis 2.2MHz, externer Sync, feste Soft-Start-Zeit, externe Kompensation. Vorteil: deutlich besser handlötbar als VFQFN/QFN und bei 3A C64-Zielstrom ausreichend Reserven. Für das Projekt ist **AP64500 gegenüber AP64501 zu bevorzugen**, wenn <40mVpp Ripple hart angestrebt wird, weil die höhere Schaltfrequenz kleinere Induktivität, geringere Ripple-Energie pro Zyklus und kompaktere Ausgangsfilter erlaubt. Nachteil: beide Low-IQ-Varianten nutzen PFM im Leichtlastbereich; niederfrequente Restwelligkeit bei kleiner Last muss gemessen und ggf. durch Mindestlast, Betriebsmodus-/Variantenauswahl oder Nachfilter adressiert werden.
- **TI TPS54560DDAR, LCSC C31966:** 4.5V bis 60V VIN, 5A, SOIC-8-EP, sehr günstig/gut verfügbar und gut selbst lötbar. Für dieses Projekt eher **verwerfen**, weil es kein voll synchroner 5A-Buck wie RT6318/AP64500/LM76005 ist, sondern einen externen Freilaufpfad benötigt; bei 20V auf 5V und hoher Last sind Wirkungsgrad und Wärme schlechter. Gut für robuste Industrieversorgung, aber nicht erste Wahl für kompakte 5V/5A-C64-Versorgung.
- **TI LM76005RNPR, LCSC C2866241:** 3.5V bis 60V VIN, 5A synchron, 200kHz bis 500kHz, WQFN-30 4x6, sehr niedriger Ruhestrom, Hiccup, einstellbarer Soft-Start, PGOOD/Tracking. Technisch robust und moderner als TPS54560, aber teurer, nur 5A Nennstrom und wegen WQFN-30 mit Thermal-Pad kaum angenehm selbst zu löten. Für das aktualisierte 3A-C64-Ziel technisch ausreichend, aber gegenüber BD9P608MFF-C ohne klaren Vorteil.
- **ADI LT8645SEV#PBF, LCSC C514430:** 3.4V bis 65V VIN, 8A synchron, 200kHz bis 2.2MHz, Silent-Switcher-Architektur, sehr EMI-stark, LQFN-32 4x6. Technisch die beste Alternative zum RT6318C, weil sie sowohl VIN-Reserve als auch 8A Stromreserve bietet. Nachteil: deutlich teuer, anspruchsvolleres Layout, mehr externe Auslegung und ebenfalls nicht handlötfreundlich. Sinnvoll, wenn Robustheit/EMI wichtiger sind als Kosten und Reflow/JLC-Assembly akzeptiert ist.
- **XLSEMI XL9028 (LCSC-Suche):** Buck-Regler, 1.25V bis 40V Ausgangsbereich laut Suchtreffer, 6A, TO-263-7L, LCSC zeigte Bestand. Handlötbarer als QFN, aber vermutlich ältere/asynchrone Architektur; Effizienz, Ripple, Schutzfunktionen und Datenblattqualität vor Einsatz streng prüfen. Eher pragmatischer Bastel-Kandidat als Premium-C64-Versorgung.
- **XLSEMI XL9026 (LCSC-Suche):** Buck-Regler, 1.25V bis 40V Ausgangsbereich laut Suchtreffer, 6A, TO-220-5L, LCSC zeigte kleinen Bestand. Sehr gut selbst lötbar und thermisch leichter mit Kühlfläche/Kühlkörper zu beherrschen, aber ebenfalls nicht automatisch moderner oder rauschärmer als AP64500/LT8645S. Nur verwenden, wenn das Datenblatt eine geeignete 20V-auf-5V/5A-Applikation und akzeptable Schutzfunktionen bestätigt.
- **ROHM BD9P608MFF-CE2, LCSC C23478093:** 3.5V bis 40V VIN, 6A synchroner Step-down, Ausgang 0.8V bis 8.5V, 440kHz oder 2.2MHz, VFQFN-20 3.5x4.5, LCSC zeigte Bestand. Enthält laut Datenblatt/Produktseite u.a. Soft-Start, Current-Mode-Control, Light-Load-Mode, Forced-PWM-Mode, Spread Spectrum, externen Sync, VOUT_SNS, OCP, UVLO, TSD, OVP und SCP. Für das Projekt ist das aktuell der passendste moderne Kandidat, wenn JLC-Assembly/Reflow akzeptiert ist: 40V VIN entspannt den 20V-PD-Transientenfall deutlich, 6A bietet komfortable Reserve gegenüber 3A Zielstrom, und der Forced-PWM-Modus plus Sense-Pin spricht für eine gleichmäßigere 5V-Schiene als bei einem reinen Leichtlast-PFM-Betrieb. Nachteil bleibt die schlechte Handlötbarkeit des VFQFN-Gehäuses.

Aktuelle Empfehlung: **AP64500/AP64501 als handlötbaren Zielpfad ernsthaft weiterverfolgen**, weil Selbstaufbau und Nacharbeit für dieses Projekt wichtig sind. Das Qualitätsziel ist jetzt nicht mehr 5A Dauerlast, sondern eine saubere 5V-Schiene mit ca. 3A C64-Zielstrom, <40mVpp Ripple als Messziel und gutem Lastsprungverhalten. Innerhalb der Diodes-Familie ist **AP64500** wegen einstellbarer Schaltfrequenz bis 2.2MHz und externer Kompensation der bessere Kandidat für <40mVpp; AP64501 bleibt einfacher/fix bei 570kHz, aber potentiell schwieriger auf sehr niedrigen Ripple zu trimmen. **BD9P608MFF-CE2/C23478093** bleibt technisch attraktiv, vor allem wegen Forced-PWM, VOUT_SNS, 6A Reserve und Schutzfunktionen, ist aber wegen VFQFN nicht der bevorzugte Handaufbau-Kandidat. **TPS54560** für dieses Projekt nicht bevorzugen.

**AP64500/AP64501-Ripple-Strategie <40mVpp:** Das Ziel ist erreichbar, aber nur mit konsequentem Buck-Layout und Filterdesign. Für den handlötbaren Zielpfad bevorzugt **AP64500** statt AP64501 prüfen: hohe Schaltfrequenz im Bereich ca. 1MHz bis 2.2MHz, externe Kompensation nach Datenblatt/EVM, Induktor mit passendem Sättigungsstrom und niedrigem DCR, mehrere eng platzierte X7R-Ausgangskeramiken plus ein Polymer-/Low-ESR-Bulk, kurze Stromschleifen und durchgehende Massefläche. Das Ripple-Ziel gilt am C64-DIN-Stecker, nicht nur direkt am Regler. AP64501 kann <40mVpp ebenfalls schaffen, aber die feste 570kHz-Frequenz erzeugt bei gleicher Induktivität/Kapazität mehr Ripple-Energie; dann werden Induktor- und Ausgangsfilter größer. Bei beiden Low-IQ-Varianten muss PFM-Leichtlastwelligkeit geprüft werden. Wenn PFM bei kleiner C64-Last sichtbar stört, Optionen prüfen: AP64500 mit höherer Frequenz/externem Sync, kleine definierte Mindestlast, gedämpfter Post-LC-Filter oder alternative Variante mit erzwungenem PWM-Betrieb. Messziel: <40mVpp am C64-DIN-Stecker bei typischer Last, 0.5A/1.5A/3A und Lastsprung, mit kurzer Massefeder am Oszilloskop messen.

**AP64500/AP64501 PCB-Layout nach Datenblatt-Figure-31:** Das Datenblatt-Layoutbild ist als verbindliche Orientierung für den Buck-Bereich zu behandeln. Für 5A-fähige AP64500/AP64501-Layouts empfiehlt Diodes u.a. 2oz Kupfer auf Top und Bottom, wenn Kosten/Herstellung es zulassen. Eingangskondensatoren müssen so dicht wie möglich zwischen VIN und GND sitzen; diese VIN-Caps bilden mit den internen MOSFETs die kritischste Hot-Loop und dürfen nicht über lange Leiterbahnen angebunden werden. Die Induktivität kommt direkt an den SW-Pin; die SW-Kupferfläche soll kurz und kompakt bleiben, nicht unter Feedback/Compensation laufen und nicht unnötig als Antenne vergrößert werden. Ausgangskondensatoren liegen so nah wie möglich an L/VOUT und GND. Feedback- und Kompensationsnetzwerk sitzen direkt bei FB/COMP, mit ruhigem GND-Bezug und Abstand zu SW/VIN. Exposed Pad, GND-Pin und VIN-Pin bekommen viele Thermal-/Stitching-Vias in die jeweiligen Kupferflächen; bei 4-lagigem Layout sollte Layer 2 bevorzugt GND sein und Layer 3 als zusätzliche GND-/Power-Fläche zur Wärmeabfuhr dienen. Breite VIN-, VOUT- und GND-Flächen sind Pflicht; die Reglerfläche muss vor dem Rest der Platine als HF-Leistungsschaltung geroutet werden, nicht als nachträgliche Verbindung.

**AP64500 Zielbeschaltung 5V mit Schutz und <40mVpp-Startwerten:** Diese Skizze ist der aktuelle Schaltungsentwurf fuer den handloetbaren 5V-Zweig. Werte sind Startwerte fuer den ersten Schaltplan und muessen mit Diodes-Rechner/Simulation, Layout und Messung verifiziert werden.

```text
USB-C PD 15V/3A
   |
  +-- FH1 XFCN XF-155 Sicherungshalter (LCSC C41371860) + F1 JDTfuse JFC2410-1400TS 4A/250V 2410-Sicherung (LCSC C136386); TS-Zeitstromkennlinie vor Freigabe im Datenblatt prüfen
   |
    +-- D_IN SMBJ18A unidirektionale TVS gegen GND als Startwert fuer festen 15V-PD-Eingang; Klemmspannung/Energie mit Sicherung oder eFuse verifizieren
   |
   +-- C_IN_BULK 47uF..100uF / >=35V Low-ESR
   |
   +-- C_IN1,C_IN2 2x 10uF / 35V oder 50V X7R direkt an VIN-GND
   |
   +-----------------------------+
                                 |
                              VIN|2
                         EN/UVLO |3  <- Supervisor/Power-Logic, nicht nur hart an VIN
                   RT/CLK        |4  <- RT fuer fsw, bevorzugt AP64500
                              BST|1 -- C_BST 100nF X7R -- SW
                            AP64500
                              SW |8 -- L1 --+---- 5V_RAW
                             GND |7        |
                         EXP PAD |9       C_OUT_LOCAL
                              FB |5        |
                            COMP |6       GND
                                 |
                                 +-- FB-Teiler vom Sense-Punkt nach Schutzpfad

5V_RAW
   |
   +-- C_OUT_LOCAL: 4x 22uF X7R 10V/16V sehr nah an L1/VOUT-GND
   +-- C_OUT_BULK: 150uF..220uF Polymer/Low-ESR 10V nahe DIN-Ausgang
   |
   +-- optional gedämpfter Post-LC nur nach Stabilitaetspruefung
   |
  +-- U6 TPS16630PWPR zentrale 15V-eFuse/Load-Switch; schaltet VIN_BUCK und spaeter den 9VAC-Zweig erst nach PD-Freigabe/Steuerlogik frei
   |
   +-- R_SENSE optional 5mΩ..10mΩ Kelvin, nicht 30mΩ
   |
   +---- 5V_PROTECTED / Sense-Punkt -----> C64 DIN 5V-Pins
   |          |
   |          +-- Crowbar SCR nach GND, aber nur hinter Abschaltelement/eFuse
   |          |
   |          +-- TLV431A/LMV431 Trigger:
   |                 R_TOP 11.8k 0.1% von 5V_PROTECTED nach REF
   |                 R_BOT 10.0k 0.1% von REF nach GND
   |                 V_TRIP ca. 5.44V nominal
   |                 Gate-Widerstand SCR 330Ω..1k nach realem Gate-Strom auslegen
   |
  GND -----------------------------------------------------------------> DIN GND
```

AP64500-Startwerte fuer 15V-PD auf 5V/3A mit Ripple-Ziel <40mVpp:

- **U1:** AP64500SP-13, SO-8EP, 3.8V bis 40V, 5A synchron.
- **FB:** R1 oben **115.8kΩ**, R2 unten **22.1kΩ** fuer ca. 5.0V nach Datenblatt; FB-Leitung vom ruhigen Sense-Punkt nach Schutzpfad fuehren, nicht vom SW-/Induktorbereich.
- **Bootstrap:** C_BST **100nF X7R** direkt zwischen BST und SW.
- **Frequenz:** AP64500 bevorzugt auf ca. **1.5MHz** als erster Versuch; RT nach Datenblattgleichung `RT[kΩ] = 100000 / fSW[kHz]`, also **66.5kΩ bis 66.8kΩ** fuer 1.5MHz. 1MHz und 2.2MHz als Messvarianten vorsehen.
- **Induktor:** L1 **2.2uH** als Startwert bei 15V->5V/1.5MHz, Sättigungsstrom mindestens **7A**, RMS-Strom mindestens **4A**, DCR moeglichst niedrig, Ziel <20mΩ und wenn machbar <10mΩ.
- **Ausgang lokal:** mindestens **4x 22uF X7R** nahe am Regler/Induktor; DC-Bias real beruecksichtigen. Bei 10V-MLCCs kann die effektive Kapazitaet stark fallen, 16V-MLCCs pruefen.
- **Ausgang Bulk:** **150uF bis 220uF Polymer/Low-ESR 10V** nahe am DIN-/Lastanschluss, aber Stabilitaet mit Kompensation pruefen.
- **Kompensation erster 1.5MHz-Entwurf:** Bei ca. 45uF bis 60uF effektiver Ausgangskapazitaet und Ziel-Crossover grob 30kHz als Start: R5 **31.6kΩ bis 42.2kΩ**, C5 **1.5nF**, C6 **4.7pF bis 6.8pF optional**, C4 Feedforward **10pF bis 22pF optional**. Das 500kHz-Datenblattbeispiel bleibt Referenz: R5 15.8kΩ, C5 2.7nF, C6 39pF, C4 47pF. Endwerte erst nach Diodes-Rechner/Simulation und Messung festlegen.
- **Input:** mindestens **2x 10uF X7R 35V/50V** direkt am IC plus Bulk 47uF..100uF. Input-RMS-Stromrating der Kondensatoren beachten. Als TVS-Startwert fuer den festgelegten 15V-AP64500-Zweig ist **SMBJ18A unidirektional, LCSC C353379** vorgesehen: VRWM 18V liegt oberhalb eines 15V-PD-Normalfalls inklusive Toleranz, VBR 22.1V, VC 29.2V bei Ipp 20.6A und 600W bei 10/1000us passen gut zur 40V-VIN-Grenze des AP64500. Die TVS muss zusammen mit Sicherung/eFuse und realer Pulsenergie validiert werden.
- **Schutz:** AP64500-interne OVP liegt grob bei +10% und ist nicht ausreichend als alleiniger C64-Überspannungsschutz. Aktuelle Architektur: **U6 TPS16630PWPR** als zentrale 15V-eFuse/Load-Switch hinter F1 und vor `VIN_BUCK`; der spaetere 9VAC-Zweig muss ebenfalls von dieser geschalteten Schiene versorgt werden. U3 **TPS3700DDCR** bleibt als 5V-OVP bei ca. 5.36V und zieht ueber `EFUSE_SHDN` U6-`SHDN` low; R13 zieht `EFUSE_SHDN` auf `CH224_VDD`, nicht auf 15V. U6-Startwerte: R14/R15/R16 = 274k/7.68k/20.0k fuer UVLO/OVP-Teiler mit LCSC C482818/C163419/C844676, R17 = 18k fuer ILIM, C16 = 22nF fuer dVdT, MODE bewusst offen fuer Latch-off. Werte und LCSC-C-Nummern vor PCB-Freigabe erneut gegen TPS1663-Datenblatt verifizieren.
- **Messpflicht:** Ripple am C64-DIN-Stecker messen: 0.5A, 1.5A, 3A, Lastsprung und kleiner Leichtlastfall wegen PFM. Oszi mit kurzer Massefeder, Bandbreitenlimit dokumentieren.

### 9 VAC / 2 A Zweig (Inverter)

- Referenz-Design: [wagiminator ATtiny412-USB-PD-Inverter](https://github.com/wagiminator/ATtiny412-USB-PD-Inverter)
- **Kritische Anforderungen:**
  - Sauberer Sinus bei 50 Hz oder 60 Hz
  - Belastbarkeit mindestens 2 A kontinuierlich
  - 9VAC RMS im Leerlauf maximal 15% über Nennspannung, also **max. ca. 10.3VAC**
  - 9VAC muss abgeschaltet werden, wenn keine 5V-Last entnommen wird; Hintergrund: Der C64-Einschalter unterbricht nur einen Pol der 9VAC-Versorgung, sonst können Teile des C64 im ausgeschalteten Zustand unerwünscht versorgt werden
  - THD (Total Harmonic Distortion) prüfen
  - Energieversorgung ausschließlich aus dem USB-C-PD-15V/3A-System; Trafo- oder Zweitnetzteil-Lösungen sind für dieses Projekt ausgeschlossen

**Aktuelle 9VAC-Architekturentscheidung:** Der 9VAC-Zweig wird als synthetischer, differentieller Sinusausgang aus der geschalteten 15V-Schiene hinter U6 aufgebaut. Pin 6 und Pin 7 des DIN-Ausgangs dürfen nicht als einseitig gegen Board-GND geschalteter AC-Ausgang betrachtet werden; der Ausgang muss als H-Brücken-/Vollbrücken-Ausgang zwischen `9VAC_6` und `9VAC_7` arbeiten. Ein Anschluss von Pin 6 oder 7 an GND ist nicht zulässig. Ziel ist eine möglichst floating wirkende AC-Quelle mit kontrolliertem Common-Mode-Verhalten, damit User-Port- und interner C64-9VAC-Pfad nicht unerwartet gegen die 5V-Masse rückgespeist werden.

**Topologie-Startpunkt:** `VIN_BUCK` bzw. die von U6 geschaltete 15V-Schiene speist einen eigenen 9VAC-Leistungspfad mit lokaler Bulk-Kapazität, Vollbrücken-Endstufe, Sinus-PWM-Erzeugung und LC-/Rekonstruktionsfilter. Das wagiminator-ATtiny412-Design bleibt ein Firmware-/Konzept-Referenzpunkt, muss fuer 9VAC/2A aber als Leistungsteil neu dimensioniert werden: MOSFETs oder H-Brücken-Treiber, Induktivitäten, Filterkondensatoren, Strommessung, thermische Kupferflächen und Abschaltpfade sind nicht aus einem kleinen Referenzdesign zu übernehmen. Fertige DC-AC-Module sind nur akzeptabel, wenn sie 15V Eingang, echten sinusfoermigen differentiellen 9VAC-Ausgang, 2A Dauerlast, THD-Nachweis, Leerlaufbegrenzung und die 5V-Last-abhängige Abschaltung erfüllen; typische Mini-Inverter- oder Rechteckwandler-Module erfüllen diese Anforderungen voraussichtlich nicht.

**Regelung und Messung:** Die 9VAC-Amplitude darf nicht nur offen per PWM-Tabelle angenommen werden. Erforderlich ist mindestens eine Mess-/Kalibrierstelle fuer die differentielle Ausgangsspannung nach Filter; bevorzugt wird eine MCU-gestuetzte RMS-/Amplitude-Korrektur oder eine analoge Begrenzung, damit Leerlauf und Lastfall innerhalb der 9VAC-Spezifikation bleiben. Messpunkte fuer `9VAC_6`, `9VAC_7`, differentielles 9VAC, 9VAC-Strom und Endstufentemperatur vorsehen.

**5V-Last-Erkennung fuer 9VAC-Abschaltung:** Die Abschaltung soll nicht die 9VAC-Last messen, sondern erkennen, ob der C64 die 5V-Schiene aktiv belastet. Startarchitektur: kleiner High-Side- oder Low-Side-Current-Sense im 5V-Pfad mit Komparator/MCU-Auswertung, Hysterese und Zeitfilter. Unterhalb einer noch zu messenden Schwelle wird die 9VAC-Endstufe deaktiviert; oberhalb der Schwelle wird sie erst nach stabiler 5V-Freigabe und kurzer Verzögerung aktiviert. Die Schwelle muss später mit echten C64-Lastprofilen festgelegt werden, damit ausgeschaltete C64s sicher erkannt werden, aber sehr kleine aktive Lasten nicht versehentlich 9VAC verlieren.

**Leistungsbudget:** 5V/3A plus 9VAC/2A liegt mit Wandlerverlusten nah am 45W-Budget von 15V/3A. Der Adapter muss deshalb nicht nur Einzelzweige, sondern auch gleichzeitige Worst-Case-Last, PD-Spannungstoleranz, U6-Stromlimit, Sicherungskennlinie, thermische Endstufenlast und Spannungseinbruch pruefen. Falls 9VAC/2A und 5V/3A gleichzeitig als Dauer-Worst-Case gefordert werden, ist das 15V/3A-Eingangsbudget ein harter Grenzfall und muss vor PCB-Freigabe rechnerisch und im Prototyp validiert werden.

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

## Aktuelle BOM (verbindlich, Stand Schaltplan USB-PD-2-C64-Adapter.kicad_sch)

Diese Tabelle ist die Quelle der Wahrheit und stimmt mit `Production/USB-PD-2-C64-Adapter_BOM.csv`, `Production/USB-PD-2-C64-Adapter_BOM.txt` und den KiCad-Symbolen überein. Die BOM enthält die Spalte **Baugruppe** mit den Werten `USB-PD Eingang`, `5VDC` und `9VAC`. Die USB-PD-Eingangsteile sind im Schaltplan aktuell bewusst **unverdrahtet** platziert; Verbindungen werden manuell gesetzt.

| Ref | Wert | Baugruppe | Funktion | Footprint | LCSC | Menge |
| --- | --- | --- | --- | --- | --- | --- |
| J3 | SHOU HAN TYPE-C16PIN | USB-PD Eingang | USB-C-Buchse 16P, 30V, 3A | USB-PD-2-C64-Adapter:TYPE-C16PIN_C393939 | C393939 | 1 |
| U4 | CH224K | USB-PD Eingang | USB-PD Sink Controller, 15V-Anforderung | USB-PD-2-C64-Adapter:CH224K_ESSOP-10-150mil-1mm | C970725 | 1 |
| U5 | TPD2S300YFFR | USB-PD Eingang | USB-C CC Short-to-VBUS- und IEC-ESD-Schutz | USB-PD-2-C64-Adapter:TPD2S300YFFR_DSBGA-9 | C2650411 | 1 |
| U6 | TPS16630PWPR | USB-PD Eingang | Zentrale 15V-eFuse/Load-Switch fuer VIN_BUCK und spaeter 9VAC | Package_SO:HTSSOP-20-1EP_4.4x6.5mm_P0.65mm_EP3.0x4.2mm | C1849461 | 1 |
| C16 | 22nF X7R 50V | USB-PD Eingang | U6 dVdT-Softstart-Kondensator, Startwert | Capacitor_SMD:C_0603_1608Metric | C106222 | 1 |
| R8 | 56k 1% | USB-PD Eingang | CH224K CFG1 nach GND fuer 15V | Resistor_SMD:R_0603_1608Metric | C114630 | 1 |
| R9 | 1k 1% | USB-PD Eingang | CH224K VDD-Zufuehrung | Resistor_SMD:R_1206_3216Metric | C131398 | 1 |
| R10 | 470R 1% | USB-PD Eingang | Gruene 15V-PD-Status-LED | Resistor_SMD:R_0603_1608Metric | C23179 | 1 |
| R11 | 1M 1% | USB-PD Eingang | AP64500 EN/UVLO oben, Startwert | Resistor_SMD:R_0603_1608Metric | C105578 | 1 |
| R12 | 100k 1% | USB-PD Eingang | AP64500 EN/UVLO unten, Startwert | Resistor_SMD:R_0603_1608Metric | C14675 | 1 |
| R13 | 100k 1% | USB-PD Eingang | U6 SHDN Pull-up nach CH224_VDD | Resistor_SMD:R_0603_1608Metric | C14675 | 1 |
| R14 | 274k 1% | USB-PD Eingang | U6 UVLO/OVP-Teiler oben, Startwert | Resistor_SMD:R_0603_1608Metric | C482818 | 1 |
| R15 | 7.68k 1% | USB-PD Eingang | U6 UVLO/OVP-Teiler Mitte, Startwert | Resistor_SMD:R_0603_1608Metric | C163419 | 1 |
| R16 | 20.0k 1% | USB-PD Eingang | U6 UVLO/OVP-Teiler unten, Startwert | Resistor_SMD:R_0603_1608Metric | C844676 | 1 |
| R17 | 18k 1% | USB-PD Eingang | U6 ILIM-Widerstand, Startwert | Resistor_SMD:R_0603_1608Metric | C114610 | 1 |
| C14 | 1uF X7R 25V | USB-PD Eingang | CH224K VDD-Stuetzung | Capacitor_SMD:C_0603_1608Metric | C29936 | 1 |
| C15 | 100nF X7R 50V | USB-PD Eingang | TPD2S300 VBIAS-Stuetzung | Capacitor_SMD:C_0603_1608Metric | C14663 | 1 |
| D2 | KT-0603G gruen | USB-PD Eingang | Gruene 0603 15V-PD-Status-LED, A=2/K=1 | USB-PD-2-C64-Adapter:KT-0603G_C12624 | C12624 | 1 |
| J1 | 2-pin 2.54mm DNP | USB-PD Eingang | Labor-/Testeingang, nicht normal bestueckt | Connector_PinHeader_2.54mm:PinHeader_1x02_P2.54mm_Vertical | DNP | 1 |
| D1 | SMBJ18A | USB-PD Eingang | Eingangs-TVS 18V VRWM, 600W | Diode_SMD:D_SMB | C353379 | 1 |
| FH1 | XFCN XF-155 | USB-PD Eingang | 2410/6125 SMD-Sicherungshalter (hält F1) | USB-PD-2-C64-Adapter:XFCN_XF-155_2410_FuseHolder | C41371860 | 1 |
| F1 | JFC2410-1400TS | USB-PD Eingang | 4A/250V 2410-Sicherung | USB-PD-2-C64-Adapter:XFCN_XF-155_2410_FuseHolder | C136386 | 1 |
| C1 | 47uF | USB-PD Eingang | Eingangs-Bulk Polymer | Capacitor_SMD:CP_Elec_6.3x5.8 | C494513 | 1 |
| C2, C3 | 10uF X7R | USB-PD Eingang | Eingangs-MLCC an VIN | Capacitor_SMD:C_1210_3225Metric | C138687 | 2 |
| U1 | AP64500SP-13 | 5VDC | Synchron-Buck 3.8-40V/5A | USB-PD-2-C64-Adapter:SOIC-8-1EP_3.9x4.9mm_P1.27mm_EP2.62x3.51mm_ThermalVias | C2070920 | 1 |
| U3 | TPS3700DDCR | 5VDC | Fenster-Komparator OVP (400mV Ref, Dual-OD) | Package_TO_SOT_SMD:SOT-23-6 | C33002 | 1 |
| L1 | 2.2uH Sunlord MWSA0603S | 5VDC | Buck-Induktor (Isat 10A, IRMS 9.5A, DCR 15mΩ) | Inductor_SMD:L_Sunlord_MWSA0603S | C408445 | 1 |
| C4 | 100nF | 5VDC | Bootstrap BST-SW | Capacitor_SMD:C_0603_1608Metric | C14663 | 1 |
| C5 | 1.5nF | 5VDC | Kompensation COMP | Capacitor_SMD:C_0603_1608Metric | C710892 | 1 |
| C6, C7, C8, C9, C13 | 22uF X7R | 5VDC | Ausgangs-MLCC | Capacitor_SMD:C_1210_3225Metric | C2918511 | 5 |
| C10 | 220uF | 5VDC | Ausgangs-Bulk Polymer | Capacitor_SMD:CP_Elec_6.3x7.7 | C46550461 | 1 |
| C11 | 100nF | 5VDC | Entkopplung | Capacitor_SMD:C_0603_1608Metric | C14663 | 1 |
| C12 | 1nF | 5VDC | Filter/Feedforward | Capacitor_SMD:C_0603_1608Metric | C507408 | 1 |
| R1 | 66.5k | 5VDC | RT (fSW ~1.5MHz) | Resistor_SMD:R_0603_1608Metric | C218113 | 1 |
| R2 | 39.2k | 5VDC | Kompensation/Bias | Resistor_SMD:R_0603_1608Metric | C217999 | 1 |
| R3 | 105k 0.1% | 5VDC | FB oben | Resistor_SMD:R_0603_1608Metric | C667149 | 1 |
| R4 | 20.0k 0.1% | 5VDC | FB unten, Vout = 5.000V | Resistor_SMD:R_0603_1608Metric | C844676 | 1 |
| R5 | 124k 0.1% | 5VDC | OVP oben (TPS3700) | Resistor_SMD:R_0603_1608Metric | C861097 | 1 |
| R6 | 10k 0.1% | 5VDC | OVP unten, Auslösung ~5.36V | Resistor_SMD:R_0603_1608Metric | C844890 | 1 |
| R7 | 10mR 1% 3W SHUNT | 5VDC | 5V-Lastmess-Shunt fuer INA180A2, Kelvin-nah anbinden | Resistor_SMD:R_2512_6332Metric | C5375464 | 1 |
| U9 | INA180A2IDBVR | 5VDC | High-Side-Current-Sense-Verstaerker 50V/V fuer 5V-Lastdetektion am ATtiny PA1/ADC | Package_TO_SOT_SMD:SOT-23-5 | C192764 | 1 |
| C26 | 100nF X7R 50V | 5VDC | Lokaler U9/INA180A2 V+-Abblockkondensator | Capacitor_SMD:C_0603_1608Metric | C14663 | 1 |
| J2 | HOOYA DIN-704-M10 C64 DIN-7 | 5VDC | DIN-7 Ausgang: 1 GND_1, 2 GND_2, 3 GND_3, 4 5VDC_4, 5 5VDC_5, 6 9VAC_6, 7 9VAC_7 | USB-PD-2-C64-Adapter:DIN-704-M10_C20611695 | C20611695 | 1 |
| U7 | DRV8256PPWPR | 9VAC | Differentieller 9VAC-H-Brueckentreiber aus VIN_BUCK | Package_SO:Texas_PWP0028V_TSSOP-28-1EP_4.4x9.7mm_P0.65mm_EP3.4x9.7mm_Mask2.94x5.62mm_ThermalVias | C3681352 | 1 |
| U8 | ATTINY412-SSNR | 9VAC | ATtiny412-Controller fuer Sinus-PWM, nSLEEP-Sequenz und 5V-Lastdetektion | Package_SO:SOIC-8_3.9x4.9mm_P1.27mm | C1337190 | 1 |
| C17 | 100nF 50V | 9VAC | DRV8256 VM lokaler Keramik-Abblockkondensator | Capacitor_SMD:C_0603_1608Metric | C14663 | 1 |
| C18 | 1uF 50V | 9VAC | DRV8256 VM lokaler Keramik-Abblockkondensator | Capacitor_SMD:C_0805_2012Metric | C28323 | 1 |
| C19 | 10uF 50V | 9VAC | DRV8256 VM Eingangskeramik nahe H-Bruecke | Capacitor_SMD:C_1210_3225Metric | C138687 | 1 |
| C20 | 100uF 35V | 9VAC | DRV8256 lokaler VIN_BUCK-Bulk, Startwert | Capacitor_SMD:CP_Elec_6.3x7.7 | C19270709 | 1 |
| C21 | 220nF X7R 50V | 9VAC | DRV8256 VCP-nach-VM Charge-Pump-Kondensator | Capacitor_SMD:C_0603_1608Metric | C237192 | 1 |
| C22 | 22nF X7R 50V | 9VAC | DRV8256 CPH-nach-CPL Charge-Pump-Kondensator | Capacitor_SMD:C_0603_1608Metric | C106222 | 1 |
| C23 | 1uF X7R 25V | 9VAC | DRV8256 DVDD-Regler-Abblockkondensator | Capacitor_SMD:C_0603_1608Metric | C29936 | 1 |
| R18, R19 | 33R | 9VAC | DRV8256 IN1/IN2 Serienwiderstaende vom ATtiny-PWM, Startwert | Resistor_SMD:R_0603_1608Metric | C23140 | 2 |
| R20 | 100k | 9VAC | DRV8256 nSLEEP-Pulldown | Resistor_SMD:R_0603_1608Metric | C14675 | 1 |
| R21 | 10k | 9VAC | DRV8256 nFAULT Pull-up | Resistor_SMD:R_0603_1608Metric | C844890 | 1 |
| R22, R23 | 10k 1% | 9VAC | DRV8256 VREF-Teiler, Startwert ITRIP ca. 3.8A Peak | Resistor_SMD:R_0603_1608Metric | C844890 | 2 |
| L2, L3 | 47uH 5A START | 9VAC | Differenzielle LC-Ausgangsfilter-Drosseln, Mess-Startwert | Inductor_SMD:L_10.4x10.4_H4.8 | C7529419 | 2 |
| C24 | 1uF X7R 50V | 9VAC | Differentieller LC-Filter-Kondensator zwischen 9VAC_6 und 9VAC_7, Startwert | Capacitor_SMD:C_1210_3225Metric | C445699 | 1 |
| C25 | 100nF X7R 50V | 9VAC | Lokaler U8/ATtiny412 VCC-Abblockkondensator | Capacitor_SMD:C_0603_1608Metric | C14663 | 1 |
| C27 | 1uF X7R 50V DNP | 9VAC | Optionaler paralleler differentieller LC-Filter-Tuning-Kondensator zwischen 9VAC_6 und 9VAC_7 | Capacitor_SMD:C_1210_3225Metric | DNP | 1 |
| J4 | 50 / 60Hz DNP | 9VAC | 2-poliger 2.54mm-Konfigurationsheader fuer Frequenzauswahl, nicht normal bestueckt | Connector_PinHeader_2.54mm:PinHeader_1x02_P2.54mm_Vertical | DNP | 1 |
| J5 | UPDI DNP | 9VAC | 3-poliger 2.54mm-UPDI-Programmierheader, nicht normal bestueckt | Connector_PinHeader_2.54mm:PinHeader_1x03_P2.54mm_Vertical | DNP | 1 |
| JP1 | Decay DNP | 9VAC | DRV8256-DECAY-Solder-Jumper, gebrueckt 1-2, kein bestuecktes Bauteil | Jumper:SolderJumper-3_P1.3mm_Bridged12_RoundedPad1.0x1.5mm | DNP | 1 |
| U10 | SN74LVC2G132DCTR | 9VAC | Dual-Schmitt-NAND fuer verzoegerten DRV8256-nFAULT-Latch, aktuell platziert aber unverdrahtet | Package_SO:VSSOP-8_3x3mm_P0.65mm | C2870897 | 1 |
| Q1 | 2N7002DW-7-F | 9VAC | Dual-NMOS fuer gelatchten nSLEEP-Pulldown und rote Fault-LED-Senke, aktuell unverdrahtet | Package_TO_SOT_SMD:SOT-363_SC-70-6 | C83571 | 1 |
| D3 | KT-0603R rot | 9VAC | Rote 0603-Fault-LED, A=2/K=1, aktuell unverdrahtet | LED_SMD:LED_0603_1608Metric | C2286 | 1 |
| D4 | 1N4148W | 9VAC | Rueckladediode fuer nFAULT-Verzoegerungskondensator nach Fault-Ende, aktuell unverdrahtet | Diode_SMD:D_SOD-123 | C81598 | 1 |
| R24, R28 | 4.7k 1% | 9VAC | PA2-Serienwiderstand und Fault-LED-Vorwiderstand, aktuell unverdrahtet | Resistor_SMD:R_0603_1608Metric | C23162 | 2 |
| R25, R26 | 100k 1% | 9VAC | nFAULT-Verzoegerungswiderstand und Reset-Pull-up, aktuell unverdrahtet | Resistor_SMD:R_0603_1608Metric | C14675 | 2 |
| R27 | 1k 1% | 9VAC | Gate-Widerstand vom Latch-Ausgang zu Q1, aktuell unverdrahtet | Resistor_SMD:R_0603_1608Metric | C21190 | 1 |
| C28 | 1uF X7R 25V | 9VAC | nFAULT-Verzoegerungskondensator, aktuell unverdrahtet | Capacitor_SMD:C_0603_1608Metric | C29936 | 1 |
| C29 | 100nF X7R 50V | 9VAC | Power-on-Reset-Kondensator fuer Fault-Latch, aktuell unverdrahtet | Capacitor_SMD:C_0603_1608Metric | C14663 | 1 |

**J2-Status:** J2 ist elektrisch als DIN-7-C64-Ausgang mit den Pin-/Padbezeichnungen 1 `GND_1`, 2 `GND_2`, 3 `GND_3`, 4 `5VDC_4`, 5 `5VDC_5`, 6 `9VAC_6` und 7 `9VAC_7` modelliert und auf den PCB-montierten HOOYA DIN-704-M10/C20611695 als Startkandidat festgelegt. Die LCSC/EasyEDA-Padtopologie passt zum C64-Wiki-Hinweis: Pin 1 liegt zwischen Pin 4 und 6, Pin 3 liegt zwischen Pin 5 und 7. Der projektlokale Footprint `DIN-704-M10_C20611695` ist trotzdem vor PCB-Freigabe gegen das reale Bauteil, Herstellerzeichnung, Außenansicht der C64-Buchse, Gehaeuseabstand und Stromtragfaehigkeit zu pruefen. Das 3D-Modell `CONN-TH_DIN-704-M10.step` wurde aus dem EasyEDA-Modell per easyeda2kicad erzeugt und im aktuellen Footprint mit Scale 0.8 eingehängt, weil der EasyEDA-Export gegenüber dem vorhandenen projektlokalen Footprint um Faktor 1.25 größer ist; diese Maßdifferenz ist bei der realen Footprint-Freigabe mitzuprüfen.

**Hinweis DRV8256P 9VAC-Leistungszweig:** U7 ist **DRV8256PPWPR/C3681352** als differenzieller H-Brueckentreiber fuer den synthetischen 9VAC-Ausgang zwischen `9VAC_6` und `9VAC_7`. Die DRV8256-Datenblattbeschaltung ist als Mindestbasis umgesetzt: VM lokale Keramikstuetzung C17/C18/C19 plus Bulk C20, VCP-VM mit C21 220nF, CPH-CPL mit C22 22nF, DVDD-GND mit C23 1uF, nFAULT-Pull-up R21 und IN1/IN2-Serienwiderstaende R18/R19. DVDD ist ein Ausgang des DRV8256-Reglers und bekommt keine externe Eingangsspannung. Digitale Dauer-High-Signale wie DECAY duerfen bevorzugt auf DVDD gelegt werden; nSLEEP darf nicht an DVDD haengen, sondern muss vom ATtiny gesteuert werden.

**Hinweis 9VAC-LC-Filter L2/L3/C24/C27:** L2 und L3 sind als bestueckte **47uH/5A-Startwerte, SHOU HAN CYA1050-47UH, LCSC/JLC C7529419** vorgesehen. C24 ist als bestückter Startwert mit **1uF X7R 50V 1210, TDK C3225X7R1H105KT0A0N, JLCPCB/LCSC C445699** zwischen `9VAC_6` und `9VAC_7` vorgesehen. C27 ist parallel dazu als identische optionale Tuning-Position angelegt, aber **DNP**; erste Bestückung daher 1uF differentiell, spätere Messvariante mit C27 bestückt ergibt 2uF. Kein Kondensator des 9VAC-Ausgangs darf nach Board-GND geführt werden. Werte sind nicht final: THD, PWM-Ripple, Leerlaufspannung <=10.3VAC RMS, 2A-Dauerlast, Induktor-Saettigung, Temperatur und Lastsprung muessen am Prototyp verifiziert werden.

**Hinweis DRV8256 VREF R22/R23:** R22/R23 sind als 10k/10k-Startwert von DVDD nach GND gesetzt. Damit liegt VREF grob bei 2.5V; nach DRV8256-Datenblatt gilt fuer VREF <=3.3V ungefaehr `ITRIP = VREF / 0.66`, also ca. 3.8A Peak. Das ist ein erster Schutz-/Regel-Startwert oberhalb 2A RMS und muss zusammen mit Filter, Sinus-PWM, thermischer Last und C64-9VAC-Verbrauch vermessen werden.

**Hinweis nSLEEP/9VAC-Abschaltung:** U8 ist **ATTINY412-SSNR/C1337190** und laeuft aus der 5V-Schiene. PA1/ADC liest den INA180A2-Ausgang der 5V-Lastmessung; PA2 steuert ueber `PA2_DRV_NSLEEP` den DRV8256-nSLEEP-Eingang. Die Firmware-/Logiksequenz soll nSLEEP erst high setzen, wenn 5V stabil ist und eine echte 5V-Last oberhalb der noch zu kalibrierenden Schwelle erkannt wird. Bei keiner 5V-Last, Fault, Unterspannung oder Abschalten muss PA2 nSLEEP low ziehen, damit der 9VAC-Zweig aus bleibt.

**Hinweis 9VAC-Schutzstatus:** Ein separater Ausgangsschutz fuer `9VAC_6`/`9VAC_7` ist noch nicht final festgelegt. Der aktuelle Schutzpfad besteht aus zentraler U6-15V-eFuse, DRV8256-Stromregelung/OCP/TSD, VREF-Begrenzung, ATtiny-Abschaltung und dem ausgewaehlten lokalen DRV8256-nFAULT-Latch aus U10/Q1/D3/D4/R24-R28/C28/C29. Q1 ist ein 2N7002DW-7-F Dual-NMOS im SOT-363; ein interner FET zieht `DRV_NSLEEP` dominant low, der andere FET schaltet die rote Fault-LED. Ziel ist: ein anhaltender DRV8256-nFAULT setzt nach RC-Verzoegerung einen Latch, Q1 zieht `DRV_NSLEEP` dominant low und D3 zeigt den Fehler an. Der Latch laeuft aus `CH224_VDD_3V3`, nicht aus DRV8256-DVDD. Nicht direkt mit `EFUSE_SHDN` verbinden, damit ein lokaler 9VAC-Fehler nicht unnoetig die 5V-Schiene abschaltet. Keiner der beiden 9VAC-Ausgaenge darf nach Board-GND abgesichert oder geklemmt werden, wenn dadurch der differentielle/floating-artige Ausgang verletzt wird.

**Hinweis 5V-Lastdetektion U9/R7/C26:** U9 ist **INA180A2IDBVR/C192764** mit Pinout A und Gain 50V/V. R7 ist **10mR 1% 3W 2512/C5375464** in Serie zwischen `5V_PROTECTED_SENSE` und dem C64-5V-Ausgang zu J2 Pin 4/5. U9-IN+ liegt auf der Regler-/Schutzseite des Shunts, U9-IN- auf der DIN-/Lastseite; U9-OUT geht auf `PA1/ADC` des ATtiny. Bei 10mR und Gain 50V/V gilt `V_ADC = I_LOAD * 0.5V/A`, also ca. 50mV bei 100mA, 250mV bei 500mA und 1.5V bei 3A. C26 ist der lokale 100nF-Abblockkondensator fuer U9-V+ nach GND. Die Shunt-Sense-Leitungen muessen im PCB Kelvin-nah direkt an den R7-Pads abgegriffen werden.

**Footprint-/3D-Hinweis USB-PD-Eingang:** `TYPE-C16PIN_C393939` ist ein projektlokaler Startfootprint mit lokalem STEP-Modell. `CH224K_ESSOP-10-150mil-1mm` ist ein projektlokaler Footprint fuer CH224K ESSOP10/C970725, gegen `Datasheet/CH224DS1.pdf` geprüft: 3.9mm Body-Breite, ca. 5.0mm Body-Laenge, 6.0mm Gesamtbreite mit Pins und 1.00mm Pitch; als 3D-Modell wird das KiCad-Standardmodell `SSOP-10-1EP_3.9x4.9mm_P1mm_EP2.1x3.3mm.step` verwendet, nicht das fruehere falsche TSSOP-10-3x3mm/0.5mm-Modell. `TPD2S300YFFR_DSBGA-9` ist ein projektlokaler Footprint fuer TI YFF/DSBGA-9 mit 0.4mm Pitch, 0.23mm Pads und lokalem VRML-3D-Modell `TPD2S300YFFR_DSBGA-9.wrl`. JLCPCB-PCBA nennt Fine-Pitch/BGA-Bestueckung ab >=0.35mm Pitch; die 0.4mm-YFF-Bestueckung ist daher grundsaetzlich plausibel, aber das Routing ist eng: zwischen 0.23mm-Pads bleiben nur ca. 0.17mm Kupferabstand, deshalb U5 mit feinen Design-Rules, kurzen Top-Layer-Ausbruechen, sauberer Loetstoppmaske/Paste und JLC-DFM-Check pruefen. Vor PCB-Freigabe müssen Padgeometrie, Gehäuseumriss, Höhenmodell und Lötmasken/Paste mit den Herstellerdatenblättern geprüft werden. Die USB-PD-Teile wurden initial bewusst unverdrahtet platziert; die CC-Schutzverdrahtung muss weiterhin manuell/gezielt geprueft werden.

**Hinweis USB-C/CC-Schutz U5:** U5 ist jetzt **TPD2S300YFFR/C2650411** statt USBLC6-2SC6. Grund: USBLC6 ist ein 5V-USB2-ESD-Baustein und sein VBUS-Referenzpin darf nicht an den 15V-PD-VBUS. TPD2S300 ist ein aktiver USB-C-CC-Schutz mit Short-to-VBUS-Isolation und IEC-ESD-Schutz. Anschlusspunkte: `C_CC1/C_CC2` an die USB-C-Buchse, `CC1/CC2` an CH224K, `GND` an Board-GND, `VBIAS` an C15 100nF/50V nach GND, `VPWR` und `VM` an die CH224K-`VDD`-Schiene. CH224K-`VDD` ist laut Datenblatt beim CH224K ein interner 3.3V-Shunt-Regler mit 3.24V bis 3.36V und bis 30mA Parallel-Sink-Faehigkeit; TPD2S300 liegt mit Mikroampere-Versorgung deutlich darunter. `FLT` ist Open-Drain und kann optional an eine spaetere Diagnose/PG-Logik, sonst offen lassen. DSBGA-9 ist nicht handloetfreundlich; fuer Handaufbau waere ein rein passiver TVS einfacher, schuetzt aber nicht gleichwertig gegen CC-Short-to-VBUS. Layout laut TI: Baustein so nah wie moeglich an die USB-C-Buchse, `C_CC1/C_CC2` ungeschuetzt kurz und gerade fuehren, scharfe Ecken vermeiden, VBIAS/VPWR/VM-Kondensatoren sehr nah und an solide Masse anbinden.

**Hinweis CH224K-PG-LED D2/R10:** CH224K-`PG` ist laut `CH224DS1.pdf` ein Open-Drain-Ausgang fuer PowerGood, aktiv low. Die Status-LED ist deshalb als Sink-Pfad verschaltet: `CH224_VDD_3V3 -> R10 -> D2 Anode (Pin 2) -> D2 Kathode (Pin 1) -> U4 PG (Pin 10)`. D2 verwendet eine projektlokale Kopie des KiCad-Standardsymbols `LED_Small` mit `Sim.Pins` = `1=K 2=A`; das alte selbstgezeichnete KT-0603G-Symbol wird nicht mehr verwendet, weil Grafik und A/K-Anordnung missverstaendlich waren. In KiCad muss `PG` an D2-Kathode/Pin 1 liegen; eine Verbindung von `PG` zur D2-Anode waere falsch gepolt. R10 ist als **470R 1% / 0603 / JLC Basic / C23179** festgelegt; mit KT-0603G/C12624 (Vf 3.1V typ bei 5mA, 430mcd) ergibt das bei 3.3V typ. ca. 0.43mA und bei niedrigerer realer Vf grob 0.6mA bis 0.85mA, also schwach bis normal sichtbar ohne nennenswerte Last fuer CH224_VDD/PG.

**Designator-Mapping zur Erzähl-/Konzeptbeschreibung weiter unten:** FB-Teiler = R3 (oben) / R4 (unten); RT-Widerstand = R1; OVP-Teiler = R5 (oben) / R6 (unten); Kompensations-C = C5; Bootstrap-C = C4. Überspannungsschutz ist final als **U3 TPS3700DDCR Fenster-Komparator** umgesetzt, nicht mehr über die früher skizzierte TLV431/LMV431+Treiber-Variante. Hinweis: Die früheren AP64500-Startwerte im Fließtext (FB 115.8k/22.1k) sind durch die verbauten 105k/20.0k ersetzt (exakt 5.000V bei Vref 0.8V).

**Hinweis Überspannungsschutz U3:** TPS3700DDCR liefert eine präzise 400mV-Referenz mit Fenster-Komparator und zwei Open-Drain-Ausgängen; mit R5/R6 = 124k/10k löst die OVP bei ca. 5.36V aus und liegt damit im geforderten Fenster 5.35-5.45V. Eine externe Crowbar/SCR-Option mit vorgeschaltetem Abschaltelement bleibt optional.

**Hinweis zentrale EingangseFuse U6:** U6 ist als TPS16630PWPR/C1849461-Symbol mit TI-PWP/HTSSOP-20-Pinout im Schaltplan eingebettet. Die USB-C-VBUS-Pins von J3 liegen direkt auf F1-Pin 1; `VBUS` ist das abgesicherte Netz hinter F1 und verbindet F1-Pin 2, U4-VBUS/R9, R14 und U6-IN 1/2/3 plus P_IN 6. `VIN_BUCK` liegt auf U6-OUT 18/19/20 und versorgt den AP64500-Eingang sowie den oberen R11-Anschluss des AP64500-EN/UVLO-Teilers. Der Teiler ist `VIN_BUCK -> R11 -> BUCK_EN -> R12 -> GND`, sodass der Buck erst von der geschalteten U6-Ausgangsschiene freigegeben wird. GND 9 und PowerPAD 21 liegen auf Board-GND. `dVdT` ist nicht mehr hart mit GND verbunden, sondern ueber C16 22nF/C106222 nach GND gefuehrt. `SHDN` liegt auf `EFUSE_SHDN`; U3-`OUTB` kann diesen Knoten low ziehen, R13 zieht ihn auf `CH224_VDD` hoch. `UVLO`/`OVP` werden vom R14/R15/R16-Teiler an `VBUS` versorgt; Startziel ist ungefaehr Einschalten nur oberhalb 12V und Abschalten unterhalb 20V-PD. R17 18k/C114610 ist der ILIM-Startwert. `MODE` ist bewusst offen, weil TPS1663 laut Datenblatt damit nach begrenzter Current-Limit-Zeit latch-off ausloest; Reset erfolgt per SHDN-Toggle oder Power-Cycle. `IMON`, `PGOOD` und `FLT` sind optional noch offen. Vor PCB-Freigabe TPS16630PWPR-Verfuegbarkeit und Werte erneut pruefen.

**Veraltete RT6318C-Ursprungs-BOM (NICHT mehr verwenden, nur Historie):** RT6318CGQUF C6288609, 1µH C15949, 10µF C440198, 47µF C178882, 22µF C45783, 150µF C12530, Post-LC 1.5µH/220µF, Shunt 0.03Ω C25110, SCR BT151 C2563, Z-Diode 5.6V C31673, Gate-R 1kΩ C17513. Diese Teile sind durch die obige AP64500-BOM ersetzt.

## Design-Vorgaben

### Elektrische Spezifikationen

- **5V Zweig:**

  - Ausgangsspannung: praktisches Ziel **5.00V am C64-DIN-Stecker unter Last, ideal ±1%**. Der C64 funktioniert typischerweise nur sauber im Bereich **4.95V bis 5.05V**; ±3% ist daher nicht mehr das Qualitätsziel.
  - Ausgangsstrom: 3A kontinuierlich als C64-Zielwert; Regler thermisch mit Reserve auslegen
  - Eingangsspannung: **15V/3A-PD fest vorgesehen und einzige Energiequelle**; 12V ist für 9VAC/2A zu knapp, 20V-PD ist nicht mehr Zielbetrieb, ein Trafo- oder Zweitnetzteilpfad ist ausgeschlossen
  - Ripple: <40mV pp als Designziel; <50mV pp absolute Obergrenze nach C64-Vorgabe
- **9V AC Zweig:**

  - Ausgangsspannung: 9V RMS
  - Leerlaufspannung: maximal **10.3VAC RMS** (15% über 9VAC Nennspannung)
  - Frequenz: 50Hz oder 60Hz (konfiguierbar)
  - Ausgangsstrom: 2A kontinuierlich
  - Wellenform: Sinus (THD zu verifizieren)
  - Pflichtabschaltung: **9VAC muss abgeschaltet werden, wenn keine 5V-Last entnommen wird**; diese Erkennung/Logik ist Teil der 9VAC-Architektur.
  - Leistungs-/Headroom-Prüfung: 9VAC/2A entspricht bis zu 18VA; synthetische Sinuserzeugung aus 12V DC ist knapp, da 9Vrms ca. 12.7V Peak plus Verluste benötigt. 15V DC aus USB-C-PD kann für eine Vollbrücken-Sinusendstufe reichen, muss aber mit Last, Filterverlusten, Modulationsreserve und Eingangsspannungstoleranz simuliert/gemessen werden. Ein Trafo- oder Zweitnetzteilpfad ist keine zulässige Projektoption.

### Schutzfunktionen

- **Crowbar-Schaltung:** SCR/eFuse-Schutz mit präzisem Triggerziel ca. 5.35V bis 5.45V; einfache 5.6V-Zener-Schaltung nur als früher Entwurf betrachten.
- **TVS-Diode:** Eingangsschutz gegen Transienten; mit BD9P608MFF-C sind 40V VIN deutlich entspannter als beim RT6318C, trotzdem TVS, Sicherung/eFuse und reale Klemmspannung passend zum gewählten Eingangskonzept auslegen.
- **Post-LC-Filter:** Zusätzliche Glättung der Ausgangsspannung nur mit Stabilitäts-/Lastsprungprüfung und geeigneter Dämpfung.
- **Kontrolliertes Ein-/Ausschalten:** C64 vor Rückstrom, Netzteil-Ausfall und schnellem Aus-/Einschalten schützen; Mindest-Auszeit und sauberes Trennen der Versorgungsleitungen prüfen.

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
- Pflichtspalte `Baugruppe` mit genau diesen Kategorien: `USB-PD Eingang`, `5VDC`, `9VAC`.
- **Bei jeder Schaltplan-Änderung BOM aktualisieren!**

## Offene Fragen / TODOs

### Best Practices: Hochwertige C64-Stromversorgung

#### 5V DC Zweig

- [ ] **Spannungsregelung:** Präzise 5.00V am C64-DIN-Stecker unter Last, ideal ±1%; typischer sauberer C64-Betriebsbereich 4.95V bis 5.05V, Zielbereich 0-3A plus Reserve
- [ ] **Ripple minimieren:** <40mV pp am C64-DIN-Stecker als Zielwert erreichen; <50mV pp ist nur die absolute Obergrenze
- [ ] **Soft-Start:** Einschaltstrom begrenzen, um Spannungsspitzen zu vermeiden
- [ ] **Überspannungsschutz:** Crowbar-Schaltung MUSS bei >5.4V auslösen (VIC-II, SID empfindlich!)
- [ ] **Thermisches Design:** Buck-Converter bei 3A Dauerlast mit Reserve <80°C halten; 5A nur als Stress-/Kurzzeittest betrachten
- [ ] **EMV:** Saubere Gleichspannung möglichst an der Quelle erreichen; Post-LC-Filter nur einsetzen, wenn Stabilität und Lastsprungverhalten nachgewiesen sind
- [ ] **Power Sequencing:** C64-Aus, Netzteil-Ausfall und schnelles Wiedereinschalten erkennen; Mindest-Auszeit zur Entladung von C64-/Erweiterungs-Kondensatoren vorsehen

#### 9V AC Zweig

- [ ] **Sinus-Reinheit:** THD <5% für CIA-Timer-Kompatibilität
- [ ] **Leerlaufspannung:** 9VAC im Leerlauf maximal 10.3VAC RMS (15% über Nennspannung)
- [ ] **Frequenzstabilität:** 50Hz oder 60Hz ±1% (Quarz-gesteuert empfohlen)
- [ ] **Strombelastbarkeit:** 2A kontinuierlich ohne Überhitzung
- [ ] **9VAC-Abschaltung:** 9VAC muss abgeschaltet werden, wenn keine 5V-Last entnommen wird
- [ ] **USB-PD-only:** 9VAC ausschließlich aus der USB-C-PD-Versorgung erzeugen; kein Trafo- oder Zweitnetzteilpfad

#### Allgemeine Schutzfunktionen

- [ ] **Überstromschutz:** Strombegrenzung bei Kurzschluss
- [ ] **Thermische Abschaltung:** Bei Übertemperatur automatisch abschalten
- [ ] **Verpolungsschutz:** USB-C ist symmetrisch, aber DIN-7 Ausgang absichern
- [ ] **Einschaltsequenz:** 5V vor 9V AC, um Einschaltstörungen zu minimieren

### Fertigung

- [ ] PCB Review vor Bestellung
- [ ] Thermische Analyse (Buck Converter, Inverter)
- [ ] EMV-Überlegungen

### Mechanik

- [ ] Gehäuse-Design (3D-Druck)
- [ ] Kabelführung USB-C ↔ DIN-7
