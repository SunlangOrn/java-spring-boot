# 📘 Phase 3, Lesson 4: Transactions & Concurrency (Locking)

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem: The "Lost Update"](#-the-problem-the-lost-update)
- [The Solution: Optimistic Locking](#-the-solution-optimistic-locking)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand the "Lost Update" concurrency problem.
- ✅ Implement Optimistic Locking using `@Version`.
- ✅ Handle `ObjectOptimisticLockingFailureException` gracefully.

---

## 🚨 The Problem: The "Lost Update"

Imagine two users (Alice and Bob) are trying to buy the last "Gaming Laptop" (Stock = 1) at the exact same millisecond.

1. **Alice** clicks "Buy". App reads stock: `1`.
2. **Bob** clicks "Buy". App reads stock: `1`.
3. **Alice's** app calculates: `1 - 1 = 0`. App saves stock: `0`.
4. **Bob's** app calculates: `1 - 1 = 0`. App saves stock: `0`.

**Result:** You just sold 2 laptops, but you only had 1 in the database! The stock is 0, but you oversold. This is the **Lost Update** problem.

---

## 💡 The Solution: Optimistic Locking

**Optimistic Locking** assumes conflicts are rare. It doesn't lock the database row. Instead, it adds a "version number" to the row.

1. Alice reads Product (Stock=1, **Version=1**).
2. Bob reads Product (Stock=1, **Version=1**).
3. Alice saves: "Update stock to 0, **but only if Version is still 1**." (Succeeds. DB Version becomes 2).
4. Bob saves: "Update stock to 0, **but only if Version is still 1**." **FAILS!** Because the version is now 2. Bob's transaction is rejected, and you tell him "Sorry, item just sold out."

---

## 🛠️ Step-by-Step Build

### Step 1: Add `@Version` to the Entity
```java
// src/main/java/com/example/product/entity/Product.java
package com.example.product.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "products")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private Double price;
    private Integer stock;

    // THE MAGIC ANNOTATION
    @Version
    private Integer version; 
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
}
```

### Step 2: Add the Column via Flyway
Create `V6__add_version_to_products.sql`:
```sql
ALTER TABLE products ADD COLUMN version INTEGER NOT NULL DEFAULT 0;
```

### Step 3: Create the Buy/Decrease Stock Method
```java
// In ProductService.java
@Transactional
public void decreaseStock(Long productId, int quantity) {
    Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

    if (product.getStock() < quantity) {
        throw new RuntimeException("Not enough stock!");
    }

    product.setStock(product.getStock() - quantity);
    // We don't need to manually increment the version. 
    // Hibernate does it automatically when we call save()!
    productRepository.save(product); 
}
```

### Step 4: Handle the Lock Exception Globally
When Bob's transaction fails, Hibernate throws an `ObjectOptimisticLockingFailureException`. Let's catch it.

```java
// In GlobalExceptionHandler.java
import org.springframework.orm.ObjectOptimisticLockingFailureException;

@ExceptionHandler(ObjectOptimisticLockingFailureException.class)
public ResponseEntity<Map<String, Object>> handleLockException(ObjectOptimisticLockingFailureException ex) {
    Map<String, Object> body = new HashMap<>();
    body.put("timestamp", LocalDateTime.now());
    body.put("status", HttpStatus.CONFLICT.value()); // 409 Conflict
    body.put("error", "Data Conflict");
    body.put("message", "This product was updated by another user. Please refresh and try again.");
    
    return new ResponseEntity<>(body, HttpStatus.CONFLICT);
}
```

---

## 🚀 Run and Test

To truly test this, you need to simulate two concurrent requests. You can use a tool like **JMeter** or **Apache Bench**, or simply run this bash script in two terminals at the exact same time:

**Terminal 1 & 2 (Run simultaneously):**
```bash
curl -X POST http://localhost:8080/api/products/1/buy \
-H "Content-Type: application/json" \
-d '{"quantity": 1}'
```

**Expected Result:**
- One terminal gets `200 OK`.
- The other terminal gets `409 Conflict` with the message: *"This product was updated by another user..."*

---

## 🚨 Common Errors
1. **`version` column is null**: You added `@Version` but didn't run the Flyway migration to add the column with a default value. *Fix:* Ensure `DEFAULT 0` in your SQL.
2. **Locking doesn't work**: You forgot `@Transactional` on the service method. Optimistic locking requires a transaction to check the version at commit time.

---

## 🛠️ Exercise
1. Create an endpoint `PUT /api/products/{id}/price` that updates the price.
2. Add `@Version` logic to it.
3. Simulate two users updating the price at the same time and verify one gets a 409 Conflict.

---

## 🧠 Quiz
1. What is the "Lost Update" problem?
2. How does `@Version` prevent it without locking the database row?
3. What HTTP status code is most appropriate for a concurrency conflict?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 5.