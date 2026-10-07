# BCI MindFlex Car - Gesamte Vorgehensweise

Vollständige Dokumentation des BCI-Projekts: Von der Hardware-Integration über Datenerfassung bis zur KI-gesteuerten Fahrzeugkontrolle. Ein Python-basiertes System, das EEG-Signale eines MindFlex-Headsets in Echtzeit klassifiziert und daraus Fahrbefehle für ein RC-Auto ableitet.

---

## 1. Projektübersicht & Ziele

### Was ist das Projekt?

Das BCI MindFlex Car Projekt ist ein **Gedankensteuersystem für ein Fahrzeug**:
- Du trägst ein **EEG-Headset** (NeuroSky MindFlex)
- Du denkst an eine Fahrtrichtung (FORWARD, STOP, LEFT, RIGHT, BACKWARD)
- Ein **Machine-Learning-Modell** übersetzt deine Gehirnwellen in Fahrbefehle
- Ein **RC-Auto mit vier Motoren** führt diese Befehle aus

### Systemarchitektur (Datenfluss)

```
MindFlex Headset (EEG-Messungen)
        |
        v
Arduino Uno (ThinkGear-Protokoll, serieller Port)
        |
        v
Python-Anwendung (Klassifikation, GUI)
        |
        +----> Signalvalidierung & Feature-Extraktion
        |
        +----> ML-Klassifikation (Decision Tree / Random Forest / KNN)
        |
        v
BLE-Funkmodul
        |
        v
Arduino Duemilanove (Motorsteuerung)
        |
        v
L293D Shield + 4 Getriebemotoren (RC-Auto)
```

### Warum diese Architektur?

- **Zwei separate Arduinos:** Der Uno liest EEG-Daten, der Duemilanove steuert die Motoren. Das erlaubt unabhängige Fehlerbehandlung — fällt das Auto aus, läuft die Klassifikation trotzdem weiter.
- **BLE statt Klassisches Bluetooth:** Stabil auf macOS 15+, während klassisches SPP (Serial Port Profile) dort anfällig für Verbindungsabbrüche ist.
- **Python statt Arduino-Code für die KI:** Auf dem Mac kann man die ML-Modelle trainieren und debuggen, bevor man sie einsetzt.

---

## 2. Hardware-Setup

### Komponenten

**EEG-Erfassung:**
- NeuroSky MindFlex Headset (misst mit einer Elektrode auf Fp1, Referenz am Ohr)
- Arduino Uno (liest ThinkGear-Protokoll über RX-Pin)
- USB-Kabel oder BLE-Modul (CC2541 / HM-10) für drahtlose Verbindung

**Fahrzeugsteuerung:**
- Arduino Duemilanove (ATMEGA328P-PU)
- L293D Motor Shield (HW-130) für 4 DC-Motoren
- 4 Getriebemotoren (90°-Zahnräder, beliebige Drehzahl)
- 4 AA-Batterien oder LiPo (nicht 9V Block — zu schwach!)
- Bluetooth Low Energy Modul (CC2541/HM-10/AT-09) am Duemilanove

### Verkabelung: EEG-Arduino (Uno)

```
MindFlex Headset  -> RX-Pin (Pin 0)
ThinkGear Chip (aus Headset) -> RX-Pin
      |
CSV-Datenzeile (fertig aufbereitet) -> TX-Pin (Pin 1)
```

**Optional: BLE-Modul statt USB**

Wenn du das Headset funken lassen willst (2. BLE-Modul):

```
Modul VCC    -> 5V
Modul GND    -> GND
Modul RXD    -> Pin 1 (TX) über Spannungsteiler 1 kΩ / 2 kΩ   ⚠️ Zwingend!
```

Der Spannungsteiler ist **notwendig**, weil Pin 1 den Datenstrom direkt empfängt.

### Verkabelung: Auto-Arduino (Duemilanove)

```
HC-05 / HM-10 Modul
    VCC -> 5V oder 3,3V (beide getestet, Verbindung steht)
    GND -> GND
    TXD -> Pin 10 (über SoftwareSerial)
    RXD -> offen (Arduino sendet nichts zurück)

L293D Shield
    M1 -> vorne rechts
    M2 -> vorne links
    M3 -> hinten rechts
    M4 -> hinten links

Alle vier INVERT-Schalter im Sketch auf true (Motoren sind einheitlich gepolt)
```

**Versorgung:** Der Duemilanove hat auch einen 3,3-V-Pin (vom FTDI-Chip FT232RL, max. ca. 50 mA). Das BLE-Modul (ca. 10 mA) läuft daran ebenfalls, getestet am 2026-09-29.

**Wichtig:** Pin 11 darf NICHT verwendet werden (PWM für Motor 1), deshalb Pin 10 für BLE.

---

## 3. Komponenten-Auswahl & Begründung

Dieses Kapitel erklärt, **warum** wir diese Hardware-Komponenten gewählt haben und welche **Vorteile** sie bringen — im Vergleich zu Alternativen.

### 3.1 EEG-Headset: NeuroSky MindFlex

#### Warum MindFlex?

**Das MindFlex ist ein Spielzeug-Headset** — das war Absicht! Es war die verfügbare und kostengünstige Lösung für Schüler:innen im Seminarfach.

| Kriterium | MindFlex | Professionell (z.B. Muse, Emotiv) | Forschung (z.B. BioSemi) |
|---|---|---|---|
| **Kosten** | €80–120 | €300–500 | €10.000+ |
| **Kanäle** | 1 (Fp1) | 4–32 | 64–256 |
| **Auflösung** | ??? (proprietär) | 14–16 Bit | 24 Bit |
| **Sampling-Rate** | ~ 512 Hz (internal) | 256–500 Hz | 2000 Hz |
| **ThinkGear API** | Ja (CSV) | Nein, eigene Protokolle | Nein |
| **Praxistauglichkeit** | Eher spielerisch | Professionell | Forschung |

#### Vorteile des MindFlex

1. **Einfaches Protokoll (ThinkGear)**
   - Das Headset sendet **aufbereitete EEG-Bänder** (Delta, Theta, Alpha, Beta, Gamma), nicht Rohdaten
   - Intern läuft ein ThinkGear-Chip (Infineon), der das Rohsignal schon filtert und analysiert
   - Ausgang ist einfaches CSV: `SignalQuality,Attention,Meditation,Delta,...`
   - **Vorteil:** Keine komplexe Signalverarbeitung nötig, einfach auswerten

2. **Komfort**
   - Trockenelektrode (kein Elektrogel nötig, schnell anzulegen)
   - Leicht (ca. 50 Gramm)
   - Ohrclip als Referenzelektrode (eher unauffällig)

3. **Kostengünstig**
   - Für ein Schulprojekt ideal
   - Wenn es kaputt geht, nicht tragisch

4. **Brain Library für Arduino**
   - Arduino-Community hat bereits eine fertige `Brain`-Bibliothek geschrieben
   - Kommunikation mit dem Headset funktioniert Out-of-the-Box

#### Nachteile & Einschränkungen

1. **Nur eine Elektrode (Fp1)**
   - Sitzt auf der Stirn
   - Misst eher Frontal-Aktivität (Aufmerksamkeit, Meditationszustand)
   - **Motorischer Kortex (C3/C4, wo Bewegungen geplant werden) ist nicht erreichbar**
   - → LEFT/RIGHT-Bewegungen sind möglicherweise nicht im Signal sichtbar

2. **Propriätärer ThinkGear-Chip**
   - Keine Rohdaten zugänglich
   - Wir wissen nicht genau, wie die Algorithmen funktionieren
   - Keine Kontrolle über Filterung/Verarbeitung

3. **Trockenelektrode ist anfällig**
   - Haare unter der Elektrode = schlechter Kontakt
   - Schweiß/Bewegung erzeugen Artefakte
   - Signal ist nicht stabil wie bei professionellen Headsets

4. **Signalqualität nicht dokumentiert**
   - Das Headset meldet `SignalQuality`, aber die Skala ist nicht klar (0–200? 0–100?)
   - Threshold `signal_quality == 200` für "kein Signal" wurde empirisch gefunden

#### Alternativen (und warum nicht?)

**Emotiv Epoc (4 Kanäle, ~€300)**
- ✅ Besser als MindFlex (4 Kanäle statt 1)
- ❌ Immer noch Spielzeug-Level
- ❌ Kostet das 3–4-fache
- ❌ Benötigt eigene Software-Stack (unterschiedlich von MindFlex)

**Muse Headband (4 Kanäle, ~€300)**
- ✅ Stabil auf macOS
- ✅ Gute BLE-Integration
- ❌ Kosten
- ❌ Weniger Gemeinschafts-Support als MindFlex

**OpenBCI (8–16 Kanäle, ~€500–800)**
- ✅ Open-Source Hardware
- ✅ Rohdaten verfügbar
- ✅ Gute Dokumentation
- ❌ Sehr teuer
- ❌ Komplexere Setup

**Fazit:** Für ein Schulprojekt mit kleinerem Budget war MindFlex die richtige Wahl.

---

### 3.2 Microcontroller: Arduino Uno & Duemilanove

#### Warum zwei Arduinos?

```
❌ Eine Lösung: Ein Arduino (Uno) für alles
   → Problem: Uno kann Python nicht ausführen
   → Python läuft auf dem Mac, Arduino-Sketch auf der Hardware
   → Wenn wir alles auf dem Uno machen, müssen wir ML-Klassifikation in C++ schreiben
   → Scikit-learn gibt es nicht für Arduino

✅ Zwei Lösungen: Uno (EEG) + Duemilanove (Auto)
   → Uno: Spezialisiert auf einfache Daten-Erfassung (EEG-Protokoll)
   → Duemilanove: Spezialisiert auf Motor-Steuerung
   → Python auf dem Mac: ML, Training, GUI
   → Entkopplung ermöglicht unabhängiges Debuggen
```

#### Arduino Uno: EEG-Erfassung

**Warum Uno?**

| Kriterium | Uno | Nano | Mega |
|---|---|---|---|
| **Größe** | Standard | Klein | Groß |
| **Speicher (Flash)** | 32 KB | 32 KB | 256 KB |
| **Speicher (RAM)** | 2 KB | 2 KB | 8 KB |
| **UART-Ports** | 1 | 1 | 4 |
| **Pin-Count** | 14 (6 PWM) | 14 (6 PWM) | 54 (15 PWM) |
| **Kosten** | €20 | €15 | €35 |
| **Häufigkeit** | Standard | Mobil/embedded | Komplex |

**Vorteile für EEG-Aufgabe:**
1. **Ausreichende Specs:** EEG-Daten sind einfache CSV-Zeilen, brauchen wenig RAM
2. **Brain Library Support:** Community hat fertige Brain-Bibliothek
3. **Standard-Hardware:** Leicht zu debuggen, überall verfügbar
4. **USB-Anschluss:** Direkte Verbindung zum Mac

**Nachteile:**
- Nur 2 KB RAM (nicht viel Platz für Buffer)
- Nur 1 UART (Hardware), aber das reicht für serielle EEG-Daten

**Warum nicht Nano?**
- Spart nur €5, das ist es nicht wert
- Uno ist handlicher auf einem Breadboard

**Warum nicht Mega?**
- Overkill für nur Datenerfassung
- Kostet mehr ohne echten Gewinn

---

#### Arduino Duemilanove: Motor-Steuerung

**Warum Duemilanove (ATMEGA328P)?**

Moment — **das ist eigentlich ungewöhnlich.** Der Duemilanove ist eine ältere Arduino-Variante (2009). Im Projekt wird er verwendet, weil:

1. **Verfügbar:** Er war im Bestand vorhanden
2. **Bootloader-kompatibel:** Funktioniert mit dem AFMotor-Shield (HW-130)
3. **Hardware-UART:** Gut für Motor-Kommunikation

Moderne Alternativen wären:

| Variante | Vorteil | Nachteil |
|---|---|---|
| **Arduino Uno** | Standardversion, überall | Nur 1 Hardware-UART |
| **Arduino Mega** | 4 Hardware-UARTs | Oversized für diese Aufgabe |
| **Arduino Nano** | Klein | Zu eng für Shield |
| **Arduino Pro Mini** | Stromsparen | Keine serielle Upload-Unterstützung |

**Duemilanove als Motor-Steuerung:**

✅ **Vorteile:**
- Genug Pins für L293D-Shield (11, 3, 5, 6, 4, 7, 8, 12 belegt)
- Kompatibel mit AFMotor-Library
- Stabil & bewährt

⚠️ **Nachteile:**
- Nur 1 Hardware-UART (aber genug für SoftwareSerial)
- Ältere Technologie (aber nicht schlecht)

---

### 3.3 Motor-Shield: L293D (HW-130) vs. Alternativen

#### Warum L293D Shield?

Das **L293D ist ein H-Brücken-IC**, das bis zu 4 DC-Motoren steuern kann. Es wird über ein Schieberegister (74HC595) angesteuert.

#### Vergleich Motor-Driver-Optionen

| Shield | Kanäle | Strom/Kanal | Kosten | Komplexität | Stabilität |
|---|---|---|---|---|---|
| **L293D (HW-130)** | 4 | 500 mA | €8–10 | Mittel (Schieberegister) | Gut |
| **L298N-Module** | 2 | 2 A | €5–7 | Einfach (direkt pins) | Gut |
| **DRV8833** | 2 | 1.5 A | €4–6 | Einfach | Sehr gut |
| **TB6612FNG** | 2 | 1.2 A | €3–5 | Einfach | Sehr gut |
| **L298N-Dual** | 4 | 2 A | €12–15 | Mittel | Gut |

#### Vorteile des L293D (HW-130)

1. **Vier unabhängige Kanäle**
   - Perfekt für 4-Rad-Fahrzeug mit separater Motorsteuerung pro Rad
   - Panzer-Lenkung möglich (jede Seite einzeln ansteuern)

2. **Arduino-fertig**
   - AFMotor-Bibliothek existiert bereits
   - Keine Custom-Verkabelung nötig
   - Plug-and-Play (auf Arduino aufgesteckt)

3. **Schieberegister-Steuerung**
   - Verwendet nur 4 Pins für Datenkommunikation (SPI-ähnlich)
   - Spart viele GPIO-Pins (Duemilanove hat nicht so viele)

4. **Stromverteilung**
   - Externe Stromversorgung möglich (Batterien direkt ans Shield)
   - Entlastet die Arduino-Stromversorgung

#### Nachteile

1. **Schieberegister ist komplex**
   - Weniger intuitiv als direkte Pin-Ansteuerung
   - Bei Fehler schwerer zu debuggen

2. **500 mA pro Kanal können grenzwertig sein**
   - Mit 4 Motoren gleichzeitig × 500 mA = 2 A Gesamtstrom nötig
   - Mit 4 AA-Batterien (2000–2500 mAh) sollte es reichen, aber nicht gemütlich
   - Daher: **9V-Block ist zu schwach!** (gibt nur 200–300 mA)

3. **Wärmeverlust**
   - Bei hohem Strom wird das IC warm
   - Kein eingebauter Kühlkörper

#### Alternative: Zwei L298N-Module (2 Kanäle each)

```
❌ Nachteile:
- Zwei separate Module nötig
- Komplexere Verdrahtung
- Mehr Platz nötig
- Höhere Kosten insgesamt

✅ Vorteile:
- Höherer Strom pro Kanal (2 A statt 500 mA)
- Einfachere Pin-Steuerung (EN + IN1/IN2 direkt)
```

**Wir haben uns für L293D entschieden**, weil:
- Platz-Effizienz (ein Shield, nicht zwei Module)
- AFMotor-Bibliothek ist fertig
- Für unser RC-Auto mit 4 AA-Batterien reicht die Stromstärke

---

### 3.4 Gleichstrommotoren: Getriebemotoren

#### Warum Getriebemotoren (mit Zahnrädern)?

```
❌ Gleichstrom-Motor (nackt)
   - Hohe RPM (~3000–10000)
   - Niedriges Drehmoment (wenig Kraft)
   → RC-Auto würde sich totdrehen, keine Zugkraft

✅ Getriebemotor
   - Niedriger RPM durch internes Zahnrad-Verhältnis (~50–500 RPM)
   - Hohes Drehmoment (kann Auto bewegen)
   - Schub ist gleichmäßig
```

#### Spezifikation der gewählten Motoren

**90°-Zahnräder (Standard für RC-Autos):**
- **Typ:** Gleichstrom-Getriebemotor (DC Gear Motor)
- **Spannung:** 3–6 V (oder 6–12 V, je nach Modell)
- **RPM:** 100–500 RPM (abhängig von Getriebe-Verhältnis)
- **Drehmoment:** 0,5–2 Nm (ausreichend für 200 g RC-Auto)

#### Vorteile

1. **Hohe Kraft bei niedriger Drehzahl**
   - Perfekt für Rad-Antrieb (brauchen hohe Zugkraft, nicht hohe RPM)

2. **Kompakt**

3. **Reversierbar**
   - Durch Spannungs-Richtung läuft Motor vorwärts oder rückwärts
   - Ideal für Panzer-Lenkung (Seiten gegeneinander drehen)

4. **Kostengünstig**
   - €5–10 pro Motor

#### Nachteile

1. **Zahnräder können verschleißen**
   - Nach 1000+ Betriebsstunden möglicherweise Spielraum
   - Für Schulprojekt: kein Problem

2. **Reibung durch Getriebe**
   - Etwas Energieverlust (aber unvermeidlich)

#### Motorenanzahl: Warum 4 statt 2?

```
Option A: 2 Motoren (links & rechts)
- ✅ Einfacher (Standard RC-Auto-Setup)
- ❌ Nur 2-Rad-Antrieb (Vorder- oder Hinterrad)
- ❌ Lenkung braucht Servo (kompliziert)

Option B: 4 Motoren (alle Räder)
- ✅ Alle Räder angetrieben (4WD — better traction)
- ✅ Panzer-Lenkung möglich (keine Servo nötig)
- ✅ Symmetrische Kraft auf allen Rädern
- ❌ 2× Stromverbrauch (aber unser Shield hält das)
- ❌ Komplexerer Arduino-Sketch
```

**Wir haben 4 Motoren gewählt**, weil:
1. L293D-Shield hat 4 Kanäle (würde ungenutzt bleiben mit nur 2 Motoren)
2. Panzer-Lenkung braucht keine Servo (weniger Hardware)
3. 4WD ist stabiler auf unebenem Untergrund

---

### 3.5 BLE-Modul: CC2541 / HM-10 statt klassisches Bluetooth

#### Warum BLE (Bluetooth Low Energy)?

```
                    BLE                  Klassisches Bluetooth
┌────────────────────────────────┬──────────────────────────────┐
│ Standard          │ 4.0 LE      │ 2.0 (SPP - Serial Port Pr.) │
│ Stromverbrauch    │ Sehr niedrig│ Höher                       │
│ macOS Support     │ Nativ/stabil│ Anfällig für Abbrüche       │
│ Profile           │ GATT        │ SPP (Serial)                │
│ Linux Support     │ Gut         │ Gut                         │
│ Geräte-Erkennung  │ CoreBluetooth│ /dev/rfcomm               │
│ Python Support    │ bleak       │ pyserial + rfcomm           │
└────────────────────────────────┴──────────────────────────────┘
```

#### Vorteile von BLE

1. **Stabil auf macOS 15 Sequoia**
   - Apple hat klassisches Bluetooth/SPP reduziert
   - BLE ist der empfohlene Standard
   - Keine Verbindungsabbrüche (erleben wir nicht)

2. **Niedriger Stromverbrauch**
   - Ideal für batteriebetriebene Geräte
   - Das Auto läuft auf 4 AA-Batterien — jede Milliampere zählt

3. **CoreBluetooth auf macOS**
   - Native Betriebssystem-API (nicht über `/dev/rfcomm`)
   - Zuverlässiger als klassisches BT

4. **Moderne Hardware**
   - CC2541 / HM-10 sind immer noch aktuell
   - Guter Support in der Maker-Community

#### Nachteile von BLE

1. **Nicht in klassischen Bluetooth-Einstellungen sichtbar**
   - `/dev/cu.Bluetooth-Incoming-Port` erscheint nicht
   - Manuelles Pairing wie bei klassischem BT nicht möglich
   - Python muss über GATT-Charakteristiken sprechen (komplexer)

2. **Mehr Code nötig**
   - Klassisches SPP: `pyserial` genügt
   - BLE: `bleak` (async/await, komplexer API)

3. **Service/Charakteristik-UUIDs kennen müssen**
   - FFE0 (Service), FFE1 (Charakteristik) müssen fest codiert sein
   - Weniger flexibel als SPP

#### Alternative: Klassisches Bluetooth (HC-05)

Würde funktionieren, **aber:**
- ❌ Auf macOS 15 anfällig für Verbindungsabbrüche
- ❌ Höherer Stromverbrauch
- ❌ Weniger zukunftsfähig

**Wir haben BLE gewählt**, weil macOS 15 es braucht.

---

### 3.6 Stromversorgung: 4 AA-Batterien statt 9V-Block

#### Das ursprüngliche Problem

Einkaufsliste sagte: **9V-Block für Motoren**

Realität nach erstem Test:
- ❌ 9V-Block liefert nur 200–300 mA
- ❌ 4 Motoren brauchen zusammen 500 mA–2 A
- ❌ Unter Last bricht die Spannung ein
- ❌ Auto "zuckelt" statt zu fahren

#### Warum 4 AA-Batterien besser sind?

| Aspekt | 9V-Block | 4 AA-Batterien | LiPo |
|---|---|---|---|
| **Spannung** | 9 V | 4,8–6 V (4×1,2–1,5 V) | 7,4 V (2S LiPo) |
| **Kapazität (mAh)** | 300–600 | 2000–3000 | 1000–2000 |
| **Max. Strom** | 200–300 mA | 1000–3000 mA | 3000–5000 mA |
| **Kosten** | €2–3 | €4–6 (Batterien) | €10–15 |
| **Verfügbarkeit** | Überall | Überall | Fachhandel |
| **Sicherheit** | Unkritisch | Unkritisch | Braucht Lader |

#### Vorteile von AA-Batterien

1. **Hohe Stromstärke**
   - 4 AA-Batterien parallel liefern locker 2–3 A
   - Motoren laufen kraftvoll, nicht schwach

2. **Lange Laufzeit**
   - 2000–3000 mAh bei 1–2 A Strom = 1–3 Stunden Fahrtdauer
   - 9V-Block schafft 30–60 Minuten

3. **Preis**
   - AA-Batterien sind überall €0,50–1 pro Stück
   - Wiederaufladbar (NiMH) möglich

4. **Einfacher Austausch**
   - Keine Lader nötig (nur Batteriegehäuse)
   - Im Notfall kann jeder AA-Batterien kaufen

#### Alternative: LiPo-Akku

**Vorteile:**
- ✅ Höhere Energiedichte
- ✅ Bessere Stromstärke
- ✅ Leichter als AA-Batterien

**Nachteile:**
- ❌ Spezieller LiPo-Lader nötig (~€15)
- ❌ Brandgefahr bei unsachgemäßer Behandlung
- ❌ Lagern erfordert Vorsicht (Überlastung, Tiefentladung)
- ❌ Teurer in der Anschaffung

**Für Schulprojekt:** 4 AA-Batterien sind praktischer.

---

### 3.8 Zusammenfassung: Hardware-Entscheidungen

| Komponente | Gewählt | Grund |
|---|---|---|
| **EEG-Headset** | MindFlex | Kostengünstig, einfaches Protokoll, Community-Support |
| **EEG-Arduino** | Uno | Standard, Brain-Library verfügbar |
| **Auto-Arduino** | Duemilanove | Kompatibel mit L293D-Shield |
| **Motor-Shield** | L293D (HW-130) | 4 Kanäle, AFMotor-Library, Platz-effizient |
| **Motoren** | Getriebemotoren | Hohes Drehmoment, Panzer-Lenkung möglich |
| **Motor-Anzahl** | 4 Stück | L293D hat 4 Kanäle, 4WD = stabiler |
| **Funk** | BLE | Stabil auf macOS 15, niedriger Stromverbrauch |
| **Stromversorgung** | 4 AA-Batterien | Genug Strom für Motoren, lange Laufzeit |

**Kernentscheidung:** Alle Komponenten sind aufeinander abgestimmt — kleine, unabhängig funktionierende Einheiten, die zusammen ein robustes System bilden.

---

## 4. Arduino-Sketches

### EEG-Arduino (Uno): `mindflex_eeg.ino`

**Aufgabe:** Liest ThinkGear-Rohsignale vom Headset und gibt eine CSV-Zeile aus

**Code-Struktur:**
```cpp
Brain brain(RX, TX);  // Initialisiert ThinkGear-Modul

void setup() {
    Serial.begin(9600);
}

void loop() {
    if (brain.update()) {  // Neue Daten vom Headset?
        Serial.println(...);  // CSV: SignalQuality,Attention,Meditation,Delta,...
    }
}
```

**CSV-Format:**
```
SignalQuality,Attention,Meditation,Delta,Theta,LowAlpha,HighAlpha,LowBeta,HighBeta,LowGamma,HighGamma
```

**Wichtig:** Headset ausschalten, bevor du den Sketch hochlädst (sonst stört der Datenstrom auf RX den Upload).

**Speicherort:** `~/Documents/Arduino/mindflex_eeg/mindflex_eeg.ino`

**Hochladen:**
```bash
~/.local/bin/arduino-cli compile --fqbn arduino:avr:diecimila:cpu=atmega328 ~/Documents/Arduino/mindflex_eeg/
~/.local/bin/arduino-cli upload -p /dev/cu.usbmodem141011 --fqbn arduino:avr:diecimila:cpu=atmega328 ~/Documents/Arduino/mindflex_eeg/
```

---

### Auto-Arduino (Duemilanove): `rc_car_4wd.ino`

**Aufgabe:** Empfängt Fahrbefehle (`F`, `B`, `L`, `R`, `S`) über BLE/SoftwareSerial und steuert 4 Motoren

**Code-Struktur:**
```cpp
#include <AFMotor.h>

AF_DCMotor motor1(1);  // M1-Klemme
AF_DCMotor motor2(2);  // M2-Klemme
AF_DCMotor motor3(3);  // M3-Klemme
AF_DCMotor motor4(4);  // M4-Klemme

SoftwareSerial ble(10, A0);  // RX=Pin10, TX=A0 (ungenutzt)

void setup() {
    Serial.begin(9600);
    ble.begin(9600);
}

void loop() {
    if (ble.available()) {
        char cmd = ble.read();
        switch(cmd) {
            case 'F': moveForward(); break;
            case 'B': moveBackward(); break;
            case 'L': turnLeft(); break;
            case 'R': turnRight(); break;
            case 'S': stop(); break;
            case 'T': testMotors(); break;  // Alle Motoren einzeln testen
        }
    }
    
    // Sicherheits-Watchdog: Nach 1s ohne Befehl stoppen
    if (millis() - lastCommand > 1000) {
        stop();
    }
}

void moveForward() {
    motor1.setSpeed(200); motor1.run(FORWARD);
    motor2.setSpeed(200); motor2.run(FORWARD);
    motor3.setSpeed(200); motor3.run(FORWARD);
    motor4.setSpeed(200); motor4.run(FORWARD);
}

void turnLeft() {
    motor1.setSpeed(255); motor1.run(BACKWARD);  // Rechts zurück
    motor2.setSpeed(255); motor2.run(FORWARD);   // Links vorwärts
    motor3.setSpeed(255); motor3.run(BACKWARD);  // Rechts zurück
    motor4.setSpeed(255); motor4.run(FORWARD);   // Links vorwärts
}

// ... analog turnRight(), moveBackward(), stop() ...
```

**Besonderheiten:**
- **4 Motoren parallel:** Alle vier Motoren bekommen die gleiche Geschwindigkeit
- **Panzer-Lenkung:** Drehen auf der Stelle (LEFT/RIGHT drehen beide Seiten gegeneinander)
- **Watchdog:** Stoppt nach 1 Sekunde ohne Signal (Sicherheit)
- **INVERT-Schalter:** Falls Motoren rückwärts drehen, einfach auf `true` setzen

**Speicherort:** `~/Documents/Arduino/rc_car_4wd/rc_car_4wd.ino`

**Hochladen:**
```bash
~/.local/bin/arduino-cli compile --fqbn arduino:avr:diecimila:cpu=atmega328 ~/Documents/Arduino/rc_car_4wd/
~/.local/bin/arduino-cli upload -p /dev/cu.usbserial-A6008mgO --fqbn arduino:avr:diecimila:cpu=atmega328 ~/Documents/Arduino/rc_car_4wd/
```

---

## 4. Python-Software (Klassifikation & GUI)

**Speicherort:** `MindFlex_BCI_Projekt/`

### 4.1 Architektur

```
main.py
    |
    +---> SerialReceiver (Thread)  liest vom EEG-Arduino
    |
    +---> BrainProcessor
    |       |
    |       +---> FeatureExtractor   (5-Sekunden-Fenster)
    |       |
    |       +---> BCIClassifier      (trainiert/lädt ML-Modell)
    |       |
    |       +---> MainWindow (PyQt5) zeigt GUI
    |
    +---> BleCarController (asyncio-Thread)  sendet an Auto-Arduino
```

### 4.2 Datenfluss im Detail

**1. EEG-Empfang** (`serial_receiver.py`)

```python
class SerialReceiver:
    def __init__(self, port, baudrate=9600):
        self.serial = Serial(port, baudrate)
    
    def read_line(self):
        # Blockiert, bis eine komplette CSV-Zeile kommt
        # Format: SignalQuality,Attention,Meditation,Delta,...
        line = self.serial.readline().decode('utf-8').strip()
        return self._parse_csv(line)
```

- Läuft in eigenem Thread
- Versucht bei Fehler automatisch neu zu verbinden
- Sucht Port automatisch oder nutzt `MINDFLEX_EEG_PORT`

**2. Signalvalidierung** (`brain_processor.py`)

```python
class BrainProcessor:
    def process(self, data):
        if data['signal_quality'] == 200:
            # Signal verloren (kein Hautkontakt)
            return None  # Ignorieren!
        
        # Gültige Daten in 60-Sekunden-Ringpuffer
        self.buffer.append(data)
```

- Verwirft Pakete mit `signal_quality == 200` (ThinkGear-Konvention für "kein Signal")
- Puffert die letzten 60 Sekunden in Ring-Buffer
- Keine GUI-Updates, keine Fahrbefehle bei ungültigem Signal

**3. Feature-Extraktion** (`feature_extractor.py`)

```python
class FeatureExtractor:
    def extract_features(self, window):
        # Zeitfenster: 5 Sekunden Daten
        features = {}
        
        for band in ['Attention', 'Meditation', 'Delta', 'Theta', ...]:
            values = [sample[band] for sample in window]
            features[f'{band}_mean'] = np.mean(values)
            features[f'{band}_max'] = np.max(values)
            features[f'{band}_min'] = np.min(values)
            features[f'{band}_std'] = np.std(values)
        
        # Abgeleitete Features
        features['beta_alpha_ratio'] = total_beta / total_alpha
        features['attention_meditation_trend'] = trend(attention_values)
        
        return features
```

- Berechnet Mittelwert, Max, Min, Standardabweichung über **alle** EEG-Bänder
- Gleiche Feature-Extraktion beim Training und bei Live-Vorhersage (wichtig!)

**4. Machine Learning** (`classifier.py`)

```python
class BCIClassifier:
    def train(self, training_data):
        # Trainiert 3 Modelle
        models = {
            'decision_tree': DecisionTreeClassifier(),
            'random_forest': RandomForestClassifier(n_estimators=100),
            'knn': KNeighborsClassifier(n_neighbors=5)
        }
        
        # Test-Split: 80/20
        X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
        
        # Training + Evaluierung
        best_model = None
        best_accuracy = 0
        for name, model in models.items():
            model.fit(X_train, y_train)
            accuracy = model.score(X_test, y_test)
            if accuracy > best_accuracy:
                best_model = model
                best_accuracy = accuracy
        
        # Speichern des besten Modells (neu trainiert auf **allen** Daten)
        best_model.fit(X, y)
        joblib.dump(best_model, 'trained_model.pkl')
        
        return best_model
    
    def predict(self, features):
        # Live-Vorhersage
        model = joblib.load('trained_model.pkl')
        label = model.predict([features])
        confidence = model.predict_proba([features])
        return label, confidence
```

- Vergleicht Decision Tree, Random Forest, KNN
- Bestes Modell wird neu auf **allen** Daten trainiert (nicht nur Train-Set)
- Wird beim Start automatisch geladen

**5. Fahrzeugkontrolle** (`ble_car_controller.py`)

```python
class BleCarController:
    async def send_label(self, label):
        """Sendet einen Fahrbefehl (nur bei Änderung)"""
        if label != self.last_command:
            char_code = self._label_to_char(label)  # F / B / L / R / S
            await self.client.write_gatt_char(UUID, char_code.encode())
            self.last_command = label
    
    async def heartbeat(self):
        """Wiederholt den Befehl alle 0,4 Sekunden (Watchdog-Reset)"""
        while True:
            if self.last_command:
                await self.send_label(self.last_command)
            await asyncio.sleep(0.4)  # BleConfig.resend_interval
```

- **Deduplizierung:** Sendet Befehl nur bei Änderung
- **Heartbeat:** Wiederholt aktiven Befehl alle 0,4 Sekunden, sonst stoppt der Arduino nach 1s
- Asynchron (asyncio), weil BLE auf macOS über CoreBluetooth läuft

**6. Trainingsmodus** (`trainer.py`)

```python
class TrainingSession:
    def record(self, label, duration_seconds=10):
        """Nimmt Daten für 10 Sekunden unter Label 'label' auf"""
        start_time = time.time()
        collected = []
        
        while time.time() - start_time < duration_seconds:
            if self.processor.has_valid_signal():
                sample = self.processor.get_latest_sample()
                collected.append((sample, label))
        
        # Append to training_data.csv
        with open('training_data.csv', 'a') as f:
            for sample, lbl in collected:
                f.write(f"{sample},{lbl}\n")
```

- Nutzer wählt Label, drückt "Start Recording", denkt 10 Sekunden an den Fahrbefehl
- Alle gültigen EEG-Samples landen mit diesem Label in `training_data.csv`
- CLI-Modus: `python main.py --mode train`
- GUI-Modus: Button in der Oberfläche

**7. GUI** (`visualizer.py`, PyQt5 + pyqtgraph)

```python
class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        
        # Status-Ampel
        self.status_light = CircleLight()  # Rot=kein Signal, Gelb=schwach, Grün=gut
        
        # Live-Graphen (letzten 60 Sekunden)
        self.plot_widget = pyqtgraph.PlotWidget()
        self.plot_curves = {}
        for band in BANDS:
            self.plot_curves[band] = self.plot_widget.plot(name=band)
        
        # Vorhersage-Pfeil (grün=sicher, gelb=unsicher)
        self.prediction_arrow = RotatingArrow()
        
        # Buttons
        self.btn_start_training = QPushButton("Training starten")
        self.btn_retrain = QPushButton("Modell neu trainieren")
        self.btn_load_model = QPushButton("Letztes Modell laden")
        self.btn_export = QPushButton("Trainingsdaten exportieren")
    
    def update_ui(self):
        """Wird ~20 Hz aufgerufen"""
        # Graphen aktualisieren
        for band in BANDS:
            values = [s[band] for s in self.processor.buffer]
            self.plot_curves[band].setData(values)
        
        # Vorhersage aktualisieren
        if self.processor.has_valid_signal():
            label, confidence = self.classifier.predict(...)
            self.prediction_arrow.set_direction(label)
            self.prediction_arrow.set_confidence(confidence)
        
        # Status-Ampel aktualisieren
        quality = self.processor.get_signal_quality()
        self.status_light.set_color(quality_to_color(quality))
```

- Update-Rate: ~20 Hz
- Graphen zeigen letzte 60 Sekunden
- Vorhersage-Pfeil dreht sich (rot/gelb = Richtung + Konfidenz)
- Aufnahme & Training in eigenen Threads (GUI friert nicht ein)

---

## 5. Trainingsdaten & Modell

### Datenstruktur

**`training_data.csv`** — wächst beim Training:

```
signal_quality,attention,meditation,delta,theta,low_alpha,high_alpha,low_beta,high_beta,low_gamma,high_gamma,label
200,40,50,5000,3000,2000,1500,1200,1000,800,600,FORWARD
200,38,52,4900,2900,1900,1400,1100,950,750,580,FORWARD
0,42,48,5100,3200,2100,1600,1300,1100,850,650,STOP
...
```

- Jede Zeile = ein EEG-Sample (ausgegeben vom Arduino)
- Label = der Fahrbefehl, den der Nutzer dachte
- Gleiche Struktur beim Training und bei Live-Vorhersage (wichtig!)

### Datenvorbereitung

**Stand 2026-08-15:**
- Alte Daten (2 Klassen) gelöscht
- `trained_model_alt_2klassen.pkl` zum Vergleich archiviert
- **Neue Daten nötig für:** `LEFT`, `RIGHT`, sowie ausgeglichene Samples für `FORWARD` und `STOP`

**Aufzeichnungs-Richtlinie:**
```
Für jede Klasse ungefähr gleich lange aufzeichnen:
- FORWARD: ~30 Sekunden
- BACKWARD: ~30 Sekunden
- LEFT: ~30 Sekunden
- RIGHT: ~30 Sekunden
- STOP: ~30 Sekunden
```

Das verhindert, dass das Modell eine überrepräsentierte Klasse bevorzugt.

### Trainingsbefehl

```bash
# Interaktiv Daten sammeln
cd MindFlex_BCI_Projekt
python3 main.py --mode train

# Ohne Aufnahme, nur Modell neu trainieren
python3 main.py --mode retrain

# Oder über die GUI: Buttons "Training starten" + "Modell neu trainieren"
```

### Modell-Speicherort

```
MindFlex_BCI_Projekt/trained_model.pkl
```

Wird beim Programmstart automatisch geladen. Falls nicht vorhanden, verwendet das System ein Dummy-Modell.

---

## 6. Bluetooth & Funk-Integration

### BLE-Module

**Hardware:**
- Typ: CC2541 / HM-10 / AT-09 / MLT-BT05 (alle kompatibel)
- Standard: Bluetooth 4.0 Low Energy (nicht klassisches Bluetooth 2.0)
- Baud-Rate: 9600
- Datendurchsatz: unbedenklich für Fahrbefehle

**Warum BLE?**
- Stabil auf macOS 15 Sequoia
- Klassisches SPP (HC-05) macht dort Verbindungsabbrüche
- Native macOS-Unterstützung über CoreBluetooth
- Energie-effizient

### BLE-Service & Charakteristik

```
Service:     FFE0
Characteristic: FFE1

Was du schreibst -> kommt am TXD-Pin heraus
```

Im Python-Code:

```python
from bleak import BleakClient

uuid_service = "0000ffe0-0000-1000-8000-00805f9b34fb"
uuid_char = "0000ffe1-0000-1000-8000-00805f9b34fb"

async with BleakClient(address) as client:
    await client.write_gatt_char(uuid_char, b"F")  # 'F' für Forward
```

### Modul-Adresse ermitteln

```bash
python3 ble_scan.py
```

Zeigt alle verfügbaren BLE-Module. **Wichtig:** Mit ausgeschaltetem Auto (sonst sind beide Module nicht auseinanderzuhalten).

Adresse setzen:

```bash
MINDFLEX_CAR_BLE_ADDRESS=12:34:56:78:9A:BC python3 main.py
```

**Jeder Mac braucht eine andere UUID!** Auf einem neuen Rechner neu ermitteln.

### Test ohne Auto

```bash
# Modul suchen
python3 ble_scan.py

# F/L/R/B/S einmal senden
python3 ble_test.py

# Controller mit Befehlswiederholung testen
python3 ble_controller_test.py
```

---

## 7. Diagnose & Fehlerbehandlung

### Ports anzeigen

```bash
ls /dev/cu.*
```

Beispiel-Ausgabe:
```
/dev/cu.usbmodem141011    # Arduino Uno (EEG)
/dev/cu.usbserial-A6008mgO  # Arduino Duemilanove (USB, nur beim Upload)
/dev/cu.Bluetooth-Incoming-Port  # klassisches Bluetooth (nicht relevant)
```

### Seriellen Monitor öffnen

```bash
# Für Duemilanove (Auto-Arduino)
~/.local/bin/arduino-cli monitor -p /dev/cu.usbserial-A6008mgO

# Für Uno (EEG-Arduino) — nur wenn kein Python läuft!
~/.local/bin/arduino-cli monitor -p /dev/cu.usbmodem141011
```

**Wichtig:** Nur ein Programm kann einen Port gleichzeitig öffnen. Wenn Python läuft, kann der IDE-Monitor nicht auf dem EEG-Port zugreifen.

### Aktuelles Log anzeigen

```bash
cd MindFlex_BCI_Projekt
tail -f mindflex_bci.log
```

### Umgebungsvariablen für Debugging

```bash
# Fester EEG-Port
MINDFLEX_EEG_PORT=/dev/cu.usbmodem141011 python3 main.py

# Fester Auto-Port (USB statt BLE)
MINDFLEX_CAR_TRANSPORT=serial MINDFLEX_CAR_PORT=/dev/cu.usbserial-A6008mgO python3 main.py

# Nur bestimmte Labels aktiv
MINDFLEX_ACTIVE_LABELS="STOP,FORWARD,LEFT" python3 main.py

# EEG per Funk statt USB
MINDFLEX_EEG_TRANSPORT=ble MINDFLEX_EEG_BLE_ADDRESS=12:34:56:78:9A:BC python3 main.py
```

---

## 10. Steuerungskonzept: Panzer-Lenkung

### Vier Fahrbefehle

```
FORWARD   Beide Seiten vorwärts
BACKWARD  Beide Seiten rückwärts
LEFT      Linke Seite vorwärts, rechte Seite rückwärts → Drehung auf der Stelle
RIGHT     Rechte Seite vorwärts, linke Seite rückwärts → Drehung auf der Stelle
STOP      Alle Motoren aus
```

### Warum kein klassisches Lenken?

- RC-Autos haben üblicherweise ein Servo-Lenkrad
- Unser Auto hat **keine Servo** — dafür sind die Pins am Shield belegt
- Mit vier Motoren können wir "Panzer-Stil" lenken: Jede Seite separat ansteuern

### Motor-Zuordnung

```
        vorne
M2 (li)  []  M1 (re)
M4 (li)  []  M3 (re)
        hinten
```

- **Linke Seite:** M2 (vorne) + M4 (hinten) immer synchron
- **Rechte Seite:** M1 (vorne) + M3 (hinten) immer synchron

### Drehgeschwindigkeit

```python
TURN_SPEED = 255  # Volle Kraft beim Drehen
FORWARD_SPEED = 200  # Etwas weniger beim Geradeausfahren
```

Beim Drehen schleifen alle Räder seitlich über den Boden — das braucht mehr Drehmoment.

---

## 11. Bekannte Probleme & Lösungen

### Problem 1: Port-Autoerkennung funktioniert nicht

**Symptom:** `Kein serieller Port gefunden (hint='Arduino')`

**Ursache:** Der Duemilanove meldet sich über seinen FTDI-Chip als "FT232R USB UART", nicht als "Arduino". Die Erkennung sucht aber nach dem Substring "Arduino".

**Lösung:** Ports fest setzen:
```bash
MINDFLEX_EEG_PORT=/dev/cu.usbmodem141011 \
MINDFLEX_CAR_PORT=/dev/cu.usbserial-A6008mgO python3 main.py
```

Der Uno wird auf dieser Mac richtig erkannt, der Duemilanove nicht.

### Problem 2: Auto stoppt nach 1 Sekunde beim Fahren

**Symptom:** Auto fährt kurz und bleibt dann stehen, obwohl der Fahrbefehl unverändert ist

**Ursache:** Der Arduino-Sketch hat einen 1-Sekunden-Watchdog (Sicherheit). Python sendet einen Befehl aber nur, wenn er sich ändert (`last_command`-Deduplizierung). Bei dauerhaftem "FORWARD" kommt also nach der ersten Sekunde kein neuer Befehl → Arduino stoppt.

**Lösung:** Ein Heartbeat-Task wiederholt den Befehl alle 0,4 Sekunden:
```python
async def heartbeat(self):
    while True:
        if self.last_command:
            await self.send_label(self.last_command)
        await asyncio.sleep(0.4)
```

Die Deduplizierung bleibt — nur die Wiederholung greift.

### Problem 3: Der Sketch liest von der falschen Schnittstelle

**Symptom:** Keine Fahrbefehle ankommen, obwohl BLE verbunden ist

**Ursache:** Der Duemilanove hat nur eine Hardware-UART (Pin 0/1). Sie ist mit dem USB-Chip verbunden. Säße das HC-05 dort, wäre weder Sketch-Upload noch Debugging möglich. Der Sketch muss auf `SoftwareSerial` umgestellt werden.

**Lösung:** `SoftwareSerial(10, A0)` im Sketch verwenden:
```cpp
SoftwareSerial ble(10, A0);  // RX=Pin 10, TX=A0 (ungenutzt)

void setup() {
    Serial.begin(9600);      // USB
    ble.begin(9600);         // Bluetooth
}

void loop() {
    if (ble.available()) {
        char cmd = ble.read();
        // ...
    }
}
```

### Problem 4: L293D vs. L298N — Shield-Pinbelegung

**Symptom:** Motoren drehen nicht oder falsch nach Motor-Shield-Upgrade

**Ursache:** Der alte Sketch war für L298N-H-Brücke geschrieben (direkte Pinansteuerung). Das neue L293D-Shield nutzt ein Schieberegister und die `AFMotor`-Bibliothek.

**Lösung:** Motor-Teil mit `AFMotor` reschreiben:
```cpp
#include <AFMotor.h>

AF_DCMotor motor1(1);
AF_DCMotor motor2(2);
AF_DCMotor motor3(3);
AF_DCMotor motor4(4);

void moveForward() {
    motor1.setSpeed(200);
    motor1.run(FORWARD);
    // ... analog für andere Motoren
}
```

---

## 12. Schritt-für-Schritt: Von der Installation zum ersten Fahrt

### Schritt 1: Voraussetzungen

```bash
# Python 3.8+
python3 --version

# Arduino CLI installiert?
~/.local/bin/arduino-cli version

```

### Schritt 2: Pythonpakete installieren

```bash
cd MindFlex_BCI_Projekt
pip3 install -r requirements.txt
```

**Dateiinhalt `requirements.txt`:**
```
pyserial==3.5
scikit-learn==1.3.0
numpy==1.24.0
PyQt5==5.15.9
pyqtgraph==0.13.3
bleak==0.20.0  # BLE
```

### Schritt 3: Arduino-Sketches hochladen

**EEG-Arduino (Uno):**
```bash
~/.local/bin/arduino-cli upload \
    -p /dev/cu.usbmodem141011 \
    --fqbn arduino:avr:diecimila:cpu=atmega328 \
    ~/Documents/Arduino/mindflex_eeg/
```

**Auto-Arduino (Duemilanove):**
```bash
~/.local/bin/arduino-cli upload \
    -p /dev/cu.usbserial-A6008mgO \
    --fqbn arduino:avr:diecimila:cpu=atmega328 \
    ~/Documents/Arduino/rc_car_4wd/
```

### Schritt 4: Headset testen

```bash
# Seriellen Monitor öffnen (Uno)
~/.local/bin/arduino-cli monitor -p /dev/cu.usbmodem141011
```

Sollte CSV-Zeilen ausgeben. Wenn nicht: Headset an, Kontakt auf Stirn prüfen.

### Schritt 5: BLE-Modul-Adresse ermitteln

```bash
cd MindFlex_BCI_Projekt
python3 ble_scan.py
```

Notiere die Adresse des Auto-Moduls (CC2541 / HM-10).

### Schritt 6: Trainingsdaten sammeln

```bash
cd MindFlex_BCI_Projekt
python3 main.py --mode train
```

Folge den Anweisungen:
1. Wähle ein Label (z. B. "FORWARD")
2. Denke 10 Sekunden an Vorwärtsfahrt
3. Wiederhöle für alle 5 Labels, jeweils ~30 Sekunden

→ Schreibt `training_data.csv`

### Schritt 7: Modell trainieren

```bash
cd MindFlex_BCI_Projekt
python3 main.py --mode retrain
```

Wertet `training_data.csv` aus, trainiert 3 Modelle, speichert das beste.

### Schritt 8: GUI starten & fahren

```bash
MINDFLEX_CAR_BLE_ADDRESS=<UUID> python3 main.py
```

GUI zeigt:
- Status-Ampel (Signalqualität)
- Live-Graphen
- Vorhersage mit Konfidenz-Pfeil
- Buttons zum Retraining

Denke an Fahrtrichtungen → Auto fährt!

---

## 13. Bekannte Einschränkungen

### 1. EEG-Hardware ist begrenzt

**MindFlex misst mit nur einer Elektrode (Fp1, Stirn).** Der Unterschied zwischen "denk LEFT" und "denk RIGHT" entsteht aber über dem motorischen Kortex (C3/C4 — beide Seiten). Die Elektrode sitzt dort gar nicht. Es ist deshalb gut möglich, dass LEFT/RIGHT im Signal nicht zu trennen sind.

**Workaround:** Umgebungsvariable auf 3 Labels reduzieren:
```bash
MINDFLEX_ACTIVE_LABELS="STOP,FORWARD,LEFT" python3 main.py
```

BACKWARD bleibt in der Konfiguration erhalten und kann jederzeit reaktiviert werden.

### 2. ML-Modell braucht individuelle Trainingsdaten

Gehirnwellen sind sehr individuell. Ein Modell, das auf Moritz trainiert wurde, funktioniert möglicherweise nicht für Anna. Jede Person sollte eigene Trainingsdaten sammeln.

### 3. Signalqualität ist nicht garantiert

- Trockenelektrode benötigt guten Hautkontakt
- Haare unter der Elektrode verschlechtern den Kontakt
- Bei Bewegung Artefakte möglich
- ThinkGear-Chip filtert aggressiv (keine Rohdaten möglich)

### 4. RC-Auto reagiert asynchron

Durch BLE-Funklatenz und Arduino-Verarbeitung gibt es eine Verzögerung von ~100–200 ms. Das ist für ein RC-Auto akzeptabel, aber nicht ideal für präzise Steuerung.

---

## 14. Nächste Schritte & Verbesserungen

### Geplant

- [ ] Trainingsdaten für LEFT & RIGHT sammeln (alle 5 Labels gleich gewichtet)
- [ ] Modell-Genauigkeit mit nur 3 Labels vs. 4 Labels vs. 5 Labels vergleichen
- [ ] Latenzmessung (wie lange von "Gedanke" bis "Auto fährt"?)
- [ ] Architektur-Verbesserungen (z. B. LSTM für Zeitabhängigkeiten?)

### Optional

- **Servo-Lenk-Upgrade:** Klassisches Lenken statt Panzer-Stil (braucht neues Shield ohne Motor 4)
- **IMU-Integration:** Beschleunigungssensor + Gyroskop für stabilere Vorhersagen
- **Web-UI:** Statt PyQt5 → Browser-Oberfläche mit Flask/Dash
- **Multi-Person-Klassifikation:** Einzelnes Modell für mehrere Personen
- **Hyperparameter-Tuning:** GridSearch für beste ML-Parameter

---

## 15. Wichtige Referenzen & Dateien

| Datei | Zweck |
|---|---|
| `MindFlex_BCI_Projekt/main.py` | Hauptprogramm |
| `MindFlex_BCI_Projekt/training_data.csv` | Trainingsdaten (wächst beim Trainieren) |
| `MindFlex_BCI_Projekt/trained_model.pkl` | Gespeichertes ML-Modell |
| `~/Documents/Arduino/mindflex_eeg/` | EEG-Arduino-Sketch |
| `~/Documents/Arduino/rc_car_4wd/` | Auto-Arduino-Sketch |
| `mindflex_bci.log` | Logs (im Projektordner) |
| `code/` | Versionsübersicht des Codes |

---

## 16. Zusammenfassung: Das System in einer Nutshell

```
┌─────────────────────────────────────────────────────────┐
│ 1. DU TRÄGST EIN EEG-HEADSET                            │
│    (NeuroSky MindFlex, eine Elektrode auf Fp1)          │
└─────────────────────────────────────────────────────────┘
        ↓ (seriell: USB oder BLE)
┌─────────────────────────────────────────────────────────┐
│ 2. ARDUINO UNO LIEST GEHIRNWELLEN                        │
│    (ThinkGear-Protokoll) → CSV-Zeile alle 500 ms        │
└─────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────┐
│ 3. PYTHON ANALYSIERT SIGNALE                            │
│    - Signalqualität prüfen                              │
│    - Features extrahieren (5 Sekunden-Fenster)          │
│    - ML-Modell vorhersagen (Decision Tree / RF / KNN)   │
│    - Label → Fahrbefehl (F/B/L/R/S)                     │
└─────────────────────────────────────────────────────────┘
        ↓ (BLE: Bluetooth Low Energy)
┌─────────────────────────────────────────────────────────┐
│ 4. ARDUINO DUEMILANOVE STEUERT AUTO                     │
│    - Empfängt Fahrbefehl über BLE                       │
│    - Weiterleitung zu L293D Shield                       │
│    - Shield steuert 4 Getriebemotoren                   │
└─────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────┐
│ 5. RC-AUTO FÄHRT!                                       │
│    (Panzer-Lenkung: Seiten unabhängig, Drehung Stelle) │
└─────────────────────────────────────────────────────────┘
```

---

**Zuletzt aktualisiert:** 2026-09-02

Weitere Details zu einzelnen Komponenten siehe verwandte Notizen und die Versionsübersicht des Codes.
