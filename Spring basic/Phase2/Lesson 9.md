# 📘 Phase 2, Lesson 9: Bean Validation (The Bouncer)

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem: Why do we need validation?](#-the-problem-why-do-we-need-validation)
- [The Solution: The "Bouncer" Analogy](#-the-solution-the-bouncer-analogy)
- [Step 1: Add the Validation Tools](#-step-1-add-the-validation-tools)
- [Step 2: Create the Rules (Annotations)](#-step-2-create-the-rules-annotations)
- [Step 3: Trigger the Bouncer (`@Valid`)](#-step-3-trigger-the-bouncer-valid)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
By the end of this lesson, you will:
- ✅ Understand why we must validate data before saving it.
- ✅ Learn the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`.
- ✅ Know how to add validation rules to a DTO.
- ✅ Know how to trigger validation in the Controller using `@Valid`.

---

## 🚨 The Problem: Why do we need validation?

Right now, your API accepts **any** data the client sends. 

If a client sends this JSON to create a product:
```json
{
  "name": "",
  "price": -50.00,
  "stock": -10
}
```
Your API will happily save it to the database! 
- You now have a product with no name.
- A product that *pays* the customer to take it (negative price).
- A product with negative inventory.

This will break your application, confuse users, and ruin your database. We need to stop bad data **before** it reaches the database.

---

## 🛡️ The Solution: The "Bouncer" Analogy

Imagine your application is a **VIP Nightclub**.
- **The Database** is the VIP lounge.
- **The Service Layer** is the bartender.
- **The Controller** is the entrance door.
- **Bean Validation** is the **Bouncer** at the door.

The Bouncer has a list of rules (e.g., "Must be 18+", "Must wear a shirt"). 
If a person (data) doesn't meet the rules, the Bouncer stops them at the door. They **never** get inside to bother the bartender or sit in the VIP lounge.

In Spring Boot, we write the rules using **Annotations** (like `@NotBlank`), and the Bouncer checks them automatically.

---

## 🛠️ Step 1: Add the Validation Tools

First, we need to give Spring Boot the "Bouncer" tools. 

Open your `pom.xml` and ensure this dependency is there:

```xml
<dependencies>
    <!-- ... your other dependencies ... -->

    <!-- The Bouncer Tools -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
</dependencies>
```
*Note: If you are using `spring-boot-starter-web`, this is usually already included, but it's good practice to add it explicitly so you don't forget.*

---

## 📝 Step 2: Create the Rules (Annotations)

Now, let's go to our Request DTO (`ProductCreateRequest.java`) and write the rules for the Bouncer.

Open `src/main/java/com/example/product/dto/request/ProductCreateRequest.java` and update it:

```java
package com.example.product.dto.request;

// 1. Import the validation rules
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;
import jakarta.validation.constraints.Size;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class ProductCreateRequest {

    // RULE 1: Name cannot be null, empty, or just spaces.
    // RULE 2: Name must be between 3 and 100 characters long.
    @NotBlank(message = "Product name is required and cannot be empty")
    @Size(min = 3, max = 100, message = "Product name must be between 3 and 100 characters")
    private String name;

    // RULE 3: Price cannot be null.
    // RULE 4: Price must be strictly greater than 0.
    @NotNull(message = "Price is required")
    @Positive(message = "Price must be greater than zero")
    private Double price;

    // RULE 5: Stock cannot be null.
    // RULE 6: Stock must be 0 or greater (we allow 0 for "out of stock").
    @NotNull(message = "Stock is required")
    @PositiveOrZero(message = "Stock cannot be negative")
    private Integer stock;
}
```

### 🧠 Let's explain the rules line-by-line:

**1. `@NotBlank` vs `@NotNull` vs `@NotEmpty` (Very Important!)**
Beginners always confuse these. Here is the exact difference for a `String`:
- `@NotNull`: The string cannot be `null`. But it **can** be `""` (empty) or `"   "` (just spaces).
- `@NotEmpty`: The string cannot be `null`, and cannot be `""`. But it **can** be `"   "` (just spaces).
- `@NotBlank`: The string cannot be `null`, cannot be `""`, and **cannot be just spaces**. It must have actual text. *Always use this for names, emails, etc.*

**2. `message = "..."`**
This is the exact text the Bouncer will shout at the client if they break the rule. If you don't write this, Spring will use a default ugly message like `"must not be blank"`. Writing custom messages makes your API professional.

**3. `@Size(min = 3, max = 100)`**
Checks the length of the string. Prevents users from sending a 1-character name or a 10,000-character name to crash your database.

**4. `@Positive` vs `@PositiveOrZero`**
- `@Positive`: The number must be `> 0` (1, 2, 3...). Good for prices.
- `@PositiveOrZero`: The number must be `>= 0` (0, 1, 2...). Good for stock/inventory, because 0 means "out of stock".

---

## 🚦 Step 3: Trigger the Bouncer (`@Valid`)

Writing the rules in the DTO is not enough. You have to tell the Controller: *"Hey, use the Bouncer before letting this data into the method!"*

We do this by adding the **`@Valid`** annotation.

Open `src/main/java/com/example/product/controller/ProductController.java`:

```java
package com.example.product.controller;

import com.example.product.dto.request.ProductCreateRequest;
import com.example.product.dto.response.ProductResponse;
import com.example.product.service.ProductService;
import jakarta.validation.Valid; // 1. Import Valid
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    // ... other endpoints ...

    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(
            @Valid @RequestBody ProductCreateRequest request) { // 2. Add @Valid here!
        
        // If the data is bad, the code NEVER reaches this line. 
        // The Bouncer stops it before.
        ProductResponse response = productService.createProduct(request);
        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }
}
```

### 🧠 How `@Valid` works internally:
1. Client sends JSON.
2. Spring converts JSON to `ProductCreateRequest`.
3. **`@Valid` triggers the Bouncer.**
4. The Bouncer checks `@NotBlank`, `@Positive`, etc.
5. **If it passes:** The code inside `createProduct()` runs normally.
6. **If it fails:** Spring immediately stops. It throws a `MethodArgumentNotValidException`. The code inside `createProduct()` **never runs**.

---

## 🚀 Run and Test

Start your application:
```bash
./mvnw spring-boot:run
```

### Test 1: Send BAD data (Trigger the Bouncer)
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{
  "name": "",
  "price": -50,
  "stock": -10
}'
```

**Expected Result:** 
You will get a `400 Bad Request`. 
*Note: The JSON body Spring returns by default is very long and ugly. **Do not worry about this!** In the very next lesson, we will learn how to catch this error and make it beautiful. For now, just be happy that the bad data was blocked and didn't reach your database!*

### Test 2: Send GOOD data
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{
  "name": "Mechanical Keyboard",
  "price": 150.00,
  "stock": 25
}'
```

**Expected Result:** 
`201 Created` with your product JSON. The Bouncer let it through!

---

## 🚨 Common Errors

1. **`@Valid` is not working, bad data is still saving.**
   - *Cause:* You forgot to add `@Valid` in the Controller method parameter.
   - *Fix:* Ensure it is exactly `@Valid @RequestBody ProductCreateRequest request`.

2. **`@NotBlank` allows empty spaces `"   "`.**
   - *Cause:* You used `@NotNull` or `@NotEmpty` instead of `@NotBlank`.
   - *Fix:* Always use `@NotBlank` for text fields like names or emails.

3. **Compilation error: `cannot find symbol: class Valid`**
   - *Cause:* Wrong import.
   - *Fix:* Make sure you import `jakarta.validation.Valid`, NOT `javax.validation.Valid` (which is for older Spring Boot versions).

---

## 🛠️ Exercise

1. Open your `ProductCreateRequest.java`.
2. Add a new field: `private String description;`
3. Add a rule to `description`: It is optional (can be null), but IF the client provides it, it cannot be longer than 500 characters. *(Hint: Look up the `@Size` annotation documentation to see how to make it allow nulls, or just use `@Size(max=500)` which allows nulls by default).*
4. Add a new field: `private String sku;` (Stock Keeping Unit, like "ABC-123").
5. Add a rule to `sku`: It must be exactly 7 characters long. *(Hint: `@Size` has `min` and `max`. How do you make them the same?)*
6. Test it with `curl` by sending invalid data to ensure the Bouncer blocks it.

---

## 🧠 Quiz

Answer these 3 questions to prove you understand the Bouncer:

1. What is the exact difference between `@NotNull`, `@NotEmpty`, and `@NotBlank` for a String?
2. If `@Valid` fails, does the code inside the Controller method (e.g., `productService.createProduct()`) still execute? Why or why not?
3. What annotation do we put in the Controller to tell Spring to run the validation checks?

---

## 🛑 STOP

**Do not move forward.** 

Reply with:
1. Confirmation that you added the validation rules, triggered `@Valid`, and saw the 400 Bad Request when sending bad data.
2. The code you wrote for the **Exercise** (the new `description` and `sku` fields with their rules).
3. Your answers to the 3 quiz questions.

Once you reply, I will review your answers. In the **next lesson**, we will fix the ugly error message and learn **Global Exception Handling** to make your API look professional!