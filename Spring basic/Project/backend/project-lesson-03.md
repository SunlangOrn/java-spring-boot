# 🛒 ShopCore Project - Lesson 3: Catalog API (Products, Categories, Redis Cache)

---

## 🎯 Goal
- ✅ Build the Category and Product CRUD APIs.
- ✅ Add Pagination and Sorting.
- ✅ Add Redis Caching to make product reads lightning fast.

---

## 🧠 The Big Picture
The "Catalog" is the storefront. Customers browse it constantly. 
If 1,000 customers load the homepage at once, querying the database 1,000 times will crash it. 
Instead, we load the products **once**, save them in **Redis** (RAM), and serve the next 999 customers instantly from memory.

---

## 🛠️ Step 1: Update Product & Category Entities

*(Assuming you have `Category` and `Product` entities from Phase 3. Ensure they look like this)*

**Category.java:**
```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "categories")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Category extends BaseEntity {
    @Column(nullable = false, unique = true)
    private String name;
}
```

**Product.java:**
```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "products")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Product extends BaseEntity {
    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Double price;

    private String description;

    @Column(nullable = false)
    private Integer stock;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
}
```

---

## 🛠️ Step 2: Add Caching Annotations to ProductService

### What we're doing:
Tell Spring to cache the results of `getAllProducts()` and clear the cache when a product is updated or deleted.

### The Code:
**Update:** `src/main/java/com/example/demo/service/impl/ProductServiceImpl.java`

```java
package com.example.demo.service.impl;

import com.example.demo.common.Paging;
import com.example.demo.dto.request.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import com.example.demo.entity.Category;
import com.example.demo.entity.Product;
import com.example.demo.exception.ResourceNotFoundException;
import com.example.demo.mapper.ProductMapper;
import com.example.demo.repository.CategoryRepository;
import com.example.demo.repository.ProductRepository;
import com.example.demo.service.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;
    private final CategoryRepository categoryRepository;

    @Override
    @Cacheable(value = "products", key = "'all'") // Cache the result!
    @Transactional(readOnly = true)
    public List<ProductResponse> getAllProducts() {
        System.out.println(">>> FETCHING FROM DATABASE (SLOW) <<<");
        return productRepository.findAll().stream()
                .map(productMapper::toResponse)
                .toList();
    }

    @Override
    @CacheEvict(value = "products", allEntries = true) // Clear cache when data changes!
    @Transactional
    public ProductResponse createProduct(ProductRequest request) {
        Category category = categoryRepository.findById(request.categoryId())
                .orElseThrow(() -> new ResourceNotFoundException("Category not found"));
        
        Product product = productMapper.toEntity(request);
        product.setCategory(category);
        
        return productMapper.toResponse(productRepository.save(product));
    }

    // ... add update and delete methods with @CacheEvict as well
}
```

### 📝 After the Code - What Just Happened?
- `@Cacheable`: The first time this runs, it hits the database and saves the result in Redis. The second time, it skips the database entirely and returns the Redis data.
- `@CacheEvict`: When we create/update/delete a product, we wipe the cache so the next `getAllProducts()` call gets fresh data from the database.

### 💡 Note
> Make sure `@EnableCaching` is on your `DemoApplication.java` class, and Redis is running in your `docker-compose.yml`!

---

## 🛠️ Step 3: Add Pagination to ProductRepository

### The Code:
**Update:** `src/main/java/com/example/demo/repository/ProductRepository.java`

```java
package com.example.demo.repository;

import com.example.demo.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

public interface ProductRepository extends JpaRepository<Product, Long> {
    
    // JOIN FETCH prevents the N+1 query problem!
    @Query("SELECT p FROM Product p JOIN FETCH p.category")
    Page<Product> findAllPaged(Pageable pageable);
}
```

### 📝 After the Code - What Just Happened?
- `JOIN FETCH` tells Hibernate to get the Product AND its Category in a single SQL query. This is critical for performance when paginating.

---

## 🧪 Run & Test

**Test 1: Create a Category**
```bash
curl -X POST http://localhost:8081/api/v1/categories \
-H "Content-Type: application/json" \
-d '{"name": "Electronics"}'
```

**Test 2: Create a Product (Requires Admin JWT)**
```bash
curl -X POST http://localhost:8081/api/v1/products \
-H "Authorization: Bearer <ADMIN_TOKEN>" \
-H "Content-Type: application/json" \
 '{"name": "Laptop", "price": 999.99, "stock": 10, "categoryId": 1}'
```

**Test 3: Get All Products (Watch the console!)**
```bash
curl http://localhost:8081/api/v1/products
```
*Run it twice. The first time you will see `>>> FETCHING FROM DATABASE (SLOW) <<<`. The second time, you won't. It came from Redis!*

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| Cache never updates | Forgot `@CacheEvict` on create/update/delete methods. | Add `@CacheEvict(value = "products", allEntries = true)`. |
| `LazyInitializationException` | Forgot `JOIN FETCH` in the repository query. | Use the `@Query` with `JOIN FETCH` as shown above. |

---

## ✏️ Exercise
1. Add a `@CacheEvict` annotation to your `updateProduct` and `deleteProduct` methods.
2. Test updating a product and verify the cache is cleared.

---

## 🧠 Quiz
1. What is the difference between `@Cacheable` and `@CacheEvict`?
2. Why do we use `JOIN FETCH` when paginating products?
3. What happens if Redis is turned off but `@Cacheable` is still in the code?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Project Lesson 4!