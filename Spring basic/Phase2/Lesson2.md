# 📘 Phase 2, Lesson 2: Configuration & Basic Web

## 🎯 Learning Goals
- Understand how to configure a Spring Boot app using `application.yml`.
- Build a simple REST API with `GET` and `POST` using an in-memory list.
- Understand the HTTP Request Lifecycle.

## 💡 Concepts Explained Simply
- **`application.yml`**: The central configuration file. YAML is clean and hierarchical.
- **`@RestController`**: Tells Spring this class handles web requests and returns JSON.
- **`@GetMapping` / `@PostMapping`**: Maps specific HTTP methods to Java methods.
- **The Restaurant Analogy**: 
  - **Client**: The customer.
  - **Controller**: The waiter (takes the order, checks if it's valid).
  - **Service**: The kitchen (cooks the food / business logic).
  - **Repository**: The pantry (database). *We will use a simple notepad (List) for now.*

## ⚙️ Step-by-Step Build

### Step 1: Basic Configuration
Open `src/main/resources/application.yml`:
```yaml
server:
  port: 8080

app:
  welcome-message: "Welcome to the Simple Product API"
```

### Step 2: Create a Simple Model
```java
// src/main/java/com/example/product/model/Product.java
package com.example.product.model;

public record Product(Long id, String name, Double price) {}
```

### Step 3: Create the Service (The Kitchen)
```java
// src/main/java/com/example/product/service/ProductService.java
package com.example.product.service;

import com.example.product.model.Product;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

@Service
public class ProductService {
    @Value("${app.welcome-message}")
    private String welcomeMessage;

    private final List<Product> products = new ArrayList<>();
    private final AtomicLong idGenerator = new AtomicLong(1);

    public String getWelcomeMessage() { return welcomeMessage; }
    public List<Product> getAllProducts() { return products; }

    public Product addProduct(String name, Double price) {
        Product newProduct = new Product(idGenerator.getAndIncrement(), name, price);
        products.add(newProduct);
        return newProduct;
    }
}
```

### Step 4: Create the Controller (The Waiter)
```java
// src/main/java/com/example/product/controller/ProductController.java
package com.example.product.controller;

import com.example.product.model.Product;
import com.example.product.service.ProductService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/products")
public class ProductController {
    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping("/welcome")
    public ResponseEntity<String> getWelcome() {
        return ResponseEntity.ok(productService.getWelcomeMessage());
    }

    @GetMapping
    public ResponseEntity<List<Product>> getAllProducts() {
        return ResponseEntity.ok(productService.getAllProducts());
    }

    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody Map<String, Object> request) {
        String name = (String) request.get("name");
        Double price = Double.valueOf(request.get("price").toString());
        return ResponseEntity.ok(productService.addProduct(name, price));
    }
}
```

## 🚀 Run and Test
```bash
./mvnw spring-boot:run
```
Test with curl:
```bash
curl -X GET http://localhost:8080/api/products/welcome
curl -X POST http://localhost:8080/api/products -H "Content-Type: application/json" -d '{"name": "Laptop", "price": 999.99}'
```

## 🛠️ Exercise
Add a `GET /api/products/{id}` endpoint to fetch a single product by ID.

## 🧠 Quiz
1. What Spring component represents the "Kitchen"?
2. What annotation makes a class return JSON instead of an HTML view?
3. If I change `server.port` to 9090, what URL do I use to test?

---