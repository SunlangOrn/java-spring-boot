# 📘 Phase 2, Lesson 2: Configuration (`application.yml`)

## 🎯 Learning Goal
- ✅ Learn how to change the server port.
- ✅ Learn how to create custom settings and read them in Java.
- ✅ Understand that we can change app behavior *without* changing Java code.

---

## 💡 The Concept
Right now, our app runs on port `8080`. What if you want to run it on `8081`? What if you want to change a welcome message without recompiling the code? 

Spring Boot uses a file called `application.yml` (or `application.properties`) to store settings.

---

## 🛠️ Step-by-Step Build

### Step 1: Find the Configuration File
1. In your IDE, look at the left project tree.
2. Go to `src/main/resources/`.
3. You will see a file named `application.properties`. 
4. **Rename it** to `application.yml`. 
   *(Why? YAML `.yml` is cleaner and easier to read than `.properties`. Spring Boot supports both!)*

### Step 2: Change the Port
Open `application.yml` and add this:

```yaml
server:
  port: 8081
```
*Explanation: This tells the embedded Tomcat server to listen on port 8081 instead of the default 8080.*

### Step 3: Add a Custom Setting
Let's add our own custom setting below the server port:

```yaml
server:
  port: 8081

app:
  welcome-message: "Hello from the configuration file!"
```

### Step 4: Read the Setting in Java
Now, let's make our Controller read this message. Update your `HelloController.java`:

```java
package com.example.demo.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    // @Value reads the exact path from application.yml
    @Value("${app.welcome-message}")
    private String message;

    @GetMapping("/hello")
    public String sayHello() {
        return message; // Returns the text from the yml file!
    }
}
```

---

## 🚀 Run and Test

1. Restart your application.
2. Notice in the console logs it now says: `Tomcat started on port(s): 8081`.
3. Open your browser and go to: `http://localhost:8081/hello`
4. **Expected Result:** `Hello from the configuration file!`

---

## 🛠️ Exercise
1. Add a new setting in `application.yml` called `app.version` with the value `"1.0.0"`.
2. Add a new endpoint `@GetMapping("/version")` in your controller.
3. Use `@Value` to inject `app.version` and return it.
4. Test it in the browser.

---

## 🧠 Quiz
1. What file does Spring Boot use to read external settings?
2. If I change `server.port` to `9000`, what URL do I use to test my app?
3. What annotation is used to inject a value from `application.yml` into a Java variable?

---

## 🛑 STOP
Reply with your exercise code and quiz answers before moving to Lesson 3.