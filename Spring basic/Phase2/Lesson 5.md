# 📘 Phase 2, Lesson 5: Accepting JSON Data (`@RequestBody`)

## 🎯 Learning Goal
- ✅ Understand the difference between `@RequestParam` and `@RequestBody`.
- ✅ Learn how to receive complex JSON data from the client.
- ✅ Convert JSON into a Java Object automatically.

---

## 💡 The Concept: URL Params vs. JSON Body
In Lesson 4, we created a product like this:
`POST /api/products?name=Laptop&price=999.99`

This uses **Query Parameters** (`@RequestParam`). It's fine for 2 or 3 simple fields. 
But what if a product has a name, price, description, stock, category, and images? The URL would be massive!

In the real world, we send complex data in the **Body** of the HTTP request as **JSON**.

---

## 🛠️ Step-by-Step Build

### Step 1: Create a Request Object
We need a class to hold the incoming JSON data. Create a new package called `dto` (Data Transfer Object) and add this:

```java
// src/main/java/com/example/demo/dto/ProductRequest.java
package com.example.demo.dto;

// We use a record again because it's just holding data coming from the client
public record ProductRequest(String name, Double price) {}
```

### Step 2: Update the Service
Update `ProductService.java` to accept the DTO instead of raw strings:

```java
// In ProductService.java
import com.example.demo.dto.ProductRequest;

// ... inside the class ...

public Product addProduct(ProductRequest request) {
    Long newId = (long) (products.size() + 1); 
    
    // Extract data from the request object
    Product newProduct = new Product(newId, request.name(), request.price());
    
    products.add(newProduct);
    return newProduct;
}
```

### Step 3: Update the Controller to use `@RequestBody`
Update `ProductController.java`:

```java
// In ProductController.java
import com.example.demo.dto.ProductRequest;

// ... inside the class ...

@PostMapping
public Product addProduct(@RequestBody ProductRequest request) {
    // @RequestBody tells Spring: 
    // "Look at the raw JSON in the HTTP body, convert it to a ProductRequest object, and give it to me."
    return productService.addProduct(request);
}
```

---

## 🚀 Run and Test

Restart your app.

**Test: Add a product using JSON**
```bash
curl -X POST http://localhost:8081/api/products \
-H "Content-Type: application/json" \
-d '{
  "name": "Wireless Mouse",
  "price": 25.50
}'
```

*Explanation of the curl command:*
- `-X POST`: We are sending a POST request.
- `-H "Content-Type: application/json"`: We are telling the server, "Hey, I am sending you JSON data."
- `-d '{...}'`: This is the actual JSON body.

**Expected Result:**
```json
{"id":1,"name":"Wireless Mouse","price":25.5}
```

---

## 🚨 Common Errors
| Error | Cause | Fix |
|---|---|---|
| `400 Bad Request` | The JSON you sent doesn't match the Java object. | Check your spelling in the JSON. It must exactly match `ProductRequest(String name, Double price)`. |
| `Missing required request body` | You forgot the `-d` flag in curl, or forgot `-H "Content-Type: application/json"`. | Ensure both flags are present in your curl command. |

---

## 🛠️ Exercise
1. Add a `description` field to the `Product` record.
2. Add a `description` field to the `ProductRequest` record.
3. Update the `ProductService` to pass the description into the new Product.
4. Test it by sending a JSON body with a description.

---

## 🧠 Quiz
1. What is the difference between `@RequestParam` and `@RequestBody`?
2. What does the `-H "Content-Type: application/json"` flag do in cURL?
3. Why do we use a `ProductRequest` DTO instead of just passing the `Product` model directly from the client? *(Hint: Think about the `id` field!)*

---

## 🛑 STOP
Reply with your exercise code and quiz answers. 

Once you complete this, you have built a fully functional, in-memory REST API! 
Reply **"Next"** to get **Lessons 6, 7, 8, and 9**, where we will finally connect this to a **Real PostgreSQL Database using Docker**!