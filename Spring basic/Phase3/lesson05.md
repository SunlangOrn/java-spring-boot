# 📘 Phase 3, Lesson 5: Caching with Redis

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem: Database Bottleneck](#-the-problem-database-bottleneck)
- [The Solution: Redis Caching](#-the-solution-redis-caching)
- [Step-by-Step Build](#-step-by-step-build)
- [Run and Test](#-run-and-test)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand why databases become bottlenecks.
- ✅ Start a Redis container using Docker.
- ✅ Implement `@Cacheable` and `@CacheEvict` in Spring Boot.

---

## 🚨 The Problem: Database Bottleneck

Imagine your API gets 10,000 requests per second for the "Homepage Products". 
Every single request hits PostgreSQL. PostgreSQL has to read from the disk, parse the SQL, and return the data. 
**Result:** The database CPU spikes to 100%, and the app crashes.

---

## 💡 The Solution: Redis Caching

**Redis** is an in-memory data store. It's like a super-fast whiteboard next to the database.

1. **First request:** App asks Redis for "Homepage Products". Redis says "I don't have it." App asks PostgreSQL, gets the data, and **writes it to Redis**.
2. **Second request:** App asks Redis. Redis says "Here you go!" (Takes 1 millisecond instead of 50ms). App **never touches PostgreSQL**.

---

## 🛠️ Step-by-Step Build

### Step 1: Start Redis with Docker
```bash
docker run --name redis-cache -p 6379:6379 -d redis:7
```

### Step 2: Add Dependencies
```xml
<!-- In pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### Step 3: Configure Redis & Enable Caching
Update `application.yml`:
```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
  cache:
    type: redis
    redis:
      time-to-live: 60000 # Cache expires after 60 seconds (1 minute)
```

Enable caching in your main class:
```java
// ProductApplication.java
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching // ← ADD THIS!
public class ProductApplication { ... }
```

### Step 4: Add `@Cacheable` to the Service
```java
// In ProductService.java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;

// When this is called, Spring checks Redis first!
@Cacheable(value = "products", key = "#id")
@Transactional(readOnly = true)
public ProductResponse getProductById(Long id) {
    System.out.println(">>> FETCHING FROM DATABASE (SLOW!) <<<");
    Product product = productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    return productMapper.toResponse(product);
}
```
*Note the `System.out.println`. We will use this to prove the cache is working!*

### Step 5: Add `@CacheEvict` to Update/Delete
When we update a product, we **must** delete it from the cache, otherwise users will see old data!

```java
@CacheEvict(value = "products", key = "#id")
@Transactional
public ProductResponse updateProduct(Long id, ProductCreateRequest request) {
    // ... update logic ...
}
```

---

## 🚀 Run and Test

```bash
./mvnw spring-boot:run
```

**Test 1: First Request (Cache Miss)**
```bash
curl http://localhost:8080/api/products/1
```
Look at your console. You will see:
`>>> FETCHING FROM DATABASE (SLOW!) <<<`

**Test 2: Second Request (Cache Hit)**
```bash
curl http://localhost:8080/api/products/1
```
Look at your console. **The print statement is GONE!** It returned instantly from Redis.

**Test 3: Update the Product (Cache Evict)**
```bash
curl -X PUT http://localhost:8080/api/products/1 \
-H "Content-Type: application/json" \
-d '{"name": "Updated Mouse", "price": 35.0, "stock": 10, "categoryId": 1}'
```
Now, if you call GET again, you will see the print statement return, because the cache was cleared and it had to fetch from the DB again!

---

## 🚨 Common Errors
1. **`Cannot get Jedis connection`**: Redis container is not running. *Fix:* `docker start redis-cache`.
2. **Cache not updating after PUT**: You forgot `@CacheEvict` on the update method.
3. **Serialization Error**: Redis needs to convert your Java object to bytes. Ensure your DTOs implement `Serializable` or configure a `RedisCacheConfiguration` with `GenericJackson2JsonRedisSerializer`. (Spring Boot handles basic DTOs fine by default).

---

## 🛠️ Exercise
1. Add `@Cacheable(value = "allProducts")` to the `getAllProducts()` method.
2. Add `@CacheEvict(value = "allProducts", allEntries = true)` to the `createProduct()` method.
3. Test it by calling GET all, then POST a new product, then GET all again.

---

## 🧠 Quiz
1. What is the main advantage of Redis over PostgreSQL?
2. What does `@Cacheable` do?
3. Why is it critical to use `@CacheEvict` when updating data?

---

## 🎉 Phase 3 Progress Check
You have now mastered Advanced JPA, Dynamic Queries, Concurrency, and Caching! 

## 🛑 STOP
Reply with your exercise code and quiz answers. Once confirmed, we will move to **Phase 4: Spring Security Fundamentals**, where we will finally add Authentication and Authorization to our API!