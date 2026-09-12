# 📘 Phase 2, Lesson 12: Global Exception Handling

## 🎯 Learning Goal
- ✅ Stop returning ugly, default Spring error pages.
- ✅ Create a centralized `@RestControllerAdvice` to catch all errors.
- ✅ Return clean, consistent JSON error responses to the client.

---

## 💡 The Concept: The Safety Net
Right now, if a product isn't found, our app throws a `RuntimeException`, and Spring returns a massive, ugly JSON stack trace. 
If validation fails, it returns a complex, hard-to-read error structure.

We need a **Safety Net** that catches *all* exceptions from *all* controllers and converts them into a clean, predictable JSON format. We do this with `@RestControllerAdvice`.

---

## 🛠️ Step-by-Step Build

### Step 1: Create Custom Exceptions
Create a new package `exception`. Add `ProductNotFoundException.java`:
```java
// src/main/java/com/example/demo/exception/ProductNotFoundException.java
package com.example.demo.exception;

public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(Long id) {
        super("Product not found with id: " + id);
    }
}
```

### Step 2: Update the Service to Throw It
Update `ProductService.java`:
```java
import com.example.demo.exception.ProductNotFoundException;

// ... inside getProductById ...
public ProductResponse getProductById(Long id) {
    Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id)); // <-- Use custom exception
    return productMapper.toResponse(product);
}
```

### Step 3: Create the Global Exception Handler
Create `GlobalExceptionHandler.java` in the `exception` package:
```java
// src/main/java/com/example/demo/exception/GlobalExceptionHandler.java
package com.example.demo.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice // Tells Spring: "This class handles exceptions for ALL controllers"
public class GlobalExceptionHandler {

    // 1. Handle "Product Not Found" -> 404
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(ProductNotFoundException ex) {
        Map<String, Object> error = new HashMap<>();
        error.put("timestamp", LocalDateTime.now());
        error.put("status", HttpStatus.NOT_FOUND.value());
        error.put("error", "Not Found");
        error.put("message", ex.getMessage());
        
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
 }

    // 2. Handle Validation Errors -> 400
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, Object> error = new HashMap<>();
        error.put("timestamp", LocalDateTime.now());
        error.put("status", HttpStatus.BAD_REQUEST.value());
        error.put("error", "Validation Failed");
        
        // Collect all field-specific errors into a neat map
        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(err -> 
            fieldErrors.put(err.getField(), err.getDefaultMessage())
        );
        error.put("details", fieldErrors);
        
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    // 3. Handle EVERYTHING ELSE -> 500
    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGeneric(Exception ex) {
        Map<String, Object> error = new HashMap<>();
        error.put("timestamp", LocalDateTime.now());
        error.put("status", HttpStatus.INTERNAL_SERVER_ERROR.value());
        error.put("error", "Internal Server Error");
        error.put("message", "An unexpected error occurred");
        
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

---

## 🚀 Run and Test

**Test 1: Trigger 404**
```bash
curl http://localhost:8081/api/products/999
```
*Expected:* Clean JSON: `{"timestamp":"...", "status":404, "error":"Not Found", "message":"Product not found with id: 999"}`

**Test 2: Trigger 400 Validation**
```bash
curl -X POST http://localhost:8081/api/products \
-H "Content-Type: application/json" \
-d '{"name": "a", "price": -10}'
```
*Expected:* Clean JSON with `"status": 400` and a `"details"` object showing exactly which fields failed and why.

---

## 🛠️ Exercise
1. Create a new custom exception: `DuplicateProductException` with the message "A product with this name already exists".
2. Add a handler in `GlobalExceptionHandler` for this exception that returns `409 Conflict`.
3. (Optional) Add logic in `ProductService` to check if a product name exists before saving, and throw this exception if it does.

---

## 🧠 Quiz
1. What annotation makes a class a global exception handler for all controllers?
2. Why is it better to throw a custom `ProductNotFoundException` instead of a generic `RuntimeException`?
3. What HTTP status code is most appropriate for a validation error?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Lesson 13.