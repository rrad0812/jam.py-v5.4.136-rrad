
# Dokumentacija izmena Jam.py framework-a za PostgreSQL Boolean podršku

## Pregled

Ovaj dokument opisuje izmene u tri glavna fajla Jam.py framework-a (`sql.py`, `server_classes.py`, `dataset.py`) koje rešavaju problem kompatibilnosti sa PostgreSQL bazom podataka u vezi sa boolean tipom podataka.

## Problem koji se rešava

### Simptomi

PostgreSQL striktno proverava tipove podataka i baca grešku kada se pokuša dodeliti boolean vrednost (`False`) integer ili text koloni:

```sh
ERROR - column "mesta_troska_id" is of type integer but expression is of type boolean
LINE 1: ...E "masine" SET "aktivna"=false, "mesta_troska_id"=false, "na...
                                                             ^
HINT:  You will need to rewrite or cast the expression.
```

### Uzroci

1. **Value reuse bug:** Promenljiva `value` nije inicijalizovana za svako polje, što je dozvoljavalo da boolean vrednost iz jednog polja "curi" u sledeće
2. **False kao marker:** Aplikacijski kod koristi Python `False` kao placeholder za praznu vrednost
3. **Nekonzistentna boolean konverzija:** Različiti tipovi vrednosti (int, float, string) nisu bili konzistentno konvertovani u boolean za PostgreSQL

### 1. Izmene u `/jam/sql.py`

#### 1.1 Dodavanje @staticmethod dekoratora za `_to_bool()` metodu

```python
@staticmethod
def _to_bool(value):
    if value is None:
        return None
    if isinstance(value, bool):
        return value
    if isinstance(value, (int, float)):
        return False if value == 0 else True
    if isinstance(value, str):
        v = value.strip().lower()
        if v in ('1', 'true', 't', 'yes', 'y', 'on'):
            return True
        if v in ('0', 'false', 'f', 'no', 'n', 'off', ''):
            return False
        try:
            return float(v) != 0.0
        except Exception:
            return True
    return bool(value)
```

**Razlog:**

- Metoda je označena kao `@staticmethod` jer ne zavisi od instance stanja
- Može se koristiti kao pomoćna funkcija izvan instance konteksta (npr. u `server_classes.py`)
- Normalizuje različite tipove vrednosti (int, float, string, bool) u Python boolean

#### 1.2 Patch u `insert_sql()` metodi

```python
############################################################################
# Patch for Postgresql and boolean field type
############################################################################

# if a non-boolean field got a Python False convert it to None 
# so we don't assign a boolean to an integer/text column

_val = field.data
if _val is False and field.data_type != consts.BOOLEAN:
    _val = None
value = (_val, field.data_type)
                
if field.data is False and field.data_type != consts.BOOLEAN:
    value = (None, field.data_type)

if field.data_type == consts.BOOLEAN:
    if db_module.DATABASE == 'POSTGRESQL':
        normalized = self._to_bool(field.data)
        value = (normalized, field.data_type)
    else:
        value = (field.data, field.data_type)

############################################################################
```

**Ključne tačke:**

- **Per-field inicijalizacija:** `value` se inicijalizuje za svako polje posebno
- **False → None konverzija:** Ako je `field.data` Python `False` ALI tip polja NIJE `consts.BOOLEAN`, konvertuje se u `None` (SQL NULL)
- **PostgreSQL boolean normalizacija:** Za boolean polja na PostgreSQL-u koristi `_to_bool()` za ispravnu vrednost
- **deleted_flag override:** `deleted_flag` polje MORA biti `0` (INTEGER), ne `None` i ne `False`

#### 1.3 Identičan patch u `update_sql()` metodi

**Implementacija:** Identična struktura kao u `insert_sql()` — osigurava konzistentnost između INSERT i UPDATE operacija.

### 2. Izmene u `/jam/server_classes.py`

#### 2.1 Boolean konverzija u `copy_database()` metodi

**Lokacija:** Unutar `copy_rows()` funkcije (linija ~955)

```python
elif field.data_type == consts.BOOLEAN:
    # Patch for PostgreSQL and boolean type of fields
    if db_module.DATABASE == 'POSTGRESQL':
        r[j] = SQL._to_bool(r[j])
    elif r[j]:
        r[j] = 1
    else:
        r[j] = 0
```

**Razlog:**

- Prilikom kopiranja baze podataka između različitih DB sistema (npr. MySQL → PostgreSQL)
- Koristi `SQL._to_bool()` za PostgreSQL da osigura ispravnu boolean vrednost
- Za druge baze (MySQL, SQLite) konvertuje u integer (1/0)

**Kontekst:** Ova funkcija se poziva kada administrator kopira podatke iz jedne baze u drugu (različiti DB tipovi).

### 3. Izmene u `/jam/dataset.py`

#### 3.1 Smart boolean konverzija u `set_value()` metodi

```python
if self.data_type == consts.BOOLEAN:
    
    # Use Python booleans for PostgreSQL, integers for other DBs
    is_postgres = False
    try:
        if self.owner and hasattr(self.owner, 'task') and self.owner.task:
            db_module = self.owner.task.db_module
            if db_module and hasattr(db_module, 'DATABASE'):
                is_postgres = db_module.DATABASE == 'POSTGRESQL'
    except Exception:
        pass
    if is_postgres:
        self.new_value = True if bool(value) else False
    else:
        self.new_value = 1 if bool(value) else 0
```

**Razlog:**

- Dinamički detektuje tip baze podataka
- **Za PostgreSQL:** koristi Python boolean (`True`/`False`)
- **Za ostale baze:** koristi integer (`1`/`0`)
- Safe fallback sa try-except blokom ako `db_module` nije dostupan

**Kontekst:** Ova metoda se poziva svaki put kada se postavlja vrednost boolean polja kroz aplikacijski kod.

### Tehnički detalji rešenja

#### Problem 1: Value Reuse Bug

**Pre izmena:**

```python
# value nije bio inicijalizovan svaki put
if field.data_type == consts.BOOLEAN:
    value = (...)  # value se postavlja samo za boolean
# ako polje nije boolean, value ostaje sa vrednošću iz prethodne iteracije!
row.append(value)  # BUG: stara vrednost se reuse-uje
```

**Posle izmena:**

```python
# value se UVEK inicijalizuje na početku
_val = field.data
if _val is False and field.data_type != consts.BOOLEAN:
    _val = None
value = (_val, field.data_type)  # default vrednost za SVA polja

if field.data_type == consts.BOOLEAN:
    # override samo za boolean
    ...
row.append(value)  # ✓ Svako polje ima svoju vrednost
```

### Problem 2: False kao marker za praznu vrednost

**Situacija:** Aplikacijski kod često postavlja `False` kao marker za prazno polje:

```python
item.field_integer.data = False  # "nema vrednosti"
```

**Rešenje:** Eksplicitna konverzija `False` → `None` za ne-boolean polja:

```python
if _val is False and field.data_type != consts.BOOLEAN:
    _val = None  # SQL NULL
```

### Problem 3: PostgreSQL strict typing

**PostgreSQL očekivanja:**

- INTEGER kolona: prima samo integer ili NULL
- BOOLEAN kolona: prima samo `true`/`false`/`NULL`
- Neće prihvatiti boolean za integer kolonu

**Rešenje:**

- Boolean polja na PostgreSQL dobijaju Python `True`/`False`
- Boolean polja na drugim bazama dobijaju `1`/`0`
- Ne-boolean polja nikada ne dobijaju `False`, već `None` (NULL)

## Testiranje

### Test 1: INSERT sa praznim integer poljem

```python
item.insert()
item.field_integer.data = False  # marker za "prazno"
item.field_boolean.data = False   # stvarni boolean
item.post()
item.apply()

# Očekivano:
# - field_integer → NULL u bazi
# - field_boolean → False (0 ili false zavisno od DB-a)
```

### Test 2: UPDATE postojećeg zapisa

```python
item.edit()
item.field_integer.data = False
item.post()
item.apply()

# Očekivano:
# - field_integer → NULL u bazi
```

### Test 3: deleted_flag polje

```python
item.edit()
# deleted_flag se interno setuje na 0
item.post()
item.apply()

# Očekivano:
# - deleted_flag → 0 u bazi (NE NULL)
```

### Test 4: Copy database (MySQL → PostgreSQL)

```python
task.copy_database('POSTGRESQL', database='target_db', ...)
```

Očekivano:

- Boolean vrednosti iz MySQL (0/1) konvertovane u PostgreSQL (false/true)
- Bez type mismatch grešaka

## Kompatibilnost

| Baza podataka | Status | Napomena |
|--------------|--------|----------|
| **PostgreSQL** | ✅ Potpuna podrška | Sve promene su dizajnirane za PostgreSQL strict typing |
| **SQLite** | ✅ Kompatibilno | Koristi integer (0/1) za boolean |
| **MySQL** | ✅ Kompatibilno | Koristi integer (0/1) za boolean |
| **MSSQL** | ✅ Kompatibilno | Koristi integer (0/1) za boolean |
| **Firebird** | ⚠️ Potrebno testiranje | Teorijski kompatibilno |
| **Oracle** | ⚠️ Potrebno testiranje | Teorijski kompatibilno |

## Preporuke za dalji rad

### 1. Unit testovi

Kreirati automatske testove koji proveravaju:

- False → NULL konverziju za ne-boolean polja
- Boolean normalizaciju za PostgreSQL
- deleted_flag = 0 logiku
- Copy database između različitih DB tipova

### 2. Code review

- Proveriti da li postoje slične situacije sa value reuse u drugim metodama
- Audit svih mesta gde se koristi `field.data = False`

### 3. Dokumentacija za developere

Dodati u developer guide:

```python
# ✗ NE postavljati False za ne-boolean polja
item.field_integer.data = False  

# ✓ Koristiti None za praznu vrednost
item.field_integer.data = None
```

### 4. Regression testovi

- Testirati na production kopiji baze
- Proveriti sve INSERT/UPDATE/DELETE operacije
- Validirati copy_database funkcionalnost

## Zaključak

Ove izmene rešavaju kritičan bug koji je sprečavao korišćenje Jam.py sa PostgreSQL bazom podataka. Rešenje je:

✅ **Backward kompatibilno** - postojeći kod za druge baze nastavlja da radi  
✅ **Forward kompatibilno** - omogućava korišćenje PostgreSQL-a  
✅ **Minimalne izmene** - samo tri fajla, jasno označene sekcije  
✅ **Safe fallback** - try-except blokovi sprečavaju runtime greške  

---
**Autor:** Radosav  
**Datum:** 12. novembar 2025.  
**Verzija:** 1.0
