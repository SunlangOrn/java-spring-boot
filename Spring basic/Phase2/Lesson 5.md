
# 📘 Phase 2, Lesson 5: Understanding DTOs (Manual Mapping)

## 🎯 Learning Goals
- Understand **why** we never expose JPA Entities directly to clients.
- Create Request and Response DTOs.
- Manually map Entities to DTOs.

## 💡 Concepts Explained Simply
**The Restaurant Menu Analogy**: The kitchen (database) has raw ingredients. The menu (DTO) only shows what the customer needs to see. 
If you expose the Entity directly, you might accidentally leak sensitive fields (like `password` or `internalNotes`), and changing the database structure will break the API for your clients.

## ⚙️ Step-by-Step Build

### Step 1: Create the Response DTO
```java
// src/main/java/com/example/product/dto/response/ProductResponse.java
package com.example.product.dto.response;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductResponse {
    private Long id;
    private String name;
    private Double price;
}
```

### Step 2: Create the Request DTO
```java
// src/main/java/com/example/product/dto/request/ProductCreateRequest.java
package com.example.product.dto.request;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductCreateRequest {
    private String name;
    private Double price;
}
```

### Step 3: Manual Mapping in Service
```java
// In ProductService.java
public ProductResponse createProduct(ProductCreateRequest request) {
    // DTO -> Entity
    Product product = Product.builder()
            .name(request.getName())
            .price(request.getPrice())
            .build();
    
    Product saved = productRepository.save(product);
    
    // Entity -> DTO
    return ProductResponse.builder()
            .id(saved.getId())
            .name(saved.getName())
            .price(saved.getPrice())
            .build();
}
```

## 🧠 Quiz
1. What does DTO stand for?
2. Name two reasons why we shouldn't expose Entities directly.
3. In the restaurant analogy, what represents the DTO?

---