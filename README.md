# 🎵 Harmoniq — Music Library Management API

Harmoniq is a full-stack **Music Library Management System** built with **FastAPI**, **SQLModel ORM**, and **PostgreSQL**. It demonstrates advanced database concepts including **Views**, **Stored Procedures**, **Triggers**, and **Transactions** — fulfilling all DBMS Mini Project evaluation requirements.

---

## 📋 Table of Contents

- [Features](#-features)
- [Tech Stack](#-tech-stack)
- [Database Schema](#-database-schema)
- [DBMS Implementation Details](#-dbms-implementation-details)
  - [Views](#1--views-2-views)
  - [Stored Procedures](#2--stored-procedures-2-procedures)
  - [Triggers](#3--triggers-2-triggers)
  - [Transactions](#4--transactions)
  - [SQL Queries](#5--sql-queries)
- [API Endpoints](#-api-endpoints)
- [Installation & Setup](#-installation--setup)
- [Project Structure](#-project-structure)
- [Contributors](#-contributors)

---

## ✨ Features

- 🎸 Full **CRUD** operations for Bands and Albums
- 🔗 **One-to-Many** relational mapping (Band → Albums)
- 🐘 **PostgreSQL** database with native enum types
- 👁️ **SQL Views** for genre-based band filtering
- ⚙️ **Stored Procedures** for complex queries and data operations
- 🔔 **Database Triggers** for automatic audit logging
- 💰 **Explicit Transactions** with `BEGIN` / `COMMIT` / `ROLLBACK`
- 📝 **Audit Log** system powered by triggers
- 🎨 **Frontend UI** with notebook-style design
- 📦 **Alembic** migrations for version-controlled schema management
- ✅ Input validation with **Pydantic**

---

## 🛠 Tech Stack

| Category       | Technology       | Purpose                          |
|----------------|------------------|----------------------------------|
| Framework      | FastAPI          | REST API development             |
| ORM            | SQLModel         | Pythonic database modeling        |
| Database       | PostgreSQL 14    | Production-grade RDBMS           |
| DB Driver      | psycopg2-binary  | PostgreSQL adapter for Python     |
| Migrations     | Alembic          | Schema version control            |
| Validation     | Pydantic         | Request/response data integrity   |
| Server         | Uvicorn          | ASGI server                       |
| Frontend       | HTML/CSS/JS      | Interactive web interface         |

---

## 🗄 Database Schema

### Entity-Relationship Diagram

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│    band       │       │    album      │       │   auditlog    │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │──┐    │ id (PK)      │       │ id (PK)      │
│ name (VARCHAR)│  │    │ title        │       │ band_name    │
│ genre (ENUM) │  └───>│ band_id (FK) │       │ action       │
│ date (DATE)  │       │ release_date │       │ timestamp    │
└──────────────┘       └──────────────┘       └──────────────┘
```

### Table Definitions

#### `band` Table
```sql
CREATE TABLE band (
    id SERIAL PRIMARY KEY,
    name VARCHAR NOT NULL,
    genre genrechoices NOT NULL,  -- PostgreSQL ENUM type
    date DATE
);

-- Custom ENUM type
CREATE TYPE genrechoices AS ENUM ('ROCK', 'ELECTRONIC', 'METAL', 'HIP_HOP');
```

#### `album` Table
```sql
CREATE TABLE album (
    id SERIAL PRIMARY KEY,
    title VARCHAR NOT NULL,
    release_date DATE NOT NULL,
    band_id INTEGER REFERENCES band(id)
);
```

#### `auditlog` Table
```sql
CREATE TABLE auditlog (
    id SERIAL PRIMARY KEY,
    band_name VARCHAR NOT NULL,
    action VARCHAR NOT NULL,        -- 'INSERT' or 'DELETE'
    timestamp TIMESTAMP NOT NULL
);
```

---

## 📚 DBMS Implementation Details

### 1. 👁️ Views (2 Views)

Views provide pre-filtered read-only perspectives of the `band` table.

#### View 1: `rock_bands_view`
```sql
CREATE OR REPLACE VIEW rock_bands_view AS
SELECT id, name, genre, date
FROM band
WHERE genre = 'ROCK';
```
**Purpose:** Returns only Rock genre bands. Accessed via `GET /views/rock_bands`.

#### View 2: `metal_bands_view`
```sql
CREATE OR REPLACE VIEW metal_bands_view AS
SELECT id, name, genre, date
FROM band
WHERE genre = 'METAL';
```
**Purpose:** Returns only Metal genre bands. Accessed via `GET /views/metal_bands`.

**File:** `db.py` (lines 19–33)

---

### 2. ⚙️ Stored Procedures (2 Procedures)

Stored procedures encapsulate complex business logic on the database server using PL/pgSQL.

#### Procedure 1: `get_bands_by_genre(p_genre)`
```sql
CREATE OR REPLACE FUNCTION get_bands_by_genre(p_genre VARCHAR)
RETURNS TABLE (
    band_id INTEGER,
    band_name VARCHAR,
    band_genre VARCHAR,
    album_count BIGINT
) AS $$
BEGIN
    RETURN QUERY
    SELECT b.id, b.name, b.genre::VARCHAR, COUNT(a.id)
    FROM band b
    LEFT JOIN album a ON a.band_id = b.id
    WHERE b.genre::VARCHAR = p_genre
    GROUP BY b.id, b.name, b.genre;
END;
$$ LANGUAGE plpgsql;
```
**Purpose:** Returns all bands of a given genre along with their album count using a `LEFT JOIN` and `GROUP BY`. Accessed via `GET /procedures/bands_by_genre/{genre}`.

#### Procedure 2: `transfer_albums(p_from_band_id, p_to_band_id)`
```sql
CREATE OR REPLACE FUNCTION transfer_albums(
    p_from_band_id INTEGER,
    p_to_band_id INTEGER
)
RETURNS TABLE (
    transferred_count INTEGER,
    from_band_name VARCHAR,
    to_band_name VARCHAR
) AS $$
DECLARE
    v_count INTEGER;
    v_from_name VARCHAR;
    v_to_name VARCHAR;
BEGIN
    -- Validate source band exists
    SELECT name INTO v_from_name FROM band WHERE id = p_from_band_id;
    IF v_from_name IS NULL THEN
        RAISE EXCEPTION 'Source band with id % not found', p_from_band_id;
    END IF;

    -- Validate target band exists
    SELECT name INTO v_to_name FROM band WHERE id = p_to_band_id;
    IF v_to_name IS NULL THEN
        RAISE EXCEPTION 'Target band with id % not found', p_to_band_id;
    END IF;

    -- Count albums to transfer
    SELECT COUNT(*)::INTEGER INTO v_count
    FROM album WHERE band_id = p_from_band_id;

    -- Transfer albums
    UPDATE album SET band_id = p_to_band_id
    WHERE band_id = p_from_band_id;

    -- Return result
    transferred_count := v_count;
    from_band_name := v_from_name;
    to_band_name := v_to_name;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;
```
**Purpose:** Atomically transfers all albums from one band to another with validation. Raises exceptions if either band doesn't exist. Accessed via `POST /procedures/transfer_albums?from_band_id=X&to_band_id=Y`.

**File:** `db.py` (lines 90–153)

---

### 3. 🔔 Triggers (2 Triggers)

Triggers automatically log band insertions and deletions into the `auditlog` table.

#### Trigger 1: `band_delete_audit` (AFTER DELETE)
```sql
CREATE OR REPLACE FUNCTION log_band_deletion()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO auditlog (band_name, action, timestamp)
    VALUES (OLD.name, 'DELETE', NOW());
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER band_delete_audit
AFTER DELETE ON band
FOR EACH ROW
EXECUTE FUNCTION log_band_deletion();
```
**Purpose:** Automatically logs the name of any deleted band into the audit log with a `DELETE` action tag.

#### Trigger 2: `band_insert_audit` (AFTER INSERT)
```sql
CREATE OR REPLACE FUNCTION log_band_insertion()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO auditlog (band_name, action, timestamp)
    VALUES (NEW.name, 'INSERT', NOW());
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER band_insert_audit
AFTER INSERT ON band
FOR EACH ROW
EXECUTE FUNCTION log_band_insertion();
```
**Purpose:** Automatically logs the name of any newly inserted band into the audit log with an `INSERT` action tag.

**File:** `db.py` (lines 36–83)

---

### 4. 💰 Transactions

Explicit transactions ensure atomicity — either all operations succeed, or none do.

#### Transaction 1: Create Band with Albums
```python
# POST /bands — Creates a band and its albums atomically
session.exec(text("BEGIN"))          # BEGIN TRANSACTION
band = Band(name=..., genre=...)
session.add(band)
session.flush()                       # Get band ID without committing
for album in albums:
    session.add(Album(..., band=band))
session.exec(text("COMMIT"))         # COMMIT TRANSACTION
# On error:
session.exec(text("ROLLBACK"))       # ROLLBACK TRANSACTION
```

#### Transaction 2: Bulk Create Multiple Bands
```python
# POST /transactions/bulk_create — Creates multiple bands in one transaction
session.exec(text("BEGIN"))          # BEGIN TRANSACTION
for band_data in bands_data:
    band = Band(name=..., genre=...)
    session.add(band)
    session.flush()
    for album in band_data.albums:
        session.add(Album(..., band=band))
session.exec(text("COMMIT"))         # COMMIT — all bands created
# On ANY error:
session.exec(text("ROLLBACK"))       # ROLLBACK — NO bands created
```

**Key Principle:** If creating the 3rd band fails, the 1st and 2nd bands are also rolled back. This guarantees **ACID** compliance.

**File:** `main.py` (lines 49–86, 160–196)

---

### 5. 📝 SQL Queries

| Operation | Query Type | Location |
|-----------|-----------|----------|
| List all bands | `SELECT` | `GET /bands` |
| Get band by ID | `SELECT ... WHERE id = ?` | `GET /bands/{id}` |
| Create band | `INSERT INTO band` | `POST /bands` |
| Update band | `UPDATE band SET ...` | `PUT /bands/{id}` |
| Delete band | `DELETE FROM band WHERE id = ?` | `DELETE /bands/{id}` |
| Rock bands view | `SELECT * FROM rock_bands_view` | `GET /views/rock_bands` |
| Metal bands view | `SELECT * FROM metal_bands_view` | `GET /views/metal_bands` |
| Genre procedure | `SELECT * FROM get_bands_by_genre(?)` | `GET /procedures/bands_by_genre/{genre}` |
| Transfer procedure | `SELECT * FROM transfer_albums(?, ?)` | `POST /procedures/transfer_albums` |
| Audit logs | `SELECT ... FROM auditlog ORDER BY timestamp DESC` | `GET /logs` |

---

## 🌐 API Endpoints

### CRUD Operations
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bands` | List all bands (supports `?genre=`, `?q=` filters) |
| `GET` | `/bands/{id}` | Get a specific band by ID |
| `POST` | `/bands` | Create a new band (with optional albums) — **uses transaction** |
| `PUT` | `/bands/{id}` | Update band name/genre |
| `DELETE` | `/bands/{id}` | Delete a band — **fires delete trigger** |

### Views
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/views/rock_bands` | Query the `rock_bands_view` SQL view |
| `GET` | `/views/metal_bands` | Query the `metal_bands_view` SQL view |

### Stored Procedures
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/procedures/bands_by_genre/{genre}` | Call `get_bands_by_genre()` procedure |
| `POST` | `/procedures/transfer_albums?from_band_id=X&to_band_id=Y` | Call `transfer_albums()` procedure |

### Transactions
| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/transactions/bulk_create` | Bulk create bands in a single transaction |

### Audit Logs
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/logs` | View trigger-generated audit logs |

---

## 🚀 Installation & Setup

### Prerequisites
- Python 3.10+
- PostgreSQL 14+
- pip / venv

### 1. Clone the Repository
```bash
git clone https://github.com/GrandRegentSarva/Harmoniq-Music_Library_Management_API.git
cd Harmoniq-Music_Library_Management_API
```

### 2. Create Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate   # Linux/Mac
# venv\Scripts\activate    # Windows
```

### 3. Install Dependencies
```bash
pip install fastapi sqlmodel uvicorn alembic psycopg2-binary
```

### 4. Setup PostgreSQL Database
```bash
# Start PostgreSQL
sudo systemctl start postgresql

# Create database and user
sudo -u postgres psql -c "CREATE DATABASE harmoniq;"
sudo -u postgres psql -c "CREATE USER harmoniq_user WITH PASSWORD 'harmoniq123';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE harmoniq TO harmoniq_user;"
sudo -u postgres psql -d harmoniq -c "GRANT ALL ON SCHEMA public TO harmoniq_user;"
```

### 5. Run the Application
```bash
uvicorn main:app --reload
```

The app will:
1. Create all tables (`band`, `album`, `auditlog`)
2. Create views (`rock_bands_view`, `metal_bands_view`)
3. Create triggers (`band_delete_audit`, `band_insert_audit`)
4. Create stored procedures (`get_bands_by_genre`, `transfer_albums`)
5. Seed initial data (4 bands + 2 albums)

### 6. Access the Application
- **Frontend:** http://localhost:8000
- **API Docs (Swagger):** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc

---

## 📁 Project Structure

```
Harmoniq-Music_Library_Management_API/
├── main.py                  # FastAPI app, routes, endpoints
├── models.py                # SQLModel/Pydantic schemas
├── db.py                    # DB engine, init (views, triggers, procedures)
├── alembic.ini              # Alembic configuration (PostgreSQL)
├── api.http                 # HTTP request templates for testing
├── frontend/
│   ├── index.html           # Main UI page
│   ├── script.js            # Frontend logic (CRUD, views, procedures)
│   └── style.css            # Notebook-style CSS theme
├── migrations/
│   ├── env.py               # Alembic environment
│   ├── script.py.mako       # Migration template
│   └── versions/            # Migration history
│       ├── 8cafc0fb6ccc_initial_generation.py
│       ├── f2fe2c5cdace_initial_generation.py
│       └── 2f8f10034ccf_add_auditlog_schema.py
└── README.md                # This file
```

---

## 👥 Contributors

| Name | Role |
|------|------|
| Sarva Dubey | Backend Development, Database Design, API Architecture |
| Samiksha | Frontend Design & Development |

---

## 📄 License

This project is developed as part of the **DBMS Mini Project** coursework.
