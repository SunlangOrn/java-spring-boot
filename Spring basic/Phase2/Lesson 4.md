# 📘 Phase 2, Lesson 4: PostgreSQL, JPA & Flyway

## 🎯 Learning Goals
- Connect to a real PostgreSQL database using Docker.
- Understand JPA, Hibernate, and Spring Data JPA.
- Use Flyway to manage database schema migrations.

## 💡 Concepts Explained Simply
- **PostgreSQL**: The actual database where data lives.
- **JPA**: The rules/interfaces for mapping Java objects to database tables.
- **Hibernate**: The engine that actually translates Java code into SQL.
- **Flyway**: A tool that runs SQL scripts automatically when the app starts. Never create tables manually!

## ⚙️ Step-by-Step Build

### Step 1: Start PostgreSQL with Docker
```bash
docker run --name product-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=product_db -p 5432:5432 -d postgres:16
```

### Step 2: Add Dependencies to `pom.xml`
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
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/product_db
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: validate # Flyway handles tables, JPA only validates
    show-sql: true
  flyway:
    enabled: true
    locations: classpath:db/migration
```

### Step 4: Create Flyway Migration
Create `src/main/resources/db/migration/V1__create_products_table.sql`:
```sql
CREATE TABLE products (
    id    BIGSERIAL PRIMARY KEY,
    name  VARCHAR(255) NOT NULL,
    price DOUBLE PRECISION NOT NULL
);
```

### Step 5: Convert Product to JPA Entity
```java
// src/main/java/com/example/product/model/Product.java
package com.example.product.model;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "products")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Double price;
}
```

### Step 6: Create Repository & Update Service
```java
// src/main/java/com/example/product/repository/ProductRepository.java
package com.example.product.repository;
import com.example.product.model.Product;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {}
```
*Update `ProductService` to inject `ProductRepository` and replace the `ArrayList` with `productRepository.findAll()` and `productRepository.save()`.*

## 🚀 Run and Test
Restart the app. Data will now persist even if you restart the server!

## 🧠 Quiz
1. What is the difference between JPA and Hibernate?
2. Why do we set `ddl-auto: validate` instead of `update`?
3. What happens if you edit a Flyway migration file after it has been applied?

---