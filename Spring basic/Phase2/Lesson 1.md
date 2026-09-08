# 📘 Phase 2, Lesson 1: Introduction to Spring Boot & Project Structure

## 📋 Table of Contents
- [Learning Goals](#-learning-goals)
- [What is Spring Boot?](#-what-is-spring-boot)
- [Core Concepts Explained Simply](#-core-concepts-explained-simply)
- [How It Works Internally](#-how-it-works-internally)
- [Step-by-Step Build](#-step-by-step-build)
- [Project Structure Explained](#-project-structure-explained)
- [Run and Test](#-run-and-test)
- [Common Errors & Troubleshooting](#-common-errors--troubleshooting)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goals

By the end of this lesson, you will:
- ✅ Understand the difference between Spring and Spring Boot.
- ✅ Understand core concepts: IoC, DI, Beans, and ApplicationContext.
- ✅ Know how `@SpringBootApplication` works under the hood.
- ✅ Set up a professional Spring Boot project structure.
- ✅ Create and run your first basic Spring Boot application.

---

## 💡 What is Spring Boot?

### The Problem with "Old" Spring
The original **Spring Framework** is incredibly powerful, but it requires a lot of manual configuration (XML files, setting up web servers, configuring database connections). It was like building a car from scratch just to drive to the store.

### The Solution: Spring Boot
**Spring Boot** is an extension of the Spring Framework that removes the boilerplate configuration. 
- It provides **"opinionated defaults"** (e.g., if it sees a database driver in your dependencies, it automatically configures the database connection).
- It includes an **embedded web server** (Tomcat), so you don't need to install a separate server to run your app.
- You can start coding business logic immediately.

*Analogy:* If Spring is a fully equipped professional kitchen, Spring Boot is a meal-kit delivery service. The ingredients and recipes are pre-measured and ready; you just need to cook.

---

## 🧠 Core Concepts Explained Simply

Before writing code, you must understand the "magic" of Spring.

### 1. IoC (Inversion of Control)
In standard Java, *you* create objects: `UserService userService = new UserService();`. You are in control.
In Spring, **the framework** creates and manages objects for you. You "invert" the control. You just say, "I need a UserService," and Spring provides it.

### 2. DI (Dependency Injection)
This is *how* Spring implements IoC. Instead of a class creating its dependencies, Spring "injects" them.
```java
// ❌ BAD (Manual creation - No DI)
public class OrderController {
    private UserService userService = new UserService(); 
}

// ✅ GOOD (Dependency Injection)
public class OrderController {
    private final UserService userService;
    
    // Spring injects the UserService here
    public OrderController(UserService userService) { 
        this.userService = userService;
    }
}
```

### 3. Beans
A **Bean** is simply any object that is created and managed by the Spring IoC Container. If you put `@Service` or `@RestController` on a class, Spring turns it into a Bean.

### 4. ApplicationContext
This is the actual **IoC Container**. It's the engine that reads your code, finds all the Beans, creates them, and wires them together when the application starts.

---

## ⚙️ How It Works Internally

When you run a Spring Boot app, it starts with the `@SpringBootApplication` annotation on your main class. This annotation is actually a combination of **three** annotations:

1. **`@Configuration`**: Marks the class as a source of Bean definitions.
2. **`@EnableAutoConfiguration`**: Tells Spring Boot to automatically configure the app based on the dependencies in your `pom.xml` (e.g., adding `spring-boot-starter-web` automatically starts an embedded Tomcat server).
3. **`@ComponentScan`**: Tells Spring to look for other components (`@RestController`, `@Service`, `@Repository`) in the current package and all sub-packages, registering them as Beans.

---

## 🏗️ Step-by-Step Build

### Step 1: Generate the Project
1. Go to [Spring Initializr](https://start.spring.io/).
2. Configure the following:
   - **Project:** Maven
   - **Language:** Java
   - **Spring Boot:** 3.3.x (or latest 3.x)
   - **Group:** `com.example`
   - **Artifact:** `product`
   - **Packaging:** Jar
   - **Java:** 21
3. Add Dependencies:
   - **Spring Web**
4. Click **Generate** and extract the ZIP file.

### Step 2: Open in IDE
Open the extracted folder in IntelliJ IDEA or Eclipse.

### Step 3: Create the Main Class
Spring Initializr already created this for you, but let's look at it:

```java
// src/main/java/com/example/product/ProductApplication.java
package com.example.product;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // The magic annotation!
public class ProductApplication {

    public static void main(String[] args) {
        // This boots up the ApplicationContext, starts Tomcat, and keeps the app running
        SpringApplication.run(ProductApplication.class, args);
    }
}
```

### Step 4: Create a Simple Controller
Let's create a simple REST endpoint to prove the web server is working.

```java
// src/main/java/com/example/product/controller/HelloController.java
package com.example.product.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController // Tells Spring this class handles HTTP requests and returns JSON/Text
public class HelloController {

    @GetMapping("/hello") // Maps HTTP GET requests to /hello
    public String sayHello() {
        return "Hello, Spring Boot!";
    }
}
```

---

## 📁 Project Structure Explained

Your project should look like this. Understanding this structure is critical for professional development.

```text
src/
 ├── main/
 │   ├── java/com/example/product/
 │   │   ├── ProductApplication.java       # Main class (Starts the app)
 │   │   ├── controller/                   # Handles HTTP requests/responses (The Waiter)
 │   │   ├── service/                      # Business logic (The Kitchen)
 │   │   ├── repository/                   # Database access (The Pantry)
 │   │   ├── entity/                       # JPA Database models
 │   │   ├── dto/                          # Data Transfer Objects (API Contracts)
 │   │   ├── mapper/                       # MapStruct interfaces
 │   │   ├── exception/                    # Global exception handling
 │   │   └── config/                       # Spring configuration classes
 │   └── resources/
 │       ├── application.yml               # Main configuration file
 │       ├── static/                       # Static files (CSS, JS, images)
 │       └── templates/                    # HTML templates (Thymeleaf)
 └── test/
     └── java/com/example/product/         # Unit and Integration tests
```

**Rule of Thumb:** The `controller` should never talk directly to the `repository`. It must always go through the `service`.

---

## 🚀 Run and Test

### 1. Run the Application
Open your terminal at the root of your project and run:
```bash
./mvnw spring-boot:run
```
*(On Windows, use `mvnw.cmd spring-boot:run`)*

You should see logs ending with:
```text
Started ProductApplication in 2.5 seconds (process running for 3.1)
```

### 2. Test the API
Open a **new** terminal window and use `curl` (or Postman):
```bash
curl -X GET http://localhost:8080/hello
```

### 3. Expected Response
```text
Hello, Spring Boot!
```

---

## 🚨 Common Errors & Troubleshooting

### Error 1: `Port 8080 was already in use`
- **Meaning:** Another application (or a previous crashed instance of your app) is using port 8080.
- **Fix:** Kill the process using port 8080, or change the port in `src/main/resources/application.yml`:
  ```yaml
  server:
    port: 8081
  ```

### Error 2: `Whitelabel Error Page (404 Not Found)`
- **Meaning:** You typed the URL wrong, or Spring didn't scan your controller.
- **Fix:** 
  1. Double-check the URL (`http://localhost:8080/hello`).
  2. Ensure your `HelloController` is inside the `com.example.product` package (or a sub-package like `com.example.product.controller`) so `@ComponentScan` can find it.

### Error 3: `BeanCreationException` or `NoSuchBeanDefinitionException`
- **Meaning:** Spring couldn't find a Bean to inject.
- **Fix:** Ensure the missing class has `@Service`, `@Component`, or `@RestController`, and that it is in the correct package hierarchy.

---

## 🛠️ Exercise

1. Create a new class called `StatusController` in the `controller` package.
2. Add a `GET` endpoint mapped to `/api/system/status`.
3. The endpoint should return the text: `"Application is running smoothly"`.
4. Run the app and test it with `curl`.

---

## 🧠 Quiz

Answer these 3 questions to prove you understand the internals:

1. What are the three annotations combined inside `@SpringBootApplication`, and what does each do?
2. Why is Constructor Injection (passing dependencies via the constructor) considered better than using `@Autowired` directly on fields?
3. If your `ProductApplication.java` is in `com.example.app`, but your `ProductService.java` is in `com.example.service`, will Spring find and inject it automatically? Why or why not?

---

## 🛑 STOP

**Do not move forward.** 

Reply with:
1. Confirmation that you have set up the project, ran the app, and the `curl` test worked.
2. The code you wrote for the **Exercise** (`StatusController`).
3. Your answers to the 3 quiz questions.

Once you reply, I will review your answers, and we will move to **Lesson 2: Configuration & Spring Web** to start building the actual Product API layers!