# 📘 Phase 4, Lesson 3: Registration & Login API

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Concept: Separation of Concerns](#-the-concept-separation-of-concerns)
- [Step 1: Create the Auth DTOs](#-step-1-create-the-auth-dtos)
- [Step 2: Build the AuthService](#-step-2-build-the-authservice)
- [Step 3: Build the AuthController](#-step-3-build-the-authcontroller)
- [Run and Test](#-run-and-test)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Build a professional Registration endpoint.
- ✅ Understand how to assign roles during registration.
- ✅ Prepare the foundation for Phase 5 (JWT) by structuring the Auth layer correctly.

---

## 💡 The Concept: Separation of Concerns

In Lesson 1, we created a user using a `CommandLineRunner`. That's great for testing, but in the real world, users register via an API. 

We must **never** put security logic (like password encoding) inside the Controller. The Controller just takes the request. The **AuthService** handles the business logic (hashing the password, assigning roles, saving to DB).

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Auth DTOs

**Registration Request:**
```java
// src/main/java/com/example/product/dto/request/RegisterRequest.java
package com.example.product.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class RegisterRequest {

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    private String username;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 6, message = "Password must be at least 6 characters")
    private String password;
}
```

**Auth Response (We will expand this in Phase 5 with JWT):**
```java
// src/main/java/com/example/product/dto/response/AuthResponse.java
package com.example.product.dto.response;

import lombok.*;
import java.util.Set;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class AuthResponse {
    private Long id;
    private String username;
    private String email;
    private Set<String> roles;
    private String message;
}
```

### Step 2: Build the AuthService

```java
// src/main/java/com/example/product/service/AuthService.java
package com.example.product.service;

import com.example.product.dto.request.RegisterRequest;
import com.example.product.dto.response.AuthResponse;
import com.example.product.entity.security.Role;
import com.example.product.entity.security.User;
import com.example.product.exception.DuplicateResourceException; // Create this if it doesn't exist
import com.example.product.repository.RoleRepository;
import com.example.product.repository.UserRepository;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashSet;
import java.util.Set;
import java.util.stream.Collectors;

@Service
public class AuthService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final PasswordEncoder passwordEncoder;

    public AuthService(UserRepository userRepository, RoleRepository roleRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.roleRepository = roleRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @Transactional
    public AuthResponse register(RegisterRequest request) {
        // 1. Check if username or email already exists
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new DuplicateResourceException("Username already exists");
        }
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateResourceException("Email already exists");
        }

        // 2. Hash the password
        String hashedPassword = passwordEncoder.encode(request.getPassword());

        // 3. Assign default role (ROLE_USER)
        Role userRole = roleRepository.findByName("ROLE_USER")
                .orElseThrow(() -> new RuntimeException("Default role not found in DB"));

        Set<Role> roles = new HashSet<>();
        roles.add(userRole);

        // 4. Create and save the user
        User newUser = User.builder()
                .username(request.getUsername())
                .email(request.getEmail())
                .passwordHash(hashedPassword)
                .isActive(true)
                .roles(roles)
                .build();

        User savedUser = userRepository.save(newUser);

        // 5. Build response
        return AuthResponse.builder()
                .id(savedUser.getId())
                .username(savedUser.getUsername())
                .email(savedUser.getEmail())
                .roles(roles.stream().map(Role::getName).collect(Collectors.toSet()))
                .message("User registered successfully!")
                .build();
    }
}
```
*(Note: You will need to add `existsByUsername`, `existsByEmail` to `UserRepository`, and `findByName` to `RoleRepository`. Spring Data JPA generates these automatically based on the method name!)*

### Step 3: Build the AuthController

```java
// src/main/java/com/example/product/controller/AuthController.java
package com.example.product.controller;

import com.example.product.dto.request.RegisterRequest;
import com.example.product.dto.response.AuthResponse;
import com.example.product.service.AuthService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(@Valid @RequestBody RegisterRequest request) {
        AuthResponse response = authService.register(request);
        return new ResponseEntity<>(response, HttpStatus.CREATED);
    }
    
    // NOTE: We will add the /login endpoint in Phase 5 when we implement JWT!
    // For Phase 4, we use HTTP Basic Auth to test the security layer.
}
```

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

**Test 1: Register a new user**
```bash
curl -X POST http://localhost:8080/api/auth/register \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "securePassword123"
}'
```
*Expected:* `201 Created` with the user details and `roles: ["ROLE_USER"]`.

**Test 2: Try to register the same username again**
```bash
curl -X POST http://localhost:8080/api/auth/register \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "email": "different@example.com",
  "password": "securePassword123"
}'
```
*Expected:* `409 Conflict` (or 400) with the message "Username already exists".

**Test 3: Login with the new user (HTTP Basic Auth)**
```bash
curl -u john_doe:securePassword123 http://localhost:8080/api/products
```
*Expected:* `200 OK`. John is authenticated!

---

## 🚨 Common Errors
1. **`Default role not found in DB`**: You forgot to insert `ROLE_USER` in your Flyway migration. *Fix:* Check `V7__create_users_and_roles.sql`.
2. **`401 Unauthorized` on login**: The password in the DB is not hashed, or you typed the wrong password in curl.

---

## 🛠️ Exercise
1. Add a `POST /api/auth/register/admin` endpoint (just for testing purposes).
2. In the `AuthService`, create a `registerAdmin` method that assigns `ROLE_ADMIN` instead of `ROLE_USER`.
3. Register an admin, then use `curl -u admin_username:password` to `POST /api/products` to prove they have admin rights.

---

## 🧠 Quiz
1. Why do we hash the password in the `AuthService` and not in the `Controller`?
2. What Spring Data JPA method would you write to check if an email exists?
3. Why don't we have a `/login` endpoint in this lesson?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 4.