# 📘 Phase 2, Lesson 5: DTOs (Data Transfer Objects)

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem](#-the-problem)
- [The Restaurant Menu Analogy](#-the-restaurant-menu-analogy)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand why we never expose Entities directly to clients
- ✅ Create Request and Response DTOs
- ✅ Manually map between Entity and DTO

---

## 🚨 The Problem

Right now, your controller returns the `Product` **Entity** directly:
```java
@GetMapping("/{id}")
public ResponseEntity<Product> getProductById(@PathVariable Long id) {
    return ResponseEntity.ok(productService.getProductById(id));
}
```

**Why is this dangerous?**

### Problem 1: Security
If you later add a `password` field to a `User` entity, it gets exposed automatically in the API response!

### Problem 2: Database Changes Break the API
If you rename `price` to `unitPrice` in the database, the API response changes from `{"price": 29.99}` to `{"unitPrice": 29.99}`. Every client breaks!

### Problem 3: You Can't Customize
What if you want to format the price as `"$29.99"`? You can't, because the Entity stores a `Double`.

---

## 🍽️ The Restaurant Menu Analogy

- **Kitchen (Entity)**: Has raw ingredients (all database columns)
- **Menu (DTO)**: Shows only what the customer needs to see

The customer doesn't need to know how much salt the chef used. They just need to see "Grilled Chicken - $15.99".

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Response DTO
```java
// src/main/java/com/example/product/dto/response/ProductResponse.java
package com.example.product.dto.response;

import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductResponse {
    private Long id;
    private String name;
    private Double price;
    // Only the fields the client needs to see!
    // No database annotations here.
}
```

### Step 2: Create the Request DTO
```java
// src/main/java/com/example/product/dto/request/ProductCreateRequest.java
package com.example.product.dto.request;

import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductCreateRequest {
    private String name;
    private Double price;
    // No ID field — the database generates it!
}
```

### Step 3: Manual Mapping in the Service
```java
// In ProductService.java
public ProductResponse createProduct(ProductCreateRequest request) {
    // 1. Convert Request DTO → Entity
    Product product = Product.builder()
            .name(request.getName())
            .price(request.getPrice())
            .build();

    // 2. Save to database
    Product saved = productRepository.save(product);

    // 3. Convert Entity → Response DTO
    return ProductResponse.builder()
            .id(saved.getId())
            .name(saved.getName())
            .price(saved.getPrice())
            .build();
}

public ProductResponse getProductById(Long id) {
    Product product = productRepository.findById(id).orElse(null);
    if (product == null) return null;

    return ProductResponse.builder()
            .id(product.getId())
            .name(product.getName())
            .price(product.getPrice())
            .build();
}
```

### Step 4: Update the Controller
```java
@PostMapping
public ResponseEntity<ProductResponse> createProduct(
        @RequestBody ProductCreateRequest request) {  // Accept DTO, not Entity!
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(productService.createProduct(request));
}

@GetMapping("/{id}")
public ResponseEntity<ProductResponse> getProductById(@PathVariable Long id) {
    return ResponseEntity.ok(productService.getProductById(id));
}
```

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "Mouse", "price": 29.99}'
```

The response looks the same, but now it's a **DTO**, not an Entity!

---

## 🛠️ Exercise
1. Add a `description` field to the `Product` entity (create a Flyway migration for it).
2. Do **NOT** add `description` to `ProductResponse`.
3. Create a product with a description.
4. Get the product and verify `description` is NOT in the response.
5. This proves the DTO controls what data is exposed!

---

## 🧠 Quiz
1. What does DTO stand for?
2. Give 2 reasons why we shouldn't expose Entities directly.
3. In the restaurant analogy, what represents the DTO?

---

## 🛑 STOP
Reply with your exercise results and quiz answers before moving to Lesson 6.