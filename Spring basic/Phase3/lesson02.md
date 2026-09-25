# 📘 Phase 3, Lesson 2: Category CRUD & Product-Category Integration

---

## 🎯 Goal
Build complete Create, Read, Update, Delete (CRUD) endpoints for Categories, and update the Product API so clients can assign a category when creating a product.

---

## 🧠 The Big Picture

In Lesson 1, we created the database tables and linked them. But our API doesn't know about categories yet. 

Right now, if a client wants to create a product, they have no way to say "This product belongs to Electronics." 

We need to:
1. Build Category endpoints (so clients can create and view categories)
2. Update the Product endpoints (so clients can assign a category to a product)

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **DTO** | Data Transfer Object - a simple class that controls what data goes in and out of the API |
| **Request DTO** | What the client sends to us (e.g., "Create a category named Books") |
| **Response DTO** | What we send back to the client (e.g., "Here is the category you created") |
| **Nested DTO** | A DTO inside another DTO (e.g., a CategoryResponse inside a ProductResponse) |

---

## 🛠️ Step 1: Create Category DTOs

### What we're doing:
Create two simple classes: one for incoming data (Request) and one for outgoing data (Response).

### The Code:

**File 1:** `src/main/java/com/example/demo/dto/request/CategoryRequest.java`
```java
package com.example.demo.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record CategoryRequest(
    @NotBlank(message = "Category name is required")
    @Size(min = 2, max = 100, message = "Name must be 2-100 characters")
    String name
) {}
```

**File 2:** `src/main/java/com/example/demo/dto/response/CategoryResponse.java`
```java
package com.example.demo.dto.response;

import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
public class CategoryResponse {
    private Long id;
    private String name;
}
```

### 📝 After the Code - What Just Happened?
- **CategoryRequest**: Uses Java `record` (a shortcut for creating simple data classes). It has one field: `name`. The `@NotBlank` and `@Size` annotations validate the data before it reaches the service.
- **CategoryResponse**: Uses Lombok `@Builder` and `@Getter`. This is what the client will see. Notice it has `id` and `name` but NO internal database fields like `createdAt`.

### 📦 Imports to Remember
```java
// For Request:
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

// For Response:
import lombok.Builder;
import lombok.Getter;
```

---

## 🛠️ Step 2: Update Product DTOs

### What we're doing:
Add `categoryId` to the Product request (so clients can specify which category) and add `CategoryResponse` to the Product response (so clients can see the category name).

### The Code:

**Update:** `src/main/java/com/example/demo/dto/request/ProductRequest.java`
```java
package com.example.demo.dto.request;

import jakarta.validation.constraints.*;

public record ProductRequest(
    @NotBlank(message = "Product name is required")
    @Size(min = 3, max = 100, message = "Name must be 3-100 characters")
    String name,

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be greater than zero")
    Double price,

    @Size(max = 500, message = "Description cannot exceed 500 characters")
    String description,

    @NotNull(message = "Category ID is required")
    Long categoryId  // NEW: The client sends the category ID number
) {}
```

**Update:** `src/main/java/com/example/demo/dto/response/ProductResponse.java`
```java
package com.example.demo.dto.response;

import lombok.Builder;
import lombok.Getter;
import java.time.LocalDateTime;

@Getter
@Builder
public class ProductResponse {
    private Long id;
    private String name;
    private Double price;
    private String description;
    private LocalDateTime createdAt;
    
    // NEW: Include the category details in the response
    private CategoryResponse category;
}
```

### 📝 After the Code - What Just Happened?
- **ProductRequest**: We added `Long categoryId`. The client will send a number (like `1`) instead of the whole category object. This is simpler and safer.
- **ProductResponse**: We added `CategoryResponse category`. When the client gets a product, they will see the category name nested inside the response JSON.

### 💡 Note
> Why send `categoryId` (a number) in the request but return `CategoryResponse` (an object) in the response? 
> - **Request**: The client only needs to tell us WHICH category. A number is enough.
> - **Response**: The client wants to SEE the category details (name, etc.) without making a second API call.

---

## 🛠️ Step 3: Update the MapStruct Mapper

### What we're doing:
Tell MapStruct how to convert between Entity and DTO, including the new category field.

### The Code:
**Update:** `src/main/java/com/example/demo/mapper/ProductMapper.java`

```java
package com.example.demo.mapper;

import com.example.demo.dto.request.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import com.example.demo.entity.Product;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface ProductMapper {

    // Entity → Response: MapStruct automatically maps nested objects
    ProductResponse toResponse(Product product);

    // Request → Entity: Ignore fields that the Service will handle
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "category", ignore = true)  // Service sets this manually
    Product toEntity(ProductRequest request);
}
```

### 📝 After the Code - What Just Happened?
- `toResponse()`: MapStruct is smart enough to see that `Product` has a `Category` field and `ProductResponse` has a `CategoryResponse` field. It automatically maps `category.id` → `category.id` and `category.name` → `category.name`. No extra code needed!
- `toEntity()`: We use `@Mapping(target = "category", ignore = true)` because the Service layer needs to fetch the real Category from the database first. MapStruct can't do that.

### 📦 Imports to Remember
```java
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
```

---

## 🛠️ Step 4: Create CategoryService & CategoryServiceImpl

### What we're doing:
Build the business logic for creating, reading, updating, and deleting categories.

### The Code:

**File 1:** `src/main/java/com/example/demo/service/CategoryService.java`
```java
package com.example.demo.service;

import com.example.demo.dto.request.CategoryRequest;
import com.example.demo.dto.response.CategoryResponse;
import java.util.List;

public interface CategoryService {
    CategoryResponse create(CategoryRequest request);
    List<CategoryResponse> getAll();
    CategoryResponse getById(Long id);
    CategoryResponse update(Long id, CategoryRequest request);
    void delete(Long id);
}
```

**File 2:** `src/main/java/com/example/demo/service/impl/CategoryServiceImpl.java`
```java
package com.example.demo.service.impl;

import com.example.demo.dto.request.CategoryRequest;
import com.example.demo.dto.response.CategoryResponse;
import com.example.demo.entity.Category;
import com.example.demo.exception.ResourceNotFoundException;
import com.example.demo.repository.CategoryRepository;
import com.example.demo.service.CategoryService;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class CategoryServiceImpl implements CategoryService {

    private final CategoryRepository categoryRepository;

    @Override
    @Transactional
    public CategoryResponse create(CategoryRequest request) {
        Category category = Category.builder()
                .name(request.name())
                .build();
        
        Category saved = categoryRepository.save(category);
        
        return CategoryResponse.builder()
                .id(saved.getId())
                .name(saved.getName())
                .build();
    }

    @Override
    @Transactional(readOnly = true)
    public List<CategoryResponse> getAll() {
        return categoryRepository.findAll().stream()
                .map(c -> CategoryResponse.builder()
                        .id(c.getId())
                        .name(c.getName())
                        .build())
                .toList();
    }

    @Override
    @Transactional(readOnly = true)
    public CategoryResponse getById(Long id) {
        Category category = categoryRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException(
                    "Category not found with id: " + id));
        
        return CategoryResponse.builder()
                .id(category.getId())
                .name(category.getName())
                .build();
    }

    @Override
    @Transactional
    public CategoryResponse update(Long id, CategoryRequest request) {
        Category category = categoryRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException(
                    "Category not found with id: " + id));
        
        category.setName(request.name());
        Category updated = categoryRepository.save(category);
        
        return CategoryResponse.builder()
                .id(updated.getId())
                .name(updated.getName())
                .build();
    }

    @Override
    @Transactional
    public void delete(Long id) {
        if (!categoryRepository.existsById(id)) {
            throw new ResourceNotFoundException("Category not found with id: " + id);
        }
        categoryRepository.deleteById(id);
    }
}
```

### 📝 After the Code - What Just Happened?
- **Interface**: Defines the "contract" (what methods exist). The Controller depends on this, not the implementation.
- **ServiceImpl**: Contains the actual logic. Each method:
  1. `create()`: Builds a Category from the request, saves it, returns the response.
  2. `getAll()`: Fetches all categories and maps them to DTOs.
  3. `getById()`: Finds one category or throws a 404 error.
  4. `update()`: Finds the category, changes the name, saves it.
  5. `delete()`: Checks if it exists first, then deletes it.

### 📦 Imports to Remember
```java
import com.example.demo.exception.ResourceNotFoundException;  // Your custom exception
import lombok.RequiredArgsConstructor;  // Generates constructor for final fields
import org.springframework.transaction.annotation.Transactional;
```

### 💡 Note
> `@Transactional(readOnly = true)` on read methods is a **performance optimization**. It tells the database "I'm only reading, not writing," which allows the database to skip some internal work and run faster.

---

## 🛠️ Step 5: Update ProductService to Link Categories

### What we're doing:
When a client creates a product with a `categoryId`, the service must fetch the real Category from the database and attach it to the product.

### The Code:
**Update:** `src/main/java/com/example/demo/service/impl/ProductServiceImpl.java`

Add `CategoryRepository` to the constructor and update `addProduct()`:

```java
@Service
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;
    private final CategoryRepository categoryRepository;  // NEW

    @Override
    @Transactional
    public ProductResponse addProduct(ProductRequest request) {
        // 1. Find the category in the database
        Category category = categoryRepository.findById(request.categoryId())
                .orElseThrow(() -> new ResourceNotFoundException(
                    "Category not found with id: " + request.categoryId()));

        // 2. Convert request to entity (category is ignored by MapStruct)
        Product product = productMapper.toEntity(request);

        // 3. Manually set the category
        product.setCategory(category);

        // 4. Save and return
        Product saved = productRepository.save(product);
        return productMapper.toResponse(saved);
    }

    // ... other methods stay the same
}
```

### 📝 After the Code - What Just Happened?
1. We fetch the Category from the database using the `categoryId` the client sent.
2. If the category doesn't exist, we throw a `ResourceNotFoundException` (404 error).
3. MapStruct converts the request to an entity, but skips the category (because we told it to ignore it).
4. We manually attach the real Category object to the Product.
5. We save the product. Hibernate automatically writes the `category_id` to the database.

### 📦 Imports to Remember
```java
import com.example.demo.entity.Category;
import com.example.demo.repository.CategoryRepository;
import com.example.demo.exception.ResourceNotFoundException;
```

---

## 🛠️ Step 6: Create CategoryController

### What we're doing:
Create the API endpoints for categories.

### The Code:
Create file: `src/main/java/com/example/demo/controller/CategoryController.java`

```java
package com.example.demo.controller;

import com.example.demo.common.BaseRestController;
import com.example.demo.common.HttpBodyResponse;
import com.example.demo.dto.request.CategoryRequest;
import com.example.demo.dto.response.CategoryResponse;
import com.example.demo.service.CategoryService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/categories")
@RequiredArgsConstructor
public class CategoryController extends BaseRestController {

    private final CategoryService categoryService;

    @PostMapping
    public ResponseEntity<HttpBodyResponse<CategoryResponse>> create(
            @Valid @RequestBody CategoryRequest request) {
        return responseCreated(categoryService.create(request));
    }

    @GetMapping
    public ResponseEntity<HttpBodyResponse<List<CategoryResponse>>> getAll() {
        return responseSucceed(categoryService.getAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<HttpBodyResponse<CategoryResponse>> getById(@PathVariable Long id) {
        return responseSucceed(categoryService.getById(id));
    }

    @PutMapping("/{id}")
    public ResponseEntity<HttpBodyResponse<CategoryResponse>> update(
            @PathVariable Long id,
            @Valid @RequestBody CategoryRequest request) {
        return responseSucceed(categoryService.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<HttpBodyResponse<Void>> delete(@PathVariable Long id) {
        categoryService.delete(id);
        return responseDeleted();
    }
}
```

### 📝 After the Code - What Just Happened?
- This controller follows the exact same pattern as `ProductController`.
- It extends `BaseRestController` to use the helper methods (`responseCreated`, `responseSucceed`, `responseDeleted`).
- Each endpoint calls the corresponding method in `CategoryService`.

---

## 🔄 How It All Connects

```text
Client sends POST /api/v1/products
    ↓
ProductController receives ProductRequest (with categoryId: 1)
    ↓
ProductServiceImpl.addProduct()
    ↓
    ├── CategoryRepository.findById(1) → finds "Electronics"
    ├── ProductMapper.toEntity(request) → creates Product (no category yet)
    ├── product.setCategory(electronics) → links them
    └── ProductRepository.save(product) → saves to database
    ↓
ProductMapper.toResponse(saved) → creates ProductResponse with nested CategoryResponse
    ↓
Client receives:
{
  "status": 201,
  "data": {
    "id": 1,
    "name": "Laptop",
    "category": { "id": 1, "name": "Electronics" }
  }
}
```

---

## 🧪 Run & Test

### Test 1: Create a category
```bash
curl -X POST http://localhost:8081/api/v1/categories \
-H "Content-Type: application/json" \
-d '{"name": "Electronics"}'
```

### Test 2: Create a product with that category
```bash
curl -X POST http://localhost:8081/api/v1/products \
-H "Content-Type: application/json" \
-d '{"name": "Laptop", "price": 999.99, "description": "Gaming", "categoryId": 1}'
```

### Test 3: Get all categories
```bash
curl http://localhost:8081/api/v1/categories
```

### Test 4: Update a category
```bash
curl -X PUT http://localhost:8081/api/v1/categories/1 \
-H "Content-Type: application/json" \
-d '{"name": "Tech & Gadgets"}'
```

### Test 5: Delete a category
```bash
curl -X DELETE http://localhost:8081/api/v1/categories/1
```

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Category not found with id: X` | Client sent a categoryId that doesn't exist | Create the category first before creating products |
| `category` is null in response | Product was created before the category link was added | Ensure `product.setCategory()` is called before `save()` |
| `400 Bad Request` on product creation | Forgot to include `categoryId` in the JSON | Add `"categoryId": 1` to the request body |

---

## ✏️ Exercise

1. Add a `description` field to `CategoryRequest` and `CategoryResponse`.
2. Update `CategoryServiceImpl` to handle the description.
3. Test creating a category with a description.

---

## 🧠 Quiz

1. Why do we send `categoryId` (a number) in the request instead of the whole category object?
2. What happens if a client sends `categoryId: 999` but category 999 doesn't exist?
3. Why do we use `@Mapping(target = "category", ignore = true)` in the ProductMapper?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 3.