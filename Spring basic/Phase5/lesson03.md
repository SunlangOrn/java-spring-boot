# 📘 Phase 5, Lesson 3: The Login Endpoint & Access Tokens

---

## 🎯 Goal
- ✅ Build a professional `/api/auth/login` endpoint.
- ✅ Use Spring's `AuthenticationManager` to verify credentials.
- ✅ Generate a JWT upon successful login and return it to the client.

---

## 🧠 The Big Picture

In the festival analogy, the Login Endpoint is the **Main Gate**. 
The user hands over their Passport and ID (Username and Password). 
The guard checks them against the guest list (Database). 
If they match, the guard puts the wristband (JWT) on their wrist and says, "Welcome, show this wristband at every door from now on."

---

## 🛠️ Step 1: Create the Login DTO

### What we're doing:
Create a simple class to hold the username and password the client sends.

### The Code:
Create file: `src/main/java/com/example/demo/dto/request/LoginRequest.java`

```java
package com.example.demo.dto.request;

import jakarta.validation.constraints.NotBlank;

public record LoginRequest(
    @NotBlank(message = "Username is required")
    String username,

    @NotBlank(message = "Password is required")
    String password
) {}
```

**Update:** `src/main/java/com/example/demo/dto/response/AuthResponse.java`
```java
package com.example.demo.dto.response;

import lombok.Builder;
import lombok.Getter;
import java.util.Set;

@Getter
@Builder
public class AuthResponse {
    private String accessToken;  // NEW: The JWT wristband
    private String tokenType = "Bearer"; // NEW: Tells the client how to use it
    private Long id;
    private String username;
    private String email;
    private Set<String> roles;
    private String message;
}
```

---

## 🛠️ Step 2: Build the Login Logic

### What we're doing:
Use the `AuthenticationManager` to verify the password, then generate the JWT.

### The Code:
**Update:** `src/main/java/com/example/demo/service/AuthService.java`

```java
// Add these imports:
import com.example.demo.dto.request.LoginRequest;
import com.example.demo.security.JwtService;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.AuthenticationException;

// ... inside AuthService class ...

private final AuthenticationManager authenticationManager; // Inject this in constructor
private final JwtService jwtService; // Inject this in constructor

public AuthResponse login(LoginRequest request) {
    // 1. Tell Spring Security to verify the username and password
    // This will automatically call our CustomUserDetailsService and check the BCrypt hash!
    try {
        authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.username(), request.password())
        );
    } catch (AuthenticationException e) {
        throw new RuntimeException("Invalid username or password");
    }

    // 2. If successful, load the user to generate the token
    var user = userDetailsService.loadUserByUsername(request.username());

    // 3. Generate the JWT
    String jwtToken = jwtService.generateToken(user);

    // 4. Return the response
    return AuthResponse.builder()
            .accessToken(jwtToken)
            .id(user.getId()) // Assuming User entity has getId()
            .username(user.getUsername())
            .email(user.getEmail())
            .roles(user.getRoles().stream().map(role -> role.getName()).collect(java.util.stream.Collectors.toSet()))
            .message("Login successful")
            .build();
}
```

### 📝 After the Code - What Just Happened?
- `authenticationManager.authenticate(...)`: This is the magic. It takes the raw password, finds the user in the DB, hashes the raw password using the stored salt, and compares them. If they don't match, it throws an `AuthenticationException`.
- If it succeeds, we know the user is who they say they are. We then generate the JWT and return it.

---

## 🛠️ Step 3: Build the Login Endpoint

### What we're doing:
Expose the login logic via a POST endpoint.

### The Code:
**Update:** `src/main/java/com/example/demo/controller/AuthController.java`

```java
// Add these imports:
import com.example.demo.dto.request.LoginRequest;

// ... inside AuthController class ...

@PostMapping("/login")
public ResponseEntity<HttpBodyResponse<AuthResponse>> login(
        @Valid @RequestBody LoginRequest request) {
    
    AuthResponse response = authService.login(request);
    return responseSucceed(response); // Returns 200 OK with the token
}
```

---

## 🧪 Run & Test

### Test 1: Login with correct credentials
```bash
curl -X POST http://localhost:8081/api/auth/login \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "password": "securePassword123"
}'
```
**Expected:**
```json
{
  "status": 200,
  "message": "Success",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huX2RvZSIs...",
    "tokenType": "Bearer",
    "username": "john_doe",
    "message": "Login successful"
  }
}
```

### Test 2: Use the returned token!
Copy the `accessToken` from the response, and use it to access the protected endpoint:
```bash
curl -H "Authorization: Bearer <PASTE_TOKEN_HERE>" http://localhost:8081/api/users/me
```
**Expected:** `Hello, john_doe! ...`

### Test 3: Login with WRONG password
```bash
curl -X POST http://localhost:8081/api/auth/login \
-H "Content-Type: application/json" \
-d '{
  "username": "john_doe",
  "password": "wrongPassword"
}'
```
**Expected:** `500 Internal Server Error` (with the message "Invalid username or password"). 

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Bad credentials` exception | The password in the database doesn't match the one you are sending. | Ensure you hashed the password during registration! |
| `NullPointerException` on `AuthenticationManager` | You forgot to inject it into the `AuthService` constructor. | Add it to the constructor and let Lombok's `@RequiredArgsConstructor` handle it. |

---

## ✏️ Exercise

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