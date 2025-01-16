# Kundensegmentierung für E-Commerce: Ask, Prepare, Process, Analyze

### Überblick
- **Datensatz**: E-Commerce-Transaktionen von 2009 - 2011, britischer Einzelhandel.
- **Ziel**: Kunden segmentieren, um gezielte Marketingstrategien zu entwickeln.

### 1. Fragen definieren (Ask)
- **Geschäftliches Ziel**: Umsatz durch gezieltes Kundenmarketing steigern.
- **Warum**: Durch die Identifizierung und Segmentierung wertvoller Kunden können maßgeschneiderte Marketingmaßnahmen entwickelt werden, die den Umsatz steigern.
- **Zentrale Fragen**:
  - Wer sind die wertvollsten Kunden?
  - Wie können Kunden basierend auf ihrem Kaufverhalten segmentiert werden?

### 2. Vorbereitung der Daten (Prepare)
- **Tabelle erstellen**: Erstellen Sie die Tabelle `online_retail_test` in SQL.
  ```sql
  CREATE TABLE online_retail_test (
    Invoice VARCHAR(10),
    StockCode VARCHAR(20),
    Description VARCHAR(255),
    Quantity INTEGER,
    InvoiceDate TIMESTAMP,
    Price NUMERIC(10, 2),
    CustomerID VARCHAR(10),
    Country VARCHAR(100)
  );
  ```
  - **Warum**: Diese Tabelle stellt die Struktur dar, in der die Daten gespeichert werden sollen.
  - **Ziel**: Eine geeignete Struktur erstellen, um die Daten korrekt zu speichern.

- **Datenimport**: Daten aus der CSV-Datei in die erstellte Tabelle importieren.
  ```sql
  COPY online_retail_test
  FROM 'I:\Datenanalyst\Projekte\online+retail+ii\online_retail_II_2009-2010.CSV'
  DELIMITER ';'
  CSV HEADER;
  ```
  - **Warum**: Der `COPY`-Befehl ermöglicht es, die CSV-Daten effizient in die Datenbank zu laden.
  - **Ziel**: Sicherstellen, dass alle Daten in der Tabelle `online_retail_test` geladen werden, um sie für die Analyse zu verwenden.

### 3. Datenverarbeitung (Process)
- **Umgang mit fehlenden Werten**:
  - **Identifizieren und Entfernen**: Fokus auf die `CustomerID`-Spalte.
    ```sql
    DELETE FROM online_retail_test WHERE CustomerID IS NULL;
    ```
    - **Warum**: Transaktionen ohne Kunden-ID sind für die Segmentierung nicht nützlich, da sie nicht einem bestimmten Kunden zugeordnet werden können.
    - **Ziel**: Sicherstellen, dass nur relevante und vollständige Daten verwendet werden.

- **Umgang mit Rücksendungen und internen Buchungen**:
  - **Rücksendungen und die dazugehörigen Bestellungen entfernen**:
    ```sql
    WITH original_orders AS (
        SELECT DISTINCT StockCode, Description, CustomerID, ABS(Quantity) AS OrderQuantity
        FROM online_retail_test
        WHERE Invoice LIKE 'C%'
    )
    DELETE FROM online_retail_test
    WHERE (StockCode, Description, CustomerID, ABS(Quantity)) IN (
        SELECT StockCode, Description, CustomerID, OrderQuantity
        FROM original_orders
    );
    ```
    - **Warum**: Wenn eine Bestellung storniert wird, sollen sowohl die Rücksendung als auch die ursprüngliche Bestellung entfernt werden, da die Transaktion am Ende keinen Umsatz generiert hat.
    - **Ziel**: Sicherstellen, dass die Daten nur gültige Verkäufe enthalten und keine Stornierungen oder fehlerhaften Transaktionen, um eine genaue Analyse zu gewährleisten.
  
  - **Interne Buchungen entfernen**:
    ```sql
    DELETE FROM online_retail_test
    WHERE StockCode IN ('POST', 'BANKCHARGE', 'DUMMY');
    ```
    - **Warum**: Interne Buchungen wie Porto, Überweisungsgebühren und Dummy-Einträge haben keinen direkten Bezug zu den Kundentransaktionen und verfälschen die Analyse.
    - **Ziel**: Nur umsatzrelevante Daten für die Analyse verwenden, um genaue Ergebnisse zu gewährleisten.

- **Korrekte Datentypen setzen**:
  - `InvoiceDate` in `TIMESTAMP` umwandeln, um Datum und Uhrzeit der Transaktion zu speichern.
    ```sql
    ALTER TABLE online_retail_test ALTER COLUMN "InvoiceDate" TYPE TIMESTAMP USING to_timestamp("InvoiceDate", 'DD.MM.YYYY HH24:MI');
    ```
    - **Warum**: Die Rechnungsdaten enthalten sowohl Datum als auch Uhrzeit, die für die Analyse wichtig sind.
    - **Ziel**: Zeitliche Analysen ermöglichen, z. B. zur Berechnung der Recency.
  - `Price` umwandeln, um die Dezimalstellen korrekt zu speichern.
    ```sql
    UPDATE online_retail_test SET "Price" = REPLACE("Price", ',', '.')::NUMERIC;
    ```
    - **Warum**: Preise enthalten Dezimalstellen und müssen präzise gespeichert werden. Das Ersetzen von Komma durch Punkt stellt sicher, dass PostgreSQL den Wert korrekt verarbeitet.
    - **Ziel**: Korrekte Preisberechnungen ermöglichen.

- **Datenumwandlung**:
  - `Revenue`-Spalte für jede Transaktion erstellen.
    ```sql
    ALTER TABLE online_retail_test ADD COLUMN "Revenue" NUMERIC(10, 2);
    UPDATE online_retail_test SET "Revenue" = "Quantity" * "Price";
    ```
    - **Warum**: Der Umsatz ist eine wichtige Kennzahl zur Analyse des Kundenwerts.
    - **Ziel**: Monetäre Kennzahlen für die Segmentierung berechnen.

### 4. Analyse (Analyze)
- **Explorative Datenanalyse (EDA)**:
  - **Statistiken berechnen**: Min-, Max-, Durchschnittsumsatz berechnen.
    ```sql
    SELECT MIN("Revenue"), MAX("Revenue"), AVG("Revenue") FROM online_retail_test;
    ```
    - **Warum**: Grundlegende Statistiken helfen dabei, ein Gefühl für die Daten zu bekommen und Anomalien zu erkennen.
    - **Ziel**: Überblick über Umsatzverteilung und -spannen erhalten.
  - **Verteilung verstehen**: Verteilung der Käufe analysieren.
    - **Warum**: Zu verstehen, wie die Umsätze verteilt sind, hilft bei der Segmentierung.
    - **Ziel**: Geeignete RFM-Schwellenwerte festlegen.

- **RFM-Metriken**:
  - **Recency**: Zeit seit dem letzten Kauf.
    ```sql
    SELECT "CustomerID", MAX("InvoiceDate") AS "LastPurchaseDate"
    FROM online_retail_test
    GROUP BY "CustomerID";
    ```
    - **Warum**: Zu wissen, wie kürzlich Kunden gekauft haben, hilft dabei, aktive und inaktive Kunden zu unterscheiden.
    - **Ziel**: Kunden nach ihrer Aktivität einordnen.
  - **Frequency**: Anzahl der Transaktionen pro Kunde.
    ```sql
    SELECT "CustomerID", COUNT("Invoice") AS "TransactionCount"
    FROM online_retail_test
    GROUP BY "CustomerID";
    ```
    - **Warum**: Kunden, die häufig kaufen, sind wertvoller.
    - **Ziel**: Kunden basierend auf ihrer Kaufhäufigkeit segmentieren.
  - **Monetary**: Gesamtausgaben pro Kunde.
    ```sql
    SELECT "CustomerID", SUM("Revenue") AS "TotalSpent"
    FROM online_retail_test
    GROUP BY "CustomerID";
    ```
    - **Warum**: Kunden, die mehr ausgeben, tragen stärker zum Umsatz bei.
    - **Ziel**: Kunden nach ihrem Wert für das Unternehmen einteilen.

- **RFM-Score berechnen**:
  - **Recency, Frequency und Monetary in Kategorien einteilen**:
    ```sql
    WITH rfm_values AS (
        SELECT "CustomerID",
               NTILE(4) OVER (ORDER BY MAX("InvoiceDate") DESC) AS RecencyScore,
               NTILE(4) OVER (ORDER BY COUNT("Invoice") DESC) AS FrequencyScore,
               NTILE(4) OVER (ORDER BY SUM("Revenue") DESC) AS MonetaryScore
        FROM online_retail_test
        GROUP BY "CustomerID"
    )
    SELECT "CustomerID", RecencyScore, FrequencyScore, MonetaryScore,
           (RecencyScore * 100 + FrequencyScore * 10 + MonetaryScore) AS RFMScore
    FROM rfm_values;
    ```
    - **Warum**: Die Einteilung in Kategorien ermöglicht eine Bewertung der Kunden basierend auf ihrem Kaufverhalten.
    - **Ziel**: Ein standardisierter RFM-Score hilft dabei, die Kunden in Segmente einzuordnen und ihre Bedeutung für das Unternehmen zu verstehen.

- **Kunden in Segmente aufteilen**:
  - **Kunden basierend auf dem RFM-Score segmentieren**:
    ```sql
    WITH rfm_segments AS (
        SELECT "CustomerID", RecencyScore, FrequencyScore, MonetaryScore,
               (RecencyScore * 100 + FrequencyScore * 10 + MonetaryScore) AS RFMScore,
               CASE
                   WHEN RecencyScore = 4 AND FrequencyScore >= 3 AND MonetaryScore >= 3 THEN 'Champions'
                   WHEN RecencyScore >= 3 AND FrequencyScore >= 3 THEN 'Loyal Customers'
                   WHEN RecencyScore = 4 THEN 'Recent Customers'
                   WHEN FrequencyScore >= 3 THEN 'Frequent Customers'
                   WHEN MonetaryScore >= 3 THEN 'Big Spenders'
                   ELSE 'At Risk'
               END AS Segment
        FROM rfm_values
    )
    SELECT * FROM rfm_segments;
    ```
    - **Warum**: Die Segmentierung hilft dabei, die Kunden nach ihrem Wert und Verhalten zu klassifizieren, um gezielte Marketingstrategien entwickeln zu können.
    - **Ziel**: Effektive Einteilung der Kunden in Gruppen wie 'Champions', 'Loyal Customers' oder 'At Risk', um eine spezifische Ansprache zu ermöglichen.
