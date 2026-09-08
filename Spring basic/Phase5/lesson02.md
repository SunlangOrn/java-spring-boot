# 📘 Phase 5, Lesson 2: The JWT Authentication Filter

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Concept: The Wristband Scanner](#-the-concept-the-wristband-scanner)
- [Step 1: Create the JWT Filter](#-step-1-create-the-jwt-filter)
- [Step 2: Update the SecurityFilterChain](#-step-2-update-the-securityfilterchain)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand how Spring Security intercepts requests to check for JWTs.
- ✅ Create a custom `OncePerRequestFilter` to extract and validate tokens.
- ✅ Set the `SecurityContextHolder` so the rest of the app knows who the user is.
- ✅ Remove HTTP Basic Auth and switch entirely to JWT.

---

## 🎟️ The Concept: The Wristband Scanner

In Lesson 1, we learned how to **create** the wristband (JWT). Now, we need a scanner at the door of every VIP room (endpoint). 

When a request arrives, the **JWT Filter** looks at the `Authorization` header. 
1. Is there a wristband (Token)? 
2. Is it glowing (Valid Signature)? 
3. Did it expire? 
4. Who does it belong to?

If everything is correct, the filter creates an **Authentication Object** and hands it to Spring Security. Spring Security then says, "Okay, this user is logged in," and lets them through to the Controller.

---

## 🛠️ Step-by-Step Build

### Step 1: Create the JWT Filter

Create a new package `security` and add this class:

```java
// src/main/java/com/example/product/security/JwtAuthenticationFilter.java
package com.example.product.security;

import com.example.product.service.CustomUserDetailsService;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final CustomUserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtService jwtService, CustomUserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain
    ) throws ServletException, IOException {

        // 1. Get the Authorization header
        final String authHeader = request.getHeader("Authorization");
        final String jwt;
        final String username;

        // 2. Check if header exists and starts with "Bearer "
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            // No token? Just continue to the next filter.
            filterChain.doFilter(request, response);
            return;
        }

        // 3. Extract the token (remove "Bearer " prefix)
        jwt = authHeader.substring(7);

        // 4. Extract username from the token
        username = jwtService.extractUsername(jwt);

        // 5. If we have a username, AND the user is not already authenticated in this request
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            
            // 6. Load the full user details from the database
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);

            // 7. Validate the token against the user details
            if (jwtService.isTokenValid(jwt, userDetails)) {
                
                // 8. Create the Authentication object
                UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null, // We don't need the password here, the token is the proof
                        userDetails.getAuthorities() // Roles and Permissions!
                );
                
                // Add extra details (like IP address, session ID)
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                
                // 9. Update the Security Context. Spring Security now knows this user is logged in!
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        // 10. Continue to the next filter in the chain
        filterChain.doFilter(request, response);
    }
}
```

**Line-by-line breakdown:**
- `OncePerRequestFilter`: Ensures this filter runs exactly once per request.
- `authHeader.substring(7)`: The header looks like `Bearer eyJhbG...`. We skip the first 7 characters (`Bearer `) to get just the token.
- `SecurityContextHolder.getContext().setAuthentication(...)`: This is the magic line. We are telling Spring, "Trust me, this user is who they say they are."

### Step 2: Update the SecurityFilterChain

Now we must tell Spring Security to use our new filter, and we must **disable HTTP Basic Auth**.

```java
// src/main/java/com/example/product/config/SecurityConfig.java
package com.example.product.config;

import com.example.product.security.JwtAuthenticationFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy; // ← NEW IMPORT
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter; // ← NEW IMPORT

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter) {
        this.jwtAuthFilter = jwtAuthFilter;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            
            // CRITICAL: Tell Spring we are using JWTs, so DO NOT create server-side sessions!
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll() // Login/Register are public
                .requestMatchers("/api/products/**").permitAll() // Keep products public for now to test easily
                .anyRequest().authenticated()
            )
            
            // REMOVE .httpBasic()! We don't need it anymore.
            
            // Add our JWT filter BEFORE Spring's default username/password filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

**Why `SessionCreationPolicy.STATELESS`?**
Because JWTs are stateless. The server doesn't need to remember who is logged in in a `HttpSession`. The client sends the token on every request. This makes your API scalable!

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

Since we haven't built the `/login` endpoint yet, we need to generate a token manually to test the filter. 

1. Start the app. The `CommandLineRunner` from Lesson 1 will print a JWT to the console. Copy it!
2. Test accessing a protected endpoint (let's temporarily change `/api/products/**` to require authentication in `SecurityConfig` to test it properly, or just use a new endpoint).

Let's create a quick "Who Am I?" endpoint to test:

```java
// src/main/java/com/example/product/controller/UserController.java
package com.example.product.controller;

import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/me")
    public String whoAmI(Authentication authentication) {
        return "Hello, " + authentication.getName() + "! Your roles are: " + authentication.getAuthorities();
    }
}
```

**Test 1: Without Token**
```bash
curl http://localhost:8080/api/users/me
```
*Expected:* `403 Forbidden` (or 401).

**Test 2: With Token**
```bash
curl -H "Authorization: Bearer <PASTE_YOUR_TOKEN_HERE>" http://localhost:8080/api/users/me
```
*Expected:* `Hello, john_doe! Your roles are: [ROLE_USER]`

**The filter worked!** Spring Security now knows who you are based purely on the JWT.

---

## 🚨 Common Errors
1. **`403 Forbidden` even with a valid token**: Your `JwtAuthenticationFilter` is not being executed. *Fix:* Ensure you added `.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)` in `SecurityConfig`.
2. **`Full authentication is required`**: The token was invalid or expired, so the filter didn't set the `SecurityContext`. Check the console for errors from `JwtService`.

---

## 🛠️ Exercise
1. Generate a token for the `admin` user (who has `ROLE_ADMIN`).
2. Call the `/api/users/me` endpoint with the admin's token.
3. Verify that the response includes `ROLE_ADMIN` in the authorities list.

---

## 🧠 Quiz
1. Why do we set `SessionCreationPolicy.STATELESS`?
2. What does `authHeader.substring(7)` do?
3. Why do we add our filter *before* `UsernamePasswordAuthenticationFilter`?

---

## 🛑 STOP
Reply with your exercise results and quiz answers before moving to Lesson 3.