# 🛒 ShopCore Project - Lesson 1: Foundation & User Entities

---

## 🎯 Goal
- ✅ Set up the database schema for Users, Roles, and Permissions.
- ✅ Create the JPA Entities for the security system.
- ✅ Create the Repositories to interact with the database.

---

## 🧠 The Big Picture

Every e-commerce app needs to know **who** is using it. 
- **Customers** can browse products and buy things.
- **Admins** can add new products and manage orders.

Instead of hardcoding "Admin" or "Customer" into our code, we will build a flexible system using **Users**, **Roles**, and **Permissions**. This is the exact same professional pattern we learned in Phase 4, but now we are applying it to our real project!

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **Migration** | A version-controlled SQL script that creates or updates database tables. |
| **Entity** | A Java class that represents a database table. |
| **`@ManyToMany`** | A database relationship where many items of type A can link to many items of type B (e.g., A User can have many Roles, and a Role can belong to many Users). |

---

## 🛠️ Step 1: Create the Database Migration

### What we're doing:
Write the SQL script to create the `users`, `roles`, `permissions`, and their linking tables.

### The Code:
Create file: `src/main/resources/db/migration/V1__init_security_tables.sql`

```sql
-- 1. Create Roles table
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

-- 2. Create Permissions table
CREATE TABLE permissions (
    id BIGSERIAL PRIMARY KEY
    name VARCHAR(100) NOT NULL UNIQUE
);

-- 3. Create Users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITHOUT TIME ZONE,
    updated_at TIMESTAMP WITHOUT TIME ZONE
);

-- 4. Link Users to Roles
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
);

-- 5. Link Roles to Permissions
CREATE TABLE role_permissions (
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    PRIMARY KEY (role_id, permission_id),
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE
);

-- 6. Insert default data
INSERT INTO roles (name) VALUES ('ROLE_CUSTOMER'), ('ROLE_ADMIN');
INSERT INTO permissions (name) VALUES ('PRODUCT_READ'), ('PRODUCT_CREATE'), ('PRODUCT_UPDATE'), ('PRODUCT_DELETE'), ('ORDER_CREATE');

-- Give ADMIN all permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p WHERE r.name = 'ROLE_ADMIN';

-- Give CUSTOMER only read and order create permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p WHERE r.name = 'ROLE_CUSTOMER' AND p.name IN ('PRODUCT_READ', 'ORDER_CREATE');
```

### 📝 After the Code - What Just Happened?
- We created 3 main tables (`users`, `roles`, `permissions`) and 2 linking tables (`user_roles`, `role_permissions`).
- We pre-loaded the database with two roles (`ROLE_CUSTOMER`, `ROLE_ADMIN`) and assigned them specific permissions. This means our app is ready to use immediately!

---

## 🛠️ Step 2: Create the Permission Entity

### What we're doing:
Create the simplest entity first. A permission just has an ID and a name.

### The Code:
Create file: `src/main/java/com/example/demo/entity/Permission.java`

```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "permissions")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Permission extends BaseEntity {

    @Column(nullable = false, unique = true)
    private String name;
}
```

### 📝 After the Code - What Just Happened?
- It extends `BaseEntity`, so it automatically gets `id`, `createdAt`, and `updatedAt`.
- `@Column(unique = true)` ensures we can't accidentally create two permissions with the exact same name.

---

## 🛠️ Step 3: Create the Role Entity

### What we're doing:
Create the Role entity and link it to Permissions.

### The Code:
Create file: `src/main/java/com/example/demo/entity/Role.java`

```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.*;
import lombok.*;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "roles")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Role extends BaseEntity {

    @Column(nullable = false, unique = true, length = 50)
    private String name;

    // A Role can have MANY Permissions
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    @Builder.Default
    private Set<Permission> permissions = new HashSet<>();
}
```

### 📝 After the Code - What Just Happened?
- `@ManyToMany(fetch = FetchType.EAGER)`: When we load a Role, we **immediately** load its permissions. This is crucial for Spring Security to check permissions quickly during login.

---

## 🛠️ Step 4: Create the User Entity

### What we're doing:
Create the User entity and link it to Roles.

### The Code:
Create file: `src/main/java/com/example/demo/entity/User.java`

```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
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
- We store `passwordHash`, NOT the plain text password. 
- We use `FetchType.EAGER` for roles because when a user logs in, we need to know their roles immediately to generate the JWT token.

### 📦 Imports to Remember
```java
import jakarta.persistence.*; // Entity, Table, Column, ManyToMany, JoinTable, JoinColumn, FetchType
import java.util.HashSet;
import java.util.Set;
```

---

## 🛠️ Step 5: Create the Repositories

### What we're doing:
Create the interfaces that let us talk to the database.

### The Code:

**File 1:** `src/main/java/com/example/demo/repository/UserRepository.java`
```java
package com.example.demo.repository;

import com.example.demo.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);
}
```

**File 2:** `src/main/java/com/example/demo/repository/RoleRepository.java`
```java
package com.example.demo.repository;

import com.example.demo.entity.Role;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface RoleRepository extends JpaRepository<Role, Long> {
    Optional<Role> findByName(String name);
}
```

### 📝 After the Code - What Just Happened?
- Spring Data JPA reads these method names (`findByUsername`, `existsByEmail`) and **automatically writes the SQL for us**. We don't have to write a single line of SQL!

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Table "users" does not exist` | Flyway didn't run the migration. | Ensure the file is named `V1__init_security_tables.sql` and is in `src/main/resources/db/migration/`. |
| `LazyInitializationException` | Trying to access `user.getRoles()` outside a `@Transactional` method. | We fixed this by using `FetchType.EAGER` for Roles in the User entity. |
| `Column "password_hash" not found` | Entity field name doesn't match DB. | Ensure `@Column(name = "password_hash")` is exactly as written. |

---

## ✏️ Exercise

1. Start your Docker containers: `docker-compose up -d`
2. Run your Spring Boot application.
3. Open pgAdmin (http://localhost:5050) and verify that the `users`, `roles`, `permissions`, `user_roles`, and `role_permissions` tables were created.
4. Check the `roles` and `permissions` tables to ensure the default data was inserted.

---

## 🧠 Quiz

1. Why do we use `FetchType.EAGER` for the `roles` collection in the `User` entity?
2. What does the `@Builder.Default` annotation do in our entities?
3. If I call `userRepository.existsByUsername("admin")`, do I need to write the SQL query for it? Why or why not?

---

## 🛑 STOP

Reply with:
1. Confirmation that you ran the app and saw the tables in pgAdmin.
2. Your answers to the 3 quiz questions.

Once you reply, we will move to **Project Lesson 2: Building the Auth API (Register, Login, and JWT)**! 🚀