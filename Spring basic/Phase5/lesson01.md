# 📘 Phase 5, Lesson 1: Introduction to JWT & Generating Tokens

---

## 🎯 Goal
- ✅ Understand why we need JWTs instead of sending passwords every time.
- ✅ Understand the 3 parts of a JWT (Header, Payload, Signature).
- ✅ Add the JJWT library to the project.
- ✅ Write a service to generate and decode JWTs.

---

## 🧠 The Big Picture

Right now, we are using **HTTP Basic Auth**. Every time the user wants to buy a product, their app sends their username and password in the header. 

**Why is this bad?**
1. **Inefficient:** The server has to query the database and run BCrypt (which is slow) on *every single request*.
2. **Security Risk:** If the connection is intercepted, the hacker gets the actual password.
3. **Stateful:** The server has to keep track of who is logged in.

### The Solution: The "Festival Wristband" Analogy
Imagine you go to a 3-day Music Festival.
- **Day 1 (HTTP Basic Auth):** Every time you buy a drink, you show your Passport (Password). The bartender calls the main office (Database) to verify it. This takes 5 minutes per drink.
- **Day 2 (JWT):** On Day 1, you show your Passport at the main gate. The guard puts a **glowing, tamper-proof wristband** on your wrist. Now, every time you buy a drink, the bartender just *looks at the wristband*. 

The wristband is the **JWT**. It contains your name, your VIP status, and an expiration time. It is mathematically sealed so no one can forge it.

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **JWT** | JSON Web Token. A standard format for securely transmitting information between parties as a JSON object. |
| **Header** | Part 1 of the JWT. Says what algorithm was used to sign it (e.g., HS256). |
| **Payload** | Part 2 of the JWT. The actual data (e.g., `username: "john"`, `role: "ADMIN"`). These are called **Claims**. |
| **Signature** | Part 3 of the JWT. A mathematical hash of the Header + Payload + a **Secret Key**. If anyone changes the Payload, the Signature breaks. |
| **Secret Key** | A long, random string known ONLY to the server. Used to sign and verify the token. |

---

## 🛠️ Step 1: Add JWT Dependencies

### What we're doing:
Add the standard Java library for creating and reading JWTs.

### The Code:
**Update:** `pom.xml`

```xml
<dependencies>
    <!-- ... your other dependencies ... -->

    <!-- JWT (JSON Web Token) -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 📝 After the Code - What Just Happened?
- `jjwt-api`: The core interfaces and classes we will use in our code.
- `jjwt-impl` & `jjwt-jackson`: The actual engine that does the cryptographic work and converts the token to/from JSON. They are `runtime` scope because we don't need them to compile our code, only to run it.

---

## 🛠️ Step 2: Configure the Secret Key

### What we're doing:
Create a secret key that the server will use to sign the tokens.

### The Code:
**Update:** `src/main/resources/application.yml`

```yaml
app:
  jwt:
    # This must be at least 256 bits (32 characters) for the HS256 algorithm.
    # In production, generate a random 64-character string!
    secret: ${JWT_SECRET:mySuperSecretKeyForDevelopmentOnlyChangeInProduction123!}
    expiration: 900000          # 15 minutes in milliseconds (15 * 60 * 1000)
    refresh-expiration: 604800000 # 7 days in milliseconds
```

### 📝 After the Code - What Just Happened?
- `secret`: The password used to sign the token. **Never hardcode this in Java!** We use an environment variable `${JWT_SECRET}` with a fallback for local development.
- `expiration`: How long the "wristband" is valid. 15 minutes is standard for security.

### 💡 Note
> ⚠️ **CRITICAL SECURITY RULE:** If you push your code to GitHub with a hardcoded secret, hackers will steal it and forge their own admin tokens! Always use environment variables in production.

---

## 🛠️ Step 3: Create the JwtService

### What we're doing:
Create the "Festival Security Guard" service. It knows how to create the wristband (generate token) and how to check if the wristband is fake (validate token).

### The Code:
Create file: `src/main/java/com/example/demo/security/JwtService.java`

```java
package com.example.demo.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.function.Function;

@Service
public class JwtService {

    @Value("${app.jwt.secret}")
    private String secretKey;

    @Value("${app.jwt.expiration}")
    private long jwtExpiration;

    /**
     * GENERATE THE WRISTBAND (TOKEN)
     */
    public String generateToken(UserDetails userDetails) {
        return generateToken(new HashMap<>(), userDetails);
    }

    public String generateToken(Map<String, Object> extraClaims, UserDetails userDetails) {
        return Jwts.builder()
                .claims(extraClaims) // Add custom data (like roles)
                .subject(userDetails.getUsername()) // The "sub" claim (usually username)
                .issuedAt(new Date(System.currentTimeMillis())) // When it was created
                .expiration(new Date(System.currentTimeMillis() + jwtExpiration)) // When it expires
                .signWith(getSignInKey(), Jwts.SIG.HS256) // Sign it with our secret key
                .compact(); // Build the final string
    }

    /**
     * CHECK THE WRISTBAND (EXTRACT DATA)
     */
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    /**
     * VERIFY THE WRISTBAND IS VALID
     */
    public boolean isTokenValid(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        // 1. Is the username in the token matching the user in our DB?
        // 2. Is the token expired?
        return (username.equals(userDetails.getUsername())) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractClaim(token, Claims::getExpiration).before(new Date());
    }

    // --- INTERNAL HELPER METHODS ---

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                .verifyWith(getSignInKey()) // Verify the signature using our secret
                .build()
                .parseSignedClaims(token) // Parse the token
                .getPayload(); // Get the data inside
    }

    private SecretKey getSignInKey() {
        // Convert the string secret to a cryptographic key
        byte[] keyBytes = secretKey.getBytes();
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

### 📝 After the Code - What Just Happened?
- `Jwts.builder()`: Starts creating the token.
- `.subject()`: Sets the "sub" claim (the username).
- `.expiration()`: Sets the exact millisecond the token dies.
- `.signWith(...)`: Takes the Header + Payload, and mathematically hashes them using our `secretKey`. This creates the Signature.
- `.compact()`: Squashes it all together into the final `eyJ...` string.
- `isTokenValid()`: Checks two things: Does the username match our database? And has the token expired?

### 📦 Imports to Remember
```java
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import javax.crypto.SecretKey;
```

---

## 🧪 Run & Test

Let's write a quick test in our `DemoApplication` to prove we can generate a token and read it.

**Update:** `src/main/java/com/example/demo/DemoApplication.java`

```java
// Add this CommandLineRunner to DemoApplication.java
@Bean
CommandLineRunner testJWT(JwtService jwtService) {
    return args -> {
        // 1. Create a fake UserDetails object (pretend this came from the DB)
        UserDetails fakeUser = org.springframework.security.core.userdetails.User.builder()
                .username("john_doe")
                .password("doesntmatter")
                .roles("USER")
                .build();

        // 2. Generate the Token
        String token = jwtService.generateToken(fakeUser);
        
        System.out.println("=================================");
        System.out.println("GENERATED JWT:");
        System.out.println(token);
        System.out.println("=================================");

        // 3. Extract data from the Token
        String extractedUsername = jwtService.extractUsername(token);
        System.out.println("Extracted Username: " + extractedUsername);
        
        // 4. Check if it's valid
        boolean isValid = jwtService.isTokenValid(token, fakeUser);
        System.out.println("Is Token Valid? " + isValid);
        System.out.println("=================================");
    };
}
```

### Run the app:
```bash
./mvnw spring-boot:run
```

Look at the console! You will see a massive string. 
Copy that entire string, go to **[jwt.io](https://jwt.io)**, and paste it into the "Encoded" box on the left. 
Watch as it magically decodes the Header, Payload (showing `john_doe`), and the Signature on the right!

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `The signing key's size is X bits which is smaller than required...` | Your `app.jwt.secret` in `application.yml` is too short. | HS256 requires at least 256 bits (32 characters). Make your secret longer. |
| `JWT expired at...` | You are trying to validate a token that was generated in the past. | Check your `app.jwt.expiration` in `application.yml`. |

---

## ✏️ Exercise

1. In the `CommandLineRunner`, generate a token for a user named `admin_user`.
2. Add a custom claim to the token: `Map.of("roles", List.of("ROLE_ADMIN"))`. Pass this map as the first argument to `generateToken()`.
3. Print the new token.
4. Paste it into [jwt.io](https://jwt.io) and verify that the `roles` array is visible in the Payload.

---

## 🧠 Quiz

1. What are the 3 parts of a JWT?
2. If a hacker changes the `role` from `USER` to `ADMIN` inside the Payload, what happens when the server checks the Signature?
3. Why do we use a `secretKey` to sign the token? What prevents the client from forging their own token?

---

## 🛑 STOP

Reply with your exercise code and quiz answers before moving to Lesson 2.