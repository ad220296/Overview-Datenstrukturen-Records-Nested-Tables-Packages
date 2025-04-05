# PLSQL-Structures-Guide

---
## 🔗 Direkt zu den Beispielen

- 📘 [Records & Rowtypes](#-21-records)
- 🧩 [Nested Tables (Array & Hashed)](#-22-nested-tables)
- 📦 [Packages & Sichtbarkeit](#-23-package)

---

# 📦 Datenstrukturen & Packages in PL/SQL – Übersicht

---

## 🧠 2.0 Allgemeine Grundlagen

### 🛠️ Deklaration & Instanziierung

Datenstrukturen wie Records und Nested Tables können:
- **lokal** in einem PL/SQL-Block deklariert werden
- **zentral** in einem Package Header (öffentlich) oder Body (privat)
- **sofort instanziiert** werden oder später mit Methoden wie `.EXTEND` vorbereitet werden

```sql
DECLARE
  TYPE my_record_t IS RECORD (id NUMBER, name VARCHAR2(100));
  rec my_record_t;
BEGIN
  rec.id := 1;
  rec.name := 'Max Mustermann';
END;
```

```sql
DECLARE
  TYPE num_list_t IS TABLE OF NUMBER;
  mylist num_list_t := num_list_t();
BEGIN
  mylist.EXTEND;
  mylist(1) := 100;
END;
```

### 🔁 Übergabe & Rückgabe

Benutzerdefinierte Typen können in Funktionen und Prozeduren **als Parameter übergeben oder zurückgegeben** werden.

```sql
CREATE OR REPLACE PACKAGE mytypes AS
  TYPE salary_rec IS RECORD (empno NUMBER, sal NUMBER);
  FUNCTION bonus(rec IN salary_rec) RETURN NUMBER;
END;

CREATE OR REPLACE PACKAGE BODY mytypes AS
  FUNCTION bonus(rec IN salary_rec) RETURN NUMBER IS
  BEGIN
    RETURN rec.sal * 0.1;
  END;
END;
```

### ➕ Verhalten bei Zuweisung & Übergabe

- Records und Nested Tables werden **by-value** kopiert (nicht referenziert)
- Änderungen an Kopien betreffen nicht das Original

```sql
my2 := my1; -- Kopie, keine Referenz
my2.feld := my2.feld + 1; -- my1 bleibt unverändert
```

---

## 📘 2.1 Records

Ein **Record** ist ein zusammengesetzter Datentyp (wie eine Struktur in C), der mehrere Werte in einem einzigen Objekt zusammenfasst.

| Typ                   | Beschreibung                                   | Beispiel                         |
|------------------------|-----------------------------------------------|----------------------------------|
| `%ROWTYPE`            | Automatischer Typ basierend auf Tabellenzeile | `my_emp EMP%ROWTYPE`             |
| Selbst definierter Typ| Benutzerdefiniert mit `TYPE IS RECORD`        | `TYPE anonemp_t IS RECORD (...)` |

➡️ **Records werden by-value übergeben** (Kopie) und sind sehr nützlich für Funktionen mit mehreren Parametern.

```sql
DECLARE
    my_emp EMP%ROWTYPE;
    TYPE anonemp_t IS RECORD (empnum NUMBER, empsal NUMBER);
    my_anon_emp anonemp_t;
    my_anon_emp2 anonemp_t;
BEGIN
    SELECT * INTO my_emp FROM EMP FETCH FIRST ROW ONLY;
    SELECT empno, sal INTO my_anon_emp FROM EMP FETCH FIRST ROW ONLY;
    my_anon_emp2 := my_anon_emp;
    my_anon_emp2.empsal := my_anon_emp2.empsal + 100;
    dbms_output.put_line(my_anon_emp.empsal);
    dbms_output.put_line(my_anon_emp2.empsal);
END;
```

---

## 🧩 2.2 Nested Tables

Eine **Nested Table** ist eine flexible Datenstruktur in PL/SQL – verwendbar als Array **oder** Hashmap.

### 🔹 Array-ähnlich (ohne `INDEX BY`)
- Elemente mit Ganzzahlen (Start bei 1)
- Methoden: `EXTEND`, `DELETE`, `FIRST`, `LAST`, `COUNT`

```sql
DECLARE
    TYPE name_table_t IS TABLE OF VARCHAR(32);
    name_table name_table_t;
BEGIN
    name_table := name_table_t();
    name_table.EXTEND;
    name_table(1) := 'Gerhard';
    name_table.EXTEND;
    name_table(name_table.LAST) := 'Clemens';
    FOR i IN name_table.FIRST .. name_table.LAST LOOP
        dbms_output.put_line(name_table(i));
    END LOOP;
END;
```

### 🔸 Hash-ähnlich (mit `INDEX BY`)
- Elemente über benutzerdefinierte Schlüssel (z. B. `VARCHAR`)

```sql
DECLARE
    TYPE name_sal_table_t IS TABLE OF NUMBER INDEX BY VARCHAR(32);
    name_sal_table name_sal_table_t := name_sal_table_t();
BEGIN
    name_sal_table('Gerhard') := 2000;
    name_sal_table('Anton') := 1000;
    name_sal_table.DELETE('Gerhard');
    dbms_output.put_line(name_sal_table('Anton'));
END;
```

### ⚡ BULK COLLECT mit Nested Table

```sql
DECLARE
    TYPE DeptRecTab IS TABLE OF dept%ROWTYPE;
    dept_recs DeptRecTab;
    CURSOR c1 IS SELECT deptno, dname, loc FROM dept WHERE deptno > 10;
BEGIN
    OPEN c1;
    FETCH c1 BULK COLLECT INTO dept_recs;
    CLOSE c1;
END;
```

---

## 📦 2.3 Package – Aufbau & Sichtbarkeit

Ein **Package** bündelt Funktionen, Prozeduren, Datentypen und Variablen logisch zusammen.

| Bereich         | Sichtbarkeit        | Inhalt                                |
|----------------|---------------------|----------------------------------------|
| Package Header | öffentlich (API)    | Typdefinitionen, Funktionssignaturen   |
| Package Body   | privat & implement. | Logik, interne Typen, Prozeduren       |

➡️ **Package-Variablen leben die ganze Session** (nützlich für globale Zustände)

### 🔧 Beispiel: Bonusberechnung mit Record im Package

```sql
CREATE OR REPLACE PACKAGE salman AS
    TYPE anonemp_t IS RECORD (empnum NUMBER, empsal NUMBER);
    FUNCTION CALCBONUS(anonemp anonemp_t) RETURN NUMBER;
END;

CREATE OR REPLACE PACKAGE BODY salman AS
    FUNCTION CALCBONUS(anonemp anonemp_t) RETURN NUMBER IS
    BEGIN
        IF anonemp.empnum > 5000 THEN
            RETURN anonemp.empsal + 200;
        ELSE
            RETURN anonemp.empsal + 100;
        END IF;
    END;
END;
```

```sql
-- Aufruf
DECLARE 
    myemp salman.anonemp_t;
    newsal NUMBER;
BEGIN
    myemp.empnum := 4999;
    myemp.empsal := 2000;
    newsal := salman.CALCBONUS(myemp);    
    dbms_output.put_line(newsal);
END;
```

---

## 🧩 Nested Tables in Compound Trigger

```sql
CREATE OR REPLACE TRIGGER no_avg_exceed FOR INSERT ON EMP COMPOUND TRIGGER
   TYPE deptno_avgsal_t IS TABLE OF NUMBER INDEX BY PLS_INTEGER;
   deptno_avgsal deptno_avgsal_t;

   BEFORE STATEMENT IS
   BEGIN
        FOR rec IN (SELECT deptno, AVG(sal) AS avgsal FROM emp GROUP BY deptno) LOOP
            deptno_avgsal(rec.deptno) := rec.avgsal;
        END LOOP;
   END BEFORE STATEMENT;

   BEFORE EACH ROW IS
   BEGIN
       IF :NEW.sal > deptno_avgsal(:NEW.deptno) THEN
         NULL; -- ggf. raise_application_error
       END IF;
   END BEFORE EACH ROW;
END;
```

---

## 🧾 Vergleich – Records, Nested Tables, Packages

| Merkmal                       | 📘 Records             | 🧩 Nested Tables                    | 📦 Package                         |
|------------------------------|------------------------|------------------------------------|------------------------------------|
| Struktur                     | fix / ROWTYPE          | dynamisch / flexibel               | logisch gruppiert                  |
| Übergabe an Subprogramme    | ✅ (by value)           | ✅                                  | ✅                                  |
| Initialisierung nötig       | nein (ROWTYPE) / ja    | ja (Array: `EXTEND`)               | automatisch (bei Sessionstart)     |
| Sichtbarkeit steuerbar      | nur bei Packages       | innerhalb Blöcken / Packages       | Header (öffentlich), Body (privat) |
| Ideal für                   | Gruppierte Werte       | Mengen (Arrays, Maps)              | Wiederverwendbare Module           |

---

## ✅ Empfehlung zur Anwendung

| Ziel / Situation                               | Geeignetes Mittel     |
|------------------------------------------------|------------------------|
| Datensatz mit mehreren Werten                 | 📘 Record              |
| Verarbeitung mehrerer Einträge                | 🧩 Nested Table        |
| Sammlung verwandter Funktionen & Typen        | 📦 Package             |
| Übergabe vieler Parameter als Objekt          | 📘 Record im Package   |
| Performanter Massenzugriff                    | 🧩 Nested Table + BULK |

---

**🔚 Ende der Übersicht – ideal für `README.md` im Haupt-Repo.**
