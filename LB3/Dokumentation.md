# M141 LB3-Praxisarbeit: Backpacker_LB3 migrieren

[TOC]

# 1. Definition Infrastruktur
## 1.1 Anforderungsdefinition

Die bestehende Access-Datenbank "Backpacker_LB3" wird bis 08.05.2025 in eine MariaDB-Datenbank auf AWSin eine EC2-Instanz migriert. Die Struktur (DDL) und Daten (CSV) werden lokal getestet, bereinigt (Pandas), mit SQL-Rollen abgesichert und automatisiert in die Cloud überführt.

## 1.2 Evaluation 3 RDBMS

| Kriterium            | AWS RDS                     | AWS EC2 mit selbstgehosteter DB | PostgreSQL auf lokalem Server |
|----------------------|---------------------------|--------------------------------|------------------------------|
| **Leistung**        | Gute Performance, aber begrenzt durch die Instanzwahl | Optimierbar für maximale Performance | Sehr leistungsfähig, aber abhängig von Hardware |
| **Skalierbarkeit**  | Automatische Skalierung, aber eingeschränkt | Volle Kontrolle über Skalierung | Eingeschränkte Skalierungsmöglichkeiten |
| **Flexibilität**    | Eingeschränkte Anpassbarkeit | Maximale Kontrolle und Konfigurationsmöglichkeiten | Hohe Anpassbarkeit, aber erfordert manuelle Pflege |
| **Kosten**         | Höhere Kosten für leistungsstarke Instanzen | Kostenoptimierung möglich durch eigene Verwaltung | Keine Cloud-Kosten, aber Hardware- und Wartungskosten |
| **Sicherheit**      | Eingebaute Sicherheitsmechanismen und Compliance | Volle Kontrolle über Sicherheitsmaßnahmen | Sicherheit abhängig von eigener Implementierung |
| **Administration**  | Automatisierte Verwaltung | Erfordert manuelle Wartung | Eigene Verwaltung erforderlich |
| **Fazit**           | Bequem für standardisierte Anwendungen | Beste Option für flexible, leistungsstarke Lösungen | Gute Open-Source-Alternative ohne Cloud-Abhängigkeit |

# 2. Lokale DBMS
## 2.1 Zugriffsmatrix

| _DB backpacker_lb3_                |              |       |       |       |
| ---------------------------------- | ------------ | ----- | ----- | ----- |
| _Benutzergruppe:_                  | **Benutzer** |       |       |       |
| Tabellen - Attribute               | S            | I     | U     | D     |
| tbl_personen                       | **x**        |       | **x** |       |
| tbl_benutzer                       |              |       |       |       |
| **-** Passwort                     | --           | --    | --    | --    |
| **-** deaktiviert                  | **x**        | --    | --    | --    |
| **-** restliche Attribute          | **x**        | **x** | **x** | --    |
| tbl_buchungen, <br> tbl_positionen | **x**        | **x** | **x** | **x** |
| tbl_land, <br> tbl_leistung        | **x**        |       |       |       |

| _DB backpacker_lb3_                |                |       |       |       |
| ---------------------------------- | -------------- | ----- | ----- | ----- |
| _Benutzergruppe:_                  | **Management** |       |       |       |
| Tabellen - Attribute               | S              | I     | U     | D     |
| tbl_positionen, <br> tbl_buchungen | **x**          |       |       |       |
| restl. Tabellen                    | **x**          | **x** | **x** | **x** |

_S = Select, I = Insert, U = Update, D = Delete, -- = nicht möglich, - = nicht mehr möglich_

## 2.2Lokales DBMS einrichten (MariaDB)
### 2.2.1 Installation (wenn nötig)

* Linux: `sudo apt install mariadb-server`
* Windows: XAMPP starten → MySQL verwenden

### 2.2.2 Datenbank erstellen

```sql
CREATE DATABASE backpacker_lb3;
USE backpacker_lb3;
```

## 2.3 CSV-Bereinigung mit Pandas

Um die CSV automatisiert zu bereeinigen haben wir ein Python Skript gemacht, welches die gegebenen CSV überprüft und dann bereeinigt Dem entsprechend haben wir auch Ordner gemacht.
**Code clean_csvs.py:**

```python
import pandas as pd
import os

input_dir = "./raw_csv"
output_dir = "./csv_cleaned"
os.makedirs(output_dir, exist_ok=True)

for filename in os.listdir(input_dir):
    if filename.endswith(".csv"):
        df = pd.read_csv(os.path.join(input_dir, filename))

        df.dropna(how='all', inplace=True)

        for col in df.select_dtypes(include='object').columns:
            df[col] = df[col].astype(str).str.strip()

        df.to_csv(os.path.join(output_dir, filename), index=False)
        print(f"{filename} bereinigt.")
```

## 2.4 Daten Import
### 2.4.1 Tabellenstruktur (DDL)
Iport Datei von Herr Kellenberger:

``mysql -u root -p backpacker_lb3 < <pfad>/backpacker_ddl_lb3.sql``

### 2.4.2 CSV Import (DML)

```sql
LOAD DATA LOCAL INFILE '<pfad>/cleaned_csv/cleaned_tbl_benutzer.csv'
INTO TABLE tbl_benutzer;
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
ESCAPED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES;
```

Dies dann auch für die anderen Tabellen machen.

## 2.5 Benutzergruppen mit Rollen (DCL)

```sql
-- Rollen erstellen
CREATE ROLE 'benutzer_role';
CREATE ROLE 'manager_role';

-- Rechte auf Rollen vergeben (gemäss Zugriffsmatrix)
GRANT SELECT, INSERT ON tbl_personen TO 'benutzer_role';
GRANT SELECT ON tbl_benutzer TO 'benutzer_role';
GRANT SELECT, INSERT, UPDATE, DELETE ON tbl_buchung TO 'benutzer_role';
GRANT SELECT, INSERT, UPDATE, DELETE ON tbl_positionen TO 'benutzer_role';
GRANT SELECT ON tbl_land TO 'benutzer_role';
GRANT SELECT ON tbl_leistung TO 'benutzer_role';

GRANT ALL PRIVILEGES ON backpacker_lb3.* TO 'manager_role';

-- Benutzer erstellen
CREATE USER 'benutzer'@'%' IDENTIFIED BY 'TBZ12345';
CREATE USER 'manager'@'%' IDENTIFIED BY 'TBZ12345';

-- Rollen zuweisen
GRANT 'benutzer_role' TO 'benutzer'@'%';
GRANT 'manager_role' TO 'manager'@'%';
```

## 2.6 Struktur exportieren
```bash
mysqldump -u root backpacker_lb3 --no-data > ddl_cloud.sql
```

## 2.7 Daten exportieren

```bash
mysqldump -u root backpacker_lb3 --no-create-info > dml_cloud.sql
```

Dies dan später auf die Instanz laden mit SCP oder SFTP z.B.

# 3. Remote Cloud-DBMS
## 3.1 EC2-Instanz installieren
1. Im GUI nach EC2 suchen
2. Instanz starten/konfigurieren
3. Dort Standard Installtion mit Ubuntu 22.04 und Port 22 und 3306 öffnen für Inbound
4. Dann auf checks warten
5. Wenn alles gut, per SSH verbinden

## 3.2 MariaDB installieren
1. `sudo apt update`
2. `sudo apt upgrade`
3. `sudo apt install maria-db`

# 4. Automatisierte Migration
## 4.1 Berechtigungen
**.bat Skript:**
```bat
@echo off
echo Starte Migration der Benutzer und Rollen...

mysql -h <EC2_PUBLIC_IP> -P 3306 -u root backpacker_lb3 < <pfad>\dcl_with_roles.sql

echo Benutzer/Rollen-Migration abgeschlossen.
pause
```

Inhalt dcl_with_roles.sql:

```sql
-- Rollen erstellen
CREATE ROLE 'benutzer_role';
CREATE ROLE 'manager_role';

-- Rechte auf Rollen vergeben (gemäss Zugriffsmatrix)
GRANT SELECT, INSERT ON tbl_personen TO 'benutzer_role';
GRANT SELECT ON tbl_benutzer TO 'benutzer_role';
GRANT SELECT, INSERT, UPDATE, DELETE ON tbl_buchung TO 'benutzer_role';
GRANT SELECT, INSERT, UPDATE, DELETE ON tbl_positionen TO 'benutzer_role';
GRANT SELECT ON tbl_land TO 'benutzer_role';
GRANT SELECT ON tbl_leistung TO 'benutzer_role';

GRANT ALL PRIVILEGES ON backpacker_lb3.* TO 'manager_role';

-- Benutzer erstellen
CREATE USER 'benutzer'@'%' IDENTIFIED BY 'TBZ12345';
CREATE USER 'manager'@'%' IDENTIFIED BY 'TBZ12345';

-- Rollen zuweisen
GRANT 'benutzer_role' TO 'benutzer'@'%';
GRANT 'manager_role' TO 'manager'@'%';
```

## 4.2 Daten
### 4.2.1 DDL

**.bat Skript:**
```bat
@echo off
echo Starte Migration der Struktur...

REM Passwort wird manuell eingegeben
mysqldump -u root --no-data backpacker_lb3 > scripts\ddl_cloud.sql

mysql -h <EC2_PUBLIC_IP> -P 3306 -u admin -p backpacker_lb3 < scripts\ddl_cloud.sql

echo Struktur-Migration abgeschlossen.
pause
```
SQL-File aus dem vorherigen Export

### 4.2.2 DML

**.bat Skript:**
```bat
@echo off
echo Starte Migration der Daten...

mysqldump -u root --no-create-info backpacker_lb3 > scripts\dml_cloud.sql

mysql -h <EC2_PUBLIC_IP> -P 3306 -u admin -p backpacker_lb3 < scripts\dml_cloud.sql

echo Daten-Migration abgeschlossen.
pause
```

SQL-File aus dem vorherigen Export

## 4.3 Alle drei Schritte automatisch

Dies könnte man mit folgendem Skript
**run_all_migration.bat:**

```bat
@echo off
echo Starte Migration der Daten...

mysqldump -u root --no-create-info backpacker_lb3 > scripts\dml_cloud.sql

mysql -h <EC2_PUBLIC_IP> -P 3306 -u admin -p backpacker_lb3 < scripts\dml_cloud.sql

echo Daten-Migration abgeschlossen.
pause
```

# 5. Testprotokoll: Überprüfung der Benutzerberechtigungen für *backpacker_lb3*
## 5.1 Überprüfung der Berechtigungen für die Tabelle *tbl_benutzer* für *benutzer*

**Ziel:**
Überprüfen, ob der Benutzer benutzer Daten aus tbl_buchung abfragen kann, da ihm SELECT, UPDATE Berechtigungen erteilt wurden.

**Befehl:**

SELECT * FROM tbl_benutzer LIMIT 5;

Erwartetes Ergebnis: Der Benutzer sollte in der Lage sein, Daten aus tbl_benutzer zu lesen. Wenn der Benutzer keine Berechtigung hat, sollte eine Fehlermeldung erscheinen.

**Output:**



## 5.2 Überprüfung der Berechtigungen, da alles gehen solte für *manager*

**Ziel:**
Überprpfen ob der manager alles machen kann, da er Alle Rechte hat ausser für *tbl_positionen* und *tbl_buchung*

**Output:**

![image](https://github.com/user-attachments/assets/6f6a3fec-50e4-423f-8b70-94b39d12f570)





