# 📘 Phase 3, Lesson 1: Connecting Two Tables (Category & Product)

---

## 🎯 Goal
Create a `Category` table and link it to the `Product` table so every product belongs to one category.

---

## 🧠 The Big Picture

Imagine a supermarket:
- The store has **sections** (Electronics, Books, Food).
- Every **product** sits in one section.
- One section has **many** products.
- One product belongs to **one** section.

In database language:
- "Section" = **Category**
- "One section has many products" = **One-to-Many relationship**

Instead of typing "Electronics" into every product row, we create a separate Category table and just store the **category ID** in each product. This saves space and prevents mistakes.

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **Entity** | A Java class that represents a database table |
| **Relationship** | A link between two tables (e.g., Product → Category) |
| **Foreign Key** | A column that stores the ID of another table's row |
| **`@ManyToOne`** | "Many of THIS can belong to ONE of THAT" |
| **`@OneToMany`** | "One of THIS can have MANY of THAT" |
| **`FetchType.LAZY`** | "Don't load the linked data until I specifically ask for it" |

---

## 🛠️ Step 1: Create the Flyway Migration

### What we're doing:
We need to create the `categories` table in the database and add a `category_id` column to the `products` table. We do this with a SQL file that Flyway will run automatically.

### The Code:
Create file: `src/main/resources/db/migration/V4__create_categories_table.sql`

```sql
-- Step A: Create the categories table
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP WITHOUT TIME ZONE,
    updated_at TIMESTAMP WITHOUT TIME ZONE
);

-- Step B: Add category_id column to products
ALTER TABLE products 
ADD COLUMN category_id BIGINT;

-- Step C: Create the link (Foreign Key)
ALTER TABLE products 
ADD CONSTRAINT fk_product_category 
FOREIGN KEY (category_id) REFERENCES categories(id);
```

### 📝 After the Code - What Just Happened?
- **Step A**: Creates a new table called `categories` with columns: `id`, `name`, `created_at`, `updated_at`. The `BIGSERIAL` means the ID will auto-increment (1, 2, 3...).
- **Step B**: Adds a new column `category_id` to the existing `products` table. This column will store the ID of the category each product belongs to.
- **Step C**: Creates a "Foreign Key constraint". This is a rule that says: "The `category_id` in products MUST match a real `id` in the categories table." This prevents you from linking a product to a category that doesn't exist.

### 💡 Note
> Flyway runs these SQL files **in order** by version number. That's why we named it `V4` (because V1, V2, V3 already exist from Phase 2). Never change the version number of a file that has already been run!

---

## 🛠️ Step 2: Create the Category Entity

### What we're doing:
Create a Java class that represents the `categories` table.

### The Code:
Create file: `src/main/java/com/example/demo/entity/Category.java`

```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "categories")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Category extends BaseEntity {

    private String name;
}
```

### 📝 After the Code - What Just Happened?
- `@Entity`: Tells Spring "This class is a database table."
- `@Table(name = "categories")`: Tells Spring "The table name in the database is 'categories'."
- `extends BaseEntity`: This is the magic part! Our `BaseEntity` (from Phase 2) already has `id`, `createdAt`, and `updatedAt`. So we don't need to write them again. We only add `name`.
- `@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor`: These are Lombok annotations that automatically create getters, setters, and constructors so we don't have to write them manually.

### 📦 Imports to Remember
```java
import com.example.demo.entity.base.BaseEntity;  // Your base entity
import jakarta.persistence.Entity;                // NOT javax.persistence (old version)
import jakarta.persistence.Table;
import lombok.*;                                  // All Lombok annotations
```

### 💡 Note
> Always use `jakarta.persistence.*` in Spring Boot 3+. If you see `javax.persistence.*`, that is the old version and will cause errors.

---

## 🛠️ Step 3: Update the Product Entity

### What we're doing:
Add a `category` field to the `Product` class so each product knows which category it belongs to.

### The Code:
Open file: `src/main/java/com/example/demo/entity/Product.java`

Add this new field at the bottom of the class:

```java
    // Link this product to a Category
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
```

### 📝 After the Code - What Just Happened?
- `@ManyToOne`: This tells JPA "Many products can belong to ONE category." Think of it like: "Many students belong to ONE classroom."
- `fetch = FetchType.LAZY`: This is a **performance setting**. It tells JPA: "When I load a product from the database, do NOT automatically load the category data. Only load it if I specifically call `product.getCategory()`." This keeps your app fast.
- `@JoinColumn(name = "category_id")`: This tells JPA "The link between Product and Category is stored in the `category_id` column in the products table." This must match the column name in your Flyway migration!

### 📦 Imports to Remember
```java
import jakarta.persistence.ManyToOne;
import jakarta.persistence.FetchType;
import jakarta.persistence.JoinColumn;
import com.example.demo.entity.Category;  // The new Category class
```

---

## 🛠️ Step 4: Create the Category Repository

### What we're doing:
Create a repository so we can save and find categories in the database.

### The Code:
Create file: `src/main/java/com/example/demo/repository/CategoryRepository.java`

```java
package com.example.demo.repository;

import com.example.demo.entity.Category;
import org.springframework.data.jpa.repository.JpaRepository;

public interface CategoryRepository extends JpaRepository<Category, Long> {
    // Spring automatically provides:
    // save(), findAll(), findById(), deleteById(), existsById(), count()
}
```

### 📝 After the Code - What Just Happened?
- `JpaRepository<Category, Long>`: The first type (`Category`) is the entity. The second type (`Long`) is the data type of the primary key (ID).
- We don't write any methods! Spring Data JPA automatically creates the SQL for basic operations like `save()` and `findAll()`.

### 📦 Imports to Remember
```java
import com.example.demo.entity.Category;
import org.springframework.data.jpa.repository.JpaRepository;
```

---

## 🔄 How It All Connects

```text
Product Table                    Category Table
┌────┬──────────┬───────┬───────────┐    ┌────┬─────────────┐
│ id │ name     │ price │category_id│    │ id │ name        │
├────┼──────────┼───────┼───────────┤    ├────┼─────────────┤
│ 1  │ Laptop   │ 999   │ 1         │───→│ 1  │ Electronics │
│ 2  │ Phone    │ 599   │ 1         │───→│ 1  │ Electronics │
│ 3  │ Novel    │ 15    │ 2         │───→│ 2  │ Books       │
└────┴──────────┴───────┴───────────┘    └────┴─────────────┘
```

The arrow shows the Foreign Key link. The `category_id` in Product points to the `id` in Category.

---

## 🧪 Run & Test

1. Restart your app: `./mvnw spring-boot:run`
2. Watch the console for Flyway logs: `Migrating schema to version 4`
3. Open pgAdmin (http://localhost:5050) and verify:
   - `categories` table exists
   - `products` table has a `category_id` column

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Table "categories" does not exist` | Flyway didn't run the migration | Check the file is in `db/migration/` and named correctly |
| `LazyInitializationException` | Trying to access `product.getCategory()` outside a `@Transactional` method | Keep database access inside `@Transactional` service methods |
| `Column "category_id" not found` | Entity column name doesn't match database | Ensure `@JoinColumn(name = "category_id")` matches the SQL exactly |

---

## ✏️ Exercise

1. Add a `description` field (String) to the `Category` entity.
2. Create Flyway migration `V5__add_description_to_categories.sql`:
   ```sql
   ALTER TABLE categories ADD COLUMN description TEXT;
   ```
3. Restart the app and verify the column appears in pgAdmin.

---

## 🧠 Quiz

1. What does `@ManyToOne` mean in plain English?
2. Why do we use `FetchType.LAZY` instead of `FetchType.EAGER`?
3. What is a Foreign Key and why is it important?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 2.