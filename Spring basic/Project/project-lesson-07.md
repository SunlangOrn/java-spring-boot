# 🛒 ShopCore Project - Lesson 7: Automated Testing (JUnit & Mockito)

---

## 🎯 Goal
- ✅ Write Unit Tests for the Service layer using Mockito.
- ✅ Write Integration Tests for the Controller layer using MockMvc.
- ✅ Understand how to test code in isolation vs. testing the whole flow.

---

## 🧠 The Big Picture
Testing is your safety net. 
- **Unit Test**: Tests *one single class* (e.g., `AuthService`) by faking (mocking) its dependencies (like the database). It runs in milliseconds.
- **Integration Test**: Tests how classes work *together* (e.g., Controller → Service). It loads a slice of the Spring context to ensure the wiring is correct.

---

## 🛠️ Step 1: Write a Unit Test for AuthService

### What we're doing:
Test the `register` method to ensure it hashes the password and saves the user, without actually touching the real database.

### The Code:
Create file: `src/test/java/com/example/demo/service/AuthServiceTest.java`

 and `org.mockito.Mockito.*`

```java
package com.example.demo.service;

import com.example.demo.dto.request.RegisterRequest;
import com.example.demo.dto.response.AuthResponse;
import com.example.demo.entity.Role;
import com.example.demo.entity.User;
import com.example.demo.repository.RoleRepository;
import com.example.demo.repository.UserRepository;
import com.example.demo.security.JwtService;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.util.Optional;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class AuthServiceTest {

    @Mock private UserRepository userRepository;
    @Mock private RoleRepository roleRepository;
    @Mock private PasswordEncoder passwordEncoder;
    @Mock private AuthenticationManager authenticationManager;
    @Mock private JwtService jwtService;
    @Mock private CustomUserDetailsService userDetailsService;

    @InjectMocks
    private AuthService authService;

    @Test
    void register_Success_ReturnsAuthResponse() {
        // 1. ARRANGE: Set up the fake data and mock behaviors
        RegisterRequest request = new RegisterRequest("testuser", "test@test.com", "password123");
        Role customerRole = new Role();
        customerRole.setName("ROLE_CUSTOMER");
        
        User savedUser = User.builder().id(1L).username("testuser").email("test@test.com").build();
        savedUser.setRoles(Set.of(customerRole));

        when(userRepository.existsByUsername("testuser")).thenReturn(false);
        when(passwordEncoder.encode("password123")).thenReturn("hashed_password");
        when(roleRepository.findByName("ROLE_CUSTOMER")).thenReturn(Optional.of(customerRole));
        when(userRepository.save(any(User.class))).thenReturn(savedUser);
        when(userDetailsService.loadUserByUsername("testuser")).thenReturn(org.springframework.security.core.userdetails.User.builder().username("testuser").password("hashed").roles("CUSTOMER").build());
        when(jwtService.generateToken(any())).thenReturn("fake_access_token");
        when(jwtService.generateRefreshToken(any())).thenReturn("fake_refresh_token");

        // 2. ACT: Call the method we are testing
        AuthResponse response = authService.register(request);

        // 3. ASSERT: Verify the results
        assertNotNull(response);
        assertEquals("testuser", response.getUsername());
        assertEquals("fake_access_token", response.getAccessToken());
        
        // Verify that the save method was called exactly once
        verify(userRepository, times(1)).save(any(User.class));
    }
}
```

### 📝 After the Code - What Just Happened?
- `@Mock`: Creates a fake version of the dependency.
- `@InjectMocks`: Creates the real `AuthService` and injects the fake mocks into it.
- `when(...).thenReturn(...)`: Tells the fake dependency how to behave when called.
- `verify(...)`: Proves that our service actually called the repository's `save` method.

---

## 🛠️ Step 2: Write an Integration Test for the Controller

### What we're doing:
Test the `AuthController` to ensure it correctly receives HTTP requests, validates them, and returns the correct HTTP status and JSON.

### The Code:
Create file: `src/test/java/com/example/demo/controller/AuthControllerTest.java`

```java
package com.example.demo.controller;

import com.example.demo.dto.request.RegisterRequest;
import com.example.demo.dto.response.AuthResponse;
import com.example.demo.service.AuthService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(AuthController.class) // Loads ONLY the web layer, not the whole app
class AuthControllerTest {

    @Autowired
    private MockMvc mockMvc; // Simulates HTTP requests

    @MockBean // Replaces the real AuthService with a Mock in the Spring Context
    private AuthService authService;

    @Autowired
    private ObjectMapper objectMapper; // Converts Java objects to JSON strings

    @Test
    void register_ValidRequest_Returns201Created() throws Exception {
        // 1. ARRANGE
        RegisterRequest request = new RegisterRequest("newuser", "new@test.com", "password123");
        AuthResponse mockResponse = AuthResponse.builder()
                .username("newuser")
                .message("User registered successfully!")
                .build();

        when(authService.register(any(RegisterRequest.class))).thenReturn(mockResponse);

        // 2. ACT & 3. ASSERT
        mockMvc.perform(post("/api/auth/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated()) // Expects HTTP 201
                .andExpect(jsonPath("$.message").value("Success")) // Checks our HttpBodyResponse wrapper
                .andExpect(jsonPath("$.data.username").value("newuser"));
    }
}
```

### 📝 After the Code - What Just Happened?
- `@WebMvcTest`: Tells Spring to only load the Controller layer, making the test very fast.
- `@MockBean`: Unlike `@Mock`, this tells the *Spring Context* to replace the real bean with a mock.
- `mockMvc.perform(...)`: Simulates a real HTTP POST request with a JSON body.
- `andExpect(...)`: Verifies the HTTP status code and the exact JSON structure returned.

---

## 🧪 Run & Test

Right-click on the `src/test/java` folder in your IDE and select **Run 'All Tests'**.
You should see all tests pass with green checkmarks in a few seconds!

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `NullPointerException` in Unit Test | Forgot `@InjectMocks` or `@Mock`. | Ensure all annotations are present and `@ExtendWith(MockitoExtension.class)` is on the class. |
| `Wanted but not invoked` | Your service didn't call the mock the way you expected. | Check your `verify()` statements match the actual service logic. |
| `ApplicationContext` fails to load in Integration Test | Missing dependencies or wrong `@WebMvcTest` target. | Ensure you only load the specific Controller you are testing. |

---

## ✏️ Exercise
1. Write a Unit Test for `ProductService.createProduct()` that verifies the `ProductRepository.save()` method is called.
2. Write an Integration Test for `GET /api/v1/products` that expects a 200 OK status.

---

## 🧠 Quiz
1. What is the main difference between `@Mock` and `@MockBean`?
2. Why do we use `MockMvc` instead of starting the whole application for Controller tests?
3. What does the `verify()` method do in Mockito?

---

## 🎉 CONGRATULATIONS! SHOPCORE PROJECT COMPLETE! 🎉

You have successfully built a **Production-Ready, Enterprise-Grade E-Commerce Backend** from scratch! 

**You now know how to:**
✅ Design a secure database schema with Users, Roles, and Permissions.
✅ Implement JWT Authentication (Register, Login, Refresh).
✅ Build a fast, cached Catalog API with Pagination and Sorting.
✅ Create a Shopping Cart and a Transactional Checkout system.
✅ Document the API beautifully with Swagger.
✅ Prove the code works with automated Unit and Integration tests.

---

## 🛑 STOP
Reply with **"ShopCore Complete"** and your exercise/quiz answers for Lesson 7. 

Once you do, you have officially mastered the Spring Boot Fundamentals and Advanced Application Building! Let me know if you want to review anything, or if you are ready to explore the next big topic (like Microservices, Kubernetes, or Advanced Architecture)! 🚀
