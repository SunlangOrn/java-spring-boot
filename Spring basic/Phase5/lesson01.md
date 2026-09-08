# 📘 Phase 5, Lesson 1: Introduction to JWT & Generating Tokens

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem with HTTP Basic Auth](#-the-problem-with-http-basic-auth)
- [The Solution: The "Festival Wristband" Analogy](#-the-solution-the-festival-wristband-analogy)
- [What is a JWT? (The Anatomy)](#-what-is-a-jwt-the-anatomy)
- [Step 1: Add JWT Dependencies](#-step-1-add-jwt-dependencies)
- [Step 2: Configure the Secret Key](#-step-2-configure-the-secret-key)
- [Step 3: Create the JwtService](#-step-3-create-the-jwtservice)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand why we need JWTs instead of sending passwords every time.
- ✅ Understand the 3 parts of a JWT (Header, Payload, Signature).
- ✅ Add the JJWT library to the project.
- ✅ Write a service to generate and decode JWTs.

---

## 🚨 The Problem with HTTP Basic Auth

Right now, when "John" wants to buy a product, his app sends this header:
`Authorization: Basic am9objpzZWN1cmVQYXNzd29yZDEyMw==` (which is just his username and password encoded in Base64).

**Why is this bad?**
1. **Inefficient:** The server has to query the database and run BCrypt (which is intentionally slow) on *every single request*.
2. **Security Risk:** If the connection is intercepted (even briefly), the hacker gets the actual password.
3. **Stateful:** The server has to keep track of who is logged in (Sessions).

---

## 🎟️ The Solution: The "Festival Wristband" Analogy

Imagine you go to a 3-day Music Festival.

**Day 1 (HTTP Basic Auth):**
Every time you want to buy a drink, you show your Passport (Password). The bartender has to call the main office (Database) to verify your Passport is real. This takes 5 minutes per drink.

**Day 2 (JWT):**
On Day 1, you show your Passport at the main gate. The security guard checks it, and puts a **glowing, tamper-proof wristband** on your wrist. 
Now, every time you buy a drink, the bartender just **looks at the wristband**. 
- Is it glowing? (Valid signature)
- Does it say "VIP"? (Payload/Claims)
- Did it expire? (Expiration date)

The bartender doesn't need to call the main office. The wristband itself is the proof. **This is a JWT.**

---

## 🧬 What is a JWT? (The Anatomy)

A JWT is just a long String that looks like this:
`eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwicm9sZSI6IkFETUlOIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQsW5c`

It is divided into **3 parts**, separated by dots (`.`):

1. **Header (Red):** "What algorithm was used to sign this?" (e.g., HS256)
2. **Payload (Purple):** The "Data" (e.g., `sub: "john", role: "ADMIN", exp: 1690000000`). This is called **Claims**.
3. **Signature (Blue):** A mathematical hash of the Header + Payload + a **Secret Key**. If anyone changes the Payload (e.g., changes "USER" to "ADMIN"), the Signature becomes invalid, and the server rejects it.

*Pro Tip: You can decode any JWT at [jwt.io](https://jwt.io) to see its contents!*

---

## 🛠️ Step-by-Step Build

### Step 1: Add JWT Dependencies

We will use the standard `jjwt` library by Jwtoken. Open your `pom.xml`:

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

---

### Step 2: Configure the Secret Key

⚠️ **CRITICAL SECURITY RULE:** Never hardcode your JWT Secret Key in Java code! If you push your code to GitHub, hackers will steal your key and forge their own admin tokens.

Add it to `application.yml`:

```yaml
# src/main/resources/application.yml
app:
  jwt:
    # This must be at least 256 bits (32 characters) for HS256 algorithm.
    # In production, generate a random 64-character string!
    secret: ${JWT_SECRET:mySuperSecretKeyForDevelopmentOnlyChangeInProduction123!}
    expiration: 86400000      # 1 day in milliseconds (24 * 60 * 60 * 1000)
    refresh-expiration: 604800000 # 7 days in milliseconds
```

---

### Step 3: Create the JwtService

This service will act as the "Festival Security Guard". It knows how to create the wristband (generate token) and how to check if the wristband is fake (validate token).

```java
// src/main/java/com/example/product/security/JwtService.java
package com.example.product.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
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
        // The secret key must be encoded in Base64 for HS256
        // For simplicity in this lesson, we convert the string directly to bytes.
        // In a real production app, you would use a Base64 encoded byte array.
        byte[] keyBytes = secretKey.getBytes();
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

**Line-by-line breakdown of the magic:**
- `Jwts.builder()`: Starts creating the token.
- `.subject()`: Sets the "sub" claim (the username).
- `.expiration()`: Sets the exact millisecond the token dies.
- `.signWith(getSignInKey(), Jwts.SIG.HS256)`: Takes the Header + Payload, and mathematically hashes them using our `secretKey`. This creates the Signature.
- `.compact()`: Squashes it all together into the final `eyJ...` string.

---

## 🚀 Run and Test

Let's write a quick test in our `ProductApplication` to prove we can generate a token and read it.

```java
// src/main/java/com/example/product/ProductApplication.java
package com.example.product;

import com.example.product.security.JwtService;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;

@SpringBootApplication
public class ProductApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProductApplication.class, args);
    }

    @Bean
    CommandLineRunner testJWT(JwtService jwtService) {
        return args -> {
            // 1. Create a fake UserDetails object (pretend this came from the DB)
            UserDetails fakeUser = User.builder()
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
}
```

Run the app:
```bash
./mvnw spring-boot:run
```

**Look at the console!** You will see a massive string. 
Copy that entire string, go to **[jwt.io](https://jwt.io)**, and paste it into the "Encoded" box on the left. 
Watch as it magically decodes the Header, Payload (showing `john_doe`), and the Signature on the right!

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| `The signing key's size is X bits which is smaller than required...` | Your `app.jwt.secret` in `application.yml` is too short. | HS256 requires at least 256 bits (32 characters). Make your secret longer. |
| `JWT expired at...` | You are trying to validate a token that was generated in the past with a 0ms expiration. | Check your `app.jwt.expiration` in `application.yml`. |

---

## 🛠️ Exercise

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

**Do not move forward.** 

Reply with:
1. Confirmation that you generated the token and successfully decoded it on jwt.io.
2. Your code for the **Exercise** (generating a token with custom role claims).
3. Your answers to the 3 quiz questions.

Once you reply, we will move to **Phase 5, Lesson 2: The JWT Authentication Filter**, where we will replace HTTP Basic Auth and make Spring Security accept these tokens!