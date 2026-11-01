# Umsatz-, Marge- und Saison-Dashboard für einen Getränkemarkt

Power-BI-Projekt zur Analyse von Umsatz, Marge und Saisoneffekten einer regionalen Getränkemarkt-Kette mit fünf Filialen im Allgäu.

> **Hinweis:** Alle Daten in diesem Repository sind **fiktiv** und synthetisch erzeugt. Sie stammen aus keinem realen Unternehmen. Namen von Orten dienen nur der Darstellung auf der Karte.

## Ziel

Das Projekt zeigt den Weg von Rohdaten bis zum Bericht:

1. Daten laden und in Power Query prüfen
2. Datenmodell als Sternschema aufbauen
3. Kennzahlen in DAX berechnen
4. Ergebnisse in einem Bericht mit drei Seiten darstellen

Leitfragen des Berichts:

- Wie entwickelt sich der Umsatz im Vergleich zum Vorjahr?
- Welche Artikelgruppen bringen Umsatz, welche Marge?
- In welchen Monaten ist Saison, und wie stark ist sie je Artikelgruppe?
- Wie unterscheiden sich die Filialen?

## Inhalt des Repositorys

```
.
├── README.md
├── data/
│   ├── Verkaeufe.csv      # Faktentabelle (ca. 81.700 Zeilen)
│   ├── Artikel.csv        # 30 Artikel in 6 Artikelgruppen
│   ├── Filialen.csv       # 5 Filialen
│   └── Datum.csv          # Kalender 01.01.2025 bis 31.10.2026
├── powerbi/
│   └── Umsatz_Saison_Dashboard_Getraenkemarkt.pbix
└── docs/
    └── screenshots/       # Bilder der Berichtsseiten
```

## Daten

| Datei | Inhalt |
|---|---|
| `Verkaeufe.csv` | VerkaufID, Datum, FilialeID, ArtikelID, Menge, Umsatz, Aktion (0/1) |
| `Artikel.csv` | ArtikelID, Artikelname, Artikelgruppe, Einkaufspreis, Verkaufspreis |
| `Filialen.csv` | FilialeID, Filiale, Bundesland, Groesse |
| `Datum.csv` | Datum, Jahr, Quartal, MonatNr, Monat, WochentagNr, Wochentag |

Eigenschaften der synthetischen Daten:

- Zeitraum: **01.01.2025 bis 31.10.2026** (2026 ist unvollständig)
- Sonntags geschlossen
- Saisoneffekte: Wasser und Bier im Sommer, Sekt und Glühwein im Dezember
- Aktionswochen mit 15 % Rabatt auf den Verkaufspreis
- Leichtes Wachstum im Jahr 2026

Format der CSV-Dateien: UTF-8 mit BOM, Trennzeichen `;`, Dezimalzeichen `,` (passend zu einem deutschen Gebietsschema).

## Datenmodell

Sternschema mit einer Faktentabelle und drei Dimensionen:

```
                 Artikel
                    |
Filialen ---- Verkaeufe ---- Datum
```

| Beziehung | Kardinalität |
|---|---|
| `Artikel[ArtikelID]` → `Verkaeufe[ArtikelID]` | 1 : n |
| `Filialen[FilialeID]` → `Verkaeufe[FilialeID]` | 1 : n |
| `Datum[Datum]` → `Verkaeufe[Datum]` | 1 : n |

Die Tabelle `Datum` ist als Datumstabelle markiert, die Spalte `Monat` ist nach `MonatNr` sortiert. Alle Kennzahlen liegen in einer eigenen Messwerttabelle `_Kennzahlen`.

## Wichtige Kennzahlen (DAX)

```dax
Umsatz = SUM ( Verkaeufe[Umsatz] )

Wareneinsatz =
SUMX ( Verkaeufe, Verkaeufe[Menge] * RELATED ( Artikel[Einkaufspreis] ) )

Rohertrag = [Umsatz] - [Wareneinsatz]

Marge % = DIVIDE ( [Rohertrag], [Umsatz] )
```

**Vorjahresvergleich auf gleichem Zeitraum**

```dax
Umsatz VJ = CALCULATE ( [Umsatz], SAMEPERIODLASTYEAR ( Datum[Datum] ) )

Umsatz Aktuell =
VAR J = MAX ( Datum[Jahr] )
RETURN CALCULATE ( [Umsatz], Datum[Jahr] = J )

Umsatz Vorjahr =
VAR J = MAX ( Datum[Jahr] )
RETURN CALCULATE ( [Umsatz VJ], Datum[Jahr] = J )

Umsatz YoY % (aktuelles Jahr) =
DIVIDE ( [Umsatz Aktuell] - [Umsatz Vorjahr], [Umsatz Vorjahr] )
```

Das Vorjahr deckt denselben Zeitraum ab wie das aktuelle Jahr (Januar bis Oktober). So bleibt der Vergleich fair, obwohl 2026 unvollständig ist.

**Saisonindex**

```dax
Saisonindex =
VAR UmsatzMonat = [Umsatz]
VAR Monatsschnitt =
    AVERAGEX ( ALLSELECTED ( Datum[MonatNr], Datum[Monat] ), [Umsatz] )
RETURN DIVIDE ( UmsatzMonat, Monatsschnitt )
```

Ein Wert von 1,0 entspricht einem durchschnittlichen Monat, 1,5 liegt 50 % darüber, 0,7 liegt 30 % darunter.

**Datenstand (für dynamische Titel)**

```dax
Datenstand =
"Datenstand: "
    & FORMAT ( CALCULATE ( MAX ( Verkaeufe[Datum] ), ALL ( Verkaeufe ) ), "DD.MM.YYYY" )
```

## Berichtsseiten

| Seite | Inhalt |
|---|---|
| **Überblick** | Karten mit Umsatz 2026, Umsatz im Vorjahreszeitraum und Umsatzentwicklung (mit Pfeil und Farbe), Säulendiagramm Umsatz nach Monat und Jahr |
| **Umsatz** | Streudiagramm Umsatz gegen Marge je Artikelgruppe, Saisonindex nach Monat, Matrix mit Marge, Rohertrag und Umsatz je Jahr, Slicer für Jahr und Monat |
| **Filiale** | Karte der Filialen (Größe = Umsatz, Farbe = Marge), Textzeile mit der aktuellen Auswahl |

## Beispielhafte Erkenntnisse

Die folgenden Aussagen gelten nur für die fiktiven Daten:

- Bier und Softdrinks tragen den Umsatz, haben aber die niedrigsten Margen (rund 32 bis 34 %).
- Wein hat mit rund 49 % die höchste Marge bei vergleichsweise kleinem Umsatz.
- Die Saison erreicht im Juli ihren Höhepunkt, am schwächsten ist der Februar.
- Der Umsatz von Januar bis Oktober 2026 liegt rund 5,4 % über dem gleichen Zeitraum 2025.

## Projekt öffnen

1. **Power BI Desktop** installieren (nur Windows).
2. Die Datei aus `powerbi/` öffnen.
3. Falls die Daten nicht geladen werden: **Start → Daten transformieren → Datenquelleneinstellungen** und die Pfade der CSV-Dateien auf den Ordner `data/` anpassen.
4. **Aktualisieren**, danach stehen alle Seiten zur Verfügung.

## Umsetzung in Microsoft Fabric (Konzept)

Das Projekt läuft lokal in Power BI Desktop. In Microsoft Fabric würde ich es so aufbauen (nicht umgesetzt):

1. **Workspace** anlegen und ein **Lakehouse** erstellen.
2. CSV-Dateien in den Bereich **Files** laden, alternativ per **Pipeline** (Copy-Aktivität) aus einer Quelle kopieren.
3. Die Dateien als **Delta-Tabellen** speichern (Load to Tables oder Notebook mit PySpark). Delta ist ein Parquet-Format mit Transaktionsprotokoll.
4. **Semantikmodell** auf den Delta-Tabellen mit **Direct Lake** aufbauen und die DAX-Kennzahlen dorthin übernehmen.
5. Bericht auf dem Semantikmodell veröffentlichen und über eine **Power BI-App** bereitstellen.
6. Bei Bedarf **Shortcuts** für externe Quellen und **Mirroring** für Datenbanken nutzen.

## Genutzte Werkzeuge

- Power BI Desktop (Power Query, DAX, Berichtsansicht)
- CSV als Datenquelle
- Git zur Versionierung

## Lizenz und Daten

Alle Daten sind fiktiv. Der Code und die Berichtsdatei dürfen zu Lernzwecken verwendet werden.
