# 📘 Phase 5, Lesson 4: Refresh Tokens & Stateless Logout

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Concept: Access vs. Refresh Tokens](#-the-concept-access-vs-refresh-tokens)
- [Step 1: Update JwtService for Refresh Tokens](#-step-1-update-jwtservice-for-refresh-tokens)
- [Step 2: Build the Refresh Endpoint](#-step-2-build-the-refresh-endpoint)
- [The Concept: Stateless Logout](#-the-concept-stateless-logout)
- [Run and Test](#-run-and-test)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand the difference between Access Tokens and Refresh Tokens.
- ✅ Implement a `/refresh` endpoint to get new access tokens without logging in again.
- ✅ Understand how "Logout" works in a stateless JWT architecture.

---

## 🎟️ The Concept: Access vs. Refresh Tokens

Imagine your Access Token is a **hotel room key card**, and the Refresh Token is your **ID Card at the front desk**.

- **Access Token**: Short-lived (e.g., 15 minutes). You use it to open your room (access the API). If it expires, the door won't open.
- **Refresh Token**: Long-lived (e.g., 7 days). You keep it safe. When your room key expires, you go to the front desk, show your ID (Refresh Token), and they give you a new room key (New Access Token).

**Why do this?**
If a hacker steals your Access Token, they only have 15 minutes to do damage. But we don't want the user to have to type their password every 15 minutes. The Refresh Token solves this!

---

## 🛠️ Step-by-Step Build

### Step 1: Update JwtService for Refresh Tokens

Update `application.yml` to add a longer expiration for refresh tokens:
```yaml
app:
  jwt:
    secret: ${JWT_SECRET:mySuperSecretKeyForDevelopmentOnlyChangeInProduction123!}
    expiration: 900000          # 15 minutes (15 * 60 * 1000)
    refresh-expiration: 604800000 # 7 days (7 * 24 * 60 * 60 * 1000)
```

Update `JwtService.java` to generate refresh tokens:
```java
// src/main/java/com/example/product/security/JwtService.java
// ... add to existing class ...

@Value("${app.jwt.refresh-expiration}")
private long refreshExpiration;

public String generateRefreshToken(UserDetails userDetails) {
    return Jwts.builder()
            .subject(userDetails.getUsername())
            .issuedAt(new Date(System.currentTimeMillis()))
            .expiration(new Date(System.currentTimeMillis() + refreshExpiration))
            .signWith(getSignInKey(), Jwts.SIG.HS256)
            .compact();
}
```

### Step 2: Build the Refresh Endpoint

Update `AuthResponse` to include the refresh token:
```java
// Add to AuthResponse.java
private String refreshToken;
```

Update `AuthService.login()` to return both:
```java
// In AuthService.login()
String jwtToken = jwtService.generateToken(user);
String refreshToken = jwtService.generateRefreshToken(user); // ← NEW

return AuthResponse.builder()
        .accessToken(jwtToken)
        .refreshToken(refreshToken) // ← NEW
        .username(user.getUsername())
        .build();
```

Create the Refresh Endpoint in `AuthController`:
```java
// src/main/java/com/example/product/controller/AuthController.java
// ... add to existing class ...

@PostMapping("/refresh")
public ResponseEntity<AuthResponse> refreshToken(@RequestParam String refreshToken) {
    AuthResponse response = authService.refreshToken(refreshToken);
    return ResponseEntity.ok(response);
}
```

Add the logic in `AuthService`:
```java
// In AuthService.java
public AuthResponse refreshToken(String refreshToken) {
    // 1. Extract username from the refresh token
    String username = jwtService.extractUsername(refreshToken);
    
    // 2. Load user details
    UserDetails userDetails = userDetailsService.loadUserByUsername(username);
    
    // 3. Validate the refresh token
    if (jwtService.isTokenValid(refreshToken, userDetails)) {
        // 4. Generate a NEW access token
        String newAccessToken = jwtService.generateToken(userDetails);
        
        return AuthResponse.builder()
                .accessToken(newAccessToken)
                .refreshToken(refreshToken) // Return the same refresh token (or rotate it)
                .username(username)
                .build();
    }
    
    throw new RuntimeException("Invalid or expired refresh token");
}
```

---

## 🚪 The Concept: Stateless Logout

**The Problem:** Because JWTs are stateless, the server **does not know** if a token has been "logged out". If a user clicks "Logout", the client app just deletes the token from local storage. But if a hacker stole that token *before* the user logged out, the hacker can still use it until it expires!

**The Solutions:**
1. **Short-lived Access Tokens**: Make the access token expire in 15 minutes. When the user logs out, the hacker's token becomes useless in 15 minutes. (This is what we are doing).
2. **Token Blacklist (Redis)**: When a user logs out, add their token to a Redis blacklist. The `JwtAuthenticationFilter` checks Redis before accepting the token. (We will cover this in Phase 11).
3. **Refresh Token Revocation**: Store refresh tokens in the database. When the user logs out, delete the refresh token from the DB.

For now, **Client-Side Logout** is sufficient for our learning phase. The client simply deletes the `accessToken` and `refreshToken` from memory/local storage.

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

**Test 1: Login and get both tokens**
```bash
curl -X POST http://localhost:8080/api/auth/login \
-H "Content-Type: application/json" \
-d '{"username": "john_doe", "password": "securePassword123"}'
```
*Expected:* JSON with `accessToken` and `refreshToken`.

**Test 2: Wait 15 minutes (or change expiration to 10 seconds for testing)**
Try using the `accessToken`. It will fail (403 Forbidden) because it expired.

**Test 3: Use the Refresh Token to get a new Access Token**
```bash
curl -X POST "http://localhost:8080/api/auth/refresh?refreshToken=<PASTE_REFRESH_TOKEN_HERE>"
```
*Expected:* A new JSON with a brand new `accessToken`!

---

## 🚨 Common Errors
1. **`JWT expired` on refresh**: Your refresh token expired. Ensure `app.jwt.refresh-expiration` is set to a future date.
2. **`400 Bad Request` on `/refresh`**: You forgot to pass the `refreshToken` as a query parameter or request body.

---

## 🛠️ Exercise
1. Implement a "Logout" endpoint: `POST /api/auth/logout`.
2. Since we are stateless, the endpoint doesn't actually need to do anything on the server side. Just return a `200 OK` with the message "Logged out successfully. Please delete your tokens on the client side."
3. (Optional) Add a `@PreAuthorize("isAuthenticated()")` to the logout endpoint so only logged-in users can call it.

---

## 🧠 Quiz
1. Why do we use a short-lived Access Token and a long-lived Refresh Token?
2. Why is "Logout" difficult in a stateless JWT architecture?
3. What is one way to implement a "true" server-side logout with JWTs?

---

## 🎉 Phase 5 Complete!

You have successfully implemented a complete, industry-standard JWT Authentication system!
- ✅ Users can register and login.
- ✅ Passwords are hashed with BCrypt.
- ✅ The server issues Access and Refresh tokens.
- ✅ The `JwtAuthenticationFilter` intercepts requests and validates tokens.
- ✅ Spring Security knows who the user is without server-side sessions.

## 🛑 STOP
Reply with "Phase 5 Complete" and your exercise results. 

Next, we will move to **Phase 6: OAuth2 & Authorization Server**, where we will learn how to let users log in using Google/GitHub, and how to build our own Authorization Server!