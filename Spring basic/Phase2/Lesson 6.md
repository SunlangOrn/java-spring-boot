# 📘 Phase 2, Lesson 6: MapStruct (Automating Mapping)

## 🎯 Learning Goals
- Understand why manual mapping is tedious.
- Add MapStruct to the project.
- Automate Entity ↔ DTO mapping.

## 💡 Concepts Explained Simply
Writing `response.setName(entity.getName())` for 50 fields is boring and error-prone. **MapStruct** generates this code automatically at compile time. It's fast (no reflection) and type-safe.

## ⚙️ Step-by-Step Build

### Step 1: Add MapStruct to `pom.xml`
```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
<!-- Add mapstruct-processor and lombok-mapstruct-binding in maven-compiler-plugin annotationProcessorPaths -->
```

### Step 2: Create the Mapper Interface
```java
// src/main/java/com/example/product/mapper/ProductMapper.java
package com.example.product.mapper;

import com.example.product.dto.request.ProductCreateRequest;
import com.example.product.dto.response.ProductResponse;
import com.example.product.model.Product;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface ProductMapper {
    
    ProductResponse toResponse(Product product);

    @Mapping(target = "id", ignore = true) // DB generates the ID
    Product toEntity(ProductCreateRequest request);
}
```

### Step 3: Use Mapper in Service
```java
// Inject ProductMapper in ProductService
public ProductResponse createProduct(ProductCreateRequest request) {
    Product product = productMapper.toEntity(request);
    Product saved = productRepository.save(product);
    return productMapper.toResponse(saved);
}
```

## 🧠 Quiz
1. Does MapStruct use reflection at runtime?
2. What does `componentModel = "spring"` do?
3. Why do we use `@Mapping(target = "id", ignore = true)`?

---