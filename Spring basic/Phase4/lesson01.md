# 📘 Phase 4, Lesson 1: Authentication, Authorization & Password Hashing

---

## 🎯 Goal
- ✅ Understand the difference between Authentication and Authorization.
- ✅ Understand why we NEVER store plain-text passwords.
- ✅ Create `User` and `Role` database tables.
- ✅ Learn how to hash passwords using BCrypt.

---

## 🧠 The Big Picture

### 1. Authentication vs. Authorization (The Club Analogy)
Imagine a VIP nightclub:
- **Authentication (AuthN)**: The bouncer checks your ID at the door. *"Who are you?"* You prove you are "John" by showing your ID (Username + Password).
- **Authorization (AuthZ)**: Once inside, the bouncer checks your wristband. *"What are you allowed to do?"* Your wristband lets you into the main bar, but blocks you from the VIP room. 

**Rule:** Authentication ALWAYS happens before Authorization. You must prove who you are before the system checks what you can do.

### 2. Password Hashing (The Blender Analogy)
We **NEVER** save passwords as plain text (e.g., "mySecret123"). If a hacker steals the database, they have everyone's passwords.

Instead, we use a **Hash Function** (like BCrypt). 
- Think of it like a meat grinder. You put a steak (the password) in, and it comes out as ground meat (the hash).
- **Crucial:** You cannot turn the ground meat back into a steak. It is a **one-way** process.
- When John logs in, we grind the password he typed. If the ground meat matches the ground meat in the database, he is allowed in!

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **Authentication** | Verifying *who* the user is (Login). |
| **Authorization** | Verifying *what* the user is allowed to do (Permissions/Roles). |
| **Hash** | A one-way mathematical transformation of data (like a fingerprint). |
| **Salt** | Random data added to a password before hashing, so two identical passwords look completely different in the database. |
| **BCrypt** | The industry-standard hashing algorithm used by Spring Security. |

---

## 🛠️ Step 1: Add Spring Security Dependency

### What we're doing:
Tell Spring Boot we want to use its built-in security features.

### The Code:
**Update:** `pom.xml`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

### 📝 After the Code - What Just Happened?
Spring Security is now active. 
⚠️ **WARNING:** By default, Spring Security locks down **everything**. If you restart your app right now, all your `/api/v1/products` endpoints will return `401 Unauthorized`. Don't panic! We will configure this in the next lesson. For now, we just need the library to get the password hasher.

### 📦 Imports to Remember
None needed for `pom.xml`, but remember to run `mvn clean install` or let your IDE reload the Maven dependencies.

---

## 🛠️ Step 2: Create the Database Schema (Flyway)

### What we're doing:
Create tables for `users`, `roles`, and a linking table `user_roles` (because one user can have many roles, and one role can belong to many users).

### The Code:
Create file: `src/main/resources/db/migration/V6__create_users_and_roles.sql`

```sql
-- Step A: Create the roles table
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

-- Step B: Create the users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL, -- Stores the BCrypt hash, NOT the plain password!
    email VARCHAR(100) NOT NULL UNIQUE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITHOUT TIME ZONE,
    updated_at TIMESTAMP WITHOUT TIME ZONE
);

-- Step C: Create the Many-to-Many linking table
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
);

-- Step D: Insert default roles
INSERT INTO roles (name) VALUES ('ROLE_USER');
INSERT INTO roles (name) VALUES ('ROLE_ADMIN');
```

### 📝 After the Code - What Just Happened?
- **Step A & B**: Creates the main tables. Notice the column is named `password_hash` to remind us it's not a plain password.
- **Step C**: Creates the linking table. `ON DELETE CASCADE` means if a user is deleted, their role links are automatically deleted too.
- **Step D**: Pre-loads two roles into the database so we can assign them to users immediately.

### 💡 Note
> Spring Security expects role names to start with `ROLE_` (e.g., `ROLE_USER`, `ROLE_ADMIN`). Always follow this convention to avoid confusing bugs later!

---

## 🛠️ Step 3: Create the User and Role Entities

### What we're doing:
Create the Java classes that represent these new tables.

### The Code:

**File 1:** `src/main/java/com/example/demo/entity/Role.java`
```java
package com.example.demo.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "roles")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Role extends BaseEntity {
    
    @Column(nullable = false, unique = true, length = 50)
    private String name;
}
```

**File 2:** `src/main/java/com/example/demo/entity/User.java`
```java
package com.example.demo.entity;

import jakarta.persistence.*;
import lombok.*;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "users")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class User extends BaseEntity {

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(name = "password_hash", nullable = false, length = 255)
    private String passwordHash;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(name = "is_active", nullable = false)
    private Boolean isActive = true;

    // A User can have MANY Roles
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    @Builder.Default
    private Set<Role> roles = new HashSet<>();
}
```

### 📝 After the Code - What Just Happened?
- Both extend `BaseEntity`, so they automatically get `id`, `createdAt`, and `updatedAt`.
- `@ManyToMany(fetch = FetchType.EAGER)`: We use `EAGER` here (unlike `LAZY` in Phase 3) because a user typically only has 1 or 2 roles. We want to load them immediately when we load the user for authentication.
- `@Builder.Default`: Required by Lombok to ensure the `HashSet` is initialized properly when using the `@Builder`.

### 📦 Imports to Remember
```java
import jakarta.persistence.*; // Entity, Table, Column, ManyToMany, JoinTable, JoinColumn, FetchType
import java.util.HashSet;
import java.util.Set;
```

---

## 🛠️ Step 4: Test the Password Hasher (BCrypt)

### What we're doing:
Before we build the login logic, let's prove that BCrypt works by writing a tiny test that runs when the app starts.

### The Code:
**Update:** `src/main/java/com/example/demo/DemoApplication.java`

```java
package com.example.demo;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.context.annotation.Bean;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@SpringBootApplication
@EnableJpaAuditing
@EnableCaching
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    // 1. Create the "Blender" (PasswordEncoder) as a Spring Bean
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    // 2. Run a quick test when the app starts
    @Bean
    CommandLineRunner testBCrypt(PasswordEncoder encoder) {
        return args -> {
            String plainPassword = "mySecret123";
            
            // Put it in the blender
            String hashedPassword = encoder.encode(plainPassword);
            
            System.out.println("=================================");
            System.out.println("Plain:  " + plainPassword);
            System.out.println("Hashed: " + hashedPassword);
            System.out.println("=================================");
            
            // Check if a raw password matches the hash
            boolean isMatch = encoder.matches("mySecret123", hashedPassword);
            System.out.println("Does it match? " + isMatch);
        };
    }
}
```

### 📝 After the Code - What Just Happened?
- `@Bean public PasswordEncoder`: We tell Spring to create one `BCryptPasswordEncoder` and share it across the whole app.
- `CommandLineRunner`: This is a special Spring interface. Any code inside its `run` method executes exactly **once** right after the app starts.
- `encoder.matches()`: This is the magic. It takes the plain password the user typed, hashes it with the same "salt" stored in the database hash, and checks if they match.

### 💡 Note
> If you run the app multiple times, the **Hashed** string will be different every single time! This is because BCrypt adds a random "salt" every time. This is a **good thing**—it prevents hackers from using pre-computed "Rainbow Tables" to crack passwords. But `encoder.matches()` will always return `true` for the correct password.

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Relation "users" does not exist` | Flyway didn't run the migration | Check the file is named `V6__...` and is in `db/migration/` |
| `Invalid column name password_hash` | Entity field name doesn't match DB | Ensure `@Column(name = "password_hash")` is on the `passwordHash` field |
| Hashing plain text in the Controller | Bad architecture | **Never** hash in the Controller. Always hash in the Service layer. |

---

## ✏️ Exercise

1. Create `UserRepository.java` and `RoleRepository.java` extending `JpaRepository`.
2. In `UserRepository`, add this method: `Optional<User> findByUsername(String username);`
3. In `RoleRepository`, add this method: `Optional<Role> findByName(String name);`
4. Update the `CommandLineRunner` in `DemoApplication` to:
   - Find the `ROLE_ADMIN` using the repository.
   - Create a new `User` (username: "admin", email: "admin@test.com", password: hash "admin123" using the encoder).
   - Add `ROLE_ADMIN` to the user's `roles` set.
   - Save the user using `userRepository.save(user)`.
5. Restart the app and check pgAdmin to see the new user!

---

## 🧠 Quiz

1. In plain English, what is the difference between Authentication and Authorization?
2. Why is a Hash function called a "one-way" process?
3. Why does BCrypt generate a **different** hash string every time we hash the exact same password?

---

## 🛑 STOP

Reply with:
1. Confirmation that you saw the BCrypt test print in the console.
2. Your code for the Exercise (Repositories and saving the admin user).
3. Your answers to the 3 quiz questions.

Once you reply, we will move to **Phase 4, Lesson 2: The Security Filter Chain & UserDetailsService**, where we teach Spring how to actually log users in!