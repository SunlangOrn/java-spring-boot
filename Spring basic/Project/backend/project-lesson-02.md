# 🛒 ShopCore Project - Lesson 2: Authentication API (Register, Login, JWT)

---

## 🎯 Goal
- ✅ Build the `/api/auth/register` endpoint to create new users.
- ✅ Build the `/api/auth/login` endpoint to verify passwords and return JWT tokens.
- ✅ Build the `/api/auth/refresh` endpoint to get new access tokens.

---

## 🧠 The Big Picture
Think of this like a gym membership:
1. **Register**: You fill out a form, and the gym gives you a membership card (User account).
2. **Login**: You show your card at the door. The guard checks it, and gives you a daily wristband (Access Token).
3. **Refresh**: If your wristband expires, you show your membership card again to get a new wristband without filling out the form again.

---

## 🛠️ Step 1: Create the Auth DTOs

### What we're doing:
Create the data structures for what the client sends (Request) and what we send back (Response).

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

**File 2:** `src/main/java/com/example/demo/dto/request/LoginRequest.java`
```java
package com.example.demo.dto.request;

import jakarta.validation.constraints.NotBlank;

public record LoginRequest(
    @NotBlank(message = "Username is required") String username,
    @NotBlank(message = "Password is required") String password
) {}
```

**File 3:** `src/main/java/com/example/demo/dto/response/AuthResponse.java`
```java
package com.example.demo.dto.response;

import lombok.Builder;
import lombok.Getter;
import java.util.Set;

@Getter
@Builder
public class AuthResponse {
    private String accessToken;
    private String refreshToken;
    private String tokenType = "Bearer";
    private Long id;
    private String username;
    private String email;
    private Set<String> roles;
    private String message;
}
```

### 📝 After the Code - What Just Happened?
- We use Java `record` for Requests because they are simple, immutable data carriers.
- We use Lombok `@Builder` and `@Getter` for the Response to make it easy to construct and read.
- `@NotBlank` and `@Email` ensure bad data is rejected *before* it reaches our database.

### 📦 Imports to Remember
```java
import jakarta.validation.constraints.*; // For validation
import lombok.Builder;
import lombok.Getter;
```

---

## 🛠️ Step 2: Build the AuthService

### What we're doing:
Write the business logic to hash passwords, assign roles, and generate JWTs.

### The Code:
Create file: `src/main/java/com/example/demo/service/AuthService.java`

```java
package com.example.demo.service;

import com.example.demo.dto.request.LoginRequest;
import com.example.demo.dto.request.RegisterRequest;
import com.example.demo.dto.response.AuthResponse;
import com.example.demo.entity.Role;
import com.example.demo.entity.User;
import com.example.demo.repository.RoleRepository;
import com.example.demo.repository.UserRepository;
import com.example.demo.security.JwtService;
import lombok.RequiredArgsConstructor;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.core.userdetails.UserDetails;
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
    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;
    private final CustomUserDetailsService userDetailsService;

    @Transactional
    public AuthResponse register(RegisterRequest request) {
        if (userRepository.existsByUsername(request.username())) {
            throw new RuntimeException("Username already exists");
        }
        
        String hashedPassword = passwordEncoder.encode(request.password());
        
        Role customerRole = roleRepository.findByName("ROLE_CUSTOMER")
                .orElseThrow(() -> new RuntimeException("Default role not found"));

        User newUser = User.builder()
                .username(request.username())
                .email(request.email())
                .passwordHash(hashedPassword)
                .isActive(true)
                .roles(new HashSet<>(Set.of(customerRole)))
                .build();

        User savedUser = userRepository.save(newUser);

        return buildAuthResponse(savedUser, "User registered successfully!");
    }

    @Transactional(readOnly = true)
    public AuthResponse login(LoginRequest request) {
        try {
            // This checks the password against the database hash automatically!
            authenticationManager.authenticate(
                    new UsernamePasswordAuthenticationToken(request.username(), request.password())
            );
        } catch (AuthenticationException e) {
            throw new RuntimeException("Invalid username or password");
        }

        UserDetails userDetails = userDetailsService.loadUserByUsername(request.username());
        User user = userRepository.findByUsername(request.username())
                .orElseThrow(() -> new RuntimeException("User not found"));

        return buildAuthResponse(user, "Login successful");
    }

    @Transactional(readOnly = true)
    public AuthResponse refreshToken(String refreshToken) {
        String username = jwtService.extractUsername(refreshToken);
        UserDetails userDetails = userDetailsService.loadUserByUsername(username);

        if (jwtService.isTokenValid(refreshToken, userDetails)) {
            String newAccessToken = jwtService.generateToken(userDetails);
            User user = userRepository.findByUsername(username)
                    .orElseThrow(() -> new RuntimeException("User not found"));
            
            return AuthResponse.builder()
                    .accessToken(newAccessToken)
                    .refreshToken(refreshToken)
                    .tokenType("Bearer")
                    .message("Token refreshed")
                    .build();
        }
        throw new RuntimeException("Invalid refresh token");
    }

    // Helper method to avoid repeating code
    private AuthResponse buildAuthResponse(User user, String message) {
        return AuthResponse.builder()
                .accessToken(jwtService.generateToken(userDetailsService.loadUserByUsername(user.getUsername())))
                .refreshToken(jwtService.generateRefreshToken(userDetailsService.loadUserByUsername(user.getUsername())))
                .tokenType("Bearer")
                .id(user.getId())
                .username(user.getUsername())
                .email(user.getEmail())
                .roles(user.getRoles().stream().map(r -> r.getName()).collect(Collectors.toSet()))
                .message(message)
                .build();
    }
}
```

### 📝 After the Code - What Just Happened?
- `register()`: Checks for duplicates, hashes the password with BCrypt, assigns `ROLE_CUSTOMER`, and saves.
- `login()`: Uses Spring's `AuthenticationManager` to safely verify the password. If it matches, it generates and returns the JWTs.
- `buildAuthResponse()`: A clean helper method that formats the final JSON response.

### 📦 Imports to Remember
```java
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.crypto.password.PasswordEncoder;
```

---

## 🛠️ Step 3: Build the AuthController

### What we're doing:
Expose the AuthService methods as HTTP endpoints.

### The Code:
Create file: `src/main/java/com/example/demo/controller/AuthController.java`

```java
package com.example.demo.controller;

import com.example.demo.common.BaseRestController;
import com.example.demo.common.HttpBodyResponse;
import com.example.demo.dto.request.LoginRequest;
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
    public ResponseEntity<HttpBodyResponse<AuthResponse>> register(@Valid @RequestBody RegisterRequest request) {
        return responseCreated(authService.register(request));
    }

    @PostMapping("/login")
    public ResponseEntity<HttpBodyResponse<AuthResponse>> login(@Valid @RequestBody LoginRequest request) {
        return responseSucceed(authService.login(request));
    }

    @PostMapping("/refresh")
    public ResponseEntity<HttpBodyResponse<AuthResponse>> refresh(@RequestParam String refreshToken) {
        return responseSucceed(authService.refreshToken(refreshToken));
    }
}
```

---

## 🧪 Run & Test

1. Start Docker: `docker-compose up -d`
2. Run the app: `./mvnw spring-boot:run`

**Test 1: Register**
```bash
curl -X POST http://localhost:8081/api/auth/register \
-H "Content-Type: application/json" \
-d '{"username": "shopper1", "email": "shopper@test.com", "password": "password123"}'
```
**Expected:** 201 Created with `accessToken`, `refreshToken`, and `roles: ["ROLE_CUSTOMER"]`.

**Test 2: Login**
```bash
curl -X POST http://localhost:8081/api/auth/login \
-H "Content-Type: application/json" \
-d '{"username": "shopper1", "password": "password123"}'
```
**Expected:** 200 OK with the tokens.

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Invalid username or password` | Password wasn't hashed during registration. | Ensure `passwordEncoder.encode()` is called in `register()`. |
| `Role not found` | Flyway migration didn't insert default roles. | Check `V1__init_security_tables.sql`. |

---

## ✏️ Exercise
1. Use the `accessToken` from the login response to call a protected endpoint (e.g., `GET /api/users/me` from Phase 5).
2. Verify it works by including the header: `-H "Authorization: Bearer <YOUR_TOKEN>"`.

---

## 🧠 Quiz
1. Why do we hash the password in the `AuthService` instead of the `Controller`?
2. What does the `AuthenticationManager` do during login?
3. Why do we return both an `accessToken` and a `refreshToken`?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Project Lesson 3!