# 📘 Phase 3, Lesson 1: JPA Relationships & E-Commerce Domain Modeling

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem: Data Doesn't Live in Isolation](#-the-problem-data-doesnt-live-in-isolation)
- [The Analogy: Folders and Files](#-the-analogy-folders-and-files)
- [Step-by-Step Build](#-step-by-step-build)
- [How It Works Internally](#-how-it-works-internally)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand how to model relationships in a database (One-to-Many, Many-to-One).
- ✅ Learn the concept of the "Owning Side" in JPA.
- ✅ Build the foundation for an E-Commerce backend (Categories and Products).
- ✅ Map nested DTOs using MapStruct.

---

## 🚨 The Problem: Data Doesn't Live in Isolation

In Phase 2, our `Product` existed all by itself. But in the real world, data is connected:
- A **Product** belongs to a **Category**.
- An **Order** contains many **OrderItems**.
- A **User** can write many **Reviews**.

If we just store `category_name` as a String inside the `Product` table, what happens if we rename the category? We'd have to update thousands of product rows! 

**The Solution:** Relational databases use **Foreign Keys** to link tables. JPA uses **Annotations** to map these links to Java objects.

---

## 📁 The Analogy: Folders and Files

Imagine your computer:
- A **Folder** (Category) can contain **many Files** (Products).
- A **File** (Product) belongs to **one Folder** (Category).

This is a **One-to-Many** relationship from the Folder's perspective, and a **Many-to-One** relationship from the File's perspective.

In databases, the "File" (Product) holds the reference (the Foreign Key) to the "Folder" (Category). The File is the **Owning Side**.

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Flyway Migrations
We need two tables: `categories` and `products`. The `products` table will have a `category_id` column.

Create `src/main/resources/db/migration/V3__create_categories_table.sql`:
```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT
);
```

Create `src/main/resources/db/migration/V4__add_category_to_products.sql`:
```sql
-- Add the category_id column to the existing products table
ALTER TABLE products 
ADD COLUMN category_id BIGINT;

-- Add a Foreign Key constraint (ensures category_id actually exists in categories table)
ALTER TABLE products 
ADD CONSTRAINT fk_product_category 
FOREIGN KEY (category_id) REFERENCES categories(id);

-- Add an index for faster lookups
CREATE INDEX idx_product_category ON products(category_id);
```

### Step 2: Create the Category Entity
```java
// src/main/java/com/example/product/entity/Category.java
package com.example.product.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "categories")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;

    private String description;

    // ONE Category can have MANY Products
    // mappedBy = "category" tells JPA: "The 'category' field in the Product class owns this relationship"
    @OneToMany(mappedBy = "category", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private java.util.List<Product> products;
}
```

### Step 3: Create the Product Entity (The Owning Side)
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

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Double price;

    @Column(nullable = false)
    private Integer stock;

    // MANY Products can belong to ONE Category
    // This is the OWNING SIDE because it has the @JoinColumn (the Foreign Key)
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id") // Matches the column in V4 migration
    private Category category;
}
```

**Line-by-line explanation of `@ManyToOne`:**
- `@ManyToOne`: Tells JPA this entity holds the foreign key to another entity.
- `fetch = FetchType.LAZY`: **Crucial for performance!** It tells JPA: "Don't load the Category data from the database unless I specifically ask for it." (We will explore this deeply in Lesson 2).
- `@JoinColumn(name = "category_id")`: Explicitly names the foreign key column in the database.

### Step 4: Update the DTOs
We need to update our DTOs to include category information.

**CategoryResponse.java:**
```java
// src/main/java/com/example/product/dto/response/CategoryResponse.java
package com.example.product.dto.response;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class CategoryResponse {
    private Long id;
    private String name;
}
```

**ProductResponse.java (Nested DTO):**
```java
// src/main/java/com/example/product/dto/response/ProductResponse.java
package com.example.product.dto.response;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductResponse {
    private Long id;
    private String name;
    private Double price;
    private Integer stock;
    
    // Nested DTO!
    private CategoryResponse category; 
}
```

**ProductCreateRequest.java:**
```java
// src/main/java/com/example/product/dto/request/ProductCreateRequest.java
package com.example.product.dto.request;

import jakarta.validation.constraints.*;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductCreateRequest {

    @NotBlank(message = "Product name is required")
    private String name;

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be greater than zero")
    private Double price;

    @NotNull(message = "Stock is required")
    @PositiveOrZero(message = "Stock cannot be negative")
    private Integer stock;

    @NotNull(message = "Category ID is required")
    private Long categoryId; // Client sends the ID, not the whole object
}
```

### Step 5: Update MapStruct for Nested Mapping
MapStruct is smart enough to handle nested objects automatically if the names match, but we need to tell it how to handle the `categoryId` from the request.

```java
// src/main/java/com/example/product/mapper/ProductMapper.java
package com.example.product.mapper;

import com.example.product.dto.request.ProductCreateRequest;
import com.example.product.dto.response.ProductResponse;
import com.example.product.entity.Product;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface ProductMapper {

    // MapStruct automatically maps product.getCategory() to CategoryResponse
    ProductResponse toResponse(Product product);

    // We tell MapStruct: "Take request.categoryId and put it in product.category.id"
    @Mapping(target = "category.id", source = "categoryId")
    @Mapping(target = "id", ignore = true)
    Product toEntity(ProductCreateRequest request);
}
```

### Step 6: Update the Service
```java
// In ProductService.java
@Transactional
public ProductResponse createProduct(ProductCreateRequest request) {
    // 1. MapStruct creates a Product with a "stub" Category (only the ID is set)
    Product product = productMapper.toEntity(request);
    
    // 2. Save to database. Hibernate will verify the category_id exists 
    //    because of the Foreign Key constraint.
    Product savedProduct = productRepository.save(product);
    
    // 3. Return the full response (Hibernate will fetch the Category details for the response)
    return productMapper.toResponse(savedProduct);
}
```

---

## ⚙️ How It Works Internally

When you call `productRepository.save(product)` where `product` has a `Category` with `id = 1`:

1. Hibernate looks at the `@JoinColumn(name = "category_id")`.
2. It generates this SQL: 
   ```sql
   INSERT INTO products (name, price, stock, category_id) VALUES ('Laptop', 999.99, 10, 1);
   ```
3. PostgreSQL checks the Foreign Key constraint. If Category `1` exists, it succeeds. If not, it throws a `DataIntegrityViolationException`.

---

## 🚀 Run and Test

1. Ensure your Docker PostgreSQL container is running.
2. Run the app: `./mvnw spring-boot:run`
3. Watch the console: Flyway will apply `V3` and `V4` automatically.

**Test 1: Create a Category first** (You can do this via a simple POST endpoint or directly in the database for now. Let's add a quick one):
```bash
# Add this to CategoryController if you want, or just insert via SQL:
# INSERT INTO categories (name, description) VALUES ('Electronics', 'Gadgets and devices');
```
*(For this test, let's assume a category with `id = 1` exists in your database).*

**Test 2: Create a Product with a Category**
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{
  "name": "Wireless Mouse",
  "price": 29.99,
  "stock": 50,
  "categoryId": 1
}'
```

**Expected Response:**
```json
{
  "id": 1,
  "name": "Wireless Mouse",
  "price": 29.99,
  "stock": 50,
  "category": {
    "id": 1,
    "name": "Electronics"
  }
}
```
Notice how the nested `category` object is beautifully formatted in the JSON response!

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| `org.hibernate.LazyInitializationException` | Trying to access `product.getCategory().getName()` outside of a `@Transactional` method. | Keep the logic inside a `@Transactional` service method, or use `JOIN FETCH` (Lesson 2). |
| `DataIntegrityViolationException: foreign key constraint` | The `categoryId` sent in the request does not exist in the database. | Ensure the category exists before creating the product. |
| Infinite JSON recursion (StackOverflow) | Jackson tries to serialize Product → Category → Products → Category... | Use `@JsonManagedReference` and `@JsonBackReference`, or better yet, **use DTOs** (which we are doing, so this is prevented!). |

---

## 🛠️ Exercise
1. Create a `User` entity with `id`, `username`, and `email`.
2. Add a `@ManyToOne` relationship in the `Product` entity called `createdBy` (type `User`), with a `@JoinColumn(name = "created_by_id")`.
3. Create the Flyway migration `V5__add_created_by_to_products.sql` to add this column.
4. Update the `ProductResponse` DTO to include `UserResponse createdBy`.
5. Test creating a product and verify the creator's username appears in the response.

---

## 🧠 Quiz
1. In a One-to-Many relationship between Category and Product, which side is the "Owning Side" and why?
2. What does `fetch = FetchType.LAZY` do, and why is it the default/recommended setting for `@ManyToOne`?
 “mappedBy” attribute in `@OneToMany`?
3. Why do we send `categoryId` (a Long) in the Request DTO, instead of sending the whole `Category` object?

---

## 🛑 STOP
**Do not move forward.** 

Reply with:
1. Confirmation that you created the Category/Product relationship, ran the migrations, and tested the nested JSON response.
2. Your code for the **Exercise** (User relationship).
3. Your answers to the 3 quiz questions.

Once you reply, we will move to **Phase 3, Lesson 2: Advanced JPA (The N+1 Problem, Pagination, and Sorting)**, which is critical for building high-performance APIs!