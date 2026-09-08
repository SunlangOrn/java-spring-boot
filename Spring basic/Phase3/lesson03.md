# 📘 Phase 3, Lesson 3: Dynamic Queries with Specifications

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem: The "Method Explosion"](#-the-problem-the-method-explosion)
- [The Solution: JPA Specifications](#-the-solution-jpa-specifications)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand why writing custom query methods for every filter is a bad idea.
- ✅ Learn how to use JPA `Specification` to build dynamic queries.
- ✅ Create a flexible product search endpoint.

---

## 🚨 The Problem: The "Method Explosion"

Imagine your client wants to search for products. They might want to filter by:
- Name
- Category
- Min Price
- Max Price

If you use Spring Data JPA method names, you end up writing this:
```java
List<Product> findByName(String name);
List<Product> findByCategoryId(Long categoryId);
List<Product> findByPriceBetween(Double min, Double max);
List<Product> findByNameAndCategoryId(String name, Long categoryId);
List<Product> findByNameAndPriceBetween(String name, Double min, Double max);
// ... 20 more methods!
```
This is unmaintainable. What if they add a "Brand" filter next month? You have to rewrite everything.

---

## 💡 The Solution: JPA Specifications

Specifications allow you to build queries **dynamically** at runtime, piece by piece, like building with Lego blocks. 

If the user provides a `name`, we add the `name` block. If they provide `minPrice`, we add the `minPrice` block. If they provide nothing, we return everything.

---

## 🛠️ Step-by-Step Build

### Step 1: Update the Repository
To use Specifications, your repository must extend `JpaSpecificationExecutor`.

```java
// src/main/java/com/example/product/repository/ProductRepository.java
package com.example.product.repository;

import com.example.product.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor; // ← ADD THIS

public interface ProductRepository extends JpaRepository<Product, Long>, 
                                           JpaSpecificationExecutor<Product> { // ← ADD THIS
    // Now you have access to findAll(Specification)
}
```

### Step 2: Create a Search Request DTO
```java
// src/main/java/com/example/product/dto/request/ProductSearchRequest.java
package com.example.product.dto.request;

import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductSearchRequest {
    private String name;
    private Long categoryId;
    private Double minPrice;
    private Double maxPrice;
}
```

### Step 3: Build the Specification in the Service
This is where the magic happens. We use the `CriteriaBuilder` to create SQL conditions dynamically.

```java
// In ProductService.java
import org.springframework.data.jpa.domain.Specification;
import jakarta.persistence.criteria.Predicate;
import java.util.ArrayList;
import java.util.List;

@Transactional(readOnly = true)
public List<ProductResponse> searchProducts(ProductSearchRequest searchReq) {
    
    // 1. Create a Specification that builds the WHERE clause
    Specification<Product> spec = (root, query, criteriaBuilder) -> {
        
        List<Predicate> predicates = new ArrayList<>();

        // IF name is provided, add: WHERE name LIKE '%keyword%'
        if (searchReq.getName() != null && !searchReq.getName().isBlank()) {
            predicates.add(criteriaBuilder.like(
                    criteriaBuilder.lower(root.get("name")), 
                    "%" + searchReq.getName().toLowerCase() + "%"
            ));
        }

        // IF categoryId is provided, add: WHERE category.id = ?
        if (searchReq.getCategoryId() != null) {
            predicates.add(criteriaBuilder.equal(
                    root.get("category").get("id"), 
                    searchReq.getCategoryId()
            ));
        }

        // IF minPrice is provided, add: WHERE price >= ?
        if (searchReq.getMinPrice() != null) {
            predicates.add(criteriaBuilder.greaterThanOrEqualTo(
                    root.get("price"), 
                    searchReq.getMinPrice()
            ));
        }

        // IF maxPrice is provided, add: WHERE price <= ?
        if (searchReq.getMaxPrice() != null) {
            predicates.add(criteriaBuilder.lessThanOrEqualTo(
                    root.get("price"), 
                    searchReq.getMaxPrice()
            ));
        }

        // Combine all conditions with AND
        return criteriaBuilder.and(predicates.toArray(new Predicate[0]));
    };

    // 2. Execute the query
    List<Product> products = productRepository.findAll(spec);
    
    // 3. Map to DTOs
    return products.stream()
            .map(productMapper::toResponse)
            .toList();
}
```

### Step 4: Add the Controller Endpoint
```java
// In ProductController.java
@PostMapping("/search")
public ResponseEntity<List<ProductResponse>> searchProducts(
        @RequestBody ProductSearchRequest searchRequest) {
    return ResponseEntity.ok(productService.searchProducts(searchRequest));
}
```
*(Note: We use POST for search because the search criteria object is complex. Some prefer GET with query params, but POST is cleaner for complex filters).*

---

## 🚀 Run and Test

**Test 1: Search by name and max price**
```bash
curl -X POST http://localhost:8080/api/products/search \
-H "Content-Type: application/json" \
-d '{
  "name": "mouse",
  "maxPrice": 50.00
}'
```
**Expected:** Returns only products with "mouse" in the name AND price <= 50.

**Test 2: Search by category only**
```bash
curl -X POST http://localhost:8080/api/products/search \
-H "Content-Type: application/json" \
-d '{
  "categoryId": 1
}'
```
**Expected:** Returns all products in category 1.

**Test 3: Empty search (Returns everything)**
```bash
curl -X POST http://localhost:8080/api/products/search \
-H "Content-Type: application/json" \
-d '{}'
```
**Expected:** Returns all products.

---

## 🚨 Common Errors
1. **`Invalid path: 'category.id'`**: You tried to access `root.get("category").get("id")` but the relationship isn't loaded. *Fix:* Ensure the entity mapping is correct, or use `root.join("category").get("id")`.
2. **Case sensitivity in search**: "Mouse" doesn't match "mouse". *Fix:* Use `criteriaBuilder.lower()` as shown in the code above to make it case-insensitive.

---

## 🛠️ Exercise
1. Add a `minStock` and `maxStock` filter to the `ProductSearchRequest`.
2. Update the Specification to include these stock filters.
3. Test it with `curl`.

---

## 🧠 Quiz
1. Why is the "Method Explosion" bad?
2. What interface must a Repository extend to use Specifications?
3. How do you combine multiple `Predicate` conditions in the CriteriaBuilder?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 4.