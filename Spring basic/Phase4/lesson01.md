# 📘 Phase 4, Lesson 1: Authentication, Authorization & Password Hashing

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Core Concepts (Analogies)](#-the-core-concepts-analogies)
- [Why We NEVER Store Plain-Text Passwords](#-why-we-never-store-plain-text-passwords)
- [Step 1: Add Spring Security](#-step-1-add-spring-security)
- [Step 2: Design the Database Schema](#-step-2-design-the-database-schema)
- [Step 3: Create the JPA Entities](#-step-3-create-the-jpa-entities)
- [Step 4: Test Password Hashing (BCrypt)](#-step-4-test-password-hashing-bcrypt)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand the difference between Authentication and Authorization.
- ✅ Understand why and how we hash passwords using BCrypt.
- ✅ Design a professional database schema for Users and Roles.
- ✅ Create the JPA Entities for the security system.

---

## 🏢 The Core Concepts (Analogies)

Before writing code, you must understand the two pillars of security. Imagine your application is a **Corporate Office Building**.

### 1. Authentication (Who are you?)
- **Analogy:** Scanning your ID badge at the front door. 
- **Meaning:** The system verifies your identity. You prove you are "John" by providing a username and password.
- **Spring Security Term:** `Authentication`

### 2. Authorization (What are you allowed to do?)
- **Analogy:** Your ID badge lets you into the main lobby and your office on the 3rd floor, but it **blocks** you from entering the CEO's office or the Server Room.
- **Meaning:** The system checks your permissions. Even though you are authenticated as "John", you are not allowed to delete all products.
- **Spring Security Terms:** `Authorization`, `Roles`, `Authorities`

**Rule of Thumb:** Authentication always happens *before* Authorization. You must prove who you are before the system checks what you can do.

---

## 🔒 Why We NEVER Store Plain-Text Passwords

Imagine you store passwords in the database like this:
```text
ID | Username | Password
1  | john     | mySecretPassword123
```
If a hacker steals your database, they instantly have everyone's passwords. (And because people reuse passwords, the hacker can now log into their bank accounts!)

### The Solution: Hashing (The Blender Analogy)
A **Hash Function** (like BCrypt) is like a meat grinder. 
1. You put a steak (the password) into the grinder.
2. It comes out as ground meat (the hash).
3. **Crucial:** You *cannot* turn the ground meat back into a steak. It is a **one-way** process.

When John registers:
1. John types: `mySecretPassword123`
2. Spring puts it through the BCrypt grinder.
3. The database stores the ground meat: `$2a$10$N9qo8uLOickgx2ZMRZoMye...`

When John logs in:
1. John types: `mySecretPassword123`
2. Spring puts it through the grinder again.
3. Spring compares the new ground meat with the ground meat in the database.
4. If they match, John is in!

---

## 🛠️ Step-by-Step Build

### Step 1: Add Spring Security
Open your `pom.xml` and add the Spring Security starter:

```xml
<dependencies>
    <!-- ... your other dependencies ... -->

    <!-- Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

⚠️ **WARNING:** As soon as you add this dependency and restart your app, **ALL** your existing endpoints (like `/api/products`) will be blocked and require a login! Spring Boot's default security is "block everything". Don't panic, we will configure it in the next lesson. For now, just add the dependency.

---

### Step 2: Design the Database Schema

We need three tables: `users`, `roles`, and a mapping table `user_roles` (because a User can have many Roles, and a Role can belong to many Users -> **Many-to-Many**).

Create `src/main/resources/db/migration/V7__create_users_and_roles.sql`:

```sql
-- 1. Create the Roles table
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

-- 2. Create the Users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL, -- Stores the BCrypt hash, NOT the plain password!
    email VARCHAR(100) NOT NULL UNIQUE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 3. Create the Many-to-Many mapping table
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
);

-- 4. Insert default roles
INSERT INTO roles (name) VALUES ('ROLE_USER');
INSERT INTO roles (name) VALUES ('ROLE_ADMIN');
```

---

### Step 3: Create the JPA Entities

Create the `entity/security` package to keep security entities organized.

**Role Entity:**
```java
// src/main/java/com/example/product/entity/security/Role.java
package com.example.product.entity.security;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "roles")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String name;
}
```

**User Entity:**
```java
// src/main/java/com/example/product/entity/security/User.java
package com.example.product.entity.security;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "users")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    // This column stores the BCrypt hash (e.g., "$2a$10$N9qo8u...")
    @Column(name = "password_hash", nullable = false)
    private String passwordHash;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(name = "is_active", nullable = false)
    private Boolean isActive = true;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();

    // MANY Users can have MANY Roles
    // We use a Set because roles should be unique for a user
    @ManyToMany(fetch = FetchType.EAGER) // EAGER is okay here because a user rarely has more than 3-4 roles
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    @Builder.Default // Required by Lombok @Builder to initialize collections
    private Set<Role> roles = new HashSet<>();
}
```

**Line-by-line explanation of `@ManyToMany`:**
- `@ManyToMany`: Tells JPA this is a many-to-many relationship.
- `fetch = FetchType.EAGER`: When we load a User, we immediately load their Roles. (We use EAGER here because a user only has 1 or 2 roles, unlike Products in a Category which could be thousands).
- `@JoinTable`: Tells JPA exactly which table and columns to use to link them (the `user_roles` table we created in Flyway).

---

### Step 4: Test Password Hashing (BCrypt)

Before we build the login endpoint, let's prove BCrypt works. We will create a simple configuration class to expose the `PasswordEncoder` as a Spring Bean.

```java
// src/main/java/com/example/product/config/SecurityConfig.java
package com.example.product.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class SecurityConfig {

    // This creates a "grinder" (BCrypt) that Spring can inject anywhere
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

Now, let's write a quick test in our `ProductApplication` (just for learning, we will delete this later) to see the hash:

```java
// src/main/java/com/example/product/ProductApplication.java
package com.example.product;

import com.example.product.config.SecurityConfig;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.security.crypto.password.PasswordEncoder;

@SpringBootApplication
public class ProductApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProductApplication.class, args);
    }

    // This runs ONCE when the application starts
    @Bean
    CommandLineRunner testBCrypt(PasswordEncoder encoder) {
        return args -> {
            String plainPassword = "mySecretPassword123";
            
            // Put it in the grinder
            String hashedPassword = encoder.encode(plainPassword);
            
            System.out.println("=================================");
            System.out.println("Plain: " + plainPassword);
            System.out.println("Hash:  " + hashedPassword);
            System.out.println("=================================");
            
            // Check if a raw password matches the hash
            boolean isMatch = encoder.matches("mySecretPassword123", hashedPassword);
            System.out.println("Does it match? " + isMatch);
        };
    }
}
```

---

## 🚀 Run and Test

```bash
./mvnw clean spring-boot:run
```

**Watch the console!** You will see Flyway apply `V7`, creating the tables. 
Then, you will see the output from our `CommandLineRunner`:

```text
=================================
Plain: mySecretPassword123
Hash:  $2a$10$E.yjMG2qV5zXpL1wZ8qOe.uX9qJ5yT1vK8mN3bC4dE5fG6hJ7kL8
=================================
Does it match? true
```

*Note: If you restart the app, the Hash will be **different** every time! This is because BCrypt adds a random "salt" to the password before hashing it. This prevents hackers from using "Rainbow Tables" (pre-computed hashes). But `encoder.matches()` will still return `true`!*

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| `Relation "users" does not exist` | Flyway didn't run. | Check that `V7__create_users_and_roles.sql` is in `src/main/resources/db/migration/`. |
| `Invalid column name password_hash` | Entity field name doesn't match DB column. | Ensure `@Column(name = "password_hash")` is on the `passwordHash` field. |
| `Circular dependency` when injecting `PasswordEncoder` | You tried to inject it in the wrong place. | Only inject it in Services or Configuration classes, not in Entities. |

---

## 🛠️ Exercise

1. Create a `UserRepository` interface that extends `JpaRepository<User, Long>`.
2. Add a method to find a user by username: `Optional<User> findByUsername(String username);`
3. Create a `RoleRepository` interface.
4. In your `ProductApplication` `CommandLineRunner`, use the repositories to:
   - Find the `ROLE_ADMIN` from the database.
   - Create a new `User` (username: "admin", email: "admin@test.com", password: use the `PasswordEncoder` to hash "admin123").
   - Add the `ROLE_ADMIN` to the user's roles set.
   - Save the user using `userRepository.save(user)`.
5. Restart the app and verify the user is created in the database (you can check via Docker: `docker exec -it product-db psql -U postgres -d product_db -c "SELECT * FROM users;"`).

---

## 🧠 Quiz

1. What is the difference between **Authentication** and **Authorization**?
2. Why is a Hash function called a "one-way" process?
3. Why does BCrypt generate a **different** hash string every time we hash the exact same password?
4. In the `User` entity, why did we use `FetchType.EAGER` for the `roles` collection, but `FetchType.LAZY` for the `category` in the `Product` entity?

---

## 🛑 STOP

**Do not move forward.** 

Reply with:
1. Confirmation that you added Spring Security, ran the migrations, and saw the BCrypt hash in the console.
2. Your code for the **Exercise** (Repositories and saving the admin user).
3. Your answers to the 4 quiz questions.

Once you reply, we will move to **Phase 4, Lesson 2: Building the Registration and Login Endpoints**, where we will finally allow users to sign up and log in!