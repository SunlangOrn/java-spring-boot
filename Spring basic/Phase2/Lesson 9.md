# 📘 Phase 2, Lesson 9: Flyway Migrations (The Professional Way)

## 🎯 Learning Goal
- ✅ Understand why `ddl-auto: update` is forbidden in production.
- ✅ Learn how to use Flyway for version-controlled database schema changes.
- ✅ Write your first SQL migration script.

---

## 💡 The Concept: Why Flyway?
In Lesson 7, we used `ddl-auto: update`. This lets Hibernate automatically create or alter tables. 
**Why is this bad?**
1. It can accidentally drop columns or data in production.
2. If you have 5 developers, they can't easily share schema changes.
3. You can't "rollback" to a previous version if a mistake is made.

**The Solution: Flyway.** 
Flyway is a database migration tool. You write plain SQL files with version numbers (e.g., `V1__create_products.sql`). When the app starts, Flyway checks which scripts have already been run, and only executes the new ones. It keeps a `flyway_schema_history` table in your database to track this.

---

## 🛠️ Step-by-Step Build

### Step 1: Add Flyway Dependency
Add this to your `pom.xml`:
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

### Step 2: Disable `ddl-auto`
Open `application.yml` and change the JPA settings:
```yaml
  jpa:
    hibernate:
      ddl-auto: validate # <-- CHANGE THIS! JPA will now ONLY check if the DB matches the Entity. It will NOT create tables.
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```
*Note: Because we changed this to `validate`, if the table doesn't exist, the app will crash on startup. This is exactly what we want! It forces us to use Flyway.*

### Step 3: Create the Migration Folder
In `src/main/resources/`, create this exact folder structure:
```text
src/main/resources/db/migration/
```

### Step 4: Write the First Migration Script
Inside the `migration` folder, create a file named exactly:
`V1__create_products_table.sql`

*(Naming rule: `V` + Version Number + `__` (two underscores) + Description + `.sql`)*

Paste this SQL into the file:
```sql
-- V1__create_products_table.sql

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DOUBLE PRECISION NOT NULL,
    description TEXT,
    stock INTEGER,
    created_at TIMESTAMP WITHOUT TIME ZONE,
    updated_at TIMESTAMP WITHOUT TIME ZOPNE
);
```
*(Note: `BIGSERIAL` is PostgreSQL's way of saying "auto-incrementing Long".)*

### Step 5: Fix the Typo and Restart
Wait, I put a typo in the SQL above (`TIME ZOPNE`). Let's see what happens!
1. Run your app: `./mvnw spring-boot:run`
2. **It will crash!** Look at the console. Flyway will say: `Validation failed: Migration checksum mismatch`. 
3. **Why?** Because Flyway calculates a "checksum" (a digital fingerprint) of the SQL file the first time it runs. If you edit the file later, the fingerprint changes, and Flyway blocks it to prevent accidental corruption.
4. **The Fix:** Fix the typo in the SQL file to `TIME ZONE`. 
5. Because we are in local development, we can reset the database. Open pgAdmin, right-click the `products` table and "Delete/Drop" it. Also delete the `flyway_schema_history` table.
6. Restart the app. It will now succeed! Flyway will create the table and record `V1` in the history table.

### Step 6: Add a New Column (The Right Way)
Let's say we want to add an `is_active` column. 
**NEVER edit `V1`.** Instead, create a new file: `V2__add_is_active_to_products.sql`.

```sql
-- V2__add_is_active_to_products.sql

ALTER TABLE products 
ADD COLUMN is_active BOOLEAN DEFAULT TRUE;
```
Restart the app. Flyway will see `V2` is new, run it, and update the history. Your data is safe!

---

## 🚀 Run and Test
1. Ensure `ddl-auto: validate` is set.
2. Ensure `V1__create_products_table.sql` exists with correct syntax.
3. Drop the old tables in pgAdmin (if they were created by `ddl-auto: update`).
4. Run the app. Check pgAdmin. You will see the `flyway_schema_history` table proving Flyway ran your script!

---

## 🚨 Common Errors
| Error | Cause | Fix |
|---|---|---|
| `Found non-empty schema(s) but no schema history table` | You have tables from `ddl-auto: update`, but Flyway is new. | Drop the tables manually in pgAdmin, or set `spring.flyway.baseline-on-migrate: true` in yml. |
| `Validation failed: Migration checksum mismatch` | You edited an already-applied SQL file. | Revert the file, or drop the DB and start fresh (for local dev only). |

---

## 🛠️ Exercise
1. Create `V3__add_category_to_products.sql`.
2. Write an `ALTER TABLE` statement to add a `category VARCHAR(100)` column.
3. Add `private String category;` to your `Product.java` Entity.
4. Restart the app and verify it starts successfully with `ddl-auto: validate`.

---

## 🧠 Quiz
1. Why is `ddl-auto: update` dangerous for production environments?
2. What is the exact naming convention for a Flyway migration file?
3. If you need to fix a mistake in `V1__create_products_table.sql` *after* it has already been applied, what is the correct professional approach?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers. 

Once you complete this batch (Lessons 6-9), you have successfully built a professional, Dockerized, database-backed Spring Boot API with proper migration tracking! Reply **"Next"** to get Lessons 10-13 (DTOs, MapStruct, Validation, and Exception Handling).