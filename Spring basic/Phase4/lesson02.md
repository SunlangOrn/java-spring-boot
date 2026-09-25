# 📘 Phase 4, Lesson 2: The Security Filter Chain & UserDetailsService

---

## 🎯 Goal
- ✅ Configure Spring Security to allow public routes (like registration) and protect private routes.
- ✅ Tell Spring Security how to find our users in the PostgreSQL database.
- ✅ Understand how the "Bouncer" checks IDs at the door.

---

## 🧠 The Big Picture

Right now, Spring Security blocks **everything** by default. We need to give it a rulebook. 

We do this with a **SecurityFilterChain**. Think of it as a series of checkpoints at the entrance of our club:
1. **Checkpoint 1 (Public Routes):** "Are you going to `/api/auth/register`? Go straight in, no ID needed."
2. **Checkpoint 2 (Protected Routes):** "Are you going to `/api/v1/products`? Show me your ID (Username/Password)."

To check the ID at Checkpoint 2, Spring needs to know how to look up the user in our database. That is the job of the **UserDetailsService**. It is the bouncer who takes the username, goes to the database, and brings back the user's details and password hash.

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **SecurityFilterChain** | The rulebook that defines which URLs are public and which require login. |
| **UserDetailsService** | An interface we implement to tell Spring how to load a user from our database. |
| **UserDetails** | A special Spring object that holds the username, password hash, and roles. |
| **AuthenticationManager** | The core engine that actually checks if the provided password matches the database hash. |

---

## 🛠️ Step 1: Create the Security Configuration

### What we're doing:
Create a configuration class to define our security rules and expose the `AuthenticationManager`.

### The Code:
Create file: `src/main/java/com/example/demo/config/SecurityConfig.java`

```java
package com.example.demo.config;

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
@EnableWebSecurity // Turns on Spring Security's web features
public class SecurityConfig {

    // 1. The "Blender" for passwords (Move this from DemoApplication if it's there)
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // 2. Expose the AuthenticationManager so we can use it in our Login endpoint later
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    // 3. The Rulebook (The Checkpoints)
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF because we are building a stateless REST API
            .csrf(csrf -> csrf.disable())
            
            // Define the rules for URLs
            .authorizeHttpRequests(auth -> auth
                // PUBLIC: Anyone can access auth endpoints (register/login)
                .requestMatchers("/api/auth/**").permitAll()
                
                // PUBLIC: Anyone can GET products or categories
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/categories/**").permitAll()
                
                // PROTECTED: Everything else requires authentication
                .anyRequest().authenticated()
            )
            
            // Use HTTP Basic Authentication for Phase 4 (Username/Password in header)
            // Note: We will replace this with JWT in Phase 5!
            .httpBasic(httpBasic -> {});

        return http.build();
    }
}
```

### 📝 After the Code - What Just Happened?
- `@EnableWebSecurity`: Activates Spring Security.
- `csrf.disable()`: CSRF protection is for traditional websites with cookies. For REST APIs that use tokens or basic auth, we turn it off.
- `requestMatchers(...).permitAll()`: These URLs are public. The bouncer lets anyone through.
- `anyRequest().authenticated()`: Any URL not explicitly listed above requires the user to be logged in.
- `httpBasic()`: Tells Spring to accept `Authorization: Basic base64(username:password)` headers. (Again, we will upgrade to JWT in Phase 5).

### 📦 Imports to Remember
```java
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
```

---

## 🛠️ Step 2: Implement UserDetailsService

### What we're doing:
Create a service that Spring Security will call every time someone tries to log in. It fetches the user from the DB and converts them into a Spring `UserDetails` object.

### The Code:
Create file: `src/main/java/com/example/demo/service/CustomUserDetailsService.java`

```java
package com.example.demo.service;

import com.example.demo.entity.User;
import com.example.demo.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 1. Find the user in our database
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        // 2. Convert our Role entities into Spring Security "Authorities"
        // Spring expects roles to start with "ROLE_" (which we already did in our DB)
        var authorities = user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority(role.getName()))
                .collect(Collectors.toList());

        // 3. Return Spring's built-in UserDetails object
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

### 📝 After the Code - What Just Happened?
- `implements UserDetailsService`: This is the magic contract. Spring Security sees this and says, "Ah! When a user tries to log in, I will call `loadUserByUsername`."
- `SimpleGrantedAuthority`: This is how Spring represents Roles internally.
- `User.builder()`: We build Spring's internal user object. We pass our database password hash. Spring handles the BCrypt matching automatically behind the scenes!

### 📦 Imports to Remember
```java
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
```

---

## 🧪 Run & Test

1. Restart your app.
2. Try to access a protected endpoint without logging in:
   ```bash
   curl -X POST http://localhost:8081/api/v1/products \
   -H "Content-Type: application/json" \
   -d '{"name": "Secret", "price": 10, "categoryId": 1}'
   ```
   **Expected:** `401 Unauthorized`. The bouncer stopped you!

3. Try to access it WITH the admin credentials you created in Lesson 1's exercise:
   ```bash
   curl -u admin:admin123 -X POST http://localhost:8081/api/v1/products \
   -H "Content-Type: application/json" \
   -d '{"name": "Secret", "price": 10, "categoryId": 1}'
   ```
   **Expected:** `201 Created`. You passed the bouncer!

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `401 Unauthorized` even with correct password | The password in the DB is not hashed, or you typed the wrong password. | Ensure your registration logic uses `passwordEncoder.encode()`. |
| `Circular Dependency` error | You injected `UserDetailsService` into `SecurityConfig`. | Keep them separate. Spring handles the wiring automatically. |
| `User not found` | The username you are passing in curl doesn't exist in the DB. | Check the `users` table in pgAdmin. |

---

##