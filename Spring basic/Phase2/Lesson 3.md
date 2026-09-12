# 📘 Phase 2, Lesson 3: The Service Layer (Moving Logic Out)

## 🎯 Learning Goal
- ✅ Understand why Controllers shouldn't do "business logic".
- ✅ Create your first `@Service` class.
- ✅ Use an in-memory list to store data temporarily.

---

## 💡 The Concept: The Restaurant Analogy
In Lesson 1, we said the Controller is the **Waiter**. 
The Waiter's job is to take the order and bring the food. The Waiter **does not cook the food**. The **Kitchen** cooks the food.

In Spring Boot:
- **Controller** = The Waiter (Handles HTTP requests).
- **Service** = The Kitchen (Handles business logic, calculations, database calls).

If you put all your code in the Controller, it becomes messy and impossible to test. We must separate them!

---

## 🛠️ Step-by-Step Build

### Step 1: Create a Simple Data Model
Before we build the kitchen, let's define what a "Product" is. 
Create a new package called `model` and add this file:

```java
// src/main/java/com/example/demo/model/Product.java
package com.example.demo.model;

// A Java Record is a modern, simple way to create a class that just holds data.
public record Product(Long id, String name, Double price) {}
```

### Step 2: Create the Service (The Kitchen)
Create a new package called `service`. Inside it, create `ProductService.java`:

```java
// src/main/java/com/example/demo/service/ProductService.java
package com.example.demo.service;

import com.example.demo.model.Product;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;

@Service // Tells Spring: "Manage this class as a Bean (Kitchen)"
public class ProductService {

    // A temporary in-memory list to hold our products
    private final List<Product> products = new ArrayList<>();

    // Method to get all products
    public List<Product> getAllProducts() {
        return products;
    }

    // Method to add a new product
    public Product addProduct(String name, Double price) {
        // We just use the list size as a fake ID for now
        Long newId = (long) (products.size() + 1); 
        Product newProduct = new Product(newId, name, price);
        
        products.add(newProduct); // Save to the list
        return newProduct;
    }
}
```

**Line-by-line:**
- `@Service`: This is the magic tag. It tells Spring to create an instance of this class and keep it in memory.
- `List<Product> products`: We are just using a standard Java ArrayList. Every time you restart the app, this list will be empty again. (We will fix this with a real database in Lesson 7!).

---

## 🚀 Run and Test
Just restart your app to make sure there are no compilation errors. We will connect this to the Controller in Lesson 4!

---

## 🛠️ Exercise
1. Add a new method to `ProductService` called `getProductById(Long id)`.
2. It should loop through the `products` list and return the matching product.
3. If not found, return `null`.

---

## 🧠 Quiz
1. In the restaurant analogy, what does the `@Service` represent?
2. What annotation tells Spring to manage a class as a "Kitchen" bean?
3. Why is it bad to put database logic or complex calculations directly inside the `@RestController`?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 4.