# 📘 Phase 2, Lesson 15: Advanced Pagination & Sorting

## 🎯 Learning Goal
- ✅ Understand why returning `List<Product>` is dangerous for large datasets.
- ✅ Use Spring Data JPA's `Pageable` interface.
- ✅ Create a `Paging<T>` wrapper to return metadata (total pages, current page, etc.).

---

## 💡 The Concept: Pagination
If your database has 1,000,000 products, `productRepository.findAll()` will load all 1,000,000 into memory and crash your app. 
Instead, we ask for "Page 0, Size 20". Spring Data JPA translates this into efficient SQL: `SELECT ... LIMIT 20 OFFSET 0`.

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Paging Wrapper
Add `Paging.java` to the `common` package:

```java
// src/main/java/com/example/demo/common/Paging.java
package com.example.demo.common;

import lombok.Builder;
import lombok.Getter;
import java.util.List;

@Getter
@Builder
public class Paging<T> {
    private List<T> items;
    private int page;
    private int size;
    private long totalElements;
    private int totalPages;
}
```

### Step 2: Update the Repository
Spring Data JPA makes this incredibly easy. Just change the return type to `Page<Product>`:

```java
// src/main/java/com/example/demo/repository/ProductRepository.java
package com.example.demo.repository;

import com.example.demo.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findAll(Pageable pageable); // Spring provides this automatically!
}
```

### Step 3: Update the Service
Update `ProductService` interface and `ProductServiceImpl`:

```java
// In ProductService.java (Interface)
import org.springframework.data.domain.Pageable;
import com.example.demo.common.Paging;

Paging<ProductResponse> getAllProductsPaged(Pageable pageable);
```

```java
// In ProductServiceImpl.java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

@Override
@Transactional(readOnly = true)
public Paging<ProductResponse> getAllProductsPaged(Pageable pageable) {
    // 1. Fetch a Page of Entities from the DB
    Page<Product> productPage = productRepository.findAll(pageable);
    
    // 2. Map Entities to DTOs
    List<ProductResponse> items = productPage.getContent().stream()
            .map(productMapper::toResponse)
            .toList();
            
    // 3. Build and return the Paging wrapper
    return Paging.<ProductResponse>builder()
            .items(items)
            .page(productPage.getNumber())
            .size(productPage.getSize())
            .totalElements(productPage.getTotalElements())
            .totalPages(productPage.getTotalPages())
            .build();
}
```

### Step 4: Update the Controller
Spring automatically converts URL parameters (`?page=0&size=5&sort=name,asc`) into a `Pageable` object!

```java
// In ProductController.java
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import com.example.demo.common.Paging;

@GetMapping("/paged")
public ResponseEntity<HttpBodyResponse<Paging<ProductResponse>>> getAllProductsPaged(
        @PageableDefault(size = 10, sort = "id") Pageable pageable) {
    
    Paging<ProductResponse> paging = productService.getAllProductsPaged(pageable);
    return responseSucceed(paging);
}
```

---

## 🚀 Run and Test

**Test: Get Page 0, Size 2, Sorted by Price Descending**
```bash
curl "http://localhost:8081/api/v1/products/paged?page=0&size=2&sort=price,desc"
```
*(Note the quotes around the URL because of the `&` symbol in bash!)*

**Expected:** A JSON response containing the `items` array, plus `page`, `size`, `totalElements`, and `totalPages`.

---

## 🛠️ Exercise
1. Add a new endpoint `GET /api/v1/products/category/{category}`.
2. Add a method to `ProductRepository`: `Page<Product> findByCategory(String category, Pageable pageable);`
3. Wire it up through the Service and Controller, returning the `Paging` wrapper.

---

## 🧠 Quiz
1. Why is returning a `List` dangerous for large datasets?
2. What does the `@PageableDefault` annotation do?
3. How does Spring know how to convert `?page=0&size=5` into a Java `Pageable` object?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Lesson 16.