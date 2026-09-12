# 📘 Phase 2, Lesson 8: Repositories & Saving Real Data

## 🎯 Learning Goal
- ✅ Understand what a Spring Data JPA Repository is.
- ✅ Replace the in-memory `ArrayList` with real database operations.
- ✅ Learn the magic of `save()`, `findAll()`, and `findById()`.

---

## 💡 The Concept: The Repository
In Lesson 4, we used an `ArrayList` to store products. When the app restarted, the data vanished. 

A **Repository** is a Spring interface that talks directly to the database. The magic of Spring Data JPA is that **you don't write any SQL**. You just create an interface that extends `JpaRepository`, and Spring writes the SQL for you automatically at runtime!

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Repository
Create a new package called `repository`. Inside, create `ProductRepository.java`:

```java
// src/main/java/com/example/demo/repository/ProductRepository.java
package com.example.demo.repository;

import com.example.demo.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository // Optional, but good practice to mark it as a Spring Bean
public interface ProductRepository extends JpaRepository<Product, Long> {
    // <Product, Long> means: 
    // 1. We are managing the 'Product' entity.
    // 2. The Primary Key (ID) of this entity is a 'Long'.
    
    // Spring automatically provides these methods for free:
    // - save(Product)
    // - findAll()
    // - findById(Long)
    // - deleteById(Long)
    // - count()
}
```

### Step 2: Update the Service to use the Repository
Now, let's replace the `ArrayList` in `ProductService` with our new `ProductRepository`.

```java
// src/main/java/com/example/demo/service/ProductService.java
package com.example.demo.service;

import com.example.demo.dto.ProductRequest;
import com.example.demo.entity.Product;
import com.example.demo.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor // Lombok generates the constructor for us!
public class ProductService {

    private final ProductRepository productRepository; // Inject the repository

    // READ: Get all products from the database
    @Transactional(readOnly = true)
    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }

    // READ: Get one product by ID
    @Transactional(readOnly = true)
    public Product getProductById(Long id) {
        // orElseThrow is a modern Java way to handle "not found"
        return productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found with id: " + id));
    }

    // CREATE: Save a new product to the database
    @Transactional
    public Product addProduct(ProductRequest request) {
        // 1. Map the Request DTO to the Entity
        Product newProduct = Product.builder()
                .name(request.name())
                .price(request.price())
                .description(request.description()) // From Lesson 7 exercise
                .build();
        
        // 2. Save to database. Hibernate will automatically generate the ID!
        return productRepository.save(newProduct);
    }
}
```

**Line-by-line:**
- `@RequiredArgsConstructor`: Because `productRepository` is `final`, Lombok creates the constructor `public ProductService(ProductRepository productRepository)` for us. Clean!
- `@Transactional`: Tells Spring to wrap this method in a database transaction. If anything fails, it rolls back. `readOnly = true` is an optimization for methods that only read data.
- `productRepository.save()`: If the `id` is null, it runs an `INSERT`. If the `id` exists, it runs an `UPDATE`.

### Step 3: Update the Controller (Minor Fix)
Our Controller from Lesson 5 is already perfect! It takes the `ProductRequest`, passes it to the Service, and returns the `Product`. Because we changed the Service to return a real `Product` entity, the Controller will automatically convert it to JSON.

*(Note: In Lesson 10, we will learn why returning the Entity directly is bad, and we will introduce Response DTOs and MapStruct. For now, this is fine to see the database working!)*

---

## 🚀 Run and Test

Restart your app. Watch the console for SQL logs!

**Test 1: Add a product**
```bash
curl -X POST http://localhost:8081/api/products \
-H "Content-Type: application/json" \
-d '{
  "name": "Mechanical Keyboard",
  "price": 150.00,
  "description": "RGB backlit"
}'
```
*Expected:* Returns the saved product, including the newly generated `id`, `createdAt`, and `updatedAt`.

**Test 2: Get all products**
```bash
curl http://localhost:8081/api/products
```
*Expected:* A JSON array containing your product.

**Test 3: Restart the app and Test 2 again**
The data is **still there**! It is safely stored in PostgreSQL.

---

## 🚨 Common Errors
| Error | Cause | Fix |
|---|---|---|
| `Product not found with id: X` | You requested an ID that doesn't exist. | This is expected behavior from our `orElseThrow` logic. |
| `could not prepare statement` | Column name mismatch between Entity and DB. | Check `@Column` names and ensure `ddl-auto: update` ran. |

---

## 🛠️ Exercise
1. Add a `deleteProduct(Long id)` method to `ProductService` that calls `productRepository.deleteById(id)`.
2. Add a `@DeleteMapping("/{id}")` to `ProductController` that calls this new service method.
3. Test it with `curl -X DELETE http://localhost:8081/api/products/1`
4. Verify the product is gone by calling GET all.

---

## 🧠 Quiz
1. Why do we extend `JpaRepository<Product, Long>`? What do those two types mean?
2. What is the difference between `@Transactional` and `@Transactional(readOnly = true)`?
3. How does `productRepository.save()` know whether to `INSERT` or `UPDATE`?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Lesson 9.