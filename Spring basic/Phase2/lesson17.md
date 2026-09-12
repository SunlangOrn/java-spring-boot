# 📘 Phase 2, Lesson 17: Integration Testing with MockMvc

## 🎯 Learning Goal
- ✅ Understand what an Integration Test is.
- ✅ Use `@SpringBootTest` and `MockMvc` to test the Controller layer.
- ✅ Verify HTTP status codes and JSON responses.

---

## 💡 The Concept: Integration Testing
A **Unit Test** checks if the *logic* inside a class works. 
An **Integration Test** checks if the *pieces work together*. Does the Controller correctly receive the HTTP request, validate it, pass it to the Service, and return the correct HTTP status and JSON?

`MockMvc` simulates HTTP requests (GET, POST) without actually starting a real network server, making tests fast but realistic.

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Integration Test
Create `ProductControllerTest.java` in `src/test/java/com/example/demo/controller/`:

```java
// src/test/java/com/example/demo/controller/ProductControllerTest.java
package com.example.demo.controller;

import com.example.demo.dto.ProductRequest;
import com.example.demo.dto.response.ProductResponse;
import com.example.demo.service.ProductService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

// @WebMvcTest loads ONLY the web layer (Controllers), not the whole app (faster)
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc; // Simulates HTTP requests

    @MockBean // Replaces the real Service with a Mock in the Spring Context
    private ProductService productService;

    @Autowired
    private ObjectMapper objectMapper; // Converts Java objects to JSON strings

    @Test
    void getAllProducts_Returns200AndWrappedData() throws Exception {
        // Arrange
        ProductResponse response = ProductResponse.builder().id(1L).name("Mouse").price(29.99).build();
        when(productService.getAllProducts()).thenReturn(java.util.List.of(response));

        // Act & Assert
        mockMvc.perform(get("/api/v1/products"))
                .andExpect(status().isOk()) // Expects HTTP 200
                .andExpect(jsonPath("$.status").value(200)) // Checks our wrapper
                .andExpect(jsonPath("$.message").value("Success"))
                .andExpect(jsonPath("$.data[0].name").value("Mouse"));
    }

    @Test
    void addProduct_WithValidData_Returns201Created() throws Exception {
        // Arrange
        ProductRequest request = new ProductRequest("Keyboard", 79.99, "Mechanical");
        ProductResponse response = ProductResponse.builder().id(2L).name("Keyboard").price(79.99).build();
        
        when(productService.addProduct(any(ProductRequest.class))).thenReturn(response);

        // Act & Assert
        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))) // Converts request to JSON
                .andExpect(status().isCreated()) // Expects HTTP 201
                .andExpect(jsonPath("$.status").value(201))
                .andExpect(jsonPath("$.data.name").value("Keyboard"));
    }
}
```

### Step 2: Run the Test
Right-click `ProductControllerTest.java` and select **Run**. 
You should see **2 tests passed**. 

*(Note: If you get an error about `ProductService` not being found, ensure you have `@MockBean` and not `@Mock`)*

---

## 🚀 The Power of Automated Testing
Now, anytime you change your code, you can run `./mvnw test`. If all tests pass, you can deploy to production with confidence, knowing the core functionality hasn't been broken!

---

## 🛠️ Exercise
1. Add a test for `GET /api/v1/products/{id}`.
2. Mock the `productService.getProductById(1L)` to return a product.
3. Use `mockMvc.perform(get("/api/v1/products/1"))` and assert the status is 200 and the JSON contains the product name.

---

## 🧠 Quiz
1. What is the difference between `@Mock` (Lesson 16) and `@MockBean` (Lesson 17)?
2. What does `MockMvc` do?
3. Why do we use `objectMapper.writeValueAsString()` in the POST test?

---

## 🎉 CONGRATULATIONS! PHASE 2 COMPLETE! 🎉

You have successfully built a **Production-Ready, Enterprise-Grade Spring Boot API** from scratch! 

**You now know how to:**
✅ Set up Dockerized PostgreSQL & pgAdmin.
✅ Use JPA, `BaseEntity`, and Flyway migrations.
✅ Implement the Controller → Service(Interface) → ServiceImpl → Repository architecture.
✅ Use DTOs and MapStruct for clean data transfer.
✅ Validate input and handle exceptions globally.
✅ Wrap responses in a standardized `HttpBodyResponse`.
✅ Implement Pagination and Sorting.
✅ Write Unit and Integration tests with JUnit and Mockito.

---

## 🛑 STOP
Reply with **"Phase 2 Complete"** and your exercise/quiz answers for Lesson 17. 

Once you do, we will officially graduate to **Phase 3: Spring Security & JWT**, where we will learn how to protect this beautiful API with logins, roles, and tokens!