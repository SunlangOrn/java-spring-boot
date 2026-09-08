# 📘 Phase 2, Lesson 4: PostgreSQL, JPA & Flyway

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [Concepts Explained](#-concepts-explained)
- [The Chain](#-the-chain)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Start PostgreSQL using Docker
- ✅ Understand JPA, Hibernate, and Spring Data JPA
- ✅ Convert `Product` into a database Entity
- ✅ Use Flyway to create tables automatically

---

## 💡 Concepts Explained

### PostgreSQL
A **relational database**. Think of it as a super-powered Excel spreadsheet that runs on a server. Data is stored in **tables** (rows and columns).

### JPA (Java Persistence API)
A **set of rules** (specification) that defines how Java objects map to database tables. JPA itself does nothing — it's just a contract.

### Hibernate
The **engine** that implements JPA. When you call `repository.save(product)`, Hibernate translates it into SQL: `INSERT INTO products (name, price) VALUES ('Laptop', 999.99)`.

### Spring Data JPA
A Spring library that gives you ready-made methods (`findAll`, `findById`, `save`, `delete`) so you don't have to write SQL yourself.

### Flyway
A **migration tool**. Instead of manually creating tables, you write SQL files with version numbers. Flyway runs them automatically when the app starts.

---

## 🔗 The Chain

```text
Your Java Code
     ↓
Spring Data JPA (provides findAll, save, etc.)
     ↓
Hibernate (translates Java → SQL)
     ↓
JDBC (Java's database connector)
     ↓
PostgreSQL (the actual database)
```

---

## 🛠️ Step-by-Step Build

### Step 1: Start PostgreSQL with Docker
```bash
docker run --name product-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=product_db \
  -p 5432:5432 \
  -d postgres:16
```

Verify it's running:
```bash
docker ps
```

### Step 2: Add Dependencies to `pom.xml`
```xml
<!-- JPA + Hibernate -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- PostgreSQL Driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Flyway -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

### Step 3: Update `application.yml`
```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/product_db
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}

  jpa:
    hibernate:
      ddl-auto: validate   # Flyway creates tables, JPA only checks them
    show-sql: true          # Print SQL to console (for learning)
    properties:
      hibernate:
        format_sql: true    # Pretty-print SQL

  flyway:
    enabled: true
    locations: classpath:db/migration

app:
  welcome-message: "Welcome to the Product API"
```

**Important:** `ddl-auto: validate` means JPA will **never** create or modify tables. Flyway handles that. This is production-safe.

### Step 4: Create Flyway Migration
Create this folder: `src/main/resources/db/migration/`

Create this file:
```sql
-- src/main/resources/db/migration/V1__create_products_table.sql

CREATE TABLE products (
    id    BIGSERIAL PRIMARY KEY,
    name  VARCHAR(255) NOT NULL,
    price DOUBLE PRECISION NOT NULL
);
```

**What does this SQL mean?**
- `BIGSERIAL` = Auto-incrementing number (1, 2, 3...)
- `PRIMARY KEY` = Unique identifier for each row
- `VARCHAR(255)` = Text up to 255 characters
- `NOT NULL` = This column cannot be empty

**Flyway naming rules:**
- `V` = Version
- `1` = Number (sequential)
- `__` = Two underscores
- `description` = What this migration does

### Step 5: Convert Product to a JPA Entity
```java
// src/main/java/com/example/product/model/Product.java
package com.example.product.model;

import jakarta.persistence.*;
import lombok.*;

@Entity                          // "This class maps to a database table"
@Table(name = "products")        // "The table name is 'products'"
@Getter @Setter @Builder
@NoArgsConstructor               // REQUIRED by Hibernate!
@AllArgsConstructor
public class Product {

    @Id                          // "This is the Primary Key"
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Auto-increment
    private Long id;

    @Column(nullable = false)    // Matches NOT NULL in SQL
    private String name;

    @Column(nullable = false)
    private Double price;
}
```

**Why `@NoArgsConstructor` is required:**
When Hibernate reads a row from the database, it first calls `new Product()` (empty constructor) to create a blank object, then uses setters to fill in the values. Without it, Hibernate crashes.

### Step 6: Create the Repository
```java
// src/main/java/com/example/product/repository/ProductRepository.java
package com.example.product.repository;

import com.example.product.model.Product;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    // You get these methods FOR FREE:
    // save(), findById(), findAll(), deleteById(), count(), existsById()
}
```

**How it works:**
- `JpaRepository<Product, Long>` means: Entity = Product, Primary Key type = Long
- Spring generates the implementation at runtime. You write an interface, Spring does the rest!

### Step 7: Update the Service
Replace the `ArrayList` with the real repository:

```java
// src/main/java/com/example/product/service/ProductService.java
package com.example.product.service;

import com.example.product.model.Product;
import com.example.product.repository.ProductRepository;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }

    public Product getProductById(Long id) {
        return productRepository.findById(id).orElse(null);
    }

    public Product addProduct(String name, Double price) {
        Product newProduct = Product.builder()
                .name(name)
                .price(price)
                .build();
        // No need to set ID — PostgreSQL generates it automatically!
        return productRepository.save(newProduct);
    }
}
```

**How `save()` works internally:**
```text
productRepository.save(product)
        ↓
Does the entity have an ID?
        ↓
  NO  → INSERT INTO products (name, price) VALUES (?, ?)
  YES → UPDATE products SET name=?, price=? WHERE id=?
```

---

## 🚀 Run and Test

```bash
./mvnw clean spring-boot:run
```

**Watch the console!** You should see Flyway logs:
```text
Flyway: Migrating schema "public" to version "1 - create products table"
Flyway: Successfully applied 1 migration
```

**Test:**
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "Keyboard", "price": 79.99}'
```

**Restart the app and test again — the data is still there!** (Unlike the in-memory list from Lesson 2)

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| `Connection refused` | PostgreSQL not running | Run `docker start product-db` |
| `Table "products" does not exist` | Flyway didn't run | Check migration file location |
| `No default constructor` | Missing `@NoArgsConstructor` | Add it to the Entity |
| `Checksum mismatch` | You edited an applied migration | Never edit applied migrations! Create a new one |

---

## 🛠️ Exercise
1. Create migration `V2__add_stock_column.sql`:
   ```sql
   ALTER TABLE products ADD COLUMN stock INTEGER NOT NULL DEFAULT 0;
   ```
2. Add `stock` field to the `Product` entity.
3. Update the service and controller to handle `stock`.
4. Restart and verify Flyway applies V2 automatically.

---

## 🧠 Quiz
1. What is the difference between JPA and Hibernate?
2. How does `save()` decide between INSERT and UPDATE?
3. Why should you never edit a Flyway migration after it's been applied?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 5.