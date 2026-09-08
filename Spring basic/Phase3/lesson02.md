# 📘 Phase 3, Lesson 2: The N+1 Problem, Pagination & Sorting

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The N+1 Problem Explained](#-the-n1-problem-explained)
- [The Solution: JOIN FETCH](#-the-solution-join-fetch)
- [Pagination & Sorting](#-pagination--sorting)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand the deadly **N+1 query problem** in JPA.
- ✅ Fix it using `JOIN FETCH`.
- ✅ Implement Pagination and Sorting for large datasets.

---

## 🚨 The N+1 Problem Explained

Imagine you are a teacher picking up 10 students (Products) and their parents (Categories).

**The Bad Way (N+1 Problem):**
1. You drive to the school and get all 10 students. (**1 query** for Products)
2. You ask Student 1, "Who is your parent?" You drive to their house to meet them. (**1 query** for Category)
3. You ask Student 2... drive to their house. (**1 query**)
4. ... You do this 10 times.
**Total trips: 11 queries.** If you have 1,000 products, you make **1,001 queries**! Your API will be incredibly slow.

**Why does this happen in Spring?**
Because we used `fetch = FetchType.LAZY` on the `@ManyToOne` relationship. When you loop through products and call `product.getCategory().getName()`, Hibernate secretly runs a `SELECT` for *each* category.

---

## 💡 The Solution: JOIN FETCH

**The Good Way (JOIN FETCH):**
1. You drive to the school. (**1 query** for Products)
2. You tell all 10 students: "Get your parents and get on the bus right now!" (**1 query** that gets Products AND Categories together).
**Total trips: 2 queries.** (Actually, just 1 big query).

In JPQL (Java Persistence Query Language), we write:
```sql
SELECT p FROM Product p JOIN FETCH p.category
```

---

## 📄 Pagination & Sorting

If you have 10,000 products, you **never** return all of them at once. You return them in pages (e.g., 20 per page). Spring Data JPA makes this easy using the `Pageable` interface.

---

## 🛠️ Step-by-Step Build

### Step 1: Fix N+1 in the Repository
Open `ProductRepository.java` and add a custom query:

```java
// src/main/java/com/example/product/repository/ProductRepository.java
package com.example.product.repository;

import com.example.product.entity.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface ProductRepository extends JpaRepository<Product, Long> {

    // 1. Fix N+1 for getting ALL products (Use with caution on huge tables)
    @Query("SELECT p FROM Product p JOIN FETCH p.category")
    List<Product> findAllWithCategory();

    // 2. Fix N+1 WITH Pagination and Sorting! (The Best Way)
    @Query("SELECT p FROM Product p JOIN FETCH p.category")
    Page<Product> findAllWithCategoryPaged(Pageable pageable);
}
```

### Step 2: Update the Service for Pagination
```java
// In ProductService.java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

@Transactional(readOnly = true)
public Page<ProductResponse> getAllProductsPaged(Pageable pageable) {
    // 1. Fetch data efficiently (1 query) with pagination
    Page<Product> productPage = productRepository.findAllWithCategoryPaged(pageable);
    
    // 2. Map the entities to DTOs
    return productPage.map(productMapper::toResponse);
}
```

### Step 3: Update the Controller
Spring Boot automatically converts URL parameters (like `?page=0&size=10&sort=name,asc`) into a `Pageable` object!

```java
// In ProductController.java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;

@GetMapping("/paged")
public ResponseEntity<Page<ProductResponse>> getProductsPaged(
        @PageableDefault(size = 10) Pageable pageable) { 
        // ↑ Defaults to 10 items per page if not specified in URL
        
    Page<ProductResponse> page = productService.getAllProductsPaged(pageable);
    return ResponseEntity.ok(page);
}
```

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

**Test 1: Get Page 0, Size 2, Sorted by Price Descending**
```bash
curl "http://localhost:8080/api/products/paged?page=0&size=2&sort=price,desc"
```
*(Note the quotes around the URL because of the `&` symbol in bash!)*

**Expected Response:**
```json
{
  "content": [
    { "id": 2, "name": "Laptop", "price": 999.99, "category": {"name": "Electronics"} },
    { "id": 1, "name": "Mouse", "price": 29.99, "category": {"name": "Electronics"} }
  ],
  "pageable": { ... },
  "totalPages": 5,
  "totalElements": 10,
  "number": 0,
  "size": 2,
  "first": true,
  "last": false
}
```
Notice how Spring gives you metadata (`totalPages`, `totalElements`, `first`, `last`) for free!

---

## 🚨 Common Errors
1. **`MultipleBagFetchException`**: You tried to `JOIN FETCH` two `List` collections at the same time. *Fix:* Change one of the `List`s to a `Set` in your Entity.
2. **`Pagination might not work correctly with JOIN FETCH` warning**: Hibernate warns you if you paginate and fetch a `@OneToMany` (One-to-Many) collection, because it messes up the total count. *Fix:* It works perfectly fine for `@ManyToOne` (Many-to-One), which is what we are doing here!

---

## 🛠️ Exercise
1. Add a method to find products by category ID using `JOIN FETCH` and Pagination.
2. Endpoint: `GET /api/products/category/{categoryId}?page=0&size=5`
3. Test it with `curl`.

---

## 🧠 Quiz
1. What is the N+1 problem?
2. How does `JOIN FETCH` solve it?
3. What interface does Spring use to handle pagination parameters from the URL?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 3.