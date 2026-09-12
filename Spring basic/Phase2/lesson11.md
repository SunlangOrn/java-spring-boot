# 📘 Phase 2, Lesson 11: Bean Validation (The Bouncer)

## 🎯 Learning Goal
- ✅ Understand why we must validate data *before* it reaches the Service layer.
- ✅ Learn the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`.
- ✅ Use `@Valid` in the Controller to trigger validation.

---

## 💡 The Concept: The Bouncer
Imagine your app is a VIP club. The Controller is the door. The Service is the bartender. 
If a client sends `{"name": "", "price": -50}`, we don't want that bad data reaching the database. 
We put a "Bouncer" at the door using **Bean Validation** annotations. If the data is bad, the Bouncer stops it immediately and returns a `400 Bad Request`.

---

## 🛠️ Step-by-Step Build

### Step 1: Ensure Validation Dependency
Make sure this is in your `pom.xml` (it's included in `spring-boot-starter-web`, but good to be explicit):
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### Step 2: Add Validation Annotations to the Request DTO
Update `ProductRequest.java`:
```java
// src/main/java/com/example/demo/dto/ProductRequest.java
package com.example.demo.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import jakarta.validation.constraints.Size;

public record ProductRequest(
    
    @NotBlank(message = "Product name is required and cannot be empty")
    @Size(min = 3, max = 100, message = "Name must be between 3 and 100 characters")
    String name,

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be greater than zero")
    Double price,

    @Size(max = 500, message = "Description cannot exceed 500 characters")
    String description
) {}
```

**Crucial Difference for Strings:**
- `@NotNull`: Cannot be `null`, but CAN be `""` or `"   "`.
- `@NotEmpty`: Cannot be `null` or `""`, but CAN be `"   "`.
- `@NotBlank`: Cannot be `null`, `""`, or `"   "`. **Always use this for names/text!**

### Step 3: Trigger the Bouncer in the Controller
Add `@Valid` to the `@RequestBody` parameter in `ProductController.java`:

```java
import jakarta.validation.Valid; // <-- Add this import

// ... inside ProductController ...

@PostMapping
public ProductResponse addProduct(@Valid @RequestBody ProductRequest request) { 
    // ^^^ @Valid tells Spring: "Check the annotations in ProductRequest before running this method!"
    return productService.addProduct(request);
}
```

---

## 🚀 Run and Test

**Test 1: Send BAD data**
```bash
curl -X POST http://localhost:8081/api/products \
-H "Content-Type: application/json" \
-d '{"name": "a", "price": -10}'
```
*Expected:* `400 Bad Request`. Spring will return a detailed error message about the `@Size` and `@Positive` violations.

**Test 2: Send GOOD data**
```bash
curl -X POST http://localhost:8081/api/products \
-H "Content-Type: application/json" \
-d '{"name": "Laptop", "price": 999.99, "description": "Great computer"}'
```
*Expected:* `200 OK` with the created product.

---

## 🚨 Common Errors
| Error | Cause | Fix |
|---|---|---|
| Validation is ignored | Forgot `@Valid` in the Controller. | Add `@Valid` before `@RequestBody`. |
| `javax.validation` not found | Using old Java EE imports. | Ensure you import from `jakarta.validation.*` (Spring Boot 3+). |

---

## 🛠️ Exercise
1. Add a `private String sku;` field to `ProductRequest`.
2. Add a validation rule that requires the `sku` to be exactly 8 characters long. *(Hint: Look up how to set `min` and `max` in `@Size` to the same number).*
3. Test it with a 7-character SKU and an 8-character SKU.

---

## 🧠 Quiz
1. What is the exact difference between `@NotNull` and `@NotBlank` for a String?
2. If `@Valid` fails, does the code inside the `addProduct` method in the Controller still execute?
3. What HTTP status code does Spring return when validation fails?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Lesson 12.