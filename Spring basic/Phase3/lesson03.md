# 📘 Phase 3, Lesson 3: Pagination with Relationships

---

## 🎯 Goal
Add pagination and sorting to the Product API so it works efficiently even with thousands of products, and correctly loads category data for each page.

---

## 🧠 The Big Picture

Imagine a library with 1,000,000 books. If someone asks "Show me all books," you don't carry 1,000,000 books to the front desk. You say "Here are the first 20. Want the next page?"

That's **pagination**. We return data in small chunks (pages) instead of all at once.

**Sorting** means the client can choose the order: "Show me products sorted by price, cheapest first."

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **Page** | A small chunk of data (e.g., 10 items) |
| **Pageable** | A Spring object that holds page number, page size, and sort order |
| **`Page<T>`** | A Spring object that holds the data PLUS metadata (total pages, total items) |
| **Offset** | How many items to skip (Page 2 with size 10 = skip 10 items) |

---

## 🛠️ Step 1: Create the Paging Wrapper

### What we're doing:
Create a generic class to hold paginated data and metadata.

### The Code:
Create file: `src/main/java/com/example/demo/common/Paging.java`

```java
package com.example.demo.common;

import lombok.Builder;
import lombok.Getter;
import java.util.List;

@Getter
@Builder
public class Paging<T> {
    private List<T> items;        // The actual data for this page
    private int page;             // Current page number (starts at 0)
    private int size;             // How many items per page
    private long totalElements;   // Total number of items in the database
    private int totalPages;       // Total number of pages
}
```

### 📝 After the Code - What Just Happened?
- `<T>` is a **Generic type**. It means this class can hold any type of data: `Paging<ProductResponse>`, `Paging<CategoryResponse>`, etc.
- `items`: The actual list of data for the current page.
- `page`: The current page number. Spring starts counting at 0 (so page 0 is the first page).
- `totalElements`: How many items exist in total across ALL pages.
- `totalPages`: How many pages exist in total.

### 📦 Imports to Remember
```java
import lombok.Builder;
import lombok.Getter;
import java.util.List;
```

---

## 🛠️ Step 2: Update the Repository

### What we're doing:
Add a method that returns a `Page<Product>` instead of a `List<Product>`.

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

    // Fetch products with their categories in ONE query (prevents N+1 problem)
    @Query("SELECT p FROM Product p JOIN FETCH p.category")
    Page<Product> findAllWithCategory(Pageable pageable);
}
```

### 📝 After the Code - What Just Happened?
- `Page<Product>`: Instead of returning a simple `List`, this returns a `Page` object that contains the data AND metadata (total count, total pages).
- `Pageable pageable`: This is the input. Spring automatically converts URL parameters like `?page=0&size=10&sort=price,asc` into this object.
- `@Query("SELECT p FROM Product p JOIN FETCH p.category")`: This is **critical for performance**. It tells JPA to load the product AND its category in a single SQL query. Without `JOIN FETCH`, JPA would run one query for products, then one query PER PRODUCT to load each category (the N+1 problem).

### 📦 Imports to Remember
```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.Query;
```

### 💡 Note
> **The N+1 Problem**: If you have 20 products and use `LAZY` loading without `JOIN FETCH`, JPA runs 1 query to get products + 20 queries to get each product's category = 21 queries total! With `JOIN FETCH`, it runs just 1 query. Always use `JOIN FETCH` when you know you'll need the related data.

---

## 🛠️ Step 3: Update the Service

### What we're doing:
Add a paginated method to the ProductService.

### The Code:

**Update interface:** `src/main/java/com/example/demo/service/ProductService.java`
```java
import com.example.demo.common.Paging;
import org.springframework.data.domain.Pageable;

// Add this method to the interface:
Paging<ProductResponse> getAllPaged(Pageable pageable);
```

**Update implementation:** `src/main/java/com/example/demo/service/impl/ProductServiceImpl.java`
```java
@Override
@Transactional(readOnly = true)
public Paging<ProductResponse> getAllPaged(Pageable pageable) {
    // 1. Fetch one page of products (with categories) from the database
    Page<Product> productPage = productRepository.findAllWithCategory(pageable);

    // 2. Convert entities to DTOs
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

### 📝 After the Code - What Just Happened?
1. `productRepository.findAllWithCategory(pageable)`: Fetches only the requested page (e.g., 10 items) with categories loaded efficiently.
2. `productPage.getContent()`: Gets the actual list of products from the Page object.
3. `.map(productMapper::toResponse)`: Converts each Product entity to a ProductResponse DTO.
4. We build the `Paging` wrapper with the data and all the metadata.

### 📦 Imports to Remember
```java
import com.example.demo.common.Paging;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
```

---

## 🛠️ Step 4: Update the Controller

### What we're doing:
Add a new endpoint that accepts pagination parameters from the URL.

### The Code:
**Update:** `src/main/java/com/example/demo/controller/ProductController.java`

```java
import com.example.demo.common.Paging;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;

// Add this new endpoint:
@GetMapping("/paged")
public ResponseEntity<HttpBodyResponse<Paging<ProductResponse>>> getAllPaged(
        @PageableDefault(size = 10, sort = "id") Pageable pageable) {
    return responseSucceed(productService.getAllPaged(pageable));
}
```

### 📝 After the Code - What Just Happened?
- `@PageableDefault(size = 10, sort = "id")`: If the client doesn't specify page size or sort order, we default to 10 items per page, sorted by ID.
- `Pageable pageable`: Spring **automatically** reads URL parameters (`?page=0&size=5&sort=price,desc`) and converts them into this object. You don't need to parse anything manually!

### 📦 Imports to Remember
```java
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import com.example.demo.common.Paging;
```

---

## 🧪 Run & Test

### Test 1: Get page 0, 5 items per page
```bash
curl "http://localhost:8081/api/v1/products/paged?page=0&size=5"
```

**Expected Response:**
```json
{
  "status": 200,
  "message": "Success",
  "data": {
    "items": [ ... 5 products ... ],
    "page": 0,
    "size": 5,
    "totalElements": 25,
    "totalPages": 5
  }
}
```

### Test 2: Get page 1, sorted by price descending
```bash
curl "http://localhost:8081/api/v1/products/paged?page=1&size=5&sort=price,desc"
```

### Test 3: Use defaults (10 items, sorted by id)
```bash
curl "http://localhost:8081/api/v1/products/paged"
```

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `LazyInitializationException` in response | Category not loaded during pagination | Use `JOIN FETCH` in the repository query |
| Page numbers start at 1 | Spring uses 0-based indexing | Page 0 is the first page, page 1 is the second |
| `sort` parameter not working | Wrong format in URL | Use `sort=fieldName,direction` (e.g., `sort=price,asc`) |

---

## ✏️ Exercise

1. Add pagination to the Category API: `GET /api/v1/categories/paged`
2. Create a `findAllPaged(Pageable pageable)` method in `CategoryRepository`
3. Wire it through the Service and Controller
4. Test with `curl`

---

## 🧠 Quiz

1. Why is pagination important for large datasets?
2. What does `JOIN FETCH` do and why do we need it with pagination?
3. If you have 100 products and use `size=10`, how many total pages will there be?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 4.