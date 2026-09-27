# 🛒 ShopCore Project - Lesson 5: Order & Checkout API (Transactions)

---

## 🎯 Goal
- ✅ Build a checkout system that creates an Order.
- ✅ Understand and use `@Transactional` to ensure data integrity (reduce stock AND create order safely).

---

## 🧠 The Big Picture
When a user checks out, two things MUST happen:
1. An `Order` is created.
2. The `Product` stock is reduced.

If step 1 succeeds, but step 2 fails (e.g., the database crashes), the user gets a free product! 
**`@Transactional`** fixes this. It groups these steps into a single "unit of work". If ANY step fails, the database **rolls back** (undoes) everything, as if nothing happened.

---

## 🛠️ Step 1: Create Order & OrderItem Entities

### The Code:
**File 1:** `src/main/java/com/example/demo/entity/Order.java`
```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.*;
import lombok.*;
import java.util.List;

@Entity
@Table(name = "orders") // "order" is a reserved SQL keyword, so we use "orders"
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Order extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(nullable = false)
    private Double totalAmount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status = OrderStatus.PENDING;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items;
}
```

**File 2:** `src/main/java/com/example/demo/entity/OrderItem.java`
```java
package com.example.demo.entity;

import com.example.demo.entity.base.BaseEntity;
import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "order_items")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class OrderItem extends BaseEntity {

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false)
    private Integer quantity;

    @Column(nullable = false)
    private Double priceAtPurchase; // Save the price in case the product price changes later!
}
```

**File 3:** `src/main/java/com/example/demo/entity/OrderStatus.java` (Enum)
```java
package com.example.demo.entity;
public enum OrderStatus { PENDING, CONFIRMED, SHIPPED, CANCELLED }
```

---

## 🛠️ Step 2: Build the Checkout Service (The Magic of @Transactional)

### The Code:
**Update:** `src/main/java/com/example/demo/service/OrderService.java`

```java
package com.example.demo.service.impl;

import com.example.demo.entity.*;
import com.example.demo.exception.ResourceNotFoundException;
import com.example.demo.repository.*;
import com.example.demo.service.OrderService;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.ArrayList;
import java.util.List;

@Service
@RequiredArgsConstructor
public class OrderServiceImpl implements OrderService {

    private final OrderRepository orderRepository;
    private final CartItemRepository cartItemRepository;
    private final ProductRepository productRepository;
    private final UserRepository userRepository;

    @Override
    @Transactional // THE MAGIC ANNOTATION!
    public Order checkout() {
        // 1. Get logged-in user
        String username = SecurityContextHolder.getContext().getAuthentication().getName();
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new ResourceNotFoundException("User not found"));

        // 2. Get user's cart
        List<CartItem> cartItems = cartItemRepository.findByUser(user);
        if (cartItems.isEmpty()) {
            throw new RuntimeException("Cart is empty");
        }

        // 3. Create Order
        Order order = Order.builder()
                .user(user)
                .status(OrderStatus.PENDING)
                .items(new ArrayList<>())
                .build();

        double totalAmount = 0.0;

        // 4. Process each cart item
        for (CartItem cartItem : cartItems) {
            Product product = cartItem.getProduct();
            
            // Check stock
            if (product.getStock() < cartItem.getQuantity()) {
                throw new RuntimeException("Not enough stock for product: " + product.getName());
            }

            // REDUCE STOCK
            product.setStock(product.getStock() - cartItem.getQuantity());
            productRepository.save(product); // Hibernate will update this

            // Create OrderItem
            OrderItem orderItem = OrderItem.builder()
                    .order(order)
                    .product(product)
                    .quantity(cartItem.getQuantity())
                    .priceAtPurchase(product.getPrice())
                    .build();
            
            order.getItems().add(orderItem);
            totalAmount += product.getPrice() * cartItem.getQuantity();
        }

        order.setTotalAmount(totalAmount);
        Order savedOrder = orderRepository.save(order);

        // 5. Clear the cart
        cartItemRepository.deleteByUser(user);

        return savedOrder;
    }
}
```

### 📝 After the Code - What Just Happened?
- `@Transactional`: If the stock check fails on the 3rd item, the entire method stops. Hibernate automatically **rolls back** the stock reductions and order creation for the first 2 items. The database remains perfectly consistent.
- `priceAtPurchase`: We save the price at the moment of purchase. If the admin changes the product price tomorrow, historical orders remain accurate.

---

## 🧪 Run & Test

**Test: Checkout (Requires Customer JWT)**
```bash
curl -X POST http://localhost:8081/api/v1/orders/checkout \
-H "Authorization: Bearer <CUSTOMER_TOKEN>"
```
**Expected:** 201 Created with the Order details, and the Product stock should be reduced in the database.

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| Stock goes negative | Forgot `@Transactional` or the stock check logic. | Ensure the `if (product.getStock() < quantity)` check is present. |
| `LazyInitializationException` | Accessing `order.getItems()` outside the `@Transactional` method. | Keep all entity manipulations inside the `@Transactional` service method. |

---

## ✏️ Exercise
1. Add a `@PreAuthorize("hasAuthority('ORDER_CREATE')")` annotation to the checkout controller endpoint.
2. Test that a user without this permission gets a `403 Forbidden` error.

---

## 🧠 Quiz
1. What does `@Transactional` do if an exception is thrown in the middle of the method?
2. Why do we save `priceAtPurchase` in the `OrderItem` instead of just referencing the Product's current price?
3. Why is the table named `orders` instead of `order`?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Project Lesson 6!