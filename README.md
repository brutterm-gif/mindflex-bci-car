# Brain-Computer-Interfaces – Seminarfacharbeit

**Wie Gedanken Maschinen steuern: Chancen und gesellschaftliche Bedenken**
Praktischer Teil einer Seminarfacharbeit – Software und Hardware von Moritz Brutter

Im praktischen Teil der Arbeit wird ein umgebautes **MindFlex-EEG-Headset** über einen Arduino ausgelesen.
Eine Python-Software wertet die Gehirnwellen aus, klassifiziert sie und steuert damit ein
selbst konstruiertes, 3D-gedrucktes **RC-Auto** (per USB oder Bluetooth Low Energy).

```
MindFlex-Headset → Arduino → (USB / BLE) → Python-Software → Klassifikator → RC-Auto
```

## Inhalt

| Pfad | Inhalt |
|---|---|
| [`ARCHITEKTUR.md`](ARCHITEKTUR.md) | Systemarchitektur: Hardware-Setup, Komponenten, Arduino-Sketches, Python-Software, Klassifikation, Bluetooth, Steuerungskonzept |
| [`code/MindFlex_BCI_Projekt/`](code/MindFlex_BCI_Projekt) | Hauptprogramm (v3.4): Datenempfang, Merkmalsextraktion, Training, Klassifikation, GUI, Autosteuerung, Arduino-Sketches |
| [`code/BrainGrapher_Processing/`](code/BrainGrapher_Processing) | Brain Grapher (Processing), erweitert um BLE-Anbindung |
| [`code/BrainGrapher_Python/`](code/BrainGrapher_Python) | Nachbau des Brain Graphers in Python (PyQt) |
| [`hardware/`](hardware) | 3D-Modell des Auto-Chassis (aktueller Stand v25) als STL |

## Software starten

Siehe [`code/MindFlex_BCI_Projekt/START_HIER.md`](code/MindFlex_BCI_Projekt/START_HIER.md) und
[`code/MindFlex_BCI_Projekt/README.md`](code/MindFlex_BCI_Projekt/README.md).

```bash
cd code/MindFlex_BCI_Projekt
pip install -r requirements.txt
python main.py
```

## Hinweise

- Die EEG-Trainingsprofile von Versuchspersonen sind aus Datenschutzgründen **nicht** enthalten.
- Der Brain Grapher basiert auf dem
  [Processing Brain Grapher](https://github.com/kitschpatrol/Processing-Brain-Grapher) von Eric Mika
  und nutzt die [Arduino Brain Library](https://github.com/kitschpatrol/Brain).
