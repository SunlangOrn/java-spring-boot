# 🛒 ShopCore Project - Lesson 4: Shopping Cart API

---

## 🎯 Goal
- ✅ Create a Shopping Cart system where users can add, update, and remove items.
- ✅ Understand how to link a User to multiple Products with quantities.

---

## 🧠 The Big Picture
A shopping cart is a list of items a user wants to buy. 
In the database, this is a **Many-to-Many relationship with extra data** (the `quantity`). 
Since JPA `@ManyToMany` doesn't handle extra columns well, we create a dedicated `CartItem` entity that links a `User` and a `Product`.

---

## 🛠️ Step 1: Create the CartItem Entity

### What we're doing:
Create the middleman table that holds the User ID, Product ID, and Quantity.

### The Code:
Create file: `src/main/java/com/example/demo/entity/CartItem.java`

```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "cart_items", uniqueConstraints = @UniqueConstraint(columnNames = {"user_id", "product_id"}))
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class CartItem extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false)
    private Integer quantity;
}
```

### 📝 After the Code - What Just Happened?
- `@UniqueConstraint`: Ensures a user can't have two separate rows for the same product in their cart. They must update the `quantity` of the existing row instead.

---

## 🛠️ Step 2: Create CartItemRepository

### The Code:
Create file: `src/main/java/com/example/demo/repository/CartItemRepository.java`

```java
package com.example.demo.repository;

import com.example.demo.entity.CartItem;
import com.example.demo.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;
import java.util.Optional;

public interface CartItemRepository extends JpaRepository<CartItem, Long> {
    List<CartItem> findByUser(User user);
    Optional<CartItem> findByUserAndProductId(User user, Long productId);
    void deleteByUser(User user);
}
```

---

## 🛠️ Step 3: Build the CartService

### What we're doing:
Write the logic to add items to the cart. If the item already exists, we just increase the quantity.

### The Code:
Create file: `src/main/java/com/example/demo/service/CartService.java` (and `CartServiceImpl`)

```java
package com.example.demo.service.impl;

import com.example.demo.dto.request.CartItemRequest;
import com.example.demo.dto.response.CartItemResponse;
import com.example.demo.entity.CartItem;
import com.example.demo.entity.Product;
import com.example.demo.entity.User;
import com.example.demo.exception.ResourceNotFoundException;
import com.example.demo.mapper.CartMapper; // Assume you created this
import com.example.demo.repository.CartItemRepository;
import com.example.demo.repository.ProductRepository;
import com.example.demo.service.CartService;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
public class CartServiceImpl implements CartService {

    private final CartItemRepository cartItemRepository;
    private final ProductRepository productRepository;
    private final CartMapper cartMapper;

    @Override
    @Transactional
    public CartItemResponse addToCart(CartItemRequest request) {
        // 1. Get the currently logged-in user
        String username = SecurityContextHolder.getContext().getAuthentication().getName();
        User user = // ... fetch user by username (assume you have UserRepository injected)

        // 2. Check if product exists
        Product product = productRepository.findById(request.productId())
                .orElseThrow(() -> new ResourceNotFoundException("Product not found"));

        // 3. Check if item is already in cart
        CartItem existingItem = cartItemRepository.findByUserAndProductId(user, product.getId()).orElse(null);

        if (existingItem != null) {
            existingItem.setQuantity(existingItem.getQuantity() + request.quantity());
            return cartMapper.toResponse(cartItemRepository.save(existingItem));
        }

        // 4. Create new cart item
        CartItem newItem = CartItem.builder()
                .user(user)
                .product(product)
                .quantity(request.quantity())
                .build();

        return cartMapper.toResponse(cartItemRepository.save(newItem));
    }
}
```

### 📝 After the Code - What Just Happened?
- `SecurityContextHolder.getContext().getAuthentication().getName()`: This is how we get the username of the person who sent the JWT token! We use this to find their User record.
- We check if the item exists. If yes, we add to the quantity. If no, we create a new row.

---

## 🧪 Run & Test

**Test: Add to Cart (Requires Customer JWT)**
```bash
curl -X POST http://localhost:8081/api/v1/cart \
-H "Authorization: Bearer <CUSTOMER_TOKEN>" \
-H "Content-Type: application/json" \
-d '{"productId": 1, "quantity": 2}'
```

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Authentication object is anonymous` | The endpoint is not protected, so no user is logged in. | Add `@PreAuthorize("isAuthenticated()")` to the Controller endpoint. |
| Duplicate cart rows | Forgot the `@UniqueConstraint` or the `if (existingItem != null)` check. | Ensure both are present in the code. |

---

## ✏️ Exercise
1. Create a `GET /api/v1/cart` endpoint that returns all items for the logged-in user.
2. Create a `DELETE /api/v1/cart/{productId}` endpoint to remove an item.

---

## 🧠 Quiz
1. Why do we use a separate `CartItem` entity instead of a `@ManyToMany` relationship between User and Product?
2. How do we get the currently logged-in user's username inside the Service layer?
3. What does `@UniqueConstraint` do in the `@Table` annotation?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Project Lesson 5!