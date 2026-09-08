# 📘 Phase 5, Lesson 3: The Login Endpoint & Access Tokens

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Concept: Exchanging the Passport](#-the-concept-exchanging-the-passport)
- [Step 1: Create the Login DTO](#-step-1-create-the-login-dto)
- [Step 2: Build the Login Endpoint](#-step-2-build-the-login-endpoint)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Build a professional `/api/auth/login` endpoint.
- ✅ Use Spring's `AuthenticationManager` to verify credentials.
- ✅ Generate a JWT upon successful login and return it to the client.

---

## 🎟️ The Concept: Exchanging the Passport

In the festival analogy, the Login Endpoint is the **Main Gate**. 
The user hands over their Passport and ID (Username and Password). 
The guard checks them against the guest list (Database). 
If they match, the guard puts the wristband (JWT) on their wrist and says, "Welcome, show this wristband at every door from now on."

---

## 🛠️ Step-by-Step Build

### Step 1: Create the Login DTO

```java
// src/main/java/com/example/product/dto/request/LoginRequest.java
package com.example.product.dto.request;

import jakarta.validation.constraints.NotBlank;
import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class LoginRequest {

    @NotBlank(message = "Username is required")
    private String username;

    @NotBlank(message = "Password is required")
    private String password;
}
```

Update the `AuthResponse` to include the token:
```java
// src/main/java/com/example/product/dto/response/AuthResponse.java
package com.example.product.dto.response;

import lombok.*;

@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class AuthResponse {
    private String accessToken;
    private String tokenType = "Bearer";
    private String username;
}
```

### Step 2: Build the Login Endpoint

Update your `AuthService` and `AuthController`.

**AuthService:**
```java
// src/main/java/com/example/product/service/AuthService.java
// ... add to existing class ...

import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.AuthenticationException;

// Inject AuthenticationManager and JwtService in the constructor!
private final AuthenticationManager authenticationManager;
private final JwtService jwtService;

// ...

public AuthResponse login(LoginRequest request) {
    // 1. Tell Spring Security to verify the username and password
    // This will automatically call our CustomUserDetailsService and check the BCrypt hash!
    try {
        authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword())
        );
    } catch (AuthenticationException e) {
        throw new RuntimeException("Invalid username or password");
    }

    // 2. If successful, load the user to generate the token
    var user = userDetailsService.loadUserByUsername(request.getUsername());

    // 3. Generate the JWT
    String jwtToken = jwtService.generateToken(user);

    // 4. Return the response
    return AuthResponse.builder()
            .accessToken(jwtToken)
            .username(user.getUsername())
            .build();
}
```

**AuthController:**
```java
// src/main/java/com/example/product/controller/AuthController.java
// ... add to existing class ...

import com.example.product.dto.request.LoginRequest;

@PostMapping("/login")
public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
    AuthResponse response = authService.login(request);
    return ResponseEntity.ok(response);
}
```

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

**Test 1: Login with correct credentials**
```bash
curl -X POST http://localhost:8080/api/auth/login \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "password": "securePassword123"
}'
```
*Expected:*
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huX2RvZSIs...",
  "tokenType": "Bearer",
  "username": "john_doe"
}
```

**Test 2: Use the returned token!**
Copy the `accessToken` from the response, and use it to access the protected endpoint:
```bash
curl -H "Authorization: Bearer <PASTE_TOKEN_HERE>" http://localhost:8080/api/users/me
```
*Expected:* `Hello, john_doe! ...`

**Test 3: Login with WRONG password**
```bash
curl -X POST http://localhost:8080/api/auth/login \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "password": "wrongPassword"
}'
```
*Expected:* `500 Internal Server Error` (with the message "Invalid username or password"). *Note: In a real app, we would catch this in the GlobalExceptionHandler and return a 401 Unauthorized.*

---

## 🚨 Common Errors
1. **`Bad credentials` exception**: The password in the database doesn't match the one you are sending. Ensure you hashed the password during registration!
2. **`NullPointerException` on `AuthenticationManager`**: You forgot to inject it into the `AuthService` constructor.

---

## 🛠️ Exercise
1. Add a handler in `GlobalExceptionHandler` for `AuthenticationException` (or `BadCredentialsException`).
2. Make it return a `401 Unauthorized` status with a clean JSON message: `{"error": "Invalid username or password"}`.
3. Test the wrong password login again and verify you get a clean 401 response.

---

## 🧠 Quiz
1. What Spring class is used to verify the username and password inside the `AuthService`?
2. Why do we return the `accessToken` to the client instead of keeping the user logged in on the server?
3. What happens if the user sends the wrong password to the `/login` endpoint?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 4.