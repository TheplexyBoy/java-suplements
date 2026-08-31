# VetCareSystem

A layered Java SE console application (JOptionPane-based UI) for managing
**Owners (Propietarios)** and **Pets (Mascotas)** for a veterinary clinic,
built as a mock technical assessment ("simulacro de prueba técnica") to
practice a strict Layered / MVC-style architecture with pure JDBC.

> **Author:** Andrés Felipe Quintero Hernández

---

## 1. Project context

This project is a focused implementation of the **VetCare** case study
(clinic management system originally scoped to owners, pets, veterinarians,
appointments, medications and users). This build implements the **Owner**
and **Pet** modules end‑to‑end, following every architectural constraint
required by the original assessment brief:

- Java SE, layered architecture, OOP, interfaces as contracts.
- JDBC persistence with `PreparedStatement` (no ORM, no frameworks).
- Business validations enforced in the Service layer.
- A modal menu-driven UI built exclusively with `JOptionPane`.
- Clear separation between presentation, business logic, and persistence
  (no SQL is ever executed from the Controller or the Presentation layer).

### Functional requirements covered

**Owner management**
- Register, list, update, search by identification number.
- Activate / deactivate (soft delete).
- View all pets belonging to an owner.
- Validations: unique identification number, unique email, mandatory
  name and phone, inactive owners cannot register new pets.

**Pet management**
- Register (linked to an existing, active owner), list, search by name,
  update, activate/deactivate.
- Validations: must belong to an existing **active** owner, weight must be
  greater than zero, birth date cannot be in the future, no duplicate pet
  (same name + birth date + owner).

---

## 2. Tech stack

| Layer            | Technology                                   |
|-------------------|-----------------------------------------------|
| Language          | Java SE 17+                                   |
| IDE               | Apache NetBeans (plain Java project, Ant-based)|
| Database          | PostgreSQL                                    |
| Persistence       | Pure JDBC (`PreparedStatement`, `ResultSet`)  |
| UI                | `javax.swing.JOptionPane`                     |
| Architecture      | Layered / MVC (Model, DAO/Impl, Service, Controller, Presentation) |

No external frameworks (no Spring, no Hibernate, no build tool plugins
beyond NetBeans' own Ant project) — everything is done with plain JDK
classes plus the PostgreSQL JDBC driver.

---

## 3. Project structure

```
VetCareSystem/
├── README.md
├── sql/
│   └── schema.sql                     # DDL + sample data for PostgreSQL
└── src/
    └── com/vetcare/
        ├── model/                     # Plain data classes (POJOs)
        │   ├── Propietario.java
        │   └── Mascota.java
        │
        ├── exception/                 # Custom exceptions
        │   ├── ValidacionException.java
        │   └── PersistenciaException.java
        │
        ├── config/                    # JDBC connection factory
        │   └── ConexionBD.java
        │
        ├── dao/                       # Persistence contracts (interfaces)
        │   ├── PropietarioDAO.java
        │   ├── MascotaDAO.java
        │   └── impl/                  # JDBC implementations
        │       ├── PropietarioDAOImpl.java
        │       └── MascotaDAOImpl.java
        │
        ├── service/                   # Business rules & validations
        │   ├── PropietarioService.java
        │   └── MascotaService.java
        │
        ├── controller/                # Exception handling / orchestration
        │   ├── PropietarioController.java
        │   └── MascotaController.java
        │
        └── presentation/              # JOptionPane UI
            └── Main.java
```

### Data flow (how a request travels through the layers)

```
Main.java (JOptionPane)
   │  builds a Propietario/Mascota object from user input
   ▼
Controller (PropietarioController / MascotaController)
   │  calls the Service and catches any exception it throws
   ▼
Service (PropietarioService / MascotaService)
   │  applies business rules/validations, then delegates persistence
   ▼
DAO interface (PropietarioDAO / MascotaDAO)
   │  contract implemented by...
   ▼
DAO Impl (PropietarioDAOImpl / MascotaDAOImpl)
   │  runs PreparedStatement SQL against PostgreSQL via ConexionBD
   ▼
PostgreSQL database
```

Every code file in `src/` is heavily commented **in Spanish**, explaining
not just *what* the code does but *why* each pattern (try-with-resources,
`PreparedStatement`, `Optional`, custom exceptions, DTO-like `Resultado<T>`
wrapper, etc.) is used — intended as a learning resource for a beginner
studying layered architecture.

---

## 4. Prerequisites

- **JDK 17 or higher** installed and configured in NetBeans.
- **Apache NetBeans** (any recent version with Java SE support).
- **PostgreSQL** server installed and running locally (or reachable).
- **PostgreSQL JDBC Driver** (`postgresql-<version>.jar`), downloadable
  from [https://jdbc.postgresql.org/download/](https://jdbc.postgresql.org/download/).

---

## 5. Database setup

1. Open **pgAdmin** (or `psql`) and connect to your local PostgreSQL server.
2. Run the first statement of `sql/schema.sql` to create the database:
   ```sql
   CREATE DATABASE vetcare_db;
   ```
3. **Connect to `vetcare_db`** (important — the rest of the script must run
   inside this database, not inside `postgres`).
4. Run the rest of `sql/schema.sql` to create the `propietarios` and
   `mascotas` tables (it also inserts one sample owner and one sample pet
   so you can test the app immediately).

### Configure the connection credentials

Open `src/com/vetcare/config/ConexionBD.java` and adjust these constants
to match your local PostgreSQL installation:

```java
private static final String URL = "jdbc:postgresql://localhost:5432/vetcare_db";
private static final String USUARIO = "postgres";
private static final String PASSWORD = "postgres";
```

---

## 6. Setting up the project in NetBeans

1. **File → New Project → Java with Ant → Java Application**, then point
   it to (or create it inside) the `VetCareSystem` folder, keeping the
   `src` layout shown above.
2. Add the PostgreSQL driver as a library:
   - Right-click the project → **Properties → Libraries → Compile → Add JAR/Folder**.
   - Select the downloaded `postgresql-<version>.jar`.
3. Set `com.vetcare.presentation.Main` as the **Main Class** of the project
   (Project Properties → Run → Main Class).
4. Make sure the project's **Source/Binary Format** is Java 17 or higher
   (Project Properties → Sources).
5. Run the project (**F6** in NetBeans, or right-click → Run).

The first screen you'll see is the main `JOptionPane` menu, from which you
can navigate into **Owners** or **Pets** management.

---

## 7. Design notes worth knowing before reading the code

- **Interfaces as contracts:** `PropietarioDAO` / `MascotaDAO` are
  interfaces; `PropietarioDAOImpl` / `MascotaDAOImpl` are their JDBC
  implementations. The Service layer depends only on the interface,
  never on the concrete implementation.
- **Custom exceptions:** `ValidacionException` represents an expected
  business-rule violation (e.g., "duplicate email"); `PersistenciaException`
  wraps any `SQLException` so upper layers never depend on JDBC details.
- **`Resultado<T>`:** a small wrapper (declared inside `PropietarioController`
  and reused from `MascotaController`) that turns "success + data" or
  "failure + message" into a single object the Presentation layer can
  read with a simple `if (resultado.isExitoso())`, without needing its own
  try/catch blocks.
- **Soft delete:** owners and pets are never physically deleted — they are
  flagged `activo = false`, preserving referential integrity and history.

---

## 8. Possible next steps (out of scope for this delivery)

- Add the Veterinarian, Appointment, Medication, Medical Attention and
  User/Role modules described in the full VetCare case study.
- Introduce a connection pool (e.g., HikariCP) instead of opening a new
  `Connection` per operation.
- Add automated unit tests (JUnit) for the Service layer.
