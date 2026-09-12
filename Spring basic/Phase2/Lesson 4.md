# 📘 Phase 2, Lesson 4: Dependency Injection (Connecting the Layers)

## 🎯 Learning Goal
- ✅ Understand what "Dependency Injection" (DI) actually means.
- ✅ Connect the Controller (Waiter) to the Service (Kitchen).
- ✅ Learn why we use **Constructor Injection**.

---

## 💡 The Concept: Dependency Injection
Right now, we have a `ProductController` and a `ProductService`. They don't know about each other. 

How does the Waiter give the order to the Kitchen? 
In normal Java, you would do this: `ProductService service = new ProductService();`. 
**But in Spring Boot, we NEVER use the `new` keyword.** 

Instead, we say: *"Spring, please create the Service and hand it to the Controller."* This hand-off is called **Dependency Injection**.

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Product Controller
Create a new class `ProductController.java` in the `controller` package:

```java
// src/main/java/com/example/demo/controller/ProductController.java
package com.example.demo.controller;

import com.example.demo.model.Product;
import com.example.demo.service.ProductService;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/products") // All endpoints here start with /api/products
public class ProductController {

    // 1. Declare the dependency as a 'final' variable
    private final ProductService productService;

    // 2. Constructor Injection: Spring sees this and automatically passes the Service in!
    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    // GET /api/products
    @GetMapping
    public List<Product> getAllProducts() {
        // The controller just asks the service for the data
        return productService.getAllProducts();
    }
}
```

**Line-by-line:**
- `private final ProductService productService`: We make it `final` because once the Kitchen is assigned to the Waiter, it never changes.
- `public ProductController(ProductService productService)`: This is the **Constructor**. When Spring starts, it sees this constructor, realizes it needs a `ProductService`, finds the one it created (because of `@Service`), and injects it here.

### Step 2: Add a POST Endpoint
Let's allow clients to add products. Add this to your `ProductController`:

```java
    // POST /api/products
    @PostMapping
    public Product addProduct(@RequestParam String name, @RequestParam Double price) {
        // The controller takes the raw data and gives it to the service to handle
        return productService.addProduct(name, price);
    }
```

---

## 🚀 Run and Test

Restart your app.

**Test 1: Get all products (Empty list)**
```bash
curl http://localhost:8081/api/products
```
*Expected:* `[]`

**Test 2: Add a product**
```bash
curl -X POST "http://localhost:8081/api/products?name=Laptop&price=999.99"
```
*Expected:* `{"id":1,"name":"Laptop","price":999.99}`

**Test 3: Get all products again**
```bash
curl http://localhost:8081/api/products
```
*Expected:* `[{"id":1,"name":"Laptop","price":999.99}]`

---

## 🚨 Common Errors
| Error | Cause | Fix |
|---|---|---|
| `No qualifying bean of type 'ProductService' available` | Spring couldn't find the Service to inject. | Ensure `ProductService` has the `@Service` annotation and is in a sub-package of your main class. |

---

## 🛠️ Exercise
1. Add a `@GetMapping("/{id}")` endpoint in the Controller to get a product by ID.
2. It should call the `getProductById` method you wrote in the Service (Lesson 3 Exercise).
3. Test it with `curl`.

---

## 🧠 Quiz
1. What is "Dependency Injection" in simple terms?
2. Why do we make the injected service variable `final`?
3. Why don't we use `new ProductService()` inside the Controller?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 5.