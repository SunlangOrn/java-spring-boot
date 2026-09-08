# 📘 Phase 2, Lesson 7: Bean Validation (The Bouncer)

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem](#-the-problem)
- [The Bouncer Analogy](#-the-bouncer-analogy)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand why we must validate incoming data
- ✅ Learn the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`
- ✅ Add validation rules to a DTO
- ✅ Trigger validation in the Controller using `@Valid`

---

## 🚨 The Problem

Right now, your API accepts **anything**:
```json
{"name": "", "price": -50, "stock": -10}
```
This saves a product with no name, negative price, and negative stock to your database!

---

## 🛡️ The Bouncer Analogy

Your app is a **VIP Nightclub**:
- **Database** = VIP lounge
- **Service** = Bartender
- **Controller** = Entrance door
- **Bean Validation** = **The Bouncer**

The Bouncer checks IDs at the door. If you don't meet the rules, you **never** get inside.

---

## 🛠️ Step-by-Step Build

### Step 1: Ensure Dependency Exists
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### Step 2: Add Rules to the Request DTO
```java
// src/main/java/com/example/product/dto/request/ProductCreateRequest.java
package com.example.product.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import jakarta.validation.constraints.PositiveOrZero;
import jakarta.validation.constraints.Size;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductCreateRequest {

    @NotBlank(message = "Product name is required")
    @Size(min = 3, max = 100, message = "Name must be 3-100 characters")
    private String name;

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be greater than zero")
    private Double price;

    @NotNull(message = "Stock is required")
    @PositiveOrZero(message = "Stock cannot be negative")
    private Integer stock;
}
```

### 🧠 The 3 String Validators (Very Important!)

| Annotation | `null` | `""` | `"   "` | `"abc"` |
|---|---|---|---|---|
| `@NotNull` | ❌ | ✅ | ✅ | ✅ |
| `@NotEmpty` | ❌ | ❌ | ✅ | ✅ |
| `@NotBlank` | ❌ | ❌ | ❌ | ✅ |

**Always use `@NotBlank` for text fields like names and emails!**

### Step 3: Trigger the Bouncer with `@Valid`
```java
// In ProductController.java
import jakarta.validation.Valid;

@PostMapping
public ResponseEntity<ProductResponse> createProduct(
        @Valid @RequestBody ProductCreateRequest request) {  // ← @Valid here!
    return ResponseEntity.status(HttpStatus.CREATED)
            .body(productService.createProduct(request));
}
```

**How it works:**
1. Client sends JSON
2. Spring converts JSON → `ProductCreateRequest`
3. `@Valid` triggers the Bouncer
4. **If valid:** Method runs normally
5. **If invalid:** Method **never runs**. Spring returns `400 Bad Request` immediately.

---

## 🚀 Run and Test

**Test 1: Bad data**
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "", "price": -50, "stock": -10}'
```
Expected: `400 Bad Request` (the Bouncer blocked it!)

**Test 2: Good data**
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "Keyboard", "price": 79.99, "stock": 25}'
```
Expected: `201 Created`

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| Validation not working | Missing `@Valid` in controller | Add `@Valid` before `@RequestBody` |
| Wrong import | Using `javax.validation` | Use `jakarta.validation` (Spring Boot 3+) |
| `@NotBlank` allows spaces | Using `@NotNull` instead | Use `@NotBlank` for strings |

---

## 🛠️ Exercise
1. Add a `description` field to `ProductCreateRequest`. It's optional (can be null), but if provided, max 500 characters.
2. Add a `sku` field (e.g., "ABC-1234"). It must be exactly 8 characters.
3. Test with invalid data to verify the Bouncer blocks it.

---

## 🧠 Quiz
1. What is the exact difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?
2. If `@Valid` fails, does the code inside the controller method still execute?
3. What annotation triggers validation in the controller?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 8.