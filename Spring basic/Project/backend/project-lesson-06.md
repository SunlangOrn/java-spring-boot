# 🛒 ShopCore Project - Lesson 6: API Documentation with Swagger (OpenAPI)

---

## 🎯 Goal
- ✅ Add Swagger UI to the project.
- ✅ Generate beautiful, interactive API documentation automatically.
- ✅ Learn how to customize the documentation with annotations.

---

## 🧠 The Big Picture
Imagine you built this amazing API, and now a frontend developer needs to use it. 
Instead of writing a 50-page Word document explaining every endpoint, you add **Swagger**. 
Swagger reads your code and automatically generates a webpage where developers can see all endpoints, required fields, and even **test the API directly from the browser**.

---

## 🛠️ Step 1: Add Swagger Dependency

### What we're doing:
Add the SpringDoc library, which integrates OpenAPI/Swagger with Spring Boot.

### The Code:
**Update:** `pom.xml`

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

---

## 🛠️ Step 2: Configure Swagger

### What we're doing:
Add basic information about our API (title, version, description) so it shows up nicely on the webpage.

### The Code:
Create file: `src/main/java/com/example/demo/config/OpenApiConfig.java`

```java
package com.example.demo.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("ShopCore E-Commerce API")
                        .version("1.0.0")
                        .description("A production-ready e-commerce backend built with Spring Boot")
                        .contact(new Contact()
                                .name("Your Name")
                                .email("your.email@example.com"))
                        .license(new License().name("Apache 2.0").url("https://springdoc.org")));
    }
}
```

### 📝 After the Code - What Just Happened?
- `@Configuration` and `@Bean` tell Spring to create this `OpenAPI` object and use it to configure the Swagger UI.
- This information will appear at the top of the Swagger webpage.

---

## 🛠️ Step 3: Add Annotations to Controllers (Optional but Recommended)

### What we're doing:
Add descriptions to our endpoints so the documentation is even more helpful.

### The Code:
**Update:** `src/main/java/com/example/demo/controller/AuthController.java`

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;

@RestController
@RequestMapping("/api/auth")
@Tag(name = "Authentication", description = "Endpoints for user registration and login")
public class AuthController extends BaseRestController {

    // ...

    @Operation(summary = "Register a new user", description = "Creates a new customer account and returns JWT tokens")
    @PostMapping("/register")
    public ResponseEntity<HttpBodyResponse<AuthResponse>> register(@Valid @RequestBody RegisterRequest request) {
        return responseCreated(authService.register(request));
    }
}
```

### 📝 After the Code - What Just Happened?
- `@Tag`: Groups related endpoints together in the UI (e.g., all Auth endpoints under one collapsible section).
- `@Operation`: Adds a human-readable summary and description to the specific endpoint.

---

## 🧪 Run & Test

1. Restart your Spring Boot application.
2. Open your browser and go to: **http://localhost:8081/swagger-ui.html**
3. You will see a beautiful webpage listing all your API endpoints!
4. Click on `POST /api/auth/register`, click "Try it out", fill in the JSON, and click "Execute". It works just like Postman, but built right into your app!

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `404 Not Found` on `/swagger-ui.html` | Wrong URL or missing dependency. | Ensure you are using Spring Boot 3.x and the URL is exactly `/swagger-ui.html`. |
| Endpoints not showing up | Controller is not in a scanned package. | Ensure your controllers are under `com.example.demo`. |

---

## ✏️ Exercise
1. Add `@Tag` and `@Operation` annotations to your `ProductController` and `OrderController`.
2. Open Swagger UI and verify the descriptions appear correctly.

---

## 🧠 Quiz
1. What is the main benefit of using Swagger/OpenAPI in a project?
2. What URL do you visit in the browser to see the Swagger UI?
3. What does the `@Tag` annotation do?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to the final Project Lesson 7!