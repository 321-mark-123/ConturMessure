# S7-1500 Konturmesssystem - Prozessbeschreibung

Dieses Projekt implementiert eine Schrittkette zur Konturmessung und Qualittssicherung von Warentrgern (Laufwagen) auf einer Frderstrecke.

## obersicht der Komponenten

Der Code ist modular in folgende SCL-Quellen unterteilt:

### 1. Datenstrukturen & Globale DBs

- **`Types.scl`**:
  - `typeArticleDefinition`: Definiert die Soll-Werte und Toleranzen eines Artikels.
  - `typeCarriageData`: Enthlt die Prozessdaten (Laufwagen-Nr, Artikel-Nr, Ist-Werte, Status) fǬr die Verfolgung in der Anlage.
- **`DB_Maximalkontur.scl`**: Globaler Datenspeicher (`MaximalkonturDB`) fǬr bis zu 999 Artikeldefinitionen.
- **`DB_KetteLaufwagen.scl`**: Abbild der Frderstrecke (`KetteLaufwagenDB`). Speichert die Daten fǬr Station 1 bis 4.

### 2. Funktionsbausteine (Logik)

- **`FB_MainChain.scl`**: Der Hauptbaustein. Er ruft alle Stations-FBs auf, steuert den Materialfluss und **kombiniert die Ausgangssignale** (Ampeln/Hupen) aller Stationen mittels ODER-Logik.
- **`FB_Station1_Entry.scl`**: Handshake mit der S7-1200. Empfngt neue Laufwagen-Daten.
- **`FB_Station2_Check.scl`**: PrǬft, ob der Artikel bekannt ist. Falls nein, wird ein Teach-In Prozess gestartet.
- **`FB_Station3_Measure.scl`**: Die Kernlogik fǬr Messung, Vergleich und Bewertung (OK/NOK/Neu).
- **`FB_Station4_Robot.scl`**: obergabe an den Lackierroboter.
- **`FB_ChainControl.scl`**: *[DEPRECATED]* Alte Version. Bitte nicht mehr verwenden.

### 3. HMI Schnittstelle

- **`04_HMI_Interface.xml`**: Definiert die UDTs und Tag-Mappings fǬr das Siemens Unified Comfort Panel. Status-Informationen und Steuerbefehle sind strukturiert gruppiert.

---

## Prozessablauf (Schrittkette)

### Step Init (Global Reset)

Werden `bBtnGreen` (GrǬner Taster) und `bBtnRed` (Roter Taster) gleichzeitig fǬr **5 Sekunden** gedrǬckt, wird die gesammte Anlage zurǬckgesetzt. Alle Daten in `KetteLaufwagenDB` werden gelscht.

### Station 1: Einlauf & Identifikation typ

- **Step 1**: Laufwagen fhrt in Stopper 1 ein.
- **Step 2**: S7-1500 empfngt Programmnummer und Laufwagennummer von der vorgelagerten S7-1200.
- **Step 2.3**: Daten werden in `Station1` gespeichert.

### Station 2: Datenbank-Check

- **Step 3**: Laufwagen fhrt zu Station 2. Daten werden von `Station1` nach `Station2` kopiert.
- **Step 4**: Es wird geprǬft, ob die Programmnummer im `MaximalkonturDB` existiert.
  - **Gefunden**: Weiter zu Station 3.
  - **Nicht Gefunden**: Anlage Halt. Leuchte Rot+GrǬn an.
    - **Bediener-Teach**: Durch DrǬcken von `bBtnGreen` wird der Artikel als "Neu" markiert (`bIsNewArticle`). Anlage luft weiter.

### Station 3: Messung & Bewertung

- **Step 6**: Laufwagen in Station 3. Messung startet (`bIniMessungStart`).
- **Step 7**: Konturmessung luft (GrǬne Leuchte blinkt).
  - **Step 7.1 (Neuer Artikel)**:
    - System sucht ersten freien Platz im DB.
    - Die gemessenen Ist-Werte werden als Soll-Werte gespeichert.
    - Artikel ist nun eingelernt.
  - **Step 7.2 (Existierender Artikel)**:
    - **7.2.1 Suche**: PrǬft erneut auf GǬltigkeit des Datensatzes.
    - **7.2.2 Vergleich**: Vergleicht Ist-Werte mit den Soll-Werten aus dem DB unter BerǬcksichtigung der Toleranzen (`nTolX`, `nTolY`).
    - **OK**: Weiter zu Station 4.
    - **Fehler (Konturabweichung)**: Rote Leuchte blinkt, Hupe an.
      - **Korrektur (7.2.3.1)**: Bediener drǬckt `bBtnGreen` (> 5s) -> Messung wird akzeptiert.
      - **Ausschuss (7.2.3.2)**: Bediener drǬckt `bBtnRed` -> Teil wird als NOK markiert.
  - **Step 7.2.5 (NOK Ausgang)**: obergabe an Roboter wird gesperrt (PrgrNr 999).

### Station 4: RoboterǬbergabe

- **Step 8**: Roboter erhlt Programmnummer (oder 999 bei Fehler) und fǬhrt den Prozess fort.

---

## Bedienung

| Signal                  | Funktion                                                          |
| :---------------------- | :---------------------------------------------------------------- |
| **bBtnGreen**           | Quittierung, Teach-In Besttigung, Fehlerkorrektur (halten > 5s)  |
| **bBtnRed**             | Fehlerbesttigung (Ausschuss), Teil des Global-Resets             |
| **bBtnGreen + bBtnRed** | **Global Reset** (beide halten > 5s)                              |
| **bLightGreen**         | Blinkt: Messung luft. Dauer: OK / Aktion erforderlich (mit Rot). |
| **bLightRed**           | Blinkt: Fehler/NOK. Dauer: Aktion erforderlich (mit GrǬn).        |
