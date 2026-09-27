# 🖥️ Frontend Lesson 1: Next.js Setup, UI Library & Login Page

---

## 🎯 Goal
- ✅ Fix the "CORS" issue in Spring Boot so it can talk to Next.js.
- ✅ Set up a Next.js project with Tailwind CSS and the DaisyUI library.
- ✅ Build a beautiful Login page.
- ✅ Connect the Login page to the Spring Boot Auth API using Axios.

---

## 🧠 The Big Picture

Right now, your Spring Boot backend is running on **port 8081**. 
Your Next.js frontend will run on **port 3000**. 

If the frontend tries to talk to the backend, the browser will block it for security reasons. This is called **CORS** (Cross-Origin Resource Sharing). We must tell Spring Boot: *"It is safe to accept requests from my Next.js app."*

Once that is fixed, we will build the Login page using **DaisyUI** (a UI library that makes styling incredibly easy) and **Axios** (a tool to send HTTP requests to your backend).

---

## 📖 Key Words

| Word | Simple Meaning |
|------|---------------|
| **Next.js** | A React framework for building fast, modern web frontends. |
| **Tailwind CSS** | A CSS framework that lets you style elements using utility classes (like `text-red-500`). |
| **DaisyUI** | A plugin for Tailwind that gives you pre-built components (like `btn`, `card`, `input`). |
| **Axios** | A JavaScript library used to make HTTP requests (GET, POST) to your Spring Boot API. |
| **CORS** | A security feature. We must configure it to allow the frontend and backend to communicate. |

---

## 🛠️ Step 0: Fix CORS in Spring Boot (Backend)

### What we're doing:
Tell Spring Boot to allow requests coming from `http://localhost:3000` (where Next.js will run).

### The Code:
Create file: `src/main/java/com/example/demo/config/CorsConfig.java`

```java
package com.example.demo.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**") // Apply to all endpoints
                .allowedOrigins("http://localhost:3000") // Allow Next.js
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS") // Allow these HTTP methods
                .allowedHeaders("*") // Allow any headers (like Authorization)
                .allowCredentials(true); // Allow cookies/tokens
    }
}
```

### 📝 After the Code - What Just Happened?
- `@Configuration`: Tells Spring to load this class when the app starts.
- `allowedOrigins("http://localhost:3000")`: This is the magic line. It opens the door specifically for your Next.js app.
- `allowCredentials(true)`: Required if we ever want to send cookies or specific auth headers.

### 📦 Imports to Remember
```java
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
```

---

## 🛠️ Step 1: Create the Next.js App

### What we're doing:
Generate a brand new Next.js project with TypeScript and Tailwind CSS pre-configured.

### The Commands:
Open your terminal (outside of your Spring Boot folder) and run:

```bash
npx create-next-app@latest shopcore-frontend
```

**When it asks you questions, choose exactly this:**
- TypeScript? **Yes**
- ESLint? **Yes**
- Tailwind CSS? **Yes**
- `src/` directory? **Yes**
- App Router? **Yes**
- Import alias? **No** (Just press Enter for default)

### 📝 After the Code - What Just Happened?
You now have a fully working Next.js app. 
To start it, run:
```bash
cd shopcore-frontend
npm run dev
```
Open your browser to **http://localhost:3000**. You will see the default Next.js welcome page!

---

## 🛠️ Step 2: Install DaisyUI (The UI Library)

### What we're doing:
Add the DaisyUI plugin to Tailwind so we can use beautiful pre-built components.

### The Commands:
Inside your `shopcore-frontend` folder, run:

```bash
npm install -D daisyui@latest
```

### The Code:
Open `tailwind.config.ts` and add `daisyui` to the plugins array:

```typescript
// tailwind.config.ts
import type { Config } from "tailwindcss";

const config: Config = {
  content: [
    "./src/pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/components/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: { extend: {} },
  plugins: [
    require('daisyui'), // <-- ADD THIS LINE
  ],
};
export default config;
```

### 📝 After the Code - What Just Happened?
Now, instead of writing 20 lines of CSS to make a button look nice, you can just write `<button className="btn btn-primary">Click Me</button>`. DaisyUI handles the rest!

---

## 🛠️ Step 3: Install Axios (The API Caller)

### What we're doing:
Install the library we will use to send POST requests to our Spring Boot `/api/auth/login` endpoint.

### The Command:
```bash
npm install axios
```

---

## 🛠️ Step 4: Create the API Client

### What we're doing:
Create a central place to configure Axios so we don't have to type the backend URL (`http://localhost:8081`) everywhere.

### The Code:
Create file: `src/lib/api.ts`

```typescript
import axios from 'axios';

// Create a pre-configured Axios instance
const apiClient = axios.create({
  baseURL: 'http://localhost:8081', // Your Spring Boot backend URL
  headers: {
    'Content-Type': 'application/json',
  },
});

export default apiClient;
```

### 📝 After the Code - What Just Happened?
- `axios.create()`: Creates a reusable configuration. Now, whenever we use `apiClient.post('/api/auth/login')`, it automatically adds `http://localhost:8081` to the beginning!

---

## 🛠️ Step 5: Build the Login Page UI

### What we're doing:
Create a beautiful Login page using Next.js App Router and DaisyUI components.

### The Code:
Create file: `src/app/login/page.tsx`

```tsx
// This tells Next.js this component uses React hooks (useState)
"use client"; 

import { useState } from 'react';
import apiClient from '@/lib/api'; // Import our Axios client

export default function LoginPage() {
  // 1. State to hold the form inputs
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  // 2. The function that runs when the form is submitted
  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault(); // Stop the page from reloading
    setError('');
    setLoading(true);

    try {
      // 3. Call the Spring Boot API
      const response = await apiClient.post('/api/auth/login', {
        username,
        password,
      });

      // 4. If successful, save the token (we will use localStorage for now)
      const token = response.data.data.accessToken;
      localStorage.setItem('accessToken', token);
      
      alert('Login successful! Token saved.');
      // window.location.href = '/'; // Redirect to home page later
      
    } catch (err: any) {
      // 5. If the backend returns an error (like 401 Unauthorized)
      setError(err.response?.data?.message || 'Invalid username or password');
    } finally {
      setLoading(false);
    }
  };

  // 6. The HTML (JSX) using DaisyUI classes
  return (
    <div className="hero min-h-screen bg-base-200">
      <div className="hero-content flex-col lg:flex-row-reverse">
        <div className="text-center lg:text-left">
          <h1 className="text-5xl font-bold">ShopCore</h1>
          <p className="py-6">Welcome back! Please login to your account to continue shopping.</p>
        </div>
        
        <div className="card shrink-0 w-full max-w-sm shadow-2xl bg-base-100">
          <form className="card-body" onSubmit={handleLogin}>
            <div className="form-control">
              <label className="label">
                <span className="label-text">Username</span>
              </label>
              {/* daisyui 'input' and 'input-bordered' classes make it look great */}
              <input 
                type="text" 
                placeholder="username" 
                className="input input-bordered" 
                required 
                value={username}
                onChange={(e) => setUsername(e.target.value)}
              />
            </div>
            
            <div className="form-control">
              <label className="label">
                <span className="label-text">Password</span>
              </label>
              <input 
                type="password" 
                placeholder="password" 
                className="input input-bordered" 
                required 
                value={password}
                onChange={(e) => setPassword(e.target.value)}
              />
            </div>

            {error && <p className="text-red-500 text-sm mt-2">{error}</p>}

            <div className="form-control mt-6">
              {/* daisyui 'btn' and 'btn-primary' classes */}
              <button 
                type="submit" 
                className={`btn btn-primary ${loading ? 'loading' : ''}`}
                disabled={loading}
              >
                {loading ? 'Logging in...' : 'Login'}
              </button>
            </div>
          </form>
        </div>
      </div>
    </div>
  );
}
```

### 📝 After the Code - What Just Happened?
- `"use client"`: Required in Next.js App Router when you use React hooks like `useState`.
- `useState`: React's way of remembering data. When the user types, `setUsername` updates the `username` variable, and the screen re-renders.
- `apiClient.post(...)`: Sends the JSON to Spring Boot. Notice we extract the token using `response.data.data.accessToken` because our backend wraps everything in `HttpBodyResponse`!
- **DaisyUI Magic**: Look at the classes! `hero`, `card`, `input input-bordered`, `btn btn-primary`. We didn't write a single line of CSS, but it looks like a professional, modern website!

### 📦 Imports to Remember
```typescript
import { useState } from 'react';
import apiClient from '@/lib/api';
```

---

## 🧪 Run & Test

1. **Start the Backend:** Ensure Spring Boot is running on port 8081.
2. **Start the Frontend:** Run `npm run dev` in the Next.js folder (port 3000).
3. **Open Browser:** Go to **http://localhost:3000/login**.
4. **Test it:** 
   - Enter a wrong password. You should see the red error text: "Invalid username or password".
   - Enter the correct password. You should see the alert: "Login successful! Token saved."
5. **Verify:** Open your browser's Developer Tools (F12) -> Application -> Local Storage. You will see the `accessToken` saved!

---

## ⚠️ Common Mistakes

| Mistake | Why It Happens | How to Fix |
|---------|---------------|------------|
| `CORS policy` error in browser console | Spring Boot is blocking the request. | Ensure `CorsConfig.java` is created and the backend is restarted. |
| `Cannot find module '@/lib/api'` | Next.js doesn't know what `@` means. | Ensure you selected "Yes" for `src/` directory when creating the app, and check `tsconfig.json` has the `@/*` path alias. |
| `response.data.data.accessToken` is undefined | The backend response structure changed. | Check the exact JSON structure in Postman/Swagger and adjust the path. |
| Page reloads when I click Login | Forgot to prevent default form behavior. | Ensure `e.preventDefault()` is at the top of `handleLogin`. |

---

## ✏️ Exercise

1. Create a Register page at `src/app/register/page.tsx`.
2. Use the same DaisyUI layout, but add an "Email" input field.
3. Connect it to `apiClient.post('/api/auth/register', { username, email, password })`.
4. Add a link at the bottom of the Login page: "Don't have an account? Register" that navigates to `/register`.

---

## 🧠 Quiz

1. Why do we need the `CorsConfig.java` file in Spring Boot?
2. What does the `"use client"` directive do in Next.js?
3. Why do we use `localStorage.setItem('accessToken', token)` after a successful login?

---

## 🛑 STOP

Reply with:
1. Confirmation that you successfully logged in and saw the token in Local Storage.
2. Your code for the Exercise (the Register page).
3. Your answers to the 3 quiz questions.

Once you reply, we will move to **Frontend Lesson 2: Building the Product Catalog with Pagination and Beautiful Cards!** 🛒