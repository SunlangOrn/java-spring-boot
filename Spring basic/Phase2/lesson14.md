# 📘 Phase 2, Lesson 14: Standardized API Responses (The Enterprise Wrapper)

## 🎯 Learning Goal
- ✅ Understand why returning raw data (like a `List` or a single `Object`) is bad for enterprise APIs.
- ✅ Create a generic `HttpBodyResponse<T>` wrapper.
- ✅ Create a `BaseRestController` to keep controller code clean.

---

## 💡 The Concept: The Envelope
Right now, our API returns raw data:
```json
{"id": 1, "name": "Laptop", "price": 999.99}
```
If an error happens, Spring returns a completely different JSON structure. This forces the frontend developer to write messy `if/else` logic to figure out if the response is data or an error.

**The Solution:** Wrap *every* response in a standard "envelope":
```json
{
  "status": 200,
  "message": "Success",
  "data": {
    "id": 1,
    "name": "Laptop",
    "price": 999.99
  }
}
```

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Generic Response Wrapper
Create a new package `common`. Add `HttpBodyResponse.java`:

```java
// src/main/java/com/example/demo/common/HttpBodyResponse.java
package com.example.demo.common;

import com.fasterxml.jackson.annotation.JsonInclude;
import lombok.Builder;
import lombok.Getter;

@Getter
@Builder
@JsonInclude(JsonInclude.Include.NON_NULL) // Hides "data" if it's null (e.g., for DELETE)
public class HttpBodyResponse<T> {
    private int status;
    private String message;
    private T data; // The Generic Box! Can hold ANY type.

    public static <T> HttpBodyResponse<T> succeed(T data) {
        return HttpBodyResponse.<T>builder()
                .status(200)
                .message("Success")
                .data(data)
                .build();
    }

    public static <T> HttpBodyResponse<T> created(T data) {
        return HttpBodyResponse.<T>builder()
                .status(201)
                .message("Created Successfully")
                .data(data)
                .build();
    }
}
```

### Step 2: Create the Base Controller
Add `BaseRestController.java` to the `common` package:

```java
// src/main/java/com/example/demo/common/BaseRestController.java
package com.example.demo.common;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

public abstract class BaseRestController {
    
    protected <T> ResponseEntity<HttpBodyResponse<T>> responseSucceed(T data) {
        return ResponseEntity.ok(HttpBodyResponse.succeed(data));
    }

    protected <T> ResponseEntity<HttpBodyResponse<T>> responseCreated(T data) {
        return ResponseEntity.status(HttpStatus.CREATED).body(HttpBodyResponse.created(data));
    }
    
    protected ResponseEntity<HttpBodyResponse<Void>> responseDeleted() {
        return ResponseEntity.ok(HttpBodyResponse.<Void>builder()
                .status(200)
                .message("Deleted Successfully")
                .build());
    }
}
```

### Step 3: Update the Controller
Make `ProductController` extend `BaseRestController` and use the helper methods:

```java
// src/main/java/com/example/demo/controller/ProductController.java
package com.example.demo.controller;

import com.example.demo.common.BaseRestController;
import com.example.demo.common.HttpBodyResponse;
import com.example.demo.dto.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import com.example.demo.service.ProductService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/products") // Added /v1 for API versioning!
@RequiredArgsConstructor
public class ProductController extends BaseRestController {

    private final ProductService productService;

    @GetMapping
    public ResponseEntity<HttpBodyResponse<java.util.List<ProductResponse>>> getAllProducts() {
        return responseSucceed(productService.getAllProducts());
    }

    @GetMapping("/{id}")
    public ResponseEntity<HttpBodyResponse<ProductResponse>> getProductById(@PathVariable Long id) {
        return responseSucceed(productService.getProductById(id));
    }

    @PostMapping
    public ResponseEntity<HttpBodyResponse<ProductResponse>> addProduct(@Valid @RequestBody ProductRequest request) {
        return responseCreated(productService.addProduct(request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<HttpBodyResponse<Void>> deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
        return responseDeleted();
    }
}
```

---

## 🚀 Run and Test
Restart the app and run:
```bash
curl http://localhost:8081/api/v1/products
```
**Expected:** A beautifully wrapped JSON response with `status`, `message`, and `data`.

---

## 🛠️ Exercise
1. Add a `@PutMapping("/{id}")` to the controller.
2. Use `responseSucceed()` to return the updated product.
3. Test it and verify the wrapper is present.

---

## 🧠 Quiz
1. Why do we use `<T>` (Generics) in `HttpBodyResponse<T>` instead of `Object`?
2. What does `@JsonInclude(JsonInclude.Include.NON_NULL)` do?
3. Why is it helpful to have a `BaseRestController`?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Lesson 15.