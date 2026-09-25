# 📘 Phase 4, Lesson 3: Registration & Login API

---

## 🎯 Goal
- ✅ Build a professional `/api/auth/register` endpoint so users can sign up.
- ✅ Understand how to assign roles during registration.
- ✅ Prepare the foundation for Phase 5 (JWT) by structuring the Auth layer correctly.

---

## 🧠 The Big Picture

In Lesson 1, we created the "admin" user using a `CommandLineRunner`. That's great for testing, but in the real world, users register via an API. 

We must **never** put security logic (like password encoding) inside the Controller. The Controller just takes the request. The **AuthService** handles the business logic (hashing the password, assigning roles, saving to DB).

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **AuthService** | The service layer dedicated purely to handling user registration and authentication logic. |
| **Separation of Concerns** | The principle that Controllers should only handle HTTP, while Services handle business logic. |

---

## 🛠️ Step 1: Create the Auth DTOs

### What we're doing:
Create a Request DTO for registration and a Response DTO to send back to the client.

### The Code:

**File 1:** `src/main/java/com/example/demo/dto/request/RegisterRequest.java`
```java
package com.example.demo.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record RegisterRequest(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    String username,

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 6, message = "Password must be at least 6 characters")
    String password
) {}
```

**File 2:** `src/main/java/com/example/demo/dto/response/AuthResponse.java`
```java
package com.example.demo.dto.response;

import lombok.Builder;
import lombok.Getter;
import java.util.Set;

@Getter
@Builder
public class AuthResponse {
    private Long id;
    private String username;
    private String email;
    private Set<String> roles;
    private String message;
}
```

### 📝 After the Code - What Just Happened?
- `RegisterRequest`: Validates that the user provides a valid email and a password of at least 6 characters.
- `AuthResponse`: Returns the user's details and their assigned roles. (In Phase 5, we will add an `accessToken` field here for JWT).

---

## 🛠️ Step 2: Build the AuthService

### What we're doing:
Write the logic to check for duplicates, hash the password, assign the default role, and save the user.

### The Code:
Create file: `src/main/java/com/example/demo/service/AuthService.java`

```java
package com.example.demo.service;

import com.example.demo.dto.request.RegisterRequest;
import com.example.demo.dto.response.AuthResponse;
import com.example.demo.entity.Role;
import com.example.demo.entity.User;
import com.example.demo.exception.ResourceNotFoundException;
import com.example.demo.repository.RoleRepository;
import com.example.demo.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashSet;
import java.util.Set;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class AuthService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final PasswordEncoder passwordEncoder;

    @Transactional
    public AuthResponse register(RegisterRequest request) {
        // 1. Check if username or email already exists
        if (userRepository.existsByUsername(request.username())) {
            throw new RuntimeException("Username already exists");
        }
        if (userRepository.existsByEmail(request.email())) {
            throw new RuntimeException("Email already exists");
        }

        // 2. Hash the password using our BCrypt "blender"
        String hashedPassword = passwordEncoder.encode(request.password());

        // 3. Assign default role (ROLE_USER)
        Role userRole = roleRepository.findByName("ROLE_USER")
                .orElseThrow(() -> new ResourceNotFoundException("Default role not found in DB"));

        Set<Role> roles = new HashSet<>();
        roles.add(userRole);

        // 4. Create and save the user
        User newUser = User.builder()
                .username(request.username())
                .email(request.email())
                .passwordHash(hashedPassword)
                .isActive(true)
                .roles(roles)
                .build();

        User savedUser = userRepository.save(newUser);

        // 5. Build and return response
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

### 📝 After the Code - What Just Happened?
- We use `existsByUsername` and `existsByEmail`. Spring Data JPA automatically generates the SQL for these methods just by looking at the method names!
- `passwordEncoder.encode()`: This is where the plain text password gets ground up into a BCrypt hash.
- We assign `ROLE_USER` by default. (Admins are usually created manually in the DB or via a special super-admin endpoint).

### 📦 Imports to Remember
```java
import org.springframework.security.crypto.password.PasswordEncoder;
```

### 💡 Note
> You need to add `existsByUsername(String username)` and `existsByEmail(String email)` to your `UserRepository` interface. Spring will implement them automatically!

---

## 🛠️ Step 3: Build the AuthController

### What we're doing:
Create the endpoint that clients will call to register.

### The Code:
Create file: `src/main/java/com/example/demo/controller/AuthController.java`

```java
package com.example.demo.controller;

import com.example.demo.common.BaseRestController;
import com.example.demo.common.HttpBodyResponse;
import com.example.demo.dto.request.RegisterRequest;
import com.example.demo.dto.response.AuthResponse;
import com.example.demo.service.AuthService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController extends BaseRestController {

    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<HttpBodyResponse<AuthResponse>> register(
            @Valid @RequestBody RegisterRequest request) {
        
        AuthResponse response = authService.register(request);
        return responseCreated(response); // Returns 201 Created
    }
    
    // NOTE: We will add the /login endpoint in Phase 5 when we implement JWT!
    // For Phase 4, we use HTTP Basic Auth (curl -u user:pass) to test the security layer.
}
```

### 📝 After the Code - What Just Happened?
- The controller is very thin. It just receives the `@Valid` request, passes it to the `AuthService`, and wraps the result in our `HttpBodyResponse`.
- Notice there is no `/login` endpoint yet. In Phase 4, we rely on Spring Security's built-in HTTP Basic login. In Phase 5, we will build a custom `/login` endpoint that returns JWT tokens.

---

## 🧪 Run & Test

### Test 1: Register a new user
```bash
curl -X POST http://localhost:8081/api/auth/register \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "securePassword123"
}'
```
**Expected:** `201 Created` with the user details and `roles: ["ROLE_USER"]`.

### Test 2: Try to register the same username again
```bash
curl -X POST http://localhost:8081/api/auth/register \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "email": "different@example.com",
  "password": "securePassword123"
}'
```
**Expected:** `500 Internal Server Error` (or 400 if you add exception handling) with the message "Username already exists".

### Test 3: Login with the new user (HTTP Basic Auth)
```bash
curl -u john_doe:securePassword123 http://localhost:8081/api/v1/products
```
**Expected:** `200 OK`. John is authenticated!

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Default role not found in DB` | You forgot to insert `ROLE_USER` in your Flyway migration. | Check `V6__create_users_and_roles.sql`. |
| `401 Unauthorized` on login | The password in the DB is not hashed. | Ensure `AuthService` uses `passwordEncoder.encode()`. |

---

## ✏️ Exercise

1. Add a `POST /api/auth/register/admin` endpoint (just for testing purposes).
2. In the `AuthService`, create a `registerAdmin` method that assigns `ROLE_ADMIN` instead of `ROLE_USER`.
3. Register an admin, then use `curl -u admin_username:password` to `POST /api/v1/products` to prove they are logged in.

---

## 🧠 Quiz

1. Why do we hash the password in the `AuthService` and not in the `Controller`?
2. What Spring Data JPA method would you write to check if an email exists?
3. Why don't we have a `/login` endpoint in this lesson?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 4.