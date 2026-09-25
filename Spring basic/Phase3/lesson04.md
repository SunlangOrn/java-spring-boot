# 📘 Phase 3, Lesson 4: Caching with Redis

---

## 🎯 Goal
Add Redis caching to the Product API so frequently requested data is served instantly without hitting the database every time.

---

## 🧠 The Big Picture

Imagine a popular restaurant:
- The waiter (API) asks the chef (Database) for the menu.
- The chef takes 5 minutes to write it out.
- Every single customer asks for the menu. The chef writes it out 100 times!

**Smart solution**: The waiter writes the menu on a whiteboard (Cache) near the entrance. Now, when customers ask, the waiter just points to the whiteboard. **Instant answer, zero work for the chef.**

That's what Redis does. It stores frequently used data in **memory** (RAM), which is 100x faster than reading from a database (disk).

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **Cache** | A temporary storage for frequently used data |
| **Redis** | A super-fast, in-memory database used for caching |
| **Cache Hit** | The data was found in the cache (fast!) |
| **Cache Miss** | The data was NOT in the cache, so we had to go to the database (slow) |
| **Cache Evict** | Removing old data from the cache when it changes |
| **TTL** | Time To Live - how long data stays in the cache before expiring |

---

## 🛠️ Step 1: Add Redis to Docker Compose

### What we're doing:
Start a Redis container alongside our existing PostgreSQL and pgAdmin containers.

### The Code:
**Update:** `docker-compose.yml` (in the project root)

Add this new service:

```yaml
  redis:
    image: redis:7
    container_name: my-redis-cache
    ports:
      - "6379:6379"
```

### 📝 After the Code - What Just Happened?
- We added a new container called `my-redis-cache` using the official Redis image.
- Port `6379` is the default Redis port (like 5432 is the default for PostgreSQL).
- Redis stores everything in RAM, so it's incredibly fast but data is lost if the container restarts (which is fine for a cache).

### 💡 Note
> Run `docker-compose up -d` to start the new Redis container. You should see 3 containers running: postgres, pgadmin, and redis.

---

## 🛠️ Step 2: Add Dependencies

### What we're doing:
Add the Spring Boot libraries needed to connect to Redis and enable caching.

### The Code:
**Update:** `pom.xml`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### 📝 After the Code - What Just Happened?
- `spring-boot-starter-data-redis`: Gives Spring the ability to connect to and talk to Redis.
- `spring-boot-starter-cache`: Gives us the `@Cacheable`, `@CachePut`, and `@CacheEvict` annotations.

---

## 🛠️ Step 3: Configure Redis and Enable Caching

### What we're doing:
Tell Spring where Redis is running and turn on the caching feature.

### The Code:

**Update:** `src/main/resources/application.yml`
```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
    redis:
      time-to-live: 60000  # Cache expires after 60 seconds (in milliseconds)
```

**Update:** `src/main/java/com/example/demo/DemoApplication.java`
```java
@SpringBootApplication
@EnableJpaAuditing
@EnableCaching  // <-- ADD THIS LINE
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 📝 After the Code - What Just Happened?
- `spring.data.redis.host`: Tells Spring "Redis is running on my local machine."
- `time-to-live: 60000`: Data in the cache will automatically expire after 60 seconds. This prevents stale data.
- `@EnableCaching`: This is the switch that turns on Spring's caching system. Without it, all the `@Cacheable` annotations will be ignored.

### 📦 Imports to Remember
```java
import org.springframework.cache.annotation.EnableCaching;
```

---

## 🛠️ Step 4: Add Caching to the Service

### What we're doing:
Tell Spring to cache the results of `getAllProducts()` and `getProductById()`.

### The Code:
**Update:** `src/main/java/com/example/demo/service/impl/ProductServiceImpl.java`

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;

// ... inside the class ...

@Override
@Cacheable(value = "products")  // Cache the result under the name "products"
@Transactional(readOnly = true)
public List<ProductResponse> getAllProducts() {
    System.out.println(">>> FETCHING FROM DATABASE (SLOW!) <<<");
    return productRepository.findAll().stream()
            .map(productMapper::toResponse)
            .toList();
}

@Override
@Cacheable(value = "product", key = "#id")  // Cache each product by its ID
@Transactional(readOnly = true)
public ProductResponse getProductById(Long id) {
    System.out.println(">>> FETCHING PRODUCT " + id + " FROM DATABASE <<<");
    Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException(
                "Product not found with id: " + id));
    return productMapper.toResponse(product);
}

@Override
@CacheEvict(value = {"products", "product"}, allEntries = true)  // Clear cache when data changes
@Transactional
public ProductResponse addProduct(ProductRequest request) {
    // ... existing code ...
}
```

### 📝 After the Code - What Just Happened?
- `@Cacheable(value = "products")`: The first time `getAllProducts()` is called, Spring runs the method, saves the result in Redis under the name "products", and returns it. The **second** time, Spring skips the method entirely and returns the cached result from Redis.
- `@Cacheable(value = "product", key = "#id")`: Each product is cached separately by its ID. So product 1 and product 2 have different cache entries.
- `@CacheEvict(value = {"products", "product"}, allEntries = true)`: When a new product is created, we **clear the entire cache**. This ensures the next GET request fetches fresh data from the database.
- The `System.out.println` lines are temporary. They help you SEE whether the cache is working (you'll see the message on cache miss, but NOT on cache hit).

### 📦 Imports to Remember
```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;
```

### 💡 Note
> **When to evict the cache**: You must add `@CacheEvict` to ALL methods that change data (create, update, delete). If you forget, users will see old, outdated data until the cache expires.

---

## 🧪 Run & Test

1. Start Redis: `docker-compose up -d`
2. Restart the app: `./mvnw spring-boot:run`

### Test 1: First request (Cache MISS)
```bash
curl http://localhost:8081/api/v1/products
```
Look at the console. You will see: `>>> FETCHING FROM DATABASE (SLOW!) <<<`

### Test 2: Second request (Cache HIT)
```bash
curl http://localhost:8081/api/v1/products
```
Look at the console. The print message is **GONE**! The data came from Redis instantly.

### Test 3: Create a product (Cache EVICT)
```bash
curl -X POST http://localhost:8081/api/v1/products \
-H "Content-Type: application/json" \
-d '{"name": "Tablet", "price": 499.99, "description": "iPad", "categoryId": 1}'
```
Now call GET again. The print message returns because the cache was cleared.

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `Cannot get Redis connection` | Redis container not running | Run `docker-compose up -d` |
| Cache never updates after PUT/DELETE | Forgot `@CacheEvict` | Add `@CacheEvict` to all write methods |
| `@Cacheable` not working | Forgot `@EnableCaching` | Add it to the main application class |

---

## ✏️ Exercise

1. Add `@CacheEvict` to the `updateProduct()` and `deleteProduct()` methods.
2. Test updating a product and verify the cache is cleared.
3. Change the TTL to 30 seconds and verify the cache expires after 30 seconds.

---

## 🧠 Quiz

1. What is the difference between a Cache Hit and a Cache Miss?
2. Why must we use `@CacheEvict` when creating, updating, or deleting data?
3. What does `time-to-live: 60000` mean?

---

## 🎉 Phase 3 Complete!

You have learned:
- ✅ Database relationships (One-to-Many)
- ✅ Category CRUD with DTOs
- ✅ Pagination and sorting with JOIN FETCH
- ✅ Redis caching with @Cacheable and @CacheEvict

## 🛑 STOP
Reply with **"Phase 3 Complete"** and your exercise/quiz answers. Next is **Phase 4: Spring Security**!