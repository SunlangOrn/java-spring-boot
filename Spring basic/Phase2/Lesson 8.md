# 📘 Phase 2, Lesson 8: Global Exception Handling

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem](#-the-problem)
- [The Solution](#-the-solution)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Create a custom exception for "not found" scenarios
- ✅ Build a Global Exception Handler using `@RestControllerAdvice`
- ✅ Return clean, professional JSON error responses
- ✅ Handle validation errors with proper field-level messages

---

## 🚨 The Problem

### Problem 1: Ugly 404 Errors
Right now, if a product isn't found, your service returns `null`, and the client gets an empty response or a crash.

### Problem 2: Ugly Validation Errors
When `@Valid` fails, Spring returns a massive, ugly JSON with stack traces. Clients can't read it.

### What We Want Instead
A clean, professional error response:
```json
{
  "timestamp": "2026-09-08T12:00:00",
  "status": 404,
  "error": "Not Found",
  "message": "Product not found with id: 999"
}
```

---

## 💡 The Solution

We use **`@RestControllerAdvice`**. Think of it as a **safety net** that catches all exceptions from all controllers and converts them into clean JSON responses.

```text
Controller throws exception
        ↓
@RestControllerAdvice catches it
        ↓
Converts to clean JSON
        ↓
Sends to client
```

---

## 🛠️ Step-by-Step Build

### Step 1: Create a Custom Exception
```java
// src/main/java/com/example/product/exception/ProductNotFoundException.java
package com.example.product.exception;

// Extends RuntimeException so we don't need to declare it with "throws"
public class ProductNotFoundException extends RuntimeException {
    
    public ProductNotFoundException(Long id) {
        super("Product not found with id: " + id);
        // super() passes the message to the parent RuntimeException class
    }
}
```

**Why create a custom exception?**
- `RuntimeException` is too generic. You don't know what went wrong.
- `ProductNotFoundException` tells you exactly what happened.
- You can handle it differently from other errors (404 vs 500).

### Step 2: Throw It in the Service
```java
// In ProductService.java
public ProductResponse getProductById(Long id) {
    Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
            // ↑ If not found, throw the custom exception
    return productMapper.toResponse(product);
}
```

**How `orElseThrow` works:**
- `findById(id)` returns an `Optional<Product>`
- If the product exists → returns it
- If the product doesn't exist → runs the lambda `() -> new ProductNotFoundException(id)`

### Step 3: Create the Global Exception Handler
```java
// src/main/java/com/example/product/exception/GlobalExceptionHandler.java
package com.example.product.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice  // "Catch exceptions from ALL controllers"
public class GlobalExceptionHandler {

    // ─── Handler 1: Product Not Found → 404 ───
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(ProductNotFoundException ex) {
        
        Map<String, Object> body = new HashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("status", HttpStatus.NOT_FOUND.value());    // 404
        body.put("error", "Not Found");
        body.put("message", ex.getMessage());                // "Product not found with id: 999"
        
        return new ResponseEntity<>(body, HttpStatus.NOT_FOUND);
    }

    // ─── Handler 2: Validation Errors → 400 ───
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(MethodArgumentNotValidException ex) {
        
        Map<String, Object> body = new HashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("status", HttpStatus.BAD_REQUEST.value());  // 400
        body.put("error", "Validation Failed");
        
        // Collect all field-level errors
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> {
            fieldErrors.put(error.getField(), error.getDefaultMessage());
            // Example: "name" → "Product name is required"
        });
        body.put("errors", fieldErrors);
        
        return new ResponseEntity<>(body, HttpStatus.BAD_REQUEST);
    }

    // ─── Handler 3: Everything Else → 500 ───
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGeneric(Exception ex) {
        
        Map<String, Object> body = new HashMap<>();
        body.put("timestamp", LocalDateTime.now());
        body.put("status", HttpStatus.INTERNAL_SERVER_ERROR.value()); // 500
        body.put("error", "Internal Server Error");
        body.put("message", "An unexpected error occurred");
        
        return new ResponseEntity<>(body, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

**Line-by-line explanation:**

1. `@RestControllerAdvice` = "This class handles exceptions for ALL controllers in the app"
2. `@ExceptionHandler(ProductNotFoundException.class)` = "When this specific exception is thrown, run this method"
3. `Map<String, Object> body` = We build the JSON response manually using a Map
4. `ex.getMessage()` = Gets the message we passed in the exception constructor
5. `ex.getBindingResult().getFieldErrors()` = Gets all validation errors from `@Valid`
6. `error.getField()` = The field name (e.g., "name", "price")
7. `error.getDefaultMessage()` = The message from the annotation (e.g., "Product name is required")

---

## 🚀 Run and Test

### Test 1: Product Not Found (404)
```bash
curl http://localhost:8080/api/products/999
```
**Expected:**
```json
{
  "timestamp": "2026-09-08T12:00:00",
  "status": 404,
  "error": "Not Found",
  "message": "Product not found with id: 999"
}
```

### Test 2: Validation Error (400)
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "", "price": -50, "stock": -10}'
```
**Expected:**
```json
{
  "timestamp": "2026-09-08T12:00:00",
  "status": 400,
  "error": "Validation Failed",
  "errors": {
    "name": "Product name is required",
    "price": "Price must be greater than zero",
    "stock": "Stock cannot be negative"
  }
}
```

### Test 3: Valid Request (201)
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "Keyboard", "price": 79.99, "stock": 25}'
```
**Expected:** `201 Created` with product JSON.

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| Exception returns 500 instead of 404 | Handler not found by Spring | Ensure `GlobalExceptionHandler` is in a scanned package |
| Validation still shows ugly errors | Missing `MethodArgumentNotValidException` handler | Add Handler 2 from above |
| `@RestControllerAdvice` vs `@ControllerAdvice` | Confusion | Use `@RestControllerAdvice` for REST APIs (it includes `@ResponseBody`) |

---

## 🛠️ Exercise
1. Create a new custom exception: `DuplicateProductException` with message "Product with this name already exists".
2. In the service `createProduct` method, check if a product with the same name already exists (add a `findByName` method to the repository).
3. If it exists, throw `DuplicateProductException`.
4. Add a handler in `GlobalExceptionHandler` that returns `409 Conflict` for this exception.
5. Test by creating two products with the same name.

---

## 🧠 Quiz
1. What does `@RestControllerAdvice` do?
2. Why is it better to throw `ProductNotFoundException` instead of a generic `RuntimeException`?
3. What HTTP status code should you return for validation errors?

---

## 🎉 Phase 2 Progress
You now have a production-ready Spring Boot API with:
- ✅ PostgreSQL database
- ✅ Flyway migrations
- ✅ JPA entities
- ✅ DTOs with MapStruct
- ✅ Bean Validation
- ✅ Global Exception Handling

## 🛑 STOP
Reply with your exercise code and quiz answers. Next, we will complete the CRUD with **Transactions, Update, Delete, Pagination**, and **Testing**!