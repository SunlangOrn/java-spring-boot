# 📘 Phase 4, Lesson 4: Permissions & Method Security (`@PreAuthorize`)

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Concept: Roles vs. Permissions](#-the-concept-roles-vs-permissions)
- [Step 1: Update Database Schema](#-step-1-update-database-schema)
- [Step 2: Update Entities & UserDetailsService](#-step-2-update-entities--userdetailsservice)
- [Step 3: Apply Method Security](#-step-3-apply-method-security)
- [Run and Test](#-run-and-test)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand the difference between Roles and Permissions.
- ✅ Implement a dynamic Permission system in the database.
- ✅ Secure specific methods using `@PreAuthorize`.

---

## 💡 The Concept: Roles vs. Permissions

**Roles** are job titles: `ADMIN`, `USER`, `MANAGER`.
**Permissions** are specific actions: `PRODUCT_CREATE`, `PRODUCT_DELETE`, `USER_VIEW`.

**Why separate them?**
If you hardcode `hasRole('ADMIN')` in your controller, what happens when you hire a "Store Manager" who needs to delete products, but shouldn't have full Admin rights? 
If you use **Permissions**, you just assign the `PRODUCT_DELETE` permission to the `MANAGER` role. **Zero code changes required!**

---

## 🛠️ Step-by-Step Build

### Step 1: Update Database Schema
Create `V8__create_permissions.sql`:

```sql
CREATE TABLE permissions (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE role_permissions (
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    PRIMARY KEY (role_id, permission_id),
    FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY (permission_id) REFERENCES permissions(id) ON DELETE CASCADE
);

-- Insert default permissions
INSERT INTO permissions (name) VALUES ('PRODUCT_READ');
INSERT INTO permissions (name) VALUES ('PRODUCT_CREATE');
INSERT INTO permissions (name) VALUES ('PRODUCT_UPDATE');
INSERT INTO permissions (name) VALUES ('PRODUCT_DELETE');

-- Assign all permissions to ADMIN
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p WHERE r.name = 'ROLE_ADMIN';

-- Assign only READ to USER
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id FROM roles r, permissions p WHERE r.name = 'ROLE_USER' AND p.name = 'PRODUCT_READ';
```

### Step 2: Update Entities & UserDetailsService

**Permission Entity:**
```java
// src/main/java/com/example/product/entity/security/Permission.java
package com.example.product.entity.security;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "permissions")
@Getter @Setter @Builder @NoArgsConstructor @AllArgsConstructor
public class Permission {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;
}
```

**Update Role Entity:**
```java
// Add to Role.java
@ManyToMany(fetch = FetchType.EAGER)
@JoinTable(
    name = "role_permissions",
    joinColumns = @JoinColumn(name = "role_id"),
    inverseJoinColumns = @JoinColumn(name = "permission_id")
)
@Builder.Default
private Set<Permission> permissions = new HashSet<>();
```

**Update CustomUserDetailsService:**
We need to pass the Permissions to Spring Security as `GrantedAuthority`.

```java
// In CustomUserDetailsService.java, inside loadUserByUsername:

// 1. Get Roles
var authorities = new HashSet<org.springframework.security.core.GrantedAuthority>();

// 2. Add Roles as authorities
user.getRoles().forEach(role -> 
    authorities.add(new SimpleGrantedAuthority(role.getName()))
);

// 3. Add Permissions as authorities!
user.getRoles().forEach(role -> 
    role.getPermissions().forEach(permission -> 
        authorities.add(new SimpleGrantedAuthority(permission.getName()))
    )
);

// Return the UserDetails object with the combined authorities
return org.springframework.security.core.userdetails.User.builder()
        .username(user.getUsername())
        .password(user.getPasswordHash())
        .authorities(authorities) // Now contains both ROLE_ADMIN and PRODUCT_DELETE!
        .build();
```

### Step 3: Apply Method Security

First, enable method security in your config:
```java
// In SecurityConfig.java or ProductApplication.java
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;

@EnableMethodSecurity // ← ADD THIS! Allows @PreAuthorize
public class SecurityConfig { ... }
```

Now, update the `ProductController`:

```java
// In ProductController.java
import org.springframework.security.access.prepost.PreAuthorize;

@PostMapping
@PreAuthorize("hasAuthority('PRODUCT_CREATE')") // Checks for the specific permission!
public ResponseEntity<ProductResponse> createProduct(...) { ... }

@DeleteMapping("/{id}")
@PreAuthorize("hasAuthority('PRODUCT_DELETE')")
public ResponseEntity<Void> deleteProduct(...) { ... }

@GetMapping
@PreAuthorize("hasAuthority('PRODUCT_READ')")
public ResponseEntity<List<ProductResponse>> getAllProducts() { ... }
```

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

**Test 1: Login as USER (john_doe) and try to DELETE**
```bash
curl -u john_doe:securePassword123 -X DELETE http://localhost:8080/api/products/1
```
*Expected:* `403 Forbidden`. John has `PRODUCT_READ`, but not `PRODUCT_DELETE`.

**Test 2: Login as ADMIN and try to DELETE**
```bash
curl -u admin:admin123 -X DELETE http://localhost:8080/api/products/1
```
*Expected:* `204 No Content` (or 200). Admin has `PRODUCT_DELETE`.

---

## 🚨 Common Errors
1. **`@PreAuthorize` is ignored**: You forgot to add `@EnableMethodSecurity` to your configuration class.
2. **`403 Forbidden` for Admin**: The `role_permissions` mapping table is empty. Check your Flyway migration `V8`.
3. **`LazyInitializationException`**: When loading permissions in `UserDetailsService`. *Fix:* Ensure `@ManyToMany` for permissions uses `FetchType.EAGER`, or keep the whole `loadUserByUsername` method inside a `@Transactional` block.

---

## 🛠️ Exercise
1. Create a new role in the DB called `ROLE_MANAGER`.
2. Assign `PRODUCT_READ`, `PRODUCT_CREATE`, and `PRODUCT_UPDATE` to `ROLE_MANAGER` (but NOT delete).
3. Register a user with the `ROLE_MANAGER` role.
4. Verify they can POST (create) but cannot DELETE.

---

## 🧠 Quiz
1. What is the main advantage of using Permissions instead of just Roles?
2. What annotation enables `@PreAuthorize` in Spring Boot?
3. What is the difference between `hasRole('ADMIN')` and `hasAuthority('PRODUCT_DELETE')`?

---

## 🎉 Phase 4 Complete!

You have successfully built a complete, database-driven Authentication and Authorization system!
- ✅ Users, Roles, and Permissions are stored in PostgreSQL.
- ✅ Passwords are securely hashed with BCrypt.
- ✅ Spring Security intercepts requests via the Filter Chain.
- ✅ Method security (`@PreAuthorize`) protects specific endpoints.

## 🛑 STOP
Reply with "Phase 4 Complete" and your exercise results. 

Next, we will enter **Phase 5: JWT (JSON Web Tokens)**. We will replace HTTP Basic Auth with industry-standard Access and Refresh tokens, making our API ready for modern frontend frameworks (React, Angular, Mobile apps)!