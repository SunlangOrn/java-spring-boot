# 📘 Phase 4, Lesson 2: The Security Filter Chain & UserDetailsService

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Concept: The Club Checkpoints](#-the-concept-the-club-checkpoints)
- [Step 1: Configure the SecurityFilterChain](#-step-1-configure-the-securityfilterchain)
- [Step 2: Implement UserDetailsService](#-step-2-implement-userdetailsservice)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand how Spring Security intercepts HTTP requests.
- ✅ Configure the `SecurityFilterChain` to allow public and protected routes.
- ✅ Implement `UserDetailsService` to tell Spring how to load users from the database.

---

## 🏢 The Concept: The Club Checkpoints

Imagine the **Security Filter Chain** as a series of checkpoints at the entrance of our VIP nightclub:
1. **Checkpoint 1 (CORS/CSRF)**: Checks if the request is from an allowed website.
2. **Checkpoint 2 (Public Routes)**: "Are you going to the public bar (`/api/auth/**`)? If yes, go straight in."
3. **Checkpoint 3 (Authentication)**: "Are you going to the VIP lounge (`/api/products`)? Show me your ID (Username/Password)."
4. **Checkpoint 4 (Authorization)**: "Okay, you're in. But are you allowed in the *VIP VIP* room? (`@PreAuthorize`)"

To pass Checkpoint 3, Spring needs to know how to check the ID. That's where **`UserDetailsService`** comes in. It is the bouncer who looks at the username, goes to the database, and brings back the user's details and password hash.

---

## 🛠️ Step-by-Step Build

### Step 1: Configure the SecurityFilterChain
Update your `SecurityConfig.java` to define the rules.

```java
// src/main/java/com/example/product/config/SecurityConfig.java
package com.example.product.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity // Enables Spring Security's web security support
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // Expose AuthenticationManager for later use (Login endpoint)
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF for REST APIs (we use stateless tokens in Phase 5)
            .csrf(csrf -> csrf.disable())
            
            // Define Authorization Rules
            .authorizeHttpRequests(auth -> auth
                // 1. Public routes (Anyone can access)
                .requestMatchers(HttpMethod.POST, "/api/auth/register").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                
                // 2. Protected routes (Must be authenticated)
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll() // Let's keep GET public for now
                .requestMatchers(HttpMethod.POST, "/api/products/**").hasRole("ADMIN") // Only ADMIN can create
                .requestMatchers(HttpMethod.PUT, "/api/products/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/products/**").hasRole("ADMIN")
                
                // 3. Any other request must be authenticated
                .anyRequest().authenticated()
            )
            
            // Use HTTP Basic Authentication for Phase 4 (Phase 5 will use JWT)
            .httpBasic(httpBasic -> {});

        return http.build();
    }
}
```

### Step 2: Implement UserDetailsService
Spring Security doesn't know about our `User` entity. We must translate our `User` into Spring's `UserDetails` object.

```java
// src/main/java/com/example/product/service/CustomUserDetailsService.java
package com.example.product.service;

import com.example.product.entity.security.User;
import com.example.product.repository.UserRepository;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.stream.Collectors;

@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 1. Fetch user from DB
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        // 2. Convert our Role entities into Spring Security "Authorities"
        // Spring expects roles to start with "ROLE_" (e.g., ROLE_ADMIN)
        var authorities = user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority(role.getName()))
                .collect(Collectors.toList());

        // 3. Return Spring's UserDetails object
        return org.springframework.security.core.userdetails.User.builder()
                .username(user.getUsername())
                .password(user.getPasswordHash()) // Spring will automatically use our PasswordEncoder to check this!
                .authorities(authorities)
                .accountExpired(!user.getIsActive())
                .accountLocked(!user.getIsActive())
                .credentialsExpired(false)
                .disabled(!user.getIsActive())
                .build();
    }
}
```

**Line-by-line explanation:**
- `loadUserByUsername`: Spring calls this method when a user tries to log in.
- `SimpleGrantedAuthority`: This is how Spring represents Roles/Permissions internally.
- `User.builder()`: We build Spring's internal `UserDetails` object, passing our database password hash. Spring handles the BCrypt matching automatically!

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

**Test 1: Access public endpoint (No login required)**
```bash
curl http://localhost:8080/api/products
```
*Expected:* 200 OK. Returns products.

**Test 2: Access protected endpoint without login**
```bash
curl -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "Secret Item", "price": 10.0, "stock": 5, "categoryId": 1}'
```
*Expected:* `401 Unauthorized`. (The bouncer stopped you!)

**Test 3: Access protected endpoint WITH login (HTTP Basic Auth)**
*(Assuming you created the 'admin' user in the Phase 4, Lesson 1 exercise with password 'admin123')*
```bash
curl -u admin:admin123 -X POST http://localhost:8080/api/products \
-H "Content-Type: application/json" \
-d '{"name": "Secret Item", "price": 10.0, "stock": 5, "categoryId": 1}'
```
*Expected:* `201 Created`. You passed the bouncer!

---

## 🚨 Common Errors
1. **`401 Unauthorized` even with correct password**: You might have stored the plain text password in the DB instead of the BCrypt hash. Ensure your registration logic uses `passwordEncoder.encode()`.
2. **`403 Forbidden` when trying to POST**: The user doesn't have the `ROLE_ADMIN` authority. Check the `user_roles` table in your database.
3. **`Circular Dependency` error**: You injected `UserDetailsService` into `SecurityConfig`. *Fix:* Keep them separate. Spring handles the wiring automatically.

---

## 🛠️ Exercise
1. Create a new user in the database with the role `ROLE_USER` (not ADMIN).
2. Try to access `POST /api/products` with this user's credentials using `curl -u username:password`.
3. Verify you get a `403 Forbidden` (Authenticated, but not Authorized).

---

## 🧠 Quiz
1. What is the purpose of the `SecurityFilterChain`?
2. What does `UserDetailsService` do?
3. Why do we disable CSRF in this configuration?

---

## 🛑 STOP
Reply with your exercise results and quiz answers before moving to Lesson 3.