# 📘 Phase 2, Lesson 3: Lombok & Preparing for the Database

## 🎯 Learning Goals
- Understand what Lombok is and how it works internally.
- Convert our `Product` from a `record` to a standard `class` using Lombok.
- Prepare our model for JPA (which requires standard classes).

## 💡 Concepts Explained Simply
**Lombok** uses annotations to generate boilerplate code (getters, setters, constructors) at **compile time**. It doesn't slow down your app at runtime because it modifies the code *before* it becomes a `.class` file.
*Why now?* JPA (Hibernate) requires standard classes with a no-argument constructor to create objects from database rows.

## ⚙️ Step-by-Step Build

### Step 1: Add Lombok to `pom.xml`
```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

### Step 2: Refactor the `Product` Model
```java
// src/main/java/com/example/product/model/Product.java
package com.example.product.model;

import lombok.*;

@Getter @Setter @Builder
@NoArgsConstructor @AllArgsConstructor
public class Product {
    private Long id;
    private String name;
    private Double price;
}
```

### Step 3: Update Service to use Builder
```java
// In ProductService.java
public Product addProduct(String name, Double price) {
    Product newProduct = Product.builder()
            .id(idGenerator.getAndIncrement())
            .name(name)
            .price(price)
            .build();
    products.add(newProduct);
    return newProduct;
}
```

## 🚀 Run and Test
Run `./mvnw clean compile` then `./mvnw spring-boot:run`. Test your POST and GET endpoints again.

## 🧠 Quiz
1. Does Lombok slow down your application at runtime? Why?
2. Why is `@NoArgsConstructor` strictly required for JPA Entities?
3. What is the difference between `@Data` and `@Getter` + `@Setter`?

---