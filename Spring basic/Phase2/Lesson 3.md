# 📘 Phase 2, Lesson 3: Lombok (Making Java Cleaner)

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem](#-the-problem)
- [What is Lombok?](#-what-is-lombok)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand what Lombok is and how it works
- ✅ Convert a `record` to a `class` using Lombok
- ✅ Prepare the model for JPA (which needs a regular class)

---

## 😩 The Problem

In Lesson 2, we used a Java `record`:
```java
public record Product(Long id, String name, Double price) {}
```

Records are great for simple data. **But** JPA (the database tool we'll add next) **requires** a regular class with:
- A no-argument constructor
- Getters and setters

Without Lombok, you'd have to write this:
```java
// ❌ So much boilerplate!
public class Product {
    private Long id;
    private String name;
    private Double price;

    public Product() {} // No-arg constructor

    public Product(Long id, String name, Double price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public Double getPrice() { return price; }
    public void setPrice(Double price) { this.price = price; }

    @Override
    public String toString() { return "Product{id=" + id + "}"; }

    @Override
    public boolean equals(Object o) { /* 10 more lines */ }

    @Override
    public int hashCode() { /* 5 more lines */ }
}
```

That's **30+ lines** for a simple class with 3 fields!

---

## ✨ What is Lombok?

Lombok is a library that **generates all that boilerplate code automatically** at compile time.

You write **5 lines**:
```java
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Product {
    private Long id;
    private String name;
    private Double price;
}
```

Lombok generates the other **25+ lines** for you.

### How It Works Internally
1. You write `@Getter` on your class
2. When you run `mvn compile`, Lombok hooks into the Java compiler
3. It modifies the code **in memory** before the `.class` file is created
4. The final `.class` file has all the getters already inside
5. **Zero runtime performance cost** — it's not reflection!

---

## 🛠️ Step-by-Step Build

### Step 1: Add Lombok to `pom.xml`
```xml
<dependencies>
    <!-- ... your other dependencies ... -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

**If using IntelliJ IDEA:**
1. Go to Settings → Plugins → Search "Lombok" → Install
2. Go to Settings → Build → Compiler → Annotation Processors → ✅ Enable

### Step 2: Convert Product to a Lombok Class
```java
// src/main/java/com/example/product/model/Product.java
package com.example.product.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter          // Generates: getId(), getName(), getPrice()
@Setter          // Generates: setId(), setName(), setPrice()
@Builder         // Generates: Product.builder().id(1L).name("Laptop").build()
@NoArgsConstructor  // Generates: public Product() {} ← REQUIRED for JPA!
@AllArgsConstructor // Generates: public Product(Long id, String name, Double price) {}
public class Product {
    private Long id;
    private String name;
    private Double price;
}
```

### Step 3: Update the Service to Use Builder
Since `Product` is no longer a `record`, we create objects differently:

```java
// In ProductService.java
public Product addProduct(String name, Double price) {
    // Using Lombok's @Builder
    Product newProduct = Product.builder()
            .id(idGenerator.getAndIncrement())
            .name(name)
            .price(price)
            .build();
    products.add(newProduct);
    return newProduct;
}
```

**Why Builder is better than constructors:**
```java
// ❌ Constructor: Hard to read, easy to mix up order
new Product(1L, "Laptop", 999.99);

// ✅ Builder: Clear and readable
Product.builder()
    .id(1L)
    .name("Laptop")
    .price(999.99)
    .build();
```

---

## 🚀 Run and Test

```bash
./mvnw clean compile
./mvnw spring-boot:run
```

Test the same curl commands from Lesson 2. Everything should work exactly the same!

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| `cannot find symbol: method builder()` | Lombok didn't run | Run `mvn clean compile` |
| `cannot find symbol: method getName()` | IDE doesn't recognize Lombok | Install Lombok plugin in IDE |
| `NoClassDefFoundError` at runtime | Lombok in final JAR | Add `<optional>true</optional>` in pom.xml |

---

## 🛠️ Exercise
1. Add a `stock` field (Integer) to the `Product` class.
2. Update the `addProduct` method to accept and set `stock`.
3. Update the Controller POST endpoint to read `stock` from the JSON.
4. Test: `curl -X POST ... -d '{"name":"Mouse","price":29.99,"stock":50}'`

---

## 🧠 Quiz
1. Does Lombok slow down your app at runtime? Why or why not?
2. Why is `@NoArgsConstructor` required for JPA Entities?
3. What is the difference between `@Data` and `@Getter` + `@Setter`? *(Hint: Think about `equals/hashCode` and JPA relationships)*

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 4.