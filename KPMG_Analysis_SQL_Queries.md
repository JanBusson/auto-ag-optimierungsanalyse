# SQL-Abfragen für KPMG-Projekt

Dieses Dokument enthält die SQL-Abfragen und deren Beschreibungen zur Analyse der Verkaufs- und Betriebsdaten der Auto AG.

---

## FR 01: Lieferantenanalyse
Ermittlung der Lieferanten, deren Zielländer, Preise und durchschnittlichen Lieferzeiten.
```sql
SELECT LIEFERANT AS Lieferantenname, ZIELLAND, PREIS, a.Lieferzeit_Tage
FROM KPMG.LIEFERANTEN l
INNER JOIN (
    SELECT LIEFERANTENNR, AVG(Lieferzeit) AS Lieferzeit_Tage
    FROM (
        SELECT f.FIN, f.LIEFERANTENNR, (f.LIEFERDATUM - b.BESTELLDATUM) AS Lieferzeit, f.AUFTRAGSNUMMER
        FROM KPMG.FAKTURA f
        INNER JOIN KPMG.BESTELLUNGEN b ON f.AUFTRAGSNUMMER = b.AUFTRAGSNUMMER
    )
    GROUP BY LIEFERANTENNR
) a ON l.LIEFERANTNR = a.LIEFERANTENNR
ORDER BY ZIELLAND, PREIS, Lieferzeit_Tage ASC;
```

---

## FR 02: Zahlungsdauer nach Land
Berechnung der durchschnittlichen Zahlungsdauer je Land.
```sql
SELECT b.LAND, AVG(Zahlungsdauer) AS durchschnit_Zahlungsdauer
FROM (
    SELECT LAND, k.KUNDENNUMMER, Zahlungsdauer
    FROM KPMG.KUNDEN k
    INNER JOIN (
        SELECT KUNDENNUMMER, f.AUFTRAGSNUMMER, Zahlungsdauer
        FROM KPMG.FAKTURA f
        INNER JOIN (
            SELECT AUFTRAGSNUMMER, ZAHLUNGSDATUM - RECHNUNGSDATUM AS Zahlungsdauer
            FROM KPMG.RECHNUNGEN
        ) a ON f.AUFTRAGSNUMMER = a.AUFTRAGSNUMMER
    ) r ON k.KUNDENNUMMER = r.KUNDENNUMMER
) b
GROUP BY b.LAND
ORDER BY durchschnit_Zahlungsdauer DESC;
```

---

## FR 03: Debitoren, die das Skonto am häufigsten nutzen
Ermittlung der Kunden, die Skonto bevorzugen.
```sql
SELECT Kunde, COUNT(Kunde) AS count
FROM (
    SELECT KUNDENNUMMER AS Kunde, (ZAHLUNGSDATUM - BUCHUNGSDATUM) AS Zahlungszeitraum
    FROM KPMG.FAKTURA
    WHERE (ZAHLUNGSDATUM - BUCHUNGSDATUM) <= (
        SELECT TAGE_BIS - SKONTO.TAGE_VON AS Zeitraum
        FROM KPMG.SKONTO
        WHERE SKONTO != 0
    )
)
GROUP BY Kunde
ORDER BY count DESC;
```

---

## FR 04: Stornierungen in Relation zum Absatz
Analyse der Länder mit der höchsten Storno-zu-Absatz-Relation.
```sql
SELECT a.LAND, Absatz, Storni, Storni / Absatz AS Relation_Storno_Absatz
FROM (
    SELECT LAND, COUNT(LAND) AS Absatz
    FROM (
        SELECT LAND
        FROM KPMG.KUNDEN
        WHERE KUNDENNUMMER IN (
            SELECT KUNDENNUMMER
            FROM KPMG.FAKTURA
            WHERE STORNIERT = 'Nein'
        )
    )
    GROUP BY LAND
) a
INNER JOIN (
    SELECT LAND, COUNT(LAND) AS Storni
    FROM (
        SELECT LAND
        FROM KPMG.KUNDEN
        WHERE KUNDENNUMMER IN (
            SELECT KUNDENNUMMER
            FROM KPMG.FAKTURA
            WHERE STORNIERT = 'Ja'
        )
    )
    GROUP BY LAND
) b ON a.LAND = b.LAND
ORDER BY Relation_Storno_Absatz DESC;
```

---

## FR 05: Häufig verkaufte Fahrzeugmerkmale (Farbe)
Analyse der Farben, die am häufigsten verkauft wurden.
```sql
SELECT FARBE, Anzahl
FROM (
    SELECT FARBE, COUNT(FARBE) AS Anzahl
    FROM KPMG.FAHRZEUGE
    GROUP BY FARBE
    ORDER BY Anzahl DESC
);
```

---

## FR 06: Preissegmente der meisten Verkäufe
Ermittlung des Preissegments, in dem die meisten Verkäufe stattfinden.
```sql
SELECT BauID, Anzahl, LISTENPREIS 
FROM (
    SELECT BauID, COUNT(BauID) AS Anzahl
    FROM (
        SELECT SUBSTR(FIN, 3, 4) AS BauID
        FROM KPMG.FAHRZEUGE
    )
    GROUP BY BauID
    ORDER BY Anzahl DESC
) x
LEFT JOIN (
    SELECT BAUREIHE, LISTENPREIS 
    FROM KPMG.BAUREIHEN
) b ON BauID = BAUREIHE;
```

---

## FR 07: Wirtschaftlich wertvollste Baureihe
Analyse der wirtschaftlich wertvollsten Baureihe anhand von Gewinn und Verkaufszahlen.
```sql
SELECT BAUREIHE, 
    (LISTENPREIS - BAUREIHEN.PRODUKTIONSPREIS) AS "Gewinn pro Fahrzeug", 
    "Anzahl an Verkäufen", 
    ROUND((LISTENPREIS - BAUREIHEN.PRODUKTIONSPREIS) * "Anzahl an Verkäufen") AS "Gesamter Gewinn"
FROM KPMG.BAUREIHEN
INNER JOIN (
    SELECT BauID, COUNT(BauID) AS "Anzahl an Verkäufen"
    FROM (
        SELECT SUBSTR(FIN, 3, 4) AS BauID
        FROM KPMG.FAHRZEUGE
    )
    GROUP BY BauID
) anzahlTab ON BAUREIHE = BauID
ORDER BY "Gesamter Gewinn" DESC;
```

---

## FR 08: Skonto fälschlicherweise berechnet
Identifikation von Fahrzeugen, bei denen Skonto fehlerhaft angewendet wurde.
```sql
SELECT FIN, 
    ZAHLUNGSDATUM - BUCHUNGSDATUM AS "Zeitraum der Zahlung", 
    VERKAUFSPREIS, 
    LISTENPREIS AS "eigentlicher Listenpreis"
FROM KPMG.FAKTURA
INNER JOIN (
    SELECT BAUREIHE, LISTENPREIS
    FROM KPMG.BAUREIHEN
) ON SUBSTR(FIN, 3, 4) = BAUREIHE
WHERE VERKAUFSPREIS IN (
    SELECT LISTENPREIS * 0.98 AS skontopreis
    FROM KPMG.BAUREIHEN
)
AND (ZAHLUNGSDATUM - BUCHUNGSDATUM) > 14;
```

---

## FR 09: Prüfung des 3-Wege-Match
Überprüfung, ob der 3-Wege-Match erfolgreich ist.
```sql
SELECT *
FROM KPMG.FAKTURA
WHERE AUFTRAGSNUMMER NOT IN (
    SELECT BESTELLUNGEN.AUFTRAGSNUMMER
    FROM KPMG.BESTELLUNGEN
)
OR AUFTRAGSNUMMER NOT IN (
    SELECT RECHNUNGEN.AUFTRAGSNUMMER
    FROM KPMG.RECHNUNGEN
);
```

---

## FR 10: Prüfung auf Cut-Off-Fälle
Ermittlung von Fällen, bei denen das Lieferdatum und das Buchungsdatum in unterschiedlichen Jahren liegen.
```sql
SELECT FIN, BUCHUNGSDATUM, LIEFERDATUM
FROM KPMG.FAKTURA
WHERE EXTRACT(YEAR FROM LIEFERDATUM) != EXTRACT(YEAR FROM BUCHUNGSDATUM);
```

---
