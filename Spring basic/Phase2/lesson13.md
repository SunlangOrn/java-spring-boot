# 📘 Phase 2, Lesson 13: The Service Interface + Impl Pattern

## 🎯 Learning Goal
- ✅ Understand why enterprise applications separate Services into an Interface and an Implementation class.
- ✅ Refactor our current `ProductService` into this professional pattern.

---

## 💡 The Concept: Programming to an Interface
Right now, our Controller depends directly on `ProductService` (the concrete class). 
In large enterprise apps, we separate the **Contract** (what the service does) from the **Implementation** (how it does it). 

Why?
1. **Testing**: We can easily create a "Mock" implementation for unit tests.
2. **Flexibility**: If we want to change how products are saved (e.g., from PostgreSQL to MongoDB), we just create a new `MongoProductServiceImpl` without changing the Controller or the Interface.

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Service Interface
Create a new file `ProductService.java` in the `service` package. This will now be an **interface**, not a class.

```java
// src/main/java/com/example/demo/service/ProductService.java
package com.example.demo.service;

import com.example.demo.dto.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import java.util.List;

public interface ProductService {
    List<ProductResponse> getAllProducts();
    ProductResponse getProductById(Long id);
    ProductResponse addProduct(ProductRequest request);
    void deleteProduct(Long id);
}
```

### Step 2: Create the Service Implementation
Rename your existing `ProductService.java` class to `ProductServiceImpl.java`. 
Make it `implement` the interface you just created.

```java
// src/main/java/com/example/demo/service/impl/ProductServiceImpl.java
package com.example.demo.service.impl; // <-- Note the 'impl' package

import com.example.demo.dto.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import com.example.demo.entity.Product;
import com.example.demo.exception.ProductNotFoundException;
import com.example.demo.mapper.ProductMapper;
import com.example.demo.repository.ProductRepository;
import com.example.demo.service.ProductService; // Import the interface!
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service // Still a Spring Bean!
@RequiredArgsConstructor
public class ProductServiceImpl implements ProductService { // <-- Implement the interface

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;

    @Override // Good practice to mark overridden interface methods
    @Transactional(readOnly = true)
    public List<ProductResponse> getAllProducts() {
        return productRepository.findAll().stream()
                .map(productMapper::toResponse)
                .toList();
    }

    @Override
    @Transactional(readOnly = true)
    public ProductResponse getProductById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ProductNotFoundException(id));
        return productMapper.toResponse(product);
    }

    @Override
    @Transactional
    public ProductResponse addProduct(ProductRequest request) {
        Product newProduct = productMapper.toEntity(request);
        Product savedProduct = productRepository.save(newProduct);
        return productMapper.toResponse(savedProduct);
    }
    
    @Override
    @Transactional
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }
}
```

### Step 3: Update the Controller
The Controller **does not change at all**! 
Because the Controller depends on the `ProductService` *interface*, and Spring sees that `ProductServiceImpl` is the only class implementing it, Spring automatically injects the implementation.

```java
// src/main/java/com/example/demo/controller/ProductController.java
// ... (No changes needed! It still says: private final ProductService productService;)
```

---

## 🚀 Run and Test
Run `./mvnw clean compile` and start the app. 
Test your GET, POST, and DELETE endpoints. Everything should work exactly as before, but your architecture is now enterprise-grade!

---

## 🚨 Common Errors
| Error | Cause | Fix |
|---|---|---|
| `Field productService in controller required a bean of type 'ProductService' that could not be found` | You forgot `@Service` on the Impl class, or it's in a package not scanned by Spring. | Ensure `@Service` is on `ProductServiceImpl` and it's in a sub-package of `DemoApplication`. |

---

## 🛠️ Exercise
1. Add a new method to the `ProductService` interface: `ProductResponse updateProduct(Long id, ProductRequest request);`
2. Implement this method in `ProductServiceImpl`. It should find the product, update its fields using the request, save it, and return the mapped response.
3. Add a `@PutMapping("/{id}")` to the Controller to expose this endpoint.
4. Test it with `curl -X PUT`.

---

## 🧠 Quiz
1. What is the main benefit of separating a Service into an Interface and an Implementation class?
2. Why doesn't the Controller need to change when we rename `ProductService` to `ProductServiceImpl`?
3. What does the `@Override` annotation do, and why is it good practice to use it?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers. 

Once you complete this, you have successfully built a professional, Dockerized, database-backed Spring Boot API with proper migrations, DTOs, validation, exception handling, and enterprise architecture! 

Reply **"Next"** to get the final lessons of Phase 2 (Pagination, Testing, and Review)!