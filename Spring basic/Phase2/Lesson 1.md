# 📘 Phase 2, Lesson 1: Your First Spring Boot Web API

## 📋 Table of Contents
- [Learning Goal](#-learning-goal)
- [What is Spring Boot? (The Kitchen Analogy)](#-what-is-spring-boot-the-kitchen-analogy)
- [Step 1: Create the Project](#-step-1-create-the-project)
- [Step 2: Understand the "Engine Starter"](#-step-2-understand-the-engine-starter)
- [Step 3: Create Your First Controller (The Waiter)](#-step-3-create-your-first-controller-the-waiter)
- [Step 4: Run the Application](#-step-4-run-the-application)
- [Step 5: Test the API](#-step-5-test-the-api)
- [Common Errors](#-common-errors)
- [Exercise](#-exercise)
- [Quiz](#-quiz)

---

## 🎯 Learning Goal
By the end of this single lesson, you will:
- ✅ Create a Spring Boot project from scratch.
- ✅ Understand what a **Controller** is.
- ✅ Create a simple **GET** endpoint that returns text.
- ✅ Run the app and test it in your browser/terminal.

*That's it! No databases, no complex folders. Just making the server talk to the client.*

---

## 🍳 What is Spring Boot? (The Kitchen Analogy)

Imagine you want to open a restaurant. 
- **Java** is the raw ingredients (flour, water, tomatoes).
- **Spring Framework** is a fully equipped professional kitchen. It has ovens, mixers, and prep stations. But you have to figure out how to turn them on and connect the gas lines yourself.
- **Spring Boot** is a **meal-kit delivery service**. The kitchen is already set up, the gas is connected, and the oven is pre-heated. You just open the box and start cooking.

Spring Boot gives you an **embedded web server** (Tomcat) out of the box. You don't need to install anything else to run a web app.

---

## 🛠️ Step 1: Create the Project

1. Open your browser and go to [https://start.spring.io](https://start.spring.io).
2. Fill in the settings exactly like this:
   - **Project:** Maven
   - **Language:** Java
   - **Spring Boot:** 3.3.x (or the latest 3.x version)
   - **Group:** `com.example`
   - **Artifact:** `demo`
   - **Name:** `demo`
   - **Description:** Demo project for Spring Boot
   - **Package name:** `com.example.demo`
   - **Packaging:** Jar
   - **Java:** 21
3. Under **Dependencies** (on the right side), click **ADD DEPENDENCIES** and search for:
   - **Spring Web** (This gives us the tools to build web APIs).
4. Click the green **GENERATE** button at the bottom.
5. A ZIP file will download. Extract (unzip) it.
6. Open the extracted folder in your IDE (IntelliJ IDEA, Eclipse, or VS Code).

---

## ⚙️ Step 2: Understand the "Engine Starter"

Look inside the folder: `src/main/java/com/example/demo/`. 
You will see a file called `DemoApplication.java`. Open it.

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication // <-- THE MAGIC ANNOTATION
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

**What does `@SpringBootApplication` do?**
Think of it as the "Start Engine" button. When you run this file, it:
1. Starts the embedded web server (Tomcat) on port 8080.
2. Scans your code for other components (like Controllers) and turns them on.

*You don't need to change anything in this file. Just leave it as is.*

---

## 🤵 Step 3: Create Your First Controller (The Waiter)

In a restaurant, the **Waiter** takes the customer's order and brings back the food. 
In Spring Boot, a **Controller** takes the HTTP Request (from the browser/Postman) and brings back the HTTP Response (JSON or Text).

Let's create our Waiter.

1. Right-click on the `com.example.demo` package.
2. Create a **New Package** called `controller`. (It is good practice to organize code into folders).
3. Inside the `controller` package, create a **New Java Class** called `HelloController`.

Add this exact code:

```java
package com.example.demo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController // 1. Tells Spring: "This class handles web requests"
public class HelloController {

    @GetMapping("/hello") // 2. Tells Spring: "Listen for GET requests at the URL /hello"
    public String sayHello() {
        // 3. This is the response we send back to the client
        return "Hello! Welcome to my first Spring Boot API.";
    }
}
```

**Line-by-line explanation:**
- `@RestController`: This is crucial. Without it, Spring doesn't know this class is supposed to handle web traffic.
- `@GetMapping("/hello")`: This creates an **endpoint**. It means: "If a client sends an HTTP GET request to `http://localhost:8080/hello`, run the `sayHello()` method."
- `return "..."`: The text inside the quotes is sent back over the internet to the client.

---

## 🚀 Step 4: Run the Application

1. Go back to `DemoApplication.java`.
2. Click the green **Play/Run** button next to the `main` method (or right-click the file and select "Run").
3. Look at the bottom console in your IDE. You will see a lot of text scrolling by. 
4. Wait until you see something like this:
   ```text
   Started DemoApplication in 2.5 seconds (process running for 3.1)
   ```
   *Congratulations! Your web server is now running and listening on port 8080.*

---

## 🧪 Step 5: Test the API

Now we need to act as the "Client" and send a request to our server.

**Option A: Using your Web Browser**
1. Open Chrome, Firefox, or Safari.
2. In the address bar, type: `http://localhost:8080/hello`
3. Press Enter.
4. **Expected Result:** The browser will display: `Hello! Welcome to my first Spring Boot API.`

**Option B: Using the Terminal (cURL)**
1. Open a new terminal window.
2. Type this command and press Enter:
   ```bash
   curl http://localhost:8080/hello
   ```
3. **Expected Result:** `Hello! Welcome to my first Spring Boot API.`

---

## 🚨 Common Errors

| Error | Cause | Fix |
|---|---|---|
| `Port 8080 was already in use` | Another app (or a previous run of this app) is using port 8080. | Stop the other app, or change the port in `application.properties` (we will learn this later). |
| `Whitelabel Error Page` (404 Not Found) | You typed the URL wrong in the browser. | Make sure you typed exactly `http://localhost:8080/hello` (case-sensitive!). |
| `Cannot resolve symbol 'RestController'` | You didn't add the "Spring Web" dependency. | Go back to [start.spring.io](https://start.spring.io), add "Spring Web", download, and replace your `pom.xml`. |

---

## 🛠️ Exercise

Let's prove you understand how Controllers work.

1. Open your `HelloController.java`.
2. Add a **new method** below `sayHello()`.
3. Name the method `sayGoodbye()`.
4. Add the `@GetMapping` annotation to it, mapping it to the URL `/goodbye`.
5. Make it return the text: `"Goodbye! See you next time."`
6. **Restart your application** (Stop the red square button, then run it again).
7. Test it in your browser: `http://localhost:8080/goodbye`

---

## 🧠 Quiz

Answer these 3 simple questions to prove you understand the basics:

1. In the restaurant analogy, what part of Spring Boot is the "Waiter"?
2. What annotation do we put on a **class** to tell Spring it should handle web requests and return data?
3. If I write `@GetMapping("/users")`, what exact URL do I type in my browser to test it?

---

## 🛑 STOP

**Do not move forward.** 

Reply with:
1. Confirmation that you created the project, ran it, and saw "Hello" in your browser.
2. The code you wrote for the **Exercise** (the new `sayGoodbye` method).
3. Your answers to the 3 quiz questions.

Take your time. Once you reply, we will move to **Lesson 2: What is `application.yml`?**, where we will learn how to change the port and read settings!