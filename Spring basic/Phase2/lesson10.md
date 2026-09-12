# 📘 Phase 2, Lesson 10: DTOs & MapStruct (The Enterprise Way)

## 🎯 Learning Goal
- ✅ Understand why we must **never** return JPA Entities directly to the client.
- ✅ Create Request and Response DTOs (Data Transfer Objects).
- ✅ Use MapStruct to automatically map between Entities and DTOs.

---

## 💡 The Concept: Why DTOs?
Right now, our Controller returns the `Product` Entity directly. This is dangerous because:
1. **Security**: If you add a `password` or `internalNotes` field to the Entity later, it will automatically be exposed to the client!
2. **Flexibility**: The database structure is forced on the client. What if the client wants the price formatted as `"$150.00"`? You can't do that with a `Double`.
3. **Performance**: You might load 5 related tables from the database, but the client only needs 2 fields.

**The Solution**: Use **DTOs**. A DTO is a simple class that contains *only* the data the client is allowed to see.

---

## 🛠️ Step-by-Step Build

### Step 1: Add MapStruct Dependencies
Open `pom.xml` and add these inside `<dependencies>`:
```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
```
And add this inside the `<build><plugins><plugin>` (maven-compiler-plugin) configuration:
```xml
<annotationProcessorPaths>
    <path>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct-processor</artifactId>
        <version>1.5.5.Final</version>
    </path>
    <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.30</version>
    </path>
    <path>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok-mapstruct-binding</artifactId>
        <version>0.2.0</version>
    </path>
</annotationProcessorPaths>
```

### Step 2: Create the Response DTO
Create a new package `dto.response` and add `ProductResponse.java`:
```java
// src/main/java/com/example/demo/dto/response/ProductResponse.java
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
    // Notice: No setters! Responses should be immutable.
}
```

### Step 3: Update the Request DTO
Ensure your `dto.ProductRequest` (from Lesson 5) looks like this:
```java
// src/main/java/com/example/demo/dto/ProductRequest.java
package com.example.demo.dto;

public record ProductRequest(String name, Double price, String description) {}
```

### Step 4: Create the MapStruct Mapper
Create a new package `mapper` and add `ProductMapper.java`:
```java
// src/main/java/com/example/demo/mapper/ProductMapper.java
package com.example.demo.mapper;

import com.example.demo.dto.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import com.example.demo.entity.Product;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

// componentModel = "spring" tells Spring to manage this mapper as a Bean
@Mapper(componentModel = "spring")
public interface ProductMapper {

    // Entity -> Response DTO
    ProductResponse toResponse(Product product);

    // Request DTO -> Entity
    @Mapping(target = "id", ignore = true) // Database generates the ID
    @Mapping(target = "createdAt", ignore = true) // JPA Auditing handles this
    @Mapping(target = "updatedAt", ignore = true)
    Product toEntity(ProductRequest request);
}
```

### Step 5: Update the Service to use the Mapper
Update `ProductService.java`:
```java
// src/main/java/com/example/demo/service/ProductService.java
package com.example.demo.service;

import com.example.demo.dto.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import com.example.demo.entity.Product;
import com.example.demo.mapper.ProductMapper;
import com.example.demo.repository.ProductRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper; // Inject the mapper!

    @Transactional(readOnly = true)
    public List<ProductResponse> getAllProducts() {
        return productRepository.findAll().stream()
                .map(productMapper::toResponse) // Map each entity to DTO
                .toList();
    }

    @Transactional(readOnly = true)
    public ProductResponse getProductById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Product not found with id: " + id));
        return productMapper.toResponse(product);
    }

    @Transactional
    public ProductResponse addProduct(ProductRequest request) {
        Product newProduct = productMapper.toEntity(request); // DTO -> Entity
        Product savedProduct = productRepository.save(newProduct);
        return productMapper.toResponse(savedProduct); // Entity -> DTO
    }
    
    @Transactional
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
}
```

### Step 6: Update the Controller
Update `ProductController.java` to return `ProductResponse` instead of `Product`:
```java
// src/main/java/com/example/demo/controller/ProductController.java
package com.example.demo.controller;

import com.example.demo.dto.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import com.example.demo.service.ProductService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @GetMapping
    public List<ProductResponse> getAllProducts() {
        return productService.getAllProducts();
    }

    @GetMapping("/{id}")
    public ProductResponse getProductById(@PathVariable Long id) {
        return productService.getProductById(id);
    }

    @PostMapping
    public ProductResponse addProduct(@RequestBody ProductRequest request) {
        return productService.addProduct(request);
    }

    @DeleteMapping("/{id}")
    public void deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
    }
}
```

---

## 🚀 Run and Test
1. Run `./mvnw clean compile` (This triggers MapStruct to generate the mapping code).
2. Run the app.
3. Test POST and GET. Notice the JSON response now perfectly matches `ProductResponse`, hiding any internal Entity fields we didn't include!

---

## 🛠️ Exercise
1. Add a `private Integer stock;` field to the `Product` Entity.
2. Add `stock` to `ProductRequest` and `ProductResponse`.
3. Update the Flyway migration (create `V3__add_stock.sql`) to add the column.
4. Test creating a product with stock and verify it appears