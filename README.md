# S7-1500 Konturmesssystem - Prozessbeschreibung

Dieses Projekt implementiert eine Schrittkette zur Konturmessung und Qualitätssicherung von Warenträgern (Laufwagen) auf einer Förderstrecke.

## Übersicht der Komponenten

Der Code ist modular in folgende SCL-Quellen unterteilt:

### 1. Datenstrukturen & Globale DBs

- **`Types.scl`**:
  - `typeArticleDefinition`: Definiert die Soll-Werte und Toleranzen eines Artikels.
  - `typeCarriageData`: Enthält die Prozessdaten (Laufwagen-Nr, Artikel-Nr, Ist-Werte, Status) für die Verfolgung in der Anlage.
- **`DB_Maximalkontur.scl`**: Globaler Datenspeicher (`MaximalkonturDB`) für bis zu 999 Artikeldefinitionen.
- **`DB_KetteLaufwagen.scl`**: Abbild der Förderstrecke (`KetteLaufwagenDB`). Speichert die Daten für Station 1 bis 4.

### 2. Funktionsbausteine (Logik)

- **`FB_MainChain.scl`**: Der Hauptbaustein. Er ruft alle Stations-FBs auf, steuert den Materialfluss (Schieben der Daten von Station zu Station) und überwacht den **Global-Reset**.
- **`FB_Station1_Entry.scl`**: Handshake mit der S7-1200. Empfängt neue Laufwagen-Daten.
- **`FB_Station2_Check.scl`**: Prüft, ob der Artikel bekannt ist. Falls nein, wird ein Teach-In Prozess gestartet.
- **`FB_Station3_Measure.scl`**: Die Kernlogik für Messung, Vergleich und Bewertung (OK/NOK/Neu).
- **`FB_Station4_Robot.scl`**: Übergabe an den Lackierroboter.

---

## Prozessablauf (Schrittkette)

### Step Init (Global Reset)

Werden `bBtnGreen` (Grüner Taster) und `bBtnRed` (Roter Taster) gleichzeitig für **5 Sekunden** gedrückt, wird die gesammte Anlage zurückgesetzt. Alle Daten in `KetteLaufwagenDB` werden gelöscht.

### Station 1: Einlauf & Identifikation typ

- **Step 1**: Laufwagen fährt in Stopper 1 ein.
- **Step 2**: S7-1500 empfängt Programmnummer und Laufwagennummer von der vorgelagerten S7-1200.
- **Step 2.3**: Daten werden in `Station1` gespeichert.

### Station 2: Datenbank-Check

- **Step 3**: Laufwagen fährt zu Station 2. Daten werden von `Station1` nach `Station2` kopiert.
- **Step 4**: Es wird geprüft, ob die Programmnummer im `MaximalkonturDB` existiert.
  - **Gefunden**: Weiter zu Station 3.
  - **Nicht Gefunden**: Anlage Halt. Leuchte Rot+Grün an.
    - **Bediener-Teach**: Durch Drücken von `bBtnGreen` wird der Artikel als "Neu" markiert (`bIsNewArticle`). Anlage läuft weiter.

### Station 3: Messung & Bewertung

- **Step 6**: Laufwagen in Station 3. Messung startet (`bIniMessungStart`).
- **Step 7**: Konturmessung läuft (Grüne Leuchte blinkt).
  - **Step 7.1 (Neuer Artikel)**:
    - System sucht ersten freien Platz im DB.
    - Die gemessenen Ist-Werte werden als Soll-Werte gespeichert.
    - Artikel ist nun eingelernt.
  - **Step 7.2 (Existierender Artikel)**:
    - **7.2.1 Suche**: Prüft erneut auf Gültigkeit des Datensatzes.
    - **7.2.2 Vergleich**: Vergleicht Ist-Werte mit den Soll-Werten aus dem DB unter Berücksichtigung der Toleranzen (`nTolX`, `nTolY`).
    - **OK**: Weiter zu Station 4.
    - **Fehler (Konturabweichung)**: Rote Leuchte blinkt, Hupe an.
      - **Korrektur (7.2.3.1)**: Bediener drückt `bBtnGreen` (> 5s) -> Messung wird akzeptiert.
      - **Ausschuss (7.2.3.2)**: Bediener drückt `bBtnRed` -> Teil wird als NOK markiert.
  - **Step 7.2.5 (NOK Ausgang)**: Übergabe an Roboter wird gesperrt (PrgrNr 999).

### Station 4: Roboterübergabe

- **Step 8**: Roboter erhält Programmnummer (oder 999 bei Fehler) und führt den Prozess fort.

---

## Bedienung

| Signal                  | Funktion                                                          |
| :---------------------- | :---------------------------------------------------------------- |
| **bBtnGreen**           | Quittierung, Teach-In Bestätigung, Fehlerkorrektur (halten > 5s)  |
| **bBtnRed**             | Fehlerbestätigung (Ausschuss), Teil des Global-Resets             |
| **bBtnGreen + bBtnRed** | **Global Reset** (beide halten > 5s)                              |
| **bLightGreen**         | Blinkt: Messung läuft. Dauer: OK / Aktion erforderlich (mit Rot). |
| **bLightRed**           | Blinkt: Fehler/NOK. Dauer: Aktion erforderlich (mit Grün).        |
