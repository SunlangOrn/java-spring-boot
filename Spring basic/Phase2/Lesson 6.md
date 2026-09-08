# 📘 Phase 2, Lesson 6: MapStruct (Automating DTO Mapping)

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [The Problem with Manual Mapping](#-the-problem-with-manual-mapping)
- [What is MapStruct?](#-what-is-mapstruct)
- [Step-by-Step Build](#-step-by-step-build)
- [How It Works Internally](#-how-it-works-internally)
- [Run and Test](#-run-and-test)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals
- ✅ Understand why manual mapping is tedious
- ✅ Add MapStruct to your project
- ✅ Create a mapper interface
- ✅ Replace manual mapping with MapStruct

---

## 😩 The Problem with Manual Mapping

In Lesson 5, you wrote this:
```java
return ProductResponse.builder()
        .id(product.getId())
        .name(product.getName())
        .price(product.getPrice())
        .build();
```

For 3 fields, this is fine. But what if your entity has **30 fields**?
- You'd write 30 lines of mapping code
- You might forget a field (bug!)
- Every time you add a field, you must update the mapping

---

## ✨ What is MapStruct?

MapStruct **generates the mapping code for you** at compile time.

You write an **interface**:
```java
ProductResponse toResponse(Product product);
```

MapStruct generates the **implementation** (the 30 lines of getters/setters) automatically.

**Key fact:** It does NOT use reflection at runtime. It generates real Java code during compilation. So there is **zero performance cost**.

---

## 🛠️ Step-by-Step Build

### Step 1: Add MapStruct to `pom.xml`
```xml
<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>1.5.5.Final</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>1.5.5.Final</version>
                    </path>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>1.18.30</version>
                    </path>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok-mapstruct-binding</artifactId>
                        <version>0.2.0</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Why 3 processors?**
1. `mapstruct-processor` = Generates the mapping code
2. `lombok` = Generates getters/setters
3. `lombok-mapstruct-binding` = Makes them work together

### Step 2: Create the Mapper Interface
```java
// src/main/java/com/example/product/mapper/ProductMapper.java
package com.example.product.mapper;

import com.example.product.dto.request.ProductCreateRequest;
import com.example.product.dto.response.ProductResponse;
import com.example.product.model.Product;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface ProductMapper {

    // Entity → Response DTO
    ProductResponse toResponse(Product product);

    // Request DTO → Entity
    @Mapping(target = "id", ignore = true) // Don't map ID (DB generates it)
    Product toEntity(ProductCreateRequest request);
}
```

**Line-by-line:**
- `@Mapper(componentModel = "spring")` = "Generate an implementation and make it a Spring Bean"
- `ProductResponse toResponse(Product product)` = "Map all matching fields from Product to ProductResponse"
- `@Mapping(target = "id", ignore = true)` = "Skip the id field because the database generates it"

### Step 3: Update the Service
```java
// In ProductService.java
private final ProductMapper productMapper;

public ProductService(ProductRepository productRepository, ProductMapper productMapper) {
    this.productRepository = productRepository;
    this.productMapper = productMapper; // Inject the mapper!
}

public ProductResponse createProduct(ProductCreateRequest request) {
    Product product = productMapper.toEntity(request);    // DTO → Entity
    Product saved = productRepository.save(product);
    return productMapper.toResponse(saved);               // Entity → DTO
}

public ProductResponse getProductById(Long id) {
    Product product = productRepository.findById(id).orElse(null);
    if (product == null) return null;
    return productMapper.toResponse(product);             // Entity → DTO
}
```

**That's it!** No more manual `.id(product.getId()).name(product.getName())...`

---

## ⚙️ How It Works Internally

```text
1. You run: mvn compile
        ↓
2. Maven sees @Mapper annotation
        ↓
3. mapstruct-processor reads the interface
        ↓
4. Checks field names in both classes
        ↓
5. Generates ProductMapperImpl.java with real getter/setter code
        ↓
6. Spring loads ProductMapperImpl as a Bean
        ↓
7. When you inject ProductMapper, you get the generated implementation
```

You can find the generated code at:
```text
target/generated-sources/annotations/com/example/product/mapper/ProductMapperImpl.java
```

---

## 🚀 Run and Test

```bash
./mvnw clean compile
./mvnw spring-boot:run
```

Test with the same curl commands. Everything works the same, but the code is much cleaner!

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| `cannot find symbol: ProductMapperImpl` | Processor didn't run | Run `mvn clean compile` |
| `No qualifying bean of type ProductMapper` | Missing `componentModel = "spring"` | Add it to `@Mapper` |
| `Unmapped target property: "id"` | MapStruct tries to map ID | Add `@Mapping(target = "id", ignore = true)` |
| Fields are null | Field names don't match | Use `@Mapping(source = "x", target = "y")` |

---

## 🛠️ Exercise
1. Add an `updateEntity` method to the mapper:
   ```java
   void updateEntity(ProductCreateRequest request, @MappingTarget Product product);
   ```
2. `@MappingTarget` tells MapStruct to update an existing object instead of creating a new one.
3. Use it in a new `updateProduct` service method.

---

## 🧠 Quiz
1. Does MapStruct use reflection at runtime?
2. What does `componentModel = "spring"` do?
3. Why do we ignore the `id` field when mapping Request → Entity?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 7.