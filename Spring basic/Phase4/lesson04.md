# 📘 Phase 4, Lesson 4: Permissions & Method Security (`@PreAuthorize`)

---

## 🎯 Goal
- ✅ Understand the difference between Roles and Permissions.
- ✅ Implement a dynamic Permission system in the database.
- ✅ Secure specific methods using `@PreAuthorize`.

---

## 🧠 The Big Picture

**Roles** are job titles: `ADMIN`, `USER`, `MANAGER`.
**Permissions** are specific actions: `PRODUCT_CREATE`, `PRODUCT_DELETE`, `USER_VIEW`.

**Why separate them?**
If you hardcode `hasRole('ADMIN')` in your controller, what happens when you hire a "Store Manager" who needs to delete products, but shouldn't have full Admin rights? 
If you use **Permissions**, you just assign the `PRODUCT_DELETE` permission to the `MANAGER` role. **Zero code changes required!**

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **Role** | A job title or group (e.g., ADMIN). |
| **Permission (Authority)** | A specific action (e.g., PRODUCT_DELETE). |
| **`@PreAuthorize`** | An annotation that checks if the user has the required permission *before* the method runs. |
| **`@EnableMethodSecurity`** | The switch that turns on `@PreAuthorize`. |

---

## 🛠️ Step 1: Update Database Schema

### What we're doing:
Create tables for `permissions` and link them to `roles`.

### The Code:
Create file: `src/main/resources/db/migration/V7__create_permissions.sql`

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

### 📝 After the Code - What Just Happened?
- We created a many-to-many relationship between Roles and Permissions.
- We pre-loaded 4 permissions and assigned them to the roles.

---

## 🛠️ Step 2: Update Entities & UserDetailsService

### What we're doing:
Create the Permission entity, link it to Role, and pass the permissions to Spring Security.

### The Code:

**File 1:** `src/main/java/com/example/demo/entity/Permission.java`
```java
package com.example.demo.entity;

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

**File 2:** Update `Role.java` to include permissions
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

**File 3:** Update `CustomUserDetailsService.java` to load permissions
```java
// Inside loadUserByUsername, replace the authorities logic with this:

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

### 📝 After the Code - What Just Happened?
- When a user logs in, Spring Security now loads both their Roles AND their Permissions.
- Both are treated as "Authorities" internally.

---

## 🛠️ Step 3: Apply Method Security

### What we're doing:
Turn on method security and protect the Product endpoints.

### The Code:

**Step A:** Enable method security in `SecurityConfig.java`
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // <-- ADD THIS! Allows @PreAuthorize
public class SecurityConfig { ... }
```

**Step B:** Update `ProductController.java`
```java
import org.springframework.security.access.prepost.PreAuthorize;

// ... inside ProductController ...

@PostMapping
@PreAuthorize("hasAuthority('PRODUCT_CREATE')") // Checks for the specific permission!
public ResponseEntity<HttpBodyResponse<ProductResponse>> create(...) { ... }

@DeleteMapping("/{id}")
@PreAuthorize("hasAuthority('PRODUCT_DELETE')")
public ResponseEntity<HttpBodyResponse<Void>> delete(...) { ... }

@GetMapping
@PreAuthorize("hasAuthority('PRODUCT_READ')")
public ResponseEntity<HttpBodyResponse<List<ProductResponse>>> getAll() { ... }
```

### 📝 After the Code - What Just Happened?
- `@EnableMethodSecurity`: Turns on the ability to use `@PreAuthorize`.
- `@PreAuthorize("hasAuthority('PRODUCT_CREATE')")`: Before this method runs, Spring checks if the logged-in user has the `PRODUCT_CREATE` permission. If not, it throws a `403 Forbidden`.

### 📦 Imports to Remember
```java
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
```

---

## 🧪 Run & Test

### Test 1: Login as USER (john_doe) and try to DELETE
```bash
curl -u john_doe:securePassword123 -X DELETE http://localhost:8081/api/v1/products/1
```
**Expected:** `403 Forbidden`. John has `PRODUCT_READ`, but not `PRODUCT_DELETE`.

### Test 2: Login as ADMIN and try to DELETE
```bash
curl -u admin:admin123 -X DELETE http://localhost:8081/api/v1/products/1
```
**Expected:** `200 OK`. Admin has `PRODUCT_DELETE`.

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `@PreAuthorize` is ignored | You forgot to add `@EnableMethodSecurity`. | Add it to `SecurityConfig`. |
| `403 Forbidden` for Admin | The `role_permissions` mapping table is empty. | Check your Flyway migration `V7`. |
| `LazyInitializationException` | Permissions not loaded during login. | Ensure `@ManyToMany` for permissions uses `FetchType.EAGER`. |

---

## ✏️ Exercise

1. Create a new role in the DB called `ROLE_MANAGER` (via Flyway or pgAdmin).
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
Reply with **"Phase 4 Complete"** and your exercise results. 

Next, we will enter **Phase 5: JWT (JSON Web Tokens)**. We will replace HTTP Basic Auth with industry-standard Access and Refresh tokens, making our API ready for modern frontend frameworks!