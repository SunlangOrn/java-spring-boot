# 📘 Phase 2, Lesson 7: JPA Entities & The `BaseEntity` Pattern

## 🎯 Learning Goal
- ✅ Connect Spring Boot to the PostgreSQL database.
- ✅ Learn the Enterprise `BaseEntity` pattern to avoid repeating code.
- ✅ Create your first JPA `@Entity`.

---

## 💡 The Concept: JPA and Entities
**JPA (Java Persistence API)** is the bridge between your Java code and the SQL database. 
An **Entity** is a Java class that represents a table in the database. Every row in the table is an object of this class.

In professional apps, *every* table needs an `id`, `createdAt`, and `updatedAt`. Instead of typing these 3 fields into every single Entity, we create a **`BaseEntity`** and let other classes inherit from it.

---

## 🛠️ Step-by-Step Build

### Step 1: Add Dependencies
Ensure these are in your `pom.xml` (Spring Initializr usually adds them if you selected "Spring Data JPA" and "PostgreSQL Driver"):
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Step 2: Configure `application.yml`
Update your `src/main/resources/application.yml` to tell Spring how to connect to Docker:

```yaml
server:
  port: 8081

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/product_db
    username: admin
    password: admin123
  jpa:
    hibernate:
      ddl-auto: update # WARNING: Only for learning! We will fix this in Lesson 9.
    show-sql: true     # Prints the SQL queries to the console (great for learning)
    properties:
      hibernate:
        format_sql: true # Makes the printed SQL easy to read

app:
  welcome-message: "Hello from the configuration file!"
  version: "1.0.0"
```

### Step 3: Enable JPA Auditing
Open `DemoApplication.java` and add `@EnableJpaAuditing`. This tells Spring to automatically fill in dates for us.

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@SpringBootApplication
@EnableJpaAuditing // <-- ADD THIS LINE
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### Step 4: Create the `BaseEntity`
Create a new package called `entity.base`. Inside, create `BaseEntity.java`:

```java
// src/main/java/com/example/demo/entity/base/BaseEntity.java
package com.example.demo.entity.base;

import jakarta.persistence.*;
import lombok.Getter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import java.time.LocalDateTime;

@Getter // We only need getters. IDs and Dates should never be changed manually!
@MappedSuperclass // Tells JPA: "Copy my fields to any class that extends me"
@EntityListeners(AuditingEntityListener.class) // Turns on the auto-date magic
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    @Column(updatable = false) // Cannot be changed after creation
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

### Step 5: Update the `Product` Model to be an Entity
Delete the old `Product.java` record in the `model` package. 
Create a new package called `entity`. Inside, create `Product.java`:

```java
// src/main/java/com/example/demo/entity/Product.java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity // Tells JPA: "This class is a database table"
@Table(name = "products") // Explicitly names the table
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Product extends BaseEntity { // <-- INHERITS id, createdAt, updatedAt!

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Double price;
    
    private String description;
}
```

---

## 🚀 Run and Test
1. Make sure Docker is running (`docker-compose up -d`).
2. Run your Spring Boot application.
3. Look at the console. You will see Hibernate automatically generate and run a `CREATE TABLE products` SQL statement!
4. Go to **pgAdmin** (http://localhost:5050), refresh, and look inside `product_db` -> Schemas -> public -> Tables. You will see the `products` table with `id`, `created_at`, `updated_at`, `name`, `price`, and `description` columns!

---

## 🚨 Common Errors
| Error | Cause | Fix |
|---|---|---|
| `Connection refused` | Docker database isn't running. | Run `docker-compose up -d`. |
| `createdAt` is null in DB | Forgot `@EnableJpaAuditing`. | Add it to `DemoApplication.java`. |
| `Cannot resolve symbol 'Entity'` | Missing JPA dependency or wrong import. | Ensure you import `jakarta.persistence.Entity`, NOT `javax`. |

---

## 🛠️ Exercise
1. Add a new field `private Integer stock;` to the `Product` entity.
2. Restart the app.
3. Check pgAdmin to verify the `stock` column was automatically added to the table.

---

## 🧠 Quiz
1. What does `@MappedSuperclass` do? Does it create a table in the database?
2. Why do we use `@Getter` but not `@Setter` on the `BaseEntity` class?
3. What does `ddl-auto: update` do, and why is it dangerous for real production apps?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Lesson 8.