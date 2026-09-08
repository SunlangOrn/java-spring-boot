# 📘 Phase 2, Lesson 2: Configuration & Your First REST API

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [Configuration Explained](#-configuration-explained)
- [The Restaurant Analogy](#-the-restaurant-analogy)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand `application.yml` and how to read values from it
- ✅ Build a simple REST API with GET and POST (no database yet)
- ✅ Understand the HTTP request lifecycle

---

## 💡 Configuration Explained

`application.yml` is your app's **settings file**. Instead of hardcoding values in Java, you put them here.

```yaml
server:
  port: 8080

app:
  welcome-message: "Welcome to the Product API"
```

To read a value in Java, use `@Value`:
```java
@Value("${app.welcome-message}")
private String welcomeMessage;
```

---

## 🍽️ The Restaurant Analogy

| Component | Restaurant | Spring Boot |
|---|---|---|
| Client | Customer | Browser / Postman / curl |
| Controller | Waiter | `@RestController` |
| Service | Kitchen | `@Service` |
| Repository | Pantry | `@Repository` (database) |

The **waiter** (Controller) takes the order, gives it to the **kitchen** (Service), which gets ingredients from the **pantry** (Repository). The customer never goes into the kitchen!

---

## 🛠️ Step-by-Step Build

### Step 1: Update `application.yml`
```yaml
# src/main/resources/application.yml
server:
  port: 8080

app:
  welcome-message: "Welcome to the Simple Product API"
```

### Step 2: Create a Simple Model
```java
// src/main/java/com/example/product/model/Product.java
package com.example.product.model;

// A Java record is a quick way to create a class with fields, constructor, and getters
public record Product(Long id, String name, Double price) {}
```

**What is a `record`?**
- Introduced in Java 16
- Automatically creates: constructor, getters, `toString()`, `equals()`, `hashCode()`
- Great for simple data holders
- *Note: We will change this to a regular class later when we add JPA*

### Step 3: Create the Service (Kitchen)
```java
// src/main/java/com/example/product/service/ProductService.java
package com.example.product.service;

import com.example.product.model.Product;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.atomic.AtomicLong;

@Service // Tells Spring: "Manage this class as a Bean"
public class ProductService {

    // Read value from application.yml
    @Value("${app.welcome-message}")
    private String welcomeMessage;

    // In-memory "database" (just a simple list for now)
    private final List<Product> products = new ArrayList<>();
    
    // Auto-increment ID generator
    private final AtomicLong idGenerator = new AtomicLong(1);

    public String getWelcomeMessage() {
        return welcomeMessage;
    }

    public List<Product> getAllProducts() {
        return products;
    }

    public Product addProduct(String name, Double price) {
        // Create new product with auto-generated ID
        Product newProduct = new Product(idGenerator.getAndIncrement(), name, price);
        products.add(newProduct);
        return newProduct;
    }
}
```

**Line-by-line:**
- `@Service` = Registers this class as a Spring Bean
- `@Value("${app.welcome-message}")` = Reads the value from `application.yml`
- `ArrayList<Product>` = Temporary storage (will be replaced by a real database later)
- `AtomicLong` = Thread-safe counter for generating unique IDs

### Step 4: Create the Controller (Waiter)
```java
// src/main/java/com/example/product/controller/ProductController.java
package com.example.product.controller;

import com.example.product.model.Product;
import com.example.product.service.ProductService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;
import java.util.Map;

@RestController // Handles HTTP requests, returns JSON
@RequestMapping("/api/products") // Base URL for all methods in this class
public class ProductController {

    private final ProductService productService;

    // Constructor Injection: Spring provides the ProductService automatically
    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    // GET /api/products/welcome
    @GetMapping("/welcome")
    public ResponseEntity<String> getWelcome() {
        return ResponseEntity.ok(productService.getWelcomeMessage());
    }

    // GET /api/products
    @GetMapping
    public ResponseEntity<List<Product>> getAllProducts() {
        return ResponseEntity.ok(productService.getAllProducts());
    }

    // POST /api/products
    @PostMapping
    public ResponseEntity<Product> createProduct(@RequestBody Map<String, Object> request) {
        String name = (String) request.get("name");
        Double price = Double.valueOf(request.get("price").toString());
        Product created = productService.addProduct(name, price);
        return ResponseEntity.ok(created);
    }
}
```

**Line-by-line:**
- `@RestController` = "Return data as JSON, not HTML pages"
- `@RequestMapping("/api/products")` = All URLs in this class start with `/api/products`
- `@GetMapping` = Responds to HTTP GET requests
- `@PostMapping` = Responds to HTTP POST requests
- `@RequestBody` = "Take the JSON body from the request and convert it to a Java object"
- `ResponseEntity.ok(...)` = Returns HTTP 200 OK with the data

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

**Test 1: Welcome message**
```bash
curl http://localhost:8080/api/products/welcome
```
Expected: `Welcome to the Simple Product API`

**Test 2: Get all products (empty)**
```bash
curl http://localhost:8080/api/products
```
Expected: `[]`

**Test 3: Create a product**
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "Laptop", "price": 999.99}'
```
Expected: `{"id":1,"name":"Laptop","price":999.99}`

**Test 4: Get all products (now has 1)**
```bash
curl http://localhost:8080/api/products
```
Expected: `[{"id":1,"name":"Laptop","price":999.99}]`

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| `404 Not Found` | Wrong URL | Check the URL matches `@RequestMapping` + `@GetMapping` |
| `400 Bad Request` | Invalid JSON | Check your JSON syntax in the curl command |
| `Port 8080 in use` | Another app running | Change `server.port` in `application.yml` |

---

## 🛠️ Exercise
1. Add a `GET /api/products/{id}` endpoint to get a single product by ID.
2. Use `@PathVariable Long id` to extract the ID from the URL.
3. In the service, loop through the list to find the matching product.
4. Return `404 Not Found` if the product doesn't exist.
5. Test: `curl http://localhost:8080/api/products/1`

---

## 🧠 Quiz
1. In the restaurant analogy, what Spring component is the "Kitchen"?
2. What does `@RequestBody` do?
3. If you change `server.port` to `9090`, what URL do you use?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 3.